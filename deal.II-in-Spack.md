Table of Contents
=================

   * [Using deal.II on Mac OS X and Linux via Spack](#using-dealii-on-mac-os-x-and-linux-via-spack)
      * [Quick installation on the desktop](#quick-installation-on-the-desktop)
      * [Installation example on a Centos7 cluster (Emmy cluster of RRZE, Erlangen, Germany)](#installation-example-on-a-centos7-cluster-emmy-cluster-of-rrze-erlangen-germany)
      * [Enabling CUDA](#enabling-cuda)
      * [Environment Modules](#environment-modules)
      * [System provided packages](#system-provided-packages)
      * [Installing GCC](#installing-gcc)
      * [Using macOS](#using-macos)
      * [Best practices using Spack:](#best-practices-using-spack)
         * [Info:](#info)
         * [Extra options:](#extra-options)
         * [Compiler flags](#compiler-flags)
         * [Different versions coexisting:](#different-versions-coexisting)
         * [Filesystem Views:](#filesystem-views)
         * [Check before build:](#check-before-build)
         * [Develop using Spack](#develop-using-spack)
         * [Keep the stage to run unit tests](#keep-the-stage-to-run-unit-tests)
         * [Licensed software](#licensed-software)
         * [Freeze package versions](#freeze-package-versions)
      * [Known issues using Spack:](#known-issues-using-spack)

# Using deal.II on Mac OS X and Linux via Spack

The deal.II suite is also available on Spack (https://github.com/spack/spack) -- a flexible package manager developed with High-Performance-Computing in mind. It is intended to let you build for many combinations of compiler, architectures, dependency libraries, and build configurations, all with a friendly, intuitive user interface.

For a quick overview of Spack's features, we recommend this short presentation https://tgamblin.github.io/files/Gamblin-Spack-SC15-Talk.pdf
or the following videos from lead developers:
[Massimiliano Culpo - SPACK: A Package Manager for Supercomputers, Linux and MacOS](https://www.youtube.com/watch?v=Qok-nXfIWfg) and 
[Todd Gamblin (LLNL) - Managing HPC Software Complexity with Spack](https://www.youtube.com/watch?v=Ie1cZTR09kk).

[Spack tutorial 101](http://spack.readthedocs.io/en/latest/tutorial.html) is good place to start as well.

Note: Spack is in active development and at the time of writing this page the following installation procedure was tested on Ubuntu, macOS and on two HPC clusters (one of which being a Cray supercomputer).

## Quick installation on the desktop

Add the following to `~/.bashrc` (or equivalent)
```
export SPACK_ROOT=/path/to/spack
export PATH="$SPACK_ROOT/bin:$PATH"
```
`SPACK_ROOT` is the destination where you want Spack to be installed (i.e. `$HOME/spack`).

Make sure that `python`, `curl`, `git`, `make`, `g++` and `gfortran` are installed, i.e. by running:
```
sudo apt install python git curl make g++ gfortran
```

Now clone Spack and checkout the latest release (stable releases are given [tags](https://github.com/spack/spack/tags); the development branch is called `develop`)
```
cd $SPACK_ROOT
git clone https://github.com/spack/spack.git .
git checkout develop
```

**Make sure C/C++/Fortran compilers are in path** (on Ubuntu you need to `sudo apt-get install gfortran`, on macOS you can compile `gcc` with spack, see [below](#installing-gcc), and you have **curl** (`sudo apt-get install curl`) to download packages. Then install the complete deal.II suite
```
spack install --test=root dealii ^boost@1.72.0 ^mumps@5.2.0 ^trilinos~exodus~netcdf
```
Additional option `--test=root` instructs Spack to run quick tests for `dealii`.

**DONE**! No extra (preliminary) configuration steps are needed on most Linux distributions. For configuration files, have a look at [xsdk project](https://github.com/xsdk-project/installxSDK/tree/master/platformFiles), [ceed project](https://ceed.exascaleproject.org/ceed-2.0/) or [spack-configs repository](https://github.com/spack/spack-configs).

**IMPORTANT:** If you compile deal.II on a cluster, see the next section on how to use externally provided MPI implementation instead.

Before configuring your project you need to set `DEAL_II_DIR` by 
```
export DEAL_II_DIR=$(spack location -i dealii)
```
You may jump ahead and read [best practices using spack](#best-practices-using-spack). Also a good starting point is [Getting Started Guide](http://spack.readthedocs.io/en/latest/getting_started.html).


## Installation example on a Centos7 cluster (Emmy cluster of RRZE, Erlangen, Germany)
In order to use Spack on a cluster, there are two options: (1) you are a sysadmin and you know hardware details and can use Spack to properly configure and build MPI providers (e.g. `openmpi`); (2) you are a user and you need to make Spack work with MPI provided on your cluster. In what follows we consider the latter case. 

Below is a brief step-by-step instruction to install deal.II on [Emmy cluster](https://www.rrze.fau.de/dienste/arbeiten-rechnen/hpc/systeme/emmy-cluster.shtml#access) of RRZE, Erlangen, Germany. For configuration examples at other HPC clusters see [spack-configs](https://github.com/spack/spack-configs) repository.

(1) Download spack
```
mkdir $WOODYHOME/spack
cd $WOODYHOME/spack
module load git
```
(1a) If you plan on using `GCC/Clang` as a compiler:
```
# Clone standard version of Spack
git clone https://github.com/spack/spack.git $WOODYHOME/spack
export PATH=$WOODYHOME/spack/bin:$PATH
```
(1b) If you plan on using `Intel` compilers (this point is expanded upon in step (5)):
```
# Clone modified version of Spack to fit RRZE setup
git clone -b pkg/intel_mpi_fix https://github.com/davydden/spack.git .
export PATH=$WOODYHOME/spack/bin:$PATH
```
(2) Load the `openmpi` module and let Spack find `GCC` compiler which is also loaded as a dependency. You might get a warning that spack has not detected any new compilers - this only means that the `compilers.yaml` file already has the correct version of `GCC` listed in it.
```
module load openmpi/2.0.2-gcc
spack compiler find
```
(3) Add `openmpi` as an external package, along with `python` and a few other self explanatory setting for `deal.ii`. That is done by adding the following to `~/.spack/linux/packages.yaml`:
```
packages:
  all:
    compiler: [gcc]
    providers:
      mpi: [openmpi]
      blas: [openblas]
      lapack: [openblas]
  openmpi:
    version: [2.0.2]
    paths:
      openmpi@2.0.2%gcc@4.8.5: /apps/OpenMPI/2.0.2-gcc/
    buildable: False
  suite-sparse:
    version: [5.1.0]
  dealii:
    variants: +optflags~python
```
Those paths are the location where external packages can be found (i.e. `<prefix>` instead of `<prefix>/bin` or `<prefix>/lib`). `providers` section essentially tells Spack which packages to use to satisfy virtual dependencies such as `MPI`, `BLAS`, `LAPACK`, `ScaLAPACK`, etc. Here we also limit version of `suite-sparse` to `5.1.0` as we build with `gcc@4.8.5` whereas `5.2.0` requires at least `gcc@4.9`. 

(4) Now install deal.II:  `spack install dealii`.

(5) Here is an alternative setup with Intel compilers. Run `module load intel64/18.0up02` followed by `spack compiler find`. You should get the following new entry in `~/.spack/linux/compilers.yaml`:
```
- compiler:
    environment: {}
    extra_rpaths: []
    flags: {}
    modules: []
    operating_system: centos7
    paths:
      cc: /apps/intel/ComposerXE2018/compilers_and_libraries_2018.2.199/linux/bin/intel64/icc
      cxx: /apps/intel/ComposerXE2018/compilers_and_libraries_2018.2.199/linux/bin/intel64/icpc
      f77: /apps/intel/ComposerXE2018/compilers_and_libraries_2018.2.199/linux/bin/intel64/ifort
      fc: /apps/intel/ComposerXE2018/compilers_and_libraries_2018.2.199/linux/bin/intel64/ifort
    spec: intel@18.0.2
    target: x86_64
```
Edit it to set environment variable (with license) and extra rpaths:
```
    environment:
      set:
        INTEL_LICENSE_FILE: 1713@license4
    extra_rpaths: # take from echo $LIBRARY_PATH
     - /apps/intel/ComposerXE2018/compilers_and_libraries_2018.2.199/linux/ipp/lib/intel64
     - /apps/intel/ComposerXE2018/compilers_and_libraries_2018.2.199/linux/compiler/lib/intel64_lin
```
Then add to `~/.spack/linux/packages.yaml` paths to `intel-mpi`, `intel-mkl` as well as `cmake` which currently does not build with `Intel`:
```
packages:
  all:
    compiler: [intel]
    providers:
      mpi: [intel-mpi]
      blas: [intel-mkl]
      lapack: [intel-mkl]
      scalapack: [intel-mkl]
  intel-mpi:
    version: [2018.2.199]
    paths:
      intel-mpi@2018.2.199%intel@18.0.2: /apps/intel/ComposerXE2018/
    buildable: False
  intel-mkl:
    version: [2018.2.199]
    paths:
      intel-mkl@2018.2.199%intel@18.0.2: /apps/intel/ComposerXE2018/
    buildable: False
  cmake:
    version: [3.6.0]
    paths:
      cmake@3.6.0%intel@18.0.2: /apps/cmake/3.6.0/
    buildable: False
  suite-sparse:
    version: [5.1.0]
  dealii:
    variants: +optflags~python
```
Finally install `deal.II` 
```
spack install -j 20 dealii%intel~assimp~petsc~slepc+mpi^intel-mpi^intel-mkl
```
Note that `%intel` specified the compiler whereas `^intel-mpi` and `^intel-mkl` specified which implementation of MPI and BLAS/LAPACK we want to use. Here we also disabled few other packages that have issues building with `Intel` compilers.

See [this discussion](https://groups.google.com/d/msg/spack/NxyNTAZyMQg/Klu2CHR8GQAJ) on more info about using Intel compilers in Spack.

Note that in order to install multiple flavors of `intel-mkl` via Spack (i.e. with and without threading), you may have to delete `$HOME/intel/.pset`, see [this comment](https://github.com/spack/spack/issues/9713#issuecomment-436081356).


## Enabling CUDA
You can build the current development version of `dealii` with CUDA. A possible configuration of `packages.yaml` is
```
packages:
 dealii:
  version: ['develop']
  variants: +cuda cuda_arch=52
 cuda:
  version: ['8.0.61']
```
where we specified explicitly CUDA architecture as well as version of `cuda` to be used.

## Environment Modules
Spack provides some integration with Environment Modules and Dotkit to make it easier to use the packages it installs. For a full description, read http://spack.readthedocs.io/en/latest/getting_started.html#environment-modules

To add the support for Environment Modules run
```
spack install environment-modules
```
and then add to `~/.bashrc` (or equivalent)
```
MODULES_HOME=$(spack location -i environment-modules)
source ${MODULES_HOME}/init/bash
. $SPACK_ROOT/share/spack/setup-env.sh
```

If you install `deal.II` before setting up environment modules,
the module files have to be regenerated
```
spack module tcl refresh
```

Then run
```
spack load dealii
spack load cmake
```
Now `DEAL_II_DIR` environment variable should be set appropriately and `cmake` executable will be available in path. Keep in mind that `spack load dealii` will also set `LD_LIBRARY_PATH` accordingly, this may or may not be what you need. An alternative is to use `export DEAL_II_DIR=$(spack location -i dealii)`.

## System provided packages
Spack is flexible to use both self-compiled and system provided packages. 
In most cases this is desirable for `MPI`, which is already installed on computational clusters. To configure external packages you need to edit `~/.spack/linux/packages.yaml`. For `openmpi` this could be
```
packages:
  openmpi:
    version: [1.8.8]
    paths:
      openmpi@1.8.8%gcc@6.2.0: /opt/openmpi-1.8.8
    buildable: False
```
In order to make sure that we use to build packages `1.8.8` version of `openmpi` and not the most recent one (i.e. `2.0.2`), we specified conretization preferences with `version: [1.8.8]`.

One can also specify which packages should be used to provide `mpi`, `blas/lapack` , preferred compilers and preferred variants:
```
packages:
  all:
    compiler: [gcc,clang]
    providers:
      mpi: [openmpi]
      blas: [openblas]
      lapack: [openblas]
    boost:
      variants: +python
```

For more elaborated discussion, see [Configuration Files in Spack](http://spack.readthedocs.io/en/latest/configuration.html).

## Installing GCC
If your system does not provide any Fortran compiler or you want to have the most recent `gcc`,
you can install it by
```
spack install gcc
```

Assuming that you configured [Environment Modules](#environment-modules), load `gcc` and let Spack find the newly installed compilers:
```
spack load gcc
spack compiler find
```
Now you can install deal.II with `gcc`
```
spack install dealii%gcc
```

## Using macOS

If you are on the mac, read the following instructions on [Mixed Toolchains](http://spack.readthedocs.io/en/latest/getting_started.html#mixed-toolchains).

Note that XCode 10 does **NOT** install command line tools (`$xcode-select --install`) in `/usr/include` (see [this discussion](https://forums.developer.apple.com/thread/104296)). Therefore prior to installing `gcc` package via Spack, please install `/Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg`.

Once you have `gcc` installed and edit `~/.spack/compilers.yaml` to use it with Apple's clang, you will be able to install deal.II
```
spack install dealii%clang
```

## Best practices using Spack:

Spack is complicated and flexible package manager primarily aimed at High-Performance-Computing.
Below are some examples of using Spack to build and develop deal.II:

### Info:
If you are not sure which options a given package has, simply run
```
spack info <package>
```
The output will contain available versions, variants, dependencies and description:
```
$ spack info dealii
CMakePackage:   dealii

Description:
    C++ software library providing well-documented tools to build finite
    element codes for a broad variety of PDEs.

Homepage: https://www.dealii.org

Preferred version:  
    8.5.1      https://github.com/dealii/dealii/releases/download/v8.5.1/dealii-8.5.1.tar.gz

Safe versions:  
    develop    [git] https://github.com/dealii/dealii.git
    8.5.1      https://github.com/dealii/dealii/releases/download/v8.5.1/dealii-8.5.1.tar.gz
    8.5.0      https://github.com/dealii/dealii/releases/download/v8.5.0/dealii-8.5.0.tar.gz
    8.4.2      https://github.com/dealii/dealii/releases/download/v8.4.2/dealii-8.4.2.tar.gz
    8.4.1      https://github.com/dealii/dealii/releases/download/v8.4.1/dealii-8.4.1.tar.gz
    8.4.0      https://github.com/dealii/dealii/releases/download/v8.4.0/dealii-8.4.0.tar.gz
    8.3.0      https://github.com/dealii/dealii/releases/download/v8.3.0/dealii-8.3.0.tar.gz
    8.2.1      https://github.com/dealii/dealii/releases/download/v8.2.1/dealii-8.2.1.tar.gz
    8.1.0      https://github.com/dealii/dealii/releases/download/v8.1.0/dealii-8.1.0.tar.gz

Variants:
    Name [Default]               Allowed values          Description


    adol-c [off]                 True, False             Compile with Adol-c
    arpack [on]                  True, False             Compile with Arpack and
                                                         PArpack (only with MPI)
    build_type [DebugRelease]    Debug, Release,         The build type to build
                                 DebugRelease            
    doc [off]                    True, False             Compile with documentation
    gsl [on]                     True, False             Compile with GSL
    hdf5 [on]                    True, False             Compile with HDF5 (only with
                                                         MPI)
    int64 [off]                  True, False             Compile with 64 bit indices
                                                         support
    metis [on]                   True, False             Compile with Metis
    mpi [on]                     True, False             Compile with MPI
    nanoflann [off]              True, False             Compile with Nanoflann
    netcdf [on]                  True, False             Compile with Netcdf (only with
                                                         MPI)
    oce [on]                     True, False             Compile with OCE
    optflags [off]               True, False             Compile using additional
                                                         optimization flags
    p4est [on]                   True, False             Compile with P4est (only with
                                                         MPI)
    petsc [on]                   True, False             Compile with Petsc (only with
                                                         MPI)
    python [on]                  True, False             Compile with Python bindings
    slepc [on]                   True, False             Compile with Slepc (only with
                                                         Petsc and MPI)
    sundials [off]               True, False             Compile with Sundials
    trilinos [on]                True, False             Compile with Trilinos (only
                                                         with MPI)

Installation Phases:
    cmake    build    install

Build Dependencies:
    adol-c     cmake     hdf5     muparser    oce     slepc         trilinos
    arpack-ng  doxygen   lapack   nanoflann   p4set   suite-sparse  zlib
    blas       graphivz  metis    netcdf      petsc   sundials      
    boost      gsl       mpi      netcdf-cxx  python  tbb

Link Dependencies:
    adol-c     doxygen   lapack   nanoflann   p4est   suite-sparse  zlib
    arpack-ng  graphviz  metis    netcdf      petsc   sundials      
    blas       graphviz  mpi      netcdf-cxx  python  tbb
    boost      hdf5      muparser oce         slepc   trilinos

Run Dependencies:
    None

Virtual Packages: 
    None
```

A lot of `spack` commands have help, for example
```
$ spack install -h
usage: spack install [-h] [--only {package,dependencies}] [-j JOBS]
                     [--keep-prefix] [--keep-stage] [--restage] [-n] [-v]
                     [--fake] [-f] [--clean | --dirty] [--run-tests]
                     [--log-format {junit}] [--log-file LOG_FILE]
                     ...

build and install packages

positional arguments:
  package               spec of the package to install

optional arguments:
  -h, --help            show this help message and exit
  --only {package,dependencies}
                        select the mode of installation. the default is to
                        install the package along with all its dependencies.
                        alternatively one can decide to install only the
                        package or only the dependencies
  -j JOBS, --jobs JOBS  explicitly set number of make jobs. default is #cpus
  --keep-prefix         don't remove the install prefix if installation fails
  --keep-stage          don't remove the build stage if installation succeeds
  --restage             if a partial install is detected, delete prior state
  -n, --no-checksum     do not check packages against checksum
  -v, --verbose         display verbose build output while installing
  --fake                fake install. just remove prefix and create a fake
                        file
  -f, --file            install from file. Read specs to install from .yaml
                        files
  --clean               clean environment before installing package
  --dirty               do NOT clean environment before installing
  --run-tests           run package level tests during installation
  --log-format {junit}  format to be used for log files
  --log-file LOG_FILE   filename for the log file. if not passed a default
                        will be used
```

### Extra options:
One can specify extra options for packages in the deal.II suite. For example if you want to have boost with `python` module, this can be done by
```
spack install dealii@develop+mpi~python ^boost+python
```

If you want to specify blas/lapack/mpi implementations, this can be done similarly
```
spack install dealii@develop+mpi ^mpich
```
To check which packages implement `mpi` run
```
spack providers mpi
```
One can also specify which Blas/Lapack implementation to use. For example to build deal.II suite with `atlas` run
```
spack install dealii ^atlas
```

### Compiler flags
You can specify compiler flags on the command line as
```
spack install dealii cppflags="-march=native -O3"
```
Note that these flags will be inherited by dependencies such as `petsc`, `trilinos`, etc. Same can be done by declaring these flags in `~/.spack/compilers.yaml`:
```
compilers:
- compiler:
    modules: []
    operating_system: centos6
    paths:
      cc: /usr/bin/gcc
      cxx: /usr/bin/g++
      f77: /usr/bin/gfortran
      fc: /usr/bin/gfortran
    flags:
      cflags: -O3 -fPIC
      cxxflags: -O3 -fPIC
      cppflags: -O3 -fPIC
    spec: gcc@4.7.2
```

See this [google forum topic](https://groups.google.com/forum/?fromgroups#!topic/dealii/3Yjy8CBIrgU) for discussion on which flags to use. You can also use `spack install dealii+optflags` to enable extra optimization flags in release build.

### Different versions coexisting:
One can easily have slightly different versions of deal.II side-by-side, e.g. to compile development version of deal.II with complex-valued PETSc and `gcc` compiler run
```
spack install dealii@develop+mpi+petsc~int64%gcc ^petsc+complex~hypre
```
The good thing is that if you already have deal.ii built with real-valued petsc, then only `petsc`, `slepc` and `deal.ii` itself will be rebuild. Everything else (`trilinos`, `mumps`, `metis`, etc) will be reused.

One can use `environment-modules` (see above) to automatically set `DEAL_II_DIR` to the complex version: 
```
spack load dealii@develop+mpi+petsc~int64%gcc ^petsc+complex~hypre
```

### Filesystem Views:
If you prefer to have the whole dealii suite (and possible something else) symlinked into a single path (like `/usr/local`), one can use [Filesystem Views](http://spack.readthedocs.io/en/latest/workflows.html#filesystem-views):
```
spack view -v symlink dealii_suite dealii@develop
```
You can also add to this view other packages, i.e.
```
spack view -v symlink dealii_suite the-silver-searcher
```


### Check before build:
It is often convenient to check which version of packages, compilers, variants etc will be used before actually starting installation. That can be done by examining the concretized spec via `spack spec` command, e.g. 
```
spack spec dealii@develop+mpi+petsc~int64%gcc ^petsc+complex~hypre
```

The `spec` command has a useful flag `-I` which will show install status of dependencies in the graph.

### Develop using Spack
Probably the easiest way to use Spack while contributing patches to the deal.II is the following.
 
1. Install `deal.II` via Spack as usual `spack install <dealii-spec>` (`<dealii-spec>` can be `dealii@develop+mpi^openmpi^openblas` or alike)
2. Clone the deal.II sources and `cd` to a build folder (we assume it's in the root of source location)
3. Get `cmake` command used by Spack to build `deal.II` from `<dealii-prefix>/.spack/build.out` and save it in `build_command.sh` via

```
cat $(spack location -i <dealii-spec>)/.spack/build.out | grep "==> 'cmake'" | sed -e "s/[^ ]*[^ ]/'..\/'/3" | cut -d " " -f2- > build_command.sh
```

You might want to remove the `DCMAKE_INSTALL_PREFIX` as you probably don't want to accidentally override the version installed by Spack. 

4. Setup build environment to be exactly the same as used by Spack

```
spack build-env dealii@develop+mpi^openmpi^openblas bash
```

5. Configure and build 
```
source build_command.sh
make all
```

#### Installation of dependencies alone
It is possible to get Spack to install only the dependencies, and leave the configuration and installation of deal.II entirely up to you. 
The manual configuration process includes having to point CMake to the Spack installed libraries, which could possibly be assisted using [Filesystem view](#filesystem-views)). 
To install only the dependencies of deal.II (specifically, the developer version), issue the following command:
```
spack install --only dependencies dealii@develop ^cmake@3.9.4
```

#### Filesystem view
An alternative is to create a [Filesystem view](#filesystem-views) for an already installed library and then compile patched version of deal.II manually by providing path to the view for each dependency. 

### Keep the stage to run unit tests
By default, the build folder (aka `stage`) for each package is created in a temporary system dependent location.
This gets purge by system on next restart.
If you want to keep the stage after the installation, you need to do two things:
First, make spack use a custom location for stage by adding the following to your `~/.spack/config.yaml`:
```
config:
  build_stage:
    - $spack/var/spack/stage
```
Second, prescribe an extra argument to `install` command to keep the stage after successful installation:
```
spack install --keep-stage dealii@develop
```

### Licensed software
Spack supports installation of [licensed software](http://spack.readthedocs.io/en/latest/packaging_guide.html#license) as well as usage of [Licensed compilers](http://spack.readthedocs.io/en/latest/getting_started.html#licensed-compilers). In order to configure Intel compilers see [this page](http://spack.readthedocs.io/en/latest/getting_started.html#vendor-specific-compiler-configuration).


### Freeze package versions
Currently Spack does not try to re-use already installed packages. On another hand, by default
the most recent version of a package will be installed. When updating deal.II build (for example to use the new version of `trilinos`), the combination of the two factors may lead to recompilation of many other packages used in the deal.II suite when one of the main build dependency like `cmake` has a new version.

To circumvent the problem, the user can specify preferred versions of packages in `~/.spack/packages.yaml`:
```
packages:
  cmake:
    version: [3.6.1]
  curl:
    version: [7.50.3]
  openssl:
    version: [1.0.2j]
  python:
    version: [2.7.12]
```
This settings will be taken into account during conretization process and thus will help to avoid rebuilding most of the deal.II suite when, for example, `openssl` is updated to the new version.

## Known issues using Spack:

### Incompatible CMake versions

At the moment there is an incompatibility between deal.II and `cmake` with version 3.10 and greater.
To install deal.II using an earlier version of `cmake`, either use the `packages.yaml` file to freeze `cmake` to an earlier version, or specify the version at install time, e.g.
```
spack install dealii^cmake@3.9.4
```