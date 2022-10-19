---
layout: default
title: Test File Format
permalink: /docs/TestFileFormat
nav_order: 6
has_children: true
has_toc: false
---

# Cmaj Unit Test Files

The test file format allows you to create test files which run multiple scripted tests of Cmajor code.

Javascript can be used to write custom functions that can perform kind of test, but a set of common helper functions are also provided.

A typical test might load a Cmaj program, check for compile errors, pass some inputs into it and check that the expected outputs are emitted.

## Running Tests

To run a test file, you can use the command-line tool:

```shell
$ cmaj test my_tests/TestFile1.cmajtest
```

You can also provide a folder instead of a filename, and it will scan for all the tests within that folder, and print out a set of total results for all of them.

The command-line switches include:

```
--singleThread      Use a single thread to run the tests
--threads=n         Run with the given number of threads, defaults to the available cores
--runDisabled       Run all tests including any marked disabled
--testToRun=n       Only run the specified test number in the test files
--xmlOutput=file    Generate a JUNIT compatible xml file containing the test results
--iterations=n      How many times to repeat the tests
```

Run `cmaj --help` for more command-line option information.

## File Format

A test file is split into blocks which are handled as separate tests.

Blocks are separated by a line starting with the characters `##`, followed by a javascript function call which performs a test function on the remaining content of the block that follows it.

e.g.

```
// All the text before the first delimiter line is parsed as javascript, and is available globally to all the tests. This is where you'd put any shared helper functions that your tests need to use.

## testFunction()

...The chunk of text in each section is provided to the test function that is called above...

## expectError ("...")

...Likewise, this chunk of text is passed to the "expectError" function...

```
Any content at the start of the file, up to the first `##` line, is parsed as Javascript. This is where you can define custom test functions which can be used multiple times within the file.

## Special Section Delimiters

### `## global`

The special delimiter `## global` is used to declare a chunk of code which will be prefixed onto all the other tests in the file. So if you're running many tests which all share a set of types or functions, you can avoid repeated code by putting the common definitions in a `global` section.

### `## disabled [test...]`

To quickly disable a test, insert the token `disabled` in front of the test directive (anything following `disabled` on the line is ignored), and the test will be counted as disabled in the results.

## Built-in Test Functions

A library of standard javascript test functions is available. The code for all of these is available, but the main ones include:

### `## testFunction`

This wraps the block of code in a dummy namespace, and finds all the functions which take no parameters and return a `bool`. Each of these is called, and if any of them return `false` it is counted as a failure.

e.g.

```
## testFunction()

bool test1()    { return 1 + 1 == 2; }  // this will pass
bool test2()    { return 1 + 2 == 3; }  // this will fail
int notATest() {}  // this function will be ignored since it doesn't return a bool
```

### `## testProcessor()`

This function will compile the block of code, and instantiate its main processor. The processor is expected to provide an output (either a `stream` or an `event` type) which emits `int` values.

The processor will then be run, and is expected to emit a stream of either 1 or 0 values, and then a -1 to indicate that the test is finished. If any 0 values are emitted, the test is registered as a fail, but if they're all ones, it's a pass.

e.g.

```
## testProcessor()

processor P
{
    output stream int out;

    void main()
    {
        out <- 1; advance();  // this is a pass
        out <- 0; advance();  // sending a zero will cause a fail to be logged

        out <- -1; advance(); // always send a -1 at the end to stop the test
    }
}
```

### `## testCompile()`

This simply checks that the code compiles without any compiler errors.

### `## testConsole()`

This instantiates a main processor from the code chunk that using the same system as `testProcessor()`. The output of the processor is ignored, but the content that was written to the console is checked against an expected value that is provided. e.g.

```
## testConsole ("hello world")

processor P
{
    output stream int out;
    void main()  { console <- "hello " <- "world"; out <- -1; advance(); }
}
```

### `## expectError ("<expected error message>")`

This wraps the chunk of code in a dummy namespace (to allow you to easily write free functions without any boilerplate) and attempts to compile it. If the compiler error matches the one specified in the test directive, it's a pass. If there's no error, or the error doesn't match, it's a fail.

```
## expectError ("2:9: error: Cannot find symbol 'XX'")

void f (XX& x) {}
```

## `## runScript()`

TODO

## `## testPatch()`

This will attempt to load and compile a patch from a filename provided. Any compile errors will be registered as a failure.
e.g.

```
## testPatch ("../../../my_patches/example_patch.cmajor_patch")
```

## Auto-updating Expected Results

For tests like `expectError` or `testConsole`, the directive contains a string which is the expected outcome of the test. To easily update these, the tool will automatically insert the correct result and re-save the test file if you write the directive without any arguments.

For example, if you create a test file which looks like this:

```
## expectError

void f (XX& x) {}
```

and then run the test tool on it, it will rewrite the test file to look like this:

```
## expectError ("2:9: error: Cannot find symbol 'XX'")

void f (XX& x) {}
```

## Creating Custom Test Functions

You can write your own javascript test functions at the start of the test file, and invoke them on a `##` line.

The javascript API available for test functions includes:

TODO