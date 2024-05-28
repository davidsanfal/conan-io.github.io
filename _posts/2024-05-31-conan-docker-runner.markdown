---
layout: post
comments: false
title: "Introducing the Conan Docker runners: Running Conan remotelly"
meta_title: "How to run Conan inside a Docker container"
description: "Runing Conan inside a Docker container"
---

Runners provide a seamless method to execute Conan on remote build environments like Docker ones, directly from your local setup by simply configuring your host profile.

## Main features

**Fully integrated with Conan**

When you define a docker runner inside your host profile you are saying that you want to build your code using conan inside a docker image. Conan will try to build or reuse a docker image and run a conan create conan inside it. After that, conan copy the cache information from your docker container inside your host cache, obtainig the same result like an standard conan create.

**Use the image you want or create one**

In order to have as much freedom as possible, you can use the precompiled image you want or use a Dockerfile in case you want to create it during the command execution. The only requirement is that the container has installed conan 2.3.0 or higher.

To use our own Dockerfile you have to define its absolute path in the `dockerfile` variable. In addition, you can define the `build_context` of the container in case it is not the same as the one defined in the `dockerfile` variable.

Finally, if you want to use a pre-existing image or you want to give a specific name to the one you are compiling, you have the variable `image` where to define it.

**Full control over the management of your host chache**

To have a better control of what is being built, there are several ways to work with the cache inside the container using the profile variable `cache`.

- `clean`: the container uses an empty cache. This is the default mode.
- `copy`: copy the host cache inside the container using the conan cache save/restore command.
- `shared`: mount the host’s Conan cache as a shared volume.

In all cases the result of the `conan create` command will be exactly the same and will end up stored in the local host cache, what changes is the starting point of the container cache and the way the result of the command ends up in the local host cache.

**Control over the lifecycle of your containers**

By default conan doesn't remove the container once the execution is finished. This way you can reuse the same container in case you want to run another create on the same container.

- `suffix`: *docker* by default. Define the suffix name used to create your container *conan-runner-*`suffix`.
- `remove`: *false* by default. If you want to delete you container after an execution set it to *true*.

**Host profile example**

This is an example of how a profile with the runner section would look like for a docker container in which we want to build the image using a specific build_context and keeping the container for future executions.

```
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux
[runner]
type=docker
dockerfile=/Users/conan/dockerfiles/Dockerfile.gnu17
build_context=/Users/conan/my_lib/
image=conan-runner-gnu17
cache=copy
suffix=gnu17
remove=true
```

## Do you need more control when running or building your containers? The configfile runner is the answer.

If you need more control over the build and execution of the container, you can define more parameters inside a configfile yaml.

For example, you can add arguments in the build step or environment variables when you launch the container.

This is the template with all the parameters it accepts:

```yaml
image: image_name # The image to build or run.
build:
    dockerfile: /dockerfile/path # Dockerfile path.
    build_context: /build/context/path # Path within the build context to the Dockerfile.
    build_args: # A dictionary of build arguments
        foo: bar
    cacheFrom: # A list of images used for build cache resolution
        - image_1
run:
    name: container_name # The name for this container.
    containerEnv: # Environment variables to set inside the container.
        env_var_1: env_value
    containerUser: user_name # Username or UID to run commands as inside the container.
    privileged: False # Run as privileged
    capAdd: # Add kernel capabilities.
        - SYS_ADMIN
        - MKNOD
    securityOpt: # A list of string values to customize labels for MLS systems, such as SELinux.
        - opt_1
    mount: # A dictionary to configure volumes mounted inside the container.
        /home/user1/: # The host path or a volume name
            bind: /mnt/vol2 # The path to mount the volume inside the container
            mode: rw # rw to mount the volume read/write, or ro to mount it read-only.
```

To use it, you just need to add it in the host profile as always.

```
[settings]
...
[runner]
type=docker
configfile=</my/runner/folder>/configfile
cache=copy
remove=false
```

## Lets try it!

In this example we are going to see how to use a docker runner configfile to define our Dockerfile base image to compile zlib. Let’s create two profiles and a Dockerfile inside your project folder.

