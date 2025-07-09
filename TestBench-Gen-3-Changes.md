![TestBench](images/test-bench-icon-130x115.png)

# TestBench Gen 3 Changes

Date: Wed Jul 9 2025 \
Author: Nathan Ladd

## Summary

A prerelease for TestBench Gen 3 is now available. It can be tested by updating your
project's `Gemfile`:

```ruby
[
  "test_bench-telemetry",
  "test_bench-session",
  "test_bench-output",
  "test_bench-fixture",
  "test_bench-run",
  "test_bench"
].each do |test_bench_gem_name|
  gem test_bench_gem_name, ">= 3.0.0.0.pre.1"
end
```

A list of changes can be found [here](https://github.com/ntl/software-development/blob/main/TestBench-Gen-3-Changes.md).

Please report any issues or concerns to the `#test-bench` channel in the [Eventide Project's Slack organization](https://join.slack.com/t/eventide-project).

## Breaking Changes

- Session substitute
  - Some predicate method aliases have been removed from the session substitute
  - The volume of aliases was inviting variation, and they didn't appear to be used
  - The predicate methods that haven't been are declared in the substitute: https://github.com/test-bench/test-bench-session/blob/master/lib/test_bench/session/substitute.rb

- Assert and refute won't accept `nil` anymore. The supplied value must now be `true` or `false`
  - Error message indicates whether assert or refute is the cause

- Fixtures
  - Fixture _modules_, which were an experimental feature, were removed due to technical complications
  - Note: fixture _classes_, which are the more common form of fixtures, aren't affected
  - The `fixture` method now always forwards block argument to the fixture's constructor
    - Previously, if the fixture's constructor didn't include a block parameter, any given block was actuated by TestBench
    - In practice, the explicitness of the fixture's constructor accepting a block argument has always been preferred

- Replacement of `TEST_BENCH_ONLY_FAILURE` environment variable with `TEST_BENCH_OUTPUT_LEVEL`
  - Previously, an "only failure" output mode could be activated, but now, four output modes are available:
    - `abort` - Only show output from test files that abort due to an uncaught exception
    - `failure` - Like `abort`, but also include files that contain test failures
    - `not-passing` - Like `failure`, but also include files with skipped contexts or tests
    - `all` - Show all output (default)

## Restored Features

- Backtrace formatting
  - Ruby exception backtraces are often very difficult to scan due to the volume of contextually irrelevant frames
  - Frames can once again be filtered with `TEST_BENCH_OMIT_BACKTRACE_PATTERN`
    - It's a shell glob pattern, rather than a regular expression
    - Multiple patterns can be set by joining them with `:`. Example: `some-pattern*:some-other-pattern*`
  - Backtrace frames with absolute paths are also reformatted with relative paths when they're inside `Dir.pwd`
  - The batch runner's file summary prints the closest location to the exception that wasn't filtered

- Abort on failure (`TEST_BENCH_ABORT_ON_FAILURE`)
  - Stops batch runner after any test file fails

## New Features

- Random number generator, `TestBench::Random`
  - `TestBench::Random` is a module that can be included in classes or extended in modules. It provides four methods:
    - `string` - base 36 string
    - `integer` - positive integer
    - `decimal` - floating point decimal
    - `boolean` - either `true` or `false`
  - The random number generator produces 64-bit pseudorandom values
  - Pseudorandom seed can be controlled by the `TEST_BENCH_RANDOM_SEED` environment variable. Must be a base 36 string.

- Strict mode (`TEST_BENCH_STRICT`)
  - Controls the CLI exit status and boolean return value of `TestBench::Run.()`
    - Also controls the return values for `context` and `test` which are occasionally used for flow control
  - Strict: at least one test must be performed, and any skipped test or context will disqualify the run
  - Non-strict (default): there's no minimum number of tests, and skipped tests or contexts won't disqualify the run

- Comments and details
  - Both methods now accept a fixture instance, which prints the fixture output. Example:

    ```ruby
    fixture = SomeFixture.new

    fixture.()

    # Prints the test output of `fixture`
    comment fixture
    ```

    Note: the fixture's output is indented and unstyled in order to be visually distinct from the surrounding test output.

  - Optional heading argument that precedes the comment text. Example:

    ```ruby
    comment "Some Comment Heading:", "Some comment"
    ```

  - New `style` keyword argument. Values:
    - `detect` - (default) text that ends in a newline is rendered with the `block` style, otherwise the style is `normal`
    - `normal` - text is indented and then printed
    - `heading` - text is indented and printed with bold styling
    - `block` - each line of the text is indented with a preceding marker (`█`)
    - `line_number` - each line of the text is indented with a line number marker
    - `raw` - text is written verbatim with no indentation whatsoever

- Context titles are given a color based on their result
  - Previously, they were always green
  - If a context aborts due to an uncaught exception, the titles is now printed in red
  - Contexts that don't perform any tests aren't printed with any styling
    - Contexts that contain tests are still printed in green

- Output device can be controlled with the `TEST_BENCH_OUTPUT_DEVICE` environment variable. Values:
  - `stdout` - Output is printed to STDOUT (default)
  - `stderr` - Output is printed to STDERR
  - `null` - No output (silent operation)

- Output summary can be disabled with the `TEST_RUN_OUTPUT_SUMMARY` environment variable
  - Parallel test runners using GNU parallel often run many batches, which makes the summaries incomplete and misleading

- Command line interface now raises an error if given a file or directory that does not exist

## Internals

- Batch file execution happens in an isolated subprocess (using `fork`)
  - Exceptions are now never rescued by TestBench
    - The current isolated subprocess is discarded and replaced with a new one instead
  - Note: `exit!` in `binding.irb` will behave like `exit` when invoked from inside an isolated subprocess
  - Authors of parallel executors should consider making use of these isolated subprocesses. See: https://github.com/test-bench/test-bench-session/blob/master/lib/test_bench/session/isolate.rb

- Sessions now have a trace object (`fixture.test_session.trace`)
  - Encapsulates machinery for the matching predicates, e.g. `test_passed?("Some Context", "Some test")`
  - Implements `join`, which returns e.g. `Some Context :: Some Other Context :: Some test`
  - Tooling that concatenates test and context titles should consider using the session's trace object
