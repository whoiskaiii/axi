# 第 5 课：赋值宏与接口定义

> **课程系列**：AXI 总线协议 30 课深度教程
> **所属单元**：单元一 - 协议基础与类型系统
> **前置知识**：第 1-4 课（协议概述、握手机制、AXI Package、Typedef 宏）
> **学习时长**：约 2-3 小时
> **对应源码**：`include/axi/assign.svh`、`include/axi/port.svh`、`src/axi_intf.sv`

---

## 学习目标

完成本课后，你将能够：

1. 解释为什么仓库需要 `assign.svh` 这样的宏层，而不是手写所有 AXI 连线
2. 说明 `AXI_ASSIGN`、`AXI_ASSIGN_TO_REQ`、`AXI_ASSIGN_FROM_REQ`、`AXI_ASSIGN_TO_RESP`、`AXI_ASSIGN_FROM_RESP` 的方向语义
3. 区分 `AXI_SET_*` 与 `AXI_ASSIGN_*` 两类宏的使用场景
4. 读懂 `AXI_BUS` / `AXI_LITE` 的 `modport Master`、`modport Slave`、`modport Monitor`
5. 使用 `axi_cut_intf` 这一类 wrapper 理解 `interface ↔ struct` 的完整转换流程
6. 说明 `port.svh` 中扁平端口宏存在的工程背景，以及它与 `interface` 风格的关系

---

## 目录

