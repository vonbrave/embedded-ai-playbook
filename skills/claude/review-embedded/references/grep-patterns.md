# Grep 预筛模式参考

本文件列出所有 Grep 预筛模式。阶段 2 中，**所有模式并行发起**（单条消息内多个 Grep 调用）。

每条 Grep 使用 `output_mode: "content"`，上下文行 `-C 3`。

---

## 核心模式（full + focus 模式均扫）

| 类别 | 模式 | 关注点 |
|------|------|--------|
| 内存分配 | `malloc\|calloc\|realloc\|aligned_alloc\|mpp_malloc` | 返回值校验、配对 free |
| DMA 分配 | `dma_buf_alloc\|mpp_buffer_get\|drm_buf_alloc` | 配对 free/put |
| mmap | `mmap\(` | MAP_FAILED 检查、配对 munmap |
| 缓冲区 | `sprintf\(` | 应替换为 snprintf |
| 缓冲区 | `strcpy\|strcat\|gets\(` | 无边界保护 |
| 资源 | `\bopen\(\|socket\(` | 所有路径 close |
| 命令注入 | `\bsystem\(` | 参数来源校验 |
| 线程 | `pthread_create` | 句柄管理、join/detach |
| 硬件 | `ioctl\(` | 结构体清零、返回值 |

---

## 扩展模式（仅 full 模式追加扫描）

| 类别 | 模式 | 关注点 |
|------|------|--------|
| 栈风险 | `char\s+\w+\[[0-9]\{4,\}\]` | 大局部数组（≥1000 字节） |
| 递归 | 函数名自调用模式 | 深层递归风险 |
| 整数溢出 | `width\s*\*\s*height\|stride\s*\*` | 乘法结果未做溢出保护 |
| sem 配对 | `sem_post\|sem_wait` | 信号量配对检查 |
| V4L2 | `VIDIOC_REQBUFS\|VIDIOC_QBUF` | 缓冲区释放链路 |
| 信号处理 | `signal\(\|sigaction\(` | 非 async-signal-safe 调用 |

---

## 项目特定模式（按需添加）

如项目有自定义分配器或平台 API，在此追加：

```
# 示例：RGA / MPP 平台
rga_blit\|RGA_BLIT
mpp_task_meta_set\|mpp_enc_cfg_set
```

将项目特定模式追加到核心模式的并行 Grep 调用中。
