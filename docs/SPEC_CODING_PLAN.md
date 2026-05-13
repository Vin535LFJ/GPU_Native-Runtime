# PetCat GPU-Native Runtime — Spec Coding 实施规划

> 本文档用于把现有设计文档转换为可执行、可验收、可逐步交付的 Spec Coding 计划。当前阶段只做架构与任务规划，不实现运行时代码。

## 1. 规划结论

PetCat Runtime 应按“先可运行骨架，再 GPU 核心链路，再资源格式闭环，再高级视觉效果”的顺序推进。不要一开始直接实现全部 Shader、BasisU、4K 与离线工具，否则会导致接口不稳定、调试成本过高。

推荐主线：

1. **工程骨架与边界契约**：Android/Kotlin、NDK/C++、JNI、CMake、测试目录、示例资产目录。
2. **最小可播放闭环（MVP）**：Timeline tick → NativeBridge → GLRenderer → 单纹理/单层 draw → 指标日志。
3. **多层合成闭环**：背景、角色、阴影、Overlay 的 layer model、blend mode、framebuffer 管理。
4. **纹理生命周期闭环**：TextureHandle、TextureCache、LRU、水位、异步上传队列。
5. **资产格式闭环**：先定义 `.anim`/`material.json` schema，再接入 KTX2/BasisU 与 petcat-builder。
6. **音频主时钟闭环**：AudioPlayer master clock 驱动 Timeline，处理 pause/seek/drift。
7. **Shader FX 迭代**：Feather、Alpha Blur、Shadow 优先；Bloom/LUT/RimLight 后置。
8. **性能与兼容性矩阵**：720P/1080P/4K、低端/中端设备、fallback 路径。

## 2. 需要补齐或澄清的设计点

### 2.1 Android 工程边界

现有文档明确了模块，但还需要在正式编码前固定：

- Android 包名、minSdk、targetSdk、Kotlin/AGP/NDK/CMake 版本。
- 是否使用 `GLSurfaceView` 作为第一版渲染宿主；建议 MVP 使用 `GLSurfaceView`，后续再抽象自定义 EGL。
- Kotlin 层是否只负责生命周期、调度与 JNI，避免业务逻辑下沉到 UI 层。

### 2.2 Native API 契约

需要先定义稳定的 JNI/Native API，再让各模块独立实现。建议把 API 分为：

- Runtime lifecycle：`init`、`resize`、`release`。
- Asset lifecycle：`loadClip`、`unloadClip`、`prefetchFrameRange`。
- Timeline control：`play`、`pause`、`seekTo`、`setPlaybackRate`。
- Render tick：`drawFrame`、`onVsync`。
- Diagnostics：`getFrameStats`、`setLogLevel`。

### 2.3 资源格式 Schema

当前文档提到了 `.anim`、`.material`、`.json`，但需要补齐字段级 schema：

- `clip.json`：分辨率、fps、duration、atlas 列表、音频文件、schema version。
- `timeline.anim`：tracks、layers、frames、frameRect、duration、events。
- `material.json`：shader、blend mode、uniform defaults、texture slots。

建议所有资源文件都带 `schemaVersion`，为未来兼容升级留出口。

### 2.4 线程模型与所有权

需要明确：

- **UI thread**：生命周期、触摸/业务事件、`Choreographer` 注册。
- **GL thread**：GL context、shader compile、texture upload、draw、FBO。
- **IO/worker threads**：磁盘 IO、KTX2 stream、Basis transcode、metadata parse。
- **Audio thread/system callback**：音频播放时间采样。

禁止 GL thread 做磁盘 IO、阻塞等待、重型 JSON 解析；禁止 worker thread 直接调用依赖当前 GL context 的 API。

### 2.5 MVP 资源策略

虽然长期目标是 KTX2/BasisU，但为了尽快验证渲染链路，MVP 可允许测试专用 fallback：

- Debug 示例使用小尺寸 RGBA raw/PNG 仅用于工程连通性。
- 正式 runtime contract 仍以 KTX2/BasisU 为目标。
- 所有 fallback 必须隔离在 debug/sample 或 loader adapter 中，不能污染核心渲染接口。

### 2.6 验收指标定义

现有指标方向正确，但需要统一统计口径：

- `decode/transcode_ms`：资源转码耗时。
- `upload_ms`：GPU 上传耗时。
- `render_cpu_ms`：提交绘制命令耗时。
- `gpu_frame_ms`：GPU 完成耗时，如设备支持 timer query。
- `late_frame_ratio`：超过帧预算的比例。
- `cache_hit_ratio`：TextureCache 命中率。
- `memory_mb`：Java heap、native heap、GPU texture budget 估算。

## 3. Spec Coding 工作流

每个任务都应包含以下固定结构：

```text
Spec ID:
目标:
范围内:
范围外:
输入/输出:
涉及文件:
接口契约:
实现步骤:
验收标准:
测试命令:
风险与降级:
```

