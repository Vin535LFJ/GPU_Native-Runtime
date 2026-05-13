

# PetCat GPU-Native Runtime — AI Coding 任务清单

## 1️⃣ 核心基础模块

| 任务 | 模块     | 功能描述              | 输入                        | 输出                      | 优先级 | 验收标准                                         |
| ---- | -------- | --------------------- | --------------------------- | ------------------------- | ------ | ------------------------------------------------ |
| T1   | Resource | AssetManager 基础类   | Asset path, KTX2/Basis 文件 | GPU-ready texture handles | 高     | 能异步加载 KTX2/Basis 到 GPU，支持回调通知       |
| T2   | Resource | TextureCache 管理     | Texture ID, usage           | LRU 管理，内存回收        | 高     | 支持 async upload, LRU eviction, prefetch 队列   |
| T3   | Bridge   | NativeBridge JNI 封装 | Kotlin function call        | C++ 函数调用              | 高     | Kotlin 调用 C++ Texture/Shader/Timeline API 成功 |
| T4   | Memory   | RAII 内存管理         | Texture/Shader对象          | 自动管理内存生命周期      | 中     | 不发生内存泄漏，支持 reference count             |

------

## 2️⃣ 渲染模块

| 任务 | 模块   | 功能描述              | 输入                         | 输出                 | 优先级 | 验收标准                                              |
| ---- | ------ | --------------------- | ---------------------------- | -------------------- | ------ | ----------------------------------------------------- |
| T5   | Render | GLRenderer 基础初始化 | GLSurfaceView / width/height | OpenGL ES context    | 高     | GL context 初始化成功，支持多线程安全                 |
| T6   | Render | Framebuffer 管理      | width, height                | framebuffer objects  | 高     | 可以创建、绑定、销毁 framebuffers，支持多层合成       |
| T7   | Render | Texture Upload        | KTX2/Basis texture           | GPU texture handle   | 高     | 上传一次，支持多帧复用，不触发 per-frame glTexImage2D |
| T8   | Render | Layer Composition     | 背景/角色/阴影/FX/Overlay    | 合成最终 framebuffer | 高     | 多层正确叠加，支持透明和 Blend 模式                   |
| T9   | Render | Render Loop           | Timeline tick                | Frame draw           | 高     | 渲染 P50 <5ms，VSync 对齐，晚帧 <5%                   |

------

## 3️⃣ Shader / FX 模块

| 任务 | 模块   | 功能描述               | 输入                           | 输出                       | 优先级 | 验收标准                    |
| ---- | ------ | ---------------------- | ------------------------------ | -------------------------- | ------ | --------------------------- |
| T10  | Shader | Feather Shader         | Character texture + background | Soft edge blend            | 高     | 边缘自然融合，无明显割裂    |
| T11  | Shader | Alpha Blur Shader      | Texture                        | Soft alpha edges           | 高     | 抠图角色边缘柔化            |
| T12  | Shader | Shadow Shader          | Character + light info         | Shadow overlay             | 高     | 阴影实时随角色位置变化      |
| T13  | Shader | Color Match Shader     | Background + Character         | Tone matched output        | 中     | 背景色彩与角色协调          |
| T14  | Shader | Bloom / LUT / RimLight | Texture                        | FX enhanced output         | 中     | Phase 2 高级特效正确渲染    |
| T15  | Shader | Future AI / Video FX   | Video frame / AI parameters    | Super resolution / Denoise | 低     | 可扩展为 AI shader pipeline |

------

## 4️⃣ Timeline & Audio 模块

| 任务 | 模块     | 功能描述                      | 输入                            | 输出                    | 优先级 | 验收标准                      |
| ---- | -------- | ----------------------------- | ------------------------------- | ----------------------- | ------ | ----------------------------- |
| T16  | Timeline | TimelinePlayer                | `.anim` timeline                | Frame index / FX params | 高     | 每帧按时间更新，支持多轨道    |
| T17  | Timeline | Scheduler                     | Choreographer.postFrameCallback | Tick callback           | 高     | 对齐 VSync，避免 Thread.sleep |
| T18  | Audio    | AudioPlayer                   | `.audio` file                   | Playback + master clock | 高     | 音频播放与 timeline 同步      |
| T19  | Audio    | AudioSync                     | AudioTime                       | Timeline update         | 高     | 避免动画 drift / desync       |
| T20  | Timeline | FX Param Update               | Shader params / timeline        | Shader uniforms         | 中     | 每帧 Shader 参数正确更新      |
| T21  | Timeline | Layer Visibility / Transition | Timeline track                  | Layer show/hide / fade  | 中     | 支持淡入淡出、隐藏层控制      |

------

## 5️⃣ 高级功能模块

| 任务 | 模块                     | 功能描述                      | 输入                                     | 输出             | 优先级 | 验收标准                              |
| ---- | ------------------------ | ----------------------------- | ---------------------------------------- | ---------------- | ------ | ------------------------------------- |
| T22  | Background Composition   | 实时背景 + 角色合成           | Background texture + Character           | Composited frame | 中     | 无明显割裂感，边缘 feather            |
| T23  | Multi-Resolution Support | 720P/1080P/4K 自动适配        | Timeline + GPU memory                    | Resized textures | 中     | 运行稳定，无 OOM，渲染 P50 <目标      |
| T24  | BasisU Integration       | Basis Transcoder + GPU Upload | .basis file                              | GPU texture      | 高     | 零 CPU 解码，上传成功，支持 fallback  |
| T25  | Offline Tool             | Petcat-builder                | 视频/Atlas → Basis + timeline + material | Asset pipeline   | 中     | 构建正确资源，兼容 runtime            |
| T26  | Debug / Profiling        | Frame metrics & GPU usage     | Runtime                                  | Logs / stats     | 低     | 可显示解码/渲染 P50/P95/Max，晚帧占比 |

------

## 6️⃣ 接口标准化

### 6.1 Kotlin → C++ 接口示例

```kotlin
// Kotlin
class NativeBridge {
    external fun initRuntime(width: Int, height: Int)
    external fun loadTexture(path: String): Int
    external fun drawFrame(timestamp: Long)
    external fun setShaderParam(shaderId: Int, paramName: String, value: Float)
}
// C++17
class RuntimeCore {
public:
    void init(int width, int height);
    int uploadTexture(const std::string& path);
    void renderFrame(uint64_t timestamp);
    void setShaderUniform(int shaderId, const std::string& name, float value);
};
```

------

## 7️⃣ AI Coding 指导原则

1. 每个任务可拆成**单独 Spec Coding 指令**。
2. 先实现核心链路（T1~T9、T16~T19）。
3. Shader FX 模块逐步添加（T10~T15）。
4. 高级功能（T22~T26）作为迭代。
5. 所有渲染/纹理操作需支持 GPU 异步、LRU、三缓冲。
6. 每阶段完成后输出指标日志（解码 P50、渲染 P50、晚帧占比）供验收。

