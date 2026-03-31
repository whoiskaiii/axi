# AXI 总线协议学习项目

## 项目背景

这是 [pulp-platform/axi](https://github.com/pulp-platform/axi) 仓库（v0.39.9），由 ETH Zurich PULP 团队维护的生产级 AXI 总线 IP 库，包含 64 个 SystemVerilog 模块。我正在基于这个仓库系统学习 AXI 总线协议，并制作一套 30 课的深度教程。

## 教程文件结构

所有教程文件保存在 `tutorial/` 目录下：

```
tutorial/
├── AXI_Learning_Guide.md           # 学习路线图和总体建议
├── AXI_30_Lessons_Syllabus.md       # 30 课课程大纲（8 个单元）
└── lessons/                         # 每节课的详细教材
    ├── Lesson_01_AXI_Protocol_Overview.md   # 第 1 课（已完成）
    ├── Lesson_01_Answers.md                 # 第 1 课练习答案
    ├── Lesson_02_Handshake_Burst_Response.md # 第 2 课（已完成）
    ├── Lesson_02_Answers.md                 # 第 2 课练习答案
    ├── Lesson_03_AXI_Package.md             # 第 3 课（已完成）
    ├── Lesson_03_Answers.md                 # 第 3 课练习答案
    ├── Lesson_04_Typedef_Macros.md          # 第 4 课（已完成）
    └── Lesson_04_Answers.md                 # 第 4 课练习答案
```

## 课程进度

- [x] 第 1 课：AXI 协议概述与五通道架构
- [x] 第 2 课：握手机制、Burst 类型与响应编码
- [x] 第 3 课：AXI Package — 类型、常量与辅助函数
- [x] 第 4 课：Typedef 宏 — 参数化类型生成
- [ ] 第 5-30 课：待编写

## 教材编写规范

### 语言与风格
- 教材语言：**中文**
- 风格：**教科书风格** — 详尽的理论讲解 + 大量 ASCII 图表 + 源码引用 + 课后练习
- 图表优先使用 **ASCII art** 绘制时序图和架构图（兼容纯文本阅读），也可使用 Mermaid 语法

### 文件命名
- 课程文件：`tutorial/lessons/Lesson_XX_Title.md`（XX 为两位数字编号）
- 答案文件：`tutorial/lessons/Lesson_XX_Answers.md`（每课配套一个单独的答案文件）

### 每课必须包含的章节
1. **课程信息头**：课程系列、所属单元、前置知识、学习时长、对应源码
2. **学习目标**：完成本课后能够做什么（动词开头的列表）
3. **目录**：带锚点链接的章节列表
4. **正文内容**：按主题分节，每节包含概念讲解 + 源码引用（标注文件名和行号）
5. **源码映射**：将协议概念对应到仓库中的具体文件和行号
6. **关键术语表**：中英文对照
7. **课后练习**：5-6 题，难度递进（基础→中级→高级→实践）
8. **下一课预告**

### 源码引用格式
引用源码时标注文件路径和行号：
```
// src/axi_pkg.sv, 第 75-87 行
localparam BURST_FIXED = 2'b00;
```

### 课程大纲（8 个单元）
| 单元 | 课程 | 主题 |
|------|------|------|
| 一 | 1-5 | 协议基础与类型系统 |
| 二 | 6-8 | 缓冲与基础从机 |
| 三 | 9-11 | 解复用器（Demux） |
| 四 | 12-14 | 复用器（Mux） |
| 五 | 15-17 | 交叉开关互连 |
| 六 | 18-21 | 协议与数据转换 |
| 七 | 22-26 | ID 管理与 Burst 处理 |
| 八 | 27-30 | 系统级功能与验证 |

详细大纲见 `tutorial/AXI_30_Lessons_Syllabus.md`。

## 仓库关键文件速查

| 文件 | 作用 |
|------|------|
| `src/axi_pkg.sv` | 所有协议常量、类型、辅助函数 |
| `src/axi_intf.sv` | AXI4/AXI4-Lite 的 SystemVerilog interface 定义 |
| `include/axi/typedef.svh` | 生成通道 struct 的宏 |
| `include/axi/assign.svh` | 接口与 struct 之间的赋值宏 |
| `src/axi_test.sv` | 验证组件（rand_master, rand_slave, scoreboard） |
| `doc/` | 模块文档和架构图（PNG） |

## 注意事项

- 本仓库遵循 *AMBA AXI and ACE Protocol Specification, Issue F.b*
- 编码风格遵循 lowRISC SystemVerilog Coding Style Guide（见 `CONTRIBUTING.md`）
- 模块依赖分 6 个层级（Level 0-6），教材应按此层级递进
- struct 端口风格是核心设计约定，`_intf` 变体仅做连线适配