# 第 2 课：课后练习参考答案

> 对应教材：`tutorial/lessons/Lesson_02_Handshake_Burst_Response.md`

---

## 练习 1：握手判断（基础）

```systemverilog
// 实现 A:
assign awready = !fifo_full && awvalid;

// 实现 B:
always_ff @(posedge clk) begin
    awvalid <= has_data && awready;  // valid 依赖 ready
end
```

**答案：实现 B 违反了 AXI 协议。**

分析：

- **实现 A 合法**：`awready`（Destination 侧）依赖 `awvalid`（Source 侧）。AXI 规范允许 ready 依赖 valid（Destination 可以在看到 valid 后才拉高 ready）。`!fifo_full` 是内部状态条件，也是合理的。

- **实现 B 违反协议**：`awvalid`（Source 侧）依赖 `awready`（Destination 侧）。虽然这里使用了寄存器（`always_ff`），看起来 valid 是在下一周期才依赖 ready，但根本问题在于**设计意图**：如果 `has_data` 为高但 `awready` 为低，Master 就不会拉高 `awvalid`。这违反了 "valid 不能等待 ready" 的规则。正确的做法是：当 `has_data` 为高时，无论 `awready` 状态如何，都应拉高 `awvalid`。

> **注意**：严格来说，实现 B 中由于使用了寄存器，valid 实际上是依赖上一周期的 ready 值，在某些特定情况下可能不会产生组合环路。但这种编码风格暗示了 valid 对 ready 的依赖关系，在实际设计中应避免。正确的写法应为 `awvalid <= has_data;`。

---

## 练习 2：INCR 地址序列（基础）

已知：Start_Address = 0x0003, AxSIZE = 1 (2 字节/拍), AxLEN = 3 (4 拍)

**计算过程：**

```
Number_Bytes = 2^AxSIZE = 2^1 = 2 字节
Aligned_Address = (0x0003 >> 1) << 1 = 0x0001 << 1 = 0x0002
```

根据 `beat_addr()` 函数的逻辑：
- beat 0：使用原始地址 `addr`
- beat N (N≥1)：`Aligned_Address + N × Number_Bytes`

```
beat 0: 0x0003                          (原始地址，非对齐)
beat 1: 0x0002 + 1 × 2 = 0x0004
beat 2: 0x0002 + 2 × 2 = 0x0006
beat 3: 0x0002 + 3 × 2 = 0x0008
```

**答案：地址序列为 0x0003, 0x0004, 0x0006, 0x0008**

> **注意**：beat 0 到 beat 1 的地址增量为 1（不是 2），因为 beat 0 使用的是非对齐的原始地址，而从 beat 1 开始使用对齐地址递增。这就是 AXI 中"非对齐传输"（Unaligned Transfer）的行为。

---

## 练习 3：WRAP 地址序列（中级）

已知：Start_Address = 0x0034, AxSIZE = 2 (4 字节/拍), AxLEN = 3 (4 拍)

**步骤 1：计算基本参数**

```
Number_Bytes = 2^AxSIZE = 2^2 = 4 字节
Burst_Length = AxLEN + 1 = 4 拍
Container_Size = Number_Bytes × Burst_Length = 4 × 4 = 16 字节
```

**步骤 2：计算 Wrap_Boundary**

使用 `wrap_boundary()` 函数的逻辑（len=3 对应 case `4'b11`）：
```
Wrap_Boundary = (addr >> (size + 2)) << (size + 2)
              = (0x0034 >> 4) << 4
              = 0x0003 << 4
              = 0x0030
```

**步骤 3：逐拍计算地址**

```
Aligned_Address = (0x0034 >> 2) << 2 = 0x000D << 2 = 0x0034  (已对齐)

wrap 触发条件：ret_addr >= Wrap_Boundary + Container_Size
             = 0x0030 + 16 = 0x0040

beat 0: 0x0034                              (原始地址)
beat 1: 0x0034 + 1 × 4 = 0x0038
beat 2: 0x0034 + 2 × 4 = 0x003C
beat 3: 0x0034 + 3 × 4 = 0x0040 ≥ 0x0040   → 回绕！
        0x0040 - 16 = 0x0030
```

**答案：地址序列为 0x0034, 0x0038, 0x003C, 0x0030**

```
内存地址空间：
0x0030: [beat 3] ← 回绕到此
0x0034: [beat 0] ← 起始
0x0038: [beat 1]
0x003C: [beat 2]
0x0040: --------- (wrap boundary + container size，不会访问到)
```

---

## 练习 4：4KB 边界检查（中级）

4KB 边界地址：0x0000, 0x1000, 0x2000, ...（每 0x1000 一个边界）

**1. Start = 0x0F00, SIZE = 3 (8B), LEN = 31 (32 拍)**

```
总字节数 = 8 × 32 = 256 字节
结束地址 = 0x0F00 + 256 - 1 = 0x0FFF   (注：Aligned_Address = 0x0F00)
```

最后一个字节地址为 0x0FFF，下一个边界为 0x1000。**恰好未跨越 4KB 边界。**

**2. Start = 0x0FFC, SIZE = 2 (4B), LEN = 0 (1 拍)**

```
总字节数 = 4 × 1 = 4 字节
访问范围 = 0x0FFC ~ 0x0FFF
```

最后一个字节地址为 0x0FFF。**未跨越 4KB 边界。**

