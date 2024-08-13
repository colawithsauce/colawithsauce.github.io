+++
title = "记录编译 LLVM 并配置 MLIR 开发环境"
author = ["colawithsauce"]
date = 2024-08-13T17:24:00+08:00
draft = false
+++

首先先把 llvm pull 下来

```shell
git clone --depth=1 https://github.com/llvm/llvm-project
cd llvm-project/
mkdir build
cd build
```

然后再编译

```shell
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON ../llvm -DCMAKE_INSTALL_PREFIX=~/.local -DLLVM_ENABLE_PROJECTS="clang;mlir;llvm;clang-tools-extra;lld" -DLLVM_TARGETS_TO_BUILD="host;NVPTX;AMDGPU;AArch64"
ninja -j$(($(nproc) - 1))
ninja install
```

再把 ~/.local/bin 加一下 PATH

```shell
echo "PATH=~/.local/bin:\${PATH}" >> ~/.bashrc
```

然后新建 MLIR 工程的时候如何让工程能够找到 LLVM/MLIR 呢？在你的工程的 CMakeLists.txt 里面加上一句：

```cmake
set(CMAKE_PREFIX_PATH ~/.local)
```
