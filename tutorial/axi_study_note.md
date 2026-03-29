# 第 1 课

## 问题：

### 1. 什么是缓存行填充？

“地址递增到边界后回绕，用于缓存行填充”中，什么是缓存行填充？

### 2. systermverilog结构体定义语法

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
解析：

```systemverilog
typedef struct packed {                                       \
} w_chan_t;
```

- `typedef`：关键字，**自定义新的数据类型别名**（给结构体起名字，后续直接用）；
- `struct`：定义**结构体**（把多个信号打包成一个整体）；
- `packed`：**打包结构体** → 所有成员位宽紧凑拼接，整体作为一个连续的位向量；
- `w_chan_t`：**最终生成的结构体类型名**（由宏的第一个参数指定）；

示例：

```systemverilog
// 调用宏：传入4个参数
`AXI_TYPEDEF_W_CHAN_T(
  my_axi_w_chan_t,  // 新类型名：my_axi_w_chan_t
  logic[31:0],     // data_t：32位数据
  logic[3:0],      // strb_t：4位选通（对应32位数据的4个字节）
  logic[0:0]       // user_t：1位用户信号
)

// 直接使用生成的类型定义变量
my_axi_w_chan_t axi_w_channel; // 定义一个AXI W通道变量

// 直接访问结构体成员
axi_w_channel.data = 32'h12345678;
axi_w_channel.strb = 4'b1111;
axi_w_channel.last = 1'b1;
axi_w_channel.user = 1'b0;
```

### 3. W 通道没有 `wid` 信号

W 通道没有 `wid` 信号（AXI4 中已移除，AXI3 中有）。这意味着：

- W 通道的数据必须与 AW 通道的地址**按顺序**对应
- 如果 Master 先后发出 AW_A 和 AW_B，那么 W 通道的数据必须先是 A 的数据，再是 B 的数据
- 这就是为什么在 `axi_demux`（解复用器）中需要 ID 计数器来追踪 W 的路由目标

`axi_demux`的代码是如何实现的？

4. **AR 通道没有 `aratop` 字段**：原子操作仅通过写通道发起，什么是原子操作。



vld 和 rdy的死锁分析：

**为什么 valid 不能等待 ready？**

考虑以下死锁场景：如果 Master 等 Slave 的 ready 才拉 valid，同时 Slave 等 Master 的 valid 才拉 ready —— 双方互相等待，永远无法完成握手。规定 valid 不依赖 ready 就打破了这个循环。



与 B 通道（每事务一个响应）不同，R 通道**每拍都有响应**：



AXI 允许 Master 在前一个读事务完成之前就发出下一个读地址，这称为**Outstanding Transaction**：