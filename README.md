# PetCat GPU-Native Runtime — 综合设计方案

## 1. 项目定位

* **目标**：轻量级 2D/多层帧动画 & AV composition runtime，支持 720P/1080P/4K。
* **核心特点**：

  * **GPU 原生渲染**：OpenGL ES 3.0 + GLSL shader，释放 GPU 并行计算。
  * **高效纹理流**：Basis Universal + KTX2，实现零 CPU 解码、直接 GPU 加载。
  * **多层合成**：支持背景、角色、阴影、FX、Overlay 分层实时组合。
  * **时间驱动**：Audio 作为主时钟驱动 Timeline，保证 AV 同步。
  * **轻量级**：不引入完整游戏引擎特性（ECS/物理），聚焦播放效率。

---

## 2. 高级架构设计

### 2.1 核心原则

| 组件       | 责任                                                        |
| -------- | --------------------------------------------------------- |
| CPU      | 调度、Timeline 更新、资源管理、异步解码、同步控制                             |
| GPU      | 渲染、纹理组合、缩放、Blend/FX、Shader 特效、背景抠图合成                      |
| Shader   | 所有视觉逻辑（Alpha Blur、Feather、Shadow、Color Match、Bloom、LUT 等） |
| Timeline | 时间驱动、帧切换、音频同步、FX 参数更新                                     |

> 核心思想：**Everything becomes GPU texture**，避免 CPU bitmap 转 GPU 冗余拷贝。

### 2.2 高层模块架构

```
            ┌─────────────────────┐
            │ Offline Asset Tool  │
            │  petcat-builder     │
            └─────────┬──────────┘
                      │
   ┌──────────────────┼──────────────────┐
   ▼                  ▼                  ▼
timeline.anim     atlas.ktx2      material.json
                      │
                      ▼
            ┌─────────────────────┐
            │ Android Runtime     │
            └─────────┬──────────┘
                      │
  ┌─────────────┬─────────────┬─────────────┐
  ▼             ▼             ▼
Timeline     Resource System  Audio System
Scheduler   (Texture Cache)  Audio Sync
  │             │
  ▼             ▼
GPU Render Runtime (OpenGL ES 3.0 + GLSL Shader)
  │
  ├─ Shader FX
  ├─ Layer Composition
  └─ Texture Streaming / Cache
```

---

## 3. 技术路线

### 3.1 短期优化（1~2 周）

* **目标**：消除同步解码和 GC 卡顿。
* **措施**：

  * 多线程预加载 (2~3 线程)。
  * Bitmap 池 + inBitmap 复用。
  * 提升预加载线程优先级。
* **指标**：720P 2×2 晚帧 < 20%。

### 3.2 中期优化（1~2 月）

* **目标**：GPU 全链路重构。
* **措施**：

  * OpenGL ES 替换 Canvas 渲染。
  * Shader 支持多层合成和特效。
  * VSync 对齐渲染 (Choreographer)。
  * 异步解码 + 三缓冲页面池。
* **指标**：720P 2×2 渲染 P50 ~5ms，晚帧 <5%。

### 3.3 长期优化（3~6 月）

* **目标**：零 CPU 解码，高分辨率支持。
* **措施**：

  * Basis Universal + KTX2 替代 WEBP。
  * 可选 MediaCodec 硬解（H.264/H.265）。
  * 高级 Shader FX：Bloom、LUT、RimLight、边缘抠图合成。
* **指标**：1080P 2×2 解码/渲染 P50 <8ms，晚帧 <1%，支持 4K。

---

## 4. 模块设计

