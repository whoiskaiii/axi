# AXI 总线协议 30 课课程大纲

> 基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库（v0.39.9）
> 遵循 AMBA AXI and ACE Protocol Specification, Issue F.b

---

## 课程总览

本课程共 30 课，分为 8 个单元，按照**从基础到进阶**的顺序编排。建议按顺序学习，每课标注了前置依赖。

| 单元 | 课程 | 主题 | 难度 |
|------|------|------|------|
| 一 | 第 1-5 课 | 协议基础与类型系统 | 入门 |
| 二 | 第 6-8 课 | 缓冲与基础从机 | 入门-中级 |
| 三 | 第 9-11 课 | 解复用器（Demux） | 中级-高级 |
| 四 | 第 12-14 课 | 复用器（Mux） | 中级-高级 |
| 五 | 第 15-17 课 | 交叉开关互连 | 高级 |
| 六 | 第 18-21 课 | 协议与数据转换 | 中级-高级 |
| 七 | 第 22-26 课 | ID 管理与 Burst 处理 | 高级 |
| 八 | 第 27-30 课 | 系统级功能与验证 | 高级 |

---

## 单元一：协议基础与类型系统（第 1-5 课）

> 目标：掌握 AXI 协议核心概念和仓库的类型系统，为阅读所有模块打下基础。

---

### 第 1 课：AXI 协议概述与五通道架构

**学习目标：** 理解 AXI 协议的基本架构和设计哲学

**知识点：**
- AMBA 总线家族概述（APB → AHB → AXI）
- AXI4 的五个独立通道及其方向：
  - Write Address (AW)：Master → Slave，写地址和控制信息
  - Write Data (W)：Master → Slave，写数据和字节选通（strobe）
  - Write Response (B)：Slave → Master，写完成响应
  - Read Address (AR)：Master → Slave，读地址和控制信息
  - Read Data (R)：Slave → Master，读数据和读响应
- 通道独立性：五个通道可以独立握手，无固定时序依赖
- AXI4 vs AXI4-Lite vs AXI5 的区别

**核心信号一览：**

```
AW 通道: awid, awaddr, awlen, awsize, awburst, awlock, awcache, awprot, awqos, awregion, awatop, awuser, awvalid, awready
W  通道: wdata, wstrb, wlast, wuser, wvalid, wready
B  通道: bid, bresp, buser, bvalid, bready
AR 通道: arid, araddr, arlen, arsize, arburst, arlock, arcache, arprot, arqos, arregion, aruser, arvalid, arready
R  通道: rid, rdata, rresp, rlast, ruser, rvalid, rready
```

**课后练习：** 画出一次写事务（AW → W → B）和一次读事务（AR → R）的时序图

**前置依赖：** 无

---

### 第 2 课：握手机制、Burst 类型与响应编码

**学习目标：** 深入理解 AXI 传输的核心机制

**知识点：**

#### 2.1 Valid/Ready 握手
- 握手规则：valid 和 ready 同时为高时完成一次传输（beat）
- 依赖规则：valid 不能依赖 ready（防止死锁），ready 可以依赖 valid
- 三种握手场景：valid 先到、ready 先到、同时到达

```
场景1: valid 先于 ready        场景2: ready 先于 valid        场景3: 同时到达
     ┌──────────┐                    ┌──────┐               ┌──────┐
valid┘          └──         valid────┘      └──      valid──┘      └──
          ┌────┐               ┌──────────┐               ┌──────┐
ready─────┘    └──         ready┘          └──      ready──┘      └──
          ▲                          ▲                      ▲
      handshake                  handshake              handshake
```

