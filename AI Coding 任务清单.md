# PetCat GPU-Native Runtime SDK — AI Coding 任务清单

> 目标：构建一个业务 App 可直接集成的 Android SDK。SDK 内部包含 GPU-Native Runtime Engine，但业务侧只通过 SDK facade 使用播放器能力。

## 0️⃣ 产品与架构边界

| 结论 | 说明 |
| ---- | ---- |
| 对外交付物 | Android SDK，而不是只交付内部 engine。 |
| 内部架构 | SDK facade → Kotlin runtime → JNI → C++ RuntimeCore → OpenGL ES renderer。 |
| 渲染路线 | 首版从 0 直接使用 OpenGL ES 3.0，不做 Canvas MVP，也不规划“Canvas → OpenGL ES 替换”阶段。 |
| 降级策略 | 优先降分辨率、禁用高阶 FX、减少层数、选择兼容压缩格式或返回明确错误码；Canvas 不作为计划内 backend。 |
| Spec 要求 | 每个任务必须先写/更新 Spec，再实现接口、测试和代码。 |

---

## 1️⃣ SDK 对外接口模块

| 任务 | 模块 | 功能描述 | 输入 | 输出 | 优先级 | 验收标准 |
| ---- | ---- | -------- | ---- | ---- | ------ | -------- |
| T0   | SDK API | PetCatPlayerView | XML/代码创建 View | 可承载渲染 Surface 的业务 View | 高 | 业务可在 Activity/Fragment 中直接使用 |
| T0.1 | SDK API | PetCatPlayer | clip、config、lifecycle | play/pause/seek/release API | 高 | 业务不直接接触 JNI/C++ |
| T0.2 | SDK API | PetCatCallback/ErrorCode | runtime 状态和错误 | onReady/onPlay/onError/onStats | 高 | 错误可诊断，状态可观测 |
| T0.3 | SDK API | PetCatConfig | 分辨率、FX、缓存、日志配置 | RuntimeConfig | 中 | 可控制质量档位和诊断级别 |

---

## 2️⃣ 核心基础模块

| 任务 | 模块     | 功能描述              | 输入                        | 输出                      | 优先级 | 验收标准                                         |
| ---- | -------- | --------------------- | --------------------------- | ------------------------- | ------ | ------------------------------------------------ |
| T1   | Resource | AssetManager 基础类   | Asset path, KTX2/Basis 文件 | GPU-ready texture handles | 高     | 能异步加载 KTX2/Basis 到 GPU，支持回调通知       |
| T2   | Resource | TextureCache 管理     | Texture ID, usage           | LRU 管理，内存回收        | 高     | 支持 async upload, LRU eviction, prefetch 队列   |
| T3   | Bridge   | NativeBridge JNI 封装 | Kotlin runtime call         | C++ RuntimeCore 调用      | 高     | Kotlin 调用 C++ Texture/Shader/Timeline API 成功 |
| T4   | Memory   | RAII 内存管理         | Texture/Shader 对象         | 自动管理内存生命周期      | 中     | 不发生内存泄漏，支持 reference count             |

------

## 3️⃣ 渲染模块（首版直接 OpenGL ES）

| 任务 | 模块   | 功能描述              | 输入                         | 输出                 | 优先级 | 验收标准                                              |
| ---- | ------ | --------------------- | ---------------------------- | -------------------- | ------ | ----------------------------------------------------- |
| T5   | Render | GLRenderer 基础初始化 | GLSurfaceView/EGL + width/height | OpenGL ES context | 高     | GL context 初始化成功，支持生命周期安全               |
| T6   | Render | Framebuffer 管理      | width, height                | framebuffer objects  | 高     | 可以创建、绑定、销毁 framebuffers，支持多层合成       |
| T7   | Render | Texture Upload        | KTX2/Basis texture           | GPU texture handle   | 高     | 上传一次，支持多帧复用，不触发 per-frame glTexImage2D |
| T8   | Render | Layer Composition     | 背景/角色/阴影/FX/Overlay    | 合成最终 framebuffer | 高     | 多层正确叠加，支持透明和 Blend 模式                   |
| T9   | Render | Render Loop           | Timeline tick                | Frame draw           | 高     | 渲染 P50 <5ms，VSync 对齐，晚帧 <5%                   |

------

## 4️⃣ Shader / FX 模块

