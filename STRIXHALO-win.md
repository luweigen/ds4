# DS4 原生 Windows + ROCm 移植可行性调查

针对 "在 Windows 上原生跑 DS4 的 Strix Halo / gfx1151 ROCm 后端" 这一方案的逐项调查与确认。

- 调查基线：commit `c1d4597`（`claude/strix-halo-rocm-windows-3zox5h`）
- 调查方法：静态普查 + **实证交叉编译**（x86_64-w64-mingw32-gcc 13）+ 外部事实核实
- 当前状态：DS4 **尚不支持** Windows。仓库中零 Windows 构建路径（全仓库 `_WIN32` 仅 1 处，在 `gguf-tools/deepseek4-quantize.c:40` 这个独立工具里）。

---

## 结论摘要

| # | 项目 | 结论 | 工作量 | 风险 |
|---|------|------|--------|------|
| P0 | 工具链 ABI 决策 | 必须全程 MSVC-ABI（clang-cl + hipcc），**不能**混用 MinGW | 决策项 | 中 |
| 1 | 构建系统 Makefile | 需新增 Windows 分支；构建宿主用 MSYS2/Git-Bash | 1–2 天 | 低 |
| 2 | 核心引擎 `ds4.c`（67k 行） | **仅 11 处报错点**，实测 shim 后 0 error | 1 天 | 低 |
| 3 | mmap / SSD streaming | mmap 需 Win32 shim；`madvise`/`fadvise`/`O_DIRECT` 已全部条件编译 | 2–3 天 | 中（性能） |
| 4 | 线程 pthread（691 处调用） | 只用可移植子集，无 barrier/rwlock/affinity | 2–3 天 | 低 |
| 5 | 网络 Winsock | 实测 `ds4_distributed.c` 0 error，`ds4_tp.c` 仅缺 `iovec` | 2–3 天 | 低 |
| 6 | 终端 / REPL | linenoise 仅 3 个接触点；已用 VT 转义 | 1–2 天 | 低 |
| 7 | RDMA / `infiniband` | **非问题**——整段被 `__APPLE__` 包住 | 0 | 无 |
| 8 | libc 缺口 | `pread`/`dprintf`/`fmemopen`/`getpagesize`/`mkdir`/`regex`/`fnmatch` | 2–3 天 | 低 |
| 9 | HIP SDK for Windows 可用性 | gfx1151 已列入支持；rocWMMA/hipBLASLt 已随 SDK 发布 | 依赖外部 | **高** |
| 10 | GPU 可见内存 & 单次分配 | Windows 上限 96 GB；且 `hipMalloc > 64 GiB` 曾失败 | 见下 | **最高** |
| 11 | 子进程（agent/web） | `fork`+`/bin/sh` 语义性移植 | 3–5 天 | 中 |
| 12 | 测试 / QA 流水线 | 全部 POSIX shell，需 MSYS2 | 1–2 天 | 低 |

**总体判断**：语言层与主机代码层的移植难度**远低于预期**——真正的风险全部集中在第 9、10 两项，即 Windows 侧 ROCm 栈本身，而不是 DS4 的代码。

---

## 0. 调查方法与可复现步骤

静态计数不足以判断工作量，所以本次调查实际装了 MinGW-w64 交叉工具链，把每个 C 源文件按 Windows 目标编译了一遍，用真实的编译器报错替代猜测。

```sh
sudo apt-get install -y gcc-mingw-w64-x86-64
CC=x86_64-w64-mingw32-gcc-posix

# 第一轮：裸编译，定位缺失头文件
$CC -fsyntax-only -std=c99 -D_GNU_SOURCE -I. ds4.c

# 第二轮：用空 stub 头文件顶掉 POSIX 头，暴露真实的符号缺口
$CC -fsyntax-only -std=c99 -D_GNU_SOURCE -I. -Istub ds4.c

# 第三轮：用附录 A 的 PoC shim，验证能否编译干净
$CC -fsyntax-only -std=c99 -I. -Ishim \
    -include shim/ds4_win_sysconf.h -include sys/file.h ds4.c
```

**重要限定**：MinGW 只是这次调查的*测量工具*。真正落地必须换成 MSVC-ABI 工具链（见 P0）。MinGW 的实验结果证明的是 **C 语言层与 POSIX 面的可移植性**，不是最终构建路径。

---

## P0. 工具链 ABI 决策（前置项）

**结论：必须全程使用 MSVC-ABI 的 clang（clang-cl）+ HIP SDK 的 hipcc；不能用 MinGW 编译主机代码再和 HIP 目标文件链接。**

Windows HIP SDK 的 `hipcc` 是面向 `x86_64-pc-windows-msvc` 的 clang++。MinGW 与 MSVC 的 C++ ABI、异常模型、CRT 都不兼容，二者产出的目标文件无法互相链接。AMD/社区的一致结论是：一个工程内 C、C++、HIP 三种语言必须统一走 MSVC 风格或 GNU 风格，不能混。

这直接决定了后续几项的实现方式：

