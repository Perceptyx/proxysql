Moving to cmake
===============

*WARNING!* This is a beta version, it is not fully finished.

While most of the external libraries are include, in the versions that were in the upstram repos,
here we use ExternalProject from cmake to download the proper version locally.

Most of the patches in single files were merged into a single patch using `diff -unR`, and
now they are included in the `patches` directory. Some patches where used in the mariadbclient
library that can be made using cmake build flags, so they were changed for that.

Many libraries may be easier to find locally using cmake's `find_package`, mostly the ones that
don't have patches. That way it would be easier for packagers to include their systems libraries.
Of course, the minimal version should be specified where needed.

To generate a new build, use this sequence:

```bash
# here we just added a submodule pointing to json library from upstream
git clone git@github.com:Perceptyx/proxysql.git
git checkout switch-build-cmake
# get external deps
git submodule update --init --recursive
# add your options here
cmake -H. -B builds -DWITH_CLICKHOUSE=OFF
cd builds
cmake --build .
# builds directory will have `proxysql` command
# you can also generate packages automagically
cpack -G RPM builds
cpack -G DEB builds
```
I would like your comments about the build process, or what libraries can be used from the system, etc.

