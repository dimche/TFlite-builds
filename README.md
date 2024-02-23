# TFlite-builds

TFlite cross-platform builds

##Why this repo?

The information provided in the [official tensorflow lite page for building `whl` packages for ARM](https://www.tensorflow.org/lite/guide/build_cmake_pip) to cross-compile TFlite for different architectures is a good start. However there seem to be several issues. This repo aims at providing more info towards successful compilation. And some binaries as well. 

The provided builds are fully compatible with  [Coral.ai EdgeTPU through the updated libedgetpu drivers](https://github.com/feranick/libedgetpu).

## Building - Docker

- Install docker:

```
$ sudo apt install docker.io
```
- Download tensorflow and checkout the relevant version.

```
$ git clone https://github.com/tensorflow/tensorflow.git
$ cd tensorflow
$ git checkout vX.XX.XX
```

- Modify the file `tensorflow/lite/tools/pip_package/Dockerfile.py3` to add missing dependencies (`apt-utils` and `tzdata`)

```
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive TZ=Etc/UTC \
    apt-get install -y \
      apt-utils \
      build-essential \
      software-properties-common \
      zlib1g-dev  \
      curl \
      wget \
      unzip \
      tzdata \
      git && \
    apt-get clean
```
If compiling against `python 3.11` or newer, you have to allow installation of whl packages from external sources. Add the following line in `tensorflow/lite/tools/pip_package/Dockerfile.py3`:
```
    RUN ln -sf /usr/bin/python$PYTHON_VERSION /usr/bin/python3
    RUN curl -OL https://bootstrap.pypa.io/get-pip.py
->  RUN rm /usr/lib/python3.11/EXTERNALLY-MANAGED
    RUN python3 get-pip.py
    RUN rm get-pip.py
```

- Edit Makefile to your system `tensorflow/lite/tools/pip_package/Makefile`:

```
# Values: debian:<version>, ubuntu:<version>
BASE_IMAGE ?= ubuntu:22.04
PYTHON_VERSION ?= 3.11
NUMPY_VERSION ?= 1.24.2
```

Note: if you are building against Ubuntu, this is all you need to change. If you are building for `debian:bookworm` or `debian:bullseye`, you need to remove/comment the following line from `tensorflow/lite/tools/pip_package/Dockerfile.py3`, since the added ppa repository is specific only to ubuntu.

```
RUN yes | add-apt-repository ppa:deadsnakes/ppa
```
- Run compilation (adjust the values for `TENSORFLOW_TARGET` and `PYTHON_VERSION` to fit your needs:

```
$ make -C tensorflow/lite/tools/pip_package docker-build TENSORFLOW_TARGET=aarch64 PYTHON_VERSION=3.11
```

These are the supported targets.

- `armhf`:  ARMv7 VFP with Neon Compatible with Raspberry Pi 3 and 4
- `rpi0`: ARMv6 Compatible with Raspberry Pi Zero
- `aarch64`: aarch64 (ARM 64-bit) Coral Mendel Linux 4.0 or Raspberry Pi with Ubuntu Server 20.04.01 LTS 64-bit of 22.04.x LTS 64-bit
- `native`: 	Your workstation 	It builds with "-mnative" optimization

## Building - Native
### Using Bazel
```
PYTHON=python3 tensorflow/lite/tools/pip_package/build_pip_package_with_bazel.sh native
```

### Using CMake
```
PYTHON=python3 tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh native
```

## GPU support for native builds
When compiling with either `docker` or native using `cmake` GPU support is disableed by default. To enable it, edit this file:
```
nano tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh
```
and add to line 110:
```
     native)
       BUILD_FLAGS=${BUILD_FLAGS:-"-march=native -I${PYTHON_INCLUDE} -I${PYBIND11_INCLUDE} -I${NUMPY_INCLUDE}"}
       cmake \
         -DCMAKE_C_FLAGS="${BUILD_FLAGS}" \
         -DCMAKE_CXX_FLAGS="${BUILD_FLAGS}" \
  ==>    -DTFLITE_ENABLE_GPU=ON \
         "${TENSORFLOW_LITE_DIR}"
```

## WARNING: RAM usage is extensive

Regardless of the method used (docker, native, etc), compilation is VERY RAM intensive (for instance, a typical compilation run using Docker, uses 22GB RAM - the max I have on my system - and 4GB/swap). If your swap is fixed (i.e. cannot be dynamically extended), it may lead to the failure (`Killed signal terminated program cc1plus`). This massive RAM use is due to the lack of limits on the number of processes that can be run during compilation. If you have plenty of RAM, that may be fine. If not, you can restrict the number of processes by changing line 126 in `tensorflow/lite/tools/pip_package/build_pip_package_with_cmake.sh` from:

```
cmake --build . --verbose -j ${BUILD_NUM_JOBS} -t _pywrap_tensorflow_interpreter_wrapper
```
to:
```
cmake --build . --verbose -j XXX -t _pywrap_tensorflow_interpreter_wrapper
```

where XXX is a number between 4 (1 process, slower) and 8 (8 processes, faster but more RAM hungry). 
