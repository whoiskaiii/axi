# 第 5 课练习答案：赋值宏与接口定义

> **对应课件**：`tutorial/lessons/Lesson_05_Assign_Macros_Interfaces.md`
> **对应源码**：`include/axi/assign.svh`、`include/axi/port.svh`、`src/axi_intf.sv`、`src/axi_cut.sv`

---

## 练习 1：方向判断（基础）

### 1. `` `AXI_ASSIGN_AW(slv, mst) ``

实际方向是：

- `mst.aw_* payload` 和 `mst.aw_valid` 从 `mst` 流向 `slv`
- `slv.aw_ready` 从 `slv` 流回 `mst`

原因：

- AW 是请求通道，payload 和 `valid` 本来就是 `Master -> Slave`
- `ready` 总是反方向返回

对应源码：`include/axi/assign.svh` 第 99-102 行。

### 2. `` `AXI_ASSIGN_B(mst, slv) ``

实际方向是：

- `slv.b_* payload` 和 `slv.b_valid` 从 `slv` 流向 `mst`
- `mst.b_ready` 从 `mst` 流回 `slv`

原因：

- B 是响应通道，payload 和 `valid` 的协议方向是 `Slave -> Master`
- 所以完整总线连线时要写成 `` `AXI_ASSIGN_B(mst, slv) ``

对应源码：`include/axi/assign.svh` 第 107-109 行、第 119-124 行。

### 3. `` `AXI_ASSIGN_TO_REQ(req, axi_if) ``

实际方向是：

- 从 `axi_if` 读取请求方向字段，赋给 `req`

包括：

- AW payload + `aw_valid`
- W payload + `w_valid`
- AR payload + `ar_valid`
- `b_ready`
- `r_ready`

原因：

- “TO_REQ” 的含义是“把接口中的请求方向信号收集到 request struct”

对应源码：`include/axi/assign.svh` 第 247-253 行。

### 4. `` `AXI_ASSIGN_FROM_RESP(axi_if, resp) ``

实际方向是：

- 从 `resp` 取出响应方向字段，驱动到 `axi_if`

包括：

- `aw_ready`
- `ar_ready`
- `w_ready`
- B payload + `b_valid`
- R payload + `r_valid`

原因：

- “FROM_RESP” 的含义是“把 response struct 中的信号送到 interface”

对应源码：`include/axi/assign.svh` 第 195-201 行。

---

## 练习 2：`SET` vs `ASSIGN`（基础）

### 1. `` `AXI_SET_FROM_REQ(my_if, my_req) ``

应该放在过程块内部，例如：

```systemverilog
always_comb begin
  `AXI_SET_FROM_REQ(my_if, my_req)
end
```

原因：

- `AXI_SET_*` 展开后不是 `assign ...`，而是普通过程赋值
- 它依赖当前所在的 `always_comb` / `always_ff` 上下文

对应源码：`include/axi/assign.svh` 第 160-177 行。

### 2. `` `AXI_ASSIGN_FROM_REQ(my_if, my_req) ``

应该放在模块级连续赋值区域，例如：

