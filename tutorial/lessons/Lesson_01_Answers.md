# 第 1 课：课后练习参考答案

> 对应教材：`tutorial/lessons/Lesson_01_AXI_Protocol_Overview.md`

---

## 练习 1：通道方向（基础）

| 信号 | 驱动方 | 说明 |
|------|--------|------|
| awvalid | **Master** | AW 通道的 valid 由请求发起方（Master）驱动 |
| awready | **Slave** | AW 通道的 ready 由接收方（Slave）驱动 |
| wdata | **Master** | W 通道的数据由 Master 发送 |
| bvalid | **Slave** | B 通道（写响应）由 Slave 发起 |
| bready | **Master** | B 通道的 ready 由 Master 驱动（Master 是响应的接收方） |
| arvalid | **Master** | AR 通道的 valid 由 Master 驱动 |
| rdata | **Slave** | R 通道的数据由 Slave 返回 |
| rready | **Master** | R 通道的 ready 由 Master 驱动 |

**规律总结**：
- AW / W / AR 通道：Master 驱动 valid 和数据，Slave 驱动 ready
- B / R 通道：Slave 驱动 valid 和数据，Master 驱动 ready
- 简记：**发送方控制 valid，接收方控制 ready**

---

## 练习 2：握手分析（基础）

```
        cycle:  1    2    3    4    5    6
                ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
  clk       ────┘  └─┘  └─┘  └─┘  └─┘  └─┘  └──
                ┌─────────────────────┐
  valid     ────┘                     └──────────
            ─────────────┐     ┌─────────────────
  ready                  └─────┘
```

逐周期分析（在每个时钟上升沿采样）：

| 周期 | valid | ready | 握手？ |
|------|-------|-------|--------|
| 1 | 1 | 1 | **是** — valid && ready 同时为高 |
| 2 | 1 | 1 | **是** — valid && ready 同时为高 |
| 3 | 1 | 0 | 否 — ready 为低 |
| 4 | 1 | 0 | 否 — ready 为低 |
| 5 | 0 | 1 | 否 — valid 为低 |
| 6 | 0 | 1 | 否 — valid 为低 |

**答案：周期 1 和周期 2 发生了握手。**

> 注意：周期 3-4 中 valid 为高但 ready 为低，这属于"Pending"（未决）状态。

---

## 练习 3：Burst 参数计算（中级）

已知：`awaddr=0x0010, awlen=7, awsize=2, awburst=INCR`

**1. Burst 传输多少拍？**

```
拍数 = awlen + 1 = 7 + 1 = 8 拍
```

**2. 每拍传输多少字节？**

```
每拍字节数 = 2^awsize = 2^2 = 4 字节
```

**3. 总共传输多少字节？**

```
总字节数 = 拍数 × 每拍字节数 = 8 × 4 = 32 字节
```

**4. 每拍的地址序列：**

INCR burst 的地址计算规则（参考 `axi_pkg.sv` 中的 `beat_addr()` 函数）：
- beat 0 使用原始地址：`addr`
- beat N (N≥1)：`Aligned_Address + N × Number_Bytes`

```
Aligned_Address = (0x0010 >> 2) << 2 = 0x0010  （已经是 4 字节对齐）
Number_Bytes = 4

beat 0: 0x0010                              (原始地址)
beat 1: 0x0010 + 1 × 4 = 0x0014
beat 2: 0x0010 + 2 × 4 = 0x0018
beat 3: 0x0010 + 3 × 4 = 0x001C
beat 4: 0x0010 + 4 × 4 = 0x0020
beat 5: 0x0010 + 5 × 4 = 0x0024
beat 6: 0x0010 + 6 × 4 = 0x0028
beat 7: 0x0010 + 7 × 4 = 0x002C
```

地址范围：0x0010 ~ 0x002F，共 32 字节，未跨越 4KB 边界。

---

## 练习 4：写事务时序图（中级）

条件：
- 地址 0x4000，2 拍 burst（awlen=1, awburst=INCR, awsize=2，即 4 字节/拍）
- 第一拍数据 0xAAAA，wstrb=4'b1111
- 第二拍数据 0xBBBB，wstrb=4'b0011（仅低 2 字节有效）
- 响应 OKAY
- Slave 始终 ready

```
          cycle 1   cycle 2   cycle 3   cycle 4
            ┌───┐     ┌───┐     ┌───┐     ┌───┐
 clk        │   │     │   │     │   │     │   │
         ───┘   └─────┘   └─────┘   └─────┘   └───

 AW 通道:
            ┌─────────┐
 awvalid ───┘         └───────────────────────────
 awaddr  ═══╡ 0x4000  ╞═══════════════════════════
 awlen   ═══╡    1    ╞═══  (2 拍)
 awsize  ═══╡    2    ╞═══  (4 字节/拍)
 awburst ═══╡  INCR   ╞═══
 awready ═══════════════════  (始终为高)
            ▲
        AW 握手 (cycle 1)

 W 通道:
            ┌───────────────────┐
 wvalid  ───┘                   └─────────────────
 wdata   ═══╡  0xAAAA  ╞╡  0xBBBB  ╞═════════════
 wstrb   ═══╡ 4'b1111 ╞╡ 4'b0011 ╞═════════════
 wlast   ───────────────┘         └───────────────
 wready  ═══════════════════  (始终为高)
            ▲           ▲
       W beat 0     W beat 1 (wlast=1)
       (cycle 1)    (cycle 2)

 B 通道:
                                  ┌─────────┐
 bvalid  ─────────────────────────┘         └─────
 bresp   ═════════════════════════╡  OKAY   ╞═════
 bready  ═══════════════════  (始终为高)
                                  ▲
                              B 握手 (cycle 3)
```

