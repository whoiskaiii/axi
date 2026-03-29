# 第 3 课：课后练习参考答案

> 对应教材：`tutorial/lessons/Lesson_03_AXI_Package.md`

---

## 练习 1：Cache 属性解读（基础）

**1. `4'b0000`**

```
bit 3 (WR_ALLOC)  = 0: 不建议写分配
bit 2 (RD_ALLOC)  = 0: 不建议读分配
bit 1 (MODIFIABLE)= 0: 不可修改
bit 0 (BUFFERABLE)= 0: 不可缓冲
```

对应 `DEVICE_NONBUFFERABLE`。这是最严格的内存属性，通常用于 MMIO 寄存器（如 UART 数据寄存器）。事务不能被缓存、缓冲、重排序或合并，必须原样送达目标。

**2. `4'b0011`**

```
bit 3 (WR_ALLOC)  = 0: 不建议写分配
bit 2 (RD_ALLOC)  = 0: 不建议读分配
bit 1 (MODIFIABLE)= 1: 可修改
bit 0 (BUFFERABLE)= 1: 可缓冲
```

对应 `NORMAL_NONCACHEABLE_BUFFERABLE`。普通内存类型，不经过缓存但允许互连修改事务特征（如合并写入）和缓冲。典型用途：DMA 缓冲区、非缓存共享内存。

**3. `4'b1111`**

```
bit 3 (WR_ALLOC)  = 1: 建议写分配
bit 2 (RD_ALLOC)  = 1: 建议读分配
bit 1 (MODIFIABLE)= 1: 可修改
bit 0 (BUFFERABLE)= 1: 可缓冲
```

对应 `WBACK_RWALLOCATE`（写回 + 读写分配）。这是性能最好的缓存策略，通常用于主内存（DDR）。读写都建议在缓存中分配缓存行，写入只更新缓存，稍后回写内存。

---

## 练习 2：ATOP 编码解析（基础）

**1. `6'b000000`**

```
ATOP[5:4] = 2'b00 = ATOP_NONE
```

**非原子操作**。这是普通写事务的 awatop 值。

**2. `6'b010011`**

```
ATOP[5:4] = 2'b01 = ATOP_ATOMICSTORE（原子存储，无返回数据）
ATOP[3]   = 1'b0  = ATOP_LITTLE_END（小端）
ATOP[2:0] = 3'b011 = ATOP_SET（置位操作）
```

**原子置位操作（AtomicStore + SET）**：`mem = mem | data`。将发送的数据与内存中的值按位或，结果写回内存。无数据返回（不经过 R 通道）。

**3. `6'b100110`**

```
ATOP[5:4] = 2'b10 = ATOP_ATOMICLOAD（原子加载，返回旧值）
ATOP[3]   = 1'b1  = ATOP_BIG_END（大端）
ATOP[2:0] = 3'b110 = ATOP_UMAX（无符号最大值）
```

**原子无符号最大值操作（AtomicLoad + UMAX + Big-Endian）**：读取内存旧值通过 R 通道返回，然后 `mem = max(mem, data)`（无符号比较）。大端字节序。

**4. `6'b110000`**

```
ATOP[5:0] = 6'b110000 = ATOP_ATOMICSWAP
```

**原子交换（AtomicSwap）**：将发送的数据与内存中的值交换。旧值通过 R 通道返回，新值写入内存。

---

## 练习 3：通道宽度计算（中级）

参数：AddrWidth=48, DataWidth=128, IdWidth=8, UserWidth=2

**1. AW 通道总位宽**

```
aw_width = id + addr + Len + Size + Burst + Lock + Cache + Prot + Qos + Region + Atop + user
         = 8  + 48   + 8   + 3    + 2     + 1    + 4     + 3    + 4   + 4      + 6    + 2
         = 93 位
```

**2. W 通道总位宽**

```
w_width = data + strobe    + last + user
        = 128  + 128/8     + 1    + 2
        = 128  + 16        + 1    + 2
        = 147 位
```

**3. R 通道总位宽**

```
r_width = id + data + resp + last + user
        = 8  + 128  + 2    + 1    + 2
        = 141 位
```

**4. 完整 req_t struct 的总位宽**

```
AR 宽度 = id + addr + Len + Size + Burst + Lock + Cache + Prot + Qos + Region + user
        = 8  + 48   + 8   + 3    + 2     + 1    + 4     + 3    + 4   + 4      + 2
        = 87 位  (比 AW 少 6 位 atop)

req_width = aw_width + 1(aw_valid) + w_width + 1(w_valid) + ar_width + 1(ar_valid)
          + 1(r_ready) + 1(b_ready)
          = 93 + 1 + 147 + 1 + 87 + 1 + 1 + 1
          = 332 位
```

---

## 练习 4：Crossbar 配置设计（中级）

```systemverilog
localparam axi_pkg::xbar_cfg_t SocXbarCfg = '{
    NoSlvPorts:         3,           // CPU + DMA + 调试接口 = 3 个 Master
    NoMstPorts:         4,           // Boot ROM + SRAM + 外设桥 + DDR = 4 个 Slave
    MaxMstTrans:        8,           // 每个 Slave 端口 8 个在途事务（平衡性能和面积）
    MaxSlvTrans:        8,           // 每个 Master 端口 8 个在途事务
    FallThrough:        1'b0,        // 不使用直通（降低组合路径长度）
    LatencyMode:        axi_pkg::CUT_ALL_AX,  // 切割所有地址通道（500MHz 时序需求）
    PipelineStages:     0,           // 无额外流水线（CUT_ALL_AX 已足够）
    AxiIdWidthSlvPorts: 4,           // 4 位 ID（取所有 Master 中最大的 ID 宽度）
    AxiIdUsedSlvPorts:  4,           // 全部 4 位用于路由
    UniqueIds:          1'b0,        // 不保证唯一（CPU 和 DMA 可能使用相同 ID）
    AxiAddrWidth:       32,          // 32 位地址
    AxiDataWidth:       64,          // 64 位数据
    NoAddrRules:        4            // 4 条地址规则（每个 Slave 一条）
};
```

