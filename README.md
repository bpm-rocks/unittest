BPM Package: UnitTest
=====================

This is a very simplified and lightweight unit testing framework. It is geared to test Bash functions, but is also handy for running any shell commands. It requires [`bpm`] to run.

This is a lighter alternative to [Bats](https://github.com/sstephenson/bats) that runs without warnings when you use `set -eE` and doesn't preprocess your script file. It allows for testing return codes in a typical Bash method, unlike [`assert.sh`](https://github.com/lehmannro/assert.sh). This also provides a cleaner output, testing in different contexts, and timing information as opposed to writing your own, simpler testing script. [`shunit2`](https://github.com/kward/shunit2) is an interesting alternative if you want the numerous assertion functions; this framework assumes you can use `[[` to test if strings are equal.


Installation
------------

First, go get [`bpm`] and install it. Next, you simply run

    bpm install unittest

This will go download the program and put it in your user's local installation of commands.

You could also list this in your `bpm.ini` file under the `devDependencies` section. If you do it this way, you can use `bpm install` to get the software and all dependencies.

    [devDependencies]
    0=unittest


Getting Started
---------------

First, you need to write a test. Here is a simple test that only checks some built-in math.

    #!/usr/bin/env bash
    . bpm
    bpm::include unittest
    unittest::run

    unittest::case::two-plus-two-equals-four() {
        [[ "$((2 + 2))" == 4 ]]
    }

Save the file as `simple-math.test` and use `chmod a+x simple-math.test` to make it executable. Run the test with `./simple-math.test` and this should be the output:

    ....
    4 passed, 0 failed in 0.024341684 seconds.

Success! In case you are wondering about the 4 tests, there's more on that later. For now, just understand that it has to do with the different contexts that Bash can run in.

Another way to run the test is to use the `unittest` command. This will run the files as a suite. There's not a lot of difference between the two ways you can run the tests.

    unittest *.test

You can add more test cases, each with their own setup and tear down functions. You can also run multiple test files together as a test suite.


Test Cases
----------

A test case is a single unit test of your function or program. It must start with `unittest::case::` and be a valid function name. Here is another that confirms a file exists.

    unittest::case::data-file-is-downloaded() {
        curl -O data.file http://example.com/data.file
        [[ -f data.file ]]
    }

The return status of the function will often be the value from the test (`[[`) command at the end of the script. When strict mode is enabled (more on that later), the `curl` command will fail because of the invalid URL and the test will never proceed to the `[[` command.

If you must check multiple aspects, make sure that you include `return` statements. Here's an example that tests for the presence of keywords.

    unittest::case::keywords() {
        local contents

        contents=$(curl ./data-file) || return 1
        [[ "$contents" == *keyword* ]] || return 1
        [[ "$contents" == *another_keyword* ]]
    }

In this example we show how to use `return` because of the `strict-ignored` test mode. Just make sure that any command that should determine the outcome of the test is either last or has a `return` statement and everything should be golden.

When test cases are executed, they all have the following set up:

* The current working directory is where the test case's file is located.
* A `unittest::setup` function will have been executed.
* When the test ends, a `unittest::teardown` function will run, regardless of the test results.
* A `$UNITTEST_MODE` variable is set so the test can handle specific modes differently.

Unit tests must have unique names. This is because you are simply running Bash functions and Bash doesn't allow for you to have two identical function names. Names can be reused and repeated in different files. The restriction is only regarding the test names in a single file.


### Test Contexts

Bash can operate in different contexts, so the testing framework supports testing in all of these:

* `default` - Use the current environment's settings.
* `loose` - Use `set +eEu +o pipefail` to disable any sort of error catching.
* `strict` - Use `set -eEu -o pipefail` to enable all possible error catching.
* `strict-ignored` - Identical to `strict`, except Bash is set up to be in the special context where error checking is completely ignored.

When `strict` is running, a test like this will fail. It's in `example/test-contexts.test`. This one test case will report a message like `$1: unbound variable` as expected when executing in `strict` mode.

    unittest::case::undefined-variable() {
        echo "$1" &> /dev/null
    }

The `strict-ignore` mode is a bit weird because other types of errors do not trigger errors. To illustrate what I mean, here is another test. Again, it's a portion of `example/test-contexts.test`.

    unittest::case::command-fails-but-next-command-works() {
        local result="$(
            set +eE
            trap - ERR
            (
                set -eE
                false
                echo "Should not get here with 'exit immediately' enabled"
            )
        )"
        if [[ -n "$result" ]]; then
            echo "$UNITTEST_MODE output: $result"
            return 1
        fi
    }

If you walk through the code you will see that the `$result` will contain the output of whatever the subshells provide. The first subshell turns off error handling. It does this so the result can be captured. A second subshell is started where the errors are enabled again and then `false` is executed. Normally one would expect that program execution will stop, so the `echo` simply states that as a comment.

When the test is executed, there is a state where Bash will fail the above test. There's a better write-up about it in the [`strict`] library. Don't take my word for it, run `example/test-contexts.test` yourself and see it in action!

You can change the modes that are tested for a suite by adding one line to the beginning of your test file.

    #!/usr/bin/env bash
    . bpm
    bpm::include unittest

    # This is the line you must add. Specify any combinarion of the
    # available modes here.
    unittest::setModes strict loose

    unittest::run


Test Suites
-----------

A collection of related tests is called a suite. For this software, all of those tests are together in a single file. You can run multiple unit tests in one suite and you can run multiple suites together to fully test your software.


### Unit Test Setup and Tear Down

Each suite share a setup and tear down function. If you need to assign environment variables, prepare files, or load a library then `unittest::setup` is the right spot. Likewise, cleaning up temporary files or restoring a database would be good in a `unittest::teardown` function.

    # Set an environment variable, create a temporary file.
    unittest::setup() {
        export TEMPFILE=$(mktemp)
    }

    # Delete the temporary file
    unittest::teardown() {
        rm -f "$TEMPFILE"
    }

The `unittest::setup` function always runs just before a test executes. `unittest::teardown` is placed on an `EXIT` hook so it always runs just after a test executes. If there is an error in the setup, the test runner will catch it, flag the test as having failed, and not run the test. When there is an error in the tear down function, it also will get caught and reported as a test failure.


[`bpm`]: https://bpm.rocks
[`strict`]: https://github.com/bpm-rocks/strict


#!/usr/bin/env bash
. bpm
bpm::include unittest

if [[ "$#" -lt 1 ]]; then
    cat <<'EOF' >&2
Bash Unit Test

Specify unit test files as arguments to this command.

Example:

    unittest test/*.test
EOF
fi

unittest::runSuite "$@"
