# Margo（面向 eBPF 与链路追踪友好定制版）

[English](README.md) | [中文](README.zh.md)

> 🔬 **研究项目 Fork 说明**：本仓库是 [mochi-hpc/mochi-margo](https://github.com/mochi-hpc/mochi-margo) 的专用 Fork，作为 [UnifyFS eBPF 性能诊断项目](https://github.com/RiveDroite1795/UnifyFS-ebpf-diagnosis) 的底层依赖组件维护。
>
> 本仓库**仅修改 `CMakeLists.txt`**，未改动 Margo、Argobots 或 Mercury 的任何源码，因此行为与 ABI 与上游完全一致；改动只影响代码被“如何编译”。

---

## 📌 本 Fork 改动说明

对用户态线程运行时做追踪，比对普通库做追踪更麻烦：Margo 在发起 Mercury 操作后会挂起调用者，操作完成时再恢复执行，所以一次 RPC 在时间线上表现为一次快速的上下文切换，而非一次长时间的调用。要让这个切换在 `uprobe` / `bpftrace` 下可观测，构建时必须保证微线程入口点不被优化消除、返回地址可信。

所有改动集中在 `CMakeLists.txt`：

1. **新增 `MARGO_EBPF_FRIENDLY` 开关（默认开启）**
   - 加入 `-g3`，保留完整的 DWARF 调试信息。
   - 加入 `-fno-omit-frame-pointer`，确保 `%rbp` 始终是真实帧指针，栈展开不会断裂。
   - 加入 `-fno-optimize-sibling-calls`，防止尾调用优化覆盖原始返回地址。
   - 加入 `-fno-inline`，保证微线程入口点与 RPC 派发辅助函数各自保有独立的符号边界。
   - 设置 `CMAKE_INSTALL_DO_STRIP FALSE`，`make install` 时不再 strip 库文件，使 `perf`、`bpftrace`、`uprobe` 能从已安装的 `.so` 中解析出符号。

2. **新增 `MARGO_FORCE_NO_OPTIMIZATION` 开关（默认开启）**
   - 提供独立的辅助函数 `_margo_strip_optimization_flags()`，先用正则从 `CMAKE_C_FLAGS` **以及每一种构建配置**（`DEBUG`、`RELEASE`、`RELWITHDEBINFO`、`MINSIZEREL`）中移除已有的 `-O0/-O1/-O2/-O3/-Os/-Og`，再统一追加 `-O0`。
   - **必须先移除再追加**：`CMAKE_C_FLAGS_RELEASE` 本身以 `-O3` 结尾，若只是简单追加 `-O0`，会被前面的 `-O3` 覆盖，代码依然被内联，探针点依然不存在。

3. **默认构建类型**
   - 使用单配置生成器（Make/Ninja）且未指定构建类型时，`CMAKE_BUILD_TYPE` 默认为 `Debug`。

4. **修复 ASan 参数覆盖问题**
   - 原有的 `ENABLE_ASAN` 分支会**覆盖** `CMAKE_C_FLAGS`，现改为**追加**，ASan 构建不再丢失上述追踪相关参数。

---

## 🔨 构建与安装

两个开关默认均为 `ON`，直接构建即为追踪就绪状态：

```bash
git clone https://github.com/RiveDroite1795/mochi-margo.git
cd mochi-margo
mkdir build && cd build

cmake .. \
  -DCMAKE_INSTALL_PREFIX=$HOME/.local/unifyfs/stack/gcc-debug \
  -DBUILD_SHARED_LIBS=ON \
  -DENABLE_TESTS=OFF \
  -DMARGO_EBPF_FRIENDLY=ON \
  -DMARGO_FORCE_NO_OPTIMIZATION=ON

make -j$(nproc)
make install
```

验证构建产物确实保留了符号与帧指针：

```bash
# 应当只看到 -g3 / -fno-omit-frame-pointer / -O0，不应出现 -O2 / -O3
grep -o '\-O[0-9gs]\|\-fno-omit-frame-pointer' build/CMakeFiles/*/flags.make | sort -u

# 安装后的库不应被 strip
file $HOME/.local/unifyfs/stack/gcc-debug/lib64/libmargo.so
nm -D $HOME/.local/unifyfs/stack/gcc-debug/lib64/libmargo.so | grep -c ' T '
```

---

*(以下为 Margo 上游官方原始说明文档)*

![](https://github.com/mochi-hpc/mochi-margo/actions/workflows/test.yml/badge.svg?branch=main)
[![codecov](https://codecov.io/gh/mochi-hpc/mochi-margo/branch/main/graph/badge.svg)](https://codecov.io/gh/mochi-hpc/mochi-margo)

Margo provides Argobots-aware bindings to the Mercury RPC library.

Mercury (https://mercury-hpc.github.io/) is a remote procedure call
library optimized for use in HPC environments.  Its native API presents a
callback-oriented interface to manage asynchronous operation.  Argobots
(https://www.argobots.org/) is a user-level threading package.

Margo combines Mercury and Argobots to simplify development of distributed
services.  Mercury operations are presented as conventional blocking
operations, and RPC handlers are presented as sequential threads.  This
configuration enables high degree of concurrency while hiding the
complexity associated with asynchronous communication progress and callback
management.

Internally, Margo suspends callers after issuing a Mercury operation, and
automatically resumes them when the operation completes.  This allows
other concurrent user-level threads to make progress while Mercury
operations are in flight without consuming operating system threads.
The goal of this design is to combine the performance advantages of
Mercury's native event-driven execution model with the progamming
simplicity of a multi-threaded execution model.

A companion library called abt-io provides similar wrappers for POSIX I/O
functions: https://github.com/mochi-hpc/mochi-abt-io

Note that Margo should be compatible with any Mercury network
transport (NA plugin).  The documentation assumes the use of
the NA SM (shared memory) plugin that is built into Mercury for
simplicity.  This plugin is only valid for communication between
processes on a single node.  See [Using Margo with other Mercury NA
plugins](##using-margo-with-other-mercury-na-plugins) for information
on other configuration options.

##  Spack

The simplest way to install Margo is by installing the "mochi-margo" package
in spack (https://spack.io/).

##  Dependencies

* mercury  (git clone --recurse-submodules https://github.com/mercury-hpc/mercury.git)
* argobots (git clone https://github.com/pmodels/argobots.git)

### Recommended Mercury build options

* Mercury must be compiled with -DMERCURY_USE_BOOST_PP:BOOL=ON to enable the
  Boost preprocessor macros for encoding.
* Mercury should be compiled with -DMERCURY_USE_SELF_FORWARD:BOOL=ON in order to enable
  fast execution path for cases in which a Mercury service is linked into the same
  executable as the client

Example Mercury compilation:

```
mkdir build
cd build
cmake .. -DMERCURY_USE_SELF_FORWARD:BOOL=ON \
 -DBUILD_TESTING:BOOL=ON -DMERCURY_USE_BOOST_PP:BOOL=ON \
 -DCMAKE_INSTALL_PREFIX=/home/pcarns/working/install \
 -DBUILD_SHARED_LIBS:BOOL=ON -DCMAKE_BUILD_TYPE:STRING=Debug
make
make install
```

## Building

```
mkdir build
cd build
cmake .. -DENABLE_TESTS=ON -DCMAKE_INSTALL_PREFIX=/home/pcarns/working/install
make
make install
```


## Running examples

The examples subdirectory contains:

* margo-example-client.c: an example client
* margo-example-server.c: an example server
* my-rpc.[ch]: an example RPC definition

The following example shows how to execute them.  Note that when the server starts it will display the address that the client can use to connect to it.


```
$ examples/margo-example-server na+sm://
# accepting RPCs on address "na+sm://13367/0"
Got RPC request with input_val: 0
Got RPC request with input_val: 1
Got RPC request with input_val: 2
Got RPC request with input_val: 3
Got RPC request with input_val: 4
Got RPC request to shutdown

$ examples/margo-example-client na+sm://13367/0
ULT [0] running.
ULT [1] running.
ULT [2] running.
ULT [3] running.
Got response ret: 0
ULT [0] done.
Got response ret: 0
ULT [1] done.
Got response ret: 0
ULT [2] done.
Got response ret: 0
ULT [3] done.
```

The client will issue 4 concurrent RPCs to the server and wait for them to
complete.

## Running tests

`make test`

## Using Margo with the other NA plugins

See the [Mercury
documentation](http://mercury-hpc.github.io/documentation/) for details.
Margo is compatible with any Mercury transport and uses the same address
format.

## Debugging

See the [Debugging documentation](doc/debugging.md) for Margo debugging
features and strategies.

## Design details

![Margo architecture](doc/fig/margo-diagram.png)

Margo provides Argobots-aware wrappers to common Mercury library functions
like HG_Forward(), HG_Addr_lookup(), and HG_Bulk_transfer().  The wrappers
have the same arguments as their native Mercury counterparts except that no
callback function is specified.  Each function blocks until the operation
is complete.  The above diagram illustrates a typical control flow.

Margo launches a long-running user-level thread internally to drive
progress on Mercury and execute Mercury callback functions (labeled
```__margo_progress()``` above).  This thread can be assigned to a
dedicated Argobots execution stream (i.e., an operating system thread)
to drive network progress with a dedicated core.  Otherwise it will be
automatically scheduled when the caller's execution stream is blocked
waiting for network events as shown in the above diagram.

Argobots eventual constructs are used to suspend and resume user-level
threads while Mercury operations are in flight.

Margo allows several different threading/multicore configurations:
* The progress loop can run on a dedicated operating system thread or not
* Multiple Margo instances (and thus progress loops) can be
  executed on different operating system threads
* (for servers) a single Margo instance can launch RPC handlers
  on different operating system threads
