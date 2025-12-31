# Hook函数原理

## Frida的hook函数的机制和原理

**Q**: Frida的hook钩子勾住函数的底层实现原理是什么？

**A**: Frida 作为一款强大的动态插桩工具，其核心能力在于通过 **Inline Hook** 或 **PLT/GOT Hook**（取决于目标平台和场景）技术，在运行时拦截并修改函数的执行逻辑。要理解其底层原理，需从**进程注入、代码注入、Hook 机制**三个层面展开：

### 一、前置：Frida 如何进入目标进程？

Frida 采用 **注入器（Injector）** 将自身 Agent（通常是 JavaScript 引擎 + Frida Core 库）加载到目标进程中：  

- 对于 Android/iOS 等移动平台，可能通过 `ptrace`（Linux/Android）、`debugserver`（iOS）或 zygote 注入等方式；  
- 注入后，Agent 会在目标进程中启动一个 JavaScript 运行时（如 V8），并通过 **Native 桥接层**（C/C++ 实现的 Frida Core）将 JS 层的 Hook 操作转化为底层的机器指令修改。

### 二、核心：函数 Hook 的底层实现原理
Frida 对函数的 Hook 主要基于两种经典技术：**Inline Hook**（针对任意函数，尤其是无符号表的函数）和 **PLT/GOT Hook**（针对动态链接库的外部函数调用）。

#### 1. Inline Hook（内联钩子）：直接修改函数体指令
这是 Frida 最常用、最灵活的 Hook 方式，适用于几乎所有可执行代码（包括静态函数、动态库函数、甚至自定义代码段）。

##### 原理步骤：

假设我们要 Hook 目标函数 `target_func`（地址为 `0x12345678`），替换为我们的 `hook_func`：  

- **Step 1：保存原函数的前 N 字节指令**  
  为了插入跳转指令，需要覆盖 `target_func` 开头的若干字节（比如 x86/x64 下覆盖 5~14 字节，对应 JMP 指令的长度）。这些被覆盖的原始字节必须保存下来，否则原函数无法恢复执行。  
- **Step 2：写入跳转指令到原函数开头**  
  在 `target_func` 的开头写入一条**无条件跳转指令**，跳转到我们的 `hook_func` 地址。例如：  
  - x86 下：`JMP hook_func`（机器码 `E9 XX XX XX XX`，相对跳转）；  
  - x64 下：若 `hook_func` 与 `target_func` 距离超过 2GB，需用 `MOV RAX, hook_func; JMP RAX`（机器码 `48 B8 XX XX XX XX XX XX XX XX; FF E0`）；  
  - ARM64 下：`LDR X16, =hook_func; BR X16`（机器码 `58 00 00 58 00 02 1F D6` 等）。  
- **Step 3：处理跳转后的执行流**  
  当程序调用 `target_func` 时，会先执行我们插入的 JMP 指令，跳转到 `hook_func`。在 `hook_func` 中，我们可以：  
  - 执行自定义逻辑（如打印参数、修改参数）；  
  - 通过之前保存的原始指令**模拟执行原函数的前半部分**（或直接跳回原函数被覆盖指令之后的位置继续执行，即“Trampoline 蹦床”）；  
  - 执行原函数的剩余逻辑（或直接返回）。  
- **Step 4：Trampoline（蹦床）机制**  
  为了无缝衔接原函数执行，Frida 会构造一个 **Trampoline 代码块**：  
  - 把 Step 1 保存的原函数前 N 字节复制到 Trampoline；  
  - 在 Trampoline 末尾添加一条跳转指令，跳回 `target_func` 中被覆盖指令之后的位置（即 `0x12345678 + N`）；  
  这样，当 `hook_func` 执行完自定义逻辑后，可以跳转到 Trampoline 来执行原函数的前半部分，再自动回到原函数后续逻辑。

#### 2. PLT/GOT Hook：修改动态链接的间接跳转表

仅适用于**动态链接的函数调用**（如 libc.so 中的 `open`、`read` 等函数），原理是利用动态链接器的 **Procedure Linkage Table（PLT）** 和 **Global Offset Table（GOT）** 机制。

##### 背景知识：

动态链接的程序调用外部函数时，不会直接写死函数地址，而是通过 PLT 中的桩代码跳转：  

- 第一次调用时，PLT 桩代码会查询 GOT 表中对应的条目（初始指向 PLT 自身的解析代码）；  
- 动态链接器会将 GOT 条目更新为**实际的函数地址**（懒加载）；  
- 后续调用直接从 GOT 条目取地址跳转。

##### 原理步骤：

- **Step 1：定位目标函数在 GOT 中的条目**  
  通过解析 ELF（Linux/Android）或 Mach-O（iOS/macOS）文件格式，找到目标函数（如 `libc.so!open`）对应的 GOT 表项地址。  
- **Step 2：修改 GOT 条目的值**  
  将 GOT 条目中原本指向 `open` 函数的地址，替换为我们的 `hook_open` 函数地址。  
- **Step 3：执行流劫持**  
  当程序调用 `open` 时，会通过 PLT → GOT 跳转到 `hook_open`；我们可以在 `hook_open` 中处理后，再手动调用原始的 `open` 函数（通过 Frida 保存的原始 GOT 值或再次查询动态链接器）。

### 三、Frida 如何支撑这些底层操作？