- 线程 shim **不能**用 MinGW 的 winpthreads，要么自己写 Win32 shim（推荐，见第 4 项），要么引入 pthreads4w。
- POSIX shim 头文件必须对 MSVC CRT 成立（例如 `_read`/`_lseeki64`/`_mkdir` 而非 mingw 的 POSIX 别名）。
- 构建宿主仍可以是 MSYS2/Git-Bash（跑 make 和测试脚本），但被调用的编译器是 Windows 原生 clang-cl。

**好消息**：代码的 C 方言层面对 clang-cl 完全友好。全量普查 84,163 行核心 C 代码：

| 特性 | 出现次数 | 位置 / 说明 |
|------|---------|------------|
| VLA（变长数组） | **0** | `-Wvla` 实测全部源文件为 0 |
| 语句表达式 `({...})` | 0 | — |
| `__builtin_*` | 0 | — |
| 内联汇编 | 0 | 含 30k 行 ROCm kernel 也是 0 |
| `__int128` | 0 | — |
| `__attribute__` | 1 | `ds4.c:746` `DS4_MAYBE_UNUSED`，clang-cl 支持 |
| `__thread` | 1 | `ds4.c:1799` `g_parallel_depth`，clang-cl 支持 |
| `__typeof__` | 1 | `ds4_tp.c:590`，**在 `__APPLE__` 分支内**，Windows 不编译 |
| x86 SIMD intrinsics | 0 | 无 `immintrin.h` 依赖 |
| NEON | 112 | 全部被 `#if defined(__ARM_NEON)` 包住（`ds4.c:365`） |

零 VLA 这一点尤其关键——它意味着连 MSVC `cl.exe` 在语言层面都不会被卡住。

---

## 1. 构建系统（Makefile）

**结论：需要新增 Windows 分支，工作量小但必须做。**

`Makefile:2` 用 `uname -s` 二分，只有 `Darwin`（Metal）和「其余即 Linux」（CUDA/ROCm）两条路径：

```make
UNAME_S := $(shell uname -s)
ifeq ($(UNAME_S),Darwin)   # Metal
else                        # 假定 Linux：-D_GNU_SOURCE、/opt/rocm/bin/hipcc、-lhipblas -lhipblaslt
endif
```

`strix-halo` 目标（`Makefile:181`）硬编码了 Linux 假设：

```make
strix-halo:
	$(MAKE) -B ds4 ds4-server ds4-bench ds4-eval ds4-agent \
		CORE_OBJS="ds4.o ds4_distributed.o ds4_tp.o ds4_ssd.o ds4_rocm.o ..." \
		CFLAGS="$(CFLAGS) $(ROCM_HOST_CFLAGS) -DDS4_ROCM_BUILD" \
		DS4_LINK="$(HIPCC) $(ROCM_CFLAGS)" \
		DS4_LINK_LIBS="$(ROCM_LDLIBS)"
```

需要改动：

- `uname -s` 在 MSYS2 下返回 `MINGW64_NT-*` / `MSYS_NT-*`，需要加 `ifneq (,$(findstring NT,$(UNAME_S)))` 分支。
- `HIPCC ?= $(shell command -v hipcc || echo /opt/rocm/bin/hipcc)` → Windows 上是 `%HIP_PATH%\bin\hipcc.bat`。
- `ROCM_LDLIBS = -lm -pthread -lhipblas -lhipblaslt` → 换成 `amdhip64.lib hipblas.lib hipblaslt.lib`，`-lm`/`-pthread` 去掉。
- `-march=native`、`-ffast-math`、`-fPIC` → clang-cl 语法（`/clang:-march=native`，`-fPIC` 在 Windows 无意义）。
- 二进制名需要 `.exe` 后缀（make 的隐式规则和 `$(MAKE) -B ds4` 目标名都要跟着改）。

**最小目标的依赖面比想象中小。** `ds4` CLI 的链接列表（`Makefile:210`）是：

```
ds4: ds4_cli.o ds4_help.o linenoise.o ds4_gpu_args.o $(CORE_OBJS)
```

即 **不包含** `ds4_server.o` / `ds4_agent.o` / `ds4_web.o` / `ds4_kvstore.o` / `rax.o`。所以第 11 项（子进程/fork）在 M1 阶段可以完全绕开。

---

## 2. 核心引擎 `ds4.c` 的 POSIX 面

**结论：67,435 行的引擎主体，整个 Windows 移植只有 11 个报错点，集中在 4 个地方。**

这是本次调查最重要的发现。用空 stub 头顶掉 POSIX 头之后，`ds4.c` 的完整错误列表就是：

```
ds4.c:1850  '_SC_NPROCESSORS_ONLN' undeclared      ← sysconf 取 CPU 核数
ds4.c:2476  'MAP_SHARED' / 'MAP_PRIVATE'           ← 模型 mmap
ds4.c:2477  'PROT_READ'
ds4.c:2478  'MAP_FAILED'
ds4.c:3088  '_SC_PAGESIZE'                          ← 页大小
ds4.c:49442 'F_SETFD' / 'FD_CLOEXEC'                ← 锁文件
ds4.c:49444 'LOCK_EX' / 'LOCK_NB'                   ← flock
ds4.c:56143 '_SC_PAGESIZE'
```

加上 `-Wall -Wextra` 后仅多出 4 个隐式声明（见第 8 项），**没有任何格式化字符串警告、没有任何 LLP64 相关的整型警告**。

