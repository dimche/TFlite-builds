# TFlite-builds

TFlite cross-platform builds

##Why this repo?

The information provided in the [official tensorflow lite page for building `whl` packages for ARM](https://www.tensorflow.org/lite/guide/build_cmake_pip) to cross-compile TFlite for different architectures is a good start. However there seem to be several issues. This repo aims at providing more info towards successful compilation. And some binaries as well.

## Building

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

Target 	Target architecture 	Comments
armhf 	ARMv7 VFP with Neon 	Compatible with Raspberry Pi 3 and 4
rpi0 	ARMv6 	Compatible with Raspberry Pi Zero
aarch64 	aarch64 (ARM 64-bit) 	Coral Mendel Linux 4.0
Raspberry Pi with Ubuntu Server 20.04.01 LTS 64-bit
native 	Your workstation 	It builds with "-mnative" optimization

