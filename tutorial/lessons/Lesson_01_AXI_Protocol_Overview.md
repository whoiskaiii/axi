# 第 1 课：AXI 协议概述与五通道架构

> **课程系列**：AXI 总线协议 30 课深度教程
> **所属单元**：单元一 — 协议基础与类型系统
> **前置知识**：数字电路基础（时钟、寄存器、组合逻辑）、Verilog/SystemVerilog 基本语法
> **学习时长**：约 2-3 小时
> **对应源码**：`src/axi_pkg.sv`, `src/axi_intf.sv`, `include/axi/typedef.svh`

---

## 学习目标

完成本课学习后，你将能够：

1. 说明 AMBA 总线家族的演进关系和各协议的定位
2. 描述 AXI4 协议的五通道架构及其设计动机
3. 列举每个通道的所有信号及其含义
4. 解释 Valid/Ready 握手机制的工作原理和依赖规则
5. 画出一次完整写事务和读事务的时序图
6. 区分 AXI4、AXI4-Lite 和 AXI5 的主要差异
7. 将协议概念对应到本仓库的源码文件

---

## 目录

- [1.1 AMBA 总线家族概述](#11-amba-总线家族概述)
- [1.2 AXI 协议的设计哲学](#12-axi-协议的设计哲学)
- [1.3 五通道架构总览](#13-五通道架构总览)
- [1.4 Write Address 通道（AW）](#14-write-address-通道aw)
- [1.5 Write Data 通道（W）](#15-write-data-通道w)
- [1.6 Write Response 通道（B）](#16-write-response-通道b)
- [1.7 Read Address 通道（AR）](#17-read-address-通道ar)
- [1.8 Read Data 通道（R）](#18-read-data-通道r)
- [1.9 AXI4 与 AXI4-Lite 信号对比](#19-axi4-与-axi4-lite-信号对比)
- [1.10 Valid/Ready 握手机制](#110-validready-握手机制)
- [1.11 写事务的完整流程](#111-写事务的完整流程)
- [1.12 读事务的完整流程](#112-读事务的完整流程)
- [1.13 通道独立性与并发](#113-通道独立性与并发)
- [1.14 源码映射：从协议到代码](#114-源码映射从协议到代码)
- [1.15 关键术语表](#115-关键术语表)
- [1.16 课后练习](#116-课后练习)
- [1.17 下一课预告](#117-下一课预告)

---

## 1.1 AMBA 总线家族概述

AMBA（Advanced Microcontroller Bus Architecture）是 ARM 公司定义的片上总线标准家族。随着芯片复杂度的提升，AMBA 经历了多代演进，每一代都针对不同的性能需求和应用场景。

### 1.1.1 三代总线协议对比

```
性能/复杂度
    ▲
    │                                    ┌─────────┐
    │                                    │  AXI4   │  高性能、高带宽
    │                                    │ (AMBA 3) │  处理器、DMA、内存控制器
    │                                    └─────────┘
    │                        ┌─────────┐
    │                        │  AHB    │  中等性能
    │                        │(AMBA 2) │  处理器总线、片上 SRAM
    │                        └─────────┘
    │            ┌─────────┐
    │            │  APB    │  低功耗、低带宽
    │            │(AMBA 2) │  UART、GPIO、Timer
    │            └─────────┘
    └───────────────────────────────────────────────► 应用场景复杂度
```

| 协议 | 全称 | 典型应用 | 关键特性 |
|------|------|----------|----------|
| **APB** | Advanced Peripheral Bus | UART, GPIO, Timer, SPI | 简单两阶段传输，无流水线，低功耗 |
| **AHB** | Advanced High-performance Bus | 处理器本地总线, 片上 SRAM | 单地址通道，流水线传输，共享总线 |
| **AXI** | Advanced eXtensible Interface | 高性能互连, DDR 控制器, DMA | 五独立通道，突发传输，乱序完成 |

### 1.1.2 为什么需要 AXI？

在现代 SoC 中，处理器核心、DMA 引擎、加速器和内存控制器之间需要同时进行大量数据传输。AHB 的**共享总线**架构成为了瓶颈：

- **单地址通道**：读和写共享地址总线，不能同时进行
- **无乱序支持**：响应必须按请求顺序返回，高延迟从机会阻塞整个总线
- **有限的并发性**：同一时刻只能有一个主机占用总线

AXI 通过以下设计解决了这些问题：

1. **读写分离**：独立的读通道和写通道，读写可以同时进行
2. **通道独立**：五个通道各自有独立的握手，互不阻塞
3. **乱序完成**：不同 ID 的事务可以乱序完成，高延迟从机不会阻塞低延迟事务
4. **突发传输**：一次地址握手可以传输多拍数据，减少地址总线开销

### 1.1.3 AXI 的版本演进

| 版本 | 规范 | 引入特性 |
|------|------|----------|
| AXI3 | AMBA 3.0 | 基础五通道架构、突发传输 |
| AXI4 | AMBA 4.0 | QoS、Region、更长 burst（256 拍）|
| AXI4-Lite | AMBA 4.0 | 简化版，仅单拍传输，适合寄存器访问 |
| AXI5 | AMBA 5.0 | 原子操作（ATOP）、持久性标记 |

> **本仓库遵循**：*AMBA AXI and ACE Protocol Specification, Issue F.b*，支持 AXI4 全部特性和 AXI5 的原子操作扩展。

---

## 1.2 AXI 协议的设计哲学

理解 AXI 的设计哲学，有助于深入理解后续的所有技术细节。

### 1.2.1 点对点连接

AXI 不是传统意义上的"总线"。它定义的是两个组件之间的**点对点接口**（point-to-point interface）：一个 **Master**（主机）和一个 **Slave**（从机）。

```
    ┌────────────┐                    ┌────────────┐
    │            │   AXI Interface    │            │
    │   Master   │◄══════════════════►│   Slave    │
    │            │   (点对点连接)      │            │
    └────────────┘                    └────────────┘
```

要构建多主多从的系统，需要在 Master 和 Slave 之间插入**互连（Interconnect）**组件，如本仓库中的 `axi_xbar`（交叉开关）：

```
    ┌──────┐         ┌─────────────┐         ┌──────┐
    │  M0  │◄═══════►│             │◄═══════►│  S0  │
    └──────┘         │             │         └──────┘
    ┌──────┐         │  axi_xbar   │         ┌──────┐
    │  M1  │◄═══════►│ (交叉开关)   │◄═══════►│  S1  │
    └──────┘         │             │         └──────┘
    ┌──────┐         │             │         ┌──────┐
    │  M2  │◄═══════►│             │◄═══════►│  S2  │
    └──────┘         └─────────────┘         └──────┘
```

### 1.2.2 通道即协议

AXI 将一次完整的数据传输分解为多个独立的**通道（Channel）**。每个通道有自己的握手信号，可以独立于其他通道运作。这种设计使得：

- 写地址和写数据可以同时发送（不必等地址被接受后才发数据）
- 读请求可以在写响应返回之前发出
- 不同通道可以运行在不同的速率上

### 1.2.3 事务模型

AXI 中的基本操作单元是**事务（Transaction）**：

- **写事务**：Master 发送地址（AW）和数据（W），Slave 返回响应（B）
- **读事务**：Master 发送地址（AR），Slave 返回数据和响应（R）

一次事务可以包含多拍数据传输，称为**突发传输（Burst Transfer）**。

---

## 1.3 五通道架构总览

AXI 协议定义了五个独立的单向通道。理解这五个通道是学习 AXI 的第一步。

### 1.3.1 通道总览图

```
                        AXI Interface
     Master                                          Slave
    ┌──────┐                                        ┌──────┐
    │      │══════ AW (Write Address) ════════════►  │      │
    │      │                                        │      │
    │      │══════ W  (Write Data)    ════════════►  │      │
    │      │                                        │      │
    │      │  ◄════ B  (Write Response) ════════════ │      │
    │      │                                        │      │
    │      │══════ AR (Read Address)  ════════════►  │      │
    │      │                                        │      │
    │      │  ◄════ R  (Read Data)    ════════════ │      │
    └──────┘                                        └──────┘

    ══════► : Master → Slave 方向（请求方向）
    ◄══════ : Slave → Master 方向（响应方向）
```

### 1.3.2 通道功能一览

| 通道 | 全称 | 方向 | 功能描述 |
|------|------|------|----------|
| **AW** | Write Address Channel | Master → Slave | 传输写事务的目标地址和控制信息 |
| **W** | Write Data Channel | Master → Slave | 传输写数据和字节选通信号 |
| **B** | Write Response Channel | Slave → Master | 传输写事务的完成状态 |
| **AR** | Read Address Channel | Master → Slave | 传输读事务的目标地址和控制信息 |
| **R** | Read Data Channel | Slave → Master | 传输读数据和读事务的完成状态 |

### 1.3.3 写路径和读路径

AXI 的读写路径是完全独立的：

```
写路径（3 个通道）：          读路径（2 个通道）：

  Master                        Master
    │                             │
    ├─── AW (地址) ───►           ├─── AR (地址) ───►
    │                             │
    ├─── W  (数据) ───►           │                    Slave
    │                             │                      │
    │                 Slave       ◄─── R  (数据+响应) ───┤
    │                   │
    ◄─── B  (响应) ────┤
```

> **设计启示**：读操作只需要 2 个通道（地址 + 数据/响应），而写操作需要 3 个通道（地址 + 数据 + 响应）。这是因为写操作需要单独确认完成状态，而读操作的响应可以和数据一起返回。

---

## 1.4 Write Address 通道（AW）

Write Address 通道传输写事务的地址和控制信息。**每次 AW 握手发起一个新的写事务。**

### 1.4.1 信号列表

| 信号名 | 位宽 | 方向 | 说明 |
|--------|------|------|------|
| `awid` | 可配置 | M→S | 写事务 ID，用于标识事务和支持乱序 |
| `awaddr` | 可配置 | M→S | 写事务的目标地址（第一拍的地址） |
| `awlen` | 8 bit | M→S | Burst 长度，实际传输拍数 = awlen + 1 |
| `awsize` | 3 bit | M→S | 每拍传输的字节数 = 2^awsize |
| `awburst` | 2 bit | M→S | Burst 类型：FIXED/INCR/WRAP |
| `awlock` | 1 bit | M→S | 原子访问锁定类型 |
| `awcache` | 4 bit | M→S | 内存属性（可缓存、可缓冲等） |
| `awprot` | 3 bit | M→S | 保护类型（特权/安全/指令数据） |
| `awqos` | 4 bit | M→S | 服务质量标识 |
| `awregion` | 4 bit | M→S | 区域标识（可选，用于从机内部地址空间划分） |
| `awatop` | 6 bit | M→S | **AXI5 扩展**：原子操作类型 |
| `awuser` | 可配置 | M→S | 用户自定义信号（协议不规定用途） |
| `awvalid` | 1 bit | M→S | 地址有效信号 |
| `awready` | 1 bit | S→M | 从机准备好接收信号 |

### 1.4.2 关键信号详解

#### awid — 事务标识符

ID 是 AXI 乱序机制的基础。关键规则：

- **同一 ID 的事务必须按序完成**：如果 Master 先后发出 ID=3 的两个写事务，Slave 必须按相同顺序返回 B 响应
- **不同 ID 的事务可以乱序完成**：ID=3 和 ID=5 的事务响应可以以任意顺序返回
- ID 宽度在实例化时确定，本仓库中通过参数 `AXI_ID_WIDTH` 配置

#### awlen — Burst 长度

```
awlen 值:   0      1      2      3     ...    255
实际拍数:   1      2      3      4     ...    256
```

AXI4 支持最多 256 拍的 burst（awlen 最大为 255）。AXI3 仅支持最多 16 拍。

#### awsize — 传输尺寸

```
awsize 值:  0     1     2     3     4      5      6       7
字节数:     1     2     4     8     16     32     64      128
```

awsize 不能超过数据总线的宽度。例如，64 位（8 字节）数据总线上，awsize 最大为 3。

#### awburst — Burst 类型

| 值 | 名称 | 说明 |
|----|------|------|
| 2'b00 | FIXED | 所有拍使用相同地址，用于 FIFO 访问 |
| 2'b01 | INCR | 地址每拍递增 2^awsize 字节，最常用 |
| 2'b10 | WRAP | 地址递增到边界后回绕，用于缓存行填充 |
| 2'b11 | — | 保留，不使用 |

在 `src/axi_pkg.sv` 中对应的定义：

```systemverilog
// src/axi_pkg.sv, 第 75-87 行
localparam BURST_FIXED = 2'b00;
localparam BURST_INCR  = 2'b01;
localparam BURST_WRAP  = 2'b10;
```

### 1.4.3 源码中的 AW 通道定义

在 `include/axi/typedef.svh` 中，AW 通道的 struct 通过宏定义生成：

```systemverilog
// include/axi/typedef.svh, 第 34-48 行
`define AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)  \
  typedef struct packed {                                        \
    id_t              id;                                        \
    addr_t            addr;                                      \
    axi_pkg::len_t    len;                                       \
    axi_pkg::size_t   size;                                      \
    axi_pkg::burst_t  burst;                                     \
    logic             lock;                                      \
    axi_pkg::cache_t  cache;                                     \
    axi_pkg::prot_t   prot;                                      \
    axi_pkg::qos_t    qos;                                       \
    axi_pkg::region_t region;                                    \
    axi_pkg::atop_t   atop;                                      \
    user_t            user;                                      \
  } aw_chan_t;
```

> **注意**：`len`、`size`、`burst` 等字段直接使用 `axi_pkg` 中定义的类型，而 `id`、`addr`、`user` 的宽度通过宏参数指定。这种设计使得每个模块可以根据需要配置不同的 ID 和地址宽度。

---

## 1.5 Write Data 通道（W）

Write Data 通道传输写事务的实际数据。

### 1.5.1 信号列表

| 信号名 | 位宽 | 方向 | 说明 |
|--------|------|------|------|
| `wdata` | 可配置 | M→S | 写数据（通常 32/64/128/256/512 位） |
| `wstrb` | DATA_WIDTH/8 | M→S | 字节选通，每位对应一个字节 |
| `wlast` | 1 bit | M→S | 标记 Burst 的最后一拍 |
| `wuser` | 可配置 | M→S | 用户自定义信号 |
| `wvalid` | 1 bit | M→S | 数据有效信号 |
| `wready` | 1 bit | S→M | 从机准备好接收信号 |

### 1.5.2 关键信号详解

#### wstrb — 字节选通（Write Strobe）

`wstrb` 是 AXI 中非常重要的概念。它指示 `wdata` 中哪些字节是有效的：

```
以 32 位数据总线为例（4 字节，wstrb 为 4 位）：

wstrb = 4'b1111    → 所有 4 字节有效（全字写入）
wstrb = 4'b0011    → 只有低 2 字节有效（半字写入）
wstrb = 4'b0001    → 只有最低字节有效（字节写入）
wstrb = 4'b1100    → 只有高 2 字节有效

   wdata:    [byte3] [byte2] [byte1] [byte0]
   wstrb:       1       1       0       0     → 只写入 byte3 和 byte2
```

> **设计要点**：`wstrb` 的位宽始终等于 `wdata` 位宽除以 8。在源码中体现为 `AXI_STRB_WIDTH = AXI_DATA_WIDTH / 8`（`src/axi_intf.sv` 第 30 行）。

#### wlast — 最后一拍标记

在 burst 传输中，`wlast` 必须在最后一拍数据时拉高：

```
4 拍 burst（awlen=3）的 W 通道：

拍序:    beat 0    beat 1    beat 2    beat 3
wlast:     0         0         0         1      ← 最后一拍
```

### 1.5.3 W 通道的特殊性：没有 ID

注意 **W 通道没有 `wid` 信号**（AXI4 中已移除，AXI3 中有）。这意味着：

- W 通道的数据必须与 AW 通道的地址**按顺序**对应
- 如果 Master 先后发出 AW_A 和 AW_B，那么 W 通道的数据必须先是 A 的数据，再是 B 的数据
- 这就是为什么在 `axi_demux`（解复用器）中需要 ID 计数器来追踪 W 的路由目标

### 1.5.4 源码中的 W 通道定义

```systemverilog
// include/axi/typedef.svh, 第 49-55 行
`define AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t) \
  typedef struct packed {                                       \
    data_t data;                                                \
    strb_t strb;                                                \
    logic  last;                                                \
    user_t user;                                                \
  } w_chan_t;
```

---

## 1.6 Write Response 通道（B）

Write Response 通道由 Slave 发出，通知 Master 写事务已完成。

### 1.6.1 信号列表

| 信号名 | 位宽 | 方向 | 说明 |
|--------|------|------|------|
| `bid` | 可配置 | S→M | 响应 ID（必须与对应 awid 匹配） |
| `bresp` | 2 bit | S→M | 写响应状态 |
| `buser` | 可配置 | S→M | 用户自定义信号 |
| `bvalid` | 1 bit | S→M | 响应有效信号 |
| `bready` | 1 bit | M→S | Master 准备接收响应 |

### 1.6.2 响应类型

| 值 | 名称 | 说明 |
|----|------|------|
| 2'b00 | OKAY | 正常完成 |
| 2'b01 | EXOKAY | 独占访问成功（用于多处理器同步） |
| 2'b10 | SLVERR | 从机错误（从机检测到异常，如非法地址） |
| 2'b11 | DECERR | 译码错误（地址不匹配任何从机，通常由互连返回） |

在 `src/axi_pkg.sv` 中对应的定义：

```systemverilog
// src/axi_pkg.sv, 第 91-100 行
localparam RESP_OKAY   = 2'b00;
localparam RESP_EXOKAY = 2'b01;
localparam RESP_SLVERR = 2'b10;
localparam RESP_DECERR = 2'b11;
```

### 1.6.3 B 通道的关键特性

- **每个写事务只有一个 B 响应**：即使是 256 拍的 burst 写入，也只返回一个 B 响应
- **bid 必须匹配 awid**：Slave 必须在 B 响应中返回与请求相同的 ID
- **B 响应在所有写数据传输完成后才能发出**：Slave 必须接收完 wlast=1 的拍后才能返回 B

### 1.6.4 源码中的 B 通道定义

```systemverilog
// include/axi/typedef.svh, 第 56-61 行
`define AXI_TYPEDEF_B_CHAN_T(b_chan_t, id_t, user_t) \
  typedef struct packed {                             \
    id_t            id;                               \
    axi_pkg::resp_t resp;                             \
    user_t          user;                             \
  } b_chan_t;
```

---

## 1.7 Read Address 通道（AR）

Read Address 通道传输读事务的地址和控制信息，与 AW 通道结构非常相似。

### 1.7.1 信号列表

| 信号名 | 位宽 | 方向 | 说明 |
|--------|------|------|------|
| `arid` | 可配置 | M→S | 读事务 ID |
| `araddr` | 可配置 | M→S | 读地址 |
| `arlen` | 8 bit | M→S | Burst 长度 |
| `arsize` | 3 bit | M→S | 每拍传输的字节数 |
| `arburst` | 2 bit | M→S | Burst 类型 |
| `arlock` | 1 bit | M→S | 原子访问锁定 |
| `arcache` | 4 bit | M→S | 内存属性 |
| `arprot` | 3 bit | M→S | 保护类型 |
| `arqos` | 4 bit | M→S | 服务质量 |
| `arregion` | 4 bit | M→S | 区域标识 |
| `aruser` | 可配置 | M→S | 用户自定义 |
| `arvalid` | 1 bit | M→S | 地址有效 |
| `arready` | 1 bit | S→M | 从机就绪 |

### 1.7.2 AR 与 AW 的区别

AR 通道与 AW 通道几乎完全相同，但有一个关键区别：

- **AR 通道没有 `aratop` 字段**：原子操作仅通过写通道发起

在 `include/axi/typedef.svh` 中可以清楚地看到这个区别：

```systemverilog
// include/axi/typedef.svh, 第 62-75 行
`define AXI_TYPEDEF_AR_CHAN_T(ar_chan_t, addr_t, id_t, user_t)  \
  typedef struct packed {                                        \
    id_t              id;                                        \
    addr_t            addr;                                      \
    axi_pkg::len_t    len;                                       \
    axi_pkg::size_t   size;                                      \
    axi_pkg::burst_t  burst;                                     \
    logic             lock;                                      \
    axi_pkg::cache_t  cache;                                     \
    axi_pkg::prot_t   prot;                                      \
    axi_pkg::qos_t    qos;                                       \
    axi_pkg::region_t region;                                    \
    user_t            user;                                      \
  } ar_chan_t;
  // 注意：相比 AW 通道，这里没有 atop 字段
```

---

## 1.8 Read Data 通道（R）

Read Data 通道由 Slave 发出，返回读取的数据和响应状态。

### 1.8.1 信号列表

| 信号名 | 位宽 | 方向 | 说明 |
|--------|------|------|------|
| `rid` | 可配置 | S→M | 读响应 ID（匹配 arid） |
| `rdata` | 可配置 | S→M | 读出的数据 |
| `rresp` | 2 bit | S→M | 读响应状态（OKAY/EXOKAY/SLVERR/DECERR） |
| `rlast` | 1 bit | S→M | Burst 最后一拍标记 |
| `ruser` | 可配置 | S→M | 用户自定义 |
| `rvalid` | 1 bit | S→M | 数据有效 |
| `rready` | 1 bit | M→S | Master 准备接收 |

### 1.8.2 R 通道的特殊性

与 B 通道（每事务一个响应）不同，R 通道**每拍都有响应**：

```
4 拍 burst 读事务：

AR 握手:    [arid=5, araddr=0x100, arlen=3]  ← 一次地址握手

R 通道返回:
  beat 0:   rid=5, rdata=..., rresp=OKAY, rlast=0
  beat 1:   rid=5, rdata=..., rresp=OKAY, rlast=0
  beat 2:   rid=5, rdata=..., rresp=OKAY, rlast=0
  beat 3:   rid=5, rdata=..., rresp=OKAY, rlast=1  ← 最后一拍
```

> **关键区别**：R 通道的每一拍都有独立的 `rresp`，这意味着同一个 burst 中不同拍的响应可以不同（例如前几拍 OKAY，最后一拍 SLVERR）。而 B 通道整个写事务只有一个响应。

### 1.8.3 源码中的 R 通道定义

```systemverilog
// include/axi/typedef.svh, 第 76-83 行
`define AXI_TYPEDEF_R_CHAN_T(r_chan_t, data_t, id_t, user_t) \
  typedef struct packed {                                     \
    id_t            id;                                       \
    data_t          data;                                     \
    axi_pkg::resp_t resp;                                     \
    logic           last;                                     \
    user_t          user;                                     \
  } r_chan_t;
```

---

## 1.9 AXI4 与 AXI4-Lite 信号对比

AXI4-Lite 是 AXI4 的简化子集，去掉了 burst、ID 和 user 信号，仅支持单拍传输。

### 1.9.1 信号对比表

| 信号组 | AXI4 | AXI4-Lite | 说明 |
|--------|------|-----------|------|
| **AW 通道** | | | |
| ID | awid | — | Lite 无 ID，不支持乱序 |
| 地址 | awaddr | awaddr | 相同 |
| Burst 控制 | awlen, awsize, awburst | — | Lite 仅单拍，无需 burst |
| Lock | awlock | — | |
| Cache | awcache | — | |
| Prot | awprot | awprot | 相同 |
| QoS/Region | awqos, awregion | — | |
| ATOP | awatop | — | |
| User | awuser | — | |
| 握手 | awvalid/awready | awvalid/awready | 相同 |
| **W 通道** | | | |
| 数据 | wdata | wdata | 相同 |
| Strobe | wstrb | wstrb | 相同 |
| Last | wlast | — | Lite 只有单拍，不需要 |
| User | wuser | — | |
| 握手 | wvalid/wready | wvalid/wready | 相同 |
| **B 通道** | | | |
| ID | bid | — | |
| 响应 | bresp | bresp | 相同 |
| User | buser | — | |
| 握手 | bvalid/bready | bvalid/bready | 相同 |
| **AR 通道** | | | |
| ID | arid | — | |
| 地址 | araddr | araddr | 相同 |
| Burst 控制 | arlen, arsize, arburst | — | |
| 其他 | arlock,arcache,arqos... | — | |
| Prot | arprot | arprot | 相同 |
| 握手 | arvalid/arready | arvalid/arready | 相同 |
| **R 通道** | | | |
| ID | rid | — | |
| 数据 | rdata | rdata | 相同 |
| 响应 | rresp | rresp | 相同 |
| Last | rlast | — | |
| User | ruser | — | |
| 握手 | rvalid/rready | rvalid/rready | 相同 |

### 1.9.2 适用场景

```
AXI4 适合:                          AXI4-Lite 适合:
┌─────────────────────────┐         ┌─────────────────────────┐
│ 高带宽数据传输           │         │ 低速控制/状态寄存器      │
│ DDR 内存控制器          │         │ UART 配置寄存器          │
│ DMA 引擎               │         │ GPIO 控制寄存器          │
│ 需要乱序优化的场景       │         │ Timer 寄存器            │
│ 高性能处理器接口         │         │ 中断控制器              │
└─────────────────────────┘         └─────────────────────────┘
```

---

## 1.10 Valid/Ready 握手机制

握手机制是 AXI 协议中最基础也是最重要的概念。每个通道都使用相同的 Valid/Ready 握手协议。

### 1.10.1 基本规则

1. **发送方**（Source）控制 `valid` 信号，表示数据已准备好
2. **接收方**（Destination）控制 `ready` 信号，表示可以接收
3. **传输发生的条件**：`valid` 和 `ready` 在时钟上升沿**同时为高**

```
信号所有者：
  AW/W/AR 通道: valid 由 Master 驱动, ready 由 Slave 驱动
  B/R 通道:     valid 由 Slave 驱动,  ready 由 Master 驱动
```

### 1.10.2 三种握手场景

#### 场景一：Valid 先到（Source 先准备好）

这是最常见的场景：Master 准备好了数据，但 Slave 还没准备好接收。

```
          ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
 clk      │   │   │   │   │   │   │   │   │   │   │   │
       ───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───
               ┌───────────────────────┐
 valid  ───────┘                       └─────────────────
 (Source)
 data   ═══════╡      VALID DATA       ╞═════════════════
                              ┌────────┐
 ready  ──────────────────────┘        └─────────────────
 (Dest)
                              ▲
                          传输发生！
                     (valid && ready 同时为高)
```

**关键规则**：一旦 `valid` 拉高，Source 必须保持 `valid` 为高以及数据不变，直到握手完成。**不允许在 `ready` 为低时撤回 `valid`**。

#### 场景二：Ready 先到（Destination 先准备好）

```
          ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
 clk      │   │   │   │   │   │   │   │   │   │   │   │
       ───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───
                              ┌────────┐
 valid  ──────────────────────┘        └─────────────────
 (Source)
 data   ══════════════════════╡ V.DATA ╞═════════════════
               ┌───────────────────────────────┐
 ready  ───────┘                               └─────────
 (Dest)
                              ▲
                          传输发生！
```

Destination **可以**在 `valid` 为低时就拉高 `ready`（表示随时可以接收）。

#### 场景三：同时到达

```
          ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
 clk      │   │   │   │   │   │   │   │   │   │   │   │
       ───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───
                      ┌────────┐
 valid  ──────────────┘        └─────────────────────────
 (Source)
 data   ══════════════╡ V.DATA ╞═════════════════════════
                      ┌────────┐
 ready  ──────────────┘        └─────────────────────────
 (Dest)
                      ▲
                  传输发生！
              (第一个上升沿就完成)
```

### 1.10.3 依赖规则（极其重要）

AXI 规范对 Valid 和 Ready 的依赖关系有严格规定，违反这些规则会导致**死锁**：

```
  ┌──────────────────────────────────────────────────────┐
  │              Valid/Ready 依赖规则                      │
  │                                                      │
  │  规则 1: valid 不能等待 ready                          │
  │     Source 必须独立地决定何时拉高 valid                  │
  │     valid 不能依赖 ready 的状态                        │
  │                                                      │
  │  规则 2: ready 可以等待 valid                          │
  │     Destination 可以在看到 valid 后才拉高 ready         │
  │     （但也可以提前拉高）                               │
  │                                                      │
  │  规则 3: valid 拉高后不可撤回                           │
  │     一旦 valid 为高，必须保持到握手完成                  │
  │     数据也必须保持稳定                                 │
  └──────────────────────────────────────────────────────┘
```

> **为什么 valid 不能等待 ready？**
>
> 考虑以下死锁场景：如果 Master 等 Slave 的 ready 才拉 valid，同时 Slave 等 Master 的 valid 才拉 ready —— 双方互相等待，永远无法完成握手。规定 valid 不依赖 ready 就打破了这个循环。

### 1.10.4 在仿真中的验证

本仓库的 `AXI_BUS_DV` 接口（`src/axi_intf.sv`）内置了握手稳定性断言，会在仿真中自动检查这些规则：

```systemverilog
// src/axi_intf.sv, AXI_BUS_DV 接口中的断言（约第 208-251 行）
// 当 valid 为高且 ready 为低时，检查所有数据信号是否保持稳定
// 例如 AW 通道的断言：
assert property (@(posedge clk_i)
    (aw_valid && !aw_ready) |=> $stable(aw_addr))
    else $error("aw_addr changed while valid && !ready");
```

---

## 1.11 写事务的完整流程

一个完整的写事务由三个通道协作完成。

### 1.11.1 单拍写事务

```
          cycle 1   cycle 2   cycle 3   cycle 4   cycle 5   cycle 6
            ┌───┐     ┌───┐     ┌───┐     ┌───┐     ┌───┐     ┌───┐
 clk        │   │     │   │     │   │     │   │     │   │     │   │
         ───┘   └─────┘   └─────┘   └─────┘   └─────┘   └─────┘   └───

 AW 通道:
            ┌─────────────────────┐
 awvalid ───┘                     └─────────────────────────────────
 awaddr  ═══╡    0x1000_0000      ╞═════════════════════════════════
 awlen   ═══╡         0           ╞═══  (单拍: len=0)
                      ┌───────────┐
 awready ─────────────┘           └─────────────────────────────────
                      ▲
                  AW 握手完成

 W 通道:
                      ┌───────────┐
 wvalid  ─────────────┘           └─────────────────────────────────
 wdata   ═════════════╡ 0xDEAD   ╞═════════════════════════════════
 wstrb   ═════════════╡  4'b1111 ╞═══
 wlast   ─────────────┘           └───  (单拍, last=1)
                      ┌───────────┐
 wready  ─────────────┘           └─────────────────────────────────
                      ▲
                  W 握手完成

 B 通道:
                                              ┌───────────┐
 bvalid  ─────────────────────────────────────┘           └─────────
 bresp   ═════════════════════════════════════╡   OKAY    ╞═════════
 bid     ═════════════════════════════════════╡   id=0    ╞═════════
                                              ┌───────────┐
 bready  ─────────────────────────────────────┘           └─────────
                                              ▲
                                          B 握手完成
                                       (写事务完成！)
```

### 1.11.2 多拍写事务（4 拍 Burst）

```
 AW 通道:
            ┌─────────┐
 awvalid ───┘         └───────────────────────────────────────
 awaddr  ═══╡ 0x1000  ╞═══  awlen=3 (4拍), awburst=INCR
            ┌─────────┐
 awready ───┘         └───────────────────────────────────────
            ▲
        AW 握手

 W 通道:
            ┌─────────────────────────────────────────┐
 wvalid  ───┘                                         └───────
 wdata   ═══╡ data_0 ╞╡ data_1 ╞╡ data_2 ╞╡ data_3  ╞═══════
 wlast   ───────────────────────────────────┘         └───────
 wready  ═══...(由 Slave 按需控制)

              beat 0    beat 1    beat 2    beat 3
                                            ▲ wlast=1

 B 通道:  (在 beat 3 的 W 握手完成后)
                                                ┌─────────┐
 bvalid  ───────────────────────────────────────┘         └───
 bresp   ═══════════════════════════════════════╡  OKAY   ╞═══
```

### 1.11.3 AW 和 W 的时序关系

AXI 规范允许以下三种时序关系：

```
情况 1: AW 先于 W（最常见）
  AW: ──┤addr├──────────────
  W:  ────────────┤data├────

情况 2: W 先于 AW（允许但不常见）
  AW: ────────────┤addr├────
  W:  ──┤data├──────────────

情况 3: AW 和 W 同时（允许）
  AW: ──┤addr├──────────────
  W:  ──┤data├──────────────
```

> **关键理解**：AW 和 W 通道是**独立的**，Master 可以在 AW 握手完成之前就开始发送 W 数据。这种灵活性允许更高的总线利用率。

---

## 1.12 读事务的完整流程

读事务只涉及两个通道：AR 和 R。

### 1.12.1 单拍读事务

```
          cycle 1   cycle 2   cycle 3   cycle 4   cycle 5
            ┌───┐     ┌───┐     ┌───┐     ┌───┐     ┌───┐
 clk        │   │     │   │     │   │     │   │     │   │
         ───┘   └─────┘   └─────┘   └─────┘   └─────┘   └───

 AR 通道:
            ┌─────────┐
 arvalid ───┘         └─────────────────────────────────────
 araddr  ═══╡ 0x2000  ╞══  arlen=0 (单拍)
            ┌─────────┐
 arready ───┘         └─────────────────────────────────────
            ▲
        AR 握手

 R 通道:                    (Slave 处理后返回数据)
                            ┌─────────┐
 rvalid  ───────────────────┘         └─────────────────────
 rdata   ═══════════════════╡ 0xBEEF ╞═════════════════════
 rresp   ═══════════════════╡  OKAY  ╞═════════════════════
 rlast   ───────────────────┘         └───  (单拍, last=1)
                            ┌─────────┐
 rready  ───────────────────┘         └─────────────────────
                            ▲
                         R 握手
                      (读事务完成！)
```

### 1.12.2 多拍读事务（4 拍 Burst）

```
 AR 通道:
            ┌─────────┐
 arvalid ───┘         └───────────────────────────────────────
 araddr  ═══╡ 0x2000  ╞══  arlen=3 (4拍), arburst=INCR
            ┌─────────┐
 arready ───┘         └───────────────────────────────────────

 R 通道:
            (Slave 从 0x2000 开始连续读取 4 个数据)
                  ┌───────────────────────────────────┐
 rvalid  ─────────┘                                   └───────
 rdata   ═════════╡ D_0x2000╞╡ D_0x2004╞╡ D_0x2008╞╡ D_0x200C╞
 rresp   ═════════╡  OKAY   ╞╡  OKAY   ╞╡  OKAY   ╞╡  OKAY  ╞
 rlast   ─────────────────────────────────────────────┘       └
                    beat 0     beat 1     beat 2     beat 3
                                                     ▲ rlast=1
```

### 1.12.3 Outstanding 读事务（流水线读）

AXI 允许 Master 在前一个读事务完成之前就发出下一个读地址，这称为**Outstanding Transaction**：

```
 AR 通道:
            ┌─────────┐     ┌─────────┐     ┌─────────┐
 arvalid ───┘ addr_A   └─────┘ addr_B   └─────┘ addr_C   └───
            ▲               ▲               ▲
         AR 握手 A       AR 握手 B       AR 握手 C

 R 通道:
                  ┌─────────┐     ┌─────────┐     ┌─────────┐
 rvalid  ─────────┘ data_A   └─────┘ data_B   └─────┘ data_C   └
                  ▲               ▲               ▲
               R 握手 A        R 握手 B        R 握手 C

 时间 ──────────────────────────────────────────────────────────►
```

这种流水线化的传输极大地提高了总线利用率，是 AXI 高性能的关键之一。

---

## 1.13 通道独立性与并发

### 1.13.1 通道之间的独立性

五个通道可以**同时活跃**，互不干扰：

```
时间 ──────────────────────────────────────────────────────►

AW:  ┤wr_addr_0├──────────────┤wr_addr_1├──────────────────
W:   ────┤wr_data_0├──────────────┤wr_data_1├──────────────
B:   ──────────────┤wr_resp_0├──────────────┤wr_resp_1├────
AR:  ──┤rd_addr_0├────┤rd_addr_1├──────────────────────────
R:   ────────┤rd_data_0├──────────┤rd_data_1├──────────────

  ↑ 五个通道在同一时间段内各自独立运作
```

### 1.13.2 通道之间的顺序依赖

虽然通道是独立的，但存在逻辑上的依赖关系（不是硬件上的时序依赖）：

```
  ┌──────────────────────────────────────────┐
  │          逻辑依赖关系（非时序约束）         │
  │                                          │
  │  写事务:                                  │
  │    AW ──(不要求先于)──► W                 │
  │    AW + W(last) ──(完成后)──► B           │
  │                                          │
  │  读事务:                                  │
  │    AR ──(完成后)──► R                     │
  │                                          │
  │  读写之间:                                │
  │    无依赖！读和写可以完全并行              │
  └──────────────────────────────────────────┘
```

- Slave 必须在收到 AW 后才知道写地址，但 Master 可以在 AW 握手前就开始发 W
- Slave 必须在完整接收 W 数据（wlast=1）后才能发出 B
- Slave 必须在收到 AR 后才能开始返回 R 数据

---

## 1.14 源码映射：从协议到代码

现在你已经理解了 AXI 协议的基本概念，让我们看看这些概念如何映射到本仓库的源码中。

### 1.14.1 三个核心文件

```
协议概念          源码文件                      作用
────────────────────────────────────────────────────────────
信号位宽/常量    src/axi_pkg.sv               定义所有协议常量和类型
通道 struct     include/axi/typedef.svh       宏生成参数化的通道结构体
接口定义        src/axi_intf.sv               SystemVerilog interface 定义
信号赋值        include/axi/assign.svh        接口与 struct 之间的赋值宏
```

### 1.14.2 协议常量 → `axi_pkg.sv`

所有在 AXI 规范中定义的常量都在 `src/axi_pkg.sv` 中以 `localparam` 形式定义：

```systemverilog
// src/axi_pkg.sv

// 信号位宽（第 25-45 行）
parameter int unsigned BurstWidth  = 32'd2;    // burst 类型位宽
parameter int unsigned RespWidth   = 32'd2;    // 响应类型位宽
parameter int unsigned LenWidth    = 32'd8;    // burst 长度位宽
parameter int unsigned SizeWidth   = 32'd3;    // 传输尺寸位宽
parameter int unsigned AtopWidth   = 32'd6;    // 原子操作位宽

// 类型定义（第 48-66 行）
typedef logic [1:0]  burst_t;    // burst 类型
typedef logic [1:0]   resp_t;    // 响应类型
typedef logic [7:0]    len_t;    // burst 长度
typedef logic [2:0]   size_t;    // 传输尺寸
typedef logic [5:0]   atop_t;    // 原子操作类型

// Burst 类型常量（第 75-87 行）
localparam BURST_FIXED = 2'b00;
localparam BURST_INCR  = 2'b01;
localparam BURST_WRAP  = 2'b10;

// 响应类型常量（第 91-100 行）
localparam RESP_OKAY   = 2'b00;
localparam RESP_EXOKAY = 2'b01;
localparam RESP_SLVERR = 2'b10;
localparam RESP_DECERR = 2'b11;
```

### 1.14.3 通道 struct → `typedef.svh`

协议中的每个通道被定义为 SystemVerilog `struct packed`，通过宏生成：

```systemverilog
// 使用示例：定义一组具体参数的 AXI 类型
typedef logic [31:0] addr_t;       // 32 位地址
typedef logic [63:0] data_t;       // 64 位数据
typedef logic [7:0]  strb_t;       // 8 字节选通
typedef logic [3:0]  id_t;         // 4 位 ID
typedef logic [0:0]  user_t;       // 1 位 user

// 一键生成所有通道类型和 req/resp 聚合类型
`AXI_TYPEDEF_ALL(my_axi, addr_t, id_t, data_t, strb_t, user_t)

// 上面的宏会生成以下 7 个类型：
//   my_axi_aw_chan_t   — AW 通道 struct
//   my_axi_w_chan_t    — W 通道 struct
//   my_axi_b_chan_t    — B 通道 struct
//   my_axi_ar_chan_t   — AR 通道 struct
//   my_axi_r_chan_t    — R 通道 struct
//   my_axi_req_t       — 请求聚合 struct（AW + W + AR + 握手信号）
//   my_axi_resp_t      — 响应聚合 struct（B + R + 握手信号）
```

### 1.14.4 接口定义 → `axi_intf.sv`

SystemVerilog `interface` 将所有通道信号打包，并通过 `modport` 区分方向：

```systemverilog
// src/axi_intf.sv, 第 20-109 行
interface AXI_BUS #(
    parameter int unsigned AXI_ADDR_WIDTH = 0,
    parameter int unsigned AXI_DATA_WIDTH = 0,
    parameter int unsigned AXI_ID_WIDTH   = 0,
    parameter int unsigned AXI_USER_WIDTH = 0
);
    // ... 所有 5 个通道的信号声明 ...

    modport Master (
        // AW: output (valid, addr, id...), input (ready)
        // W:  output (valid, data, strb, last), input (ready)
        // B:  input (valid, id, resp), output (ready)
        // AR: output (valid, addr, id...), input (ready)
        // R:  input (valid, data, id, resp, last), output (ready)
    );

    modport Slave  ( /* 方向与 Master 完全相反 */ );
    modport Monitor( /* 所有信号都是 input，用于观察 */ );
endinterface
```

### 1.14.5 两种端口风格的关系

本仓库中的模块有两种端口风格：

```
1. Struct 风格（核心模块使用）：
   module axi_mux #(
       parameter type slv_req_t  = logic,
       parameter type slv_resp_t = logic,
       ...
   ) (
       input  slv_req_t  [NoSlvPorts-1:0] slv_reqs_i,
       output slv_resp_t [NoSlvPorts-1:0] slv_resps_o,
       ...
   );

2. Interface 风格（_intf 变体模块）：
   module axi_mux_intf #(...) (
       AXI_BUS.Slave  slv [NoSlvPorts-1:0],
       AXI_BUS.Master mst,
       ...
   );
   // 内部仅做连线：用 assign 宏将 interface 转为 struct，
   // 然后实例化 struct 风格的核心模块
```

---

## 1.15 关键术语表

以下术语将在整个课程中反复出现（部分定义来自 `doc/README.md`）：

| 术语 | 英文 | 定义 |
|------|------|------|
| 握手 | Handshake | valid 和 ready 在时钟上升沿同时为高的事件 |
| 拍 | Beat | Burst 中的一次数据传输（一次握手） |
| 突发传输 | Burst Transfer | 一次地址握手后的多拍数据传输 |
| 事务 | Transaction | 一个完整的读或写操作（地址 + 数据 + 响应） |
| 在途 | In Flight | AX 握手已完成但最后一个响应尚未返回的事务 |
| 未决 | Pending | valid 为高但 ready 为低的状态 |
| 主机 | Master | 发起事务的一方（发送地址，控制读写方向） |
| 从机 | Slave | 响应事务的一方（接受地址，返回数据或确认） |
| 互连 | Interconnect | 连接多个 Master 和 Slave 的中间组件 |
| 字节选通 | Write Strobe | wstrb 信号，指示哪些字节有效 |
| 乱序 | Out-of-order | 不同 ID 的事务可以不按请求顺序完成 |
| 反压 | Back-pressure | 接收方通过不拉高 ready 来减缓发送方速率 |

---

## 1.16 课后练习

### 练习 1：通道方向（基础）

填写下表中每个信号的驱动方（Master 或 Slave）：

| 信号 | 驱动方 |
|------|--------|
| awvalid | ? |
| awready | ? |
| wdata | ? |
| bvalid | ? |
| bready | ? |
| arvalid | ? |
| rdata | ? |
| rready | ? |

### 练习 2：握手分析（基础）

以下时序中，哪些时钟周期发生了握手？

```
        cycle:  1    2    3    4    5    6
                ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
  clk       ────┘  └─┘  └─┘  └─┘  └─┘  └─┘  └──
                ┌─────────────────────┐
  valid     ────┘                     └──────────
            ─────────────┐     ┌─────────────────
  ready                  └─────┘
```

### 练习 3：Burst 参数计算（中级）

一个写事务的参数为：`awaddr=0x0010, awlen=7, awsize=2, awburst=INCR`

1. 这个 burst 传输多少拍？
2. 每拍传输多少字节？
3. 总共传输多少字节？
4. 写出每拍的地址序列。

### 练习 4：写事务时序图（中级）

画出以下写事务的完整时序图（包括 AW、W、B 三个通道）：

- 地址：0x4000
- 数据：2 拍 burst（awlen=1, awburst=INCR, awsize=2）
- 第一拍数据：0xAAAA（全字节有效）
- 第二拍数据：0xBBBB（仅低 2 字节有效）
- 响应：OKAY
- 假设 Slave 在所有通道上始终 ready

### 练习 5：协议规则判断（高级）

判断以下场景是否违反 AXI 协议规则，并说明原因：

1. Master 拉高 awvalid，但在 awready 为低的时候将 awvalid 拉低了
2. Master 在 AW 握手完成之前就开始发送 W 数据
3. Slave 在 wlast=1 的 W 握手完成之前就发出了 B 响应
4. Slave 返回的 bid 与 awid 不匹配
5. Master 的 awvalid 的拉高依赖于 awready 已经为高

### 练习 6：源码探索（实践）

打开 `src/axi_pkg.sv`，回答以下问题：

1. `CACHE_BUFFERABLE` 的值是什么？它在 cache 信号的哪一位？
2. `atop_t` 类型的位宽是多少？为什么 AR 通道没有对应的 atop 字段？
3. 找到 `num_bytes()` 函数，解释它的实现逻辑。

---

## 1.17 下一课预告

**第 2 课：握手机制、Burst 类型与响应编码**

在第 2 课中，我们将深入探讨：

- Valid/Ready 握手的更多边界情况
- 三种 Burst 类型的详细地址计算
- 响应优先级规则（`resp_precedence` 函数）
- `axi_pkg.sv` 中的地址计算辅助函数
- 4KB 边界限制

---

## 参考资料

1. **ARM AMBA AXI and ACE Protocol Specification, Issue F.b** — 官方协议规范
2. `src/axi_pkg.sv` — 本仓库的协议常量和类型定义
3. `src/axi_intf.sv` — AXI 接口定义
4. `include/axi/typedef.svh` — 通道 struct 宏定义
5. `doc/README.md` — 术语表

---

> **版权声明**：本教材基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库编写，该仓库遵循 Solderpad Hardware License v0.51。
