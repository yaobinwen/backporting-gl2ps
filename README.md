# Backporting `gl2ps` from Ubuntu Impish to Ubuntu Bionic

## Overview

This repository discusses the issue I ran into when backporting [`gl2ps`](https://geuz.org/gl2ps/) from Ubuntu Impish to Ubuntu Bionic by looking deep into two packages:

- [Debian's `debhelper`](https://salsa.debian.org/debian/debhelper)
- [CMake](https://cmake.org/cmake/help/v3.23/)

I looked into the source code of these two packages in order to understand the causes of the issue.

## What the Issue is

`debuild -us -uc` will report errors of `Cannot find (any matches for) "usr/lib/*/lib*.so" (tried in ., debian/tmp)`. See the section "Reproduce the Issue" for more details.

## Reproduce the Issue

You need to meet the following prerequisites:
- [Vagrant 2.2.14+](https://www.vagrantup.com/)
- [VirtualBox 6.1.32+](https://www.virtualbox.org/)
- Network connection.

Follow the steps below to reproduce the issue:
- 1). Run `cd impish`.
- 2). Run `vagrant up` to create the virtual machine for Ubuntu Impish. The `Vagrantfile` has built-in script to install the needed packages to reproduce the issue.
  - 2.1). Run `vagrant ssh` to log into the VM.
  - 2.2). Run `cd /vagrant/official/gl2ps`.
  - 2.3). Run `debuild -us -uc && echo "Succeed!"` and it should print "Succeed!" in the end. `impish/official/debuild.txt` is a build log for reference.

- 3). Run `cd bionic`.
- 4). Run `vagrant up` to create the virtual machine for Ubuntu Bionic. The `Vagrantfile` has built-in script to install the needed packages to reproduce the issue.
  - 4.1). Run `vagrant ssh` to log into the VM.
  - 4.2). Run `cd /vagrant/official/gl2ps`.
  - 4.3). Run `debuild -us -uc && echo "Succeed!"` and it should print "Succeed!" in the end. `bionic/official/debuild.txt` is a build log for reference.
  - 4.4). Run `cd /vagrant/backported/gl2ps`.
  - 4.5). Run `debuild -us -uc || echo "Failed!"` and it should print "Failed" in the end. `bionic/backported/debuild.txt` is a build log for reference. The error messages are pasted as follows:

```
Install the project...
/usr/bin/cmake -P cmake_install.cmake
-- Install configuration: "None"
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/lib/libgl2ps.a
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/lib/libgl2ps.so.1.4.2
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/lib/libgl2ps.so.1.4
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/lib/libgl2ps.so
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/include/gl2ps.h
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/share/doc/gl2ps/README.txt
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/share/doc/gl2ps/COPYING.LGPL
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/share/doc/gl2ps/COPYING.GL2PS
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/share/doc/gl2ps/gl2psTest.c
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/share/doc/gl2ps/gl2psTestSimple.c
-- Installing: /vagrant/backported/gl2ps/debian/tmp/usr/share/doc/gl2ps/gl2ps.pdf
make[1]: Leaving directory '/vagrant/backported/gl2ps/obj-x86_64-linux-gnu'
   dh_install -O--buildsystem=cmake
dh_install: The use of "debhelper-compat (= 11)" is experimental and may change (or be retired) without notice
dh_install: Cannot find (any matches for) "usr/lib/*/lib*.so" (tried in ., debian/tmp)

dh_install: libgl2ps-dev missing files: usr/lib/*/lib*.so
dh_install: Cannot find (any matches for) "usr/lib/*/lib*.so.*" (tried in ., debian/tmp)

dh_install: libgl2ps1.4 missing files: usr/lib/*/lib*.so.*
dh_install: missing files, aborting
debian/rules:8: recipe for target 'binary' failed
make: *** [binary] Error 25
dpkg-buildpackage: error: debian/rules binary subprocess returned exit status 2
debuild: fatal error at line 1152:
dpkg-buildpackage -rfakeroot -us -uc -ui failed
```

The built files were all under the path `usr/lib/lib*.so` which doesn't match the pattern `usr/lib/*/lib*.so`, hence the error.

## Understand the Issue

The cause of the build errors about `usr/lib/*/` has something to do with the `debian/patches`. To be more exact, it is about where the CMake statement `INCLUDE(GNUInstallDirs)` in inserted into `CMakeLists.txt`:

- 1). I looked at the official packaging of `gl2ps` on Bionic and noticed that after the `debian/patches` were applied, the `INCLUDE(GNUInstallDirs)` statement was inserted **after** `project(gl2ps C)`, then this official packaging could build without problems:

```cmake
project(gl2ps C)
...
INCLUDE(GNUInstallDirs)
```

- 2). But with the backported version, after the `debian/patches` were applied, the `INCLUDE(GNUInstallDirs)` was inserted **before** `project(gl2ps C)`:

```cmake
INCLUDE(GNUInstallDirs)
...
project(gl2ps C)
```

- 3). It turned out to be that:
  - a). The path of the library file installation is determined by `CMAKE_INSTALL_LIBDIR` which by default is `lib` (so together with `CMAKE_INSTALL_PREFIX` being `/usr` the whole default path is `/usr/lib`).
  - b). The CMake module `GNUInstallDirs` (`/usr/share/cmake-3.10/Modules/GNUInstallDirs.cmake`) determines the actual value of `CMAKE_INSTALL_LIBDIR` during CMake configuration. Its logic is generally as follows:

```cmake
set(_LIBDIR_DEFAULT "lib")
if(CMAKE_SYSTEM_NAME MATCHES "^(Linux|kFreeBSD|GNU)$"
      AND NOT CMAKE_CROSSCOMPILING)
  # CMake code that forms the `x86-64_linux-gnu` part.
endif()
```

  - c). In other words, if `CMAKE_SYSTEM_NAME` is not `Linux/FreeBSD/GNU`, `CMAKE_INSTALL_LIBDIR` will not be `lib/x86-64_linux-gnu`.
- 4). And it turned out to be that the statement `project(gl2ps C)` actually defined `CMAKE_SYSTEM_NAME` (based on my black-box testing, not reading its code): Before running `project(gl2ps C)`, `CMAKE_SYSTEM_NAME` was empty (or undefined); after running `project(gl2ps C)`, `CMAKE_SYSTEM_NAME` has the value "Linux".
  - a). As a result, the official packaging on Bionic could work because `INCLUDE(GNUIntallDirs)` was run after project(), so `CMAKE_INSTALL_LIBDIR` became lib/x86-64_linux-gnu , and the usr/lib/*/ pattern could work with it.
  - b). But the backported version ran `INCLUDE(GNUInstallDirs)` before `project(gl2ps C)` got the chance to define `CMAKE_SYSTEM_NAME`, so `CMAKE_INSTALL_LIBDIR` ended up being only `lib`, so the built library files were placed under the path `usr/lib/lib*.so`, so the pattern `usr/lib/*/lib*.so` wouldn't work.
- 5). The official packaging on Impish is a different story: `INCLUDE(GNUInstallDirs)` is still inserted before `project(gl2ps C)` and that's why the backported version also does so because the backported version uses the Debian packaging files on Impish. However, on Impish, the build scripts of `debhelper` change:
  - a). When we run `debuild` with `cmake` being the build system, eventually, the `Debhelper/Buildsystem/cmake.pm` Perl script of `debhelper` is called.
  - b). On Impish, the CMake configuration option `-DCMAKE_INSTALL_LIBDIR=lib/x86-64_linux-gnu` is added unconditionally. See the code snippet below. As a result, although `INCLUDE(GNUInstallDirs)` still says `CMAKE_INSTALL_LIBDIR` is just `lib`, this CMake CLI option overrides the value and makes it `lib/x86-64_linux-gnu`. This can be confirmed by the content of `impish/official/debuild.txt`.

```perl
sub configure {
        if (is_cross_compiling()) {
           ...
        }
        push(@flags, "-DCMAKE_INSTALL_LIBDIR=lib/" . dpkg_architecture_value("DEB_HOST_MULTIARCH"));
}
```

  - c). On Bionic, the CMake configuration option `-DCMAKE_INSTALL_LIBDIR=lib/x86-64_linux-gnu` is only added when it is cross-compiling. See the code snippet below. But we were not cross-compiling in our case, so there was no forced `-DCMAKE_INSTALL_LIBDIR` on the CMake command line, so `CMAKE_INSTALL_LIBDIR` ended up whatever value `INCLUDE(GNUInstallDirs)` determined.

```perl
sub configure {
        if (is_cross_compiling()) {
                ...
                push(@flags, "-DCMAKE_INSTALL_LIBDIR=lib/" . dpkg_architecture_value("DEB_HOST_MULTIARCH"));
        }
}
```
