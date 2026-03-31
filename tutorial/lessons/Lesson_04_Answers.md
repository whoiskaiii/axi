# 第 4 课练习答案：Typedef 宏 — 参数化类型生成

---

## 练习 1：宏参数辨析

**1. `AXI_TYPEDEF_AW_CHAN_T` 宏有几个参数？分别是什么含义？**

4 个参数：
- `aw_chan_t`：生成的 struct 类型名
- `addr_t`：地址字段的类型（决定地址宽度）
- `id_t`：ID 字段的类型（决定 ID 宽度）
- `user_t`：用户自定义字段的类型（决定 user 宽度）

**2. `AXI_TYPEDEF_W_CHAN_T` 的参数为什么没有 `id_t`？**

因为在 AXI4 协议中，W 通道不携带 ID 信息。AXI3 中曾有 WID 信号，但 AXI4 将其移除。W 数据与写事务的对应关系通过发送顺序确定：W 数据必须按照 AW 发出的顺序依次传输。

**3. `AXI_TYPEDEF_ALL` 和 `AXI_TYPEDEF_ALL_CT` 的区别是什么？**

- `AXI_TYPEDEF_ALL(__name, ...)` 自动将 req 和 resp 命名为 `__name_req_t` 和 `__name_resp_t`
- `AXI_TYPEDEF_ALL_CT(__name, __req, __rsp, ...)` 允许自定义 req 和 resp 的类型名

例如：
```systemverilog
`AXI_TYPEDEF_ALL(axi, ...)          // → axi_req_t, axi_resp_t
`AXI_TYPEDEF_ALL_CT(axi, my_req, my_rsp, ...)  // → my_req, my_rsp
```

`AXI_TYPEDEF_ALL` 实际上只是调用 `AXI_TYPEDEF_ALL_CT` 并自动填充 req/resp 名称。

---

## 练习 2：手动展开宏

```systemverilog
// `AXI_TYPEDEF_B_CHAN_T(my_b_t, my_id_t, my_user_t) 展开为：

