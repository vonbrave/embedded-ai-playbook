# 平台：NVIDIA Jetson Xavier NX / Orin NX

## 自动检测特征

Grep 命中以下任意一项即判定为 Nvidia NX 平台：

```
cuda_runtime\.h|cuda\.h|cuda_runtime_api\.h
NvBufSurface\.h|nvbuf_utils\.h|NvBufSurfaceUtils\.h
nvinfer\.h|NvInfer\.h|NvOnnxParser\.h
tegra_drm\.h|nvmedia
cudaMalloc\|cudaFree\|cudaMemcpy
```

---

## 追加 Grep 预筛模式

| 类别 | 模式 | 关注点 |
|------|------|--------|
| CUDA 内存 | `cudaMalloc\|cudaMallocManaged\|cudaMallocHost` | 配对 cudaFree、返回值检查 |
| CUDA 同步 | `cudaMemcpy\|cudaLaunchKernel\|kernel<<<` | 异步操作后的同步 |
| CUDA 错误 | `cudaGetLastError\|checkCudaErrors\|CUDA_CHECK` | 错误检查是否覆盖所有调用 |
| NvBufSurface | `NvBufSurfaceCreate\|NvBufSurfaceDestroy` | 配对释放 |
| NvBufSurface Map | `NvBufSurfaceMap\|NvBufSurfaceUnMap` | 配对、cache sync |
| TensorRT | `createInferRuntime\|createExecutionContext\|deserializeCudaEngine` | 对象生命周期 |
| GStreamer | `gst_element_factory_make\|gst_object_unref\|gst_bus_timed_pop` | ref 计数管理 |
| V4L2 | `VIDIOC_REQBUFS\|VIDIOC_QBUF\|V4L2_MEMORY_DMABUF` | DMA-BUF 模式缓冲区 |
| CUDA Stream | `cudaStreamCreate\|cudaStreamDestroy\|cudaStreamSynchronize` | stream 生命周期 |

---

## 平台专项审查清单

### CUDA 内存管理

- [ ] `cudaMalloc` / `cudaMallocManaged` / `cudaMallocHost` 是否配对 `cudaFree` / `cudaFreeHost`
- [ ] CUDA API 返回值是否每次都检查（`cudaError_t` 不是 `cudaSuccess` 时处理）
- [ ] `cudaMemcpy` 方向参数（`cudaMemcpyHostToDevice` 等）是否与指针实际位置一致
- [ ] Unified Memory（`cudaMallocManaged`）在 CPU 访问前是否调用 `cudaDeviceSynchronize`

### CUDA 异步与同步

- [ ] Kernel 启动（`<<<...>>>`）后是否在适当位置调用 `cudaStreamSynchronize` 或 `cudaDeviceSynchronize`
- [ ] 异步 `cudaMemcpyAsync` 完成前 CPU 端是否提前读取了目标缓冲区
- [ ] `cudaGetLastError()` 是否在 Kernel 启动后检查（捕获 Kernel 配置错误）
- [ ] CUDA Stream 是否正确 `cudaStreamDestroy`，错误路径是否遗漏

### TensorRT 推理

- [ ] `IRuntime` / `ICudaEngine` / `IExecutionContext` 是否调用 `destroy()`（或用智能指针管理）
- [ ] 推理 binding buffer（`cudaMalloc` 的 GPU buffer）是否在所有路径 `cudaFree`
- [ ] `enqueueV2` / `executeV2` 返回值是否检查
- [ ] 序列化/反序列化 engine 时，`IHostMemory` 是否 `destroy()`

### NvBufSurface（零拷贝图像流）

- [ ] `NvBufSurfaceCreate` 是否配对 `NvBufSurfaceDestroy`
- [ ] CPU 访问前是否调用 `NvBufSurfaceMap`，访问后是否 `NvBufSurfaceUnMap`
- [ ] Map 后 CPU 写入，DMA 读取前是否调用 `NvBufSurfaceSyncForDevice`
- [ ] DMA 写入后，CPU 读取前是否调用 `NvBufSurfaceSyncForCpu`
- [ ] `NvBufSurface` 在多线程间传递时是否有同步保护

### GStreamer Pipeline

- [ ] `gst_element_factory_make` 返回值是否检查 NULL
- [ ] Pipeline 中所有 element 是否在销毁前 `gst_object_unref`
- [ ] `gst_bus_timed_pop_filtered` 是否处理了 `GST_MESSAGE_ERROR`
- [ ] appsink/appsrc 的 `GstSample` / `GstBuffer` 是否及时 `gst_sample_unref` / `gst_buffer_unref`

### V4L2 + DMA-BUF（Camera）

- [ ] `V4L2_MEMORY_DMABUF` 模式下，DMA-BUF fd 是否 `close`
- [ ] `VIDIOC_DQBUF` 后是否及时 `VIDIOC_QBUF` 归还缓冲区
- [ ] 退出时是否调用 `VIDIOC_STREAMOFF` + `VIDIOC_REQBUFS(0)`

---

## 严重程度参考（Nvidia NX 特有）

| 场景 | 等级 |
|------|------|
| CUDA 内存未释放（长期运行 GPU 内存 OOM） | 🔴 |
| 异步 Kernel 后无同步直接读取结果 → 数据错误 | 🔴 |
| `NvBufSurface` CPU 访问前未 Map → 段错误 | 🔴 |
| TensorRT 对象未 destroy → GPU 内存泄漏 | 🟡 |
| `NvBufSurfaceSyncForDevice/Cpu` 缺失 → 偶发图像错误 | 🟡 |
| CUDA API 返回值未检查 | 🟡 |
| GStreamer ref 计数泄漏（少量内存） | 🔵 |