Frida 的 Native 层（C/C++）提供了以下关键组件：  

1. ** GumJS 引擎**：连接 JavaScript 层和 Native 层，将 JS 中的 `Interceptor.attach(target, callback)` 转化为底层的 Inline Hook/PLT Hook 调用；  
2. ** Gum 库**：Frida 的核心 Native 库，封装了跨平台的 Inline Hook 实现（如 `gum_module_find_export_by_name` 找函数、`gum_interceptor_attach` 设置 Hook）、内存读写（`gum_memory_read/write`）、线程挂起/恢复等功能；  
3. **Stalker（可选）**：用于跟踪函数调用流程（如记录所有函数调用栈），基于动态二进制翻译（DBI）技术，能更细粒度地控制执行流。

### 四、关键细节：跨平台与兼容性

Frida 需要处理不同架构（x86/x64/ARM/ARM64/MIPS）和系统的指令差异：  

- 针对不同架构设计对应的 JMP 指令模板；  
- 处理指令对齐问题（如某些架构要求指令地址按 4/16 字节对齐）；  
- 绕过系统保护（如 Linux 的 `W^X` 内存保护：Frida 可通过 `mprotect` 临时修改内存页的可写属性，写完跳转指令后再改回可执行）；  
- 对抗反调试/反 Hook（如检测内存是否被修改、指令是否被破坏，Frida 会动态调整 Hook 策略）。

### 五、总结

Frida 的 Hook 底层本质是**通过修改目标函数的执行指令（Inline Hook）或间接跳转表（PLT/GOT Hook），将函数调用重定向到自定义的 Hook 函数，再通过 Trampoline 机制保证原函数逻辑的正常执行**。其核心是利用操作系统提供的进程内存读写、指令修改能力，结合跨平台的 Hook 框架，实现对运行时函数行为的拦截与控制。

## 相关背景知识

### PLT和GOT

PLT/GOT 是 动态链接（Dynamic Linking）​ 机制中的两个关键数据结构

#### `PLT`=`Procedure Linkage Table`=`过程链接表`

- **全称**：Procedure Linkage Table（直译为“过程链接表”或“函数链接表”）
- **典型含义**：
  * 是**目标文件（ELF/PE/Mach-O 等）代码段中的一个跳转表**，用于在**编译时不确定外部函数地址**的场景下（如调用动态库中的函数），实现程序的“延迟绑定（Lazy Binding）”或“即时绑定（Immediate Binding）”。  
  * 简单来说，当程序调用一个动态库函数（如 `printf`）时，编译器不会直接生成调用 `printf` 的代码，而是生成调用 **PLT 中对应桩函数（Stub Function）** 的代码。PLT 桩函数的作用是**通过 GOT 间接跳转到实际的函数地址**（第一次调用时会触发动态链接器解析地址，后续直接使用已解析的地址）

#### `GOT`=`Global Offset Table`=`全局偏移表`

- **全称**：Global Offset Table（直译为“全局偏移表”）
- **典型含义**：
  * 是**数据段中的一个指针表**，存储了**动态链接函数的实际地址（或间接地址）**，以及全局变量/静态变量的地址。  
  * GOT 与 PLT 配合使用：  
    - 对于**外部函数**：GOT 中对应条目最初指向 PLT 的解析代码（用于第一次调用时让动态链接器填充真实地址）；之后会被动态链接器更新为**目标函数的实际内存地址**（懒加载完成后）。  
    - 对于**全局/静态变量**：GOT 条目直接存储变量的运行时地址（因为变量地址在加载时才确定）。  

#### PLT/GOT 的典型协作流程（以 ELF 动态链接为例）  

以调用 `libc.so` 中的 `open` 函数为例：  

1. **编译阶段**：编译器生成调用 `open@plt` 的指令（而非直接调用 `open`）——`open@plt` 是 PLT 中的桩函数。  
2. **第一次调用 `open@plt`**：  
  - PLT 桩函数会先跳转到 GOT 中 `open` 对应的条目（此时 GOT 条目指向 PLT 的解析逻辑，而非 `open` 本身）；  
  - 触发动态链接器（`ld-linux.so`）解析 `open` 的实际地址，并将该地址写入 GOT 中 `open` 的条目；  
3. **后续调用 `open@plt`**：  
  - PLT 桩函数直接跳转到 GOT 中已更新的 `open` 地址，无需再次解析，效率等同于直接调用。  


### 核心作用

PLT/GOT 解决了**动态链接的两个关键问题**：

- **地址不确定性**：动态库加载地址不固定（ASLR 机制下随机化），编译时无法写死函数地址；  
- **延迟绑定优化**：避免程序启动时一次性解析所有动态函数地址（减少启动时间），仅在函数首次调用时解析。  

### 扩展：不同文件格式的对应概念

- ELF（Linux/Android）：明确使用 PLT/GOT；  
- PE（Windows）：类似机制称为 **Import Address Table（IAT，导入地址表）** + **Thunk 函数**（对应 PLT）；  
- Mach-O（macOS/iOS）：类似机制称为 **Lazy Symbol Pointer Table（懒符号指针表）** + **Stub Helper（桩辅助代码）**（对应 PLT）。  

但在讨论 Frida 等工具的 Hook 时，通常默认指 ELF 下的 PLT/GOT 机制。