| 模块           | 功能                 | 核心类/文件                                                   | 说明                                                             |
| ------------ | ------------------ | -------------------------------------------------------- | -------------------------------------------------------------- |
| Timeline     | 时间驱动，帧切换，音频同步      | TimelinePlayer.kt / timeline_player.cpp                  | Audio 作为主时钟                                                    |
| Resource     | 纹理管理，异步加载，缓存       | AssetManager.kt / TextureCache.cpp / BasisTranscoder.cpp | LRU 缓存、预加载、GPU 上传                                              |
| GPU Render   | Shader 渲染，Layer 合成 | GLRenderer.kt / gl_renderer.cpp / shader_program.cpp     | 多层合成、FX Pass                                                   |
| Shader       | Visual FX          | *.frag / shader/*.h                                      | Feather, Alpha Blur, Shadow, Color Match, Bloom, LUT, RimLight |
| Audio        | 播放与同步              | AudioPlayer.kt / AudioSync.kt                            | Master clock                                                   |
| Bridge       | JNI / NDK          | NativeBridge.kt                                          | Kotlin <-> C++17                                               |
| Offline Tool | 资源构建               | petcat-builder                                           | WEBP → BasisU、Atlas Packing、Timeline、Material metadata         |

---

## 5. 项目文件夹结构

```
app/
├── src/main/
│   ├── java/com/petcat/runtime/
│   │   ├── app/
│   │   ├── ui/
│   │   ├── timeline/
│   │   ├── audio/
│   │   ├── resource/
│   │   ├── bridge/
│   │   └── config/
├── cpp/
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
│   │   ├── basisu/
│   │   ├── ktx/
│   │   └── glm/
│   └── CMakeLists.txt
├── assets/
│   ├── clips/
│   ├── shaders/
│   ├── materials/
│   └── configs/
```

---


## 6. Spec Coding 规划入口

本项目后续实现建议采用 **Spec Coding**：先固定接口、资源 schema、线程模型和验收指标，再按小步任务逐个实现。详细实施波次、首批 Spec 文件建议、任务模板见：

- `docs/SPEC_CODING_PLAN.md`：从设计文档到可执行 Spec 的总规划。
- `docs/SPEC_TEMPLATE.md`：每个 Spec 任务必须包含的字段模板。
- `AI Coding 任务清单.md`：模块级任务清单与优先级。

当前建议的首个编码任务不是 BasisU 或复杂 Shader，而是 **Android/NDK 最小工程骨架 + NativeBridge init/release + 空 GL context**，用于尽早验证 Gradle、NDK、CMake、JNI 与设备 GL 环境。

---
## 7. 实现计划 (Spec Coding 指导)

| 阶段 | 核心动作                                  | 负责人       | 时间周期  |
| -- | ------------------------------------- | --------- | ----- |
| S1 | 线程池 + Bitmap 池 + 优先级                  | AI Coding | 1 周   |
| S2 | OpenGL ES 渲染替换 Canvas                 | AI Coding | 2 周   |
| S3 | Shader 系统开发：Feather/Alpha Blur/Shadow | AI Coding | 2 周   |
| S4 | Basis Universal 集成 + GPU Upload       | AI Coding | 2~3 周 |
| S5 | Timeline 音频同步 & Scheduler             | AI Coding | 1~2 周 |
| S6 | 高级 FX Shader：Bloom/LUT/RimLight       | AI Coding | 2 周   |
| S7 | 多层背景+角色+阴影实时合成                        | AI Coding | 2~3 周 |
| S8 | 1080P/4K 测试 & 性能调优                    | AI Coding | 2~3 周 |

> 每阶段完成后可进行验收测试，指标包括解码 P50、渲染 P50、晚帧占比。

---

## 8. 核心风险与降级策略

1. **GPU 兼容性**：RK3288 Mali-T760 对 OpenGL ES 3.0/Shader 支持需测试；低端设备降级 Canvas + Bitmap。
2. **内存压力**：GPU 纹理 + Bitmap 池，需 LRU 回收 + 水位控制。
3. **工具链迁移**：BasisU 构建流程需验证。
4. **性能回退**：提供低分辨率 Atlas 或 WEBP 软解路径，确保稳定播放。

---

## 9. 总结

* 短期先解决同步解码与 GC 卡顿问题。
* 中期全面迁移到 GPU 原生 + Shader + BasisU。
* 长期实现多层 AV composition，支持高分辨率、复杂 FX、实时背景合成。
* 项目结构和模块设计完全对齐 Spec Coding，可直接用于 AI Coding 实现。

---