typedef struct packed {
    my_id_t         id;     // logic [1:0]  → 2 位
    axi_pkg::resp_t resp;   //              → 2 位
    my_user_t       user;   // logic        → 1 位
} my_b_t;
```

总位宽 = 2 + 2 + 1 = **5 位**

---

## 练习 3：req_t 字段归属

| # | 信号 | 归属 | 理由 |
|---|------|------|------|
| 1 | `aw_valid` | `req_t` | AW 的 valid 由 Master 驱动（M→S 方向），属于请求 |
| 2 | `aw_ready` | `resp_t` | AW 的 ready 由 Slave 驱动（S→M 方向），属于响应 |
| 3 | `r_valid` | `resp_t` | R 通道的 valid 由 Slave 驱动（S→M 方向），属于响应 |
| 4 | `r_ready` | `req_t` | R 通道的 ready 由 Master 驱动（M→S 方向），属于请求 |
| 5 | `b_valid` | `resp_t` | B 通道的 valid 由 Slave 驱动（S→M 方向），属于响应 |
| 6 | `w.data` | `req_t` | W 通道数据由 Master 驱动（M→S 方向），属于请求 |

规律总结：
- AW/W/AR 通道是 M→S 方向：其 **data + valid** 在 `req_t`，**ready** 在 `resp_t`
- B/R 通道是 S→M 方向：其 **data + valid** 在 `resp_t`，**ready** 在 `req_t`

---

## 练习 4：AXI4 vs Lite 对比

**1. AXI4-Lite 的 AW 通道为什么不需要 `len`、`size`、`burst` 字段？**

AXI4-Lite 只支持**单拍传输**，不支持 burst。因此：
- `len` 隐含为 0（恒定 1 拍）
- `size` 隐含为数据总线宽度（每拍传输全部字节）
- `burst` 无意义（只有一拍，无需定义地址递增模式）

同理，`lock`（独占访问需要多拍）、`cache`（缓存行为对单拍意义有限）、`qos`、`region`、`atop`、`id`、`user` 在 Lite 简化协议中都被省略。

**2. AXI4-Lite 的 W 通道为什么不需要 `last` 字段？**

`last` 信号用于标记 burst 传输中的最后一拍数据。由于 AXI4-Lite 只支持单拍传输，每次写数据就是唯一的一拍，隐含 `last=1`，因此不需要显式的 `last` 字段。

**3. AXI4-Lite 的 `AXI_LITE_TYPEDEF_ALL` 为什么只需要 3 个基础类型参数而不是 5 个？**

AXI4 版本需要 5 个参数：`addr_t, id_t, data_t, strb_t, user_t`

AXI4-Lite 省略了 `id_t` 和 `user_t`：
- **没有 `id_t`**：Lite 不支持事务 ID，所有事务按顺序处理，不支持乱序完成
- **没有 `user_t`**：Lite 不支持用户自定义 sideband 信号

因此 Lite 只需要：`addr_t, data_t, strb_t`。

---

## 练习 5：位宽计算

配置：AddrWidth=48, DataWidth=256, IdWidth=8, UserWidth=4

### 五个通道 struct 位宽

**AW 通道：**
```
id(8) + addr(48) + len(8) + size(3) + burst(2) + lock(1) + cache(4)
+ prot(3) + qos(4) + region(4) + atop(6) + user(4)
= 8 + 48 + 8 + 3 + 2 + 1 + 4 + 3 + 4 + 4 + 6 + 4 = 95 位
```

**W 通道：**
```
data(256) + strb(256/8=32) + last(1) + user(4)
= 256 + 32 + 1 + 4 = 293 位
```

**B 通道：**
```
id(8) + resp(2) + user(4)
= 8 + 2 + 4 = 14 位
```

**AR 通道：**
```
id(8) + addr(48) + len(8) + size(3) + burst(2) + lock(1) + cache(4)
+ prot(3) + qos(4) + region(4) + user(4)
= 8 + 48 + 8 + 3 + 2 + 1 + 4 + 3 + 4 + 4 + 4 = 89 位
（比 AW 少 atop 的 6 位：95 - 6 = 89）
```

**R 通道：**
```
id(8) + data(256) + resp(2) + last(1) + user(4)
= 8 + 256 + 2 + 1 + 4 = 271 位
```

### 聚合 struct 位宽

**axi_req_t：**
```
aw(95) + aw_valid(1) + w(293) + w_valid(1) + b_ready(1)
+ ar(89) + ar_valid(1) + r_ready(1)
= 95 + 1 + 293 + 1 + 1 + 89 + 1 + 1 = 482 位
```

**axi_resp_t：**
```
aw_ready(1) + ar_ready(1) + w_ready(1) + b_valid(1) + b(14)
+ r_valid(1) + r(271)
= 1 + 1 + 1 + 1 + 14 + 1 + 271 = 290 位
```

### 总结

| 类型 | 位宽 |
|------|------|
| `axi_aw_chan_t` | 95 |
| `axi_w_chan_t` | 293 |
| `axi_b_chan_t` | 14 |
| `axi_ar_chan_t` | 89 |
| `axi_r_chan_t` | 271 |
| **`axi_req_t`** | **482** |
| **`axi_resp_t`** | **290** |

---

## 练习 6：实践练习

在 `test/` 目录中搜索，可以找到多个使用 `AXI_TYPEDEF_ALL` 的文件。以 `test/tb_axi_bus_compare.sv` 为例分析：

```bash
$ grep -rn 'AXI_TYPEDEF_ALL' test/
test/tb_axi_bus_compare.sv:62:  `AXI_TYPEDEF_ALL(axi, addr_t, id_t, data_t, strb_t, user_t)
test/tb_axi_slave_compare.sv:64:  `AXI_TYPEDEF_ALL(axi, addr_t, id_t, data_t, strb_t, user_t)
```

**1. 定义了哪些基础类型参数？**

`tb_axi_bus_compare.sv` 中（约第 50-60 行）：

```systemverilog
localparam int unsigned AddrWidth = 32;
localparam int unsigned DataWidth = 64;
localparam int unsigned IdWidth   = 4;
localparam int unsigned UserWidth = 2;

typedef logic [AddrWidth-1:0]   addr_t;   // logic [31:0]
typedef logic [DataWidth-1:0]   data_t;   // logic [63:0]
typedef logic [DataWidth/8-1:0] strb_t;   // logic [7:0]
typedef logic [IdWidth-1:0]     id_t;     // logic [3:0]
typedef logic [UserWidth-1:0]   user_t;   // logic [1:0]
```

**2. 生成的类型名前缀是什么？**

前缀是 `axi`，因此生成的类型为：
- `axi_aw_chan_t`, `axi_w_chan_t`, `axi_b_chan_t`, `axi_ar_chan_t`, `axi_r_chan_t`
- `axi_req_t`, `axi_resp_t`

**3. 这些类型被传给了哪个核心模块？**

被传给 `axi_bus_compare` 模块：

```systemverilog
axi_bus_compare #(
    .AxiIdWidth  ( IdWidth   ),
    .aw_chan_t   ( axi_aw_chan_t ),
    .w_chan_t    ( axi_w_chan_t  ),
    .b_chan_t    ( axi_b_chan_t  ),
    .ar_chan_t   ( axi_ar_chan_t ),
    .r_chan_t    ( axi_r_chan_t  ),
    .req_t       ( axi_req_t    ),
    .resp_t      ( axi_resp_t   )
) i_axi_bus_compare ( ... );
```

这正是 4.8 节描述的使用模式：先用宏定义类型，再通过 `parameter type` 传递给核心模块。

---

> **版权声明**：本教材基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库编写，该仓库遵循 Solderpad Hardware License v0.51。
