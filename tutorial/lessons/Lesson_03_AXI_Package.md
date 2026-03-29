# 第 3 课：AXI Package — 类型、常量与辅助函数

> **课程系列**：AXI 总线协议 30 课深度教程
> **所属单元**：单元一 — 协议基础与类型系统
> **前置知识**：第 1-2 课（协议概述、握手机制、Burst 类型与响应编码）
> **学习时长**：约 2-3 小时
> **对应源码**：`src/axi_pkg.sv`（全文，约 543 行）

---

## 学习目标

完成本课学习后，你将能够：

1. 完整描述 `axi_pkg.sv` 的代码结构和各部分功能
2. 解释 Cache 属性的四个位域及其对应的内存类型
3. 说明 ATOP（原子操作）的完整编码方案
4. 使用通道宽度计算函数推导任意参数下的信号位宽
5. 配置 `xbar_cfg_t` 结构体来描述一个交叉开关互连
6. 解释 `xbar_latency_e` 各模式对 Crossbar 性能和面积的影响

---

## 目录

- [3.1 axi_pkg.sv 文件结构总览](#31-axi_pkgsv-文件结构总览)
- [3.2 Cache 属性与内存类型](#32-cache-属性与内存类型)
- [3.3 Protection 类型](#33-protection-类型)
- [3.4 原子操作（ATOP）编码](#34-原子操作atop编码)
- [3.5 通道宽度计算函数](#35-通道宽度计算函数)
- [3.6 Crossbar 配置结构体](#36-crossbar-配置结构体)
- [3.7 延迟模式与 Spill Register](#37-延迟模式与-spill-register)
- [3.8 地址规则类型](#38-地址规则类型)
- [3.9 工具函数 iomsb](#39-工具函数-iomsb)
- [3.10 源码映射：axi_pkg.sv 完整索引](#310-源码映射axi_pkgsv-完整索引)
- [3.11 关键术语表](#311-关键术语表)
- [3.12 课后练习](#312-课后练习)
- [3.13 下一课预告](#313-下一课预告)

---

## 3.1 axi_pkg.sv 文件结构总览

`src/axi_pkg.sv` 是整个仓库的**基石文件**，所有模块都依赖它（Level 0）。本文件定义了 AXI 协议中所有不依赖于具体参数化宽度的类型、常量和函数。

### 3.1.1 文件结构地图

```
src/axi_pkg.sv (约 543 行)
│
├── [25-45]   信号位宽参数 .............. 第 1 课已讲
├── [47-66]   基础类型定义 .............. 第 1 课已讲
├── [68-87]   Burst 类型常量 ............ 第 2 课已讲
├── [89-100]  响应类型常量 .............. 第 2 课已讲
│
├── [102-113] Cache 属性常量 ............ ★ 本课重点
├── [115-118] num_bytes() 函数 .......... 第 2 课已讲
├── [120-123] largest_addr_t 类型 ....... 第 2 课已讲
├── [125-216] 地址计算函数族 ............ 第 2 课已讲
├── [218-226] Cache 查询函数 ............ ★ 本课重点
├── [228-280] 内存类型枚举与转换 ........ ★ 本课重点
├── [282-319] 响应优先级函数 ............ 第 2 课已讲
│
├── [321-378] 通道宽度计算函数 .......... ★ 本课重点
├── [380-447] ATOP 原子操作常量 ......... ★ 本课重点
├── [449-479] Crossbar 延迟模式 ......... ★ 本课重点
├── [481-522] xbar_cfg_t 配置结构体 ..... ★ 本课重点
├── [524-536] 地址规则结构体 ............ ★ 本课重点
└── [538-541] iomsb 工具函数 ............ ★ 本课重点
```

> 本课将逐一讲解标记为 ★ 的部分。前两课已覆盖的内容仅做简要回顾。

---

## 3.2 Cache 属性与内存类型

AXI 的 `AxCACHE` 信号（4 位）描述了事务的内存属性，决定了互连和缓存对事务的处理方式。

### 3.2.1 四个属性位

```systemverilog
// src/axi_pkg.sv, 第 102-113 行
localparam CACHE_BUFFERABLE = 4'b0001;  // bit 0
localparam CACHE_MODIFIABLE = 4'b0010;  // bit 1
localparam CACHE_RD_ALLOC   = 4'b0100;  // bit 2
localparam CACHE_WR_ALLOC   = 4'b1000;  // bit 3
```

```
AxCACHE[3:0] 位域：
  ┌─────────────┬─────────────┬─────────────┬─────────────┐
  │   bit 3     │   bit 2     │   bit 1     │   bit 0     │
  │ WR_ALLOC    │ RD_ALLOC    │ MODIFIABLE  │ BUFFERABLE  │
  │ 写分配      │ 读分配       │ 可修改       │ 可缓冲      │
  └─────────────┴─────────────┴─────────────┴─────────────┘
```

| 位 | 名称 | 含义 |
|----|------|------|
| bit 0 | **Bufferable** | 互连可以延迟事务到达最终目的地（数据可以暂存在中间缓冲区） |
| bit 1 | **Modifiable** | 互连可以修改事务特征（如合并写入、拆分 burst、改变 size） |
| bit 2 | **Read-Allocate** | 建议（但不强制）在缓存中为读操作分配缓存行 |
| bit 3 | **Write-Allocate** | 建议（但不强制）在缓存中为写操作分配缓存行 |

### 3.2.2 Cache 查询函数

```systemverilog
// src/axi_pkg.sv, 第 218-226 行
function automatic logic bufferable(cache_t cache);
    return |(cache & CACHE_BUFFERABLE);    // 检查 bit 0
endfunction

function automatic logic modifiable(cache_t cache);
    return |(cache & CACHE_MODIFIABLE);    // 检查 bit 1
endfunction
```

这两个函数在互连模块中频繁使用，例如 `axi_xbar` 需要判断事务是否可修改来决定是否允许 burst 合并。

### 3.2.3 内存类型枚举

`axi_pkg` 将常见的 cache 属性组合抽象为一个枚举类型 `mem_type_t`：

```systemverilog
// src/axi_pkg.sv, 第 228-242 行
typedef enum logic [3:0] {
    DEVICE_NONBUFFERABLE,                // 0: 设备型，不可缓冲
    DEVICE_BUFFERABLE,                   // 1: 设备型，可缓冲
    NORMAL_NONCACHEABLE_NONBUFFERABLE,   // 2: 普通型，不可缓存，不可缓冲
    NORMAL_NONCACHEABLE_BUFFERABLE,      // 3: 普通型，不可缓存，可缓冲
    WTHRU_NOALLOCATE,                    // 4: 写透(Write-Through)，不分配
    WTHRU_RALLOCATE,                     // 5: 写透，读分配
    WTHRU_WALLOCATE,                     // 6: 写透，写分配
    WTHRU_RWALLOCATE,                    // 7: 写透，读写分配
    WBACK_NOALLOCATE,                    // 8: 写回(Write-Back)，不分配
    WBACK_RALLOCATE,                     // 9: 写回，读分配
    WBACK_WALLOCATE,                     // 10: 写回，写分配
    WBACK_RWALLOCATE                     // 11: 写回，读写分配
} mem_type_t;
```

### 3.2.4 内存类型层级

```
内存类型层级：
┌─────────────────────────────────────────────────────────┐
│                    所有内存类型                           │
├───────────────────────┬─────────────────────────────────┤
│  Device（设备型）      │  Normal（普通型）                 │
│  - 不可缓存           │  ┌───────────────┬──────────────┤
│  - 严格顺序           │  │ Non-cacheable │  Cacheable   │
│  - 不可修改(MMIO)     │  │ 不可缓存       │ ┌───────────┤
│                       │  │               │ │Write-Thru │
│  Non-bufferable (0)   │  │               │ │Write-Back │
│  Bufferable     (1)   │  │ Non-buf (2)   │ │           │
│                       │  │ Buf     (3)   │ │ +Allocate │
└───────────────────────┴──┴───────────────┴─┴───────────┘
```

- **Device 型**：用于外设寄存器（UART, GPIO），事务不可重排序、不可合并
- **Normal Non-cacheable**：用于共享内存、DMA 缓冲区，不经过缓存
- **Write-Through**：写入同时更新缓存和内存
- **Write-Back**：写入只更新缓存，稍后回写内存（性能最好）

### 3.2.5 AxCACHE 转换函数

`mem_type_t` 到实际 `AxCACHE` 值的转换分读写两个方向，因为**同一内存类型的读写 cache 属性可以不同**：

```systemverilog
// src/axi_pkg.sv, 第 244-261 行 (ARCACHE - 读方向)
function automatic logic [3:0] get_arcache(mem_type_t mtype);
    unique case (mtype)
        DEVICE_NONBUFFERABLE              : return 4'b0000;
        DEVICE_BUFFERABLE                 : return 4'b0001;
        NORMAL_NONCACHEABLE_NONBUFFERABLE : return 4'b0010;
        NORMAL_NONCACHEABLE_BUFFERABLE    : return 4'b0011;
        WTHRU_NOALLOCATE                  : return 4'b1010;
        WTHRU_RALLOCATE                   : return 4'b1110;
        WTHRU_WALLOCATE                   : return 4'b1010;  // 读不需要写分配
        WTHRU_RWALLOCATE                  : return 4'b1110;
        WBACK_NOALLOCATE                  : return 4'b1011;
        WBACK_RALLOCATE                   : return 4'b1111;
        WBACK_WALLOCATE                   : return 4'b1011;  // 读不需要写分配
        WBACK_RWALLOCATE                  : return 4'b1111;
        default                           : return 4'bxxxx;
    endcase
endfunction
```

```systemverilog
// src/axi_pkg.sv, 第 263-280 行 (AWCACHE - 写方向)
function automatic logic [3:0] get_awcache(mem_type_t mtype);
    unique case (mtype)
        DEVICE_NONBUFFERABLE              : return 4'b0000;
        DEVICE_BUFFERABLE                 : return 4'b0001;
        NORMAL_NONCACHEABLE_NONBUFFERABLE : return 4'b0010;
        NORMAL_NONCACHEABLE_BUFFERABLE    : return 4'b0011;
        WTHRU_NOALLOCATE                  : return 4'b0110;  // ← 不同于 ARCACHE
        WTHRU_RALLOCATE                   : return 4'b0110;
        WTHRU_WALLOCATE                   : return 4'b1110;
        WTHRU_RWALLOCATE                  : return 4'b1110;
        WBACK_NOALLOCATE                  : return 4'b0111;  // ← 不同于 ARCACHE
        WBACK_RALLOCATE                   : return 4'b0111;
        WBACK_WALLOCATE                   : return 4'b1111;
        WBACK_RWALLOCATE                  : return 4'b1111;
        default                           : return 4'bxxxx;
    endcase
endfunction
```

> **为什么读写的 cache 值不同？** 以 `WTHRU_NOALLOCATE` 为例：读方向返回 `4'b1010`（Modifiable + Read-Allocate），写方向返回 `4'b0110`（Modifiable + Write-Through 标记）。这是因为 AXI 规范中 ARCACHE 和 AWCACHE 的位域含义在 Cacheable 类型下略有不同。

---

## 3.3 Protection 类型

`AxPROT` 信号（3 位）定义了事务的保护级别：

```
AxPROT[2:0] 位域：
  ┌──────────────┬──────────────┬──────────────┐
  │    bit 2     │    bit 1     │    bit 0     │
  │ Instruction  │  Non-secure  │ Privileged   │
  │ /Data        │  /Secure     │ /Unprivileged│
  └──────────────┴──────────────┴──────────────┘
```

| 位 | 值=0 | 值=1 | 说明 |
|----|------|------|------|
| bit 0 | Unprivileged | Privileged | 特权级别（用户态 vs 内核态） |
| bit 1 | Secure | Non-secure | 安全状态（TrustZone 相关） |
| bit 2 | Data | Instruction | 访问类型（数据访问 vs 指令取指） |

`axi_pkg` 中定义了 `prot_t` 类型（第 54 行）：

```systemverilog
typedef logic [2:0] prot_t;
```

> **实际使用**：在 SoC 中，TrustZone 控制器会检查 AxPROT[1] 来决定是否允许非安全世界访问安全内存。处理器在取指令时设置 AxPROT[2]=1，在读写数据时设置 AxPROT[2]=0。

---

## 3.4 原子操作（ATOP）编码

AXI5 引入了原子操作（Atomic Operations），允许在内存侧执行"读-修改-写"操作，无需将数据传回处理器。这对多处理器同步至关重要。

### 3.4.1 ATOP 字段结构

`awatop` 信号共 6 位，编码结构如下：

```
awatop[5:0] 编码：
  ┌─────┬─────┬──────┬──────┬──────┬──────┐
  │ [5] │ [4] │  [3] │  [2] │  [1] │  [0] │
  │ R_RESP    │ Endian│      Operation     │
  │ 操作类型   │大小端 │     具体操作        │
  └─────┴─────┴──────┴──────┴──────┴──────┘
```

### 3.4.2 操作类型 — ATOP[5:4]

```systemverilog
// src/axi_pkg.sv, 第 399-415 行
localparam ATOP_NONE        = 2'b00;  // 非原子操作
localparam ATOP_ATOMICSTORE = 2'b01;  // 原子存储（无返回数据）
localparam ATOP_ATOMICLOAD  = 2'b10;  // 原子加载（返回旧值）
```

以及两个完整的 6 位操作码：

```systemverilog
// src/axi_pkg.sv, 第 387-397 行
localparam ATOP_ATOMICSWAP = 6'b110000;  // 原子交换
localparam ATOP_ATOMICCMP  = 6'b110001;  // 原子比较并交换（CAS）
```

### 3.4.3 四种原子操作对比

```
┌─────────────────┬────────┬──────────┬──────────┬──────────────┐
│ 操作类型         │ ATOP值 │ 发送数据  │ 返回数据  │ 说明          │
├─────────────────┼────────┼──────────┼──────────┼──────────────┤
│ AtomicStore     │ 01xxxx │ 1个值    │ 无       │ mem = op(mem,│
│                 │        │          │          │        data) │
├─────────────────┼────────┼──────────┼──────────┼──────────────┤
│ AtomicLoad      │ 10xxxx │ 1个值    │ 旧值     │ ret = mem    │
│                 │        │          │          │ mem = op(mem,│
│                 │        │          │          │        data) │
├─────────────────┼────────┼──────────┼──────────┼──────────────┤
│ AtomicSwap      │ 110000 │ 1个值    │ 旧值     │ ret = mem    │
│                 │        │          │          │ mem = data   │
├─────────────────┼────────┼──────────┼──────────┼──────────────┤
│ AtomicCompare   │ 110001 │ 2个值    │ 旧值     │ if mem==cmp: │
│                 │        │ (cmp+swp)│          │   mem = swp  │
└─────────────────┴────────┴──────────┴──────────┴──────────────┘
```

### 3.4.4 算术/逻辑操作 — ATOP[2:0]

当操作类型为 AtomicStore 或 AtomicLoad 时，低 3 位指定具体的运算：

```systemverilog
// src/axi_pkg.sv, 第 425-444 行
localparam ATOP_ADD  = 3'b000;  // mem = mem + data
localparam ATOP_CLR  = 3'b001;  // mem = mem & ~data  （清除位）
localparam ATOP_EOR  = 3'b010;  // mem = mem ^ data   （异或）
localparam ATOP_SET  = 3'b011;  // mem = mem | data    （置位）
localparam ATOP_SMAX = 3'b100;  // mem = max(mem, data)（有符号）
localparam ATOP_SMIN = 3'b101;  // mem = min(mem, data)（有符号）
localparam ATOP_UMAX = 3'b110;  // mem = max(mem, data)（无符号）
localparam ATOP_UMIN = 3'b111;  // mem = min(mem, data)（无符号）
```

### 3.4.5 大小端 — ATOP[3]

```systemverilog
// src/axi_pkg.sv, 第 420-423 行
localparam ATOP_LITTLE_END = 1'b0;  // 小端（仅对算术操作有效）
localparam ATOP_BIG_END    = 1'b1;  // 大端（仅对算术操作有效）
```

### 3.4.6 读响应标志 — ATOP[5]

```systemverilog
// src/axi_pkg.sv, 第 446-447 行
localparam ATOP_R_RESP = 32'd5;
// 用法: if (req_i.aw.atop[axi_pkg::ATOP_R_RESP]) begin
//         // 此原子操作需要通过 R 通道返回数据
//       end
```

当 ATOP[5]=1 时，原子操作会通过 **R 通道**返回旧值数据。这意味着一个写事务（通过 AW+W 发起）会同时产生 B 响应和 R 响应。

### 3.4.7 ATOP 编码完整示例

```
示例: 原子加法（AtomicLoad + ADD + Little-Endian）
  awatop = {2'b10, 1'b0, 3'b000} = 6'b100000
             │      │     │
             │      │     └─ ADD 操作
             │      └─────── Little-Endian
             └────────────── AtomicLoad（返回旧值）

示例: 原子比较并交换（AtomicCompare）
  awatop = 6'b110001 = ATOP_ATOMICCMP
```

> **在仓库中的应用**：`axi_atop_filter`（第 27 课）会过滤掉不支持原子操作的从机端口上的 ATOP 请求，将其转换为错误响应。

---

## 3.5 通道宽度计算函数

`axi_pkg` 提供了一组函数来计算各通道的总位宽。这些函数在模块参数化中非常有用。

### 3.5.1 单通道宽度

```systemverilog
// src/axi_pkg.sv, 第 321-356 行

// AW 通道宽度
function automatic int unsigned aw_width(int unsigned addr_width,
    int unsigned id_width, int unsigned user_width);
    return (id_width + addr_width + LenWidth + SizeWidth + BurstWidth +
            LockWidth + CacheWidth + ProtWidth + QosWidth + RegionWidth +
            AtopWidth + user_width);
    // = id + addr + 8 + 3 + 2 + 1 + 4 + 3 + 4 + 4 + 6 + user
    // = id + addr + user + 35
endfunction

// W 通道宽度
function automatic int unsigned w_width(int unsigned data_width,
    int unsigned user_width);
    return (data_width + data_width/8 + 1 + user_width);
    //      ^data       ^strobe        ^last
endfunction

// B 通道宽度
function automatic int unsigned b_width(int unsigned id_width,
    int unsigned user_width);
    return (id_width + RespWidth + user_width);
    // = id + 2 + user
endfunction

// AR 通道宽度 (注意：没有 AtopWidth)
function automatic int unsigned ar_width(int unsigned addr_width,
    int unsigned id_width, int unsigned user_width);
    return (id_width + addr_width + LenWidth + SizeWidth + BurstWidth +
            LockWidth + CacheWidth + ProtWidth + QosWidth + RegionWidth +
            user_width);
    // = id + addr + user + 29  (比 AW 少 6 位 atop)
endfunction

// R 通道宽度
function automatic int unsigned r_width(int unsigned data_width,
    int unsigned id_width, int unsigned user_width);
    return (id_width + data_width + RespWidth + 1 + user_width);
    //                              ^resp       ^last
endfunction
```

### 3.5.2 计算示例

以常见配置为例：AddrWidth=32, DataWidth=64, IdWidth=4, UserWidth=0

```
AW 宽度 = 4 + 32 + 8 + 3 + 2 + 1 + 4 + 3 + 4 + 4 + 6 + 0 = 71 位
W  宽度 = 64 + 64/8 + 1 + 0 = 64 + 8 + 1 = 73 位
B  宽度 = 4 + 2 + 0 = 6 位
AR 宽度 = 4 + 32 + 8 + 3 + 2 + 1 + 4 + 3 + 4 + 4 + 0 = 65 位
R  宽度 = 4 + 64 + 2 + 1 + 0 = 71 位
```

### 3.5.3 聚合宽度

```systemverilog
// src/axi_pkg.sv, 第 358-378 行

// 请求通道总宽度 (AW + W + AR + 握手信号)
function automatic int unsigned req_width(...);
    return (aw_width(...) + 1 +      // +1 for aw_valid
            w_width(...)  + 1 +      // +1 for w_valid
            ar_width(...) + 1 +      // +1 for ar_valid
            1 + 1);                  // r_ready + b_ready
endfunction

// 响应通道总宽度 (R + B + 握手信号)
function automatic int unsigned rsp_width(...);
    return (r_width(...) + 1 +       // +1 for r_valid
            b_width(...) + 1 +       // +1 for b_valid
            1 + 1 + 1);             // aw_ready + ar_ready + w_ready
endfunction
```

> **用途**：这些函数在 CDC（跨时钟域）模块和 flat port 变体中被使用，用于计算序列化后的总线宽度。

---

## 3.6 Crossbar 配置结构体

`xbar_cfg_t` 是配置 `axi_xbar`（交叉开关）的核心数据结构，将在第 16 课详细使用。这里先建立完整理解。

### 3.6.1 结构体定义

```systemverilog
// src/axi_pkg.sv, 第 481-522 行
typedef struct packed {
    int unsigned NoSlvPorts;         // 从机端口数（接 Master 的数量）
    int unsigned NoMstPorts;         // 主机端口数（接 Slave 的数量）
    int unsigned MaxMstTrans;        // 每个主机端口的最大在途事务数
    int unsigned MaxSlvTrans;        // 每个从机端口的最大在途事务数
    bit          FallThrough;        // FIFO 直通模式（0=否, 1=是）
    bit [9:0]    LatencyMode;        // 延迟模式（见 3.7 节）
    int unsigned PipelineStages;     // 额外流水线级数
    int unsigned AxiIdWidthSlvPorts; // 从机端口 ID 宽度
    int unsigned AxiIdUsedSlvPorts;  // 用于区分路由的 ID 位数
    bit          UniqueIds;          // ID 是否保证唯一
    int unsigned AxiAddrWidth;       // 地址宽度
    int unsigned AxiDataWidth;       // 数据宽度
    int unsigned NoAddrRules;        // 地址映射规则数量
} xbar_cfg_t;
```

### 3.6.2 命名约定注意事项

本仓库中的 **"Slave Port"** 和 **"Master Port"** 指的是**交叉开关自身的端口角色**，而不是连接到它的设备角色：

```
┌──────┐    Slave Port     ┌─────────┐    Master Port     ┌──────┐
│Master│◄═════════════════►│ axi_xbar│◄══════════════════►│Slave │
│(CPU) │  (xbar 做从机)    │         │  (xbar 做主机)     │(SRAM)│
└──────┘                   └─────────┘                    └──────┘

NoSlvPorts = 连接 Master 设备的端口数
NoMstPorts = 连接 Slave 设备的端口数
```

### 3.6.3 关键参数详解

#### MaxMstTrans / MaxSlvTrans

控制每个端口允许的最大在途事务数。增大此值可以提高性能（允许更多并发事务），但会增加面积（需要更多计数器和 ID 追踪逻辑）。

```
MaxMstTrans = 1:  每次只能有 1 个未完成事务（最小面积，最低性能）
MaxMstTrans = 8:  允许 8 个并发事务（典型值）
MaxMstTrans = 32: 高性能配置（大面积开销）
```

#### AxiIdWidthSlvPorts / AxiIdUsedSlvPorts

- `AxiIdWidthSlvPorts`：从机端口的 ID 位宽（即 Master 发出的 ID 宽度）
- `AxiIdUsedSlvPorts`：Demux 实际用于路由查找的 ID 位数

当 `AxiIdUsedSlvPorts < AxiIdWidthSlvPorts` 时，只有低位参与路由判断，可以节省面积。

#### UniqueIds

如果能保证所有 Master 不会同时使用相同的 ID，设为 1 可以优化 Demux 内部的 ID 计数器逻辑。

### 3.6.4 配置示例

```systemverilog
// 典型 SoC 互连配置：2 个 Master（CPU + DMA），3 个 Slave（SRAM + UART + GPIO）
localparam axi_pkg::xbar_cfg_t XbarCfg = '{
    NoSlvPorts:         2,           // 2 个 Master 连接
    NoMstPorts:         3,           // 3 个 Slave 连接
    MaxMstTrans:        8,           // 每个 Slave 端口最多 8 个在途事务
    MaxSlvTrans:        8,           // 每个 Master 端口最多 8 个在途事务
    FallThrough:        1'b0,        // 不使用 FIFO 直通
    LatencyMode:        axi_pkg::CUT_ALL_AX,  // 地址通道插入寄存器
    PipelineStages:     0,           // 无额外流水线
    AxiIdWidthSlvPorts: 4,           // 4 位 ID
    AxiIdUsedSlvPorts:  4,           // 全部 4 位用于路由
    UniqueIds:          1'b0,        // ID 不保证唯一
    AxiAddrWidth:       32,          // 32 位地址
    AxiDataWidth:       64,          // 64 位数据
    NoAddrRules:        3            // 3 条地址映射规则
};
```

---

## 3.7 延迟模式与 Spill Register

Crossbar 的延迟模式决定了在哪些位置插入 spill register（寄存器切割），影响时序和延迟。

### 3.7.1 通道切片位定义

每个位控制一个通道是否插入 spill register：

```systemverilog
// src/axi_pkg.sv, 第 449-469 行
localparam bit [9:0] DemuxAw = (1 << 9);  // Demux 侧 AW
localparam bit [9:0] DemuxW  = (1 << 8);  // Demux 侧 W
localparam bit [9:0] DemuxB  = (1 << 7);  // Demux 侧 B
localparam bit [9:0] DemuxAr = (1 << 6);  // Demux 侧 AR
localparam bit [9:0] DemuxR  = (1 << 5);  // Demux 侧 R
localparam bit [9:0] MuxAw   = (1 << 4);  // Mux 侧 AW
localparam bit [9:0] MuxW    = (1 << 3);  // Mux 侧 W
localparam bit [9:0] MuxB    = (1 << 2);  // Mux 侧 B
localparam bit [9:0] MuxAr   = (1 << 1);  // Mux 侧 AR
localparam bit [9:0] MuxR    = (1 << 0);  // Mux 侧 R
```

### 3.7.2 预定义延迟模式

```systemverilog
// src/axi_pkg.sv, 第 471-479 行
typedef enum bit [9:0] {
    NO_LATENCY    = 10'b000_00_000_00,  // 无寄存器，最低延迟，最长路径
    CUT_SLV_AX    = DemuxAw | DemuxAr,  // 仅切割从机侧地址通道
    CUT_MST_AX    = MuxAw | MuxAr,      // 仅切割主机侧地址通道
    CUT_ALL_AX    = DemuxAw | DemuxAr | MuxAw | MuxAr,  // 所有地址通道
    CUT_SLV_PORTS = DemuxAw | DemuxW | DemuxB | DemuxAr | DemuxR,  // 从机侧全通道
    CUT_MST_PORTS = MuxAw | MuxW | MuxB | MuxAr | MuxR,            // 主机侧全通道
    CUT_ALL_PORTS = 10'b111_11_111_11   // 全部切割，最高频率，最大延迟
} xbar_latency_e;
```

### 3.7.3 模式选择指南

```
性能优先 ◄──────────────────────────────────────────► 时序优先

NO_LATENCY    CUT_SLV_AX    CUT_ALL_AX    CUT_ALL_PORTS
   │              │              │              │
   │  0 级寄存器   │  2 级寄存器   │  4 级寄存器   │  10 级寄存器
   │  最低延迟     │              │              │  最高频率
   │  最长关键路径 │              │              │  最大面积
   └──────────────┴──────────────┴──────────────┘
```

> **实际选择**：`CUT_ALL_AX`（切割所有地址通道）是最常用的配置，在时序和延迟之间取得良好平衡。只切割地址通道是因为地址通道通常是时序关键路径（需要经过地址译码逻辑）。

---

## 3.8 地址规则类型

`axi_xbar` 使用地址规则来决定事务的路由目标。`axi_pkg` 提供了两种预定义的规则结构体：

```systemverilog
// src/axi_pkg.sv, 第 524-536 行

// 64 位地址规则
typedef struct packed {
    int unsigned idx;          // 目标从机端口索引
    logic [63:0] start_addr;   // 起始地址（含）
    logic [63:0] end_addr;     // 结束地址（不含）
} xbar_rule_64_t;

// 32 位地址规则
typedef struct packed {
    int unsigned idx;          // 目标从机端口索引
    logic [31:0] start_addr;   // 起始地址（含）
    logic [31:0] end_addr;     // 结束地址（不含）
} xbar_rule_32_t;
```

### 使用示例

```systemverilog
// 为 3 个 Slave 定义地址映射
localparam axi_pkg::xbar_rule_32_t [2:0] AddrMap = '{
    '{idx: 0, start_addr: 32'h0000_0000, end_addr: 32'h0001_0000},  // SRAM: 0-64KB
    '{idx: 1, start_addr: 32'h4000_0000, end_addr: 32'h4000_1000},  // UART: 4KB
    '{idx: 2, start_addr: 32'h4001_0000, end_addr: 32'h4001_1000}   // GPIO: 4KB
};
// 不在以上范围内的地址 → DECERR
```

> **注意**：`end_addr` 是**不含的**（exclusive），即 Slave 0 的实际地址范围是 `[0x0000_0000, 0x0001_0000)`。

---

## 3.9 工具函数 iomsb

```systemverilog
// src/axi_pkg.sv, 第 538-541 行
function automatic integer unsigned iomsb(input integer unsigned width);
    return (width != 32'd0) ? unsigned'(width-1) : 32'd0;
endfunction
```

这是一个简单但实用的工具函数：给定位宽 `width`，返回最高位索引 `width-1`，当 `width=0` 时安全返回 0。

**典型用途**：

```systemverilog
// 声明向量时避免 width=0 导致的 [-1:0] 问题
logic [axi_pkg::iomsb(UserWidth):0] user_signal;
// 当 UserWidth=4 时: logic [3:0] user_signal;
// 当 UserWidth=0 时: logic [0:0] user_signal; (而不是 logic [-1:0])
```

---

## 3.10 源码映射：axi_pkg.sv 完整索引

| 行号 | 内容 | 所属课程 |
|------|------|----------|
| 25-45 | 信号位宽参数（BurstWidth, RespWidth 等） | 第 1 课 |
| 47-66 | 基础类型定义（burst_t, resp_t 等） | 第 1 课 |
| 68-87 | Burst 类型常量（FIXED, INCR, WRAP） | 第 2 课 |
| 89-100 | 响应类型常量（OKAY, EXOKAY, SLVERR, DECERR） | 第 2 课 |
| 102-113 | **Cache 属性常量** | **第 3 课** |
| 115-118 | num_bytes() 函数 | 第 2 课 |
| 120-123 | largest_addr_t 类型 | 第 2 课 |
| 125-128 | aligned_addr() 函数 | 第 2 课 |
| 130-163 | wrap_boundary() 函数 | 第 2 课 |
| 165-196 | beat_addr() 函数 | 第 2 课 |
| 198-216 | beat_lower_byte() / beat_upper_byte() | 第 2 课 |
| 218-226 | **bufferable() / modifiable() 函数** | **第 3 课** |
| 228-242 | **mem_type_t 枚举** | **第 3 课** |
| 244-280 | **get_arcache() / get_awcache() 函数** | **第 3 课** |
| 282-319 | resp_precedence() 函数 | 第 2 课 |
| 321-378 | **通道宽度计算函数族** | **第 3 课** |
| 380-447 | **ATOP 原子操作常量** | **第 3 课** |
| 449-479 | **延迟模式定义** | **第 3 课** |
| 481-522 | **xbar_cfg_t 结构体** | **第 3 课** |
| 524-536 | **地址规则结构体** | **第 3 课** |
| 538-541 | **iomsb() 工具函数** | **第 3 课** |

至此，`axi_pkg.sv` 的全部 543 行代码已在前三课中完整覆盖。

---

## 3.11 关键术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| 可缓冲 | Bufferable | 互连可以延迟事务到达目的地（数据可暂存） |
| 可修改 | Modifiable | 互连可以修改事务特征（合并、拆分等） |
| 读分配 | Read-Allocate | 建议缓存为读操作分配缓存行 |
| 写分配 | Write-Allocate | 建议缓存为写操作分配缓存行 |
| 写透 | Write-Through | 写入同时更新缓存和内存 |
| 写回 | Write-Back | 写入只更新缓存，延迟回写内存 |
| 原子操作 | Atomic Operation (ATOP) | 在内存侧执行读-修改-写的不可分割操作 |
| 比较并交换 | Compare-and-Swap (CAS) | 仅当内存值匹配时才写入的原子操作 |
| Spill Register | Spill Register | 用于打断组合路径的寄存器级（不增加气泡） |
| 延迟模式 | Latency Mode | 控制 Crossbar 中 spill register 放置位置的配置 |
| 地址规则 | Address Rule | 定义地址范围到从机端口的映射关系 |
| 直通模式 | Fall-Through | FIFO 数据入队后同一周期即可出队 |

---

## 3.12 课后练习

### 练习 1：Cache 属性解读（基础）

给定以下 `AxCACHE` 值，说明每个位的含义和对应的内存行为：

1. `4'b0000`
2. `4'b0011`
3. `4'b1111`

### 练习 2：ATOP 编码解析（基础）

解析以下 `awatop` 值代表的原子操作：

1. `6'b000000`
2. `6'b010011`
3. `6'b100110`
4. `6'b110000`

### 练习 3：通道宽度计算（中级）

给定参数：AddrWidth=48, DataWidth=128, IdWidth=8, UserWidth=2

计算：
1. AW 通道的总位宽
2. W 通道的总位宽
3. R 通道的总位宽
4. 完整 req_t struct 的总位宽（假设所有 user 宽度相同）

### 练习 4：Crossbar 配置设计（中级）

为以下 SoC 设计一个合适的 `xbar_cfg_t` 配置：

- 3 个 Master：CPU（ID 宽度 4）、DMA（ID 宽度 4）、调试接口（ID 宽度 2）
- 4 个 Slave：Boot ROM（64KB）、SRAM（256KB）、外设桥（4KB）、DDR 控制器（1GB）
- 目标频率 500MHz，要求时序收敛
- 地址宽度 32 位，数据宽度 64 位

### 练习 5：延迟模式分析（高级）

分析 `CUT_ALL_AX` 模式下，一个读事务从 Master 发出 AR 到收到第一个 R 数据的最小延迟（以时钟周期计）。假设 Slave 零延迟响应。

提示：画出信号经过的每个 spill register 级。

### 练习 6：源码验证（实践）

在 `axi_pkg.sv` 中：

1. 对比 `get_arcache(WBACK_RALLOCATE)` 和 `get_awcache(WBACK_RALLOCATE)` 的返回值，解释为什么不同。
2. 如果自定义一个延迟模式，只切割 Demux 侧的 AW 和 Mux 侧的 R，应该用什么 bit 组合？用已有的 `localparam` 表达出来。

---

## 3.13 下一课预告

**第 4 课：Typedef 宏 — 参数化类型生成**

在第 4 课中，我们将精读 `include/axi/typedef.svh`，涵盖：

- 为什么使用 struct 而不是散列信号（本仓库的核心设计约定）
- 单通道 Typedef 宏的展开过程
- `AXI_TYPEDEF_REQ_T` 和 `AXI_TYPEDEF_RESP_T` 聚合结构
- `AXI_TYPEDEF_ALL` 一键生成宏
- AXI4-Lite 等效宏及其简化
- 如何在自己的模块中使用这些宏

---

## 参考资料

1. **ARM AMBA AXI and ACE Protocol Specification, Issue F.b** — A4 节（Memory Attributes）、E1 节（Atomic Accesses）
2. `src/axi_pkg.sv` — 本课覆盖第 102-541 行
3. `doc/axi_xbar.md` — Crossbar 配置详细文档

---

> **版权声明**：本教材基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库编写，该仓库遵循 Solderpad Hardware License v0.51。
