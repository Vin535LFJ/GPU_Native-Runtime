# PetCat GPU-Native Runtime SDK — 综合设计方案

## 1. 项目定位

* **产品形态**：面向业务 App 直接集成的 **Android SDK**。
* **内部核心**：SDK 内部包含一个轻量级 **GPU-Native Runtime Engine**，负责 Timeline、资源、音频同步和 OpenGL ES 渲染。
* **目标能力**：轻量级 2D/多层帧动画与 AV composition，支持 720P/1080P，并为 4K 留出资源与渲染扩展空间。
* **非目标**：不做完整游戏引擎，不引入 ECS/物理/复杂编辑器；业务只应感知 SDK API，而不是直接操作底层 runtime。

### 1.1 核心特点

* **首版直接 GPU 渲染**：从 0 构建时第一版就使用 OpenGL ES 3.0 + GLSL shader，不再规划 Canvas 渲染路径。
* **高效纹理流**：长期目标为 KTX2 + Basis Universal，减少播放时 CPU 解码和重复上传。
* **多层合成**：支持背景、角色、阴影、FX、Overlay 分层实时组合。
* **时间驱动**：Audio 作为 master clock 驱动 Timeline，保证 AV 同步。
* **业务友好**：通过 `PetCatPlayerView` / `PetCatPlayer` / `PetCatClip` 等 SDK 门面给业务调用。

---

## 2. SDK / Runtime / Engine 的关系

| 名称 | 对谁可见 | 责任 |
| --- | --- | --- |
| **PetCat SDK** | 业务 App 可见 | 提供依赖包、View、Player、生命周期、回调、错误码、配置项。 |
| **Runtime** | SDK 内部为主，少量诊断 API 可见 | 管理播放状态、Timeline、资源缓存、Audio master clock、渲染调度。 |
| **Engine Core** | Native/C++ 内部实现 | OpenGL ES 渲染、TextureCache、Shader、FBO、KTX2/BasisU 上传、性能统计。 |
| **petcat-builder** | 资产生产链路可见 | 离线生成 atlas、timeline、material、audio 和 clip metadata。 |

> 结论：对外交付物应叫 **SDK**；Runtime/Engine 是 SDK 内部架构分层。后续 Spec Coding 必须同时描述“业务 SDK API”和“内部 Runtime/Native API”，避免 README、任务清单和 Spec 模板脱节。

---

## 3. 高级架构设计

### 3.1 核心原则

| 组件 | 责任 |
| --- | --- |
| SDK Facade | 业务调用、生命周期绑定、错误回调、播放控制、配置管理。 |
| Kotlin Runtime | Timeline 调度、Audio clock 采样、资源请求编排、JNI bridge。 |
| C++ Engine Core | OpenGL ES 渲染、纹理生命周期、Shader/FBO、性能指标。 |
| GPU | 纹理组合、缩放、Blend/FX、边缘羽化、阴影、LUT/Bloom 等视觉计算。 |
| Offline Tool | 资源预处理，避免 runtime 播放时做重型格式转换。 |

> 核心思想：**Everything becomes GPU texture**。首版实现即围绕 GPU-native pipeline 设计，不把 Canvas 作为计划内渲染后端。

### 3.2 高层模块架构

```text
Business App
   │
   ▼
PetCat Android SDK
   ├─ PetCatPlayerView / PetCatPlayer
   ├─ Lifecycle / Callback / Config
   └─ Kotlin Runtime Facade
          │
          ▼
      NativeBridge (JNI)
          │
          ▼
C++ Runtime Engine Core
   ├─ Timeline Runtime
   ├─ Resource System / TextureCache
   ├─ Audio Sync
   └─ GPU Render Runtime (OpenGL ES 3.0 + GLSL)
          │
          ├─ Shader FX
          ├─ Layer Composition
          └─ Texture Streaming / Cache

Offline Asset Tool: petcat-builder
   └─ clip.json + timeline.anim + atlas.ktx2 + material.json + audio
```

---

## 4. 技术路线

### 4.1 Wave 0：SDK 契约与工程骨架

