# build scaffold for reflection library,  using git submodules

## build + install
```
$ cd xo-submodule-sm
$ mkdir build
$ cd build
$ INSTALL_PREFIX=/usr/local   # for example
$ cmake -DCMAKE_INSTALL_PREFIX=${INSTALL_PREFIX} ..
$ make
$ make install
```

## update all submodules in dev tree
```
$ cd xo-submodule
$ git submodule update --recursive --remote  # preserves branch
```

## run unit tests
```
$ cd xo-submodule/build
$ ctest
```

## build+run code coverage

note:
- Individual codebases use cmake code-coverage module.
- Not using that here,  because it doesn't work with external cmake projects.

```
$ mkdir xo-submodule/ccov
$ cd xo-submodule/ccov
$ cmake -DCODE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug ..
$ make -j     # build with code coverage instrumentation
```

capture raw coverage data by running instrumented executables
```
$ ctest       # coverage info captured in .gcda files
```

post-process coverage data to useful report
```
$ lcov -c --directory . --output-file cov1.info                  # capture coverage data from .gcda files
$ lcov --remove cov1.info '/nix/store*' --output-file cov2.info  # suppress outside source files
$ lcov --remove cov2.info $(pwd)'/local/*' --output-file cov3.info       # suppress outside-codebase files
$ genhtml cov3.info --output-directory html
```
To view report,  point browser to `file:///path/to/xo-submodule/ccov/html/index.html`
