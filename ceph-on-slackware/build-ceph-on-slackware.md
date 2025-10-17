# Building Ceph 20.1.0 on Slackware

## Set Up Build Environment

### Packages

Make sure package sets A, AP, D, K, L, N, TCL, and X are installed.

Remove GCC Go.  It seems to cause nothing but problems, and I can't find a situation where it is needed over the official Go.  Slackware-current contains official Go that will be the default when GCC Go is removed.

```bash
sudo removepkg gcc-go-15.2.0-x86_64-1
```

#### tox

Some build scripts use this utility.  This is only required on the build system.

```bash
sudo pip install tox
```

#### pyo3 (with subinterpreters patch)

Source: <https://github.com/PyO3/pyo3>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/pyo3-subint>  
Build time: 20 sec

This package provides Rust bindings in Python.  It is patched to allow its use in Python sub-interpreters if the `unsafe-allow-subinterpreters` feature is requested.  This is required by bcrypt and cryptography for use with Ceph.  This does not need to be packaged, just installed on the build system.

```diff
diff -ruN pyo3-0.26.0/Cargo.toml pyo3-0.26.0-patched/Cargo.toml
--- pyo3-0.26.0/Cargo.toml    2025-08-29 08:43:50.000000000 -0400
+++ pyo3-0.26.0-patched/Cargo.toml    2025-09-13 15:05:13.285000000 -0400
@@ -141,6 +141,9 @@
 # Optimizes PyObject to Vec conversion and so on.
 nightly = []

+# Disables the checks for use in subinterpreters.  This is needed for bcrypt for Ceph.
+unsafe-allow-subinterpreters = []
+
 # Activates all additional features
 # This is mostly intended for testing purposes - activating *all* of these isn't particularly useful.
 full = [
diff -ruN pyo3-0.26.0/src/impl_/pymodule.rs pyo3-0.26.0-patched/src/impl_/pymodule.rs
--- pyo3-0.26.0/src/impl_/pymodule.rs    2025-08-29 08:43:50.000000000 -0400
+++ pyo3-0.26.0-patched/src/impl_/pymodule.rs    2025-09-13 15:06:51.833000000 -0400
@@ -100,7 +100,9 @@
         // that static data is not reused across interpreters.
         //
         // PyPy does not have subinterpreters, so no need to check interpreter ID.
-        #[cfg(not(any(PyPy, GraalPy)))]
+        // Also bypass this check if the feature is specifically enabled.  This is needed
+        // for bcrypt for Ceph.    
+        #[cfg(not(any(PyPy, GraalPy, feature = "unsafe-allow-subinterpreters")))]
         {
             // PyInterpreterState_Get is only available on 3.9 and later, but is missing
             // from python3.dll for Windows stable API on 3.9
```

Patch based on: <https://git.st8l.com/luxolus/pyo3/commit/338c71d0ad10f7ae38b7b44e576d49b91ed20d99>  

Apply patch:

```bash
patch -p1 < $CWD/pyo3-0.26.0-subinterpreters.patch
```

The patched pyo3 can then be built with *cargo*.

```bash
cargo build
```

#### bcrypt (with subinterpreters patch)

Source: <https://github.com/pyca/bcrypt/>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/bcrypt-subint>  
Build time: 25 sec

This package provides a hashing algorithm.  It is patched to request the `unsafe-allow-subinterpreters` feature from pyo3 to allow use in Python subinterpreters.  

```diff
diff -ruN bcrypt-4.3.0/src/_bcrypt/Cargo.toml bcrypt-4.3.0-patched/src/_bcrypt/Cargo.toml
--- bcrypt-4.3.0/src/_bcrypt/Cargo.toml    2025-02-27 20:17:02.000000000 -0500
+++ bcrypt-4.3.0-patched/src/_bcrypt/Cargo.toml    2025-09-13 15:29:52.475000000 -0400
@@ -6,7 +6,7 @@
 publish = false

 [dependencies]
-pyo3 = { version = "0.23.5", features = ["abi3"] }
+pyo3 = { version = "0.26", features = ["abi3", "unsafe-allow-subinterpreters" ], path="/tmp/SBo/pyo3-0.26.0-subint" }
 bcrypt = "0.17"
 bcrypt-pbkdf = "0.10.0"
 base64 = "0.22.1"
```

The patched **bcrypt** uses a Python build process using the *build module*.

```bash
patch -p1 < $CWD/bcrypt-4.3.0-subinterpreters.patch
python3 -m build
pip install dist/*.whl --root=$PKG --root-user-action ignore
```

#### cryptography (with subinterpreters patch)

