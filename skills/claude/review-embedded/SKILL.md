---
name: review-embedded
description: 嵌入式 C/C++ 代码审查，检查内存泄漏、资源未释放、缓冲区溢出、线程安全、DMA 一致性等嵌入式常见问题。当用户要求审查代码、检查代码质量、或提到 review 时触发。
allowed-tools: Read, Write, Grep, Glob, Bash, Agent
argument-hint: "[文件或目录 ...]  不传参数则审查当前目录下所有 C/C++ 源文件"
---

# 嵌入式代码审查

## 审查目标解析

根据 `$ARGUMENTS` 确定待审查的文件范围，规则如下：

1. **无参数**（`$ARGUMENTS` 为空）：扫描当前工作目录下所有 `*.c` 和 `*.cpp` 文件（递归子目录），跟随符号链接。
2. **参数为目录**（一个或多个）：扫描指定目录下所有 `*.c` 和 `*.cpp` 文件（递归子目录），跟随符号链接。
3. **参数为文件**（一个或多个）：仅审查指定的文件。
4. **混合参数**：可同时指定目录和文件，分别按上述规则处理后合并去重。

## 执行策略

### 第一阶段：收集文件列表

使用 `Glob` 工具收集目标文件：
- 无参数时：`pattern: "**/*.c"` 和 `pattern: "**/*.cpp"`
- 指定目录时：`pattern: "<目录>/**/*.c"` 和 `pattern: "<目录>/**/*.cpp"`
- 指定文件时：直接使用文件路径

统计文件总数并展示给用户：`共发现 N 个源文件，开始审查...`

### 第二阶段：Grep 预筛高风险模式

**在读取任何文件全文之前**，先用 `Grep` 工具在目标范围内搜索以下危险模式，快速定位高风险代码位置（每个模式使用 `output_mode: "content"` 附带上下文行 `-C 3`）：

| 审查类别 | Grep 模式 | 目的 |
|---------|-----------|------|
| 内存管理 | `malloc\|calloc\|realloc\|aligned_alloc\|mpp_malloc` | 检查返回值校验、配对释放 |
| 内存管理 | `mmap\(` | 检查 MAP_FAILED、配对 munmap |
| 内存管理 | `dma_buf_alloc\|mpp_buffer_get\|drm_buf_alloc` | 检查配对释放 |
| 缓冲区 | `sprintf\(` | 应替换为 snprintf |
| 缓冲区 | `strcpy\|strcat\|gets\(` | 无边界保护 |
| 资源泄漏 | `\bopen\(\|socket\(` | 检查所有路径上的 close |
| 命令注入 | `\bsystem\(` | 检查参数来源 |
| 线程安全 | `pthread_create` | 检查线程句柄管理 |
| 硬件相关 | `ioctl\(` | 检查结构体清零、返回值 |

**关键**：每个 Grep 搜索应并行发起（单条消息内多个 Grep 调用），并记录命中的文件和行号。

### 第三阶段：精读命中文件

仅对 Grep 命中的文件调用 `Read` 读取全文（或使用 offset/limit 读取命中行附近上下文）。按以下优先级排序：

- **优先级 1**：内存分配器、DMA、设备驱动、平台 HAL
- **优先级 2**：网络通信、协议处理、录制
- **优先级 3**：算法、OSD、工具类

### 第四阶段：Agent 分治（文件数 > 20 时启用）

当待审查文件较多时，**必须**使用 `Agent` 工具并行分组审查。按模块拆分为多个子 Agent，每个子 Agent 拥有独立上下文窗口：

```
Agent 1: 平台层     → source/platform/, source/common/
Agent 2: 设备/网络  → source/device/, source/protocol/, source/rtsp/
Agent 3: 算法       → source/algorithm/
Agent 4: 适配器     → adapter/, source/hetero/, source/factorys/, source/system/
Agent 5: 其他       → source/osd/, source/drm/, source/record/, source/log/
```

每个子 Agent 的 prompt 模板：
```
对以下文件进行嵌入式 C/C++ 代码审查。
审查范围：<文件列表>

按以下类别检查，只报告发现的实际问题：
1. 内存管理（malloc/free 配对、返回值检查、DMA/MPP 缓冲区配对）
2. 缓冲区安全（sprintf→snprintf、memcpy 越界、整数溢出）
3. 资源泄漏（fd 未 close、错误路径缺少清理）
4. 线程安全（全局变量锁保护、pthread 管理）
5. 硬件相关（DMA sync、ioctl 清零、fd 有效性）
6. 错误处理（返回值未检查、错误路径日志）
7. 栈使用（大局部数组、深层递归）

对每个问题输出：
[严重程度 🔴/🟡/🔵] 问题类别 — 文件:行号
问题描述（一句话）
关键代码片段
修复建议
```

