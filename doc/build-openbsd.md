OpenBSD build guide
======================
(updated for OpenBSD 6.1)

This guide describes how to build garlicoind and command-line utilities on OpenBSD.

OpenBSD is most commonly used as a server OS, so this guide does not contain instructions for building the GUI.

Preparation
-------------

Run the following as root to install the base dependencies for building:

```bash
pkg_add git gmake libevent libtool
pkg_add autoconf # (select highest version, e.g. 2.69)
pkg_add automake # (select highest version, e.g. 1.15)
pkg_add python # (select highest version, e.g. 3.5)
```

The default C++ compiler that comes with OpenBSD 5.9 is g++ 4.2. This version is old (from 2007), and is not able to compile the current version of Garlicoin Core, primarily as it has no C++11 support, but even before there were issues. So here we will be installing a newer compiler.

GCC
-------

git clone https://github.com/GarlicoinOrg/Garlicoin.git
```

See [dependencies.md](dependencies.md) for a complete overview.

### Building boost

Do not use `pkg_add boost`! The boost version installed thus is compiled using the `g++` compiler not `eg++`, which will result in a conflict between `/usr/local/lib/libestdc++.so.XX.0` and `/usr/lib/libstdc++.so.XX.0`, resulting in a test crash:

    test_garlicoin:/usr/lib/libstdc++.so.57.0: /usr/local/lib/libestdc++.so.17.0 : WARNING: symbol(_ZN11__gnu_debug17_S_debug_me ssagesE) size mismatch, relink your program
    ...
    Segmentation fault (core dumped)

This makes it necessary to build boost, or at least the parts used by Garlicoin Core, manually:

```
# Pick some path to install boost to, here we create a directory within the garlicoin directory
GARLICOIN_ROOT=$(pwd)
BOOST_PREFIX="${GARLICOIN_ROOT}/boost"
mkdir -p $BOOST_PREFIX

# Fetch the source and verify that it is not tampered with
curl -o boost_1_64_0.tar.bz2 https://netcologne.dl.sourceforge.net/project/boost/boost/1.64.0/boost_1_64_0.tar.bz2
echo '7bcc5caace97baa948931d712ea5f37038dbb1c5d89b43ad4def4ed7cb683332 boost_1_64_0.tar.bz2' | sha256 -c
# MUST output: (SHA256) boost_1_64_0.tar.bz2: OK
tar -xjf boost_1_64_0.tar.bz2

# Boost 1.64 needs one small patch for OpenBSD
cd boost_1_64_0
# Also here: https://gist.githubusercontent.com/laanwj/bf359281dc319b8ff2e1/raw/92250de8404b97bb99d72ab898f4a8cb35ae1ea3/patch-boost_test_impl_execution_monitor_ipp.patch
patch -p0 < /usr/ports/devel/boost/patches/patch-boost_test_impl_execution_monitor_ipp

# Build w/ minimum configuration necessary for garlicoin
echo 'using gcc : : eg++ : <cxxflags>"-fvisibility=hidden -fPIC" <linkflags>"" <archiver>"ar" <striper>"strip"  <ranlib>"ranlib" <rc>"" : ;' > user-config.jam
config_opts="runtime-link=shared threadapi=pthread threading=multi link=static variant=release --layout=tagged --build-type=complete --user-config=user-config.jam -sNO_BZIP2=1"
./bootstrap.sh --without-icu --with-libraries=chrono,filesystem,program_options,system,thread,test
./b2 -d2 -j2 -d1 ${config_opts} --prefix=${BOOST_PREFIX} stage
./b2 -d0 -j4 ${config_opts} --prefix=${BOOST_PREFIX} install
```

### Building BerkeleyDB

BerkeleyDB is only necessary for the wallet functionality. To skip this, pass
`--disable-wallet` to `./configure` and skip to the next section.

It is recommended to use Berkeley DB 4.8. You cannot use the BerkeleyDB library
from ports, for the same reason as boost above (g++/libstd++ incompatibility).
If you have to build it yourself, you can use [the installation script included
in contrib/](/contrib/install_db4.sh) like so

```bash
# Pick some path to install BDB to, here we create a directory within the garlicoin directory
GARLICOIN_ROOT=$(pwd)
BDB_PREFIX="${GARLICOIN_ROOT}/db4"
mkdir -p $BDB_PREFIX

# Fetch the source and verify that it is not tampered with
curl -o db-4.8.30.NC.tar.gz 'http://download.oracle.com/berkeley-db/db-4.8.30.NC.tar.gz'
echo '12edc0df75bf9abd7f82f821795bcee50f42cb2e5f76a6a281b85732798364ef  db-4.8.30.NC.tar.gz' | sha256 -c
# MUST output: (SHA256) db-4.8.30.NC.tar.gz: OK
tar -xzf db-4.8.30.NC.tar.gz

# Build the library and install to specified prefix
cd db-4.8.30.NC/build_unix/
#  Note: Do a static build so that it can be embedded into the executable, instead of having to find a .so at runtime
../dist/configure --enable-cxx --disable-shared --with-pic --prefix=$BDB_PREFIX CC=egcc CXX=eg++ CPP=ecpp
make install # do NOT use -jX, this is broken
```

from the root of the repository. Then set `BDB_PREFIX` for the next section:

```shell
export BDB_PREFIX="$PWD/db4"
```

### Building Garlicoin Core

**Important**: use `gmake`, not `make`. The non-GNU `make` will exit with a horrible error.

Preparation:
```bash
export AUTOCONF_VERSION=2.69 # replace this with the autoconf version that you installed
export AUTOMAKE_VERSION=1.15 # replace this with the automake version that you installed
./autogen.sh
```
Make sure `BDB_PREFIX` is set to the appropriate path from the above steps.

To configure with wallet:
```bash
./configure --with-gui=no CC=cc CXX=c++ \
    BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" BDB_CFLAGS="-I${BDB_PREFIX}/include"
```

To configure without wallet:
```bash
./configure --disable-wallet --with-gui=no CC=cc CXX=c++
```

Build and run the tests:
```bash
gmake # use -jX here for parallelism
gmake check
```

Resource limits
-------------------

If the build runs into out-of-memory errors, the instructions in this section
might help.

The standard ulimit restrictions in OpenBSD are very strict:

    data(kbytes)         1572864

This, unfortunately, in some cases not enough to compile some `.cpp` files in the project,
(see issue [#6658](https://github.com/bitcoin/bitcoin/issues/6658)).
If your user is in the `staff` group the limit can be raised with:

    ulimit -d 3000000

The change will only affect the current shell and processes spawned by it. To
make the change system-wide, change `datasize-cur` and `datasize-max` in
`/etc/login.conf`, and reboot.

