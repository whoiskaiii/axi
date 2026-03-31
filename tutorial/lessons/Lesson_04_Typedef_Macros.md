# 第 4 课：Typedef 宏 — 参数化类型生成

> **课程系列**：AXI 总线协议 30 课深度教程
> **所属单元**：单元一 — 协议基础与类型系统
> **前置知识**：第 1-3 课（协议概述、握手机制、AXI Package）
> **学习时长**：约 2-3 小时
> **对应源码**：`include/axi/typedef.svh`（全文，约 245 行）

---

## 学习目标

完成本课学习后，你将能够：

1. 解释为什么本仓库使用 struct 端口风格而非散列信号
2. 逐行理解五个单通道 Typedef 宏的展开结果
3. 说明 `req_t` 和 `resp_t` 聚合结构体的字段组成及信号方向含义
4. 使用 `AXI_TYPEDEF_ALL` 宏一键生成完整类型定义
5. 对比 AXI4 和 AXI4-Lite 宏的差异，说明 Lite 精简了哪些字段
6. 在自己的模块中正确使用 typedef 宏定义 AXI 通道类型

---

## 目录

- [4.1 为什么使用 struct 端口？](#41-为什么使用-struct-端口)
- [4.2 typedef.svh 文件结构总览](#42-typedefsvh-文件结构总览)
- [4.3 五个单通道 Typedef 宏](#43-五个单通道-typedef-宏)
- [4.4 请求与响应聚合结构体](#44-请求与响应聚合结构体)
- [4.5 一键生成宏：AXI_TYPEDEF_ALL](#45-一键生成宏axi_typedef_all)
- [4.6 AXI4-Lite Typedef 宏](#46-axi4-lite-typedef-宏)
- [4.7 宏展开完整示例](#47-宏展开完整示例)
- [4.8 在模块中使用 Typedef 宏](#48-在模块中使用-typedef-宏)
- [4.9 req_t / resp_t 与波形的对应关系](#49-req_t--resp_t-与波形的对应关系)
- [4.10 源码映射](#410-源码映射)
- [4.11 关键术语表](#411-关键术语表)
- [4.12 课后练习](#412-课后练习)
- [4.13 下一课预告](#413-下一课预告)

---

## 4.1 为什么使用 struct 端口？

在传统的 AXI 设计中，模块端口需要逐一声明所有通道信号。一个 AXI4 Master 端口有 **50+** 个独立信号，两个端口就是 100+，模块实例化时极易出错。

### 4.1.1 散列信号方式的痛点

```systemverilog
// 传统方式：每个信号都是独立端口
module my_module (
    // AW channel - 14 个信号
    output logic [3:0]  awid,
    output logic [31:0] awaddr,
    output logic [7:0]  awlen,
    output logic [2:0]  awsize,
    output logic [1:0]  awburst,
    output logic        awlock,
    output logic [3:0]  awcache,
    output logic [2:0]  awprot,
    output logic [3:0]  awqos,
    output logic [3:0]  awregion,
    output logic [5:0]  awatop,
    output logic [0:0]  awuser,
    output logic        awvalid,
    input  logic        awready,
    // W channel - 4 个信号
    // B channel - 4 个信号
    // AR channel - 13 个信号
    // R channel - 6 个信号
    // ... 总计 40+ 个信号端口
);
```

问题：
- **连线易错**：信号名拼写错误、宽度不匹配难以发现
- **参数化困难**：修改 ID 宽度需要改动所有 ID 相关端口
- **代码冗长**：实例化时需要逐一连接每个信号

### 4.1.2 struct 方式的优势

```systemverilog
// struct 方式：两个端口搞定
module my_module #(
    parameter type axi_req_t  = logic,
    parameter type axi_resp_t = logic
) (
    input  axi_req_t  slv_req_i,
    output axi_resp_t slv_resp_o
);
```

优势：
- **2 个端口**替代 40+ 个散列信号
- **类型安全**：struct 字段不匹配会产生编译错误
- **参数化灵活**：通过 `parameter type` 传入，同一模块支持任意宽度配置
- **一致性**：所有模块使用相同的 struct 字段名，降低学习成本

### 4.1.3 仓库的设计约定

根据 `CONTRIBUTING.md` 的规定：

> User-facing modules **must** have SystemVerilog `struct`s as AXI ports. The concrete `struct` type **must** be defined as `parameter` to the module.

这是本仓库最核心的设计约定。总结为两条规则：

1. **核心模块使用 struct 端口**（如 `axi_cut`、`axi_xbar`）
2. **`_intf` 后缀的变体**提供 SystemVerilog interface 端口，内部只做连线适配

```
核心模块 (struct 端口)              接口变体 (_intf 后缀)
┌─────────────────────┐           ┌──────────────────────────┐
│ module axi_cut #(   │           │ module axi_cut_intf #(   │
│   parameter type    │           │   parameter ADDR_WIDTH,  │
│     axi_req_t,      │◄─────────│   parameter DATA_WIDTH,  │
│     axi_resp_t      │ 内部实例化 │   ...                    │
│ ) (                 │           │ ) (                      │
│   input  axi_req_t  │           │   AXI_BUS.Slave  in,     │
│   output axi_resp_t │           │   AXI_BUS.Master out     │
│ );                  │           │ );                       │
└─────────────────────┘           └──────────────────────────┘
```

---

## 4.2 typedef.svh 文件结构总览

`include/axi/typedef.svh` 是一个纯宏定义文件，提供了生成 AXI 通道 struct 的所有宏。

```
include/axi/typedef.svh (约 245 行)
│
├── [20-21]  Include guard ........................ `AXI_TYPEDEF_SVH_
│
├── [24-104] AXI4+ATOP 宏 ........................ ★ 本课重点
│   ├── [34-48]   AXI_TYPEDEF_AW_CHAN_T ........... AW 通道 struct
│   ├── [49-55]   AXI_TYPEDEF_W_CHAN_T ............ W 通道 struct
│   ├── [56-61]   AXI_TYPEDEF_B_CHAN_T ............ B 通道 struct
│   ├── [62-75]   AXI_TYPEDEF_AR_CHAN_T ........... AR 通道 struct
│   ├── [76-83]   AXI_TYPEDEF_R_CHAN_T ............ R 通道 struct
│   ├── [84-94]   AXI_TYPEDEF_REQ_T .............. 请求聚合 struct
│   └── [95-104]  AXI_TYPEDEF_RESP_T ............. 响应聚合 struct
│
├── [108-127] AXI4 一键生成宏 .................... ★ 本课重点
│   ├── [119-126] AXI_TYPEDEF_ALL_CT ............. 自定义 req/resp 名称
│   └── [141-142] AXI_TYPEDEF_ALL ................ 默认命名
│
├── [146-201] AXI4-Lite 宏 ....................... ★ 本课重点
│   ├── [157-161] AXI_LITE_TYPEDEF_AW_CHAN_T ..... Lite AW (仅 addr + prot)
│   ├── [162-166] AXI_LITE_TYPEDEF_W_CHAN_T ...... Lite W (无 last)
│   ├── [167-170] AXI_LITE_TYPEDEF_B_CHAN_T ...... Lite B (仅 resp)
│   ├── [171-175] AXI_LITE_TYPEDEF_AR_CHAN_T ..... Lite AR (仅 addr + prot)
│   ├── [176-180] AXI_LITE_TYPEDEF_R_CHAN_T ...... Lite R (无 last, 无 id)
│   ├── [181-191] AXI_LITE_TYPEDEF_REQ_T ........ Lite 请求聚合
│   └── [192-201] AXI_LITE_TYPEDEF_RESP_T ....... Lite 响应聚合
│
└── [205-241] AXI4-Lite 一键生成宏 ............... ★ 本课重点
    ├── [217-224] AXI_LITE_TYPEDEF_ALL_CT ........ 自定义名称
    └── [240-241] AXI_LITE_TYPEDEF_ALL ........... 默认命名
```

---

## 4.3 五个单通道 Typedef 宏

每个 AXI 通道对应一个宏，调用后生成一个 `typedef struct packed`。

### 4.3.1 AW 通道 — 写地址

```systemverilog
// include/axi/typedef.svh, 第 34-48 行
`define AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)  \
  typedef struct packed {                                       \
    id_t              id;                                       \
    addr_t            addr;                                     \
    axi_pkg::len_t    len;                                      \
    axi_pkg::size_t   size;                                     \
    axi_pkg::burst_t  burst;                                    \
    logic             lock;                                     \
    axi_pkg::cache_t  cache;                                    \
    axi_pkg::prot_t   prot;                                     \
    axi_pkg::qos_t    qos;                                      \
    axi_pkg::region_t region;                                   \
    axi_pkg::atop_t   atop;                                     \
    user_t            user;                                     \
  } aw_chan_t;
```

宏参数说明：

| 参数 | 含义 | 示例 |
|------|------|------|
| `aw_chan_t` | 生成的类型名 | `my_axi_aw_chan_t` |
| `addr_t` | 地址类型 | `logic [31:0]` |
| `id_t` | ID 类型 | `logic [3:0]` |
| `user_t` | 用户自定义信号类型 | `logic [0:0]` |

注意观察：**可参数化的字段**（id, addr, user）通过宏参数传入，**固定宽度的字段**（len, size, burst 等）直接使用 `axi_pkg` 中定义的类型。这完美地分离了"协议规定的固定部分"和"设计者可调整的部分"。

### 4.3.2 W 通道 — 写数据

```systemverilog
// include/axi/typedef.svh, 第 49-55 行
`define AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t)  \
  typedef struct packed {                                       \
    data_t data;                                                \
    strb_t strb;                                                \
    logic  last;                                                \
    user_t user;                                                \
  } w_chan_t;
```

W 通道的特点：**没有 id 字段**。在 AXI4 中，W 通道不携带 ID 信息（AXI3 中有 WID，AXI4 已移除）。W 数据的归属通过发送顺序与 AW 通道匹配。

### 4.3.3 B 通道 — 写响应

```systemverilog
// include/axi/typedef.svh, 第 56-61 行
`define AXI_TYPEDEF_B_CHAN_T(b_chan_t, id_t, user_t)  \
  typedef struct packed {                             \
    id_t            id;                               \
    axi_pkg::resp_t resp;                             \
    user_t          user;                             \
  } b_chan_t;
```

B 通道最简洁：id（标识是哪个写事务的响应）+ resp（响应状态）+ user。

### 4.3.4 AR 通道 — 读地址

```systemverilog
// include/axi/typedef.svh, 第 62-75 行
`define AXI_TYPEDEF_AR_CHAN_T(ar_chan_t, addr_t, id_t, user_t)  \
  typedef struct packed {                                       \
    id_t              id;                                       \
    addr_t            addr;                                     \
    axi_pkg::len_t    len;                                      \
    axi_pkg::size_t   size;                                     \
    axi_pkg::burst_t  burst;                                    \
    logic             lock;                                     \
    axi_pkg::cache_t  cache;                                    \
    axi_pkg::prot_t   prot;                                     \
    axi_pkg::qos_t    qos;                                      \
    axi_pkg::region_t region;                                   \
    user_t            user;                                     \
  } ar_chan_t;
```

AR 通道与 AW 通道几乎相同，唯一区别：**没有 `atop` 字段**。原子操作只通过写通道发起（`awatop`），读地址通道不需要此字段。

### 4.3.5 R 通道 — 读数据

```systemverilog
// include/axi/typedef.svh, 第 76-83 行
`define AXI_TYPEDEF_R_CHAN_T(r_chan_t, data_t, id_t, user_t)  \
  typedef struct packed {                                     \
    id_t            id;                                       \
    data_t          data;                                     \
    axi_pkg::resp_t resp;                                     \
    logic           last;                                     \
    user_t          user;                                     \
  } r_chan_t;
```

R 通道携带：id + data + resp + last + user。注意 R 通道有 `id` 字段（用于乱序完成时标识响应属于哪个读事务），但 W 通道没有。

### 4.3.6 五个通道 struct 字段对比

```
┌────────────┬────────┬────────┬────────┬────────┬────────┐
│  字段       │  AW    │   W    │   B    │  AR    │   R    │
├────────────┼────────┼────────┼────────┼────────┼────────┤
│  id        │   ●    │        │   ●    │   ●    │   ●    │
│  addr      │   ●    │        │        │   ●    │        │
│  len       │   ●    │        │        │   ●    │        │
│  size      │   ●    │        │        │   ●    │        │
│  burst     │   ●    │        │        │   ●    │        │
│  lock      │   ●    │        │        │   ●    │        │
│  cache     │   ●    │        │        │   ●    │        │
│  prot      │   ●    │        │        │   ●    │        │
│  qos       │   ●    │        │        │   ●    │        │
│  region    │   ●    │        │        │   ●    │        │
│  atop      │   ●    │        │        │        │        │
│  data      │        │   ●    │        │        │   ●    │
│  strb      │        │   ●    │        │        │        │
│  last      │        │   ●    │        │        │   ●    │
│  resp      │        │        │   ●    │        │   ●    │
│  user      │   ●    │   ●    │   ●    │   ●    │   ●    │
├────────────┼────────┼────────┼────────┼────────┼────────┤
│  可参数化   │id,addr │data,  │id,user │id,addr │data,id │
│  字段       │user    │strb,  │        │user    │user    │
│            │        │user   │        │        │        │
└────────────┴────────┴────────┴────────┴────────┴────────┘

● = 存在该字段
```

---

## 4.4 请求与响应聚合结构体

五个通道 struct 定义了通道内部的数据字段，但模块之间的连接还需要 valid/ready 握手信号。`req_t` 和 `resp_t` 将通道数据与握手信号打包在一起。

### 4.4.1 req_t — 请求结构体

```systemverilog
// include/axi/typedef.svh, 第 84-94 行
`define AXI_TYPEDEF_REQ_T(req_t, aw_chan_t, w_chan_t, ar_chan_t)  \
  typedef struct packed {                                         \
    aw_chan_t aw;                                                 \
    logic     aw_valid;                                           \
    w_chan_t  w;                                                  \
    logic     w_valid;                                            \
    logic     b_ready;                                            \
    ar_chan_t ar;                                                 \
    logic     ar_valid;                                           \
    logic     r_ready;                                            \
  } req_t;
```

### 4.4.2 resp_t — 响应结构体

```systemverilog
// include/axi/typedef.svh, 第 95-104 行
`define AXI_TYPEDEF_RESP_T(resp_t, b_chan_t, r_chan_t)  \
  typedef struct packed {                               \
    logic     aw_ready;                                 \
    logic     ar_ready;                                 \
    logic     w_ready;                                  \
    logic     b_valid;                                  \
    b_chan_t  b;                                        \
    logic     r_valid;                                  \
    r_chan_t  r;                                        \
  } resp_t;
```

### 4.4.3 信号方向划分原理

`req_t` 和 `resp_t` 的划分遵循一个简洁的原则：**按信号方向分组**。

```
                        req_t (Master → Slave)
                    ┌────────────────────────────┐
                    │  aw      (AW 通道数据)       │
                    │  aw_valid                   │───────►
                    │  w       (W 通道数据)        │
                    │  w_valid                    │───────►
                    │  b_ready                    │───────►
    ┌────────┐      │  ar      (AR 通道数据)       │      ┌────────┐
    │        │      │  ar_valid                   │───────►│        │
    │ Master │──────│  r_ready                    │───────►│ Slave  │
    │        │      └────────────────────────────┘      │        │
    │        │      ┌────────────────────────────┐      │        │
    │        │◄─────│  aw_ready                  │◄──────│        │
    │        │      │  ar_ready                  │◄──────│        │
    └────────┘      │  w_ready                   │◄──────└────────┘
                    │  b_valid                   │◄──────
                    │  b       (B 通道数据)        │
                    │  r_valid                   │◄──────
                    │  r       (R 通道数据)        │
                    └────────────────────────────┘
                        resp_t (Slave → Master)
```

理解要点：
- **valid 信号** 跟随数据所在的方向：AW/W/AR 的 valid 在 `req_t` 中，B/R 的 valid 在 `resp_t` 中
- **ready 信号** 总是反方向：AW/W/AR 的 ready 在 `resp_t` 中，B/R 的 ready 在 `req_t` 中
- 这样做的好处：模块只需要 `input req_t` 和 `output resp_t` 两个端口，方向关系通过类型名即可推断

### 4.4.4 req_t / resp_t 字段一览表

| 结构体 | 字段 | 类型 | 说明 |
|--------|------|------|------|
| `req_t` | `aw` | `aw_chan_t` | AW 通道数据（M→S） |
| `req_t` | `aw_valid` | `logic` | AW 握手 valid（M→S） |
| `req_t` | `w` | `w_chan_t` | W 通道数据（M→S） |
| `req_t` | `w_valid` | `logic` | W 握手 valid（M→S） |
| `req_t` | `b_ready` | `logic` | B 握手 ready（M→S） |
| `req_t` | `ar` | `ar_chan_t` | AR 通道数据（M→S） |
| `req_t` | `ar_valid` | `logic` | AR 握手 valid（M→S） |
| `req_t` | `r_ready` | `logic` | R 握手 ready（M→S） |
| `resp_t` | `aw_ready` | `logic` | AW 握手 ready（S→M） |
| `resp_t` | `ar_ready` | `logic` | AR 握手 ready（S→M） |
| `resp_t` | `w_ready` | `logic` | W 握手 ready（S→M） |
| `resp_t` | `b_valid` | `logic` | B 握手 valid（S→M） |
| `resp_t` | `b` | `b_chan_t` | B 通道数据（S→M） |
| `resp_t` | `r_valid` | `logic` | R 握手 valid（S→M） |
| `resp_t` | `r` | `r_chan_t` | R 通道数据（S→M） |

---

## 4.5 一键生成宏：AXI_TYPEDEF_ALL

当不需要精确控制每个通道类型名时，可以使用一键生成宏。

### 4.5.1 AXI_TYPEDEF_ALL_CT（自定义 req/resp 名称）

```systemverilog
// include/axi/typedef.svh, 第 119-126 行
`define AXI_TYPEDEF_ALL_CT(__name, __req, __rsp, __addr_t, __id_t, __data_t, __strb_t, __user_t) \
  `AXI_TYPEDEF_AW_CHAN_T(__name``_aw_chan_t, __addr_t, __id_t, __user_t)                         \
  `AXI_TYPEDEF_W_CHAN_T(__name``_w_chan_t, __data_t, __strb_t, __user_t)                         \
  `AXI_TYPEDEF_B_CHAN_T(__name``_b_chan_t, __id_t, __user_t)                                     \
  `AXI_TYPEDEF_AR_CHAN_T(__name``_ar_chan_t, __addr_t, __id_t, __user_t)                         \
  `AXI_TYPEDEF_R_CHAN_T(__name``_r_chan_t, __data_t, __id_t, __user_t)                           \
  `AXI_TYPEDEF_REQ_T(__req, __name``_aw_chan_t, __name``_w_chan_t, __name``_ar_chan_t)           \
  `AXI_TYPEDEF_RESP_T(__rsp, __name``_b_chan_t, __name``_r_chan_t)
```

`__name` 参数作为所有通道类型名的前缀，通过 SystemVerilog 的 `` ` `` 拼接运算符生成完整名称。

### 4.5.2 AXI_TYPEDEF_ALL（默认命名）

```systemverilog
// include/axi/typedef.svh, 第 141-142 行
`define AXI_TYPEDEF_ALL(__name, __addr_t, __id_t, __data_t, __strb_t, __user_t)                                \
  `AXI_TYPEDEF_ALL_CT(__name, __name``_req_t, __name``_resp_t, __addr_t, __id_t, __data_t, __strb_t, __user_t)
```

这是最常用的宏，只需指定前缀名和五个基础类型，即可生成全部 7 个 typedef。

### 4.5.3 宏展开示意图

```
`AXI_TYPEDEF_ALL(axi, addr_t, id_t, data_t, strb_t, user_t)

                    │ 展开为
                    ▼
┌──────────────────────────────────────────────────┐
│ `AXI_TYPEDEF_AW_CHAN_T(axi_aw_chan_t, ...)        │──► typedef struct packed {...} axi_aw_chan_t;
│ `AXI_TYPEDEF_W_CHAN_T(axi_w_chan_t, ...)          │──► typedef struct packed {...} axi_w_chan_t;
│ `AXI_TYPEDEF_B_CHAN_T(axi_b_chan_t, ...)          │──► typedef struct packed {...} axi_b_chan_t;
│ `AXI_TYPEDEF_AR_CHAN_T(axi_ar_chan_t, ...)        │──► typedef struct packed {...} axi_ar_chan_t;
│ `AXI_TYPEDEF_R_CHAN_T(axi_r_chan_t, ...)          │──► typedef struct packed {...} axi_r_chan_t;
│ `AXI_TYPEDEF_REQ_T(axi_req_t, ...)               │──► typedef struct packed {...} axi_req_t;
│ `AXI_TYPEDEF_RESP_T(axi_resp_t, ...)             │──► typedef struct packed {...} axi_resp_t;
└──────────────────────────────────────────────────┘

共生成 7 个类型：5 个通道 + 2 个聚合
```

---

## 4.6 AXI4-Lite Typedef 宏

AXI4-Lite 是 AXI4 的简化子集，用于低带宽的控制/状态寄存器访问。其 typedef 宏精简了大量字段。

### 4.6.1 Lite AW 通道

```systemverilog
// include/axi/typedef.svh, 第 157-161 行
`define AXI_LITE_TYPEDEF_AW_CHAN_T(aw_chan_lite_t, addr_t)  \
  typedef struct packed {                                   \
    addr_t          addr;                                   \
    axi_pkg::prot_t prot;                                   \
  } aw_chan_lite_t;
```

对比 AXI4 AW 通道的 12 个字段，Lite 版本只有 **2 个字段**：`addr` 和 `prot`。这意味着 AXI4-Lite 不支持 burst、lock、cache、QoS、region、ATOP、ID 和 user。

### 4.6.2 Lite W 通道

```systemverilog
// include/axi/typedef.svh, 第 162-166 行
`define AXI_LITE_TYPEDEF_W_CHAN_T(w_chan_lite_t, data_t, strb_t)  \
  typedef struct packed {                                         \
    data_t   data;                                                \
    strb_t   strb;                                                \
  } w_chan_lite_t;
```

没有 `last`（因为 Lite 只支持单拍传输，不需要标记最后一拍）和 `user`。

### 4.6.3 Lite B 通道

```systemverilog
// include/axi/typedef.svh, 第 167-170 行
`define AXI_LITE_TYPEDEF_B_CHAN_T(b_chan_lite_t)  \
  typedef struct packed {                         \
    axi_pkg::resp_t resp;                         \
  } b_chan_lite_t;
```

极简：只有 `resp`，没有 `id`（Lite 无 ID 机制）和 `user`。

### 4.6.4 Lite AR 通道

```systemverilog
// include/axi/typedef.svh, 第 171-175 行
`define AXI_LITE_TYPEDEF_AR_CHAN_T(ar_chan_lite_t, addr_t)  \
  typedef struct packed {                                   \
    addr_t          addr;                                   \
    axi_pkg::prot_t prot;                                   \
  } ar_chan_lite_t;
```

与 Lite AW 完全对称，只有 `addr` + `prot`。

### 4.6.5 Lite R 通道

```systemverilog
// include/axi/typedef.svh, 第 176-180 行
`define AXI_LITE_TYPEDEF_R_CHAN_T(r_chan_lite_t, data_t)  \
  typedef struct packed {                                 \
    data_t          data;                                 \
    axi_pkg::resp_t resp;                                 \
  } r_chan_lite_t;
```

只有 `data` + `resp`，没有 `id`、`last`、`user`。

### 4.6.6 Lite 一键生成宏

```systemverilog
// include/axi/typedef.svh, 第 240-241 行
`define AXI_LITE_TYPEDEF_ALL(__name, __addr_t, __data_t, __strb_t)
```

注意 Lite 版本只需要 **3 个基础类型参数**（addr, data, strb），不需要 id 和 user。

### 4.6.7 AXI4 vs AXI4-Lite 通道字段对比

```
┌────────────┬─────────────────────────┬─────────────────────────┐
│  通道       │      AXI4 字段          │    AXI4-Lite 字段        │
├────────────┼─────────────────────────┼─────────────────────────┤
│  AW        │ id, addr, len, size,    │ addr, prot              │
│            │ burst, lock, cache,     │                         │
│            │ prot, qos, region,      │ (12 → 2 个字段)          │
│            │ atop, user              │                         │
├────────────┼─────────────────────────┼─────────────────────────┤
│  W         │ data, strb, last, user  │ data, strb              │
│            │                         │ (4 → 2 个字段)           │
├────────────┼─────────────────────────┼─────────────────────────┤
│  B         │ id, resp, user          │ resp                    │
│            │                         │ (3 → 1 个字段)           │
├────────────┼─────────────────────────┼─────────────────────────┤
│  AR        │ id, addr, len, size,    │ addr, prot              │
│            │ burst, lock, cache,     │                         │
│            │ prot, qos, region, user │ (11 → 2 个字段)          │
├────────────┼─────────────────────────┼─────────────────────────┤
│  R         │ id, data, resp,         │ data, resp              │
│            │ last, user              │ (5 → 2 个字段)           │
└────────────┴─────────────────────────┴─────────────────────────┘
```

### 4.6.8 Lite RESP_T 细节差异

值得注意的是，AXI4-Lite 的 `RESP_T` 中 ready 信号的顺序与 AXI4 版本略有不同：

```systemverilog
// AXI4 RESP_T (第 95-104 行)                 // AXI4-Lite RESP_T (第 192-201 行)
typedef struct packed {                       typedef struct packed {
    logic     aw_ready;                           logic          aw_ready;
    logic     ar_ready;   // ← 先 ar            logic          w_ready;    // ← 先 w
    logic     w_ready;    // ← 后 w              b_chan_lite_t  b;
    logic     b_valid;                            logic          b_valid;
    b_chan_t  b;                                   logic          ar_ready;   // ← 后 ar
    logic     r_valid;                            r_chan_lite_t  r;
    r_chan_t  r;                                   logic          r_valid;
} resp_t;                                     } resp_lite_t;
```

这个差异不影响功能正确性（因为 struct 成员通过名称访问），但在序列化为 flat bit vector 时需要注意。

---

## 4.7 宏展开完整示例

以一个典型配置为例，完整展示 `AXI_TYPEDEF_ALL` 的展开过程。

### 4.7.1 前提：定义基础类型

```systemverilog
// 假设参数
localparam int unsigned AddrWidth = 32;
localparam int unsigned DataWidth = 64;
localparam int unsigned IdWidth   = 4;
localparam int unsigned UserWidth = 1;

// 基础类型
typedef logic [AddrWidth-1:0]   addr_t;   // logic [31:0]
typedef logic [DataWidth-1:0]   data_t;   // logic [63:0]
typedef logic [DataWidth/8-1:0] strb_t;   // logic [7:0]
typedef logic [IdWidth-1:0]     id_t;     // logic [3:0]
typedef logic [UserWidth-1:0]   user_t;   // logic [0:0]
```

### 4.7.2 一行宏调用

```systemverilog
`AXI_TYPEDEF_ALL(axi, addr_t, id_t, data_t, strb_t, user_t)
```

### 4.7.3 展开结果 — 完整 7 个 typedef

```systemverilog
// ═══════════════════════════════════════════════════════
//  以下代码由上面一行宏自动生成
// ═══════════════════════════════════════════════════════

// (1) AW 通道 struct - 共 71 位
typedef struct packed {
    logic [3:0]         id;      // 4 位
    logic [31:0]        addr;    // 32 位
    axi_pkg::len_t      len;     // 8 位
    axi_pkg::size_t     size;    // 3 位
    axi_pkg::burst_t    burst;   // 2 位
    logic               lock;    // 1 位
    axi_pkg::cache_t    cache;   // 4 位
    axi_pkg::prot_t     prot;    // 3 位
    axi_pkg::qos_t      qos;     // 4 位
    axi_pkg::region_t   region;  // 4 位
    axi_pkg::atop_t     atop;    // 6 位
    logic [0:0]         user;    // 1 位
} axi_aw_chan_t;                 // 合计: 4+32+8+3+2+1+4+3+4+4+6+1 = 72 位

// (2) W 通道 struct - 共 74 位
typedef struct packed {
    logic [63:0]  data;   // 64 位
    logic [7:0]   strb;   // 8 位
    logic         last;   // 1 位
    logic [0:0]   user;   // 1 位
} axi_w_chan_t;           // 合计: 64+8+1+1 = 74 位

// (3) B 通道 struct - 共 7 位
typedef struct packed {
    logic [3:0]         id;    // 4 位
    axi_pkg::resp_t     resp;  // 2 位
    logic [0:0]         user;  // 1 位
} axi_b_chan_t;                // 合计: 4+2+1 = 7 位

// (4) AR 通道 struct - 共 66 位
typedef struct packed {
    logic [3:0]         id;      // 4 位
    logic [31:0]        addr;    // 32 位
    axi_pkg::len_t      len;     // 8 位
    axi_pkg::size_t     size;    // 3 位
    axi_pkg::burst_t    burst;   // 2 位
    logic               lock;    // 1 位
    axi_pkg::cache_t    cache;   // 4 位
    axi_pkg::prot_t     prot;    // 3 位
    axi_pkg::qos_t      qos;     // 4 位
    axi_pkg::region_t   region;  // 4 位
    logic [0:0]         user;    // 1 位
} axi_ar_chan_t;                 // 合计: 4+32+8+3+2+1+4+3+4+4+1 = 66 位
                                 // 注意：比 AW 少 6 位（无 atop）

// (5) R 通道 struct - 共 72 位
typedef struct packed {
    logic [3:0]         id;    // 4 位
    logic [63:0]        data;  // 64 位
    axi_pkg::resp_t     resp;  // 2 位
    logic               last;  // 1 位
    logic [0:0]         user;  // 1 位
} axi_r_chan_t;                // 合计: 4+64+2+1+1 = 72 位

// (6) 请求聚合 struct
typedef struct packed {
    axi_aw_chan_t aw;          // 72 位
    logic         aw_valid;   // 1 位
    axi_w_chan_t  w;           // 74 位
    logic         w_valid;    // 1 位
    logic         b_ready;    // 1 位
    axi_ar_chan_t ar;          // 66 位
    logic         ar_valid;   // 1 位
    logic         r_ready;    // 1 位
} axi_req_t;                  // 合计: 72+1+74+1+1+66+1+1 = 217 位

// (7) 响应聚合 struct
typedef struct packed {
    logic         aw_ready;   // 1 位
    logic         ar_ready;   // 1 位
    logic         w_ready;    // 1 位
    logic         b_valid;    // 1 位
    axi_b_chan_t  b;           // 7 位
    logic         r_valid;    // 1 位
    axi_r_chan_t  r;           // 72 位
} axi_resp_t;                 // 合计: 1+1+1+1+7+1+72 = 84 位
```

### 4.7.4 位宽总结

| 类型 | 位宽 | 说明 |
|------|------|------|
| `axi_aw_chan_t` | 72 | id(4)+addr(32)+len(8)+size(3)+burst(2)+lock(1)+cache(4)+prot(3)+qos(4)+region(4)+atop(6)+user(1) |
| `axi_w_chan_t` | 74 | data(64)+strb(8)+last(1)+user(1) |
| `axi_b_chan_t` | 7 | id(4)+resp(2)+user(1) |
| `axi_ar_chan_t` | 66 | 同 AW 但无 atop（72-6=66） |
| `axi_r_chan_t` | 72 | id(4)+data(64)+resp(2)+last(1)+user(1) |
| `axi_req_t` | 217 | 5 个通道数据 + 5 个握手信号 |
| `axi_resp_t` | 84 | 2 个通道数据 + 5 个握手信号 |

---

## 4.8 在模块中使用 Typedef 宏

### 4.8.1 核心模块的典型用法

以 `axi_cut` 为例，核心模块通过 `parameter type` 接收 struct 类型：

```systemverilog
// src/axi_cut.sv, 第 20-46 行
module axi_cut #(
  parameter bit  Bypass     = 1'b0,
  // AXI channel structs — 由调用方定义
  parameter type  aw_chan_t = logic,
  parameter type   w_chan_t = logic,
  parameter type   b_chan_t = logic,
  parameter type  ar_chan_t = logic,
  parameter type   r_chan_t = logic,
  // AXI request & response structs
  parameter type  axi_req_t = logic,
  parameter type axi_resp_t = logic
) (
  input logic       clk_i,
  input logic       rst_ni,
  input  axi_req_t  slv_req_i,     // ← 一个端口代替 20+ 个信号
  output axi_resp_t slv_resp_o,
  output axi_req_t  mst_req_o,
  input  axi_resp_t mst_resp_i
);
```

默认值 `= logic` 是占位符，实例化时必须提供实际类型。

### 4.8.2 _intf 变体中使用宏的完整流程

`axi_cut_intf` 模块展示了 typedef 宏的典型使用模式：

```systemverilog
// src/axi_cut.sv, 第 116-191 行 (axi_cut_intf 模块)
module axi_cut_intf #(
    parameter int unsigned ADDR_WIDTH = 0,
    parameter int unsigned DATA_WIDTH = 0,
    parameter int unsigned ID_WIDTH   = 0,
    parameter int unsigned USER_WIDTH = 0
) (
    input logic     clk_i,
    input logic     rst_ni,
    AXI_BUS.Slave   in,        // SystemVerilog interface 端口
    AXI_BUS.Master  out
);
    // 第 1 步：定义基础类型
    typedef logic [ID_WIDTH-1:0]     id_t;
    typedef logic [ADDR_WIDTH-1:0]   addr_t;
    typedef logic [DATA_WIDTH-1:0]   data_t;
    typedef logic [DATA_WIDTH/8-1:0] strb_t;
    typedef logic [USER_WIDTH-1:0]   user_t;

    // 第 2 步：使用单通道宏生成 struct 类型
    `AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)
    `AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t)
    `AXI_TYPEDEF_B_CHAN_T(b_chan_t, id_t, user_t)
    `AXI_TYPEDEF_AR_CHAN_T(ar_chan_t, addr_t, id_t, user_t)
    `AXI_TYPEDEF_R_CHAN_T(r_chan_t, data_t, id_t, user_t)
    `AXI_TYPEDEF_REQ_T(axi_req_t, aw_chan_t, w_chan_t, ar_chan_t)
    `AXI_TYPEDEF_RESP_T(axi_resp_t, b_chan_t, r_chan_t)

    // 第 3 步：声明 struct 变量
    axi_req_t  slv_req,  mst_req;
    axi_resp_t slv_resp, mst_resp;

    // 第 4 步：使用赋值宏在 interface 和 struct 之间转换
    `AXI_ASSIGN_TO_REQ(slv_req, in)        // interface → struct
    `AXI_ASSIGN_FROM_RESP(in, slv_resp)    // struct → interface

    `AXI_ASSIGN_FROM_REQ(out, mst_req)     // struct → interface
    `AXI_ASSIGN_TO_RESP(mst_resp, out)     // interface → struct

    // 第 5 步：实例化核心模块
    axi_cut #(
        .aw_chan_t  ( aw_chan_t  ),
        .w_chan_t   ( w_chan_t   ),
        .b_chan_t   ( b_chan_t   ),
        .ar_chan_t  ( ar_chan_t  ),
        .r_chan_t   ( r_chan_t   ),
        .axi_req_t ( axi_req_t ),
        .axi_resp_t( axi_resp_t)
    ) i_axi_cut (
        .clk_i,
        .rst_ni,
        .slv_req_i  ( slv_req  ),
        .slv_resp_o ( slv_resp ),
        .mst_req_o  ( mst_req  ),
        .mst_resp_i ( mst_resp )
    );
endmodule
```

### 4.8.3 使用流程图

```
┌───────────────────────────────────────────────────────┐
│ 第 1 步：定义基础类型（根据设计参数）                      │
│   typedef logic [31:0] addr_t;                        │
│   typedef logic [63:0] data_t;  ...                   │
└───────────────────┬───────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────┐
│ 第 2 步：调用 typedef 宏（生成通道和聚合 struct）          │
│   `AXI_TYPEDEF_ALL(axi, addr_t, id_t, ...)            │
│   或逐个调用单通道宏                                     │
└───────────────────┬───────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────┐
│ 第 3 步：声明 struct 变量                                │
│   axi_req_t  req;                                     │
│   axi_resp_t resp;                                    │
└───────────────────┬───────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────┐
│ 第 4 步：将 struct 传入模块（parameter type）              │
│   my_module #(.axi_req_t(axi_req_t)) i_mod (          │
│       .req_i(req), .resp_o(resp)                      │
│   );                                                  │
└───────────────────────────────────────────────────────┘
```

### 4.8.4 访问 struct 字段

```systemverilog
axi_req_t req;

// 访问写地址通道的各字段
req.aw.addr   = 32'h1000_0000;
req.aw.len    = 8'd3;            // 4 拍 burst
req.aw.size   = 3'd3;            // 每拍 8 字节
req.aw.burst  = axi_pkg::BURST_INCR;
req.aw_valid  = 1'b1;            // 注意：valid 在 req 层级，不在 aw 内

// 检查读响应
if (resp.r_valid && req.r_ready) begin
    // 握手完成
    data = resp.r.data;
    if (resp.r.last) begin
        // 最后一拍
    end
end
```

---

## 4.9 req_t / resp_t 与波形的对应关系

当我们在波形查看器中观察 AXI 信号时，struct 端口的信号会展开为层次化的名称。以下示例展示一次简单的写事务在 struct 端口上的波形表现。

### 4.9.1 单拍写事务时序

Master 向地址 `0x1000` 写入数据 `0xDEAD_BEEF`，Slave 返回 `OKAY` 响应：

```wavedrom
{signal: [
  {name: 'clk',              wave: 'p........'},
  {},
  ['req_t (M→S)',
    {name: 'req.aw.addr',    wave: 'x.3x.....', data: ['0x1000']},
    {name: 'req.aw.len',     wave: 'x.3x.....', data: ['0x00']},
    {name: 'req.aw.size',    wave: 'x.3x.....', data: ['0x2']},
    {name: 'req.aw_valid',   wave: '0.10.....'},
    {name: 'req.w.data',     wave: 'x..3x....', data: ['0xDEAD_BEEF']},
    {name: 'req.w.strb',     wave: 'x..3x....', data: ['0xF']},
    {name: 'req.w.last',     wave: '0..10....'},
    {name: 'req.w_valid',    wave: '0..10....'},
    {name: 'req.b_ready',    wave: '0....1.0.'}
  ],
  {},
  ['resp_t (S→M)',
    {name: 'resp.aw_ready',  wave: '0.10.....'},
    {name: 'resp.w_ready',   wave: '0..10....'},
    {name: 'resp.b.resp',    wave: 'x....3x..', data: ['OKAY']},
    {name: 'resp.b_valid',   wave: '0....1.0.'}
  ]
],
  head: {text: '单拍写事务 — struct 端口视角'},
  foot: {text: 'AW 握手 → W 握手 → B 握手'}
}
```

要点：
- `req.aw` 和 `req.aw_valid` 是不同层级的信号：`aw` 在通道 struct 内，`aw_valid` 在 req 层级
- `resp.aw_ready` 在 `resp_t` 中，因为它是 Slave 到 Master 方向

### 4.9.2 单拍读事务时序

Master 从地址 `0x2000` 读取一个 32 位数据：

```wavedrom
{signal: [
  {name: 'clk',              wave: 'p.......'},
  {},
  ['req_t (M→S)',
    {name: 'req.ar.addr',    wave: 'x.3x....', data: ['0x2000']},
    {name: 'req.ar.len',     wave: 'x.3x....', data: ['0x00']},
    {name: 'req.ar_valid',   wave: '0.10....'},
    {name: 'req.r_ready',    wave: '0...1.0.'}
  ],
  {},
  ['resp_t (S→M)',
    {name: 'resp.ar_ready',  wave: '0.10....'},
    {name: 'resp.r.data',    wave: 'x...3x..', data: ['0xCAFE']},
    {name: 'resp.r.resp',    wave: 'x...3x..', data: ['OKAY']},
    {name: 'resp.r.last',    wave: '0...10..'},
    {name: 'resp.r_valid',   wave: '0...1.0.'}
  ]
],
  head: {text: '单拍读事务 — struct 端口视角'},
  foot: {text: 'AR 握手 → R 握手'}
}
```

### 4.9.3 4 拍 Burst 读事务

Master 从地址 `0x1000` 发起 4 拍 INCR burst（`len=3, size=2`），展示 `last` 信号的变化：

```wavedrom
{signal: [
  {name: 'clk',              wave: 'p..........'},
  {},
  ['req_t',
    {name: 'req.ar.addr',    wave: 'x.3x.......', data: ['0x1000']},
    {name: 'req.ar.len',     wave: 'x.3x.......', data: ['0x03']},
    {name: 'req.ar.burst',   wave: 'x.3x.......', data: ['INCR']},
    {name: 'req.ar_valid',   wave: '0.10.......'},
    {name: 'req.r_ready',    wave: '0...1....0.'}
  ],
  {},
  ['resp_t',
    {name: 'resp.ar_ready',  wave: '0.10.......'},
    {name: 'resp.r.data',    wave: 'x...3333x..', data: ['D0', 'D1', 'D2', 'D3']},
    {name: 'resp.r.resp',    wave: 'x...3333x..', data: ['OK', 'OK', 'OK', 'OK']},
    {name: 'resp.r.last',    wave: '0...0001.0.'},
    {name: 'resp.r_valid',   wave: '0...1111.0.'}
  ]
],
  head: {text: '4 拍 INCR Burst 读事务'},
  foot: {text: 'AR 握手一次 → R 握手四次，last 仅在最后一拍为高'}
}
```

要点：
- AR 通道只需握手一次（burst 信息通过 `len` 和 `burst` 传递）
- R 通道握手四次（每拍一次），只有最后一拍的 `resp.r.last` 为 1

---

## 4.10 源码映射

| 文件 | 行号 | 内容 | 说明 |
|------|------|------|------|
| `include/axi/typedef.svh` | 34-48 | `AXI_TYPEDEF_AW_CHAN_T` | AW 通道 struct 宏 |
| `include/axi/typedef.svh` | 49-55 | `AXI_TYPEDEF_W_CHAN_T` | W 通道 struct 宏 |
| `include/axi/typedef.svh` | 56-61 | `AXI_TYPEDEF_B_CHAN_T` | B 通道 struct 宏 |
| `include/axi/typedef.svh` | 62-75 | `AXI_TYPEDEF_AR_CHAN_T` | AR 通道 struct 宏 |
| `include/axi/typedef.svh` | 76-83 | `AXI_TYPEDEF_R_CHAN_T` | R 通道 struct 宏 |
| `include/axi/typedef.svh` | 84-94 | `AXI_TYPEDEF_REQ_T` | 请求聚合 struct 宏 |
| `include/axi/typedef.svh` | 95-104 | `AXI_TYPEDEF_RESP_T` | 响应聚合 struct 宏 |
| `include/axi/typedef.svh` | 119-126 | `AXI_TYPEDEF_ALL_CT` | 一键生成宏（自定义名称） |
| `include/axi/typedef.svh` | 141-142 | `AXI_TYPEDEF_ALL` | 一键生成宏（默认命名） |
| `include/axi/typedef.svh` | 157-201 | AXI4-Lite 单通道宏 | Lite 版本的 7 个宏 |
| `include/axi/typedef.svh` | 217-224 | `AXI_LITE_TYPEDEF_ALL_CT` | Lite 一键生成（自定义名称） |
| `include/axi/typedef.svh` | 240-241 | `AXI_LITE_TYPEDEF_ALL` | Lite 一键生成（默认命名） |
| `CONTRIBUTING.md` | 11-14 | struct 端口设计约定 | 仓库核心编码规范 |
| `src/axi_cut.sv` | 20-46 | `axi_cut` 模块参数 | 核心模块 struct 端口示例 |
| `src/axi_cut.sv` | 116-191 | `axi_cut_intf` 模块 | _intf 变体使用宏的完整流程 |

---

## 4.11 关键术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| 结构体 | struct | SystemVerilog 中将多个信号打包为一个复合类型的机制 |
| packed struct | packed struct | 位域连续排列的结构体，可直接作为位向量操作 |
| 参数化类型 | parameterized type | 通过 `parameter type` 传入的类型参数，允许同一模块支持不同宽度 |
| typedef 宏 | typedef macro | 使用预处理器宏自动生成 `typedef struct packed` 定义 |
| 请求结构体 | request struct (req_t) | 聚合所有 Master→Slave 方向信号的结构体 |
| 响应结构体 | response struct (resp_t) | 聚合所有 Slave→Master 方向信号的结构体 |
| 通道结构体 | channel struct | 聚合单个通道内所有数据字段的结构体（不含 valid/ready） |
| 一键生成宏 | all-in-one macro | 一次调用生成全部 7 个 typedef 的快捷宏 |
| 接口变体 | interface variant (_intf) | 使用 SystemVerilog interface 端口的模块包装器 |
| 散列信号 | flat/loose signals | 将 AXI 信号逐个声明为独立端口的传统方式 |
| 拼接运算符 | concatenation operator (`` ` ``) | SystemVerilog 预处理器中用于拼接标识符名称的运算符 |
| Include guard | include guard | 防止头文件被重复包含的预处理指令（`ifndef`/`define`/`endif`） |

---

## 4.12 课后练习

### 练习 1：宏参数辨析（基础）

回答以下问题：

1. `AXI_TYPEDEF_AW_CHAN_T` 宏有几个参数？分别是什么含义？
2. `AXI_TYPEDEF_W_CHAN_T` 的参数为什么没有 `id_t`？
3. `AXI_TYPEDEF_ALL` 和 `AXI_TYPEDEF_ALL_CT` 的区别是什么？

### 练习 2：手动展开宏（基础）

给定以下基础类型定义：

```systemverilog
typedef logic [15:0]  my_addr_t;
typedef logic [31:0]  my_data_t;
typedef logic [3:0]   my_strb_t;
typedef logic [1:0]   my_id_t;
typedef logic         my_user_t;
```

手动写出 `AXI_TYPEDEF_B_CHAN_T(my_b_t, my_id_t, my_user_t)` 展开后的完整 struct 定义，并计算总位宽。

### 练习 3：req_t 字段归属（中级）

判断以下信号属于 `req_t` 还是 `resp_t`，并说明理由：

1. `aw_valid`
2. `aw_ready`
3. `r_valid`
4. `r_ready`
5. `b_valid`
6. `w.data`

### 练习 4：AXI4 vs Lite 对比（中级）

1. AXI4-Lite 的 AW 通道为什么不需要 `len`、`size`、`burst` 字段？
2. AXI4-Lite 的 W 通道为什么不需要 `last` 字段？
3. AXI4-Lite 的 `AXI_LITE_TYPEDEF_ALL` 为什么只需要 3 个基础类型参数而不是 5 个？

### 练习 5：位宽计算（高级）

在以下配置下计算 `axi_req_t` 和 `axi_resp_t` 的总位宽：

```
AddrWidth = 48, DataWidth = 256, IdWidth = 8, UserWidth = 4
```

提示：先算出 5 个通道 struct 的位宽，再加上 req/resp 中的 valid/ready 信号位数。

### 练习 6：实践练习（实践）

在仓库中找到一个使用 `AXI_TYPEDEF_ALL` 宏的测试文件（在 `test/` 目录下），分析：

1. 它定义了哪些基础类型参数？
2. 生成的类型名前缀是什么？
3. 这些类型被传给了哪个核心模块？

提示：使用 `grep -rn 'AXI_TYPEDEF_ALL' test/` 搜索。

---

## 4.13 下一课预告

**第 5 课：赋值宏与接口定义**

在第 5 课中，我们将精读 `include/axi/assign.svh` 和 `src/axi_intf.sv`，涵盖：

- 赋值宏的完整体系：`AXI_ASSIGN`、`AXI_ASSIGN_TO_REQ`、`AXI_ASSIGN_FROM_REQ` 等
- SystemVerilog interface 定义：`AXI_BUS`、`AXI_LITE` 的 modport
- 两种端口风格（struct vs interface）在实际模块中的完整互转流程
- 端口定义宏（`port.svh`）在 Vivado 等工具中的应用

有了第 4 课的 typedef 基础和第 5 课的赋值宏，你将具备阅读仓库中所有模块端口的能力。

---

## 参考资料

1. **ARM AMBA AXI and ACE Protocol Specification, Issue F.b** — A1 节（AXI4 信号定义）
2. `include/axi/typedef.svh` — 本课覆盖全文（约 245 行）
3. `CONTRIBUTING.md` — struct 端口设计约定
4. `src/axi_cut.sv` — struct 端口使用示例（核心模块 + _intf 变体）

---

> **版权声明**：本教材基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库编写，该仓库遵循 Solderpad Hardware License v0.51。
