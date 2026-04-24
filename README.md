# 异步双时钟 FIFO 复现项目

本仓库用于复现并分析开源 Verilog 项目 [`dpretet/async_fifo`](https://github.com/dpretet/async_fifo)。它面向 ICS6204/MCE5913 Final Project：选择一个开源 RTL 工程，完成技术分析报告和 5-8 分钟课堂展示。

相关文档：

- 详细复现与 Git/GitHub 协作指南：[guide.md](guide.md)
- 英文 README：[README.en.md](README.en.md)
- 中文 README 备份：[README.zh-CN.md](README.zh-CN.md)
- 上游原始 README：[README.upstream.md](README.upstream.md)

## 项目定位

Async FIFO 用于在两个不同时钟域之间安全传输数据。写端使用 `wclk`，读端使用 `rclk`，内部通过双口存储器、二进制/Gray 码指针、两级同步器和满/空判断逻辑完成跨时钟域通信。

选择该项目的原因：

- 规模适中，比 UART/SPI 更能体现 CDC 和微架构分析深度。
- 顶层和子模块边界清晰，适合画模块图并讲解数据流。
- 包含仿真、lint、综合脚本，便于后续扩展验证和 STA。
- 与课程 PDF 中推荐的 Async FIFO 方向一致。

## 当前交付物

- `guide.md`：详细复现步骤、Git 安装与协作流程、课程任务拆解、模块分析、报告写作建议。
- `README.md`：中文主 README。
- `README.zh-CN.md`：中文 README 备份。
- `README.en.md`：英文项目说明。
- `README.upstream.md`：上游原始 README。
- `.gitattributes`：固定常见源码和文档文件为 LF，减少 Windows/WSL 换行符导致的伪修改。

## 仓库结构

```text
.
├── rtl/                         # RTL 源码
│   ├── async_fifo.v             # 基础异步 FIFO 顶层
│   ├── async_bidir_fifo.v       # 双向 FIFO 顶层，内部 RAM
│   ├── async_bidir_ramif_fifo.v # 双向 FIFO 顶层，外部 RAM 接口
│   ├── fifomem.v                # 基础双时钟 FIFO 存储器
│   ├── fifomem_dp.v             # 双端口存储器
│   ├── rptr_empty.v             # 读指针和 empty/almost-empty 逻辑
│   ├── wptr_full.v              # 写指针和 full/almost-full 逻辑
│   ├── sync_r2w.v               # 读指针同步到写时钟域
│   ├── sync_w2r.v               # 写指针同步到读时钟域
│   └── sync_ptr.v               # 通用指针同步器
├── sim/                         # SVUT/Icarus Verilog 测试平台
├── syn/                         # Yosys 综合脚本和 cell library
├── doc/                         # 原项目规格和测试计划
├── flow.sh                      # lint/sim/syn 统一入口
└── guide.md                     # 本复现项目详细指导文档
```

## 核心微架构

基础顶层 `rtl/async_fifo.v` 由五类模块组成：

- `fifomem`：保存 FIFO 数据，写端写入，读端读出。
- `wptr_full`：维护写端二进制指针和 Gray 码指针，生成 `wfull` 与 `awfull`。
- `rptr_empty`：维护读端二进制指针和 Gray 码指针，生成 `rempty` 与 `arempty`。
- `sync_r2w`：把读指针 Gray 码同步到写时钟域，供 full 判断使用。
- `sync_w2r`：把写指针 Gray 码同步到读时钟域，供 empty 判断使用。

关键思想是：地址计算使用二进制指针，跨时钟域传输使用 Gray 码指针，因为 Gray 码相邻值只改变 1 bit，可降低多 bit 指针跨域采样时产生错误组合的风险。

## 主要接口

`async_fifo` 参数：

| 参数 | 作用 |
| --- | --- |
| `DSIZE` | 数据位宽，默认 8 bit |
| `ASIZE` | 地址位宽，FIFO 深度为 `2**ASIZE` |
| `FALLTHROUGH` | `"TRUE"` 时首字直通，降低读侧可见延迟 |

`async_fifo` 端口：

| 端口 | 方向 | 说明 |
| --- | --- | --- |
| `wclk` / `wrst_n` | input | 写时钟和写侧低有效复位 |
| `winc` / `wdata` | input | 写使能和写数据 |
| `wfull` / `awfull` | output | 写侧满和将满标志 |
| `rclk` / `rrst_n` | input | 读时钟和读侧低有效复位 |
| `rinc` / `rdata` | input/output | 读使能和读数据 |
| `rempty` / `arempty` | output | 读侧空和将空标志 |

使用约束：

- 写端应在 `wfull == 0` 时拉高 `winc`，否则写入会被屏蔽或丢失。
- 读端应在 `rempty == 0` 时拉高 `rinc`，否则读出的数据不应被视为有效。
- 两侧复位最好同时进行，原项目规格也强调使用前两侧都要复位。

## 可选验证命令

当前环境是 WSL 且本次任务明确不要求测试，所以本次没有执行仿真或 GUI 波形检查。后续如果环境具备工具链，可以运行：

```bash
./flow.sh lint
./flow.sh sim
./flow.sh syn
```

依赖工具：

- lint：Verilator
- 仿真：Icarus Verilog + SVUT
- 综合：Yosys

## 课程报告建议

Final Project 报告建议围绕以下结构展开：

1. Project overview：说明异步 FIFO 的用途、场景和关键规格。
2. Build & simulation：可选；若后续能跑工具链，记录命令和 pass 标准。
3. Module diagram：画出 `async_fifo` 顶层、指针逻辑、同步器和存储器。
4. Interface description：描述写端、读端、复位、满/空标志和握手规则。
5. Micro-architecture：重点讲 Gray code、CDC synchronizer、full/empty 判断。
6. Code walk-through：解释关键文件职责，不需要逐行翻译。
7. Verification：可选；说明现有 SVUT testbench 覆盖了单次写读、连续写读、满/空标志和并发读写。
8. Findings：写 3-5 条具体观察。
9. References：列出 GitHub 仓库、Clifford Cummings FIFO 论文、课程 PDF 等。

更详细的执行步骤和 Git/GitHub 协作流程见 [`guide.md`](guide.md)。

## 版权和致谢

本项目基于 Damien Pretet 的 [`dpretet/async_fifo`](https://github.com/dpretet/async_fifo) 进行课程复现和文档整理。请保留原作者版权和许可证信息。仓库顶层 `LICENSE` 为 MIT License，部分 RTL 文件头部包含 Apache-2.0 许可证声明，使用前应按文件头和许可证要求保留声明。