* **目标**：先固定 SDK 对外 API、Native API、资源 schema、线程模型和验收指标。
* **措施**：
  * 定义 `PetCatPlayerView`、`PetCatPlayer`、`PetCatClip`、`PetCatConfig`、回调和错误码。
  * 定义 Kotlin → JNI → C++ RuntimeCore 的生命周期 API。
  * 建立 Android SDK module、sample app、CMake/NDK 空工程。
* **指标**：业务可通过 SDK facade 创建播放器；native runtime 可 init/release；GL context 可创建。

### 4.2 Wave 1：首版 OpenGL ES MVP

* **目标**：直接实现 OpenGL ES 渲染闭环，而不是先做 Canvas 再替换。
* **措施**：
  * GLSurfaceView 或等价 EGL 宿主。
  * ShaderProgram、全屏 quad、viewport、clear/draw。
  * VSync 对齐渲染，禁止 `Thread.sleep` 驱动渲染。
  * 单纹理/单层 draw + FrameStats。
* **指标**：单层纹理可稳定显示，渲染 P50/P95 可记录。

### 4.3 Wave 2：Timeline、Layer 与 Resource 闭环

* **目标**：建立真实 runtime 播放模型。
* **措施**：
  * Timeline tick → frame index / layer visibility / shader uniform。
  * 背景、角色、阴影、Overlay 的 z-order 与 blend mode。
  * TextureHandle、异步上传、LRU、水位和引用计数。
* **指标**：多层顺序正确；不在每帧触发大纹理重建；缓存命中率可统计。

### 4.4 Wave 3：KTX2/BasisU 与 petcat-builder

* **目标**：把资源生产和 runtime 加载闭环打通。
* **措施**：
  * 定义并校验 `clip.json`、`timeline.anim`、`material.json`。
  * 接入 KTX2/BasisU loader/transcoder。
  * petcat-builder MVP 生成 runtime 可加载的示例 clip。
* **指标**：示例 clip 可由 builder 生成，并由 SDK 播放。

### 4.5 Wave 4：高级 FX、性能矩阵与降级策略

* **目标**：提升画质并覆盖设备差异。
* **措施**：
  * Feather、Alpha Blur、Shadow 优先；Bloom/LUT/RimLight 后置。
  * 设备能力探测：GL extension、最大纹理尺寸、压缩纹理格式。
  * 降级策略优先采用低分辨率资源、禁用高阶 FX、减少层数或返回明确不支持错误。
* **指标**：输出 P50/P95/Max、late frame ratio、cache hit ratio、内存水位报告。

---

## 5. 模块设计

