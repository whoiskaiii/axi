# AXI 总线协议学习指南

> 基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库（v0.39.9）

本指南为深入学习 AXI 总线协议提供了一条系统化的学习路线，结合本仓库中的 64 个 SystemVerilog 模块，从基础到进阶逐步展开。

---

## 目录

- [仓库概览](#仓库概览)
- [第一阶段：AXI 协议基础](#第一阶段axi-协议基础)
- [第二阶段：核心定义与类型系统](#第二阶段核心定义与类型系统)
- [第三阶段：从简单模块到复杂模块](#第三阶段从简单模块到复杂模块)
- [第四阶段：验证与仿真](#第四阶段验证与仿真)
- [第五阶段：进阶主题](#第五阶段进阶主题)
- [实践建议](#实践建议)
- [核心设计理念](#核心设计理念)
- [参考资源](#参考资源)

---

## 仓库概览

本仓库是由 ETH Zurich PULP 团队维护的生产级 AXI 总线 IP 库，主要特点：

- **64 个 SystemVerilog 模块**，按 6 个依赖层级组织
- **完整支持 AXI4 和 AXI4-Lite** 协议，包含 AXI5 原子操作扩展
- **模块化设计**，可自由组合构建任意互连拓扑
- **完善的验证基础设施**，含随机化测试和记分板

### 目录结构

```
axi/
├── src/                    # 64 个 SystemVerilog 源文件（可综合 + 仿真）
├── include/axi/            # 头文件：类型定义、信号赋值、端口定义宏
├── test/                   # 测试平台和波形配置文件
├── doc/                    # 模块文档和架构图
├── scripts/                # 构建、综合、仿真脚本
├── Bender.yml              # 依赖管理
├── Makefile                # 构建自动化
└── axi.core                # FuseSoC 集成文件
```

### 依赖层级

```
Level 0: axi_pkg           （无依赖，基础类型定义）
Level 1: axi_intf           （接口定义）
Level 2: axi_cut, axi_mux, axi_demux_simple ...（基础模块）
Level 3: axi_burst_splitter, axi_cdc, axi_demux ...（组合模块）
Level 4: axi_iw_converter, axi_lite_xbar, axi_xbar_unmuxed ...（高级模块）
Level 5: axi_xbar           （全连接交叉开关）
Level 6: axi_xp             （交叉点封装）
```

---

## 第一阶段：AXI 协议基础

> 目标：在看代码之前，先理解 AXI 协议本身。

### 1.1 AXI 协议核心概念

AXI（Advanced eXtensible Interface）是 ARM AMBA 总线家族中面向高性能场景的协议。本仓库遵循 **AMBA AXI and ACE Protocol Specification, Issue F.b**。

#### 五个独立通道

| 通道 | 缩写 | 方向 | 功能 |
|------|------|------|------|
| Write Address | AW | Master → Slave | 写事务的地址和控制信息 |
| Write Data | W | Master → Slave | 写数据和字节选通 |
| Write Response | B | Slave → Master | 写事务的完成响应 |
| Read Address | AR | Master → Slave | 读事务的地址和控制信息 |
| Read Data | R | Slave → Master | 读数据和响应 |

#### 握手机制（Handshake）

每个通道都使用 `valid` / `ready` 握手：
- **发送方**拉高 `valid` 表示数据有效
- **接收方**拉高 `ready` 表示可以接收
- 当 `valid` 和 `ready` 在时钟上升沿同时为高时，完成一次**握手（Handshake）**

```
         ┌───┐   ┌───┐   ┌───┐   ┌───┐
 clk     │   │   │   │   │   │   │   │
      ───┘   └───┘   └───┘   └───┘   └───
              ┌───────────────┐
 valid  ──────┘               └───────────
                      ┌───────┐
 ready  ──────────────┘       └───────────
                      ▲
                  Handshake!
```

#### Burst 类型

| 类型 | 值 | 说明 |
|------|---|------|
| FIXED | 2'b00 | 地址不变，用于 FIFO 访问 |
| INCR | 2'b01 | 地址递增，最常用 |
| WRAP | 2'b10 | 地址回绕，用于缓存行填充 |

#### 响应类型

| 类型 | 值 | 说明 |
|------|---|------|
| OKAY | 2'b00 | 正常完成 |
| EXOKAY | 2'b01 | 独占访问成功 |
| SLVERR | 2'b10 | 从机错误 |
| DECERR | 2'b11 | 译码错误（地址无法匹配） |

### 1.2 关键术语

参见 `doc/README.md` 中的定义：

- **Handshake**：valid 和 ready 同时为高的时钟沿
- **In Flight**：AX 握手已完成但最后一个响应尚未返回的事务
- **Pending**：valid 为高但 ready 为低的状态

### 1.3 AXI4 vs AXI4-Lite

| 特性 | AXI4 | AXI4-Lite |
|------|------|-----------|
| Burst 支持 | 是（最多 256 拍） | 否（仅单拍） |
| ID 信号 | 是 | 否 |
| User 信号 | 是 | 否 |
| 数据位宽 | 任意（通常 32/64/128/256/512） | 32 或 64 |
| 适用场景 | 高性能数据通路 | 低速控制/配置寄存器 |

---

## 第二阶段：核心定义与类型系统

> 目标：理解仓库的类型系统和编码风格，这是阅读所有模块的基础。

### 2.1 AXI Package — `src/axi_pkg.sv`

**这是第一个要读的文件。** 包含：

- AXI 信号位宽常量（`BURST_WIDTH`, `RESP_WIDTH` 等）
- 枚举类型（`burst_t`, `resp_t`, `cache_t` 等）
- Burst 类型常量（`BURST_FIXED`, `BURST_INCR`, `BURST_WRAP`）
- 响应常量（`RESP_OKAY`, `RESP_EXOKAY`, `RESP_SLVERR`, `RESP_DECERR`）
- ATOP（原子操作）相关定义
- 交叉开关配置结构体 `xbar_cfg_t`
- 地址计算辅助函数

**学习重点：** 关注每个类型的位宽和含义，这些在所有模块中反复使用。

### 2.2 Typedef 宏 — `include/axi/typedef.svh`

定义了生成 AXI channel struct 的宏：

```systemverilog
// 单独定义各通道 struct
`AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)
`AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t)
`AXI_TYPEDEF_B_CHAN_T(b_chan_t, id_t, user_t)
`AXI_TYPEDEF_AR_CHAN_T(ar_chan_t, addr_t, id_t, user_t)
`AXI_TYPEDEF_R_CHAN_T(r_chan_t, data_t, id_t, user_t)

// 一次性生成所有 struct
`AXI_TYPEDEF_ALL(axi, addr_t, id_t, data_t, strb_t, user_t)
```

**学习重点：** 理解 struct-based 端口风格，这是本仓库的核心设计模式。

### 2.3 赋值宏 — `include/axi/assign.svh`

提供接口与 struct 之间的信号赋值宏：

```systemverilog
`AXI_ASSIGN(slave, master)          // 接口 → 接口
`AXI_ASSIGN_TO_REQ(req, axi_if)    // 接口 → struct
`AXI_ASSIGN_FROM_REQ(axi_if, req)  // struct → 接口
```

### 2.4 接口定义 — `src/axi_intf.sv`

SystemVerilog `interface` 定义，包含 Master 和 Slave modport：

```systemverilog
interface AXI_BUS #(
    parameter int unsigned AXI_ADDR_WIDTH = 0,
    parameter int unsigned AXI_DATA_WIDTH = 0,
    parameter int unsigned AXI_ID_WIDTH   = 0,
    parameter int unsigned AXI_USER_WIDTH = 0
);
    // ... 五个通道的所有信号 ...
    modport Master ( /* ... */ );
    modport Slave  ( /* ... */ );
endinterface
```

**学习重点：** 对比 interface 风格和 struct 风格的优劣。

---

## 第三阶段：从简单模块到复杂模块

> 目标：按复杂度递增的顺序阅读模块，逐步建立对 AXI 互连设计的理解。

### 3.1 入门模块

#### `axi_cut.sv` — 寄存器切割（Pipeline Stage）

- **功能：** 在 AXI 通道中插入寄存器级，打断组合逻辑路径
- **复杂度：** 低
- **学习要点：**
  - 如何在不破坏协议的前提下插入流水线级
  - spill register 的概念
  - 每个通道独立处理

#### `axi_fifo.sv` — 通道 FIFO

- **功能：** 为每个 AXI 通道添加 FIFO 缓冲
- **复杂度：** 低
- **学习要点：**
  - FIFO 深度对性能的影响
  - 反压（back-pressure）机制

#### `axi_err_slv.sv` — 错误从机

- **功能：** 对所有请求返回错误响应（SLVERR 或 DECERR）
- **复杂度：** 低
- **学习要点：**
  - AXI 从机如何正确应答
  - 错误处理协议要求
  - 用于地址空间中未映射区域

### 3.2 核心互连模块

#### `axi_mux.sv` — 多主合一（Multiplexer）

- **功能：** 将多个 AXI slave port 合并到一个 master port
- **文档：** `doc/axi_mux.md`
- **学习要点：**
  - 仲裁策略（round-robin）
  - ID 前置（prepend）以区分不同来源
  - 读写通道的独立仲裁

#### `axi_demux.sv` — 一主分多从（Demultiplexer）

- **功能：** 将一个 slave port 的事务路由到多个 master port
- **文档：** `doc/axi_demux.md`
- **学习要点：**
  - 地址译码和路由选择
  - 使用 `select` 信号而非内部地址映射（解耦设计）
  - ID 计数器防止死锁
  - 未完成事务追踪

#### `axi_xbar.sv` — 全连接交叉开关（Crossbar）

- **功能：** N 主 x M 从的全连接互连
- **文档：** `doc/axi_xbar.md`
- **复杂度：** 高（Level 5）
- **学习要点：**
  - 由 demux + mux 组合构建
  - 地址映射规则配置
  - ATOP 支持
  - 默认从机路由

### 3.3 数据转换模块

#### `axi_dw_converter.sv` — 数据位宽转换

- **功能：** 在不同数据位宽的 AXI 端口之间转换
- **内部模块：**
  - `axi_dw_upsizer.sv` — 窄转宽
  - `axi_dw_downsizer.sv` — 宽转窄
- **学习要点：**
  - 字节对齐和字节选通（strobe）计算
  - Burst 长度调整

#### `axi_lite_to_axi.sv` / `axi_to_axi_lite.sv` — 协议转换

- **学习要点：**
  - AXI4 和 AXI4-Lite 之间的信号映射
  - Burst 拆分（AXI4 → Lite 需将 burst 拆为单拍）

---

## 第四阶段：验证与仿真

> 目标：理解 AXI 验证方法论，学会使用仿真工具。

### 4.1 验证组件 — `src/axi_test.sv`

这个文件包含丰富的验证类：

| 类 | 功能 |
|----|------|
| `axi_driver` | 低层级 beat-by-beat 驱动器 |
| `axi_rand_master` | 随机化事务生成器 |
| `axi_rand_slave` | 可配置响应从机 |
| `axi_scoreboard` | 内存模型验证（黄金参考） |
| `axi_file_master` | 基于文件的测试场景 |
| `axi_chan_logger` | 事务日志记录 |

**学习要点：**
- 受约束的随机化（constrained random）验证方法
- 如何构建 AXI 主机/从机模型
- 记分板如何检查数据一致性

### 4.2 测试平台示例

推荐阅读顺序：

1. **`test/tb_axi_sim_mem.sv`** — 简单的内存仿真测试，入门级
2. **`test/tb_axi_xbar.sv`** — 交叉开关测试，展示完整验证流程
3. **`test/tb_axi_lite_xbar.sv`** — AXI-Lite 交叉开关测试

### 4.3 运行仿真

```bash
# 使用 Questasim/ModelSim
make compile.log       # 编译
make sim-axi_xbar.log  # 仿真交叉开关测试
make sim_all           # 运行所有测试

# 使用 Verilator（仅 lint）
scripts/run_verilator.sh

# 查看所有可用测试
make help
```

---

## 第五阶段：进阶主题

> 目标：深入理解高级功能和设计技巧。

### 5.1 原子操作（ATOP）

- **文件：** `axi_atop_filter.sv`
- AXI5 引入的原子操作扩展，用于无锁同步
- 支持 AtomicLoad、AtomicStore、AtomicCompare、AtomicSwap
- `axi_atop_filter` 将 ATOP 事务过滤掉，用于不支持原子操作的从机

### 5.2 跨时钟域（CDC）

- **文件：** `axi_cdc_src.sv`, `axi_cdc_dst.sv`
- 基于格雷码 FIFO 的跨时钟域传输
- 源端（src）和目标端（dst）分开设计

### 5.3 ID 管理

| 模块 | 功能 |
|------|------|
| `axi_id_remap.sv` | 将宽 ID 重映射为窄 ID |
| `axi_id_serialize.sv` | 将不同 ID 的事务串行化 |
| `axi_id_prepend.sv` | 在 ID 高位添加/移除位 |
| `axi_iw_converter.sv` | 任意 ID 宽度转换 |

**学习要点：** AXI ID 机制是理解乱序完成（out-of-order completion）的关键。

### 5.4 Burst 处理

| 模块 | 功能 |
|------|------|
| `axi_burst_splitter.sv` | 将 burst 拆为单拍事务 |
| `axi_burst_splitter_gran.sv` | 运行时可配置粒度的拆分 |
| `axi_burst_unwrap.sv` | 将 WRAP burst 转为 INCR burst |

### 5.5 Memory 接口

| 模块 | 功能 |
|------|------|
| `axi_to_mem.sv` | AXI → SRAM 接口（基础版） |
| `axi_to_mem_banked.sv` | AXI → 分体（banked）SRAM |
| `axi_to_mem_interleaved.sv` | AXI → 交织（interleaved）SRAM |
| `axi_from_mem.sv` | SRAM 接口 → AXI 主机 |
| `axi_sim_mem.sv` | 无限大仿真内存（仅仿真） |

---

## 实践建议

### 1. 画框图

每读一个模块，手动画出它的端口和内部数据流：

```
                ┌─────────────┐
  slave port ──►│  axi_demux  ├──► master port 0
   (1 input)   │             ├──► master port 1
               │             ├──► master port 2
               └─────────────┘
```

### 2. 跟踪一次完整事务

选择 `axi_xbar`，从 master 发出 AW 请求到收到 B 响应，跟踪信号在整个交叉开关中的完整路径：

```
Master → demux (地址译码) → 路由选择 → mux (仲裁) → Slave
Slave → mux (响应路由) → demux (ID 匹配) → Master
```

### 3. 对比 AXI4 和 AXI4-Lite

对照阅读同类模块的两个版本：
- `axi_mux.sv` vs `axi_lite_mux.sv`
- `axi_demux.sv` vs `axi_lite_demux.sv`
- `axi_xbar.sv` vs `axi_lite_xbar.sv`

观察 Lite 版本省略了哪些信号和逻辑。

### 4. 修改参数并仿真

尝试修改 `axi_xbar` 的参数（主机数、从机数、地址映射），观察行为变化：

```systemverilog
localparam axi_pkg::xbar_cfg_t XbarCfg = '{
    NoSlvPorts:         4,       // 尝试修改主机数量
    NoMstPorts:         6,       // 尝试修改从机数量
    MaxMstTrans:        10,
    MaxSlvTrans:        6,
    // ...
};
```

### 5. 阅读编码规范

`CONTRIBUTING.md` 中的规则对理解代码风格很重要：
- 所有模块名以 `axi_` 开头
- 用户接口模块必须使用 struct 端口
- 接口变体模块以 `_intf` 后缀命名，仅做连线

---

## 核心设计理念

### 组合优于配置（Composition over Configuration）

本仓库最值得学习的设计思想：

> 用简单模块（mux、demux、cut）自由组合出任意拓扑，而不是用一个巨大的参数化模块去覆盖所有场景。

例如，`axi_xbar` 本身就是由 `axi_demux` + `axi_mux` 组合而成：

```
        ┌──────────┐     ┌──────────┐
M0 ────►│ demux[0] ├──┬──►│  mux[0]  ├────► S0
        └──────────┘  │  └──────────┘
                      │
        ┌──────────┐  │  ┌──────────┐
M1 ────►│ demux[1] ├──┼──►│  mux[1]  ├────► S1
        └──────────┘  │  └──────────┘
                      │
                      └──► ...
```

### Unix 哲学

每个模块只做一件事，做好一件事：
- `axi_cut` — 只插寄存器
- `axi_err_slv` — 只返回错误
- `axi_id_prepend` — 只处理 ID 前置

需要复杂功能时，组合多个模块即可。

---

## 参考资源

### 协议规范
- **AMBA AXI and ACE Protocol Specification, Issue F.b** — ARM 官方规范（需注册下载）
- **AMBA AXI Protocol Specification, Issue A** — 较旧但更易入门的版本

### 仓库文档
- `doc/axi_xbar.md` — 交叉开关详细文档
- `doc/axi_demux.md` — 解复用器文档
- `doc/axi_mux.md` — 复用器文档
- `CHANGELOG.md` — 版本历史，了解设计演进

### 相关项目
- [pulp-platform/common_cells](https://github.com/pulp-platform/common_cells) — 通用基础单元库
- [pulp-platform/pulpissimo](https://github.com/pulp-platform/pulpissimo) — 使用本 AXI 库的 SoC 平台
- [lowRISC SystemVerilog Style Guide](https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md) — 本仓库遵循的编码规范