```systemverilog
`AXI_ASSIGN_FROM_REQ(my_if, my_req)
```

原因：

- `AXI_ASSIGN_*` 展开后包含 `assign`
- 它适合做模块级组合连线，不适合写在过程块内部

对应源码：`include/axi/assign.svh` 第 182-201 行。

### 为什么不能混用

- 在 `always_comb` 中用 `AXI_ASSIGN_*`，会把 `assign` 放进过程块，语法错误
- 在模块级连线处用 `AXI_SET_*`，又缺少过程上下文，同样不成立

本质区别不是“语义不同”，而是“展开后的 Verilog 语法环境不同”。

---

## 练习 3：读 `axi_cut_intf`（中级）

### 1. 为什么需要先 `typedef` 出 `id_t`、`addr_t`、`data_t`、`strb_t`、`user_t`？

因为后面的 typedef 宏需要这些基础类型作为参数：

```systemverilog
`AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)
`AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t)
...
```

也就是说，`axi_cut_intf` 先把位宽参数 `ID_WIDTH`、`ADDR_WIDTH`、`DATA_WIDTH`、`USER_WIDTH` 映射成具体类型，再交给第 4 课学过的 typedef 宏生成 `aw_chan_t`、`axi_req_t`、`axi_resp_t` 等结构体。

对应源码：`src/axi_cut.sv` 第 147-159 行。

### 2. 为什么 `in` 使用 `AXI_BUS.Slave`，而 `out` 使用 `AXI_BUS.Master`？

因为 `axi_cut` 这个模块站在“中间级联模块”的视角看：

- `in` 是它接收上游请求的一侧，所以该端口对本模块来说是 Slave 视角
- `out` 是它向下游发起请求的一侧，所以该端口对本模块来说是 Master 视角

这并不表示整个系统中它是最终 Slave 或最终 Master，而是表示该 wrapper 在这两个接口上的驱动/接收方向。

对应源码：`src/axi_cut.sv` 第 141-144 行。

### 3. 四个赋值宏各自承担什么方向的转换？

```
`AXI_ASSIGN_TO_REQ(slv_req, in)
```

- `in -> slv_req`
- 把输入口 interface 的请求方向字段收集成 `slv_req`

```
`AXI_ASSIGN_FROM_RESP(in, slv_resp)
```

- `slv_resp -> in`
- 把核心模块输出的响应 struct 写回输入侧 interface

```
`AXI_ASSIGN_FROM_REQ(out, mst_req)
```

- `mst_req -> out`
- 把核心模块发向下游的请求 struct 驱动到输出侧 interface