**设计考虑：**

- **AxiIdWidthSlvPorts = 4**：虽然调试接口只用 2 位 ID，但 Crossbar 的从机端口 ID 宽度必须统一。调试接口的高 2 位 ID 将始终为 0。
- **LatencyMode = CUT_ALL_AX**：500MHz 目标频率较高，需要切割地址通道的组合路径。如果时序仍不满足，可以升级到 `CUT_ALL_PORTS`。
- **FallThrough = 0**：直通模式会增加组合路径长度，在高频设计中应避免。

对应的地址映射规则：

```systemverilog
localparam axi_pkg::xbar_rule_32_t [3:0] AddrMap = '{
    '{idx: 0, start_addr: 32'h0000_0000, end_addr: 32'h0001_0000},  // Boot ROM: 64KB
    '{idx: 1, start_addr: 32'h0001_0000, end_addr: 32'h0005_0000},  // SRAM: 256KB
    '{idx: 2, start_addr: 32'h4000_0000, end_addr: 32'h4000_1000},  // 外设桥: 4KB
    '{idx: 3, start_addr: 32'h8000_0000, end_addr: 32'hC000_0000}   // DDR: 1GB
};
```

---

## 练习 5：延迟模式分析（高级）

`CUT_ALL_AX` 模式切割了 4 个通道：DemuxAw, DemuxAr, MuxAw, MuxAr。

对于读事务（AR → R），信号经过的路径：

```
Master                                                             Slave
  │                                                                  │
  ├── AR ──► [Demux AR spill reg] ──► Demux 路由 ──► [Mux AR spill reg] ──► Mux 仲裁 ──► AR ──►│
  │                (+1 cycle)                            (+1 cycle)                             │
  │                                                                                            │
  │◄── R ◄────────────── Demux 路由 ◄─────────────── Mux 响应路由 ◄───────────── R ◄────────────│
  │          (无 spill register, 组合逻辑)    (无 spill register)              (0 cycle)        │
```

最小延迟分析：
1. Master 发出 AR，经过 Demux AR spill register：**+1 周期**
2. 经过 Demux 路由逻辑（组合逻辑）：**+0 周期**（同一周期）
3. 经过 Mux AR spill register：**+1 周期**
4. Mux 仲裁（假设无竞争，组合逻辑）：**+0 周期**
5. AR 到达 Slave，Slave 零延迟处理：**+0 周期**
6. R 数据从 Slave 返回，经过 Mux R 和 Demux R（`CUT_ALL_AX` 模式下无 R 通道 spill register）：**+0 周期**

**答案：最小延迟 = 2 个时钟周期**（2 级 spill register 各引入 1 周期延迟）

对比其他模式：
- `NO_LATENCY`：最小延迟 = 0 周期（纯组合路径，但时序可能很差）
- `CUT_ALL_PORTS`：最小延迟 = 4 周期（AR 路径 2 级 + R 路径 2 级）

---

## 练习 6：源码验证（实践）

**1. `get_arcache(WBACK_RALLOCATE)` vs `get_awcache(WBACK_RALLOCATE)`**

```systemverilog
get_arcache(WBACK_RALLOCATE) = 4'b1111
//  bit3=1 (WR_ALLOC), bit2=1 (RD_ALLOC), bit1=1 (MODIFIABLE), bit0=1 (BUFFERABLE)

get_awcache(WBACK_RALLOCATE) = 4'b0111
//  bit3=0 (WR_ALLOC), bit2=1 (RD_ALLOC), bit1=1 (MODIFIABLE), bit0=1 (BUFFERABLE)
```

**差异在 bit 3（Write-Allocate）**：

- 读方向（ARCACHE）= `4'b1111`：设置了 WR_ALLOC。这看起来矛盾（读操作为什么设置写分配？），实际上 AXI 规范中 ARCACHE 的 bit 3 在 Cacheable 类型下表示"Other Allocate"，对于读来说"Other"就是写方向。`WBACK_RALLOCATE` 表示推荐读分配，所以 ARCACHE 中 RD_ALLOC=1 且标记其他分配属性。
- 写方向（AWCACHE）= `4'b0111`：不设置 WR_ALLOC（bit 3=0），因为 `WBACK_RALLOCATE` 的语义是"只推荐读分配"，写操作时不建议分配缓存行。

简而言之：`RALLOCATE` 意味着只推荐读分配，所以写方向不设置写分配位。

**2. 自定义延迟模式：Demux AW + Mux R**

```systemverilog
localparam bit [9:0] CUSTOM_LATENCY = axi_pkg::DemuxAw | axi_pkg::MuxR;
// = (1 << 9) | (1 << 0) = 10'b10_000_00001
```

这种模式适合写密集型场景：
- 切割 Demux AW：优化写地址路径的时序
- 切割 Mux R：优化读数据返回路径的时序
- 其他通道保持组合逻辑直连以降低延迟
