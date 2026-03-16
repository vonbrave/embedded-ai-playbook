# 平台：Rockchip（RK3588 / RK3568 / RK356x 系列）

## 自动检测特征

Grep 命中以下任意一项即判定为 Rockchip 平台：

```
mpp_api\.h|rk_mpi\.h|mpp_buffer\.h|rk_type\.h
rga\.h|im2d\.h|RgaApi\.h
rockchip_drm\.h|drm_fourcc\.h
```

---

## 追加 Grep 预筛模式

在通用模式基础上，**并行追加**以下 Grep：

| 类别 | 模式 | 关注点 |
|------|------|--------|
| MPP 编解码 | `mpp_create\|mpp_init\|mpp_destroy` | 生命周期配对 |
| MPP 缓冲区 | `mpp_buffer_get\|mpp_buffer_put\|mpp_buffer_group` | 配对释放 |
| MPP 帧/包 | `mpp_frame_deinit\|mpp_packet_deinit` | 每帧处理后释放 |
| MPP 任务 | `mpp_task_meta_set\|mpp_task_meta_get` | 任务对象管理 |
| RGA 操作 | `im2d::\|imresize\|imcopy\|imblend\|imsync` | 返回值、sync 调用 |
| RGA 缓冲区 | `importbuffer_\|releasebuffer_handle` | 配对释放 |
| DRM 显示 | `drmOpen\|drmModeGetResources\|drmModeAddFB` | fd 和资源释放 |
| V4L2 | `VIDIOC_REQBUFS\|VIDIOC_QBUF\|VIDIOC_STREAMON` | 缓冲区管理 |

---

## 平台专项审查清单

### MPP（Media Process Platform）

- [ ] `mpp_create` 后是否调用 `mpp_init`，销毁时是否调用 `mpp_destroy`
- [ ] `mpp_buffer_group_get` 是否配对 `mpp_buffer_group_put`
- [ ] `mpp_buffer_get` 是否配对 `mpp_buffer_put`（**每条路径**，含错误分支）
- [ ] `mpp_frame_init` / `mpp_packet_init` 是否配对 `deinit`
- [ ] 解码循环中，每帧 `mpp_frame_get` 后是否及时 `mpp_frame_deinit`（否则缓冲区池耗尽）
- [ ] `mpp_task_meta_set_*` 填充的任务，失败后是否释放已设置的资源
- [ ] MPP API 返回值（`MPP_OK` 判断）是否逐一检查

### RGA（2D 图形加速）

- [ ] `importbuffer_fd` / `importbuffer_virtualaddr` 是否配对 `releasebuffer_handle`
- [ ] `im2d` 操作返回值是否检查（`IM_STATUS_SUCCESS`）
- [ ] 异步操作后是否调用 `imsync()` 等待完成（否则 CPU 读到未完成数据）
- [ ] RGA 操作的源/目缓冲区格式（`rga_buffer_t`）是否与实际内存布局一致

### DRM 显示

- [ ] `drmOpen` 后是否 `close(fd)`（所有路径）
- [ ] `drmModeGetResources` 是否配对 `drmModeFreeResources`
- [ ] `drmModeGetConnector` 是否配对 `drmModeFreeConnector`
- [ ] `drmModeAddFB` / `drmModeAddFB2` 是否配对 `drmModeRmFB`
- [ ] GEM buffer / dumb buffer 是否通过 `DRM_IOCTL_GEM_CLOSE` 释放

### V4L2 + DMA-BUF

- [ ] `VIDIOC_REQBUFS` 申请的缓冲区，退出时是否调用 `VIDIOC_REQBUFS(count=0)` 释放
- [ ] MMAP 模式：每个缓冲区的 `mmap` 是否配对 `munmap`
- [ ] DMA-BUF 导出的 fd 是否 `close`
- [ ] `VIDIOC_DQBUF` 后处理完毕是否及时 `VIDIOC_QBUF` 归还（否则驱动缓冲区耗尽）

