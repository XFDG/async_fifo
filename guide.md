# Async FIFO 复现与协作开发指南

本文档记录本次复现 [`dpretet/async_fifo`](https://github.com/dpretet/async_fifo) 的完整思路、课程任务拆解、仓库结构、关键 RTL 分析、后续报告写作路线，以及上传到 GitHub 后的协作流程。

## 1. 课程任务结论

根据 `FinalProj.pdf` 和 `Final_Project_detailed_plan(1).pdf`，当前任务不是从零设计一个复杂芯片，而是选择一个开源 Verilog/RTL 项目，读懂它、复现它、分析它，并最终产出技术报告和课堂展示。

关键时间点：

- 技术报告 PDF 截止日期：2026-05-15。
- Presentation 日期：2026-04-30 和 2026-05-07。
- 展示时长：5-8 分钟。

评分和提交：

- Final Project 占课程总评 40%。
- Final Project Presentation 占课程总评 15%。
- STA report 是 bonus，不是必做项。
- 最终提交文件命名格式应为 `{student_id}_Name.pdf` 或 `{student_id}_Name.zip`，不符合命名规则会扣 5%。

报告必须覆盖的核心内容：

1. Project overview。
2. Build & simulation，可选但推荐。
3. Module diagram。
4. Interface description。
5. Micro-architecture。
6. Code walk-through。
7. Verification，可选。
8. Findings，要求 3-5 条具体观察。
9. References。

本仓库选择 Async FIFO，理由是它规模适中、模块边界清晰、CDC 主题有深度，非常适合在有限报告和 5-8 分钟 presentation 中讲清楚。

## 2. Git 安装与 GitHub 协作流程

这一章放在前面，是因为后续所有协作都依赖 Git。没有 Git，大家只能互相传压缩包，很容易覆盖彼此修改；使用 Git 后，每个人可以在自己的分支上改文档、画图或补分析，再通过 Pull Request 合并。

### 2.1 Git 是什么

Git 是版本控制工具，解决三个问题：

- 记录每一次修改，方便回看历史。
- 允许多人并行开发，降低互相覆盖文件的风险。
- 通过 GitHub 把本地仓库同步到云端，方便协作、review 和备份。

几个基本概念：

| 概念 | 含义 |
| --- | --- |
| repository / repo | 一个 Git 仓库，也就是一个被 Git 管理的项目目录 |
| commit | 一次明确的修改记录，建议每个 commit 只做一类事情 |
| branch | 分支，用来隔离不同任务，例如报告、图、验证记录 |
| remote | 远程仓库，例如 GitHub 上的 `XFDG/async_fifo` |
| push | 把本地 commit 上传到远程仓库 |
| pull | 从远程仓库拉取别人已经上传的修改 |
| Pull Request / PR | 在 GitHub 上请求把一个分支合并到主分支 |

### 2.2 Windows 安装 Git

如果同学使用 Windows，推荐安装 Git for Windows：

1. 打开 https://git-scm.com/download/win。
2. 下载 64-bit Git for Windows installer。
3. 双击安装。
4. 安装组件保持默认即可，确保包含 Git Bash。
5. Editor 可选择 VS Code；如果不确定，默认 Vim 也可以，但新手不太友好。
6. PATH 选项选择 `Git from the command line and also from 3rd-party software`。
7. SSH 选项选择 `Use bundled OpenSSH`。
8. Line ending 选项建议选择 `Checkout as-is, commit Unix-style line endings`；如果已经装完，也可以后面用命令配置。
9. 安装完成后，打开 Git Bash 或 PowerShell，检查版本：

```bash
git --version
```

如果能看到类似 `git version 2.xx.x`，说明安装成功。

### 2.3 WSL / Ubuntu 安装 Git

如果同学使用 WSL 或 Ubuntu：

```bash
sudo apt update
sudo apt install -y git
git --version
```

本项目在 WSL 下操作时，建议进入 Linux 路径或 `/mnt/c/...` 项目路径后再运行 Git 命令。Windows 和 WSL 共用同一个目录时，最常见问题是换行符变化，所以本仓库已经加入 `.gitattributes` 来固定常见源码和文档文件为 LF。

### 2.4 第一次使用 Git 的全局配置

每位同学第一次使用 Git 时，都应该配置自己的名字和邮箱。这个信息会出现在 commit 记录里。

```bash
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
```

检查配置：

```bash
git config --global --list
```

建议再设置换行符，减少 Windows/WSL 伪修改：

```bash
git config --global core.autocrlf false
git config --global core.safecrlf warn
```

如果使用 VS Code，可以设置默认编辑器：

```bash
git config --global core.editor "code --wait"
```

### 2.5 配置 GitHub SSH Key

推荐用 SSH 方式连接 GitHub，这样 push 时不需要每次输入账号密码。

生成 SSH key：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

一路回车即可。默认会生成：

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

查看公钥内容：

```bash
cat ~/.ssh/id_ed25519.pub
```

把输出的整行内容复制到 GitHub：

1. 打开 GitHub。
2. 点击头像。
3. 进入 `Settings`。
4. 左侧选择 `SSH and GPG keys`。
5. 点击 `New SSH key`。
6. Title 填电脑名，例如 `Laptop WSL`。
7. Key 粘贴 `id_ed25519.pub` 的内容。
8. 点击 `Add SSH key`。

测试 SSH：

```bash
ssh -T git@github.com
```

如果看到类似下面的信息，说明认证成功：

```text
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

### 2.6 获取本项目仓库

如果团队仓库已经创建，最推荐直接克隆团队仓库：

```bash
git clone git@github.com:XFDG/async_fifo.git
cd async_fifo
```

如果你先从上游原项目开始：

```bash
git clone https://github.com/dpretet/async_fifo.git
cd async_fifo
git remote rename origin upstream
git remote add origin git@github.com:XFDG/async_fifo.git
```

检查远程仓库：

```bash
git remote -v
```

推荐含义：

- `origin`：团队自己的仓库，通常是 `git@github.com:XFDG/async_fifo.git`。
- `upstream`：原作者仓库，通常是 `https://github.com/dpretet/async_fifo.git`。

如果当前仓库使用 `xfdg` 作为团队远程名，也可以继续使用：

```bash
git remote add xfdg git@github.com:XFDG/async_fifo.git
git push -u xfdg master
```

### 2.7 每天开始工作前先同步

开始改文件前先拉取远程更新：

```bash
git checkout master
git pull
```

如果团队远程叫 `xfdg`：

```bash
git checkout master
git pull xfdg master
```

这样可以减少你和同学改到同一份旧文件后产生冲突。

### 2.8 使用分支完成任务

不要所有人都直接改 `master`。每个任务建一个分支：

```bash
git checkout -b docs/report-outline
```

分支命名建议：

| 分支名 | 用途 |
| --- | --- |
| `docs/report-outline` | 报告大纲 |
| `docs/module-diagram` | 模块图和图示 |
| `analysis/micro-architecture` | 微架构分析 |
| `analysis/findings` | findings 整理 |
| `verification/sim-notes` | 仿真和测试记录 |
| `slides/presentation` | PPT 和讲稿 |

查看当前分支：

```bash
git branch
git status --short --branch
```

### 2.9 修改、暂存和提交

查看改了哪些文件：

```bash
git status --short
git diff
```

把需要提交的文件加入暂存区：

```bash
git add guide.md README.md
```

提交：

```bash
git commit -m "docs: update git collaboration guide"
```

提交信息建议格式：

```text
<type>: <short description>
```

常用 type：

| type | 场景 |
| --- | --- |
| `docs` | 文档、报告、README、guide |
| `rtl` | RTL 代码修改 |
| `sim` | 仿真或 testbench 修改 |
| `syn` | 综合脚本修改 |
| `chore` | 仓库配置、格式、杂项 |

好的 commit 示例：

```bash
git commit -m "docs: add async fifo module diagram notes"
git commit -m "docs: refine findings section"
git commit -m "sim: document existing unit tests"
```

不推荐的 commit：

```bash
git commit -m "update"
git commit -m "fix"
git commit -m "final final version"
```

### 2.10 推送到 GitHub

第一次推送当前分支：

```bash
git push -u origin docs/report-outline
```

如果团队远程叫 `xfdg`：

```bash
git push -u xfdg docs/report-outline
```

之后同一分支继续推送：

```bash
git push
```

如果推送 `master`：

```bash
git push origin master
```

### 2.11 创建 Pull Request

推荐所有协作者都通过 PR 合并：

1. 把分支 push 到 GitHub。
2. 打开仓库页面 `https://github.com/XFDG/async_fifo`。
3. GitHub 通常会提示 `Compare & pull request`。
4. Base 选择 `master`，Compare 选择你的任务分支。
5. 写清楚本次修改内容。
6. 至少请一名同学 review。
7. 解决评论后再 merge。

PR 描述模板：

```markdown
## Summary
- Added/updated ...

## Scope
- Report section:
- RTL files referenced:
- Diagrams affected:

## Checks
- [ ] Markdown reviewed
- [ ] Links checked
- [ ] No unrelated RTL changes
```

### 2.12 Issue 拆分任务

建议在 GitHub Issues 中拆任务：

- `report: project overview`
- `diagram: async_fifo top-level module graph`
- `analysis: Gray code pointer and full/empty logic`
- `analysis: code walk-through`
- `slides: 5-8 min presentation`
- `verification: optional sim/lint notes`

每个 issue 写清楚：

- 目标是什么。
- 输出文件是什么。
- 参考哪些 RTL 文件。
- 截止时间。

### 2.13 处理冲突

如果 `git pull` 或 PR 显示 conflict，不要慌，通常是两个人改了同一个文件相近位置。

基本处理流程：

```bash
git status
```

打开冲突文件，找到类似标记：

```text
  <<<<<<< HEAD
当前分支内容
  =======
另一个分支内容
  >>>>>>> branch-name
```

手动改成最终想保留的版本，然后：

```bash
git add <conflict-file>
git commit
git push
```

如果不确定应该保留谁的内容，先在群里问，不要盲目删除。

### 2.14 协作注意事项

- 不要提交仿真生成的 `.vcd`、`.vvp`、`.out` 等临时文件。
- 不要因为换行符变化提交全仓库大 diff。
- 如果只写报告，尽量只改 Markdown、图片和文档。
- 如果修改 RTL，必须说明为什么改，并补充验证记录。
- 不要随意运行 `git reset --hard`，它会丢掉本地未提交修改。
- 不要随意强推 `git push --force`，除非全组明确知道影响。
- 合并上游原作者仓库前，先看 diff，避免引入不理解的变化。

### 2.15 与上游原项目同步

如果以后想同步 `dpretet/async_fifo` 的更新：

```bash
git fetch upstream
git checkout master
git merge upstream/master
git push origin master
```

如果只想查看上游多了什么：

```bash
git fetch upstream
git log --oneline master..upstream/master
git diff master..upstream/master --stat
```

## 3. 从零复现步骤

如果其他同学从空目录开始，可以按以下步骤复现。

### 3.1 准备基础工具

最低要求：

- Git，安装和配置见第 2 章。
- 文本编辑器，如 VS Code。
- Bash/WSL/Ubuntu shell。

可选工具：

- Verilator：用于 lint。
- Icarus Verilog：用于仿真。
- SVUT：原项目使用的 SystemVerilog unit test 工具。
- Yosys：用于逻辑综合。
- GTKWave：用于查看波形；本 WSL 无 GUI 时可跳过。

Ubuntu/WSL 可参考：

```bash
sudo apt update
sudo apt install -y make iverilog verilator yosys
```

本次任务没有运行测试，因此这些工具不是当前文档交付的硬性前置条件。

### 3.2 克隆团队仓库或上游仓库

如果团队仓库已经可用，直接克隆团队仓库：

```bash
git clone git@github.com:XFDG/async_fifo.git
cd async_fifo
```

如果只是复现上游原始项目，也可以克隆原作者仓库：

```bash
git clone https://github.com/dpretet/async_fifo.git
cd async_fifo
```

如果已经有本地仓库，进入目录后先看远程和状态：

```bash
git remote -v
git status --short --branch
```

团队协作时，远程仓库配置和推送规则以第 2 章为准。

### 3.3 建议的阅读顺序

不要一开始逐行读所有文件。推荐顺序：

1. `README.md`：了解项目目标和已有说明。
2. `doc/specification.rst`：了解功能规格、复位要求和接口行为。
3. `rtl/async_fifo.v`：先抓顶层连接关系。
4. `rtl/wptr_full.v` 和 `rtl/rptr_empty.v`：理解指针、Gray code、满/空判断。
5. `rtl/sync_r2w.v` 和 `rtl/sync_w2r.v`：理解两级同步器。
6. `rtl/fifomem.v`：理解实际数据存储。
7. `sim/async_fifo_unit_test.sv`：了解已有测试覆盖了哪些场景。
8. `flow.sh`、`.github/workflows/ci.yaml`、`syn/fifo.ys`：了解工程化流程。

### 3.4 仓库目录说明

```text
async_fifo
├── .github/workflows/ci.yaml       # 上游 GitHub Actions：lint、simulation、synthesis
├── doc/
│   ├── specification.rst           # 功能规格
│   └── testplan.rst                # 测试计划
├── rtl/
│   ├── async_fifo.v                # 基础异步 FIFO 顶层
│   ├── async_bidir_fifo.v          # 双向 FIFO，内部 RAM
│   ├── async_bidir_ramif_fifo.v    # 双向 FIFO，外部 RAM 接口
│   ├── async_fifo.list             # 基础 FIFO 文件列表
│   ├── fifomem.v                   # 基础 FIFO 存储器
│   ├── fifomem_dp.v                # 双端口存储器
│   ├── rptr_empty.v                # 读指针和 empty 判断
│   ├── sync_ptr.v                  # 通用两级同步器
│   ├── sync_r2w.v                  # 读指针同步到写域
│   ├── sync_w2r.v                  # 写指针同步到读域
│   └── wptr_full.v                 # 写指针和 full 判断
├── sim/
│   ├── async_fifo_unit_test.sv     # SVUT 单元测试
│   ├── files.f                     # 仿真文件列表
│   ├── Makefile                    # 测试入口
│   └── wave.gtkw                   # GTKWave 配置
├── syn/
│   ├── fifo.ys                     # Yosys 综合脚本
│   └── syn_asic.sh                 # 综合执行脚本
├── flow.sh                         # lint/sim/syn 统一命令入口
├── README.md                       # 中文主 README
├── README.zh-CN.md                 # 中文 README 备份
├── README.en.md                    # 本复现项目英文说明
├── README.upstream.md              # 上游原始 README
└── guide.md                        # 本指南
```

## 4. 核心设计说明

### 4.1 Async FIFO 解决的问题

同步 FIFO 只有一个时钟，读写指针都在同一时钟域中更新。Async FIFO 的难点在于写端和读端属于不同 clock domain：

- 写端根据 `wclk` 写入数据。
- 读端根据 `rclk` 读出数据。
- 两端频率、相位都可能不同。
- 写端需要知道读指针位置，避免覆盖未读数据。
- 读端需要知道写指针位置，避免读取不存在的数据。

直接跨域传递多 bit 二进制指针是不安全的，因为多个 bit 可能同时变化，接收时钟域可能采样到过渡态。该项目使用 Gray code 指针跨域，因为相邻 Gray code 只有 1 bit 改变，可降低错误采样组合的风险。

### 4.2 顶层模块图

基础 FIFO 的结构可以用 Mermaid 表示：

```mermaid
flowchart LR
    subgraph W["Write clock domain: wclk"]
        WIF["winc / wdata"]
        WPF["wptr_full\nbinary waddr + Gray wptr\nwfull / awfull"]
        R2W["sync_r2w\nread Gray ptr -> write domain"]
    end

    subgraph MEM["FIFO storage"]
        RAM["fifomem\n2^ASIZE x DSIZE"]
    end

    subgraph R["Read clock domain: rclk"]
        RIF["rinc / rdata"]
        RPE["rptr_empty\nbinary raddr + Gray rptr\nrempty / arempty"]
        W2R["sync_w2r\nwrite Gray ptr -> read domain"]
    end

    WIF --> WPF
    WPF -- waddr --> RAM
    WIF -- wdata --> RAM
    RAM -- rdata --> RIF
    RIF --> RPE
    RPE -- raddr --> RAM
    RPE -- rptr --> R2W
    R2W -- wq2_rptr --> WPF
    WPF -- wptr --> W2R
    W2R -- rq2_wptr --> RPE
```

报告中可以用 draw.io、PowerPoint、Mermaid 或手绘图重画这个结构。若最终报告要导出 PDF，建议使用 draw.io 或 PPT 画图，更易控制版式。

### 4.3 `async_fifo.v`

`rtl/async_fifo.v` 是基础顶层，实例化：

- `sync_r2w`：读指针同步到写域。
- `sync_w2r`：写指针同步到读域。
- `wptr_full`：写指针和 full 判断。
- `fifomem`：FIFO 存储阵列。
- `rptr_empty`：读指针和 empty 判断。

它的参数：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `DSIZE` | `8` | 数据位宽 |
| `ASIZE` | `4` | 地址位宽，深度为 `2**ASIZE` |
| `FALLTHROUGH` | `"TRUE"` | 首字直通，降低读端首次读延迟 |

它的端口：

| 信号 | 方向 | 时钟域 | 说明 |
| --- | --- | --- | --- |
| `wclk` | input | write | 写时钟 |
| `wrst_n` | input | write | 写侧低有效复位 |
| `winc` | input | write | 写请求，高有效 |
| `wdata` | input | write | 写入数据 |
| `wfull` | output | write | FIFO 满标志 |
| `awfull` | output | write | FIFO 将满标志 |
| `rclk` | input | read | 读时钟 |
| `rrst_n` | input | read | 读侧低有效复位 |
| `rinc` | input | read | 读请求，高有效 |
| `rdata` | output | read | 读出数据 |
| `rempty` | output | read | FIFO 空标志 |
| `arempty` | output | read | FIFO 将空标志 |

### 4.4 `wptr_full.v`

该模块工作在写时钟域，负责：

- 保存写端二进制指针 `wbin`。
- 输出写地址 `waddr = wbin[ADDRSIZE-1:0]`。
- 将下一个二进制指针转换为 Gray code，输出 `wptr`。
- 使用同步后的读指针 `wq2_rptr` 判断 `wfull`。
- 提前一拍判断 `awfull`。

写指针更新条件：

```verilog
assign wbinnext = wbin + ((winc & ~wfull) ? 1 : 0);
assign wgraynext = (wbinnext >> 1) ^ wbinnext;
```

满判断核心：

```verilog
assign wfull_val =
    (wgraynext == {~wq2_rptr[ADDRSIZE:ADDRSIZE-1], wq2_rptr[ADDRSIZE-2:0]});
```

含义是：当写指针追上读指针并绕回一圈时，FIFO 满。Gray 指针高两位取反、低位相等是经典异步 FIFO full 判断方式。

### 4.5 `rptr_empty.v`

该模块工作在读时钟域，负责：

- 保存读端二进制指针 `rbin`。
- 输出读地址 `raddr = rbin[ADDRSIZE-1:0]`。
- 将下一个二进制读指针转换为 Gray code，输出 `rptr`。
- 使用同步后的写指针 `rq2_wptr` 判断 `rempty`。
- 提前一拍判断 `arempty`。

读指针更新条件：

```verilog
assign rbinnext = rbin + ((rinc & ~rempty) ? 1 : 0);
assign rgraynext = (rbinnext >> 1) ^ rbinnext;
```

空判断核心：

```verilog
assign rempty_val = (rgraynext == rq2_wptr);
```

含义是：当下一读指针等于同步后的写指针时，读侧认为 FIFO 为空。

### 4.6 `sync_r2w.v` 和 `sync_w2r.v`

这两个模块都是两级寄存器同步器：

- `sync_r2w` 把读指针 `rptr` 同步到写域，输出 `wq2_rptr`。
- `sync_w2r` 把写指针 `wptr` 同步到读域，输出 `rq2_wptr`。

两级同步器不能消灭亚稳态，但可以显著降低亚稳态传播到后级逻辑的概率。因为跨域的是 Gray code 指针，即使同步采样发生在边沿附近，也只涉及单 bit 变化，风险比二进制多 bit 指针小。

### 4.7 `fifomem.v`

该模块是 FIFO 的数据存储：

```verilog
localparam DEPTH = 1 << ADDRSIZE;
reg [DATASIZE-1:0] mem [0:DEPTH-1];
```

写侧逻辑：

```verilog
always @(posedge wclk) begin
    if (wclken && !wfull)
        mem[waddr] <= wdata;
end
```

读侧根据 `FALLTHROUGH` 有两种模式：

- `"TRUE"`：`assign rdata = mem[raddr];`，首字直通，延迟更低。
- 非 `"TRUE"`：在 `rclk` 上寄存读数据，时序更规整。

### 4.8 双向 FIFO 扩展

`async_bidir_fifo.v` 和 `async_bidir_ramif_fifo.v` 扩展了基础 FIFO：

- `async_bidir_fifo.v`：A/B 两侧都可以读写，内部使用 `fifomem_dp`。
- `async_bidir_ramif_fifo.v`：保留指针、同步和控制逻辑，但把 RAM 接口暴露给外部。

报告主线建议聚焦 `async_fifo.v`，双向版本可以作为扩展设计说明，不要让 presentation 过度发散。

## 5. 可选验证和测试说明

本次没有运行测试，原因是当前任务明确说明 WSL 无图形界面，可以不用测试。需要注意的是，原项目的主要仿真不一定依赖 GUI，未来如果工具链完整，仍建议至少跑一次命令行仿真。

### 5.1 Lint

```bash
./flow.sh lint
```

作用：

- 调用 Verilator。
- 检查 `async_fifo` 及其基础子模块。
- 若 `lint.log` 中没有有效 `%Error:`，脚本认为 lint 成功。

### 5.2 Simulation

```bash
./flow.sh sim
```

作用：

- 进入 `sim/`。
- 使用 SVUT 和 Icarus Verilog 跑 `async_fifo_unit_test.sv`。
- `script/setup.sh` 会在找不到 `svutRun` 时尝试克隆 SVUT 到本地脚本目录。

已有测试覆盖：

- idle 状态下 `wfull == 0`、`rempty == 1`。
- 单次写入后读出。
- 多次连续写入后按顺序读出。
- full flag。
- empty flag。
- almost-empty flag。
- almost-full flag。
- 并发读写。

### 5.3 Synthesis

```bash
./flow.sh syn
```

作用：

- 进入 `syn/`。
- 调用 `syn/syn_asic.sh`。
- 使用 Yosys 执行 `fifo.ys`。
- 生成 `async_fifo_syn.v`。

课程 PDF 中 STA 是 bonus。若时间紧，不建议把 STA 放在主路径；如果后续要做，应先保证基础报告和 presentation 完成。

## 6. 报告写作模板

下面是建议直接用于技术报告的结构。

### 6.1 Project overview

可写内容：

- Async FIFO 是跨时钟域数据缓冲器。
- 适用于 producer 和 consumer 频率不同、相位未知或完全异步的系统。
- 典型场景包括总线桥接、数据采集、串并转换接口、SoC 子系统跨域通信。
- 本设计支持参数化数据宽度 `DSIZE` 和深度 `2**ASIZE`。
- 设计提供 `full`、`empty`、`almost full`、`almost empty` 状态标志。

### 6.2 Build & simulation

由于本次 WSL 环境不做测试，报告中可以诚实写：

- 原项目提供 Verilator lint、Icarus Verilog + SVUT 仿真、Yosys 综合脚本。
- 本次复现阶段重点为代码阅读和结构分析，没有执行 GUI 波形验证。
- 后续具备工具链时可运行 `./flow.sh lint`、`./flow.sh sim`、`./flow.sh syn`。

如果后续实际跑通，再补充 pass log 和关键截图。

### 6.3 Module diagram

报告中必须有图。建议至少画一张基础 `async_fifo` 的模块图：

- 左侧写时钟域：`winc`、`wdata`、`wptr_full`、`sync_r2w`。
- 中间 FIFO memory：`fifomem`。
- 右侧读时钟域：`rptr_empty`、`sync_w2r`、`rinc`、`rdata`。
- 标出 `wptr` 从写域跨到读域，`rptr` 从读域跨到写域。
- 标出跨域指针是 Gray code。

### 6.4 Interface description

建议分成写端和读端：

写端：

- `wclk`：写时钟。
- `wrst_n`：写域复位。
- `winc`：写请求。
- `wdata`：写数据。
- `wfull`：满标志，满时不应继续写。
- `awfull`：将满标志，可用于提前背压。

读端：

- `rclk`：读时钟。
- `rrst_n`：读域复位。
- `rinc`：读请求。
- `rdata`：读数据。
- `rempty`：空标志，空时不应继续读。
- `arempty`：将空标志，可用于提前停止读。

### 6.5 Micro-architecture

建议按数据路径和控制路径拆开写。

数据路径：

- 写端接收 `wdata`。
- `wptr_full` 产生 `waddr`。
- `fifomem` 在 `wclk` 上写入 `mem[waddr]`。
- `rptr_empty` 产生 `raddr`。
- `fifomem` 根据 `raddr` 输出 `rdata`。

控制路径：

- 写域维护 `wbin` 和 `wptr`。
- 读域维护 `rbin` 和 `rptr`。
- `wptr` 以 Gray code 形式通过 `sync_w2r` 到读域。
- `rptr` 以 Gray code 形式通过 `sync_r2w` 到写域。
- 写域根据同步后的读指针判断 full。
- 读域根据同步后的写指针判断 empty。

### 6.6 Code walk-through

建议不要逐行解释，而是按文件职责写：

- `async_fifo.v`：顶层连接。
- `wptr_full.v`：写指针、写地址、满判断。
- `rptr_empty.v`：读指针、读地址、空判断。
- `sync_r2w.v` / `sync_w2r.v`：跨域指针同步。
- `fifomem.v`：数据存储和 fall-through 读模式。
- `async_bidir_fifo.v`：双向扩展。
- `sim/async_fifo_unit_test.sv`：现有验证用例。

### 6.7 Verification

如果不跑测试，可以写成“现有验证计划分析”：

- 原项目测试计划考虑写时钟快于读时钟、慢于读时钟、相近频率等情况。
- 单元测试覆盖基础写读、连续写读、满/空标志、almost 标志和并发读写。
- 当前缺口包括更大随机压力、更系统的覆盖率统计、形式验证、复位异步释放场景等。

### 6.8 Findings

报告可以选择 3-5 条，要求具体。

可选 findings：

1. 设计采用 Gray code 指针跨域，这是异步 FIFO 的经典可靠做法，比直接同步二进制指针更安全。
2. full 和 empty 判断分别放在写域和读域，避免了跨域组合逻辑，结构清晰。
3. 两级同步器设计简单，但同步延迟会导致 full/empty 状态具有保守性，这是 CDC 设计中常见且合理的取舍。
4. `FALLTHROUGH` 参数提供低延迟读数据路径，但不同 FPGA/ASIC memory 推断方式可能不同，综合时需要检查目标工艺结果。
5. 双向 FIFO 文件中的部分注释与 `dir` 信号含义需要仔细核对，报告中可以作为代码可读性改进点提出。

### 6.9 References

建议引用：

- Upstream repository: https://github.com/dpretet/async_fifo
- Clifford Cummings, “Simulation and Synthesis Techniques for Asynchronous FIFO Design”.
- Course Final Project PDF。
- Upstream `doc/specification.rst` and `doc/testplan.rst`。

## 7. Presentation 建议

5-8 分钟不要讲太散，建议 7 页左右：

1. 题目页：Async FIFO Reproduction and Analysis。
2. 背景页：为什么需要跨时钟域 FIFO。
3. 顶层结构页：模块图。
4. 核心机制页：Gray code pointer + two-flop synchronizer。
5. full/empty 判断页：写域 full、读域 empty。
6. 代码 walkthrough 页：关键文件职责。
7. Findings 和总结页：3-5 条观察。

时间分配：

- 背景和用途：1 分钟。
- 模块结构：1.5 分钟。
- 微架构核心：2-3 分钟。
- 代码和验证：1 分钟。
- findings：1 分钟。

不要把 presentation 做成逐行代码朗读。重点是让听众看懂结构和设计取舍。

## 8. 后续开发建议

短期优先级：

1. 用 `guide.md` 的报告模板写技术报告初稿。
2. 画 `async_fifo` 模块图。
3. 把 Gray code、同步器、full/empty 判断整理成 2-3 张 PPT 图。
4. 整理 3-5 条 findings。
5. 如果还有时间，再尝试命令行 lint/sim，不要把 STA 放在主路径。

建议新增的文档或素材：

- `doc/report_outline.zh-CN.md`：中文报告草稿。
- `doc/module_diagram.drawio`：模块图源文件。
- `doc/presentation_outline.zh-CN.md`：展示讲稿。
- `doc/findings.md`：观察和改进点。

## 9. 快速命令清单

安装 Git，WSL / Ubuntu：

```bash
sudo apt update
sudo apt install -y git
git --version
```

首次配置 Git：

```bash
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
git config --global core.autocrlf false
git config --global core.safecrlf warn
```

配置 SSH：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com
```

克隆团队仓库：

```bash
git clone git@github.com:XFDG/async_fifo.git
cd async_fifo
```

查看状态：

```bash
git status --short --branch
git remote -v
```

忽略 CRLF 干扰查看差异：

```bash
git diff --ignore-cr-at-eol
git diff --numstat --ignore-cr-at-eol
```

可选验证：

```bash
./flow.sh lint
./flow.sh sim
./flow.sh syn
```

添加自己的 GitHub remote：

```bash
git remote rename origin upstream
git remote add origin git@github.com:XFDG/async_fifo.git
```

创建分支并提交：

```bash
git checkout -b docs/my-task
git add guide.md README.md
git commit -m "docs: update project documentation"
```

推送：

```bash
git push -u origin docs/my-task
```
