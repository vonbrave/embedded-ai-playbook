---
name: review-embedded
description: 嵌入式 C/C++ 代码审查。当用户提到「审查代码」「code review」「review」「检查代码质量」「查内存泄漏」「找 bug」时必须触发此技能。支持两种模式：full（全量问题）和 focus（重点高危问题，默认）。
allowed-tools: Read, Write, Grep, Glob, Bash, Agent
argument-hint: "[--full | --focus] [--platform rockchip|zynq|nvidia] [文件或目录 ...]"
---

# 嵌入式代码审查

## 参数解析

从 `$ARGUMENTS` 中提取：

| 参数 | 含义 |
|------|------|
| `--full` | **全量模式**：报告全部 🔴🟡🔵 问题，覆盖所有审查类别 |
| `--focus`（默认） | **重点模式**：只报告 🔴 严重 + 🟡 高价值警告，略过低优先级建议 |
| `--platform <名称>` | **手动指定平台**，跳过自动检测。可选值：`rockchip` / `zynq` / `nvidia` |
| 其余参数 | 文件或目录路径，规则见下 |

**路径规则**（无路径参数 → 扫描当前目录）：
- 目录 → 递归收集 `*.c` / `*.cpp` / `*.cu`
- 文件 → 直接使用
- 混合 → 合并去重

---

## 执行流程（六阶段）

### 阶段 1 · 文件收集

用 `Glob` 收集目标文件，统计总数后输出：
```
🔍 共发现 N 个源文件 | 模式：<full/focus> | 开始审查...
```

### 阶段 2 · 平台检测

**若已有 `--platform` 参数**：直接使用指定平台，跳过检测，输出：
```
📌 平台：<名称>（手动指定）
```

**若无 `--platform`**：用 `Grep` 并行扫描以下特征（`output_mode: "files_with_matches"`），命中即确定平台：

| 平台 | 检测模式 |
|------|---------|
| Rockchip | `mpp_api\.h\|rk_mpi\.h\|rga\.h\|im2d\.h\|rockchip_drm\.h` |
| Zynq | `xil_io\.h\|xparameters\.h\|xaxidma\.h\|\/dev\/uio\|\/dev\/xdma` |
| Nvidia | `cuda_runtime\.h\|NvBufSurface\.h\|nvinfer\.h\|cudaMalloc` |

命中后输出：
```
🔎 自动识别平台：<名称>（依据：<命中文件>）
```

未命中任何平台时输出：
```
⚠️  未识别到已知平台，仅执行通用审查
```

确定平台后，读取对应的平台文件，将其**追加 Grep 模式**合并到阶段 3，将其**专项审查清单**合并到阶段 4/5 的审查范围：

| 平台 | 参考文件 |
|------|---------|
| Rockchip | `references/platforms/rockchip.md` |
| Zynq | `references/platforms/zynq.md` |
| Nvidia | `references/platforms/nvidia.md` |

> 新增平台只需在 `references/platforms/` 下添加新文件并遵循相同格式，无需修改 SKILL.md。

### 阶段 3 · Grep 预筛高风险模式

**在读全文之前**，并行发起以下 Grep（`output_mode: "content"`, `-C 3`），快速定位高风险行：

> 详细模式列表见 → `references/grep-patterns.md`

核心必查模式（每次审查都扫）：
- `malloc|calloc|realloc|mpp_malloc|dma_buf_alloc`
- `mmap\(`
- `sprintf\(`
- `strcpy|strcat|gets\(`
- `\bopen\(|socket\(`
- `\bsystem\(`
- `pthread_create`
- `ioctl\(`

记录命中的**文件 + 行号**，作为阶段 3 的输入。

### 阶段 4 · 精读命中文件

仅对 Grep 命中的文件调用 `Read`（用 offset/limit 聚焦命中行附近）。

读取优先级：
1. 内存分配器、DMA、设备驱动、HAL（含平台特定驱动）
2. 网络通信、协议、录制
3. 算法、OSD、工具类

### 阶段 5 · 分治审查（文件数 > 20 时启用 Agent）

按模块拆分，并行启动子 Agent：

> 子 Agent prompt 模板见 → `references/agent-prompt.md`（模板中包含平台专项清单的注入位置）

默认模块拆分：
```
Agent 1: source/platform/, source/common/
Agent 2: source/device/, source/protocol/, source/rtsp/
Agent 3: source/algorithm/
Agent 4: adapter/, source/hetero/, source/factorys/, source/system/
Agent 5: source/osd/, source/drm/, source/record/, source/log/
```

文件 ≤ 20 时，主进程直接审查，无需拆分。

### 阶段 6 · 生成报告

收集所有结果，去重合并后：

1. `mkdir -p doc`
2. 写入 `doc/review-embedded-<YYYY-MM-DD>.md`（同日多次则加 `-2`、`-3` 后缀）
3. 终端只打印汇总统计 + 报告路径

报告头部需包含识别到的平台信息，平台专项问题单独归为「**平台专项**」类别。

> 报告模板见 → `references/report-template.md`

---

## 模式差异

| 项目 | `--focus`（默认） | `--full` |
|------|-----------------|---------|
| 报告 🔴 严重 | ✅ 全部 | ✅ 全部 |
| 报告 🟡 警告 | ✅ 仅高价值（见下） | ✅ 全部 |
| 报告 🔵 建议 | ❌ 跳过 | ✅ 全部 |
| 审查类别 | 1–6（跳过栈使用） | 1–7 全部 |
| 适用场景 | 快速 PR 审查、CI 卡点 | 深度代码审计 |

**focus 模式下，🟡 只报告以下高价值警告**：
- ioctl 未检查返回值
- 错误路径缺少资源清理
- 全局变量无锁保护
- DMA sync 缺失

---

## 审查清单

> 详细逐条检查项见 → `references/checklist.md`

快速索引：
1. **内存管理** — malloc/free 配对、DMA/MPP 缓冲区、mmap 配对
2. **缓冲区安全** — sprintf→snprintf、memcpy 越界、整数溢出
3. **资源泄漏** — fd 所有路径 close、goto cleanup 覆盖
4. **线程安全** — 共享变量加锁、pthread 句柄管理
5. **硬件相关** — DMA sync、ioctl 清零、fd 有效性
6. **错误处理** — 返回值检查、错误路径日志
7. **栈使用**（仅 full）— 大局部数组、深层递归

---

## 严重程度分级

| 等级 | 标志 | 含义 |
|------|------|------|
| 严重 | 🔴 | 必定导致崩溃、数据损坏或安全漏洞 |
| 警告 | 🟡 | 特定条件下可能出问题 |
| 建议 | 🔵 | 代码质量改进，非紧急 |