#### 2.2 Burst 类型（对应 `axi_pkg.sv` 中的常量）
- `BURST_FIXED` (2'b00)：地址不变，适用于 FIFO 访问
- `BURST_INCR` (2'b01)：地址递增，最常用的类型
- `BURST_WRAP` (2'b10)：地址回绕，用于缓存行填充

#### 2.3 Burst 参数
- `AxLEN`：Burst 长度（实际拍数 = AxLEN + 1，最大 256 拍）
- `AxSIZE`：每拍传输的字节数（2^AxSIZE 字节）
- Burst 不能跨越 4KB 边界（协议规定）

#### 2.4 响应类型
- `RESP_OKAY` (2'b00)：正常完成
- `RESP_EXOKAY` (2'b01)：独占访问成功
- `RESP_SLVERR` (2'b10)：从机错误
- `RESP_DECERR` (2'b11)：译码错误（地址无法匹配任何从机）

**对应源码：** `src/axi_pkg.sv` 第 25-95 行

**课后练习：** 计算以下 Burst 的地址序列：起始地址 0x1000, SIZE=2 (4字节), LEN=3, BURST=INCR

**前置依赖：** 第 1 课

---

### 第 3 课：AXI Package — 类型、常量与辅助函数

**学习目标：** 精读 `axi_pkg.sv`，掌握仓库的类型基础设施

**知识点：**

#### 3.1 信号宽度常量
```systemverilog
BurstWidth  = 2;   RespWidth  = 2;   CacheWidth  = 4;
ProtWidth   = 3;   QosWidth   = 4;   RegionWidth = 4;
LenWidth    = 8;   SizeWidth  = 3;   LockWidth   = 1;
AtopWidth   = 6;   NsaidWidth = 4;
```

#### 3.2 枚举类型
- `burst_t`, `resp_t`, `cache_t`, `prot_t`, `qos_t`, `region_t`, `len_t`, `size_t`, `atop_t`

#### 3.3 Cache 属性位域
- Bufferable, Modifiable, Read-Allocate, Write-Allocate
- 不同组合对应的内存类型（Device / Normal / Write-Through / Write-Back）

#### 3.4 ATOP（原子操作）编码
- `ATOP_ATOMICSWAP`, `ATOP_ATOMICCOMPARE`
- `ATOP_ATOMICLOAD`, `ATOP_ATOMICSTORE`
- 原子操作的算术功能位（ADD, CLR, EOR, SET, SMAX, SMIN, UMAX, UMIN）

#### 3.5 辅助函数
- `num_bytes(size)`：根据 AxSIZE 计算字节数
- `aligned_addr(addr, size)`：计算对齐地址
- `wrap_boundary(addr, len, size)`：计算 WRAP 回绕边界
- `beat_addr(addr, size, len, burst, i)`：计算第 i 拍的地址
- `beat_lower_byte(addr, size, len, burst, strb_width, i)`：计算有效字节范围

#### 3.6 Crossbar 配置结构体
```systemverilog
typedef struct packed {
    int unsigned NoSlvPorts;    // 从机端口数（连接 Master）
    int unsigned NoMstPorts;    // 主机端口数（连接 Slave）
    int unsigned MaxMstTrans;   // 每个主机端口最大未完成事务数
    int unsigned MaxSlvTrans;   // 每个从机端口最大未完成事务数
    // ...
} xbar_cfg_t;
```

**实践：** 在仿真器中调用 `beat_addr()` 函数，验证不同 Burst 类型下的地址序列

**对应源码：** `src/axi_pkg.sv` 全文

**前置依赖：** 第 1-2 课

---

### 第 4 课：Typedef 宏 — 参数化类型生成

**学习目标：** 理解 struct-based 端口风格，掌握宏驱动的类型系统

**知识点：**

#### 4.1 为什么使用 struct 而不是散列信号？
- 减少端口连接错误
- 参数化更灵活
- 模块间传递更简洁
- 本仓库的核心设计约定（见 `CONTRIBUTING.md`）

#### 4.2 单通道 Typedef 宏
```systemverilog
`AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)  // 写地址通道
`AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t)   // 写数据通道
`AXI_TYPEDEF_B_CHAN_T(b_chan_t, id_t, user_t)              // 写响应通道
`AXI_TYPEDEF_AR_CHAN_T(ar_chan_t, addr_t, id_t, user_t)   // 读地址通道
`AXI_TYPEDEF_R_CHAN_T(r_chan_t, data_t, id_t, user_t)     // 读数据通道
```

#### 4.3 请求/响应聚合 struct
```systemverilog
`AXI_TYPEDEF_REQ_T(req_t, aw_chan_t, w_chan_t, ar_chan_t)
`AXI_TYPEDEF_RESP_T(resp_t, b_chan_t, r_chan_t)
```
- `req_t` 包含：aw, aw_valid, w, w_valid, ar, ar_valid, b_ready, r_ready
- `resp_t` 包含：b, b_valid, r, r_valid, aw_ready, w_ready, ar_ready

#### 4.4 一键生成宏
```systemverilog
`AXI_TYPEDEF_ALL(axi, addr_t, id_t, data_t, strb_t, user_t)
// 自动生成: axi_aw_chan_t, axi_w_chan_t, axi_b_chan_t,
//          axi_ar_chan_t, axi_r_chan_t, axi_req_t, axi_resp_t
```

#### 4.5 AXI4-Lite 等效宏
```systemverilog
`AXI_LITE_TYPEDEF_ALL(axi_lite, addr_t, data_t, strb_t)
// 无 ID 和 User 字段
```

**实践：** 用宏展开一组具体参数（如 AddrWidth=32, DataWidth=64, IdWidth=4）的完整类型定义

**对应源码：** `include/axi/typedef.svh`

**前置依赖：** 第 1-3 课

---

### 第 5 课：赋值宏与接口定义

**学习目标：** 掌握信号赋值工具和 SystemVerilog interface 用法

**知识点：**

#### 5.1 赋值宏（`assign.svh`）

| 宏 | 功能 |
|----|------|
| `AXI_ASSIGN(dst, src)` | 接口到接口的完整赋值 |
| `AXI_ASSIGN_TO_REQ(req, axi_if)` | 接口到 req struct 赋值 |
| `AXI_ASSIGN_FROM_REQ(axi_if, req)` | req struct 到接口赋值 |
| `AXI_ASSIGN_TO_RESP(resp, axi_if)` | 接口到 resp struct 赋值 |
| `AXI_ASSIGN_FROM_RESP(axi_if, resp)` | resp struct 到接口赋值 |

#### 5.2 端口定义宏（`port.svh`）
- 用于 Vivado 等不完整支持 interface 的工具
- 将 AXI 信号展平为独立端口

#### 5.3 AXI Interface 定义（`axi_intf.sv`）
```systemverilog
interface AXI_BUS #(
    parameter int unsigned AXI_ADDR_WIDTH = 0,
    parameter int unsigned AXI_DATA_WIDTH = 0,
    parameter int unsigned AXI_ID_WIDTH   = 0,
    parameter int unsigned AXI_USER_WIDTH = 0
);
    modport Master ( /* 输出: AW/W/AR 的 valid 和数据; 输入: B/R 和 ready */ );
    modport Slave  ( /* 方向相反 */ );
endinterface
```

#### 5.4 两种端口风格的对比
- **Interface 风格 (`_intf` 后缀模块)**：简洁，适合顶层连接
- **Struct 风格（核心模块）**：灵活，适合参数化设计
- 本仓库约定：核心模块用 struct，`_intf` 变体仅做连线适配

**实践：** 写一个小 wrapper，用赋值宏在 interface 和 struct 之间互转

**对应源码：** `include/axi/assign.svh`, `include/axi/port.svh`, `src/axi_intf.sv`

**前置依赖：** 第 1-4 课

---

## 单元二：缓冲与基础从机（第 6-8 课）

> 目标：理解 AXI 中最基本的功能模块，为理解复杂互连做准备。

---

### 第 6 课：寄存器切割 — `axi_cut`

**学习目标：** 理解如何在保持协议正确性的前提下插入流水线级

**知识点：**
- Spill Register 的原理：解耦 valid/ready 组合路径
- 五个通道独立的旁路（Bypass）控制
- 参数 `Bypass`, `BypassAw`, `BypassW`, `BypassB`, `BypassAr`, `BypassR`
- 应用场景：时序优化、打断长组合路径
- 与普通寄存器的区别：spill register 不增加气泡（bubble-free）

**设计模式：** 每通道独立实例化 spill_register

**代码结构：**
```
axi_cut
├── spill_register (AW channel)
├── spill_register (W channel)
├── spill_register (B channel)
├── spill_register (AR channel)
└── spill_register (R channel)
```

**课后练习：** 分析插入 `axi_cut` 前后，AW→W→B 路径的时序变化

**对应源码：** `src/axi_cut.sv`

**前置依赖：** 第 1-5 课

---

### 第 7 课：通道 FIFO — `axi_fifo`

**学习目标：** 理解 FIFO 缓冲在 AXI 互连中的作用

**知识点：**
- 每通道独立的 FIFO，深度可配置
- 参数：`Depth`（FIFO 深度），`FallThrough`（是否直通）
- 深度为 0 时退化为直连（passthrough）
- FallThrough 模式：数据入队后同一周期即可出队
- FIFO 满时反压（back-pressure）上游
- 与 `axi_cut` 的区别：FIFO 可以吸收突发流量，cut 仅做时序优化

**应用场景：**
- 吸收主从速度不匹配的突发流量
- 增加未完成事务容量
- 在 crossbar 内部缓冲

**对应源码：** `src/axi_fifo.sv`

**前置依赖：** 第 1-6 课

---

### 第 8 课：错误从机与仿真内存

**学习目标：** 理解 AXI 从机的基本应答行为

**知识点：**

#### 8.1 错误从机 — `axi_err_slv`
- 对所有请求返回错误响应（可配置 SLVERR 或 DECERR）
- 参数 `Resp`：响应类型；`RespData`：读数据填充值
- 参数 `ATOPs`：是否支持原子操作
- 参数 `MaxTrans`：最大未完成事务数
- 内部使用 FIFO 排队 B 和 R 响应
- 用途：crossbar 中未映射地址空间的默认从机

#### 8.2 零内存 — `axi_zero_mem`
- 读操作总是返回 0，写操作静默丢弃
- 用途：/dev/zero 的硬件等价物

#### 8.3 仿真内存 — `axi_sim_mem`（仅仿真）
- 无限大关联数组存储
- 支持多端口访问
- 可配置未初始化数据警告
- 支持错误注入（`rerror_i`, `werror_i`）
- 可配置应用延迟和采样延迟

**课后练习：** 在测试中实例化 `axi_err_slv`，发送一个写事务并观察 B 响应

**对应源码：** `src/axi_err_slv.sv`, `src/axi_zero_mem.sv`, `src/axi_sim_mem.sv`

**前置依赖：** 第 1-5 课

---

## 单元三：解复用器 — Demux（第 9-11 课）

> 目标：深入理解一主分多从的路由机制，这是 crossbar 的核心组件之一。

---

### 第 9 课：ID 计数器 — `axi_demux_id_counters`

**学习目标：** 理解 Demux 中防死锁的 ID 追踪机制

**知识点：**
- 为什么需要 ID 计数器：同一 ID 的事务必须路由到同一从机（保证有序性）
- 每个 ID 维护一个计数器，记录该 ID 有多少未完成事务
- 每个 ID 记录其路由目标（选择了哪个从机端口）
- 参数：
  - `AxiIdBits`：参与查找的 ID 位数（可只用低位以节省面积）
  - `CounterWidth`：计数器位宽（决定最大未完成事务数）
- 操作接口：push（新事务）、inject（ATOP 注入）、pop（事务完成）
- `any_outstanding_trx_o`：是否有任何未完成事务

**设计要点：** 当一个 ID 已有未完成事务时，新事务必须路由到相同目标

**对应源码：** `src/axi_demux_id_counters.sv`

**前置依赖：** 第 1-5 课

---

### 第 10 课：简化解复用器 — `axi_demux_simple`

**学习目标：** 理解 Demux 的核心路由逻辑（不含 spill register）

**知识点：**
- 输入：一个 AXI slave port + 选择信号（`slv_aw_select_i`, `slv_ar_select_i`）
- 输出：多个 AXI master port
- AW/AR 通道：根据选择信号路由到对应端口
- W 通道：没有地址信息，需通过 ID 计数器追踪 AW 的路由结果
- B/R 通道：从多个从机端口收集响应，用 round-robin 仲裁返回
- 单端口情况下的 passthrough 优化
- 参数 `UniqueIds`：所有事务 ID 唯一时的优化路径

**架构图：**
```
                    ┌──────────────────────┐
slv_req ──────────►│                      ├──► mst_req[0]
slv_aw_select ────►│  axi_demux_simple    ├──► mst_req[1]
slv_ar_select ────►│                      ├──► mst_req[2]
slv_resp ◄─────────│   (ID counters +     │◄── mst_resp[0]
                   │    RR arbiter)       │◄── mst_resp[1]
                   └──────────────────────┘◄── mst_resp[2]
```

**对应源码：** `src/axi_demux_simple.sv`

**前置依赖：** 第 1-9 课

---

### 第 11 课：完整解复用器 — `axi_demux`

**学习目标：** 理解带 spill register 的完整 Demux 设计

**知识点：**
- 在 `axi_demux_simple` 基础上增加了每通道可选的 spill register
- Spill 参数：`SpillAw`, `SpillW`, `SpillB`, `SpillAr`, `SpillR`
- 外部提供选择信号（而非内部地址译码）— 解耦设计
- 与地址映射的关系：crossbar 负责译码，demux 只负责路由
- 参数 `MaxTrans`：每个 ID 最大未完成事务数
- 参数 `AxiLookBits`：ID 查找使用的位数（面积 vs. 灵活性权衡）

**文档：** `doc/axi_demux.md`（含架构图 `doc/axi_demux.png`）

**关键设计决策：**
1. 选择信号由外部提供 → 模块可复用于不同地址映射方案
2. W 通道路由依赖 AW 的路由历史 → 需要 ID 计数器
3. Spill register 可逐通道开关 → 灵活的时序/面积权衡

**课后练习：** 画出一个 1-to-4 Demux 中，写事务（AW→W→B）和读事务（AR→R）的完整数据流

**对应源码：** `src/axi_demux.sv`, `doc/axi_demux.md`

**前置依赖：** 第 1-10 课

---

## 单元四：复用器 — Mux（第 12-14 课）

> 目标：理解多主合一的仲裁和 ID 管理机制。

---

### 第 12 课：AXI 复用器 — `axi_mux`

**学习目标：** 理解多个主机共享一个从机端口的仲裁机制

**知识点：**
- 输入：多个 AXI slave port（从多个 Master 接收请求）
- 输出：一个 AXI master port（向一个 Slave 发送请求）
- ID 宽度扩展：在 ID 高位添加 slave port 索引位
  ```
  Master ID Width = Slave ID Width + $clog2(NoSlvPorts)
  ```
- AW/AR 通道：round-robin 仲裁选择一个 slave port
- W 通道：跟随 AW 仲裁结果，按顺序传输
- B/R 通道：根据 ID 高位路由回正确的 slave port
- 参数 `MaxWTrans`：最大同时进行的写事务数
- 参数 `FallThrough`：FIFO 直通模式

**与 Demux 的对偶关系：**
```
Demux: 1 slave port → N master ports（地址路由）
Mux:   N slave ports → 1 master port（仲裁合并）
```

**文档：** `doc/axi_mux.md`

**对应源码：** `src/axi_mux.sv`

**前置依赖：** 第 1-5 课

---

### 第 13 课：AXI4-Lite 复用器与解复用器

**学习目标：** 理解 Lite 协议的简化互连模块

**知识点：**

#### 13.1 `axi_lite_mux` — Lite 复用器
- 无 ID 字段 → 不需要 ID 扩展
- 仍使用 round-robin 仲裁
- 每通道可选 spill register
- 参数 `MaxTrans`：每 slave port 最大未完成事务数

#### 13.2 `axi_lite_demux` — Lite 解复用器
- 无 ID 字段 → 不需要 ID 计数器
- 使用 FIFO 记录路由历史（替代 ID 追踪）
- 选择信号仍由外部提供

#### 13.3 与 AXI4 版本的对比
| 特性 | AXI4 Mux | AXI4-Lite Mux |
|------|----------|---------------|
| ID 扩展 | 需要 | 不需要 |
| W 通道跟踪 | 通过 AW ID | 通过 FIFO |
| 参数复杂度 | 较高 | 较低 |
| 支持乱序 | 是 | 否 |

**对应源码：** `src/axi_lite_mux.sv`, `src/axi_lite_demux.sv`

**前置依赖：** 第 1-5, 12 课

---

### 第 14 课：仲裁策略与响应路由

**学习目标：** 深入理解互连中的仲裁和响应回传机制

**知识点：**
- Round-Robin 仲裁器的实现（来自 `common_cells` 库）
- 仲裁的公平性：每个 Master 获得均等的访问机会
- W 通道仲裁的特殊性：必须跟随 AW 的仲裁结果（写数据必须与写地址配对）
- B/R 响应路由：
  - 在 Mux 中：根据 ID 高位（端口索引）路由
  - 在 Demux 中：round-robin 从多个端口收集
- Mux-tree 构建：多路响应归并的硬件结构
- 仲裁锁定（locking）：burst 传输期间锁定仲裁结果

**综合练习：** 画出 2-Master-to-1-Slave Mux 中以下场景的时序：
1. 两个 Master 同时发起写请求
2. 一个写事务进行中，另一个 Master 发起读请求

**前置依赖：** 第 12-13 课

---

## 单元五：交叉开关互连（第 15-17 课）

> 目标：理解仓库中最复杂的模块 — 全连接交叉开关的架构与配置。

---

### 第 15 课：Crossbar 架构原理 — `axi_xbar_unmuxed`

**学习目标：** 理解 Crossbar 的 Demux 侧架构

**知识点：**
- Crossbar = N 个 Demux + M 个 Mux
- `axi_xbar_unmuxed`：只包含 Demux 侧，输出未经 Mux 合并
- 地址译码：将地址映射规则转化为 Demux 的选择信号
- 地址映射规则 `rule_t`：
  ```systemverilog
  typedef struct packed {
      int unsigned idx;     // 目标从机端口索引
      addr_t       start;   // 起始地址（含）
      addr_t       end;     // 结束地址（不含）
  } rule_t;
  ```
- 默认从机端口：地址不匹配任何规则时的目标
- 错误从机：对默认路由的请求返回 DECERR
- ATOP 过滤器的集成位置

**架构图：**
```
         ┌─────────────────────────────────┐
         │       axi_xbar_unmuxed          │
         │                                 │
M0 ─────►│ [demux[0]] ──┬── to mux[0] ───►│──► S0 端口组
         │              ├── to mux[1] ───►│──► S1 端口组
         │              └── to mux[2] ───►│──► S2 端口组
         │                                 │
M1 ─────►│ [demux[1]] ──┬── to mux[0] ───►│──► S0 端口组
         │              ├── to mux[1] ───►│──► S1 端口组
         │              └── to mux[2] ───►│──► S2 端口组
         └─────────────────────────────────┘
```

**对应源码：** `src/axi_xbar_unmuxed.sv`

**前置依赖：** 第 1-14 课

---

### 第 16 课：全连接交叉开关 — `axi_xbar`

**学习目标：** 掌握 Crossbar 的完整配置和使用方法

**知识点：**

#### 16.1 配置结构体 `xbar_cfg_t`
```systemverilog
localparam axi_pkg::xbar_cfg_t Cfg = '{
    NoSlvPorts:         2,         // Master 数量（连接在 slave port）
    NoMstPorts:         3,         // Slave 数量（连接在 master port）
    MaxMstTrans:        8,         // 每个 master port 最大未完成事务
    MaxSlvTrans:        8,         // 每个 slave port 最大未完成事务
    FallThrough:        1'b0,      // FIFO 直通模式
    LatencyMode:        axi_pkg::CUT_ALL_AX,  // 延迟模式
    PipelineStages:     0,         // 额外流水线级数
    AxiIdWidthSlvPorts: 4,         // slave port ID 宽度
    AxiIdUsedSlvPorts:  4,         // 实际使用的 ID 位数
    UniqueIds:          1'b0,      // ID 是否保证唯一
    AxiAddrWidth:       32,        // 地址宽度
    AxiDataWidth:       64,        // 数据宽度
    NoAddrRules:        3          // 地址映射规则数量
};
```

#### 16.2 延迟模式（LatencyMode）
| 模式 | 说明 |
|------|------|
| `NO_LATENCY` | 无额外寄存器 |
| `CUT_SLV_AX` | slave port 的 AW/AR 插入寄存器 |
| `CUT_MST_AX` | master port 的 AW/AR 插入寄存器 |
| `CUT_ALL_AX` | 所有 AW/AR 插入寄存器 |
| `CUT_SLV_PORTS` | slave port 全通道插入寄存器 |
| `CUT_MST_PORTS` | master port 全通道插入寄存器 |
| `CUT_ALL_PORTS` | 全部端口全通道插入寄存器 |

#### 16.3 连通性矩阵（Connectivity）
- 默认全连接（每个 Master 可访问每个 Slave）
- 可通过 `Connectivity` 参数限制连通性，节省面积

#### 16.4 ATOP 支持
- 参数 `ATOPs`：是否使能原子操作
- 启用时在每个 slave port 前插入 `axi_atop_filter`

**文档：** `doc/axi_xbar.md`（含架构图 `doc/axi_xbar.png`）

**课后练习：** 配置一个 3 Master × 2 Slave 的 crossbar，定义地址映射并分析数据流

**对应源码：** `src/axi_xbar.sv`, `doc/axi_xbar.md`

**前置依赖：** 第 1-15 课

---

### 第 17 课：AXI4-Lite Crossbar 与 Crosspoint

**学习目标：** 理解 Lite Crossbar 和高层封装模块

**知识点：**

#### 17.1 `axi_lite_xbar` — Lite 交叉开关
- 架构与 `axi_xbar` 类似：Demux + Mux
- 简化：无 ID 管理、无 Burst、无 ATOP
- 同样支持地址映射规则和默认从机
- 配置结构体使用同一个 `xbar_cfg_t`

#### 17.2 `axi_xp` — Crosspoint 封装（Level 6）
- 在 `axi_xbar` 之上的高层封装
- 集成了 ID 宽度转换（`axi_iw_converter`）
- 集成了 ATOP 过滤
- 用于构建多级互连网络

#### 17.3 互连生成器 — `scripts/axi_intercon_gen.py`
- Python 脚本，根据配置自动生成互连 RTL
- 输入：JSON/YAML 格式的拓扑描述
- 输出：参数化的 SystemVerilog 互连模块

**文档：** `doc/axi_lite_xbar.md`

**对应源码：** `src/axi_lite_xbar.sv`, `src/axi_xp.sv`

**前置依赖：** 第 1-16 课

---

## 单元六：协议与数据转换（第 18-21 课）

> 目标：掌握不同协议和数据位宽之间的转换技术。

---

### 第 18 课：AXI4-Lite ↔ AXI4 协议转换

**学习目标：** 理解两种 AXI 协议之间的适配

**知识点：**

#### 18.1 `axi_lite_to_axi` — Lite 转 Full AXI4
- 简单的信号扩展：
  - 填充默认值：burst=INCR, len=0（单拍）, size=log2(数据宽度)
  - ID 和 User 字段清零
  - Cache 和 Prot 直接映射
- 零延迟（纯组合逻辑）

#### 18.2 `axi_to_axi_lite` — Full AXI4 转 Lite
- 复杂得多，内部集成多个子模块：
  1. `axi_atop_filter`：原子操作转错误响应
  2. `axi_burst_splitter`：多拍 burst 拆为单拍
  3. ID 反射队列：保存请求 ID，在响应中回填
- 参数：`AxiMaxWriteTxns`, `AxiMaxReadTxns`
- 为什么复杂？Lite 不支持 burst 和 ID，需要拆分和追踪

**对比：**
```
Lite → AXI4:  简单扩展（组合逻辑）
AXI4 → Lite:  复杂转换（需要 burst 拆分 + ID 追踪 + ATOP 过滤）
```

**对应源码：** `src/axi_lite_to_axi.sv`, `src/axi_to_axi_lite.sv`

**前置依赖：** 第 1-5 课

---

### 第 19 课：AXI4-Lite 到 APB4 桥接

**学习目标：** 理解 AXI 到低速外设总线的转换

**知识点：**
- APB4 协议简介：两阶段传输（SETUP → ACCESS）
- `axi_lite_to_apb`：支持多个 APB 从机端口
- 内部地址译码：将 AXI-Lite 请求路由到正确的 APB 从机
- 流水线选项：`PipelineRequest`, `PipelineResponse`
- 错误处理：APB SLVERR 映射到 AXI SLVERR
- 典型应用：SoC 中连接 UART, GPIO, Timer 等低速外设

**架构：**
```
AXI4-Lite ──► [axi_lite_to_apb] ──┬──► APB Slave 0 (UART)
                                   ├──► APB Slave 1 (GPIO)
                                   └──► APB Slave 2 (Timer)
```

**对应源码：** `src/axi_lite_to_apb.sv`

**前置依赖：** 第 1-5 课

---

### 第 20 课：数据位宽上转换 — `axi_dw_upsizer`

**学习目标：** 理解窄数据通路到宽数据通路的转换

**知识点：**
- 场景：32 位 Master 连接 64 位 Slave
- 写路径：多个窄拍合并为一个宽拍，拼接 strobe
- 读路径：一个宽拍拆分为多个窄拍返回
- Burst 长度调整：宽端 burst 更短
- WRAP burst 限制：不支持，返回错误
- 参数：`AxiSlvPortDataWidth`, `AxiMstPortDataWidth`, `AxiMaxReads`

**示例：32b→64b 写事务**
```
窄端（32b）: beat0[31:0] + beat1[31:0] → 宽端（64b）: beat0[63:0]
strobe:     4'b1111    + 4'b1111     →              8'b11111111
```

**对应源码：** `src/axi_dw_upsizer.sv`

**前置依赖：** 第 1-5 课

---

### 第 21 课：数据位宽下转换与统一转换器

**学习目标：** 理解宽转窄和自适应位宽转换

**知识点：**

#### 21.1 `axi_dw_downsizer` — 宽转窄
- 场景：64 位 Master 连接 32 位 Slave
- 写路径：一个宽拍拆分为多个窄拍
- 读路径：多个窄拍合并为一个宽拍返回
- Burst 长度增加，地址对齐处理

#### 21.2 `axi_dw_converter` — 统一转换器
- 自动选择 upsizer 或 downsizer
- 位宽相同时直连（passthrough）
- 判断逻辑：
  ```systemverilog
  if (SlvPortDataWidth == MstPortDataWidth) → passthrough
  if (SlvPortDataWidth <  MstPortDataWidth) → upsizer
  if (SlvPortDataWidth >  MstPortDataWidth) → downsizer
  ```

#### 21.3 `axi_lite_dw_converter` — Lite 版位宽转换
- 更简单：Lite 只有单拍，不涉及 burst 调整

**课后练习：** 分析 64b Master 通过 downsizer 写入 32b Slave 时，一个 4 拍 burst 如何变成 8 拍

**对应源码：** `src/axi_dw_downsizer.sv`, `src/axi_dw_converter.sv`

**前置依赖：** 第 1-5, 20 课

---

## 单元七：ID 管理与 Burst 处理（第 22-26 课）

> 目标：掌握 AXI 中最精妙的两个主题 — ID 空间管理和 Burst 变换。

---

### 第 22 课：AXI ID 机制深入理解

**学习目标：** 理解 AXI ID 的语义和乱序完成

**知识点：**
- AXI ID 的作用：标识事务来源，允许乱序完成
- 同 ID 事务的有序保证：同一 ID 的响应必须按请求顺序返回
- 不同 ID 的乱序能力：不同 ID 的响应可以交叉返回
- ID 宽度影响：宽 ID → 更多并发事务 → 更大面积开销
- 在互连中 ID 如何变化：Mux 扩展 ID，Demux 追踪 ID

**示例：乱序完成**
```
请求顺序: AR(id=0, addr=A) → AR(id=1, addr=B) → AR(id=0, addr=C)
可能的响应: R(id=1, data_B) → R(id=0, data_A) → R(id=0, data_C)
                                                    ↑ id=0 保序
```

**前置依赖：** 第 1-5 课

---

### 第 23 课：ID 前置与 ID 重映射

**学习目标：** 掌握两种 ID 变换技术

**知识点：**

#### 23.1 `axi_id_prepend` — ID 前置
- 最简单的 ID 变换：在高位添加固定位
- 不需要查找表，纯组合逻辑
- 用途：Mux 内部用于区分来源端口
- 响应路径：从高位提取端口索引，移除后返回原始 ID

#### 23.2 `axi_id_remap` — ID 重映射
- 复杂的 ID 变换：将稀疏的宽 ID 映射到紧凑的窄 ID
- 内部维护查找表（lookup table）
- 参数 `AxiSlvPortMaxUniqIds`：最大同时活跃的唯一 ID 数
- 参数 `AxiMaxTxnsPerId`：每个 ID 最大未完成事务数
- 映射保证：不同的 slave ID 映射到不同的 master ID

**应用场景：**
```
Master (ID=8bit) → [axi_id_remap] → Slave (ID=4bit)
256 种可能的 ID → 映射到 16 种 ID（需追踪映射关系）
```

**对应源码：** `src/axi_id_prepend.sv`, `src/axi_id_remap.sv`

**前置依赖：** 第 22 课

---

### 第 24 课：ID 串行化与 ID 宽度转换

**学习目标：** 理解 ID 串行化和通用 ID 宽度适配

**知识点：**

#### 24.1 `axi_serializer` — ID 串行化
- 将所有事务强制使用 ID=0
- 内部使用 FIFO 队列保存原始 ID
- 响应返回时从队列中恢复原始 ID
- 读写通道独立的 FIFO
- ATOP 事务需要特殊状态机处理
- 用途：连接只支持单 ID 的从机

#### 24.2 `axi_id_serialize` — 更完整的 ID 串行化
- 支持不同 ID 的事务排队
- 比 `axi_serializer` 功能更丰富

#### 24.3 `axi_iw_converter` — ID 宽度转换器
- 自动选择最优的 ID 转换策略：
  ```
  if (SlvIdWidth <= MstIdWidth) → axi_id_prepend（简单扩展）
  if (SlvIdWidth >  MstIdWidth) → axi_id_remap（需要查找表）
  ```
- 统一接口，屏蔽底层实现细节

**对应源码：** `src/axi_serializer.sv`, `src/axi_id_serialize.sv`, `src/axi_iw_converter.sv`

**前置依赖：** 第 22-23 课

---

### 第 25 课：Burst 拆分 — `axi_burst_splitter`

**学习目标：** 理解多拍 Burst 如何拆分为单拍事务

**知识点：**
- 为什么需要拆分：某些从机不支持 burst（如 AXI-Lite、简单 SRAM）
- 拆分过程：
  ```
  原始: AW(addr=0x100, len=3, size=2) → 4 拍 burst
  拆分: AW(addr=0x100, len=0) → AW(addr=0x104, len=0) → AW(addr=0x108, len=0) → AW(addr=0x10C, len=0)
  ```
- INCR burst 的地址递增计算
- FIXED burst 的地址保持不变
- WRAP burst：不支持，返回错误
- W 通道：每拍的 wlast 都设为 1
- B 通道：只在所有子事务完成后返回一个 B 响应
- R 通道：收集所有子事务的 R 响应，合并为原始 burst

**参数：**
- `MaxReadTxns`, `MaxWriteTxns`：最大未完成事务数
- `FullBW`：全带宽模式

**变体：** `axi_burst_splitter_gran` — 运行时可配置拆分粒度

**对应源码：** `src/axi_burst_splitter.sv`, `src/axi_burst_splitter_gran.sv`

**前置依赖：** 第 1-5, 22 课

---

### 第 26 课：Burst 解绕 — `axi_burst_unwrap`

**学习目标：** 理解 WRAP burst 到 INCR burst 的转换

**知识点：**
- WRAP burst 的语义：地址在边界处回绕
- 示例（4 拍 WRAP，起始地址 0x10，size=4 字节，wrap boundary=0x10）：
  ```
  WRAP: 0x10 → 0x14 → 0x18 → 0x1C → 0x10 (回绕)
  转换为 INCR: 0x10 → 0x14 → 0x18 → 0x1C (地址不回绕，可能需要调整起始地址)
  ```
- 地址计算：使用 `axi_pkg` 中的 `wrap_boundary()` 和 `beat_addr()`
- 用途：很多简单从机不支持 WRAP burst
- 参数：`MaxTrans`

**对应源码：** `src/axi_burst_unwrap.sv`

**前置依赖：** 第 1-5, 25 课

---

## 单元八：系统级功能与验证（第 27-30 课）

> 目标：掌握系统级 AXI 功能和验证方法论。

---

### 第 27 课：原子操作过滤 — `axi_atop_filter`

**学习目标：** 理解 AXI5 原子操作及其过滤机制

**知识点：**
- AXI5 ATOP 类型：
  - AtomicStore：原子写（无返回数据）
  - AtomicLoad：原子读-修改-写（返回旧值）
  - AtomicSwap：原子交换
  - AtomicCompare：原子比较并交换
- ATOP 编码在 `aw.atop` 字段（6 位）
- `axi_atop_filter` 的作用：为不支持原子操作的从机过滤 ATOP 请求
- 内部状态机：
  - 写路径状态：W_FEEDTHROUGH → BLOCK_AW → ABSORB_W → INJECT_B
  - 读路径状态：R_FEEDTHROUGH → INJECT_R
- 过滤后将 `atop` 字段清零
- ATOP 请求返回 SLVERR 响应

**对应源码：** `src/axi_atop_filter.sv`

**前置依赖：** 第 1-5 课

---

### 第 28 课：隔离与节流控制

**学习目标：** 理解系统级 AXI 管理功能

**知识点：**

#### 28.1 `axi_isolate` — 端口隔离
- 功能：将一个 AXI slave port 与其下游从机优雅隔离
- 隔离过程：
  1. 拉高 `isolate_i` 信号
  2. 等待所有未完成事务完成
  3. 后续新请求被拦截并返回错误响应
  4. `isolated_o` 指示隔离完成
- 参数 `TerminateTransaction`：是否强制终止进行中的事务
- 用途：电源管理（关断从机前隔离）、动态重配置

#### 28.2 `axi_throttle` — 事务节流
- 功能：限制 AXI 端口的未完成事务数量
- 参数：读/写方向独立的限制值
- 用途：QoS（服务质量）控制、防止某个 Master 独占带宽

#### 28.3 `axi_modify_address` — 地址修改
- 功能：动态修改事务的 AW/AR 地址
- 纯组合逻辑，零延迟
- 用途：地址偏移、重映射

**对应源码：** `src/axi_isolate.sv`, `src/axi_throttle.sv`, `src/axi_modify_address.sv`

**前置依赖：** 第 1-5 课

---

### 第 29 课：AXI 到 Memory 接口转换

**学习目标：** 理解 AXI 与 SRAM 接口的桥接

**知识点：**

#### 29.1 `axi_to_mem` — 基础 Memory 接口
- AXI slave port → 简单的 req/gnt + addr/wdata/rdata 接口
- 内部 burst 展开：自动将 burst 转为逐拍内存访问
- 读写仲裁：读优先或写优先可配
- 参数：`BufDepth`（响应缓冲深度）

#### 29.2 `axi_to_mem_banked` — 分体内存接口
- 输出多个 memory bank 端口
- 地址交织（interleaving）支持
- 并行访问提升带宽

#### 29.3 `axi_to_mem_interleaved` — 交织内存接口
- 按地址低位分配到不同 bank
- 适用于均匀访问模式

#### 29.4 `axi_from_mem` — Memory 到 AXI
- 方向相反：SRAM 接口 → AXI master port
- 用于 DMA、加速器等需要发起 AXI 事务的模块

#### 29.5 辅助模块
- `axi_rw_join`：将独立的读/写通道合并为完整 AXI 端口
- `axi_rw_split`：将完整 AXI 端口拆分为独立的读/写通道

**课后练习：** 设计一个简单的 SoC 子系统：CPU (AXI Master) → Crossbar → SRAM (通过 axi_to_mem) + UART (通过 axi_to_axi_lite → axi_lite_to_apb)

**对应源码：** `src/axi_to_mem.sv`, `src/axi_to_mem_banked.sv`, `src/axi_from_mem.sv`, `src/axi_rw_join.sv`, `src/axi_rw_split.sv`

**前置依赖：** 第 1-5 课

---

### 第 30 课：验证方法论与测试基础设施

**学习目标：** 掌握 AXI 验证的完整方法论

**知识点：**

#### 30.1 验证类 — `axi_test.sv`

| 类 | 功能 | 用途 |
|----|------|------|
| `axi_driver` | 低层 beat-by-beat 驱动 | 精确控制每个信号 |
| `axi_rand_master` | 随机化事务发生器 | 功能验证主力 |
| `axi_rand_slave` | 随机化响应从机 | 配合 master 使用 |
| `axi_scoreboard` | 记分板（黄金参考模型） | 数据一致性检查 |
| `axi_file_master` | 文件驱动的测试序列 | 回归测试 |
| `axi_chan_logger` | 事务日志 | 调试辅助 |

#### 30.2 随机化验证方法
- 受约束随机（Constrained Random）：
  ```systemverilog
  // axi_rand_master 可配置的约束：
  // - 地址范围
  // - Burst 类型分布
  // - 数据模式
  // - 延迟分布
  // - ATOP 概率
  ```
- 功能覆盖率驱动

#### 30.3 比较与监控
- `axi_bus_compare`：双端口 AXI 总线比较（可综合）
- `axi_slave_compare`：从机行为比较
- `axi_chan_compare`：通道级比较（仅仿真）
- `axi_dumper`：波形 dump + `scripts/axi_dumper_interpret.py` 解析

#### 30.4 Testbench 结构
以 `tb_axi_xbar.sv` 为例：
```
                    ┌───────────────────┐
rand_master[0] ────►│                   │──── rand_slave[0]
rand_master[1] ────►│   axi_xbar (DUT)  │──── rand_slave[1]
rand_master[2] ────►│                   │──── rand_slave[2]
                    └───────────────────┘
                           │
                    scoreboard (检查数据一致性)
```

#### 30.5 运行测试
```bash
make compile.log           # 编译
make sim-axi_xbar.log      # 运行 xbar 测试
make sim_all                # 运行全部测试
scripts/run_vsim.sh         # Questasim 仿真脚本
scripts/run_verilator.sh    # Verilator lint
```

**综合实践：** 编写一个简单的 testbench，使用 `axi_rand_master` 和 `axi_sim_mem` 测试 `axi_cut` 模块

**对应源码：** `src/axi_test.sv`, `test/tb_axi_xbar.sv`

**前置依赖：** 第 1-29 课（综合应用）

---

## 附录 A：模块依赖关系图

```
Level 0: axi_pkg
            │
Level 1: axi_intf
            │
Level 2: axi_cut ──── axi_demux_simple ──── axi_mux
         axi_err_slv   axi_id_prepend       axi_atop_filter
         axi_fifo       axi_lite_to_axi      axi_modify_address
            │
Level 3: axi_demux ── axi_burst_splitter ── axi_cdc
         axi_dw_*     axi_id_serialize      axi_to_mem
         axi_to_axi_lite
            │
Level 4: axi_iw_converter ── axi_lite_xbar ── axi_xbar_unmuxed
            │
Level 5: axi_xbar
            │
Level 6: axi_xp
```

## 附录 B：推荐学习时间安排

| 阶段 | 课程 | 建议时长 | 说明 |
|------|------|----------|------|
| 基础 | 第 1-5 课 | 1-2 周 | 反复阅读，确保理解透彻 |
| 入门 | 第 6-8 课 | 3-4 天 | 代码量少，容易上手 |
| 核心 | 第 9-17 课 | 2-3 周 | 最重要的部分，需配合仿真 |
| 转换 | 第 18-21 课 | 1-2 周 | 工程实用性强 |
| 进阶 | 第 22-26 课 | 2-3 周 | 概念较抽象，需仔细推导 |
| 系统 | 第 27-30 课 | 1-2 周 | 结合实际 SoC 设计理解 |

## 附录 C：课后项目建议

1. **Mini Crossbar**：自己实现一个 2×2 的简化 crossbar，不参考源码
2. **SoC 子系统**：用仓库模块搭建 CPU + SRAM + UART 子系统
3. **自定义从机**：设计一个支持 AXI4 接口的 DMA 控制器
4. **性能分析**：修改 crossbar 参数，用仿真测量吞吐量和延迟变化
