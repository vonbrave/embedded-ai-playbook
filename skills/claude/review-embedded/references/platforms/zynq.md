# 平台：Xilinx Zynq / Zynq UltraScale+（XCZU 系列）

## 自动检测特征

Grep 命中以下任意一项即判定为 Zynq 平台：

```
xil_io\.h|xparameters\.h|xil_cache\.h
xaxidma\.h|xscugic\.h|xil_exception\.h
zynq|zynqmp|xlnx
/dev/mem|/dev/uio|/dev/xdma
```

---

## 追加 Grep 预筛模式

| 类别 | 模式 | 关注点 |
|------|------|--------|
| /dev/mem 映射 | `\/dev\/mem\|\/dev\/uio` | mmap 配对、权限检查 |
| AXI DMA | `XAxiDma\|xdma\|axidma` | 传输完成轮询/中断、缓冲区对齐 |
| Cache 一致性 | `Xil_DCacheFlush\|Xil_DCacheInvalidate\|__sync_synchronize` | DMA 前后 cache 操作 |
| UIO 驱动 | `uio_open\|\/dev\/uio[0-9]` | fd 释放、中断等待 |
| XDMA 驱动 | `\/dev\/xdma[0-9]\|xdma_read\|xdma_write` | 传输大小对齐、返回值 |
| 寄存器访问 | `Xil_In32\|Xil_Out32\|mmap.*MAP_SHARED` | volatile、地址对齐 |
| 共享内存 | `shm_open\|\/dev\/shm` | PS-PL 数据交换区管理 |

---

## 平台专项审查清单

### /dev/mem 与物理地址映射

- [ ] `open("/dev/mem", ...)` 后是否在所有路径 `close(fd)`
- [ ] `mmap` 映射物理地址时是否检查 `MAP_FAILED`
- [ ] `mmap` 是否配对 `munmap`，且 `munmap` 的 size 与 `mmap` 一致
- [ ] 映射的物理地址是否与 `xparameters.h` 中的基地址定义一致（防止地址写错）
- [ ] 是否使用了 `volatile` 修饰寄存器指针（防编译器优化）

### AXI DMA / XDMA

- [ ] DMA 传输前是否对源缓冲区做 `Xil_DCacheFlush`（ARM → PL 方向）
- [ ] DMA 传输完成后是否对目标缓冲区做 `Xil_DCacheInvalidate`（PL → ARM 方向）
- [ ] DMA 缓冲区地址是否满足对齐要求（通常 64B 或 4K 对齐）
- [ ] 传输完成检测：轮询模式下是否有超时保护（避免死等 PL 无响应）
- [ ] XDMA 设备 fd（`/dev/xdma0_h2c_0` 等）是否在所有路径 `close`
- [ ] `xdma_write` / `xdma_read` 的传输大小是否按 XDMA IP 要求对齐

### UIO 中断驱动

- [ ] `open("/dev/uio0", ...)` 是否在所有路径 `close`
- [ ] `read(uio_fd, ...)` 等待中断时是否有超时（`select` / `poll` + 超时参数）
- [ ] 中断计数 `uint32_t` 是否处理了计数溢出回绕

### PS-PL 数据交换（共享内存 / AXI BRAM）

- [ ] PS 写入共享内存后是否有内存屏障（`__sync_synchronize()` 或 `dsb()`）保证 PL 可见
- [ ] PL 更新的状态寄存器读取是否用 `volatile` 防止被缓存
- [ ] 环形缓冲区的读写指针是否做了原子操作或加锁保护

### 设备树 / sysfs 操作

- [ ] sysfs 属性文件（`/sys/class/...`）读写后是否 `close`
- [ ] GPIO sysfs 操作（export / direction / value）是否做了完整的错误处理

---

## 严重程度参考（Zynq 特有）

| 场景 | 等级 |
|------|------|
| DMA 传输前后缺少 cache flush/invalidate → 数据静默错误 | 🔴 |
| DMA 缓冲区未对齐 → 传输异常或总线错误 | 🔴 |
| 轮询 DMA 完成无超时 → 系统挂死 | 🔴 |
| PS 写共享内存后无内存屏障 → PL 读到旧数据 | 🟡 |
| `/dev/mem` fd 泄漏 | 🟡 |
| UIO 中断等待无超时 | 🟡 |
| sysfs fd 未 close（少量泄漏） | 🔵 |
