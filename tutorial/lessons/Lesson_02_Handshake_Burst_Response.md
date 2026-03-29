# 第 2 课：握手机制、Burst 类型与响应编码

> **课程系列**：AXI 总线协议 30 课深度教程
> **所属单元**：单元一 — 协议基础与类型系统
> **前置知识**：第 1 课（AXI 协议概述与五通道架构）
> **学习时长**：约 2-3 小时
> **对应源码**：`src/axi_pkg.sv`（第 68-319 行），`src/axi_intf.sv`（第 204-261 行）

---

## 学习目标

完成本课学习后，你将能够：

1. 运用 Valid/Ready 握手的依赖规则分析死锁场景
2. 精确计算三种 Burst 类型（FIXED / INCR / WRAP）的地址序列
3. 说明 4KB 边界限制的含义以及违反时的后果
4. 解释四种响应编码的语义和优先级规则
5. 阅读并理解 `axi_pkg.sv` 中的地址计算辅助函数
6. 运用 `resp_precedence` 函数判断响应合并结果

---

## 目录

- [2.1 握手机制深入](#21-握手机制深入)
- [2.2 Burst 传输基础](#22-burst-传输基础)
- [2.3 FIXED Burst — 固定地址传输](#23-fixed-burst--固定地址传输)
- [2.4 INCR Burst — 递增地址传输](#24-incr-burst--递增地址传输)
- [2.5 WRAP Burst — 回绕地址传输](#25-wrap-burst--回绕地址传输)
- [2.6 4KB 边界限制](#26-4kb-边界限制)
- [2.7 响应编码与优先级](#27-响应编码与优先级)
- [2.8 源码精读：地址计算函数](#28-源码精读地址计算函数)
- [2.9 源码映射](#29-源码映射)
- [2.10 关键术语表](#210-关键术语表)
- [2.11 课后练习](#211-课后练习)
- [2.12 下一课预告](#212-下一课预告)

---

## 2.1 握手机制深入

第 1 课介绍了 Valid/Ready 的基本握手规则。本节深入探讨更多细节和边界情况。

### 2.1.1 握手规则的形式化表述

AXI 规范（A3.2.1 节）用三条规则定义了握手协议：

```
  ┌─────────────────────────────────────────────────────────────┐
  │  规则 R1: 发送方拉高 valid 后，必须保持到握手完成             │
  │           valid 一旦拉高，不允许在 ready 为低时撤回            │
  │                                                             │
  │  规则 R2: valid 拉高后，所有数据/控制信号必须保持稳定          │
  │           直到握手完成（valid && ready 同时为高）              │
  │                                                             │
  │  规则 R3: valid 不可依赖 ready                               │
  │           （Source 必须独立决定何时拉高 valid）                │
  │                                                             │
  │  附加:   ready 可以依赖 valid                                │
  │           （Destination 可以等看到 valid 后才拉 ready）        │
  │           ready 也可以不依赖 valid（提前拉高 ready）           │
  └─────────────────────────────────────────────────────────────┘
```

### 2.1.2 规则 R2 的仿真断言

本仓库在 `AXI_BUS_DV` 接口中用 SystemVerilog 断言检查规则 R2：

```systemverilog
// src/axi_intf.sv, 第 206-251 行
// 以 AW 通道为例：当 aw_valid 为高且 aw_ready 为低时，
// 下一个时钟沿所有 AW 信号必须保持不变

assert property (@(posedge clk_i) (aw_valid && !aw_ready |=> $stable(aw_id)));
assert property (@(posedge clk_i) (aw_valid && !aw_ready |=> $stable(aw_addr)));
assert property (@(posedge clk_i) (aw_valid && !aw_ready |=> $stable(aw_len)));
assert property (@(posedge clk_i) (aw_valid && !aw_ready |=> $stable(aw_size)));
assert property (@(posedge clk_i) (aw_valid && !aw_ready |=> $stable(aw_burst)));
// ...（每个信号都有对应的断言）
// 并且 valid 本身也必须保持高:
assert property (@(posedge clk_i) (aw_valid && !aw_ready |=> aw_valid));
```

> **断言解读**：`|=>` 是 SystemVerilog 的蕴含操作符（implication），含义是"如果左边条件为真，则在下一个时钟沿右边条件也必须为真"。`$stable()` 检查信号值相对上一个时钟沿没有变化。

### 2.1.3 死锁分析

理解规则 R3 为什么至关重要，考虑以下错误设计：

```
  错误设计（会导致死锁）：
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Master 端:  awvalid = awready;   // valid 等待 ready       │
  │  Slave 端:   awready = awvalid;   // ready 等待 valid       │
  │                                                             │
  │  结果: 两个信号互相依赖，永远都是 0，握手永远无法完成         │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  正确设计：
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Master 端:  awvalid = has_pending_request;  // 独立判断     │
  │  Slave 端:   awready = !fifo_full;           // 独立判断     │
  │                      或                                     │
  │  Slave 端:   awready = awvalid & !fifo_full; // 可以看valid  │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 2.1.4 连续握手（Back-to-Back）

高性能设计中，连续传输不应有气泡（空闲周期）：

```
          ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
 clk      │   │ │   │ │   │ │   │ │   │ │   │ │   │ │   │
       ───┘   └─┘   └─┘   └─┘   └─┘   └─┘   └─┘   └─┘   └───

          ┌───────────────────────────────────────┐
 valid ───┘                                       └────────────
 data  ═══╡ DATA_A ╞╡ DATA_B ╞╡ DATA_C ╞╡ DATA_D ╞═══════════
          ┌───────────────────────────────────────┐
 ready ───┘                                       └────────────
          ▲         ▲         ▲         ▲
       握手 A    握手 B    握手 C    握手 D
     (每个周期完成一次握手 — 最大吞吐率)
```

### 2.1.5 带反压的握手

当 Slave 无法连续接收时，通过拉低 ready 实施反压（Back-pressure）：

```
          ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
 clk      │   │ │   │ │   │ │   │ │   │ │   │ │   │ │   │
       ───┘   └─┘   └─┘   └─┘   └─┘   └─┘   └─┘   └─┘   └───

          ┌───────────────────────────────────────────────┐
 valid ───┘                                               └────
 data  ═══╡ DATA_A ╞══════════╡ DATA_B ╞══════════╡ DATA_C ╞══
          ┌─────────┐         ┌─────────┐         ┌─────────┐
 ready ───┘         └─────────┘         └─────────┘         └──
          ▲                   ▲                   ▲
       握手 A              握手 B              握手 C
          (Slave 每隔一拍才 ready — 吞吐率降为 50%)
```

注意：根据规则 R2，valid 为高期间数据必须保持稳定。当 DATA_A 握手完成后，Master 可以立即将数据切换为 DATA_B，即使此时 ready 已经拉低。

---

## 2.2 Burst 传输基础

AXI 的 Burst 传输允许用一次地址握手传输多拍数据，大幅提高总线效率。

### 2.2.1 三个关键参数

| 参数 | 信号 | 位宽 | 含义 |
|------|------|------|------|
| Burst Length | AxLEN | 8 bit | 传输拍数 = AxLEN + 1（范围 1-256） |
| Burst Size | AxSIZE | 3 bit | 每拍字节数 = 2^AxSIZE（范围 1-128） |
| Burst Type | AxBURST | 2 bit | 地址变化方式：FIXED/INCR/WRAP |

### 2.2.2 总传输量计算

```
总传输字节数 = (AxLEN + 1) × 2^AxSIZE
            = Burst_Length × Number_Bytes
```

**示例**：

| AxLEN | AxSIZE | 拍数 | 每拍字节 | 总字节 |
|-------|--------|------|----------|--------|
| 0 | 2 | 1 | 4 | 4 |
| 3 | 2 | 4 | 4 | 16 |
| 7 | 3 | 8 | 8 | 64 |
| 15 | 2 | 16 | 4 | 64 |
| 255 | 3 | 256 | 8 | 2048 |

### 2.2.3 窄传输（Narrow Transfer）

当 AxSIZE 小于数据总线宽度时，称为**窄传输**。此时每拍只使用数据总线的部分字节：

```
64 位数据总线（8 字节），AxSIZE = 2（4 字节/拍）：

数据总线:  [byte7] [byte6] [byte5] [byte4] [byte3] [byte2] [byte1] [byte0]

beat 0:    ────── 无效 ──────────────     [  有效数据 (4字节)  ]
wstrb:         0       0       0       0       1       1       1       1

beat 1:    [  有效数据 (4字节)  ]     ────── 无效 ──────────────
wstrb:         1       1       1       1       0       0       0       0
```

窄传输中，有效字节的位置由地址的低位决定（地址对齐）。

### 2.2.4 对齐地址

**对齐地址（Aligned Address）** 是将起始地址向下对齐到传输尺寸的边界：

```
Aligned_Address = (Start_Address / Number_Bytes) × Number_Bytes
                = (Start_Address >> AxSIZE) << AxSIZE
```

在源码中的实现：

```systemverilog
// src/axi_pkg.sv, 第 126-128 行
function automatic largest_addr_t aligned_addr(largest_addr_t addr, size_t size);
    return (addr >> size) << size;
endfunction
```

**示例**：

| Start_Address | AxSIZE | Number_Bytes | Aligned_Address |
|---------------|--------|--------------|-----------------|
| 0x0003 | 2 (4B) | 4 | 0x0000 |
| 0x1005 | 2 (4B) | 4 | 0x1004 |
| 0x0007 | 3 (8B) | 8 | 0x0000 |
| 0x1010 | 2 (4B) | 4 | 0x1010（已对齐）|

---

## 2.3 FIXED Burst — 固定地址传输

### 2.3.1 定义

```systemverilog
// src/axi_pkg.sv, 第 68-75 行
/// In a fixed burst:
/// - The address is the same for every transfer in the burst.
/// - The byte lanes that are valid are constant for all beats in the burst.
///   However, within those byte lanes, the actual bytes that have `wstrb`
///   asserted can differ for each beat in the burst.
/// This burst type is used for repeated accesses to the same location such as
/// when loading or emptying a FIFO.
localparam BURST_FIXED = 2'b00;
```

### 2.3.2 地址计算

所有拍使用相同的地址：

```
Address_N = Start_Address    （对所有 N）
```

### 2.3.3 典型应用 — FIFO 访问

```
Master 从 FIFO 连续读取 4 个数据：
  AxADDR  = 0x4000 (FIFO 数据寄存器地址)
  AxLEN   = 3 (4 拍)
  AxSIZE  = 2 (4 字节)
  AxBURST = FIXED

  beat 0: addr = 0x4000, 读出 FIFO 第 1 个数据
  beat 1: addr = 0x4000, 读出 FIFO 第 2 个数据
  beat 2: addr = 0x4000, 读出 FIFO 第 3 个数据
  beat 3: addr = 0x4000, 读出 FIFO 第 4 个数据
            ↑ 地址始终不变
```

### 2.3.4 FIXED Burst 限制

- Burst 长度不能超过 16 拍（AxLEN ≤ 15）
- 所有拍使用相同的字节通道（byte lanes）

---

## 2.4 INCR Burst — 递增地址传输

### 2.4.1 定义

```systemverilog
// src/axi_pkg.sv, 第 76-81 行
/// In an incrementing burst, the address for each transfer in the burst is an
/// increment of the address for the previous transfer. The increment value
/// depends on the size of the transfer.
/// This burst type is used for accesses to normal sequential memory.
localparam BURST_INCR  = 2'b01;
```

### 2.4.2 地址计算

```
beat 0:  Address_0 = Start_Address
beat N:  Address_N = Aligned_Address + N × Number_Bytes    （N ≥ 1）
```

> **注意**：从第 1 拍开始，地址基于**对齐地址**递增，而非起始地址。如果起始地址未对齐，第 0 拍是唯一使用未对齐地址的拍。

### 2.4.3 地址对齐的起始 — 完整示例

```
参数: Start_Address = 0x0002, AxSIZE = 2 (4B), AxLEN = 3
      Aligned_Address = 0x0000

beat 0: addr = 0x0002  (起始地址，未对齐)
beat 1: addr = 0x0000 + 1×4 = 0x0004  (从对齐地址递增)
beat 2: addr = 0x0000 + 2×4 = 0x0008
beat 3: addr = 0x0000 + 3×4 = 0x000C
```

### 2.4.4 地址对齐的起始 — 已对齐示例

```
参数: Start_Address = 0x1000, AxSIZE = 2 (4B), AxLEN = 3
      Aligned_Address = 0x1000

beat 0: addr = 0x1000
beat 1: addr = 0x1000 + 1×4 = 0x1004
beat 2: addr = 0x1000 + 2×4 = 0x1008
beat 3: addr = 0x1000 + 3×4 = 0x100C
```

### 2.4.5 地址可视化

```
内存地址空间（每格 4 字节）：

    0x1000   0x1004   0x1008   0x100C   0x1010   0x1014
   ┌────────┬────────┬────────┬────────┬────────┬────────┐
   │ beat 0 │ beat 1 │ beat 2 │ beat 3 │        │        │
   └────────┴────────┴────────┴────────┴────────┴────────┘
   ▲                                    ▲
   Start_Address                        End (不含)
   0x1000                               0x1010

   总传输 = 4 拍 × 4 字节 = 16 字节
```

---

## 2.5 WRAP Burst — 回绕地址传输

WRAP 是三种 Burst 类型中最复杂的，但在缓存系统中至关重要。

### 2.5.1 定义

```systemverilog
// src/axi_pkg.sv, 第 82-87 行
/// A wrapping burst is similar to an incrementing burst, except that the address
/// wraps around to a lower address if an upper address limit is reached.
/// The following restrictions apply to wrapping bursts:
/// - The start address must be aligned to the size of each transfer.
/// - The length of the burst must be 2, 4, 8, or 16 transfers.
localparam BURST_WRAP  = 2'b10;
```

### 2.5.2 WRAP 的限制条件

1. **起始地址必须对齐**到传输尺寸（AxSIZE 对应的字节数）
2. **Burst 长度只能是 2、4、8 或 16 拍**（AxLEN 只能是 1、3、7、15）

### 2.5.3 关键概念 — 回绕边界

```
Wrap_Boundary = 回绕地址空间的下界
Container_Size = Number_Bytes × Burst_Length = 2^AxSIZE × (AxLEN + 1)
Upper_Wrap_Boundary = Wrap_Boundary + Container_Size
```

地址递增到 Upper_Wrap_Boundary 时，回绕到 Wrap_Boundary。

### 2.5.4 完整示例

```
参数: Start_Address = 0x0024, AxSIZE = 2 (4B), AxLEN = 3 (4拍)
      Number_Bytes = 4
      Burst_Length = 4
      Container_Size = 4 × 4 = 16 字节
      Wrap_Boundary = 0x0020  (= INT(0x0024 / 16) × 16)
      Upper_Wrap   = 0x0020 + 16 = 0x0030

地址计算:
  beat 0: addr = 0x0024  (起始)
  beat 1: addr = 0x0028  (递增)
  beat 2: addr = 0x002C  (递增)
  beat 3: addr = 0x0020  (达到 0x0030，回绕到 Wrap_Boundary!)
```

可视化：

```
Container (16 字节):
    0x0020   0x0024   0x0028   0x002C
   ┌────────┬────────┬────────┬────────┐
   │ beat 3 │ beat 0 │ beat 1 │ beat 2 │
   └────────┴────────┴────────┴────────┘
   ▲ Wrap_Boundary    ↑ Start   Upper_Wrap ▲
   (回绕目标)                              (= 0x0030)

   传输顺序: 0x24 → 0x28 → 0x2C → 0x20（回绕）
```

### 2.5.5 WRAP 的典型应用 — 缓存行填充

处理器缓存未命中时，需要从内存加载整个缓存行。WRAP burst 的优势在于**优先加载处理器需要的数据**：

```
场景: 缓存行 32 字节，处理器访问地址 0x0018

INCR burst（从缓存行起始加载）:
  0x0000 → 0x0004 → 0x0008 → 0x000C → 0x0010 → 0x0014 → [0x0018] → 0x001C
                                                          ↑ 处理器需要的数据
                                                          在第 7 拍才到达！

WRAP burst（从访问地址开始加载）:
  [0x0018] → 0x001C → 0x0000 → 0x0004 → 0x0008 → 0x000C → 0x0010 → 0x0014
  ↑ 处理器需要的数据
  第 1 拍就到达！ → 处理器可以更早恢复执行
```

### 2.5.6 源码中的回绕边界计算

```systemverilog
// src/axi_pkg.sv, 第 134-163 行
function automatic largest_addr_t wrap_boundary(
    largest_addr_t addr, size_t size, len_t len);
  largest_addr_t wrap_addr;

  // Wrap_Boundary = INT(Start_Address / (Number_Bytes × Burst_Length))
  //                 × (Number_Bytes × Burst_Length)
  // 利用移位实现除法和乘法（因为 len+1 只能是 2 的幂）
  unique case (len)
    len_t'(4'b1   ): // Burst_Length = 2
      wrap_addr = (addr >> (unsigned'(size) + 1)) << (unsigned'(size) + 1);
    len_t'(4'b11  ): // Burst_Length = 4
      wrap_addr = (addr >> (unsigned'(size) + 2)) << (unsigned'(size) + 2);
    len_t'(4'b111 ): // Burst_Length = 8
      wrap_addr = (addr >> (unsigned'(size) + 3)) << (unsigned'(size) + 3);
    len_t'(4'b1111): // Burst_Length = 16
      wrap_addr = (addr >> (unsigned'(size) + 4)) << (unsigned'(size) + 4);
    default: wrap_addr = '0;
  endcase
  return wrap_addr;
endfunction
```

> **实现精解**：`(addr >> (size + N)) << (size + N)` 等效于将地址对齐到 `2^(size+N)` 字节边界。其中 `2^size` 是每拍字节数，`2^N` 是 burst 长度，它们的乘积就是 Container_Size。这是一个很优雅的位操作技巧。

---

## 2.6 4KB 边界限制

### 2.6.1 规则说明

AXI 规范规定：

> **一个 Burst 传输不能跨越 4KB（0x1000）地址边界。**

这是 AXI 中最重要的协议约束之一。

### 2.6.2 为什么是 4KB？

4KB 是大多数操作系统和 MMU（内存管理单元）使用的**最小页面大小**。一个 4KB 页面内的虚拟地址到物理地址的映射是连续的，但跨越页面边界后映射可能完全不同。如果允许 burst 跨越 4KB：

- 一个 burst 可能需要两次 TLB 查找
- 物理地址可能不连续，无法用单个 burst 完成
- 互连和从机的设计会显著复杂化

### 2.6.3 哪些 Burst 类型受影响？

| Burst 类型 | 是否可能跨越 4KB | 说明 |
|------------|------------------|------|
| FIXED | 不可能 | 地址不变，不会跨越任何边界 |
| INCR | **可能** | 需要程序员/硬件确保不跨越 |
| WRAP | 不可能 | 地址回绕，始终在 Container 内 |

**INCR 是唯一需要关注 4KB 边界的类型。**

### 2.6.4 计算是否跨越 4KB

```
最后一拍地址 = Aligned_Address + AxLEN × Number_Bytes
检查: Start_Address[31:12] == 最后一拍地址[31:12]
      （即两者在同一个 4KB 页面内）
```

### 2.6.5 仿真断言

本仓库在 `AXI_BUS_DV` 中内置了 4KB 边界检查：

```systemverilog
// src/axi_intf.sv, 第 252-261 行
// AW 通道的 4KB 检查：
assert property (@(posedge clk_i) aw_valid |->
    (aw_burst != axi_pkg::BURST_INCR) || (
    axi_pkg::beat_addr(aw_addr, aw_size, aw_len, aw_burst, 0) >> 12 ==
    axi_pkg::beat_addr(aw_addr, aw_size, aw_len, aw_burst, aw_len) >> 12
)) else $error("AW burst crossing 4 KiB page boundary detected!");

// AR 通道同理（第 258-261 行）
```

> **断言解读**：将第 0 拍和最后一拍的地址右移 12 位（除以 4096），如果相等则在同一个 4KB 页面内。仅对 INCR burst 检查（FIXED 和 WRAP 不会跨越）。

### 2.6.6 违规示例

```
违规: Start_Address = 0x0FF0, AxSIZE = 2 (4B), AxLEN = 7 (8拍)

beat 0: 0x0FF0
beat 1: 0x0FF4
beat 2: 0x0FF8
beat 3: 0x0FFC  ← 4KB 页面末尾
beat 4: 0x1000  ← 跨入下一个 4KB 页面！违规！
beat 5: 0x1004
beat 6: 0x1008
beat 7: 0x100C

修复方案: 拆分为两个 burst
  burst 1: Start = 0x0FF0, LEN = 3 (4拍), 0xFF0→0xFFC
  burst 2: Start = 0x1000, LEN = 3 (4拍), 0x1000→0x100C
```

---

## 2.7 响应编码与优先级

### 2.7.1 四种响应

```systemverilog
// src/axi_pkg.sv, 第 89-100 行

// Normal access success.
localparam RESP_OKAY   = 2'b00;  // 正常完成

// Exclusive access okay.
localparam RESP_EXOKAY = 2'b01;  // 独占访问成功

// Slave error.
localparam RESP_SLVERR = 2'b10;  // 从机错误

// Decode error.
localparam RESP_DECERR = 2'b11;  // 译码错误
```

### 2.7.2 各响应的详细语义

#### OKAY (2'b00) — 正常完成

- 最常见的响应
- 写事务：数据已成功写入
- 读事务：数据已成功返回
- **注意**：对于独占访问，OKAY 表示独占访问**失败**（非独占完成）

#### EXOKAY (2'b01) — 独占访问成功

- 仅在独占访问（Exclusive Access）中使用
- 表示独占读或独占写成功
- 独占访问用于多处理器系统的原子操作（配合 AxLOCK 信号）

#### SLVERR (2'b10) — 从机错误

- 事务成功到达从机，但从机返回错误
- 可能原因：
  - 写入只读寄存器
  - 访问了从机不支持的地址范围
  - 传输尺寸（AxSIZE）不被从机支持
  - FIFO 溢出等

#### DECERR (2'b11) — 译码错误

- 通常由**互连**（而非从机）生成
- 表示事务的地址无法匹配任何从机
- 在本仓库中，`axi_err_slv` 模块专门用于生成这种响应：

```
  ┌─────────┐         ┌──────────┐
  │ Master  │────────►│ axi_xbar │──┬──► Slave 0 (0x0000-0x0FFF)
  │(addr=   │         │          │  ├──► Slave 1 (0x1000-0x1FFF)
  │ 0x5000) │         │          │  └──► axi_err_slv ──► DECERR
  └─────────┘         └──────────┘      (0x5000 不在任何映射中)
```

### 2.7.3 B 通道 vs R 通道的响应差异

| 特性 | B 通道（写响应） | R 通道（读响应） |
|------|------------------|------------------|
| 响应数量 | 整个事务一个 | **每拍一个** |
| 可否混合 | 不适用 | 同一 burst 不同拍可有不同 resp |

读事务中，每拍可以有不同的响应状态。例如：

```
4 拍读 burst:
  beat 0: rresp = OKAY    ← 前 3 拍正常
  beat 1: rresp = OKAY
  beat 2: rresp = OKAY
  beat 3: rresp = SLVERR  ← 最后一拍出错
```

### 2.7.4 响应优先级（`resp_precedence`）

当互连需要**合并**多个响应时（例如 burst splitter 将一个 burst 拆分为多个子事务，需要将子事务的多个 B 响应合并为一个），需要一个优先级规则：

```
  优先级: DECERR > SLVERR > OKAY > EXOKAY
  （越严重的错误优先级越高）
```

源码实现：

```systemverilog
// src/axi_pkg.sv, 第 282-319 行
/// RESP precedence: DECERR > SLVERR > OKAY > EXOKAY.
/// This is not defined in the AXI standard but depends on the implementation.
/// Rationale:
/// - EXOKAY means an exclusive access was successful, whereas OKAY means it
///   was not. Thus if OKAY and EXOKAY merge, OKAY precedes.
/// - Both DECERR and SLVERR mean (part of) a transaction was unsuccessful.
///   Thus both precede OKAY.
/// - DECERR means (part of) a transaction could not be routed to a slave,
///   whereas SLVERR means the transaction reached a slave but caused an error.
///   Thus DECERR precedes SLVERR.

function automatic resp_t resp_precedence(resp_t resp_a, resp_t resp_b);
    unique case (resp_a)
      RESP_OKAY: begin
        if (resp_b == RESP_EXOKAY) return resp_a;  // OKAY > EXOKAY
        else return resp_b;                          // 其他都优先于 OKAY
      end
      RESP_EXOKAY: begin
        return resp_b;   // 任何响应都优先于 EXOKAY
      end
      RESP_SLVERR: begin
        if (resp_b == RESP_DECERR) return resp_b;   // DECERR > SLVERR
        else return resp_a;                           // SLVERR > OKAY/EXOKAY
      end
      RESP_DECERR: begin
        return resp_a;   // DECERR 最高优先级
      end
    endcase
endfunction
```

### 2.7.5 响应合并示例

```
场景: axi_burst_splitter 将一个 4 拍 burst 写入拆分为 4 个单拍写入

子事务 0 的 B 响应: OKAY
子事务 1 的 B 响应: OKAY
子事务 2 的 B 响应: SLVERR   ← 从机返回错误
子事务 3 的 B 响应: OKAY

合并过程:
  merge(OKAY, OKAY)   = OKAY
  merge(OKAY, SLVERR) = SLVERR    ← SLVERR 优先于 OKAY
  merge(SLVERR, OKAY) = SLVERR    ← SLVERR 仍然保持

最终返回给 Master 的 B 响应: SLVERR
（只要有一个子事务出错，整个事务就报错）
```

---

## 2.8 源码精读：地址计算函数

`axi_pkg.sv` 中的地址计算函数是理解 Burst 机制的钥匙。

### 2.8.1 `num_bytes` — 每拍字节数

```systemverilog
// src/axi_pkg.sv, 第 115-118 行
/// Maximum number of bytes per burst, as specified by `size`.
function automatic shortint unsigned num_bytes(size_t size);
    return shortint'(1 << size);
endfunction
```

等效于 `2^size`。利用左移 1 实现 2 的幂运算。

### 2.8.2 `beat_addr` — 任意拍的地址

```systemverilog
// src/axi_pkg.sv, 第 165-196 行
function automatic largest_addr_t beat_addr(
    largest_addr_t addr,         // 起始地址
    size_t         size,         // AxSIZE
    len_t          len,          // AxLEN
    burst_t        burst,        // AxBURST
    shortint unsigned i_beat     // 第几拍（从 0 开始）
);
    largest_addr_t ret_addr = addr;
    largest_addr_t wrp_bond = '0;

    if (burst == BURST_WRAP) begin
      wrp_bond = wrap_boundary(addr, size, len);
    end

    if (i_beat != 0 && burst != BURST_FIXED) begin
      // INCR 和 WRAP 的基本公式:
      // Address_N = Aligned_Address + N × Number_Bytes
      ret_addr = aligned_addr(addr, size) + i_beat * num_bytes(size);

      // WRAP 的额外处理: 地址超过上界时回绕
      if (burst == BURST_WRAP &&
          ret_addr >= wrp_bond + (num_bytes(size) * (largest_addr_t'(len) + 1)))
      begin
        // 减去 Container_Size 实现回绕
        ret_addr = ret_addr - (num_bytes(size) * (largest_addr_t'(len) + 1));
      end
    end
    return ret_addr;
endfunction
```

**函数执行流程**：

```
输入: addr=0x0024, size=2, len=3, burst=WRAP, i_beat=3

1. wrp_bond = wrap_boundary(0x0024, 2, 3) = 0x0020
2. i_beat != 0 && burst != FIXED → 进入计算
3. ret_addr = aligned_addr(0x0024, 2) + 3 × 4
            = 0x0024 + 12 = 0x0030
4. 检查 WRAP: 0x0030 >= 0x0020 + (4 × 4) = 0x0030 → 是！
5. ret_addr = 0x0030 - 16 = 0x0020 → 回绕到边界
6. 返回 0x0020 ✓
```

### 2.8.3 `beat_lower_byte` 和 `beat_upper_byte`

这两个函数计算每拍在数据总线上的**有效字节范围**，用于生成 `wstrb`：

```systemverilog
// src/axi_pkg.sv, 第 198-216 行
function automatic shortint unsigned beat_lower_byte(...);
    // 返回该拍在数据总线内的最低有效字节索引
endfunction

function automatic shortint unsigned beat_upper_byte(...);
    // 返回该拍在数据总线内的最高有效字节索引
    // 第 0 拍特殊处理（可能未对齐）
    // 后续拍: lower_byte + num_bytes(size) - 1
endfunction
```

**示例**（64 位总线 = 8 字节，size=2 即 4 字节传输）：

```
beat 0 (addr=0x0004): lower=4, upper=7  → wstrb = 8'b1111_0000
beat 1 (addr=0x0008): lower=0, upper=3  → wstrb = 8'b0000_1111
beat 2 (addr=0x000C): lower=4, upper=7  → wstrb = 8'b1111_0000
```

---

## 2.9 源码映射

### 2.9.1 本课涉及的源码位置

| 概念 | 文件 | 行号 | 内容 |
|------|------|------|------|
| Burst 类型常量 | `src/axi_pkg.sv` | 68-87 | BURST_FIXED/INCR/WRAP 定义及注释 |
| 响应类型常量 | `src/axi_pkg.sv` | 89-100 | RESP_OKAY/EXOKAY/SLVERR/DECERR |
| `num_bytes()` | `src/axi_pkg.sv` | 115-118 | 计算每拍字节数 |
| `largest_addr_t` | `src/axi_pkg.sv` | 123 | 128 位通用地址类型 |
| `aligned_addr()` | `src/axi_pkg.sv` | 126-128 | 地址对齐计算 |
| `wrap_boundary()` | `src/axi_pkg.sv` | 134-163 | WRAP 回绕边界计算 |
| `beat_addr()` | `src/axi_pkg.sv` | 166-196 | 任意拍地址计算 |
| `beat_lower_byte()` | `src/axi_pkg.sv` | 199-204 | 有效字节下界 |
| `beat_upper_byte()` | `src/axi_pkg.sv` | 207-216 | 有效字节上界 |
| `resp_precedence()` | `src/axi_pkg.sv` | 292-319 | 响应优先级合并 |
| 握手稳定性断言 | `src/axi_intf.sv` | 206-251 | AXI_BUS_DV 中的协议检查 |
| 4KB 边界断言 | `src/axi_intf.sv` | 252-261 | INCR burst 4KB 检查 |

### 2.9.2 `largest_addr_t` 的设计考量

```systemverilog
// src/axi_pkg.sv, 第 120-123 行
/// An overly long address type.
/// It lets us define functions that work generically for shorter addresses.
/// We rely on the synthesizer to optimize the unused bits away.
typedef logic [127:0] largest_addr_t;
```

这是一个实用的工程技巧：使用超大的地址类型让函数具有通用性，综合器会自动优化掉未使用的高位。

---

## 2.10 关键术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| 突发传输 | Burst Transfer | 一次地址握手后传输多拍数据 |
| 突发长度 | Burst Length | Burst 中的拍数（= AxLEN + 1） |
| 传输尺寸 | Transfer Size | 每拍传输的字节数（= 2^AxSIZE） |
| 对齐地址 | Aligned Address | 起始地址按传输尺寸向下对齐的地址 |
| 回绕边界 | Wrap Boundary | WRAP burst 回绕的下界地址 |
| 容器大小 | Container Size | Number_Bytes × Burst_Length |
| 窄传输 | Narrow Transfer | AxSIZE 小于数据总线宽度的传输 |
| 反压 | Back-pressure | 接收方通过拉低 ready 控制发送速率 |
| 响应优先级 | Response Precedence | 多个响应合并时的优先级规则 |
| 4KB 边界 | 4KB Boundary | 地址空间中每 0x1000 字节的边界 |
| 字节通道 | Byte Lane | 数据总线中对应某个字节的位段 |
| 蕴含 | Implication (|=>) | SVA 时序断言操作符 |

---

## 2.11 课后练习

### 练习 1：握手判断（基础）

以下两种 ready 实现，哪种违反了 AXI 协议？为什么？

```systemverilog
// 实现 A:
assign awready = !fifo_full && awvalid;

// 实现 B:
always_ff @(posedge clk) begin
    awvalid <= has_data && awready;  // valid 依赖 ready
end
```

### 练习 2：INCR 地址序列（基础）

计算以下 INCR burst 每拍的地址：

- Start_Address = 0x0003
- AxSIZE = 1 (2 字节/拍)
- AxLEN = 3 (4 拍)

提示：第 0 拍使用原始地址，后续拍从对齐地址递增。

### 练习 3：WRAP 地址序列（中级）

计算以下 WRAP burst 每拍的地址：

- Start_Address = 0x0034
- AxSIZE = 2 (4 字节/拍)
- AxLEN = 3 (4 拍)

步骤：
1. 计算 Wrap_Boundary
2. 计算 Container_Size
3. 依次计算每拍地址，判断是否回绕

### 练习 4：4KB 边界检查（中级）

判断以下 INCR burst 是否跨越 4KB 边界。如果跨越，给出拆分方案：

1. Start = 0x0F00, SIZE = 3 (8B), LEN = 31 (32 拍)
2. Start = 0x0FFC, SIZE = 2 (4B), LEN = 0 (1 拍)
3. Start = 0x0FF0, SIZE = 2 (4B), LEN = 15 (16 拍)

### 练习 5：响应合并（高级）

一个 8 拍 burst 被 `axi_burst_splitter` 拆分为 8 个单拍事务。各子事务的响应如下：

```
子事务 0: OKAY
子事务 1: OKAY
子事务 2: EXOKAY
子事务 3: OKAY
子事务 4: SLVERR
子事务 5: OKAY
子事务 6: DECERR
子事务 7: OKAY
```

用 `resp_precedence` 函数逐步合并，最终返回什么响应？

### 练习 6：源码验证（实践）

手动执行 `beat_addr()` 函数，验证以下 WRAP burst 的地址序列：

```
参数: addr = 0x0038, size = 3'd2, len = 8'd7, burst = BURST_WRAP
```

计算 beat 0 到 beat 7 的地址，并验证回绕点。

提示：
1. Number_Bytes = `num_bytes(2)` = 4
2. Wrap_Boundary = `wrap_boundary(0x0038, 2, 7)` = ?
3. Container_Size = 4 × 8 = 32

---

## 2.12 下一课预告

**第 3 课：AXI Package — 类型、常量与辅助函数**

在第 3 课中，我们将完整精读 `src/axi_pkg.sv`，涵盖：

- Cache 属性位域（Bufferable / Modifiable / Allocate）
- 内存类型枚举 `mem_type_t` 和 `get_arcache()` / `get_awcache()` 转换
- ATOP（原子操作）的完整编码方案
- 通道宽度计算函数 `aw_width()`, `w_width()`, `ar_width()` 等
- Crossbar 配置结构体 `xbar_cfg_t`
- 延迟模式 `xbar_latency_e`

---

## 参考资料

1. **ARM AMBA AXI and ACE Protocol Specification, Issue F.b** — A3.4 节（Burst 传输）、A3.2 节（握手）
2. `src/axi_pkg.sv` — 第 68-319 行（Burst 常量、响应常量、地址计算函数、响应优先级）
3. `src/axi_intf.sv` — 第 204-261 行（AXI_BUS_DV 断言）

---

> **版权声明**：本教材基于 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库编写，该仓库遵循 Solderpad Hardware License v0.51。