用附录 A 的 shim 重编：

```
ds4.c          errors=0
ds4_ssd.c      errors=0
```

**原因**：整个引擎的数值/图/量化/调度代码是纯 C99 + 自管理内存，操作系统接触面被有意收敛在装载与流式两处。这是可移植性上非常好的既有结构。

---

## 3. 内存映射与 SSD streaming

**结论：mmap 需要一个 Win32 shim；但流式路径的所有 advise/direct-IO 调用点都已经条件编译，Windows 上会自动降级而不是编译失败。**

### 3.1 模型映射（必须实现）

`ds4.c:2477` 是唯一的模型映射点：

```c
const int mmap_flags = metal_mapping ? MAP_SHARED : MAP_PRIVATE;
void *map = mmap(NULL, (size_t)st.st_size, PROT_READ, mmap_flags, fd, 0);
```

Windows 对应实现：`CreateFileMappingW(PAGE_READONLY)` + `MapViewOfFile(FILE_MAP_READ)`；`MAP_PRIVATE` → `FILE_MAP_COPY`。注意 80 GiB 级别的视图必须用 64 位偏移的 `MapViewOfFileEx`，且进程要有足够的地址空间（x64 下不是问题）。

`ds4_ssd.c` 里的 `mmap`/`mlock`（`ds4_ssd.c:149,183`）**只服务于 `--simulate-used-memory` 这个测试开关**，不是推理路径，可以在 M1 阶段直接 stub 掉返回失败。

### 3.2 流式提示（已条件编译，自动降级）

这几处都已经带保护，Windows 上无需改代码即可编译：

```c
// ds4.c:18475 — 已被 #if defined(POSIX_MADV_WILLNEED) 包住，
// 且后面紧跟一个手动逐页 touch 的循环作为 fallback
#if defined(POSIX_MADV_WILLNEED)
    (void)posix_madvise(..., POSIX_MADV_WILLNEED);
#endif

// rocm/ds4_rocm_runtime.cuh:5318  #if defined(POSIX_MADV_DONTNEED) ... #else (void)... #endif
// rocm/ds4_rocm_runtime.cuh:5337  #if defined(POSIX_FADV_DONTNEED) ... #else (void)... #endif
// rocm/ds4_rocm_runtime.cuh:1953/5432/6292  #if defined(__linux__) && defined(O_DIRECT)
```

**代价是性能而不是正确性**：Windows 上会失去 `POSIX_FADV_DONTNEED` 的页缓存回收和 `O_DIRECT` 的绕缓存读取。SSD streaming 模式下，80 GiB 模型反复经过页缓存会造成明显的内存压力。Windows 的对应手段是 `CreateFile(FILE_FLAG_NO_BUFFERING | FILE_FLAG_OVERLAPPED)`，属于 M2/M3 阶段的性能工作，不阻塞 M1。

### 3.3 `pread`

`ds4.c:18515` 在流式 page-in 的读回退路径上用了 `pread`：

```c
nread = pread(model->fd, buf, want, (off_t)pos);
```

Windows 无 `pread`。实现方式：`ReadFile` 配 `OVERLAPPED.Offset/OffsetHigh`（线程安全），不要用 `_lseeki64`+`_read`（多线程 page-in 会互相踩文件指针）。

---

## 4. 线程（pthread，691 处调用）

**结论：调用量很大，但只用了 pthread 的可移植子集，一个 ~250 行的 Win32 shim 可以全覆盖。**

分布：

```
ds4.c:74  ds4_distributed.c:205  ds4_tp.c:10  ds4_server.c:262  ds4_eval.c:8  ds4_agent.c:132
```

**用到的原语全集**（这是关键——不是调用次数，而是种类）：

```
pthread_mutex_lock/unlock/init/destroy          439 + 78
pthread_cond_wait/signal/broadcast/init/destroy  23 + 22 + 30 + 16 + 22
pthread_cond_timedwait                            2
pthread_create / join / detach                   28 / 24 / 6
pthread_once                                      2
pthread_mutexattr_init/settype/destroy            3   (一个递归锁)
```

**没有用到**：`pthread_barrier_*`、`pthread_rwlock_*`、`pthread_spin_*`、`pthread_setaffinity_np`、`pthread_cancel`、`pthread_key_*`。

这意味着整套可以直接映射到 Win32 原生原语，而且是 1:1 的：

| pthread | Win32 |
|---------|-------|
| `pthread_mutex_t` | `SRWLOCK`（递归锁那一处用 `CRITICAL_SECTION`） |
| `pthread_cond_t` | `CONDITION_VARIABLE` |
| `pthread_cond_timedwait` | `SleepConditionVariableSRW` 带超时 |
| `pthread_create/join` | `_beginthreadex` + `WaitForSingleObject` |
| `pthread_once` | `InitOnceExecuteOnce` |

自写 shim 优于引入 pthreads4w：没有额外 DLL 依赖，且能保证 MSVC-ABI 一致（见 P0）。

另外 `ds4.c:1850` 的 `sysconf(_SC_NPROCESSORS_ONLN)` → `GetSystemInfo().dwNumberOfProcessors`；Strix Halo 是 16 核 32 线程单 processor group，不需要处理 Windows 的 processor group 分组问题。