```bash
$ cd </my/runner/folder>
$ tree
.
├── Dockerfile
├── configfile
├── docker_example_build
└── docker_example_host
```

``docker_example_host`` profile

```
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux
[runner]
type=docker
configfile=</my/runner/folder>/configfile
cache=copy
remove=false
```

``docker_example_build`` profile

```
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux
```

```dockerfile
ARG BASE_IMAGE
FROM $BASE_IMAGE
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential \
        cmake \
        python3 \
        python3-pip \
        python3-venv \
    && rm -rf /var/lib/apt/lists/*
RUN pip install conan
```

Now, we need to write the `configfile` to defile the `BASE_IMAGE` variable defined inside the Dockerfile.

``configfile``

```yaml

    image: my-conan-runner-image
    build:
        dockerfile: </my/runner/folder>
        build_context: </my/runner/folder>
        build_args:
            BASE_IMAGE: ubuntu:22.04
    run:
        name: my-conan-runner-container
```

We are going to start from a totally clean docker, without containers or images. In addition, we are going to have the conan cache also completely empty.

```bash
$ conan list "*:*"
Found 0 pkg/version recipes matching * in local cache

$ docker ps --all
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

$ docker images  
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE
```

Now, lets clone and build zlib from conan-center-index and create it using our new runner definition.