**3. Start = 0x0FF0, SIZE = 2 (4B), LEN = 15 (16 拍)**

```
总字节数 = 4 × 16 = 64 字节
结束地址 = 0x0FF0 + 64 - 1 = 0x102F
```

结束地址 0x102F 超过了 0x1000 边界。**跨越了 4KB 边界，违反 AXI 协议！**

**拆分方案**：将原始 burst 拆为两个不跨界的 burst：
```
Burst A: Start = 0x0FF0, SIZE = 2, LEN = 3  (4 拍, 到 0x0FFF)
Burst B: Start = 0x1000, SIZE = 2, LEN = 11 (12 拍, 到 0x102F)
```

验证：4 + 12 = 16 拍，总数据量不变。两个 burst 都不跨越 4KB 边界。

---

## 练习 5：响应合并（高级）

使用 `resp_precedence` 函数逐步合并。优先级：DECERR > SLVERR > OKAY > EXOKAY。

```
子事务响应: OKAY, OKAY, EXOKAY, OKAY, SLVERR, OKAY, DECERR, OKAY
```

逐步合并（从左到右依次应用 `resp_precedence`）：

```
步骤 1: resp_precedence(OKAY, OKAY)     = OKAY      ← OKAY 不优先于 OKAY，返回 resp_b
步骤 2: resp_precedence(OKAY, EXOKAY)   = OKAY      ← OKAY 优先于 EXOKAY
步骤 3: resp_precedence(OKAY, OKAY)     = OKAY
步骤 4: resp_precedence(OKAY, SLVERR)   = SLVERR    ← SLVERR 优先于 OKAY
步骤 5: resp_precedence(SLVERR, OKAY)   = SLVERR    ← SLVERR 维持
步骤 6: resp_precedence(SLVERR, DECERR) = DECERR    ← DECERR 优先于 SLVERR
步骤 7: resp_precedence(DECERR, OKAY)   = DECERR    ← DECERR 维持（最高优先级）
```

**答案：最终合并响应为 `DECERR` (2'b11)。**

> **原理**：`resp_precedence` 取两者中优先级更高的响应。只要子事务中出现过 DECERR，最终结果一定是 DECERR，因为它是最高优先级。实际上只要看到第 6 个子事务的 DECERR，结果就已经确定了。

---

## 练习 6：源码验证（实践）

已知：addr = 0x0038, size = 3'd2, len = 8'd7, burst = BURST_WRAP

**步骤 1：计算基本参数**

```
Number_Bytes = num_bytes(2) = 1 << 2 = 4
Burst_Length = len + 1 = 8
Container_Size = 4 × 8 = 32 字节
```

**步骤 2：计算 Wrap_Boundary**

len = 7 = 4'b0111，对应 `wrap_boundary` 中的 case `len_t'(4'b111)`：
```
wrap_boundary = (addr >> (size + 3)) << (size + 3)
              = (0x0038 >> 5) << 5
              = 0x0001 << 5
              = 0x0020
```

**步骤 3：计算 Aligned_Address**

```
Aligned_Address = (0x0038 >> 2) << 2 = 0x000E << 2 = 0x0038  (已对齐)
```

**步骤 4：逐拍计算地址**

回绕触发条件：`ret_addr >= Wrap_Boundary + Container_Size = 0x0020 + 32 = 0x0040`

```
beat 0: addr = 0x0038                              (原始地址)
beat 1: 0x0038 + 1×4 = 0x003C                      (< 0x0040, 不回绕)
beat 2: 0x0038 + 2×4 = 0x0040 ≥ 0x0040 → 回绕！
        0x0040 - 32  = 0x0020
beat 3: 0x0038 + 3×4 = 0x0044 ≥ 0x0040 → 回绕！
        0x0044 - 32  = 0x0024
beat 4: 0x0038 + 4×4 = 0x0048 ≥ 0x0040 → 回绕！
        0x0048 - 32  = 0x0028
beat 5: 0x0038 + 5×4 = 0x004C ≥ 0x0040 → 回绕！
        0x004C - 32  = 0x002C
beat 6: 0x0038 + 6×4 = 0x0050 ≥ 0x0040 → 回绕！
        0x0050 - 32  = 0x0030
beat 7: 0x0038 + 7×4 = 0x0054 ≥ 0x0040 → 回绕！
        0x0054 - 32  = 0x0034
```

**答案：地址序列为**

| beat | 地址 |
|------|------|
| 0 | 0x0038 |
| 1 | 0x003C |
| 2 | 0x0020 ← 回绕点 |
| 3 | 0x0024 |
| 4 | 0x0028 |
| 5 | 0x002C |
| 6 | 0x0030 |
| 7 | 0x0034 |

```
内存地址空间（Container: 0x0020 ~ 0x003F, 32 字节）：
0x0020: [beat 2] ← 回绕后从这里继续
0x0024: [beat 3]
0x0028: [beat 4]
0x002C: [beat 5]
0x0030: [beat 6]
0x0034: [beat 7]
0x0038: [beat 0] ← 起始地址
0x003C: [beat 1]
```

可以看到，8 拍 WRAP burst 恰好覆盖了整个 32 字节的容器（0x0020 ~ 0x003F），每个地址恰好被访问一次。这正是 WRAP burst 用于缓存行填充的典型模式。