---

## 5. 网络（Winsock）

**结论：实测最轻的一项。`ds4_distributed.c`（8,437 行）在一个 ~25 行的 Winsock shim 头下编译 0 error。**

实测结果：

```
ds4_distributed.c  errors=0  warnings=11
ds4_tp.c           errors=0  warnings=4   (加上 struct iovec 定义后)
```

11 + 4 个警告全部是 Winsock 的签名差异，属于机械修正：

- `setsockopt` 第 4 参在 Windows 上是 `const char *` 而非 `const void *`（7 + 4 处）
- `send`/`recv` 的缓冲区是 `char *` 而非 `void *`（2 处 `-Wpointer-sign`）
- `if_nametoindex`（`ds4_distributed.c:1298`）在 Windows 上存在，只是要从 `ws2tcpip.h` 取
- `poll` → `WSAPoll`（`ds4_distributed.c:4131`）

需要额外补的：

- `WSAStartup`/`WSACleanup` 的进程级初始化（DS4 目前没有这个概念，要挂在 `main` 上）
- socket 句柄类型：代码用 `int` 存 socket，Windows 上是 `SOCKET`（`UINT_PTR`）。实践中 Windows socket 句柄值很小，但严格来说是截断，建议引入 `ds4_sock_t` typedef 统一。
- `close(sock)` → `closesocket(sock)`
- `struct iovec` + `writev`/`readv`（`ds4_tp.c` 唯一的 2 处）→ `WSABUF` + `WSASend`/`WSARecv`
- `SIGPIPE` / `MSG_NOSIGNAL` — Windows 无此概念，`ds4_tp.c:207,240` 已经用 `#ifdef MSG_NOSIGNAL` / `#ifdef SO_NOSIGPIPE` 保护，自动降级

---

## 6. 终端 / REPL

**结论：接触点极少，且 linenoise 已经在用 VT 转义序列，Windows 10+ 开 VT 模式即可。**

`linenoise.c`（2,690 行）报 45 个错误，但全部来自 3 个函数里的 termios/ioctl 常量：

```
linenoise.c:598-609  enableRawMode()   isatty + tcgetattr + tcsetattr(TCSAFLUSH)
linenoise.c:627      disableRawMode()  tcsetattr
linenoise.c:687      getColumns()      ioctl(1, TIOCGWINSZ, &ws)
```

Windows 实现：

- raw mode → `GetConsoleMode`/`SetConsoleMode`，清掉 `ENABLE_LINE_INPUT | ENABLE_ECHO_INPUT | ENABLE_PROCESSED_INPUT`，加上 `ENABLE_VIRTUAL_TERMINAL_INPUT`
- 输出端加 `ENABLE_VIRTUAL_TERMINAL_PROCESSING`——**加了之后 linenoise 里 22 行 `\x1b[` 转义序列全部原样可用**，不需要改写渲染逻辑
- `getColumns()` → `GetConsoleScreenBufferInfo`
- `read(STDIN_FILENO, &c, 1)`（`linenoise.c:2444`）→ `ReadConsoleInputW` 或直接 `_read`（VT 输入模式下可用）
- `isatty` → `_isatty`

信号：`ds4_cli.c:1584` 是唯一的 `sigaction` 使用点（REPL 的 SIGINT 处理）。MSVC CRT 支持 `signal(SIGINT, ...)`，但 Ctrl+C 的正确做法是 `SetConsoleCtrlHandler`。`ds4_server.c:13309-13315` 另有 SIGPIPE/SIGINT/SIGTERM 三处（M2 阶段）。

---

## 7. RDMA / InfiniBand —— 非问题

**结论：这一项在初步评估里被误列为风险，实际不存在。**

`ds4_tp.c` 的 verbs 依赖整段被 Apple 专属条件包住：

```c
// ds4_tp.c:32-38
#if defined(__APPLE__) && defined(__has_include)
#if __has_include(<infiniband/verbs.h>)
#include <infiniband/verbs.h>
#include <dlfcn.h>
#define DS4_TP_HAVE_VERBS 1
#endif
#endif
```

后续所有 9 处 `ibv_*` 调用和 4 处 `dlopen`/`dlsym` 都在 `#ifdef DS4_TP_HAVE_VERBS` 内，且 `dlopen` 的目标是 `/usr/lib/librdma.dylib`（`ds4_tp.c:585`）——这是 macOS 上的 RDMA 路径，不是 Linux ibverbs。Windows 编译时全部消失，**零工作量**。

---

## 8. libc 缺口清单

**结论：7 个函数，都不难，但 `regex` 是唯一需要引入第三方代码的。**

`-Wall -Wextra` 下 `ds4.c` 的 4 个隐式声明 + 其余文件的缺口：