| 任务 | 模块   | 功能描述               | 输入                           | 输出                       | 优先级 | 验收标准                    |
| ---- | ------ | ---------------------- | ------------------------------ | -------------------------- | ------ | --------------------------- |
| T10  | Shader | Feather Shader         | Character texture + background | Soft edge blend            | 高     | 边缘自然融合，无明显割裂    |
| T11  | Shader | Alpha Blur Shader      | Texture                        | Soft alpha edges           | 高     | 抠图角色边缘柔化            |
| T12  | Shader | Shadow Shader          | Character + light info         | Shadow overlay             | 高     | 阴影实时随角色位置变化      |
| T13  | Shader | Color Match Shader     | Background + Character         | Tone matched output        | 中     | 背景色彩与角色协调          |
| T14  | Shader | Bloom / LUT / RimLight | Texture                        | FX enhanced output         | 中     | Phase 2 高级特效正确渲染    |
| T15  | Shader | Future AI / Video FX   | Video frame / AI parameters    | Super resolution / Denoise | 低     | 可扩展为 AI shader pipeline |

------

## 5️⃣ Timeline & Audio 模块

| 任务 | 模块     | 功能描述                      | 输入                            | 输出                    | 优先级 | 验收标准                      |
| ---- | -------- | ----------------------------- | ------------------------------- | ----------------------- | ------ | ----------------------------- |
| T16  | Timeline | TimelinePlayer                | `.anim` timeline                | Frame index / FX params | 高     | 每帧按时间更新，支持多轨道    |
| T17  | Timeline | Scheduler                     | Choreographer.postFrameCallback | Tick callback           | 高     | 对齐 VSync，避免 Thread.sleep |
| T18  | Audio    | AudioPlayer                   | `.audio` file                   | Playback + master clock | 高     | 音频播放与 timeline 同步      |
| T19  | Audio    | AudioSync                     | AudioTime                       | Timeline update         | 高     | 避免动画 drift / desync       |
| T20  | Timeline | FX Param Update               | Shader params / timeline        | Shader uniforms         | 中     | 每帧 Shader 参数正确更新      |
| T21  | Timeline | Layer Visibility / Transition | Timeline track                  | Layer show/hide / fade  | 中     | 支持淡入淡出、隐藏层控制      |

------

## 6️⃣ 高级功能模块

| 任务 | 模块                     | 功能描述                      | 输入                                     | 输出             | 优先级 | 验收标准                              |
| ---- | ------------------------ | ----------------------------- | ---------------------------------------- | ---------------- | ------ | ------------------------------------- |
| T22  | Background Composition   | 实时背景 + 角色合成           | Background texture + Character           | Composited frame | 中     | 无明显割裂感，边缘 feather            |
| T23  | Multi-Resolution Support | 720P/1080P/4K 自动适配        | Timeline + GPU memory                    | Resized textures | 中     | 运行稳定，无 OOM，渲染 P50 <目标      |
| T24  | BasisU Integration       | Basis Transcoder + GPU Upload | .basis/.ktx2 file                        | GPU texture      | 高     | 减少 CPU 解码，上传成功，支持格式选择 |
| T25  | Offline Tool             | Petcat-builder                | 视频/Atlas → KTX2/Basis + timeline + material | Asset pipeline | 中 | 构建正确资源，兼容 runtime            |
| T26  | Debug / Profiling        | Frame metrics & GPU usage     | Runtime                                  | Logs / stats     | 低     | 可显示 P50/P95/Max、晚帧、缓存、内存  |

------

## 7️⃣ 接口标准化

### 7.1 业务 SDK API 草案

```kotlin
class PetCatPlayerView : FrameLayout {
    fun bind(player: PetCatPlayer)
}

class PetCatPlayer {
    fun load(clip: PetCatClip, config: PetCatConfig = PetCatConfig())
    fun play()
    fun pause()
    fun seekTo(positionUs: Long)
    fun release()
    fun setCallback(callback: PetCatCallback?)
}
```

### 7.2 Kotlin Runtime → C++ 接口草案

```kotlin
class NativeBridge {
    external fun initRuntime(width: Int, height: Int): Long
    external fun resize(runtime: Long, width: Int, height: Int)
    external fun loadClip(runtime: Long, clipPath: String): Int
    external fun play(runtime: Long, clipId: Int)
    external fun pause(runtime: Long)
    external fun seekTo(runtime: Long, positionUs: Long)
    external fun drawFrame(runtime: Long, vsyncTimeNs: Long)
    external fun releaseRuntime(runtime: Long)
}
```