Source: <https://github.com/pyca/cryptography>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/cryptography-subint>  
Build time: 20 sec

This package provides cryptographic "recipies and primitives."  It is patched to request the `unsafe-allow-subinterpreters` feature from pyo3 to allow use in Python subinterpreters.  A `cargo update` is required for the build system to pick up the changes.  

```diff
diff -ruN cryptography-45.0.7/Cargo.toml cryptography-45.0.7-subint/Cargo.toml
--- cryptography-45.0.7/Cargo.toml    2025-09-01 07:05:40.000000000 -0400
+++ cryptography-45.0.7-subint/Cargo.toml    2025-09-15 18:19:01.779000000 -0400
@@ -22,8 +22,8 @@

 [workspace.dependencies]
 asn1 = { version = "0.21.3", default-features = false }
-pyo3 = { version = "0.25", features = ["abi3"] }
-pyo3-build-config = { version = "0.25" }
+pyo3 = { version = "0.26", features = ["abi3", "unsafe-allow-subinterpreters" ], path="/tmp/SBo/pyo3-0.26.0-subint" }
+pyo3-build-config = { version = "0.26", path="/tmp/SBo/pyo3-0.26.0-subint/pyo3-build-config" }
 openssl = "0.10.72"
 openssl-sys = "0.9.108"
```

The patched **cryptography** uses a Python build process using the *build module*.

```bash
patch -p1 < $CWD/cryptography-45.0.7-subinterpreters.patch
cargo update
python3 -m build
pip install dist/*.whl --root=$PKG --root-user-action ignore
```

### Additional Python Packages

Additional Python packages are needed for Ceph.  These are not currently built from source, but are install via `pip` instead.  These packages will need to be installed on the build system and on systems that will run a Ceph component.

```bash
pip install scipy cherrypy jsonpatch python-dateutil prettytable jmespath xmltodict pyOpenSSL Routes
```

## Build the Ceph Prerequisites

#### 

Source: <>
SlackBuild: <>

This package provides the # library.  **#** uses a typical *cmake* build process.

#### rdma-core

Source: <https://github.com/linux-rdma/rdma-core>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/rdma-core>  
Build time: 4 min

This package provides the libiverbs library.  It is used by Ceph.  **rdma-core** uses a typical *cmake* build process.

```bash
mkdir -p build
cd build
cmake \
  -DCMAKE_C_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_CXX_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DLIB_SUFFIX=${LIBDIRSUFFIX} \
  -DMAN_INSTALL_DIR=/usr/man \
  -DCMAKE_PREFIX_PATH=/usr \
  -DCMAKE_SHARED_LIBRARY_PREFIX=/usr/lib64 \
  -DCMAKE_BUILD_TYPE=Release ${@:3} ..

make install DESTDIR=$PKG
```

#### libnbd

Source: <https://download.libguestfs.org/libnbd/1.22-stable/>  
SlackBuild: https://github.com/scottr131/slackbuilds/tree/main/libnbd  
Build time: 20 sec

This package provides a client library for the NBD protocol.  It is used by Ceph.  **libnbd** uses a typical *Autoconf* build process.

```bash
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux 

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### googletest

Source: <https://github.com/google/googletest>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/googletest>  
Build time: 20 sec

This package provides the Google testing and mocking framework.  It is used by Google **benchmark**.  **googletest** uses a typical *cmake* build process.

```bash
mkdir -p build
cd build
cmake \
  -DCMAKE_C_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_CXX_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DLIB_SUFFIX=${LIBDIRSUFFIX} \
  -DMAN_INSTALL_DIR=/usr/man \
  -DCMAKE_PREFIX_PATH=/usr \
  -DCMAKE_SHARED_LIBRARY_PREFIX=/usr/lib64 \
  -DCMAKE_BUILD_TYPE=Release ${@:3} ..

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### benchmark

Source: <https://github.com/google/benchmark>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/benchmark>  
Build time: 1 min

This package provides the benchmark library from Google.  It is used by **Snappy**. **benchmark ** uses a typical *cmake* build process.