子 Agent 应使用 `Grep`（预筛）+ `Read`（精读）的两阶段方法审查其负责的文件。

### 第五阶段：生成报告文件

主进程收集所有子 Agent 返回的结果，去重、合并后：

1. 使用 `Bash` 确保 `doc/` 目录存在：`mkdir -p doc`
2. 使用 `Write` 工具将完整审查报告写入 markdown 文件，路径规则：
   - 文件路径：`doc/review-embedded-report-<YYYY-MM-DD>.md`
   - 日期取当天日期（从系统获取）
   - 若同一天多次审查，后缀追加序号：`review-embedded-report-<YYYY-MM-DD>-2.md`
3. 在终端向用户展示报告摘要（仅汇总统计部分），并告知完整报告路径。

报告文件内容使用下方的「输出格式」组织。

## 审查清单

按以下类别逐项检查，**只报告发现的实际问题**，没问题的类别跳过不提：

### 1. 内存管理
- malloc/calloc 后是否检查返回值
- 每条分配路径（含错误分支）是否都有对应的 free
- mmap 是否配对 munmap，且检查了 MAP_FAILED
- DMA 缓冲区(dma_buf_alloc)是否配对 dma_buf_free
- MPP 缓冲区(mpp_buffer_get)是否配对 mpp_buffer_put
- 释放后是否置 NULL/-1 防止 double-free（本项目约定）
- 是否存在在循环中重复分配而未释放的情况

### 2. 缓冲区安全
- memcpy/memset 的 size 参数是否可能越界
- sprintf → 应使用 snprintf
- 固定大小数组是否有越界写入风险
- 字符串操作(strcpy/strcat)是否有长度保护
- 尺寸计算 width*height*bpp 是否可能整数溢出

### 3. 资源泄漏
- 文件描述符(open/socket)在所有路径上是否 close
- ioctl 失败后是否清理已获取的资源
- goto cleanup 路径是否覆盖了所有已分配的资源
- 错误返回前是否遗漏了 releasebuffer_handle 等清理调用
- V4L2 MMAP 缓冲区是否正确 munmap + REQBUFS(0) 释放

### 4. 线程安全
- 全局变量/共享状态的读写是否有锁保护
- sem_post/sem_wait 配对是否正确
- pthread_create 后线程句柄是否可用于 join/detach
- 信号处理函数中是否调用了非 async-signal-safe 函数

### 5. 硬件相关
- DMA 缓冲区在 CPU 读写前后是否调用了 dma_sync 同步
- ioctl 调用前结构体是否用 memset 清零
- 设备文件打开前是否做了 stat 检查
- fd 有效性检查（本项目用 fd < 0 或 fd == 0 表示无效）

### 6. 错误处理
- 函数返回值是否被检查（特别是 ioctl、mpp_*、rga_* 系列）
- 错误路径是否有日志输出（SLOGI/logger_info/printf）
- 部分初始化失败时是否清理了已成功初始化的部分

### 7. 栈使用
- 函数内是否有大的局部数组（嵌入式线程栈空间有限）
- 是否有深层递归调用风险

## 输出格式

报告文件（markdown）使用以下结构：

```markdown
# 嵌入式代码审查报告

- **审查日期**：YYYY-MM-DD
- **审查范围**：<目标描述，如"当前目录所有 C/C++ 文件"或具体路径>
- **文件总数**：N 个
- **发现问题**：🔴 X 个 / 🟡 Y 个 / 🔵 Z 个

---

## 问题详情

### 1. 内存管理

**🔴 严重 — 文件:行号**
> 问题描述（一句话）

```c
现有代码片段（关键几行）
```

**修复建议**：具体修复方案

---

（按类别分组列出所有问题，无问题的类别跳过不列）

---

## 汇总统计

### 按严重程度

| 严重程度 | 数量 |
|---------|------|
| 🔴 严重 | X |
| 🟡 警告 | Y |
| 🔵 建议 | Z |
| **合计** | **N** |

### 按文件分布

| 文件 | 🔴 | 🟡 | 🔵 | 合计 |
|------|-----|-----|-----|------|
| source/xxx.c | 1 | 2 | 0 | 3 |
| ... | ... | ... | ... | ... |

### 按类别分布

| 类别 | 数量 |
|------|------|
| 内存管理 | X |
| 缓冲区安全 | X |
| 资源泄漏 | X |
| 线程安全 | X |
| 硬件相关 | X |
| 错误处理 | X |
| 栈使用 | X |
```

严重程度分级：
- 🔴 严重：必定导致崩溃、数据损坏或安全漏洞
- 🟡 警告：特定条件下可能出问题
- 🔵 建议：代码质量改进，非紧急

终端输出仅展示汇总统计表格，并提示：`完整报告已保存至 doc/review-embedded-report-<日期>.md`