### RKNN / NPU 推理

- [ ] `rknn_init` 是否配对 `rknn_destroy`（所有路径）
- [ ] `rknn_inputs_set` / `rknn_run` / `rknn_outputs_get` 返回值是否检查
- [ ] **`rknn_buf_cpu_a/b/c` 等 CPU 缓冲区分配后，初始化失败时是否释放**（本项目 `data_engine.cpp` 存在爆炸式泄漏）
- [ ] NPU 推理结构体（非 POD 类型）禁止使用 `memset` 初始化，应使用构造函数或逐字段赋值
- [ ] RKNN 输出 tensor 使用完毕后是否调用 `rknn_outputs_release`
- [ ] 多路推理线程（yolo_A/B/C）共享 context 时是否加锁

### PCIe 设备

- [ ] PCIe fd（`fd_r`、`fd_w`）在 `pcie_impl_init` 的所有错误路径是否 `close`
- [ ] `rk3588_pcie_init` 中 4 个 `pthread_attr_t`（`attr_ir/read/swir/tv`）是否全部 `pthread_attr_destroy`
- [ ] **`get_mipi_frame_data` 中 `frame_info_chn.offset` 和 `.trans_mat[]` 必须在使用前初始化**（直接影响 DMA 偏移量，可能造成内存越界）
- [ ] `pcie_read_double()` 中时间戳变量（`t01/t4/t10/t20`）是否初始化（该函数同时存在无限循环，需确认有退出机制）
- [ ] PCIe 设备函数（`reg_rw_pcie_deinit`、`reset_pcie_ir_end`）有无返回值供调用方判断

### 看门狗（rk_wdt）

- [ ] `rk_watchdog_init()` 中 `fd_watchdog` 在错误路径是否 `close`
- [ ] `rk_watchdog_init()` 是否有 `return` 语句（本项目该函数缺少 return，调用方拿到未定义值）
- [ ] 喂狗间隔是否小于看门狗超时时间（避免系统意外重启）

### GPU 接口（RGA + OpenGL ES）

- [ ] `gpu_img_pip()` 中 `w/h/buf_c/buf_g` 等局部变量在所有分支路径上是否初始化
- [ ] `gpu_buf_osd()` 中 GPU buffer 地址变量是否初始化后再传入着色器
- [ ] `mmap_y_trace_buffer` / `unmmap_y_trace_buffer` 中 `gpu_mmap_addr` 未初始化时操作后果极严重，需检查所有调用路径

### 第三方库排除规则

审查时以下目录的问题**标注跳过**，不计入报告：
- `Eigen/`（线性代数库：`VOIDRET`、non-POD memcpy 均为已知特性）
- `OpenCV/` 头文件（`core_c.h`、`kmeans_index.h` 等）
- 其他 `third_party/` 或 `3rdparty/` 子目录

---

## 严重程度参考（Rockchip 特有）

| 场景 | 等级 |
|------|------|
| 解码循环未 `mpp_frame_deinit` → 缓冲区池耗尽 OOM | 🔴 |
| `importbuffer_*` 未 `releasebuffer_handle` → 长期 fd 泄漏 | 🔴 |
| `frame_info_chn.offset` 未初始化 → DMA 偏移错误可能越界写 | 🔴 |
| RKNN CPU 缓冲区初始化失败不回滚 → 内存泄漏 | 🔴 |
| YOLO 线程 `frame_info` 字段未初始化（MUST 类型）| 🔴 |
| `rk_watchdog_init` 缺 return + fd 泄漏 | 🔴 |
| RGA 异步操作后未 `imsync` → 图像数据错误 | 🟡 |
| MPP API 返回值未检查 | 🟡 |
| PCIe `pthread_attr_t` 未 destroy | 🟡 |
| GPU 接口局部变量未初始化（MIGHT 类型）| 🟡 |
| `drmModeGetResources` 未 `Free` → 少量内存泄漏 | 🟡 |
