# build scaffold for reflection library,  using git submodules

### build + install
```
$ cd xo-submodule-sm
$ mkdir build
$ cd build
$ INSTALL_PREFIX=/usr/local   # for example
$ cmake -DCMAKE_INSTALL_PREFIX=${INSTALL_PREFIX} ..
$ make
$ make install
```

# update all submodules in dev tree
```
$ cd xo-submodule
$ git submodule update --recursive --remote  # preserves branch
```