执行顺序：

1. 先写/更新 spec。
2. 再创建最小接口与测试。
3. 再实现内部逻辑。
4. 最后补指标日志与文档。

## 4. 推荐交付波次

### Wave 0 — 设计冻结与工程约束

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S0.1 | 工程技术栈锁定 | Gradle/NDK/CMake/包名决策记录 | 文档明确版本与目录 |
| S0.2 | Native API 契约 | JNI API 表、错误码、生命周期状态机 | Kotlin/C++ 接口无歧义 |
| S0.3 | 资源 Schema | clip/timeline/material schema | 示例资源可被 schema 校验 |

### Wave 1 — 最小可运行骨架

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S1.1 | Android app skeleton | Activity、SurfaceView、Renderer | App 可启动并创建 GL context |
| S1.2 | Native runtime skeleton | RuntimeCore、JNI bridge、CMake | Kotlin 可调用 native init/release |
| S1.3 | Diagnostics skeleton | 日志、FrameStats 数据结构 | 可输出基础帧统计 |

### Wave 2 — MVP 渲染闭环

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S2.1 | ShaderProgram | shader 编译、uniform 设置 | 编译失败有明确错误日志 |
| S2.2 | GLRenderer basic draw | 全屏 quad、viewport、clear/draw | 单纹理可绘制到屏幕 |
| S2.3 | Render loop | VSync tick 到 drawFrame | 无 `Thread.sleep` 驱动渲染 |

### Wave 3 — Timeline 与 Layer Model

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S3.1 | Timeline data model | Clip、Track、Frame、Layer | 时间戳可解析到 frame index |
| S3.2 | Layer composition | z-order、blend mode、visibility | 背景/角色/Overlay 顺序正确 |
| S3.3 | FBO manager | framebuffer 创建/绑定/释放 | resize 后资源正确重建 |

### Wave 4 — Resource 与 Texture Cache

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S4.1 | TextureHandle & ownership | RAII、引用计数、释放队列 | 无重复释放/悬空句柄 |
| S4.2 | Async load/upload queue | IO worker + GL upload task | 渲染线程不做磁盘 IO |
| S4.3 | LRU/watermark | GPU 纹理预算、淘汰策略 | 超水位可回收非活跃纹理 |

### Wave 5 — Audio Master Clock

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S5.1 | AudioPlayer | 播放、暂停、seek | 可返回单调播放时间 |
| S5.2 | AudioSync | 音频时间驱动 Timeline | drift 被限制在目标阈值内 |
| S5.3 | Playback state machine | idle/loading/playing/paused/released | 状态切换可预测且可测试 |

### Wave 6 — KTX2/BasisU 与 Offline Builder

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S6.1 | KTX2 loader adapter | KTX2 header/metadata parse | 错误资源有可诊断日志 |
| S6.2 | Basis transcoder | BasisU 到设备 GPU 格式 | 支持 fallback 格式选择 |
| S6.3 | petcat-builder MVP | frame extract、atlas、timeline、material | 示例 clip 可被 runtime 加载 |

### Wave 7 — Shader FX 与画质

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S7.1 | Feather/Alpha Blur | 边缘柔化 shader | 角色边缘无明显割裂 |
| S7.2 | Shadow | 阴影 pass 与参数 | 阴影跟随角色位置变化 |
| S7.3 | Color/LUT/Bloom/RimLight | 高级 FX pass | 可按 material 开关 |

### Wave 8 — 性能矩阵与降级

| Spec | 目标 | 主要产出 | 验收 |
| --- | --- | --- | --- |
| S8.1 | Benchmark harness | 自动播放、统计 P50/P95/Max | 输出可比较报告 |
| S8.2 | Device capability | GL extension、texture format 探测 | 自动选择最佳格式 |
| S8.3 | Fallback strategy | 低分辨率、WEBP/Canvas/禁用 FX | 低端设备稳定播放 |

## 5. 首批建议创建的 Spec 文件

建议后续在 `docs/specs/` 下创建以下文件：

```text
docs/specs/S0.1-engine-decisions.md
docs/specs/S0.2-native-api-contract.md
docs/specs/S0.3-asset-schema.md
docs/specs/S1.1-android-app-skeleton.md
docs/specs/S1.2-native-runtime-skeleton.md
docs/specs/S1.3-diagnostics-skeleton.md
```

## 6. 首个编码任务建议

不建议首个任务直接做 BasisU 或复杂 Shader。首个实现任务建议是：

```text
S1.1 + S1.2：Android/NDK 最小工程骨架 + NativeBridge init/release + 空 GL context。
```

原因：

- 能最早暴露 Gradle、NDK、CMake、设备兼容问题。
- 后续所有渲染、纹理、音频模块都依赖这条链路。
- 可以尽早建立测试与 CI 命令。