| 函数 | 位置 | 处数 | Windows 方案 |
|------|------|------|-------------|
| `getpagesize` | `ds4.c:18455` | 1 | `GetSystemInfo().dwPageSize` |
| `pread` | `ds4.c:18515` | 1 | `ReadFile` + `OVERLAPPED`（见 3.3） |
| `dprintf` | `ds4.c:49472` | 1 | `vsnprintf` + `_write` |
| `fmemopen` | `ds4.c:52420, 52449` | 2 | 需要 ~60 行内存流实现；用于 session snapshot 存取。也可以改写成直接缓冲区读写 |
| `mkdir(path, mode)` | `ds4_kvstore.c:361,367`；`ds4_web.c` 2 处 | 4 | `_mkdir(path)`（单参数） |
| `fcntl(F_GETFL/F_SETFL, O_NONBLOCK)` | `ds4_server.c`、`ds4_web.c` | 各 1 组 | `ioctlsocket(FIONBIO)` |
| `flock(LOCK_EX\|LOCK_NB)` + `FD_CLOEXEC` | `ds4.c:49442-49444` | 1 组 | `LockFileEx(LOCKFILE_EXCLUSIVE_LOCK\|LOCKFILE_FAIL_IMMEDIATELY)`；CLOEXEC → `SetHandleInformation(HANDLE_FLAG_INHERIT, 0)` |
| `strcasecmp` / `strncasecmp` | 多处 | 25 | `_stricmp` / `_strnicmp` 宏映射 |
| `opendir`/`readdir` | `ds4_server.c` 3、`ds4_agent.c` 15、`ds4_kvstore.c` 3 | 21 | `FindFirstFileW`/`FindNextFileW` 封装，或用 dirent-for-Windows 单头文件 |
| `regcomp`/`regexec`/`regfree` | `ds4_agent.c` | 4 | **无 Windows 实现**，需引入 musl regex / TRE / PCRE2 兼容层（仅 M3 需要） |
| `fnmatch` | `ds4_agent.c` | 1 | `PathMatchSpecA`，或自己写 ~40 行 glob |

注意 `printf` 系列的格式化：MinGW 实测无格式警告；clang-cl 下 UCRT 已支持 `%zu`/`%llu`，无需 `__USE_MINGW_ANSI_STDIO` 之类的补丁。

---

## 9. HIP SDK for Windows 的可用性

**结论：三项依赖（HIP runtime / rocWMMA / hipBLASLt）在 Windows HIP SDK 中都已提供，gfx1151 也已列入支持列表——但这是本次调查中我无法在本地验证的部分，且历史上文档与实际支持存在不一致。**

DS4 的 ROCm 后端依赖面很干净。30,403 行 ROCm 代码，外部依赖只有三个头：

```
ds4_rocm.cu:3                 #include <hipblaslt/hipblaslt.h>
rocm/ds4_rocm_moe.cuh:7       #include <rocwmma/rocwmma.hpp>
（各处）                       #include <hip/hip_runtime.h>
```

主机侧 POSIX 面几乎为零——`ds4_rocm.cu` 只 include 了 `pthread.h` / `sys/stat.h` / `unistd.h`（第 30/34/36 行）。GPU 侧：

- `__builtin_amdgcn_wmma_f32_16x16x16_f16_w32` × 8、`__builtin_amdgcn_perm` × 3 —— clang 内建，任何 HIP clang 都有
- **零内联汇编**
- rocWMMA 用的是 `fragment<matrix_a, 16,16,16, half>`（`rocm/ds4_rocm_moe.cuh:4807`），即 RDNA3/3.5 的 WMMA 指令，gfx1151 原生支持

### 外部事实核实

- **gfx1151 官方支持**：ROCm 7.14.0 兼容性矩阵已把 Ryzen AI Max+ 395 / gfx1151 列入官方支持 GPU 列表；HIP SDK for Windows 7.2.0 的系统需求页面列入了 Ryzen AI 300/400 系列与 Ryzen AI Max 400 系列。
- **rocWMMA on Windows**：Windows HIP SDK 的 Math Libraries 组件中包含 rocWMMA。
- **hipBLASLt on Windows**：自 HIP SDK 6.4.2 起提供，但公开文档明确点名的是 **gfx1101**；gfx1151 是否有对应的 hipBLASLt tuning 库，文档未明确。
- **反面信号**：ROCm issue #5339「Confusing rocm support for gfx1151」至今标记为 "Under Investigation"，反映营销口径与兼容性矩阵长期不一致；ROCm 官方文档也仍在说明「整个 ROCm 栈尚未在 Windows 上完全支持」。

### 需要在真机上先验证的三件事

在写任何移植代码之前，应该先在目标 Windows 机器上跑这三个探针，成本极低但能一次性决定项目是否值得启动：

1. `hipInfo` / `hipconfig --full` 能否识别 gfx1151，`hipcc --offload-arch=gfx1151` 能否产出可执行 kernel。
2. 编译一个只有 `#include <rocwmma/rocwmma.hpp>` + 一次 `mma_sync` 的最小程序，确认 Windows SDK 的 rocWMMA 头树完整（Linux 上 `librocwmma-dev` 就缺 `rocwmma/internal/`，见 `STRIXHALO.md` 第 1 节——Windows 上很可能有同类问题）。
3. 跑一次 `hipblasLtMatmul` 的最小 GEMM，确认 gfx1151 有可用的 algo。若无，需要评估退回 `hipblas`（`rocm/ds4_rocm_hipblaslt.cuh` 目前是无条件 include，没有 fallback 分支，退回需要新增条件编译）。

---

## 10. GPU 可见内存与 >64 GiB 单次分配 —— 最高风险项