```bash
mkdir -p build
cd build
cmake \
  -DCMAKE_C_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_CXX_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DLIB_SUFFIX=${LIBDIRSUFFIX} \
  -DBENCHMARK_USE_BUNDLED_GTEST=OFF \
  -DMAN_INSTALL_DIR=/usr/man \
  -DCMAKE_PREFIX_PATH=/usr \
  -DCMAKE_SHARED_LIBRARY_PREFIX=/usr/lib64 \
  -DCMAKE_BUILD_TYPE=Release ${@:3} ..

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

In addition, include a copy of the source files at /usr/src.  This may not be necessary and may be removed in the future, but may be useful during testing and debugging.

```bash
mkdir -p $PKG/usr/src/google-benchmark
cp -R * $PKG/usr/src/google-benchmark/
```

#### snappy

Source: <https://github.com/google/snappy>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/snappy-rtti>  
Build time: 30 sec

This package provides the Snappy compression library from Google.  It is used by Ceph.  **snappy** needs to be built with RTTI support for Ceph and requires a patch to enable this.  After the patch, **snappy** uses a typical *cmake* build process.

The patch below removes the lines from the cmake configuration that disable RTTI.  This may change.

```diff
diff --unified --recursive --text --new-file snappy-1.2.2.orig/CMakeLists.txt snappy-1.2.2/CMakeLists.txt
--- snappy-1.2.2.orig/CMakeLists.txt    2025-04-19 23:20:48.985433149 +0200
+++ snappy-1.2.2/CMakeLists.txt    2025-04-19 23:20:27.703912696 +0200
@@ -51,10 +51,6 @@
   string(REGEX REPLACE "/EH[a-z]+" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHs-c-")
   add_definitions(-D_HAS_EXCEPTIONS=0)
-
-  # Disable RTTI.
-  string(REGEX REPLACE "/GR" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
-  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /GR-")
 else(MSVC)
   # Use -Wall for clang and gcc.
   if(NOT CMAKE_CXX_FLAGS MATCHES "-Wall")
@@ -81,10 +77,6 @@
   # Disable C++ exceptions.
   string(REGEX REPLACE "-fexceptions" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
-
-  # Disable RTTI.
-  string(REGEX REPLACE "-frtti" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
-  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
 endif(MSVC)

 # BUILD_SHARED_LIBS is a standard CMake variable, but we declare it here to make
```

 Patch based on: <https://gitlab.archlinux.org/archlinux/packaging/packages/snappy/-/blob/main/snappy-reenable_rtti.patch?ref_type=heads>  

 Apply patch:

```bash
patch -p1 < $CWD/snappy-reenable_rtti.patch
```

The patched Snappy can then be built as follows:

```bash
mkdir -p build
cd build
cmake \
  -DCMAKE_C_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_CXX_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DLIB_SUFFIX=${LIBDIRSUFFIX} \
  -DMAN_INSTALL_DIR=/usr/man \
  -DCMAKE_PREFIX_PATH=/usr \
  -DCMAKE_CXX_FLAGS="-frtti" \
  -DBUILD_SHARED_LIBS=ON \
  -DCMAKE_SHARED_LIBRARY_PREFIX=/usr/lib64 \
  -DCMAKE_BUILD_TYPE=Release ${@:3} ..

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="-frtti $SLKCFLAGS" \
make -j ${BLDTHREADS:-1}

make install DESTDIR=$PKG
```

#### oath-toolkit

Source: <https://download.savannah.nongnu.org/releases/oath-toolkit/>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/oath-toolkit>  
Build time: 50 sec

This package provides components for one-time-password (OTP) authentication.  **oath-toolkit** uses a typical *Autoconf* build process.

```bash
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux 

make -j ${BLDTHREADS:-1}
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### numactl

Source: <https://github.com/numactl/numactl>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/numactl>  
Build time: 5 sec

This package provides NUMA policy support with the libnuma library.  It is used by Ceph. **numactl** uses a typical *Autoconf* build process.

```bash
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux 

make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### lttng-ust

Source: <https://lttng.org/>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/lttng-ust>  
Build time: 40 sec

This package provides a library for instrumentation and tracing.  It is used by babeltrace.  **lttng-ust** uses a typical *cmake* build process.

```bash
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux 

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### babeltrace

Source: <https://babeltrace.org/>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/babeltrace>  
Build time: 20 sec

This package provides the babeltrace trace read and write library.  **babeltrace** uses a typical *Autoconf* build process.

```bash
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --disable-static \
  --build=$ARCH-slackware-linux 

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### boost

Source: <https://www.boost.org/>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/boost>  
Build time: 10 min

This package provides various C++ source libraries.  **boost** uses a build process with the b2 tool.

```bash
./bootstrap.sh --prefix=$PKG/usr --libdir=$PKG/usr/lib64

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./b2 \
  --prefix=$PKG/usr \
  --libdir=$PKG/usr/lib64 \
  -j ${BLDTHREADS:-1}

./b2 install
```

#### thrift

Source: <https://thrift.apache.org/>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/thrift>  
Build time: 2 min 30 sec

This package provides the the Apache Thrift library for point-to-point RPC communications.  **thrift** needs its `bootstrap.sh` run first, and then uses a typical *Automake* build process.

```bash
./bootstrap.sh

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --docdir=/usr/doc/$PRGNAM-$VERSION \
  --with-boost=yes --without-java --without-nodejs --without-nodets --without-lua --without-php \
 --without-dart  --without-ruby --without-swift --without-rs --without-cl --without-haxe --without-netstd --without-d \
  --disable-static \
  --build=$ARCH-slackware-linux 

make -j ${BLDTHREADS:-1}
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### rabbitmq-c

Source: <http://github.com/alanxz/rabbitmq-c>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/rabbitmq-c>  
Build time: 5 sec

This package provides a C AMQP library for use with RabbitMQ.  **rabbitmq-c** uses a typical *cmake* build process.

```bash
mkdir -p build
cd build
cmake \
  -DCMAKE_C_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_CXX_FLAGS:STRING="$SLKCFLAGS" \
  -DCMAKE_INSTALL_PREFIX=/usr \
  -DLIB_SUFFIX=${LIBDIRSUFFIX} \
  -DMAN_INSTALL_DIR=/usr/man \
  -DCMAKE_PREFIX_PATH=/usr \
  -DCMAKE_SHARED_LIBRARY_PREFIX=/usr/lib64 \
  -DCMAKE_BUILD_TYPE=Release ${@:3} ..

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### librdkafka

Source: <https://github.com/confluentinc/librdkafka>  
SlackBuild: <https://github.com/scottr131/slackbuilds/tree/main/librdkafka>  
Build time: 1 min

This package provides a C implementation of the Apache Kafka Producer, Consumer, and Admin clients.  It is used by Ceph  **librdkafka** uses a typical *Automake* build process.

```bash
CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
./configure \
  --prefix=/usr \
  --libdir=/usr/lib${LIBDIRSUFFIX} \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --mandir=/usr/man \
  --build=$ARCH-slackware-linux 

CFLAGS="$SLKCFLAGS" \
CXXFLAGS="$SLKCFLAGS" \
make -j ${BLDTHREADS:-1}
make install DESTDIR=$PKG
```

#### Ceph

Source: <https://download.ceph.com/tarballs/>  
SlackBuild: <>
Build time 

This is the entire Ceph system.  It looks like it requires around 16GB of RAM to build without much swapping.  It's probably better to build on a system with 32GB of RAM.  The source code takes 1.6GB of disk and will grow during compile.

Ceph will need patched to allow the version CMake that ships with Slackware-current to build some components. 

```diff
diff -ruN ceph-orig/cmake/modules/BuildArrow.cmake ceph-cmake/cmake/modules/BuildArrow.cmake
--- ceph-orig/cmake/modules/BuildArrow.cmake    2025-09-04 15:35:40.000000000 -0400
+++ ceph-cmake/cmake/modules/BuildArrow.cmake   2025-09-16 00:37:56.621000000 -0400
@@ -4,6 +4,8 @@
   # only enable the parquet component
   set(arrow_CMAKE_ARGS -DARROW_PARQUET=ON)

+  list(APPEND arrow_CMAKE_ARGS -DCMAKE_POLICY_VERSION_MINIMUM=3.5)
+
   # only use preinstalled dependencies for arrow, don't fetch/build any
   list(APPEND arrow_CMAKE_ARGS -DARROW_DEPENDENCY_SOURCE=SYSTEM)

diff -ruN ceph-orig/cmake/modules/BuildOpentelemetry.cmake ceph-cmake/cmake/modules/BuildOpentelemetry.cmake
--- ceph-orig/cmake/modules/BuildOpentelemetry.cmake    2025-09-04 15:35:40.000000000 -0400
+++ ceph-cmake/cmake/modules/BuildOpentelemetry.cmake   2025-09-16 00:37:56.621000000 -0400
@@ -10,6 +10,7 @@
   set(opentelemetry_BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/opentelemetry-cpp")
   set(opentelemetry_cpp_targets opentelemetry_trace opentelemetry_exporter_jaeger_trace)
   set(opentelemetry_CMAKE_ARGS -DCMAKE_POSITION_INDEPENDENT_CODE=ON
+                               -DCMAKE_POLICY_VERSION_MINIMUM=3.5
                                -DWITH_JAEGER=ON
                                -DBUILD_TESTING=OFF
                                -DCMAKE_BUILD_TYPE=Release
diff -ruN ceph-orig/cmake/modules/BuildUtf8proc.cmake ceph-cmake/cmake/modules/BuildUtf8proc.cmake
--- ceph-orig/cmake/modules/BuildUtf8proc.cmake 2025-09-04 15:35:40.000000000 -0400
+++ ceph-cmake/cmake/modules/BuildUtf8proc.cmake        2025-09-16 00:37:56.621000000 -0400
@@ -4,6 +4,8 @@
   # only build static version
   list(APPEND utf8proc_CMAKE_ARGS -DBUILD_SHARED_LIBS=OFF)

+  list(APPEND utf8proc_CMAKE_ARGS -DCMAKE_POLICY_VERSION_MINIMUM=3.5)
+
   # cmake doesn't properly handle arguments containing ";", such as
   # CMAKE_PREFIX_PATH, for which reason we'll have to use some other separator.
   string(REPLACE ";" "!" CMAKE_PREFIX_PATH_ALT_SEP "${CMAKE_PREFIX_PATH}")
diff -ruN ceph-orig/cmake/modules/BuildZstd.cmake ceph-cmake/cmake/modules/BuildZstd.cmake
--- ceph-orig/cmake/modules/BuildZstd.cmake     2025-09-04 15:35:40.000000000 -0400
+++ ceph-cmake/cmake/modules/BuildZstd.cmake    2025-09-16 00:37:56.621000000 -0400
@@ -10,6 +10,7 @@
                -DCMAKE_C_FLAGS=${ZSTD_C_FLAGS}
                -DCMAKE_AR=${CMAKE_AR}
                -DCMAKE_POSITION_INDEPENDENT_CODE=${ENABLE_SHARED}
+               -DCMAKE_POLICY_VERSION_MINIMUM=3.5
     BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/libzstd
     BUILD_COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR> --target libzstd_static
     BUILD_BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/libzstd/lib/libzstd.a"
diff -ruN ceph-orig/src/arrow/cpp/cmake_modules/ThirdpartyToolchain.cmake ceph-cmake/src/arrow/cpp/cmake_modules/ThirdpartyToolchain.cmake
--- ceph-orig/src/arrow/cpp/cmake_modules/ThirdpartyToolchain.cmake     2024-01-16 09:38:51.000000000 -0500
+++ ceph-cmake/src/arrow/cpp/cmake_modules/ThirdpartyToolchain.cmake    2025-09-16 00:57:50.297000000 -0400
@@ -2379,7 +2379,9 @@
                       PREFIX "${CMAKE_BINARY_DIR}"
                       URL ${XSIMD_SOURCE_URL}
                       URL_HASH "SHA256=${ARROW_XSIMD_BUILD_SHA256_CHECKSUM}"
-                      CMAKE_ARGS ${XSIMD_CMAKE_ARGS})
+                      CMAKE_ARGS
+                               -DCMAKE_POLICY_VERSION_MINIMUM=3.5
+                               ${XSIMD_CMAKE_ARGS})

   set(XSIMD_INCLUDE_DIR "${XSIMD_PREFIX}/include")
   # The include directory must exist before it is referenced by a target.
diff -ruN ceph-orig/src/rocksdb/db/blob/blob_file_meta.h ceph-cmake/src/rocksdb/db/blob/blob_file_meta.h
--- ceph-orig/src/rocksdb/db/blob/blob_file_meta.h      2023-05-24 15:55:23.000000000 -0400
+++ ceph-cmake/src/rocksdb/db/blob/blob_file_meta.h     2025-09-16 00:37:56.621000000 -0400
@@ -10,6 +10,7 @@
 #include <memory>
 #include <string>
 #include <unordered_set>
+#include <cstdint>

 #include "rocksdb/rocksdb_namespace.h"

diff -ruN ceph-orig/src/rocksdb/include/rocksdb/trace_record.h ceph-cmake/src/rocksdb/include/rocksdb/trace_record.h
--- ceph-orig/src/rocksdb/include/rocksdb/trace_record.h        2023-05-24 15:55:23.000000000 -0400
+++ ceph-cmake/src/rocksdb/include/rocksdb/trace_record.h       2025-09-16 00:37:56.621000000 -0400
@@ -8,6 +8,7 @@
 #include <memory>
 #include <string>
 #include <vector>
+#include <cstdint>

 #include "rocksdb/rocksdb_namespace.h"
 #include "rocksdb/slice.h"
```

The Ceph build process will need some arguments so the "Release" build is compiled instead of a much larger debug build.  Also RTTI is enabled and directory locations are configured for Slackware. 

```bash
ARGS="-DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_PREFIX_PATH=/usr -DWITH_PYTHON3=3.12 -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS=\"-frtti\" " ./do_cmake.sh 
cd build
ninja
```
