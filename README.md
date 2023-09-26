# build scaffold for reflection library,  using git submodules

### build + install
```
$ cd reflect-sm
$ mkdir build
$ cd build
$ INSTALL_PREFIX=/usr/local   # for example
$ cmake -DCMAKE_INSTALL_PREFIX=${INSTALL_PREFIX} ..
$ make
$ make install
```