| 模块 | 功能 | 核心类/文件 | 说明 |
| --- | --- | --- | --- |
| SDK API | 业务集成入口 | PetCatPlayerView.kt / PetCatPlayer.kt / PetCatConfig.kt | 业务只依赖这一层。 |
| Timeline | 时间驱动，帧切换，音频同步 | TimelinePlayer.kt / timeline_player.cpp | Audio master clock。 |
| Resource | 纹理管理，异步加载，缓存 | AssetManager.kt / TextureCache.cpp / BasisTranscoder.cpp | LRU、预取、水位、GPU 上传。 |
| GPU Render | Shader 渲染，Layer 合成 | GLRenderer.kt / gl_renderer.cpp / shader_program.cpp | 首版直接 OpenGL ES。 |
| Shader | Visual FX | *.frag / shader/*.h | Feather、Alpha Blur、Shadow、Color Match、Bloom、LUT。 |
| Audio | 播放与同步 | AudioPlayer.kt / AudioSync.kt | 提供 master clock。 |
| Bridge | JNI / NDK | NativeBridge.kt / native_bridge.cpp | Kotlin Runtime ↔ C++17。 |
| Offline Tool | 资源构建 | petcat-builder | Atlas Packing、KTX2/BasisU、Timeline、Material metadata。 |

---

## 6. 推荐项目文件夹结构

```text
petcat-sdk/
├── src/main/java/com/petcat/sdk/
│   ├── PetCatPlayerView.kt
│   ├── PetCatPlayer.kt
│   ├── PetCatClip.kt
│   ├── PetCatConfig.kt
│   ├── PetCatCallback.kt
│   ├── runtime/
│   ├── timeline/
│   ├── audio/
│   ├── resource/
│   └── bridge/
├── src/main/cpp/
│   ├── core/
│   │   ├── runtime/
│   │   ├── timeline/
│   │   ├── render/
│   │   ├── shader/
│   │   ├── texture/
│   │   ├── material/
│   │   ├── memory/
│   │   └── threading/
│   ├── renderer/
│   ├── texture/
│   ├── timeline/
│   ├── audio/
│   ├── shader/
│   ├── third_party/
│   └── CMakeLists.txt
├── src/main/assets/shaders/
└── build.gradle

sample-app/
└── src/main/java/com/petcat/sample/

tools/petcat-builder/

docs/
├── SPEC_CODING_PLAN.md
├── SPEC_TEMPLATE.md
└── specs/
```

---


## 7. Spec Coding 规划入口

本项目后续实现采用 **Spec Coding**：先固定业务 SDK API、Native API、资源 schema、线程模型和验收指标，再按小步任务逐个实现。详细实施波次、首批 Spec 文件建议、任务模板见：


- `docs/SPEC_CODING_PLAN.md`：从设计文档到可执行 Spec 的总规划。
- `docs/SPEC_TEMPLATE.md`：每个 Spec 任务必须包含的字段模板。
- `docs/specs/S0.4-sdk-product-contract.md`：业务可直接调用的 SDK 契约。
- `AI Coding 任务清单.md`：模块级任务清单与优先级。

当前建议的首个编码任务是 **Android SDK module + sample-app + NativeBridge init/release + 空 OpenGL ES context**，用于尽早验证 SDK 集成方式、Gradle、NDK、CMake、JNI 与设备 GL 环境。

---


## 8. 实现计划（与 Spec Coding 对齐）

| 阶段 | 核心动作 | 负责人 | 时间周期 |
| --- | --- | --- | --- |
| S0 | SDK/API/Schema/线程模型契约冻结 | AI Coding | 1 周 |
| S1 | Android SDK module + sample-app + NativeBridge + 空 GL context | AI Coding | 1 周 |
| S2 | OpenGL ES MVP：ShaderProgram、单纹理 draw、VSync render loop | AI Coding | 2 周 |
| S3 | Timeline + Layer composition + FBO | AI Coding | 2 周 |
| S4 | TextureCache + 异步上传 + LRU/水位 | AI Coding | 2 周 |
| S5 | Audio master clock + Timeline sync | AI Coding | 1~2 周 |
| S6 | KTX2/BasisU + petcat-builder MVP | AI Coding | 2~3 周 |
| S7 | Feather/Alpha Blur/Shadow 与高级 FX | AI Coding | 2~3 周 |
| S8 | 1080P/4K 性能矩阵、设备能力探测与降级策略 | AI Coding | 2~3 周 |


> 每阶段完成后进行验收测试，指标包括 render P50/P95、upload/transcode P50/P95、late frame ratio、cache hit ratio、内存水位。

---


## 9. 核心风险与降级策略


1. **GPU 兼容性**：首版目标是 OpenGL ES 3.0；需要检测 GL version、extension、最大纹理尺寸与压缩纹理格式。
2. **内存压力**：GPU 纹理 + native 资源需 LRU 回收、引用计数和水位控制。
3. **工具链迁移**：KTX2/BasisU 与 petcat-builder 构建流程需单独验证。
4. **设备降级**：不把 Canvas 作为计划内 backend；优先使用低分辨率资源、减少层数、禁用高阶 FX、选择 ETC2 等兼容纹理格式，必要时向业务返回明确错误码。

---

## 10. 总结

* 对外交付物是 **业务可直接集成的 Android SDK**。
* SDK 内部包含轻量级 GPU-Native Runtime Engine。
* 首版从 0 开始直接实现 OpenGL ES 渲染闭环，不再规划 Canvas 替换路线。
* 后续所有任务必须按 Spec Coding 固定业务 API、内部 API、资源 schema 和验收指标。
