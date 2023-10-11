Code Coverage
-------------

PowerDNS uses [coveralls](https://coveralls.io/) to generate code coverage reports from our Continuous Integration tests. The resulting analysis can then be consulted [online](https://coveralls.io/github/PowerDNS/pdns), and gives insight into which parts of the code are automatically tested.

Code coverage is generated during our Continuous Integration tests, for every pull request. In addition to the dashboard on Coveralls' website, a summary is posted on pull requests.

# Technical Details

## DebugInfo vs Source-based Code Coverage

There are two main ways of generating code coverage: `GCOV` and `source-based`.

### GCOV

The `GCOV` approach, supported by both `g++` and `clang++`, is enabled by passing the `--coverage` flag (equivalent to `-ftest-coverage -fprofile-arcs`) to the compiler and linker. It operates on debugging information (`DebugInfo`), usually [DWARF](https://dwarfstd.org/), generated by the compiler, and also used by debuggers.
This approach generates `.gcno` files during the compilation, which are stored along the object files, and `.gcda` files at runtime when the final program is executed.

* There are as many `.gcno` and `.gcda` files as object files, which may be a lot.
* Every invocation of a program updates the `.gcda` files corresponding to the code that has been executed. It will append to existing `.gcda` files, but only process can update a given file so parallel execution will result in corrupted data.
* Writing to each `.gcda` might take a while for large programs, and has been known to slow down execution quite a lot.
* Accurate reporting of lines and branches may be problematic when optimizations are enabled, so it is advised to disable optimizations to get useful analysis.
* Note that the `.gcda` files produced by `clang++` are not fully compatible with the `g++` ones, and with the existing tools, but [`llvm-cov gcov`](https://llvm.org/docs/CommandGuide/llvm-cov.html#llvm-cov-gcov) can produce `.gcov` files that should be compatible. A symptom of this incompatiblity looks like this:

```
Processing pdns/ednssubnet.gcda
__w/pdns/pdns/pdns/ednssubnet.gcno:version '408', prefer 'B02'
```

### Source Based

`clang++` supports [source-based coverage](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html), which operates on `AST` and preprocessor information directly. This is enabled by passing `-fprofile-instr-generate -fcoverage-mapping` to the compiler and leads to `.profraw` files being produced when the binary is executed. 
The `.profraw` file(s) can be merged by [`llvm-profdata merge`](https://llvm.org/docs/CommandGuide/llvm-profdata.html#profdata-merge) into a `.profdata` file which can then be used by [`llvm-cov show`](https://llvm.org/docs/CommandGuide/llvm-cov.html#llvm-cov-show) to generate HTML and text reports, or by [`llvm-cov export`](https://llvm.org/docs/CommandGuide/llvm-cov.html#llvm-cov-export) to export `LCOV` data that is compatible with other tools.

* Source-based coverage can generate accurate data with optimizations enabled, and has a much lower overhead that `GCOV`.
* The path and exact name of the `.profraw` files generated when a program is executed can be controlled via the `LLVM_PROFILE_FILE` environment variable, which supports [patterns](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html#running-the-instrumented-program) like `%p`, which expands to the process ID. That allows running several programs in parallel, each program generating its own file at the end.

## Implementation

We use `clang++`'s source-based coverage method in our CI, as it allows running our regression tests in parallel with several workers. It is enabled by passing the `--enable-coverage=clang` flag during `configure` for all products.
The code coverage generation is done as part of the [build-and-test-all.yml](https://github.com/PowerDNS/pdns/blob/master/.github/workflows/build-and-test-all.yml) workflow.

Since we have a `monorepo` for three products which share the same code-base, the process is a bit tricky:

* We use coveralls's `parallel` feature, which allows us to generate partial reports from several steps of our CI process, then merge them during the `collect` phase and upload the resulting `LCOV` file to coveralls.
* After executing our tests, the `generate_coverage_info` method in [`tasks.py`](https://github.com/PowerDNS/pdns/blob/master/tasks.py) merges the `.profraw` files that have been generated every time a binary has been executed into a single `.profdata` file via [`llvm-profdata merge`](https://llvm.org/docs/CommandGuide/llvm-profdata.html#profdata-merge). We enable the `sparse` mode to get a smaller `.profdata` file, since we do not do Profile-Guided Optimization (PGO).
* It then generates a `.lcov` file from the `.profdata` via [`llvm-cov export`](https://llvm.org/docs/CommandGuide/llvm-cov.html#llvm-cov-export), telling it to ignore reports for files under `/usr` in the process (via the `-ignore-filename-regex` parameter).
* We then normalize the paths of the source files to prevent duplicates for files that are used by more than one product, and to account for the fact that our CI actually compiles from a `distdir`. This is handled by a Python script, [.github/scripts/normalize_paths_in_coverage.py](https://github.com/PowerDNS/pdns/blob/master/.github/scripts/normalize_paths_in_coverage.py) that parses the `LCOV` data and updates the paths.
* We call [Coveralls's github action](https://github.com/coverallsapp/github-action) to upload the resulting `LCOV` data for this step.
* After all steps have completed, we call that action again to let it know that our workflow is finished and the data can be consolidated.

One important thing to remember is that the content is only written into a `.profraw` file is the program terminates correctly, calling `exit` handlers, and if the `__llvm_profile_write_file()` function is called. Our code base has a wrapper around that, `pdns::coverage::dumpCoverageData()`.
This is especially important for us because our products often terminates by calling `_exit()`, bypassing the `exit` handlers, to avoid issues with the destruction order of global objects.

## Generating Coverage Outside Of the CI

It is possible to generate a code coverage report without going through the CI, for example to test the coverage of a new feature in a given product.

### Source-based Coverage With clang++

* Run the `configure` script with the `--enable-coverage=clang` option, setting the `CC` and `CXX` environment variables to use the `clang` compiler: `CC=clang CXX=clang++ ./configure --enable-coverage=clang`
* Compile the product as usual with: `make`
* Run the test(s) that are expected to cover the new feature, via `./testrunner` or `make check` for the unit tests, and the instructions of the corresponding `regression-tests*` directory for the regression tests. It is advised to set the `LLVM_PROFILE_FILE` environment variable in such a way that an invocation of the product do not override the results from the previous invocation. For example setting `LLVM_PROFILE_FILE="/tmp/code-%p.profraw"` will result in each invocation writing a new file into the `/tmp` directory, replacing `%p` with the process ID.
* Merge the resulting `*.profraw` file into a single `code.profdata` file by running `llvm-profdata merge -sparse -o /tmp/code.profdata /tmp/code-*.profraw`
* Generate a HTML report into the `/tmp/html-report` directory by running `llvm-cov show --instr-profile /tmp/code.profdata -format html -output-dir /tmp/html-report -object </path/to/product/binary>`

### GCOV

* Run the `configure` script with the `--enable-coverage` option, using either `g++` or `clang++`: `./configure --enable-coverage`
* Compile as usual with: `make`. This will generate `.gcno` files along with the usual `.o` object files and the final binaries.
* Run the test(s) that are expected to cover the new feature, via `./testrunner` or `make check` for the unit tests, and the instructions of the corresponding `regression-tests*` directory for the regression tests. Note that the regression should not be run in parallel, as it would corrupt the `.gcna` files that will be generated in the process. For dnsdist, that means running `pytest` without the `--dist=loadfile -n auto` options.
* Generate a HTML report using `gcovr`, or `gcov` then `lcov`

# Remaining Tasks

The way our code coverage report is generated does not currently handle the different authoritative server tools (that end up in the `pdns-tools` package) very well. Consequently the coverage report for these tools, and the related code parts, is not accurate.
It is likely possible to pass several `--object </path/to/binary>` options to `llvm-cov` when processing the `.profdata` file.