```cpp
class RuntimeCore {
public:
    bool init(int width, int height);
    void resize(int width, int height);
    int loadClip(const std::string& clipPath);
    void play(int clipId);
    void pause();
    void seekTo(uint64_t positionUs);
    void drawFrame(uint64_t vsyncTimeNs);
    void release();
};
```

------

## 8️⃣ AI Coding 指导原则

1. 每个任务必须拆成**单独 Spec Coding 指令**。
2. 先实现 SDK 门面和核心链路（T0~T9、T16~T19）。
3. 首版渲染直接 OpenGL ES，不实现 Canvas backend。
4. Shader FX 模块逐步添加（T10~T15）。
5. 高级功能（T22~T26）作为迭代。
6. 所有渲染/纹理操作需支持 GPU 异步、LRU、三缓冲。
7. 每阶段完成后输出指标日志（render/upload/transcode P50/P95、晚帧占比、cache hit、memory watermark）供验收。

---

## 9️⃣ Spec Coding 波次规划

> 本节把 T0~T26 任务转换为更适合 AI Coding 的交付波次。每个波次都应先写 Spec，再实现接口、测试与代码。详细模板见 `docs/SPEC_TEMPLATE.md`。

| 波次 | 目标 | 覆盖任务 | 交付物 | 通过标准 |
| ---- | ---- | -------- | ------ | -------- |
| Wave 0 | SDK/API/Schema 契约冻结 | T0~T4/T16/T24 | SDK API、Native API、资源 schema、状态机、错误码 | README、任务清单、Spec 模板与 S0 specs 对齐 |
| Wave 1 | 最小 SDK 工程骨架 | T0/T3/T5 | petcat-sdk、sample-app、GLSurfaceView/EGL、CMake、JNI init/release | 业务 sample 可接入 SDK，GL context 创建成功 |
| Wave 2 | MVP OpenGL ES 渲染闭环 | T5/T7/T9 | ShaderProgram、单纹理 draw、VSync render loop | 单层纹理可稳定显示，渲染指标可记录 |
| Wave 3 | Timeline + Layer | T8/T16/T17/T20/T21 | Timeline data model、layer z-order、blend mode | 时间戳可映射帧，多层顺序正确 |
| Wave 4 | Resource + Cache | T1/T2/T4/T7 | TextureHandle、异步上传、LRU、水位 | 不在每帧触发 `glTexImage2D`，可回收纹理 |
| Wave 5 | Audio 主时钟 | T18/T19 | AudioPlayer、AudioSync、播放状态机 | Timeline 由音频时间驱动，无明显 drift |
| Wave 6 | KTX2/BasisU + Builder | T24/T25 | KTX2 loader、Basis transcoder、petcat-builder MVP | 示例 clip 可完整构建并加载 |
| Wave 7 | Shader FX | T10~T15/T22 | Feather、Alpha Blur、Shadow、Color/LUT/Bloom | FX 可通过 material 开关并输出正确画面 |
| Wave 8 | 性能与设备降级 | T23/T26 | Benchmark、设备能力探测、质量档位策略 | 输出 P50/P95/late frame/cache/memory 报告 |

## 🔟 单任务 Spec 模板要求

每个任务进入实现前，必须包含：

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

## 1️⃣1️⃣ 首批建议拆分任务

1. `S0.1-engine-decisions`：锁定 Android/Gradle/NDK/CMake/Kotlin 版本与包名。
2. `S0.2-native-api-contract`：定义 `init/resize/loadClip/play/pause/seek/drawFrame/release/getFrameStats`。
3. `S0.3-asset-schema`：定义 `clip.json`、`timeline.anim`、`material.json` 的字段和版本。
4. `S0.4-sdk-product-contract`：定义业务可直接调用的 SDK facade、View、Player、Callback、ErrorCode。
5. `S1.1-android-sdk-skeleton`：建立 Android SDK module、sample-app、PetCatPlayerView。
6. `S1.2-native-runtime-skeleton`：建立 C++ RuntimeCore、JNI bridge、CMake。
7. `S1.3-diagnostics-skeleton`：建立 FrameStats 与统一日志。