```bash    
$ git clone https://github.com/conan-io/conan-center-index.git --depth 1
$ conan create ./conan-center-index/recipes/zlib/all --version 1.3.1 -pr:h </my/runner/folder>/docker_example_host -pr:b </my/runner/folder>/docker_example_build

...

┌──────────────────────────────────────────────────┐
| Building the Docker image: my-conan-runner-image |
└──────────────────────────────────────────────────┘

Dockerfile path: '</my/runner/folder>/Dockerfile'
Docker build context: '</my/runner/folder>'

Step 1/5 : ARG BASE_IMAGE

Step 2/5 : FROM $BASE_IMAGE

...

Successfully built 286df085400f
Successfully tagged my-conan-runner-image:latest

...

┌───────────────────────────────┐
| Creating the docker container |
└───────────────────────────────┘
...

┌─────────────────────────────────────────┐
| Container my-conan-runner-image running |
└─────────────────────────────────────────┘
...

┌─────────────────────────────────────────┐
| Running in container: "conan --version" |
└─────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
| Running in container: "conan cache restore "/root/conanrunner/all/.conanrunner/local_cache_save.tgz"" |
└───────────────────────────────────────────────────────────────────────────────────────────────────────┘

Restore: zlib/1.3.1 in p/zlib95420566fc0dd
Local Cache
zlib
    zlib/1.3.1
    revisions
        e20364c96c45455608a72543f3a53133 (2024-04-29 17:19:32 UTC)
        packages
        recipe_folder: p/zlib95420566fc0dd


┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
| Running in container: "conan create /root/conanrunner/all --version 1.3.1 -pr:h /root/conanrunner/all/.conanrunner/profiles/docker_example_host_1 -pr:b /root/conanrunner/all/.conanrunner/profiles/docker_example_build_0 -f json > create.json" |
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘


======== Exporting recipe to the cache ========
zlib/1.3.1: Exporting package recipe: /root/conanrunner/all/conanfile.py
zlib/1.3.1: exports: File 'conandata.yml' found. Exporting it...
zlib/1.3.1: Calling export_sources()
zlib/1.3.1: Copied 1 '.yml' file: conandata.yml
zlib/1.3.1: Copied 1 '.py' file: conanfile.py
zlib/1.3.1: Copied 1 '.patch' file: 0001-fix-cmake.patch
zlib/1.3.1: Exported to cache folder: /root/.conan2/p/zlib95420566fc0dd/e
zlib/1.3.1: Exported: zlib/1.3.1#e20364c96c45455608a72543f3a53133 (2024-04-29 17:19:32 UTC)

======== Input profiles ========
Profile host:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux

Profile build:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux


======== Computing dependency graph ========
Graph root
    cli
Requirements
    zlib/1.3.1#e20364c96c45455608a72543f3a53133 - Cache

======== Computing necessary packages ========
zlib/1.3.1: Forced build from source
Requirements
    zlib/1.3.1#e20364c96c45455608a72543f3a53133:b647c43bfefae3f830561ca202b6cfd935b56205 - Build

======== Installing packages ========
zlib/1.3.1: Calling source() in /root/.conan2/p/zlib95420566fc0dd/s/src

-------- Installing package zlib/1.3.1 (1 of 1) --------
zlib/1.3.1: Building from source
zlib/1.3.1: Package zlib/1.3.1:b647c43bfefae3f830561ca202b6cfd935b56205
zlib/1.3.1: Copying sources to build folder
zlib/1.3.1: Building your package in /root/.conan2/p/b/zlib8dd8e27348e8c/b
zlib/1.3.1: Calling generate()
zlib/1.3.1: Generators folder: /root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release/generators
zlib/1.3.1: CMakeToolchain generated: conan_toolchain.cmake
zlib/1.3.1: CMakeToolchain generated: /root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release/generators/CMakePresets.json
zlib/1.3.1: CMakeToolchain generated: /root/.conan2/p/b/zlib8dd8e27348e8c/b/src/CMakeUserPresets.json
zlib/1.3.1: Generating aggregated env files
zlib/1.3.1: Generated aggregated env files: ['conanbuild.sh', 'conanrun.sh']
zlib/1.3.1: Calling build()
zlib/1.3.1: Apply patch (conan): separate static/shared builds, disable debug suffix
zlib/1.3.1: Running CMake.configure()
zlib/1.3.1: RUN: cmake -G "Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE="generators/conan_toolchain.cmake" -DCMAKE_INSTALL_PREFIX="/root/.conan2/p/b/zlib8dd8e27348e8c/p" -DCMAKE_POLICY_DEFAULT_CMP0091="NEW" -DCMAKE_BUILD_TYPE="Release" "/root/.conan2/p/b/zlib8dd8e27348e8c/b/src"
-- Using Conan toolchain: /root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release/generators/conan_toolchain.cmake
-- Conan toolchain: Setting CMAKE_POSITION_INDEPENDENT_CODE=ON (options.fPIC)
-- Conan toolchain: Setting BUILD_SHARED_LIBS = OFF
-- The C compiler identification is GNU 11.4.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Looking for sys/types.h
-- Looking for sys/types.h - found
-- Looking for stdint.h
-- Looking for stdint.h - found
-- Looking for stddef.h
-- Looking for stddef.h - found
-- Check size of off64_t
-- Check size of off64_t - done
-- Looking for fseeko
-- Looking for fseeko - found
-- Looking for unistd.h
-- Looking for unistd.h - found
-- Renaming
--     /root/.conan2/p/b/zlib8dd8e27348e8c/b/src/zconf.h
-- to 'zconf.h.included' because this file is included with zlib
-- but CMake generates it automatically in the build directory.
-- Configuring done
-- Generating done
-- Build files have been written to: /root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release
zlib/1.3.1: Running CMake.build()
zlib/1.3.1: RUN: cmake --build "/root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release" -- -j16
[ 12%] Building C object CMakeFiles/zlib.dir/adler32.c.o
[ 12%] Building C object CMakeFiles/zlib.dir/compress.c.o
[ 18%] Building C object CMakeFiles/zlib.dir/deflate.c.o
[ 25%] Building C object CMakeFiles/zlib.dir/crc32.c.o
[ 31%] Building C object CMakeFiles/zlib.dir/gzlib.c.o
[ 37%] Building C object CMakeFiles/zlib.dir/gzread.c.o
[ 43%] Building C object CMakeFiles/zlib.dir/gzclose.c.o
[ 56%] Building C object CMakeFiles/zlib.dir/infback.c.o
[ 56%] Building C object CMakeFiles/zlib.dir/gzwrite.c.o
[ 62%] Building C object CMakeFiles/zlib.dir/inflate.c.o
[ 68%] Building C object CMakeFiles/zlib.dir/inffast.c.o
[ 75%] Building C object CMakeFiles/zlib.dir/trees.c.o
[ 81%] Building C object CMakeFiles/zlib.dir/zutil.c.o
[ 87%] Building C object CMakeFiles/zlib.dir/uncompr.c.o
[ 93%] Building C object CMakeFiles/zlib.dir/inftrees.c.o
[100%] Linking C static library libz.a
[100%] Built target zlib
zlib/1.3.1: Package 'b647c43bfefae3f830561ca202b6cfd935b56205' built
zlib/1.3.1: Build folder /root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release
zlib/1.3.1: Generating the package
zlib/1.3.1: Packaging in folder /root/.conan2/p/b/zlib8dd8e27348e8c/p
zlib/1.3.1: Calling package()
zlib/1.3.1: Running CMake.install()
zlib/1.3.1: RUN: cmake --install "/root/.conan2/p/b/zlib8dd8e27348e8c/b/build/Release" --prefix "/root/.conan2/p/b/zlib8dd8e27348e8c/p"
-- Install configuration: "Release"
-- Installing: /root/.conan2/p/b/zlib8dd8e27348e8c/p/lib/libz.a
-- Installing: /root/.conan2/p/b/zlib8dd8e27348e8c/p/include/zconf.h
-- Installing: /root/.conan2/p/b/zlib8dd8e27348e8c/p/include/zlib.h

zlib/1.3.1: package(): Packaged 1 file: LICENSE
zlib/1.3.1: package(): Packaged 2 '.h' files: zlib.h, zconf.h
zlib/1.3.1: package(): Packaged 1 '.a' file: libz.a
zlib/1.3.1: Created package revision fd85b1346d5377ae2465645768e62bf2
zlib/1.3.1: Package 'b647c43bfefae3f830561ca202b6cfd935b56205' created
zlib/1.3.1: Full package reference: zlib/1.3.1#e20364c96c45455608a72543f3a53133:b647c43bfefae3f830561ca202b6cfd935b56205#fd85b1346d5377ae2465645768e62bf2
zlib/1.3.1: Package folder /root/.conan2/p/b/zlib8dd8e27348e8c/p
WARN: deprecated: Usage of deprecated Conan 1.X features that will be removed in Conan 2.X:
WARN: deprecated:     'cpp_info.names' used in: zlib/1.3.1

======== Launching test_package ========

======== Computing dependency graph ========
Graph root
    zlib/1.3.1 (test package): /root/conanrunner/all/test_package/conanfile.py
Requirements
    zlib/1.3.1#e20364c96c45455608a72543f3a53133 - Cache

======== Computing necessary packages ========
Requirements
    zlib/1.3.1#e20364c96c45455608a72543f3a53133:b647c43bfefae3f830561ca202b6cfd935b56205#fd85b1346d5377ae2465645768e62bf2 - Cache

======== Installing packages ========
zlib/1.3.1: Already installed! (1 of 1)
WARN: deprecated: Usage of deprecated Conan 1.X features that will be removed in Conan 2.X:
WARN: deprecated:     'cpp_info.names' used in: zlib/1.3.1

======== Testing the package ========
Removing previously existing 'test_package' build folder: /root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release
zlib/1.3.1 (test package): Test package build: build/gcc-11-x86_64-gnu17-release
zlib/1.3.1 (test package): Test package build folder: /root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release
zlib/1.3.1 (test package): Writing generators to /root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release/generators
zlib/1.3.1 (test package): Generator 'CMakeToolchain' calling 'generate()'
zlib/1.3.1 (test package): CMakeToolchain generated: conan_toolchain.cmake
zlib/1.3.1 (test package): CMakeToolchain generated: /root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release/generators/CMakePresets.json
zlib/1.3.1 (test package): CMakeToolchain generated: /root/conanrunner/all/test_package/CMakeUserPresets.json
zlib/1.3.1 (test package): Generator 'CMakeDeps' calling 'generate()'
zlib/1.3.1 (test package): CMakeDeps necessary find_package() and targets for your CMakeLists.txt
    find_package(ZLIB)
    target_link_libraries(... ZLIB::ZLIB)
zlib/1.3.1 (test package): Generator 'VirtualRunEnv' calling 'generate()'
zlib/1.3.1 (test package): Generating aggregated env files
zlib/1.3.1 (test package): Generated aggregated env files: ['conanrun.sh', 'conanbuild.sh']

======== Testing the package: Building ========
zlib/1.3.1 (test package): Calling build()
zlib/1.3.1 (test package): Running CMake.configure()
zlib/1.3.1 (test package): RUN: cmake -G "Unix Makefiles" -DCMAKE_TOOLCHAIN_FILE="generators/conan_toolchain.cmake" -DCMAKE_INSTALL_PREFIX="/root/conanrunner/all/test_package" -DCMAKE_POLICY_DEFAULT_CMP0091="NEW" -DCMAKE_BUILD_TYPE="Release" "/root/conanrunner/all/test_package"
-- Using Conan toolchain: /root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release/generators/conan_toolchain.cmake
-- Conan toolchain: C++ Standard 17 with extensions ON
-- The C compiler identification is GNU 11.4.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Conan: Target declared 'ZLIB::ZLIB'
-- Configuring done
-- Generating done
-- Build files have been written to: /root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release
zlib/1.3.1 (test package): Running CMake.build()
zlib/1.3.1 (test package): RUN: cmake --build "/root/conanrunner/all/test_package/build/gcc-11-x86_64-gnu17-release" -- -j16
[ 50%] Building C object CMakeFiles/test_package.dir/test_package.c.o
[100%] Linking C executable test_package
[100%] Built target test_package

======== Testing the package: Executing test ========
zlib/1.3.1 (test package): Running test()
zlib/1.3.1 (test package): RUN: ./test_package
Compressed size is: 21
Compressed string is: Conan Package Manager
Compressed size is: 22
Compressed string is: xsKHLNLOUMRE
ZLIB VERSION: 1.3.1


┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
| Restore host cache from: </my/runner/folder>/conan-center-index/recipes/zlib/all/.conanrunner/docker_cache_save.tgz |
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

Restore: zlib/1.3.1 in p/zlib95420566fc0dd
Restore: zlib/1.3.1:b647c43bfefae3f830561ca202b6cfd935b56205 in p/zlibd59462fc4358e/p
Restore: zlib/1.3.1:b647c43bfefae3f830561ca202b6cfd935b56205 metadata in p/zlibd59462fc4358e/d/metadata

┌────────────────────┐
| Stopping container |
└────────────────────┘
```

