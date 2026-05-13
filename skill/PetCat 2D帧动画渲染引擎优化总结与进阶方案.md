# PetCat 2D 帧动画渲染引擎优化总结与进阶方案


> 该文档聚焦从 2D 帧动画播放瓶颈到业务可集成的 GPU-Native Runtime SDK 的设计路线。当前项目从 0 构建，首版直接使用 OpenGL ES，不再规划 Canvas backend。


## 1. 当前问题归纳

轻量级 2D/多层帧动画引擎的主要瓶颈通常不在单一 draw call，而在以下链路叠加：

- 播放时 PNG/WEBP/Bitmap 解码占用 CPU。
- Bitmap 到 GPU 纹理的重复上传造成主线程或渲染线程阻塞。
- Canvas 路径缺少可控的 shader pass、FBO、多层合成与 GPU 资源生命周期管理。
- 多层背景、角色、阴影、特效分离绘制时，边缘融合与同步控制容易失真。
- Timeline 与 Audio 时钟未统一时，动画与声音会逐渐 drift。

## 2. 优化目标

| 阶段 | 目标 | 核心指标 |
| --- | --- | --- |
| 短期 | 消除同步解码与 GC 卡顿 | 720P 晚帧明显下降 |
| 中期 | OpenGL ES 全链路渲染 | 720P 渲染 P50 约 5ms |
| 长期 | KTX2/BasisU GPU 纹理流 | 1080P/4K 稳定、多层实时合成 |

## 3. 推荐技术路线


### 3.1 产品形态

对外交付物应是 Android SDK，业务通过 `PetCatPlayerView`、`PetCatPlayer`、`PetCatClip`、`PetCatConfig` 和 callback 集成；Runtime/Engine 是 SDK 内部实现，不应要求业务直接管理 GL、JNI 或 C++ 对象。

### 3.2 渲染后端

第一版直接使用 OpenGL ES 3.0，理由是 Android 设备覆盖率高、接入复杂度低、足以支撑 2D atlas、多层 blend、FBO 与基础 shader FX。

后续可在架构层预留 RenderBackend 接口，但不建议首版同时实现 Vulkan，也不建议先做 Canvas MVP 再替换，以免拉长验证周期。

### 3.3 资源格式


长期格式应以 KTX2 + BasisU 为核心：

- 离线阶段完成帧提取、atlas packing、纹理压缩、timeline/material 生成。
- Runtime 阶段只做轻量 metadata parse、按需 transcode/upload、cache 管理。
- 播放中禁止重型图片解码与逐帧 `glTexImage2D`。


### 3.4 多层合成


建议 layer 顺序固定为：

```text
Background → Back FX → Character Shadow → Character → Front FX → Overlay/UI
```

每层需要包含：

- texture handle
- transform
- opacity
- blend mode
- material/shader reference
- visibility time range


### 3.5 Timeline 与 Audio


Audio 应作为 master clock。Timeline 每帧根据 audio time 计算当前 frame、layer visibility、transition 与 shader uniforms。无音频 clip 可使用 monotonic clock fallback，但接口仍保持同一套 master clock 抽象。

## 4. 分阶段落地建议


1. **先做 SDK 工程骨架**：Android SDK module + sample-app + NDK + JNI + 空 GL context。

2. **再做最小绘制**：全屏 quad + 单纹理 + render loop + FrameStats。
3. **再做 layer/timeline**：不要过早引入复杂 shader。
4. **再做 TextureCache**：明确所有权、LRU、水位与异步上传。
5. **再接 KTX2/BasisU**：资源 schema 稳定后再做。
6. **最后做高级 FX 与 4K**：以 benchmark 数据决定优化优先级。

## 5. 设计红线

- GL thread 不做磁盘 IO、重型 JSON parse、阻塞等待。
- Runtime 播放路径不做 PNG/WEBP 重型解码。
- 不允许每帧重新创建 shader、framebuffer 或大纹理。
- 不把 UI 生命周期逻辑写入 C++ 渲染核心。

- 不要求业务直接调用 NativeBridge 或管理 GL 资源。
- 不引入完整 ECS/物理系统，保持 runtime 聚焦播放与合成。
- Canvas 不作为计划内渲染 backend；设备不支持时通过质量档位、禁用 FX、减少层数或明确错误码处理。


## 6. 与 Spec Coding 的关系

本文提供优化方向；真正实现时应拆到 `docs/SPEC_CODING_PLAN.md` 中定义的波次，并为每个任务创建独立 spec。首批任务应优先固定工程版本、Native API、资源 schema 与诊断指标。