- [5.1 为什么需要赋值宏](#51-为什么需要赋值宏)
- [5.2 assign.svh 的分层结构](#52-assignsvh-的分层结构)
- [5.3 接口到接口：AXI_ASSIGN 的真正语义](#53-接口到接口axi_assign-的真正语义)
- [5.4 struct 与 interface 互转宏](#54-struct-与-interface-互转宏)
- [5.5 AXI_BUS interface 与 modport](#55-axi_bus-interface-与-modport)
- [5.6 AXI_LITE 与其他 interface 变体](#56-axi_lite-与其他-interface-变体)
- [5.7 port.svh：为什么还需要扁平端口宏](#57-portsvh为什么还需要扁平端口宏)
- [5.8 在模块中使用赋值宏的完整流程](#58-在模块中使用赋值宏的完整流程)
- [5.9 interface / struct / 波形的对应关系](#59-interface--struct--波形的对应关系)
- [5.10 源码映射](#510-源码映射)
- [5.11 关键术语表](#511-关键术语表)
- [5.12 课后练习](#512-课后练习)
- [5.13 下一课预告](#513-下一课预告)

---

## 5.1 为什么需要赋值宏

第 4 课我们已经知道，本仓库的核心模块更倾向于使用 `struct` 端口，而顶层封装、验证环境、某些工具链又经常使用 `interface` 或扁平端口。这样一来，工程里自然会出现三种连线形态：

1. `interface → interface`
2. `interface → struct` / `struct → interface`
3. `struct → struct`

如果每次都手写这些连线，会有三个明显问题：

- **重复劳动极大**：一个 AXI4 端口有几十个信号，写一次就很长
- **方向容易写错**：尤其是 `valid/ready`、`req_t/resp_t` 的方向并不对称
- **接口演化成本高**：一旦协议字段增加或改名，所有手写连线都要改

`include/axi/assign.svh` 的设计目标，就是把“字段拷贝”和“握手方向”编码成可复用宏。以后写 wrapper 或 adapter 时，思路就变成：

- 先定义好类型
- 再选对赋值宏
- 最后把精力放在真正的协议适配逻辑上

---

## 5.2 assign.svh 的分层结构

`include/axi/assign.svh` 不是几个零散宏，而是一整套分层体系。最底层先定义“字段搬运”的内部宏，再往上组合成 channel 级、request/response 级、Lite 版本、扁平端口版本。

```
assign.svh
├── 内部字段拷贝宏
│   ├── `__AXI_TO_AW
│   ├── `__AXI_TO_W
│   ├── `__AXI_TO_B
│   ├── `__AXI_TO_AR
│   ├── `__AXI_TO_R
│   ├── `__AXI_TO_REQ
│   └── `__AXI_TO_RESP
│
├── interface → interface
│   ├── `AXI_ASSIGN_AW / W / B / AR / R
│   └── `AXI_ASSIGN
│
├── struct ↔ interface
│   ├── `AXI_SET_FROM_* / `AXI_ASSIGN_FROM_*
│   └── `AXI_SET_TO_*   / `AXI_ASSIGN_TO_*
│
├── struct ↔ struct
│   ├── `AXI_SET_*_STRUCT
│   └── `AXI_ASSIGN_*_STRUCT
│
├── AXI4-Lite 对应版本
│   └── `AXI_LITE_*`
│
└── 扁平端口相关宏
    ├── `AXI_ASSIGN_MASTER_TO_FLAT
    ├── `AXI_ASSIGN_SLAVE_TO_FLAT
    └── `AXI_ASSIGN_*_TO_FLAT_PORT
```

### 5.2.1 最底层：`__AXI_TO_*` 内部宏

`assign.svh` 的核心不是 `AXI_ASSIGN`，而是最底层的 `__AXI_TO_*` 系列。它们负责“把一侧字段逐个复制到另一侧”。

```systemverilog
// include/axi/assign.svh, 第 26-82 行
`define __AXI_TO_AW(__opt_as, __lhs, __lhs_sep, __rhs, __rhs_sep)   \
  __opt_as __lhs``__lhs_sep``id     = __rhs``__rhs_sep``id;         \
  __opt_as __lhs``__lhs_sep``addr   = __rhs``__rhs_sep``addr;       \
  // ...

`define __AXI_TO_REQ(__opt_as, __lhs, __lhs_sep, __rhs, __rhs_sep)  \
  `__AXI_TO_AW(__opt_as, __lhs.aw, __lhs_sep, __rhs.aw, __rhs_sep)  \
  __opt_as __lhs.aw_valid = __rhs.aw_valid;                         \
  // ...
```

这里最值得注意的是两个技巧：

- `__opt_as`：让同一套宏既能生成 `assign a = b;`，也能生成过程块内的 `a = b;`
- `__lhs_sep` / `__rhs_sep`：同时兼容 `aw.id` 这种 `struct.field` 路径和 `aw_id` 这种 `interface_signal` 路径

也就是说，这一层不是“协议语义层”，而是“文本拼接层”。

### 5.2.2 为什么能同时兼容 `.` 和 `_`

看两个典型调用：

```systemverilog
`__AXI_TO_AW(assign, dst.aw, _, src.aw, _)
`__AXI_TO_AW(assign, aw_struct, ., axi_if.aw, _)
```

第一种会展开成：

```systemverilog
assign dst.aw_id = src.aw_id;
```

第二种会展开成：

```systemverilog
assign aw_struct.id = axi_if.aw_id;
```

这就是 `assign.svh` 能同时支持 `interface` 和 `struct` 的关键原因。

---

## 5.3 接口到接口：AXI_ASSIGN 的真正语义

很多人第一次看到下面这组宏时，会误以为“所有通道都是 `dst = src`”。

```systemverilog
// include/axi/assign.svh, 第 99-124 行
`define AXI_ASSIGN_AW(dst, src)               \
  `__AXI_TO_AW(assign, dst.aw, _, src.aw, _)  \
  assign dst.aw_valid = src.aw_valid;         \
  assign src.aw_ready = dst.aw_ready;

`define AXI_ASSIGN(slv, mst)  \
  `AXI_ASSIGN_AW(slv, mst)    \
  `AXI_ASSIGN_W(slv, mst)     \
  `AXI_ASSIGN_B(mst, slv)     \
  `AXI_ASSIGN_AR(slv, mst)    \
  `AXI_ASSIGN_R(mst, slv)
```

这里真正的语义不是“左边拷到右边”，而是：

- 对请求通道 AW/W/AR：`master → slave`
- 对响应通道 B/R：`slave → master`

所以 `AXI_ASSIGN(slv, mst)` 展开时：

- AW/W/AR 用 `slv` 作为目标，`mst` 作为来源
- B/R 则反过来，用 `mst` 作为目标，`slv` 作为来源

这是因为 AXI 的五个通道本来就是双向不对称的。

### 5.3.1 单通道宏如何处理握手方向

以 AW 为例：

```
AW payload + aw_valid : Master -> Slave
aw_ready             : Slave  -> Master
```

所以 `AXI_ASSIGN_AW(dst, src)` 实际做了两件事：

1. 把 `src.aw_*` payload 和 `src.aw_valid` 送到 `dst`
2. 把 `dst.aw_ready` 送回 `src`

这正好与 AXI 握手方向一致。

### 5.3.2 为什么 B 和 R 要“反着写”

`AXI_ASSIGN_B(dst, src)` 的语义是“把 B 通道从 `src` 接到 `dst`”。  
但在整条总线语义里，B 是 Slave 发给 Master 的响应。因此完整总线宏才会写成：

```systemverilog
`AXI_ASSIGN_B(mst, slv)
`AXI_ASSIGN_R(mst, slv)
```

如果这里写成 `AXI_ASSIGN_B(slv, mst)`，方向就完全错了。

---

## 5.4 struct 与 interface 互转宏

这一部分是第 5 课最重要的实战内容。因为仓库里最常见的 wrapper 模式，就是在 `interface` 端口和 `struct` 端口之间做双向映射。

### 5.4.1 `AXI_ASSIGN_FROM_*`：struct -> interface

```systemverilog
// include/axi/assign.svh, 第 182-201 行
`define AXI_ASSIGN_FROM_REQ(axi_if, req_struct)   `__AXI_TO_REQ(assign, axi_if, _, req_struct, .)
`define AXI_ASSIGN_FROM_RESP(axi_if, resp_struct) `__AXI_TO_RESP(assign, axi_if, _, resp_struct, .)
```

含义是：

- 把 `req_struct` 中的请求方向字段，赋值到 `axi_if`
- 把 `resp_struct` 中的响应方向字段，赋值到 `axi_if`

注意这里的方向是“谁提供信号值”，不是“谁是 Master/Slave”。

### 5.4.2 `AXI_ASSIGN_TO_*`：interface -> struct

```systemverilog
// include/axi/assign.svh, 第 233-253 行
`define AXI_ASSIGN_TO_REQ(req_struct, axi_if)   `__AXI_TO_REQ(assign, req_struct, ., axi_if, _)
`define AXI_ASSIGN_TO_RESP(resp_struct, axi_if) `__AXI_TO_RESP(assign, resp_struct, ., axi_if, _)
```

含义正好相反：

- 从 `axi_if` 把请求方向字段收集到 `req_struct`
- 从 `axi_if` 把响应方向字段收集到 `resp_struct`

### 5.4.3 `AXI_SET_*` vs `AXI_ASSIGN_*`

两类宏的差别不在功能，而在**使用位置**：

| 类别 | 使用位置 | 展开形式 | 场景 |
|------|------|------|------|
| `AXI_SET_*` | `always_comb` / `always_ff` 内 | 普通赋值 | 过程块内部 |
| `AXI_ASSIGN_*` | 模块级连续赋值 | `assign` | 模块外部连线 |

例如：

```systemverilog
// include/axi/assign.svh, 第 171-177 行
`define AXI_SET_FROM_REQ(axi_if, req_struct)    `__AXI_TO_REQ(, axi_if, _, req_struct, .)

// include/axi/assign.svh, 第 247-253 行
`define AXI_ASSIGN_TO_REQ(req_struct, axi_if)   `__AXI_TO_REQ(assign, req_struct, ., axi_if, _)
```

不要在 `always_comb` 里用 `AXI_ASSIGN_*`，也不要在模块级裸连线里用 `AXI_SET_*`。

### 5.4.4 struct -> struct 宏什么时候有用

`assign.svh` 还定义了：

- `AXI_ASSIGN_REQ_STRUCT(lhs, rhs)`
- `AXI_ASSIGN_RESP_STRUCT(lhs, rhs)`
- `AXI_SET_REQ_STRUCT(lhs, rhs)`
- `AXI_SET_RESP_STRUCT(lhs, rhs)`

这类宏不涉及 `interface`，纯粹做 `struct` 到 `struct` 复制。它们在多级 wrapper、流水线寄存器、事务重命名等场景下很有用。

---

## 5.5 AXI_BUS interface 与 modport

第 4 课主要从 `struct` 视角理解 AXI；这一课要切换到 `interface` 视角。

### 5.5.1 AXI_BUS 中到底定义了什么

`src/axi_intf.sv` 中的 `AXI_BUS` 先参数化位宽，再逐个声明五通道信号。

```systemverilog
// src/axi_intf.sv, 第 20-83 行
interface AXI_BUS #(
  parameter int unsigned AXI_ADDR_WIDTH = 0,
  parameter int unsigned AXI_DATA_WIDTH = 0,
  parameter int unsigned AXI_ID_WIDTH   = 0,
  parameter int unsigned AXI_USER_WIDTH = 0
);
  // typedef id_t / addr_t / data_t / strb_t / user_t
  // AW/W/B/AR/R 全部信号
endinterface
```

也就是说，`interface` 版本本质上还是把所有 AXI 信号平铺出来，只是把它们包进了一个命名空间。

### 5.5.2 `modport Master` 的方向

```systemverilog
// src/axi_intf.sv, 第 85-90 行
modport Master (
  output aw_id, aw_addr, ..., aw_valid, input aw_ready,
  output w_data, w_strb, ..., w_valid, input w_ready,
  input  b_id, b_resp, ..., b_valid, output b_ready,
  output ar_id, ar_addr, ..., ar_valid, input ar_ready,
  input  r_id, r_data, ..., r_valid, output r_ready
);
```

这恰好对应协议定义：

- Master 驱动 AW/W/AR 的 payload 和 `valid`
- Master 接收 AW/W/AR 的 `ready`
- Master 接收 B/R 的 payload 和 `valid`
- Master 驱动 B/R 的 `ready`

### 5.5.3 `modport Slave` 刚好相反

```systemverilog
// src/axi_intf.sv, 第 93-98 行
modport Slave (
  input aw_id, aw_addr, ..., aw_valid, output aw_ready,
  input w_data, w_strb, ..., w_valid, output w_ready,
  output b_id, b_resp, ..., b_valid, input b_ready,
  input ar_id, ar_addr, ..., ar_valid, output ar_ready,
  output r_id, r_data, ..., r_valid, input r_ready
);
```

这说明 `modport` 的核心价值是：

- 把“接口中有哪些信号”与“当前模块以什么方向看这些信号”分开
- 同一个 `AXI_BUS` 可以被不同模块以 `Master` 或 `Slave` 视角使用

### 5.5.4 `Monitor` 为什么有用

`Monitor` modport 把所有信号都定义为 `input`。  
这对验证环境非常自然，因为 monitor 只观察、不驱动。

---

## 5.6 AXI_LITE 与其他 interface 变体

`src/axi_intf.sv` 不只有 `AXI_BUS`，还包括多种变体：

- `AXI_BUS_DV`
- `AXI_BUS_ASYNC`
- `AXI_BUS_ASYNC_GRAY`
- `AXI_LITE`
- `AXI_LITE_DV`
- `AXI_LITE_ASYNC_GRAY`

### 5.6.1 AXI_LITE：字段显著减少

```systemverilog
// src/axi_intf.sv, 第 410-462 行
interface AXI_LITE #(
  parameter int unsigned AXI_ADDR_WIDTH = 0,
  parameter int unsigned AXI_DATA_WIDTH = 0
);
```

与 AXI4 相比，AXI-Lite 没有：

- `id`
- `len`
- `size`
- `burst`
- `last`
- `user`
- `atop`

所以 Lite 版本接口更像“寄存器访问总线”。

### 5.6.2 DV 版本：把时钟带进 interface

`AXI_BUS_DV` 和 `AXI_LITE_DV` 把 `clk_i` 纳入 interface，并在内部写入 assertions。  
这使验证环境能直接围绕接口本身加协议约束，而不必把检查器散落在 testbench 各处。

### 5.6.3 ASYNC / ASYNC_GRAY：不再用 valid/ready

`AXI_BUS_ASYNC`、`AXI_BUS_ASYNC_GRAY` 不是普通同步 AXI interface 的简单复制，而是为 CDC 方案服务的特殊接口。  
这里的关键不再是 `valid/ready`，而是 token、pointer、Gray pointer 等跨时钟域信号。

这说明：`axi_intf.sv` 不只是“给顶层连线用的 interface 文件”，而是整个仓库端口风格的统一入口。

---

## 5.7 port.svh：为什么还需要扁平端口宏

如果 `interface` 已经这么方便，为什么还要保留 `include/axi/port.svh`？

答案很工程化：**并不是所有工具都对 SystemVerilog interface 支持得足够好**，尤其是某些 FPGA 工具流程和 IP integrator。

### 5.7.1 `AXI_M_PORT` / `AXI_S_PORT`

```systemverilog
// include/axi/port.svh, 第 21-67 行
`define AXI_M_PORT(__name, __addr_t, __data_t, __strb_t, __id_t, ...)

// include/axi/port.svh, 第 71-110 行
`define AXI_S_PORT(__name, __addr_t, __data_t, __strb_t, __id_t, ...)
```

它们做的事情很直接：

- 把 AXI 信号全部展开成扁平端口
- 命名遵循 `m_axi_*` / `s_axi_*` 风格
- 与 Vivado IP Integrator 这类工具更兼容

### 5.7.2 `assign.svh` 如何配合 `port.svh`

在 `assign.svh` 里还能看到：

- `AXI_ASSIGN_MASTER_TO_FLAT`
- `AXI_ASSIGN_SLAVE_TO_FLAT`
- `AXI_ASSIGN_MASTER_TO_FLAT_PORT`
- `AXI_ASSIGN_SLAVE_TO_FLAT_PORT`

这说明仓库其实支持三种世界互转：

```
struct  <->  interface  <->  flat ports
```

所以 `port.svh` 不是“落后的遗留物”，而是工具兼容层。

---

## 5.8 在模块中使用赋值宏的完整流程

`src/axi_cut.sv` 中的 `axi_cut_intf` 是第 5 课最好的实战样例。

```systemverilog
// src/axi_cut.sv, 第 124-168 行
module axi_cut_intf #(
  parameter int unsigned ADDR_WIDTH = 0,
  parameter int unsigned DATA_WIDTH = 0,
  parameter int unsigned ID_WIDTH   = 0,
  parameter int unsigned USER_WIDTH = 0
) (
  input logic     clk_i,
  input logic     rst_ni,
  AXI_BUS.Slave   in,
  AXI_BUS.Master  out
);
  typedef logic [ID_WIDTH-1:0]     id_t;
  typedef logic [ADDR_WIDTH-1:0]   addr_t;
  typedef logic [DATA_WIDTH-1:0]   data_t;
  typedef logic [DATA_WIDTH/8-1:0] strb_t;
  typedef logic [USER_WIDTH-1:0]   user_t;

  `AXI_TYPEDEF_...

  axi_req_t  slv_req,  mst_req;
  axi_resp_t slv_resp, mst_resp;

  `AXI_ASSIGN_TO_REQ(slv_req, in)
  `AXI_ASSIGN_FROM_RESP(in, slv_resp)

  `AXI_ASSIGN_FROM_REQ(out, mst_req)
  `AXI_ASSIGN_TO_RESP(mst_resp, out)
```

这个 wrapper 的完整思路如下：

```
AXI_BUS.Slave in
    |
    | `AXI_ASSIGN_TO_REQ
    v
  slv_req ----------------------+
                                |
                            [核心模块 axi_cut]
                                |
  slv_resp <--------------------+
    ^
    | `AXI_ASSIGN_FROM_RESP
    |
AXI_BUS.Slave in

AXI_BUS.Master out
    ^
    | `AXI_ASSIGN_FROM_REQ
  mst_req ----------------------+
                                |
                            [核心模块 axi_cut]
                                |
  mst_resp <--------------------+
    |
    | `AXI_ASSIGN_TO_RESP
    v
AXI_BUS.Master out
```

### 5.8.1 为什么需要四个宏

因为 wrapper 两侧各有一组 request/response：

- 输入口 `in`：
  - `in -> slv_req`
  - `slv_resp -> in`
- 输出口 `out`：
  - `mst_req -> out`
  - `out -> mst_resp`

四个方向都不同，所以不能只靠一个万能宏解决。

### 5.8.2 本质上 wrapper 只做“端口风格转换”

`axi_cut_intf` 的协议行为仍由核心模块 `axi_cut` 决定。  
`_intf` wrapper 做的事情只有三步：

1. 定义类型
2. 在 `interface` 和 `struct` 间搬运信号
3. 实例化核心模块

这也是整个仓库反复出现的设计模式。

---

## 5.9 interface / struct / 波形的对应关系

当你在 wrapper 中把 `interface` 和 `struct` 互转后，波形中常会同时看到两套名字。理解这层对应关系很重要。

### 5.9.1 名称映射关系

| interface 视角 | struct 视角 | 含义 |
|------|------|------|
| `in.aw_addr` | `slv_req.aw.addr` | AW 地址字段 |
| `in.aw_valid` | `slv_req.aw_valid` | AW valid |
| `in.aw_ready` | `slv_resp.aw_ready` | AW ready |
| `out.r_data` | `mst_resp.r.data` | R 数据 |
| `out.r_valid` | `mst_resp.r_valid` | R valid |
| `out.r_ready` | `mst_req.r_ready` | R ready |

### 5.9.2 单拍写事务在 wrapper 中的波形

下面用一个最简单的“单拍写”例子说明 `interface` 信号与 `struct` 信号如何同拍对应。

```wavedrom
{signal: [
  {name: 'clk',                wave: 'p........'},
  {},
  ['in : AXI_BUS.Slave',
    {name: 'in.aw_addr',       wave: 'x.3x.....', data: ['0x1000']},
    {name: 'in.aw_valid',      wave: '0.10.....'},
    {name: 'in.aw_ready',      wave: '0.10.....'},
    {name: 'in.w_data',        wave: 'x..3x....', data: ['0x55AA']},
    {name: 'in.w_valid',       wave: '0..10....'},
    {name: 'in.w_ready',       wave: '0..10....'},
    {name: 'in.b_valid',       wave: '0....1.0.'},
    {name: 'in.b_ready',       wave: '0....1.0.'}
  ],
  {},
  ['slv_req / slv_resp',
    {name: 'slv_req.aw.addr',  wave: 'x.3x.....', data: ['0x1000']},
    {name: 'slv_req.aw_valid', wave: '0.10.....'},
    {name: 'slv_resp.aw_ready',wave: '0.10.....'},
    {name: 'slv_req.w.data',   wave: 'x..3x....', data: ['0x55AA']},
    {name: 'slv_req.w_valid',  wave: '0..10....'},
    {name: 'slv_resp.w_ready', wave: '0..10....'},
    {name: 'slv_resp.b_valid', wave: '0....1.0.'},
    {name: 'slv_req.b_ready',  wave: '0....1.0.'}
  ]
],
  head: {text: 'interface 与 struct 在 wrapper 中的一一对应'},
  foot: {text: '`AXI_ASSIGN_TO_REQ` 与 `AXI_ASSIGN_FROM_RESP` 不改变时序，只改变信号组织方式'}
}
```

要点：

- 宏不会引入寄存器，默认只是组合连线
- `valid/ready` 的拍位不会变化
- 变化的只是“信号名字属于哪个层级”

---

## 5.10 源码映射

| 文件 | 行号 | 内容 | 说明 |
|------|------|------|------|
| `include/axi/assign.svh` | 26-82 | `__AXI_TO_*` | 内部字段搬运宏 |
| `include/axi/assign.svh` | 99-124 | `AXI_ASSIGN_*`, `AXI_ASSIGN` | interface 到 interface 的总线连线宏 |
| `include/axi/assign.svh` | 171-177 | `AXI_SET_FROM_*` | 过程块内 struct -> interface |
| `include/axi/assign.svh` | 195-201 | `AXI_ASSIGN_FROM_*` | 模块级 struct -> interface |
| `include/axi/assign.svh` | 222-228 | `AXI_SET_TO_*` | 过程块内 interface -> struct |
| `include/axi/assign.svh` | 247-253 | `AXI_ASSIGN_TO_*` | 模块级 interface -> struct |
| `include/axi/assign.svh` | 358-383 | `AXI_LITE_ASSIGN` | AXI4-Lite 版本总线赋值宏 |
| `include/axi/assign.svh` | 547-710 | `AXI_ASSIGN_*_TO_FLAT*` | 扁平端口相关宏 |
| `include/axi/port.svh` | 23-67 | `AXI_M_PORT` | 扁平 Master 端口宏 |
| `include/axi/port.svh` | 73-110 | `AXI_S_PORT` | 扁平 Slave 端口宏 |
| `src/axi_intf.sv` | 20-109 | `AXI_BUS` | AXI4 interface 与 modport 定义 |
| `src/axi_intf.sv` | 410-462 | `AXI_LITE` | AXI4-Lite interface 定义 |
| `src/axi_cut.sv` | 124-168 | `axi_cut_intf` | struct/interface 互转的典型 wrapper |

---

## 5.11 关键术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| 赋值宏 | assignment macro | 用预处理器宏批量生成 AXI 信号连线的机制 |
| 连续赋值 | continuous assignment | 使用 `assign` 的模块级组合连线 |
| 过程赋值 | procedural assignment | 在 `always_comb` / `always_ff` 中执行的赋值 |
| 接口 | interface | SystemVerilog 中对一组相关信号的封装机制 |
| 调制端口 | modport | 为同一个 interface 指定不同方向视图的机制 |
| 监视端口 | monitor modport | 只读观察所有接口信号的 modport |
| wrapper | wrapper | 包装核心模块、做端口风格转换或协议适配的外层模块 |
| 扁平端口 | flat ports | 把总线中每个信号单独展开为模块端口的写法 |
| 组合透传 | combinational pass-through | 不加寄存器、只做组合连线的信号传递 |
| 工具兼容层 | tool compatibility layer | 为适配 Vivado 等工具而保留的额外接口风格 |

---

## 5.12 课后练习

### 练习 1：方向判断（基础）

判断以下宏调用中，信号值实际从哪一侧流向哪一侧，并说明原因：

1. `` `AXI_ASSIGN_AW(slv, mst) ``
2. `` `AXI_ASSIGN_B(mst, slv) ``
3. `` `AXI_ASSIGN_TO_REQ(req, axi_if) ``
4. `` `AXI_ASSIGN_FROM_RESP(axi_if, resp) ``

### 练习 2：`SET` vs `ASSIGN`（基础）

说明下面两种写法分别应该放在什么位置，为什么不能混用：

1. `` `AXI_SET_FROM_REQ(my_if, my_req) ``
2. `` `AXI_ASSIGN_FROM_REQ(my_if, my_req) ``

### 练习 3：读 `axi_cut_intf`（中级）

在 `src/axi_cut.sv` 的 `axi_cut_intf` 中回答：

1. 为什么需要先 `typedef` 出 `id_t`、`addr_t`、`data_t`、`strb_t`、`user_t`？
2. 为什么 `in` 使用 `AXI_BUS.Slave`，而 `out` 使用 `AXI_BUS.Master`？
3. 四个赋值宏各自承担什么方向的转换？

### 练习 4：modport 方向表（中级）

根据 `src/axi_intf.sv` 中的 `AXI_BUS` 定义，写出下列信号在 `Master` modport 下是 `input` 还是 `output`：

1. `aw_valid`
2. `aw_ready`
3. `b_valid`
4. `b_ready`
5. `r_data`
6. `r_ready`

### 练习 5：扁平端口场景（高级）

解释为什么仓库同时保留 `interface` 风格和 `port.svh` 中的扁平端口风格。回答时至少覆盖以下三点：

1. 工具链兼容性
2. 信号命名约定
3. `struct <-> interface <-> flat ports` 三层转换关系

### 练习 6：小型 wrapper 设计（实践）

设计一个最小 wrapper，要求：

- 外部端口使用 `AXI_BUS.Slave in` 与 `AXI_BUS.Master out`
- 内部使用 `axi_req_t` / `axi_resp_t`
- 中间插入一层“直接透传”的核心模块接口

要求写出：

1. 基础类型定义
2. `typedef` 宏调用
3. 四个 `AXI_ASSIGN_*` 互转宏
4. 核心模块实例化端口连接

---

## 5.13 下一课预告

**第 6 课：寄存器切片 - `axi_cut`**

在第 6 课中，我们将正式进入第一个具体模块 `axi_cut`，重点分析：

- 为什么 AXI 需要 `spill register`
- 五个通道如何独立插入流水线级
- `Bypass` / `BypassAw` / `BypassW` 等参数如何控制时序与面积
- `axi_cut` 与 `axi_cut_intf` 在仓库中的分工关系

有了第 5 课的赋值宏与 interface 基础，你将能顺畅读懂 `axi_cut_intf` 如何把外部总线接入核心模块。

---

## 参考资料

1. `include/axi/assign.svh` - 本课重点，覆盖 AXI4、AXI4-Lite、flat port 三套赋值宏
2. `src/axi_intf.sv` - `AXI_BUS`、`AXI_LITE` 及其 DV / ASYNC 变体定义
3. `include/axi/port.svh` - 扁平端口宏
4. `src/axi_cut.sv` - `axi_cut_intf` 作为 wrapper 实例
5. `CONTRIBUTING.md` - struct 端口是仓库核心设计约定

---

> **版权声明**：本教材基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库编写，该仓库遵循 Solderpad Hardware License v0.51。