If we now check the status of our Conan and docker cache, we will see the zlib package compiled for Linux and the new docker image and container.

```bash
$ conan list "*:*"
Found 1 pkg/version recipes matching * in local cache
Local Cache
zlib
    zlib/1.3.1
    revisions
        e20364c96c45455608a72543f3a53133 (2024-04-29 17:18:07 UTC)
        packages
            b647c43bfefae3f830561ca202b6cfd935b56205
            info
                settings
                arch: x86_64
                build_type: Release
                compiler: gcc
                compiler.version: 11
                os: Linux
                options
                fPIC: True
                shared: False

$ docker ps --all
CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS                       PORTS     NAMES
1379072ae424   my-conan-runner-image   "/bin/bash -c 'while…"   17 seconds ago   Exited (137) 2 seconds ago             my-conan-runner-image

$ docker images  
REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
my-conan-runner   latest    383b905f352e   22 minutes ago   531MB
ubuntu            22.04     437ec753bef3   12 days ago      77.9MB
```

If we run the ``conan create`` command again we will see how Conan reuses the previous container because we have set ``remove=False``.

```bash    
$ conan create ./conan-center-index/recipes/zlib/all --version 1.3.1 -pr:h </my/runner/folder>/docker_example_host -pr:b </my/runner/folder>/docker_example_build

...

┌───────────────────────────────┐
| Starting the docker container |
└───────────────────────────────┘

...
```