```
`AXI_ASSIGN_TO_RESP(mst_resp, out)
```

- `out -> mst_resp`
- 从输出侧 interface 收集下游返回的响应，送回核心模块

对应源码：`src/axi_cut.sv` 第 161-168 行。

---

## 练习 4：modport 方向表（中级）

根据 `src/axi_intf.sv` 中 `AXI_BUS` 的 `modport Master`：

### 1. `aw_valid`

- `output`

原因：AW 请求由 Master 发出。

### 2. `aw_ready`

- `input`

原因：AW ready 由 Slave 返回给 Master。

### 3. `b_valid`

- `input`

原因：B 响应由 Slave 发出。

### 4. `b_ready`

- `output`

原因：B ready 由 Master 给出。

### 5. `r_data`

- `input`

原因：R 数据由 Slave 返回给 Master。

### 6. `r_ready`

- `output`

原因：R ready 由 Master 给出。

对应源码：`src/axi_intf.sv` 第 85-90 行。

---

## 练习 5：扁平端口场景（高级）

### 1. 工具链兼容性

虽然 `interface` 风格很清晰，但某些工具，尤其是 Vivado IP Integrator 一类流程，对 `SystemVerilog interface` 的支持并不完整或不方便集成。因此仓库仍保留扁平端口宏作为兼容层。

### 2. 信号命名约定

`port.svh` 中的 `AXI_M_PORT` / `AXI_S_PORT` 直接展开为：

- `m_axi_<name>_awvalid`
- `s_axi_<name>_araddr`
- `m_axi_<name>_rdata`

这样的命名对工具、IP 封装器、脚本来说都更友好，也更容易自动识别总线。

对应源码：`include/axi/port.svh` 第 23-67 行、第 73-110 行。

### 3. `struct <-> interface <-> flat ports` 三层转换关系

仓库的真实工程世界并不是只有一种端口风格，而是三层并存：

```
核心模块：struct ports
wrapper / 顶层：interface ports
工具适配层：flat ports
```

`assign.svh` 中的扁平端口宏说明，仓库希望在保持核心模块 `struct` 风格的同时，仍然能接入工具偏好的扁平接口。

所以扁平端口不是“替代 interface”，而是“对工具链的补充适配”。

---

## 练习 6：小型 wrapper 设计（实践）

下面给出一个最小可读版本。它不是完整可综合模块，只是展示第 5 课要求的结构骨架。

```systemverilog
module my_passthrough_intf #(
  parameter int unsigned ADDR_WIDTH = 32,
  parameter int unsigned DATA_WIDTH = 64,
  parameter int unsigned ID_WIDTH   = 4,
  parameter int unsigned USER_WIDTH = 1
) (
  AXI_BUS.Slave  in,
  AXI_BUS.Master out
);

  typedef logic [ID_WIDTH-1:0]     id_t;
  typedef logic [ADDR_WIDTH-1:0]   addr_t;
  typedef logic [DATA_WIDTH-1:0]   data_t;
  typedef logic [DATA_WIDTH/8-1:0] strb_t;
  typedef logic [USER_WIDTH-1:0]   user_t;

  `AXI_TYPEDEF_AW_CHAN_T(aw_chan_t, addr_t, id_t, user_t)
  `AXI_TYPEDEF_W_CHAN_T(w_chan_t, data_t, strb_t, user_t)
  `AXI_TYPEDEF_B_CHAN_T(b_chan_t, id_t, user_t)
  `AXI_TYPEDEF_AR_CHAN_T(ar_chan_t, addr_t, id_t, user_t)
  `AXI_TYPEDEF_R_CHAN_T(r_chan_t, data_t, id_t, user_t)
  `AXI_TYPEDEF_REQ_T(axi_req_t, aw_chan_t, w_chan_t, ar_chan_t)
  `AXI_TYPEDEF_RESP_T(axi_resp_t, b_chan_t, r_chan_t)

  axi_req_t  slv_req,  mst_req;
  axi_resp_t slv_resp, mst_resp;

  `AXI_ASSIGN_TO_REQ(slv_req, in)
  `AXI_ASSIGN_FROM_RESP(in, slv_resp)

  `AXI_ASSIGN_FROM_REQ(out, mst_req)
  `AXI_ASSIGN_TO_RESP(mst_resp, out)

  // 这里用纯透传举例，真实设计中可替换为核心模块实例
  `AXI_ASSIGN_REQ_STRUCT(mst_req, slv_req)
  `AXI_ASSIGN_RESP_STRUCT(slv_resp, mst_resp)

endmodule
```

### 这段代码的逻辑

1. 先根据参数定义基础类型。
2. 再用 typedef 宏生成通道和聚合 struct。
3. `in` 被转换成 `slv_req`。
4. `out` 返回的响应被转换成 `mst_resp`。
5. 中间用 `mst_req` / `slv_resp` 作为核心模块边界。
6. 这里为了最小示意，直接用 `struct -> struct` 宏做透明传递。

### 如果换成真实核心模块

中间两句结构体直连通常会换成：

```systemverilog
my_core #(
  .axi_req_t ( axi_req_t ),
  .axi_resp_t( axi_resp_t )
) i_core (
  .slv_req_i  ( slv_req  ),
  .slv_resp_o ( slv_resp ),
  .mst_req_o  ( mst_req  ),
  .mst_resp_i ( mst_resp )
);
```

这正是 `axi_cut_intf` 的基本模式。

---

## 小结

第 5 课的核心不是“背宏名”，而是建立一个稳定心智模型：

- `interface` 负责端口组织
- `struct` 负责内部抽象
- `assign.svh` 负责在不同端口风格之间搬运信号
- `port.svh` 负责工具兼容

只要这个模型建立起来，后面阅读 `axi_cut_intf`、`axi_mux_intf`、`axi_demux_intf`、`axi_xbar_intf` 之类 wrapper 就会顺畅很多。

---

## 参考提示

如果你做完这些题后还不够稳，建议回看第 5 课正文里的三处内容：

1. `5.3 AXI_ASSIGN 的真正语义`
2. `5.4 struct 与 interface 互转宏`
3. `5.8 在模块中使用赋值宏的完整流程`