**结论：这是整个方案最可能直接失败的地方，且失败点在 DS4 代码之外。**

### 10.1 GPU 可见内存上限

`STRIXHALO.md` 第 3 节的关键一步在 Windows 上没有等价物：

```text
amd_iommu=off amdgpu.gttsize=126976 ttm.pages_limit=32505856 ttm.page_pool_size=32505856
```

Linux 上这套 GTT 内核参数能把 128 GB 机器的 GPU 可见内存推到 ~110–126 GB。**Windows 上的上限是 96 GB**——通过 BIOS UMA frame buffer + AMD Adrenalin 的 Variable Graphics Memory (VGM) 设为 Custom，128 GB 机器最多给 96 GB，留 32 GB 给系统。

DS4 的目标模型是 80.76 GiB，加运行时缓冲。96 GB ≈ 89.4 GiB 可见内存，**理论上放得下，但余量只有 ~8 GiB**，比 Linux 下紧张得多。`STRIXHALO.md` 第 5 节已经警告过：混合 IQ2/IQ4 的 GGUF 在 ROCm 路径上内存压力大到会触发系统 OOM 而非 DS4 的干净失败——Windows 上这个边界只会更窄。

### 10.2 单次分配 > 64 GiB（关键）

DS4 的常驻模型上传是**一次性分配整个模型镜像**，不是逐张量分配：

```c
// rocm/ds4_rocm_runtime.cuh:5711
cudaError_t err = cudaMalloc(&dev, (size_t)map_size);   // → hipMalloc
...
// rocm/ds4_rocm_runtime.cuh:5780
g_model_images.push_back({model_map, map_size, (char *)dev, map_offset});
```

（`cudaMalloc` 在 `ds4_rocm.h:33` 被 `#define` 成 `hipMalloc`。）

对 80.76 GiB 模型，单机不分层时 `map_size ≈ 80.76 GiB`，即**一次 >64 GiB 的 `hipMalloc`**。

而 ROCm/rocm-systems issue #1786 记录了：**Windows 上 `hipMalloc` 超过 64 GiB 会失败，同样的代码在 Linux 上成功**——复现环境正是 Radeon 8060S / gfx1151 / Ryzen AI Max+ PRO 395，Windows 10.0.26200 + ROCm 6.4，BIOS 已分配 96 GiB。该 issue 目前状态是 "fix submitted" 且已关闭，但修复进入哪个 Windows HIP SDK 版本需要确认。

**现有的规避路径（代码里已经有）**：

- `--layers a:b` 分层切片会产生多个不相交的 device image（`rocm/ds4_rocm_runtime.cuh:5782` 的注释明确说明「Sparse layer slices may create several disjoint device images」），每片可以压到 64 GiB 以下。
- SSD streaming 模式下只有非路由权重常驻，常驻分配远小于全模型。
- `README.md:409-416` 描述的 128 GB Strix Halo 推荐配置本来就是 GLM-5.2 routed Q2_K + 4096 context。

所以这一项不是死路，但意味着 **Windows 上很可能无法用「整模型常驻」这个最快的配置**，只能走分层或流式——这会改变性能预期，而不只是改变构建方式。

---

## 11. 子进程（ds4-agent / ds4-web）

**结论：唯一需要"语义性"移植而非"API 映射"的部分。建议排在 M3，M1/M2 不受影响。**

`ds4_agent.c:7715` 的工具执行：

```c
if (pipe(pipefd) != 0) ...
pid_t pid = fork();
    dup2(pipefd[1], STDOUT_FILENO);
    dup2(pipefd[1], STDERR_FILENO);
    execl("/bin/sh", "sh", "-c", cmd ? cmd : "", (char *)NULL);
```

Windows 没有 `fork`，要重写成 `CreateProcessW` + 匿名管道 + `CREATE_NO_WINDOW`。更麻烦的是语义：`/bin/sh -c` 在 Windows 上没有对等物，agent 生成的 shell 命令（管道、glob、`&&`）在 `cmd.exe` 下语义不同。现实选项是要求 MSYS2/Git-Bash 的 `sh.exe` 在 PATH 里，或者切到 PowerShell 并接受行为差异。

`ds4_agent.c:10582` 的 `pipe(w->wake_fd)` 自唤醒管道 → Windows 上用 `WSAEventSelect` + `WSACreateEvent`，或环回 socketpair。

`ds4_web.c:1039-1072` 的 `fork` + `execlp` 是拉起 Chrome（`--remote-allow-origins=*`），Windows 上换成 `CreateProcessW` 即可，比 agent 简单。

`waitpid(WNOHANG)`（`ds4_web.c:1089,1109`；`ds4_agent.c:7580,7669,7692`）→ `WaitForSingleObject(h, 0)` + `GetExitCodeProcess`。

---

## 12. 测试 / QA 流水线

**结论：全部依赖 POSIX shell，必须在 MSYS2/Git-Bash 下跑。**

```
tests/test_gpu_args_cli.sh          （make test-rocm 的一环，Makefile:208）
tests/dspark_acceptance_fixture.sh  （Makefile:495）
tests/glm_long_context_smoke.sh
download_model.sh
```

`Makefile:30-34` 还有内嵌的 shell 逻辑（`if [ -x ... ]`、`command -v`、`dirname`）。