**说明**：
- 因为 Slave 始终 ready，所以 AW 和 W 可以在同一周期握手
- W 的两拍数据在 cycle 1 和 cycle 2 连续传输
- B 响应在 wlast 握手完成后的下一周期返回

---

## 练习 5：协议规则判断（高级）

**1. Master 拉高 awvalid，但在 awready 为低的时候将 awvalid 拉低了**

**违反协议。** 根据 AXI 规范 A3.2.1 节：一旦 valid 拉高，在握手完成（valid && ready）之前，**不允许撤回 valid**，且数据信号必须保持稳定。

**2. Master 在 AW 握手完成之前就开始发送 W 数据**

**不违反协议。** AXI 规范允许 W 数据在 AW 握手之前或同时发送。AW 和 W 是独立的通道，没有严格的先后顺序要求。

**3. Slave 在 wlast=1 的 W 握手完成之前就发出了 B 响应**

**违反协议。** Slave 必须在接收完整个 burst 的所有写数据（即 wlast=1 的那一拍完成握手）之后才能发出 B 响应。提前发出 B 意味着 Slave 在数据未完全写入前就确认了事务完成。

**4. Slave 返回的 bid 与 awid 不匹配**

**违反协议。** B 响应的 bid 必须与其对应写事务的 awid 一致。这是事务追踪和乱序完成机制的基础。

**5. Master 的 awvalid 的拉高依赖于 awready 已经为高**

**违反协议。** 根据 AXI 规范的依赖规则：**valid 不能等待 ready**。如果 valid 依赖 ready，而 Slave 的 ready 也依赖 valid（这是允许的），就会产生死锁。规范明确要求 Source 必须独立地决定何时拉高 valid。

---

## 练习 6：源码探索（实践）

**1. `CACHE_BUFFERABLE` 的值是什么？它在 cache 信号的哪一位？**

```systemverilog
// src/axi_pkg.sv, 第 104 行
localparam CACHE_BUFFERABLE = 4'b0001;
```

- 值为 `4'b0001`
- 位于 cache 信号的**第 0 位**（最低位）
- 含义：当此位为 1 时，互连或任何组件可以延迟事务到达最终目的地

cache 信号的 4 个位分别是：
```
bit 3: Write-Allocate  (CACHE_WR_ALLOC  = 4'b1000)
bit 2: Read-Allocate   (CACHE_RD_ALLOC  = 4'b0100)
bit 1: Modifiable      (CACHE_MODIFIABLE = 4'b0010)
bit 0: Bufferable      (CACHE_BUFFERABLE = 4'b0001)
```

**2. `atop_t` 类型的位宽是多少？为什么 AR 通道没有对应的 atop 字段？**

```systemverilog
// src/axi_pkg.sv, 第 64 行
typedef logic [5:0] atop_t;  // 6 位宽
```

- 位宽为 **6 位**
- AR 通道没有 atop 字段的原因：**原子操作本质上是"读-修改-写"操作，必须通过写通道发起**。原子操作需要向目标地址发送数据（操作数），这只能通过 AW+W 通道完成。读响应（如果有）通过 R 通道返回。因此 atop 只存在于 AW 通道中。

对比 `typedef.svh` 中的定义可以验证：
- AW struct（第 34-48 行）：包含 `axi_pkg::atop_t atop` 字段
- AR struct（第 62-75 行）：**没有** atop 字段

**3. 找到 `num_bytes()` 函数，解释它的实现逻辑。**

```systemverilog
// src/axi_pkg.sv, 第 116-118 行
function automatic shortint unsigned num_bytes(size_t size);
    return shortint'(1 << size);
endfunction
```

实现逻辑：
- 输入：`size`（3 位，即 `size_t`，取值 0-7）
- 输出：每拍传输的最大字节数
- 计算：`1 << size` 即 2 的 size 次方

| size | 1 << size | 含义 |
|------|-----------|------|
| 0 | 1 | 1 字节/拍 |
| 1 | 2 | 2 字节/拍 |
| 2 | 4 | 4 字节/拍 |
| 3 | 8 | 8 字节/拍 |
| 4 | 16 | 16 字节/拍 |
| 5 | 32 | 32 字节/拍 |
| 6 | 64 | 64 字节/拍 |
| 7 | 128 | 128 字节/拍 |

这对应 AXI 规范 Table A3-2 中的定义。实际使用时，size 不能超过数据总线宽度对应的值（例如 64 位总线最大 size=3）。
