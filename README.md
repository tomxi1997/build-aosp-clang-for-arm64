# Toolchain build scripts

主要是使用此脚本更标准化的编译经打好（入）补丁的aosp clang的源码，为什么有这个东西呢？
注意只想在arm64设备上编译安卓内核比如在lxc/chroot/proot环境中或arm架构的电脑比Mac m1，没有与之匹配的编译器,因为全部都是x86平台所用。众所周知标准的linux内核一般都是用gcc,clang来编译，对于编译标准的linux内核用ubuntu自带的gcc,clang是没多大问题，但安卓内核就不一样了，安卓内核是经过多重修改（Google,高通，手机厂商）这样就导致不在符合标准，用一般的编译器编译就会出现各种问题，编译报错不过，警告，内核刷入不开机，反正就是各种问题，为解决这些问题于是Google就搞了aosp clang 。这个就是从llvm拉来源码然后搞出初始编译器，然后呢编译安卓内核，然后测试内核及编译器，然后呢发现错误对编译器补丁，周而复始。比如clang 487747就多达111个补丁。

一句来说就是在 arm64设备（手机，树莓派，Mac m1 ）上的｛linux/lxc/chroot/proot/等环境中｝借用此脚本标准化编译经补丁后的aosp clang源码，从而产生在arm64设备上的AOSP-Clang {事实是接近于真正的AOSP-Clang, 想完全达到x64上的AOSP clang 效果是不可能的除非Google 支持。


使用方法，手机lxc/chroot/proot环境均可

1.克隆此项目，注意此脚本只适合较新的clang版本比如clang16

git clone -b new https://github.com/tomxi1997/build-aosp-clang-for-arm64.git tc-build

cd ./tc-build/src/

git clone https://android.googlesource.com/toolchain/llvm-project

git clone https://android.googlesource.com/toolchain/llvm_android

然后参考这https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/refs/heads/main
实际编译好的x64的工具链的manifest_10742551.xml，用git checkout检出相应的提交，然后同理检出llvm_android中相对应的提交，并将补丁运用于llvm-project，编译。
以clang-r498229b为例

cd llvm-project

git checkout e34ed7d63863b45858e74126edaa738e75887800

cd ..

cd llvm_android

git checkout 67b7374615b157459d52e8e145745c9ee6dc86aa

cp -R ./patches/*.patch ../llvm-project/

cp -R ./patches/cherry/*.patch ../llvm-project/

cd ../llvm-project/


#!/bin/bash  
for file in *.patch ;  
do  
patch -p1 < $file ;  
done  


cd ...

chmod +x build.sh

./build.sh

3.最终在三星s10 骁龙855的lxc ubuntu 22.04 总耗时间1小时44分编译完成最终会安装在 /root/Toolchain/clang-r498229b,打包测试。

cd /root/Toolchain

XZ_OPT="-9" tar --warning=no-file-changed -cJf clang-r498229b.tar.xz clang-r498229b

或者安装多线程打包工具pixz
apt install pixz

tar -I pixz -cf clang-r498229b.tar.xz clang-r498229b














～－－－－－－－－～
分割线




There are times where a tip of tree LLVM build will have some issue fixed and it isn't available to you, maybe because it isn't in a release or it isn't available through your distribution's package management system. At that point, to get that fix, LLVM needs to be compiled, which sounds scary but is [rather simple](https://llvm.org/docs/GettingStarted.html). The `build-llvm.py` script takes it a step farther by trying to optimize both LLVM's build time by:

* Trimming down a lot of things that kernel developers don't care about:
  * Documentation
  * LLVM tests
  * Ocaml bindings
  * libfuzzer
* Building with the faster tools available (in order of fastest to slowest):
  * clang + lld
  * clang/gcc + ld.gold
  * clang/gcc + ld.bfd

## Getting started

These scripts have been tested in a Docker image of the following distributions with the following packages installed. LLVM has [minimum host tool version requirements](https://llvm.org/docs/GettingStarted.html#software) so the latest stable version of the chosen distribution should be used whenever possible to ensure recent versions of the tools are used. Build errors from within LLVM are expected if the tool version is not recent enough, in which case it will need to be built from source or installed through other means.

* ### Debian/Ubuntu

  ```
  apt install bc \
              binutils-dev \
              bison \
              build-essential \
              ca-certificates \
              ccache \
              clang \
              cmake \
              curl \
              file \
              flex \
              git \
              libelf-dev \
              libssl-dev \
              libstdc++-$(apt list libstdc++6 2>/dev/null | grep -Eos '[0-9]+\.[0-9]+\.[0-9]+' | head -1 | cut -d . -f 1)-dev \
              lld \
              make \
              ninja-build \
              python3-dev \
              texinfo \
              u-boot-tools \
              xz-utils \
              zlib1g-dev
  ```

* ### Fedora

  ```
  dnf install bc \
              binutils-devel \
              bison \
              ccache \
              clang \
              cmake \
              elfutils-libelf-devel \
              flex \
              gcc \
              gcc-c++ \
              git \
              lld \
              make \
              ninja-build \
              openssl-devel \
              python3 \
              texinfo-tex \
              uboot-tools \
              xz \
              zlib-devel
  ```

* ### Arch Linux / Manjaro

  ```
  pacman -S base-devel \
            bc \
            bison \
            ccache \
            clang \
            cpio \
            cmake \
            flex \
            git \
            libelf \
            lld \
            llvm \
            ninja \
            openssl \
            python3 \
            uboot-tools
  ```

* ### Clear Linux

  ```
  swupd bundle-add c-basic \
                   ccache \
                   curl \
                   dev-utils \
                   devpkg-elfutils \
                   devpkg-openssl \
                   git \
                   python3-basic \
                   which
  ```

  Additionally, to build PowerPC kernels, you will need to build the U-Boot tools because there is no distribution package. The U-Boot tarballs can be found [here](https://ftp.denx.de/pub/u-boot/) and they can be built and used like so:

  ```
  $ curl -LSs https://ftp.denx.de/pub/u-boot/u-boot-2021.01.tar.bz2 | tar -xjf -
  $ cd u-boot-2021.01
  $ make -j"$(nproc)" defconfig tools-all
  ...
  $ sudo install -Dm755 tools/mkimage /usr/local/bin/mkimage
  $ mkimage -V
  mkimage version 2021.01
  ```

  Lastly, Clear Linux has `${CC}`, `${CXX}`, `${CFLAGS}`, and `${CXXFLAGS}` in the environment, which messes with the heuristics of the script for selecting a compiler. By default, the script will attempt to use `clang` and `ld.lld` but the environment's value of `${CC}` and `${CXX}` is respected first so `gcc` and `g++` will be used. Clear Linux has optimized their `gcc` and `g++` so this is fine but if you would like to use `clang` and `clang++` instead, invoke the script like so:

  ```
  $ CC=clang CFLAGS= CXX=clang++ CXXFLAGS= ./build-llvm.py ...
  ```


Python 3.5.3+ is recommended, as that is what the script has been tested against. These scripts should be distribution agnostic. Please feel free to add different distribution install commands here through a pull request.

## build-llvm.py

By default, `./build-llvm.py` will clone LLVM, grab the latest binutils tarball (for the LLVMgold.so plugin), and build LLVM, clang, and lld, and install them into `install`.

The script automatically clones and manages the [`llvm-project`](https://github.com/llvm/llvm-project). If you would like to do this management yourself, such as downloading a release tarball from [releases.llvm.org](https://releases.llvm.org/), doing a more aggressive shallow clone (versus what is done in the script via `--shallow-clone`), or doing a bisection of LLVM, you just need to make sure that your source is in an `llvm-project` folder within the root of this repository and pass `--no-update` into the script. See [this comment](https://github.com/ClangBuiltLinux/tc-build/issues/75#issuecomment-604374071) for an example.

Run `./build-llvm.py -h` for more options and information.

## build-binutils.py

This script builds a standalone copy of binutils. By default, `./build-binutils.py` will download the [latest stable version](https://www.gnu.org/software/binutils/) of binutils, build for all architectures we currently care about (see the help text or script for the full list), and install them into `install`. Run `./build-binutils.py -h` for more options.

Building a standalone copy of binutils might be needed because certain distributions like Arch Linux (whose options the script uses) might symlink `/usr/lib/LLVMgold.so` to `/usr/lib/bfd-plugins` ([source](https://bugs.archlinux.org/task/28479)), which can cause issues when using the system's linker for LTO (even with `LD_LIBRARY_PATH`):

```
bfd plugin: LLVM gold plugin has failed to create LTO module: Unknown attribute kind (60) (Producer: 'LLVM9.0.0svn' Reader: 'LLVM 7.0.1')
```

Having a standalone copy of binutils (ideally in the same folder at the LLVM toolchain so that only one `PATH` modification is needed) works around this without any adverse side effects. Another workaround is bind mounting the new `LLVMgold.so` to `/usr/lib/LLVMgold.so`.

## Contributing

This repository openly welcomes pull requests! There are a few presubmit checks that run to make sure the code stays consistently formatted and free of bugs.

1. All Python files must be passed through [`yapf`](https://github.com/google/yapf). See the installation section for how to get it (it may also be available through your package manager).

2. All shell files must be passed through [`shfmt`](https://github.com/mvdan/sh) (specifically `shfmt -ci -i 4 -w`) and emit no [`shellcheck`](https://github.com/koalaman/shellcheck) warnings.

The presubmit checks will do these things for you and fail if the code is not formatted properly or has a shellcheck warning. Running these tools on the command line before submitting will make it easier to get your code merged.

Additionally, please write a detailed commit message about why you are submitting your change.

## Getting help

Please open an issue on this repo and include your distribution, shell, the command you ran, and the error output.