`make test-mxfp4-rocm`（`Makefile` 的 ROCm 核心回归，不需要完整模型 GGUF，覆盖 1/3/32/128/512 token 的常驻解码与批式 routed-MoE）是 Windows 移植最好的第一个正确性门禁——它不依赖 80 GiB 模型，因此不受第 10 项内存问题影响，可以在移植早期就跑通。

`QA_BEFORE_RELEASES.md` 第 9 节的 ROCm 验收流程绑定了一台 Linux Framework Desktop（`antirez@strixhalo`），Windows 需要新增一条并行的验收路径。

---

## 未验证项（诚实清单）

以下结论**没有**在本次调查中实测，需要真机确认：

1. Windows HIP SDK 能否为 gfx1151 编译并运行 DS4 的 rocWMMA kernel（第 9 项的三个探针）。本环境无 Windows、无 AMD GPU、无 HIP SDK。
2. clang-cl 编译 `ds4.c` 的实际结果。本次用 MinGW 做代理测量；C 方言层面（零 VLA、零语句表达式、零 `__builtin`）强烈支持 clang-cl 也能通过，但未实测。
3. `hipMalloc > 64 GiB` 的修复是否已进入某个公开的 Windows HIP SDK 版本。
4. Windows 上 96 GB VGM 配置下，80.76 GiB 模型 + 运行时缓冲的实际余量。
5. ROCm 在 Windows 上对 gfx1151 的**性能**表现（有社区报告称该架构上 Vulkan 后端的 decode 性能优于 HIP，但那是 llama.cpp 的数据，与 DS4 的 kernel 无关）。

---

## 分阶段交付建议

**M1 — `ds4.exe` 单机推理（最小可用）**

范围：`ds4_cli.o` + `ds4_help.o` + `linenoise.o` + `ds4_gpu_args.o` + `ds4.o` + `ds4_distributed.o` + `ds4_tp.o` + `ds4_ssd.o` + ROCm 三个 TU + `ds4_layer_pack.o`。
不含 server / agent / web，因此**完全绕开第 11 项**。
工作项：P0、1、2、3、4、5、6、8（除 regex/fnmatch）。
先决条件：第 9 项的三个探针全绿。
预估：**2–3 周**（1 人），其中 shim 层本身约 1 周。

**M2 — `ds4-server.exe`**

增加 `ds4_server.c`（4 个错误点）、`ds4_kvstore.c`（2 个）、`rax.c`（0 个）。
增加 `fcntl(O_NONBLOCK)` → `ioctlsocket`、`opendir` 封装、SIGINT/SIGTERM 处理。
预估：**+1 周**。

**M3 — `ds4-agent.exe`**

第 11 项全部 + regex/fnmatch 引入。
预估：**+2–3 周**，且 shell 语义差异会持续产生行为差异 bug。

**总计 5–7 周**，前提是第 9、10 两项在真机上不出现阻断性问题。若 rocWMMA/hipBLASLt 在 Windows 上对 gfx1151 不可用，项目直接不成立。

---

## 建议

按成本-收益排序，建议的执行顺序是：

1. **先花半天跑第 9 项的三个探针**。这是唯一能决定项目生死的信息，成本却最低。
2. 探针通过后，**先做 M1 并且只用 `make test-mxfp4-rocm` 做门禁**——它不需要 80 GiB 模型，能在不触碰第 10 项风险的前提下验证整条 kernel 链路。
3. 只有 `test-mxfp4-rocm` 在 Windows 上通过后，再去处理第 10 项的内存配置和分层策略。

如果目标只是"在这台 Strix Halo 上跑 DS4"，装 Ubuntu 26.04 走 `STRIXHALO.md` 仍然是唯一被上游验证过的路径，成本是本方案的百分之一。本文档的价值在于：**如果确实需要原生 Windows，代码侧的障碍比预期小得多，真正的不确定性在 AMD 的 Windows ROCm 栈。**

---

## 附录 A：本次调查使用的 PoC shim

以下 shim 使 `ds4.c` 与 `ds4_ssd.c` 在 MinGW/Windows 目标下编译为 0 error。仅用于测量，不是生产实现（生产版需按 P0 改为 MSVC-ABI）。

`shim/sys/mman.h`：

```c
#ifndef DS4_WIN_MMAN_H
#define DS4_WIN_MMAN_H
#include <stddef.h>
#define PROT_READ   0x1
#define PROT_WRITE  0x2
#define MAP_SHARED  0x01
#define MAP_PRIVATE 0x02
#define MAP_ANON    0x20
#define MAP_ANONYMOUS MAP_ANON
#define MAP_FAILED  ((void *)-1)
void *ds4_win_mmap(void *addr, size_t len, int prot, int flags, int fd, long long off);
int   ds4_win_munmap(void *addr, size_t len);
int   ds4_win_mlock(const void *addr, size_t len);
int   ds4_win_munlock(const void *addr, size_t len);
#define mmap(a,l,p,f,fd,o)  ds4_win_mmap((a),(l),(p),(f),(fd),(o))
#define munmap(a,l)         ds4_win_munmap((a),(l))
#define mlock(a,l)          ds4_win_mlock((a),(l))
#define munlock(a,l)        ds4_win_munlock((a),(l))
#endif
```

`shim/sys/file.h`：

```c
#ifndef DS4_WIN_FILE_H
#define DS4_WIN_FILE_H
#define LOCK_SH 1
#define LOCK_EX 2
#define LOCK_NB 4
#define LOCK_UN 8
int ds4_win_flock(int fd, int op);
#define flock(fd,op) ds4_win_flock((fd),(op))
#ifndef F_SETFD
#define F_SETFD 2
#endif
#ifndef FD_CLOEXEC
#define FD_CLOEXEC 1
#endif
int ds4_win_fcntl(int fd, int cmd, ...);
#define fcntl ds4_win_fcntl
#endif
```

`shim/ds4_win_sysconf.h`：

```c
#ifndef DS4_WIN_SYSCONF_H
#define DS4_WIN_SYSCONF_H
#define _SC_PAGESIZE          1
#define _SC_NPROCESSORS_ONLN  2
long ds4_win_sysconf(int name);
#define sysconf(n) ds4_win_sysconf((n))
#endif
```

`shim/ds4_win_net.h`（由 `sys/socket.h`、`netinet/in.h`、`netinet/tcp.h`、`arpa/inet.h`、`netdb.h`、`poll.h`、`net/if.h`、`sys/uio.h` 各自 `#include` 进来）：

```c
#ifndef DS4_WIN_NET_H
#define DS4_WIN_NET_H
#ifndef WIN32_LEAN_AND_MEAN
#define WIN32_LEAN_AND_MEAN
#endif
#include <winsock2.h>
#include <ws2tcpip.h>
#include <mswsock.h>
struct iovec { void *iov_base; size_t iov_len; };
long long ds4_win_writev(int fd, const struct iovec *v, int n);
long long ds4_win_readv(int fd, const struct iovec *v, int n);
#define writev(f,v,n) ds4_win_writev((f),(v),(n))
#define readv(f,v,n)  ds4_win_readv((f),(v),(n))
#ifndef SHUT_RDWR
#define SHUT_RD   SD_RECEIVE
#define SHUT_WR   SD_SEND
#define SHUT_RDWR SD_BOTH
#endif
#define pollfd  WSAPOLLFD
#define poll(f,n,t) WSAPoll((f),(n),(t))
#ifndef SIGPIPE
#define SIGPIPE 13
#endif
#endif
```

## 附录 B：实测数据汇总

用空 stub 头（仅顶掉头文件，不提供任何符号）编译，各文件的真实符号缺口：

| 文件 | 行数 | 错误数 | 主要缺口 |
|------|------|--------|---------|
| `ds4.c` | 67,435 | **11** | mmap 常量 ×4、`_SC_*` ×3、`F_SETFD`/`FD_CLOEXEC`/`LOCK_EX`/`LOCK_NB` |
| `ds4_cli.c` | 2,204 | 2 | `struct sigaction` |
| `ds4_help.c` | 589 | **0** | — |
| `linenoise.c` | 2,690 | 45 | termios/ioctl 常量（集中在 3 个函数） |
| `ds4_gpu_args.c` | 230 | **0** | — |
| `ds4_distributed.c` | 8,437 | 81 → **0**（Winsock shim 后） | 全部为 socket 常量/类型 |
| `ds4_tp.c` | 2,219 | 30 → **0**（+`struct iovec`） | socket 常量 + iovec |
| `ds4_ssd.c` | 210 | 6 → **0**（mman shim 后） | mmap 常量 |
| `ds4_layer_pack.c` | 149 | **0** | — |
| `ds4_server.c` | 18,451 | 4 | `sigaction`、`F_GETFL`/`F_SETFL`/`O_NONBLOCK` |
| `ds4_web.c` | 1,385 | 7 | `mkdir` ×2、`WNOHANG` ×2、`fcntl` 组 |
| `ds4_bench.c` | 822 | **0** | — |
| `ds4_eval.c` | 4,312 | 1 | `sys/ioctl.h` |
| `ds4_agent.c` | 11,431 | — | `regex.h`、`fnmatch.h`（第三方依赖） |
| `ds4_kvstore.c` | 1,346 | 2 | `mkdir` 参数数 |
| `rax.c` | 2,747 | **0** | — |

`ds4.c` 在 `-Wall -Wextra` 下的全部 6 条警告：`getpagesize`、`pread`、`dprintf`、`fmemopen`（隐式声明 ×4）+ `fmemopen` 返回值转换 ×2。**无格式化字符串警告，无整型宽度警告。**

## 参考

- ROCm/rocm-systems#1786 — Large hip allocations bigger than 64GiB fail on Windows but pass on Linux
- ROCm/ROCm#5339 — Confusing rocm support for gfx1151
- ROCm/ROCm#6294 — Official Windows ONNX Runtime / ROCm support for Ryzen AI MAX+ 395
- AMD HIP SDK for Windows 7.2.0 — System requirements / Component support
- ggml-org/llama.cpp discussion #20856 — Known-Good Strix Halo ROCm Stack（Linux only）
- 本仓库：`STRIXHALO.md`、`QA_BEFORE_RELEASES.md` 第 9 节、`README.md:185-188`
