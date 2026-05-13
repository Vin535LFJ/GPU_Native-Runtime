# PetCat Runtime Engine v2

## GPU-Native Realtime AV Composition Runtime

### Final Architecture Design Document (Spec Coding)

> Target Platform: Android (RK3288 first-class support)
> Rendering Backend: OpenGL ES 3.0 + GLSL
> Language: Kotlin(Java wrapper) + C++17(NDK core)
> Resource Pipeline: KTX2 + Basis Universal
> Runtime Type: Lightweight Realtime GPU AV Runtime
> Positioning: Animation/AV Composition Runtime (NOT Full Game Engine)

------

# 1. Project Positioning

# 1.1 What This Engine Actually Is

PetCat Runtime v2 is:

```text
Realtime GPU AV Composition Runtime
```

NOT:

```text
Simple frame animation player
```

and NOT:

```text
Full game engine
```

------

# 1.2 Core Technical Direction

The runtime evolves from:

```text
CPU Bitmap Playback
```

to:

```text
GPU Texture Streaming + Shader Composition
```

------

# 1.3 Long-Term Capability Goals

The engine must eventually support:

| Capability                       | Target |
| -------------------------------- | ------ |
| 720P/1080P/4K playback           | ✅      |
| Multi-layer realtime composition | ✅      |
| Shader FX pipeline               | ✅      |
| Audio synchronization            | ✅      |
| Timeline runtime                 | ✅      |
| GPU texture streaming            | ✅      |
| Video texture integration        | ✅      |
| Realtime transitions             | ✅      |
| AI video enhancement             | Future |
| Video editing runtime            | Future |

------

# 2. Core Architectural Philosophy

# 2.1 CPU Responsibility

CPU should ONLY handle:

- scheduling
- timeline update
- IO streaming
- resource management
- synchronization

------

# 2.2 GPU Responsibility

GPU should handle:

- all rendering
- texture composition
- scaling
- blending
- filtering
- transitions
- FX
- color grading

------

# 2.3 Runtime Philosophy

```text
Everything becomes GPU texture
```

NOT:

```text
Everything becomes Bitmap
```

------

# 3. High-Level Runtime Architecture

```text
                ┌────────────────────┐
                │ Offline Asset Tool │
                │ petcat-builder     │
                └─────────┬──────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼

   timeline.anim     atlas.ktx2      material.json

                          │
                          ▼

                ┌────────────────────┐
                │ Android Runtime    │
                └─────────┬──────────┘
                          │
      ┌───────────────────┼──────────────────┐
      ▼                   ▼                  ▼

 Timeline System     Resource System     Audio System

      │                   │                  │
      ▼                   ▼                  ▼

 Frame Scheduler    Texture Streaming   Audio Sync

      │                   │
      ▼                   ▼

                ┌────────────────────┐
                │ GPU Render Runtime │
                └─────────┬──────────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼

         Shader FX   Layer System   Texture Cache

                          │
                          ▼

                   OpenGL ES 3.0
```

------

# 4. Technology Stack

# 4.1 Android Layer

| Module                | Tech                   |
| --------------------- | ---------------------- |
| UI                    | Kotlin                 |
| GLSurfaceView wrapper | Kotlin                 |
| Lifecycle             | AndroidX               |
| Audio                 | ExoPlayer / AudioTrack |
| JNI bridge            | JNI                    |

------

# 4.2 Native Layer

| Module            | Tech            |
| ----------------- | --------------- |
| Rendering         | OpenGL ES 3.0   |
| Runtime Core      | C++17           |
| Texture format    | KTX2            |
| Compression       | Basis Universal |
| Shader            | GLSL ES 3.0     |
| Threading         | std::thread     |
| Memory management | RAII            |

------

# 5. Resource Pipeline Design

# 5.1 Resource Philosophy

Runtime MUST NOT perform:

- PNG decode
- WEBP decode
- heavy conversion

at playback time.

------

# 5.2 Offline Asset Builder

Must implement independent tool:

```text
petcat-builder
```

------

# 5.3 Asset Builder Responsibilities

| Task                   | Required |
| ---------------------- | -------- |
| Video frame extraction | ✅        |
| Atlas packing          | ✅        |
| BasisU encoding        | ✅        |
| KTX2 packaging         | ✅        |
| Timeline generation    | ✅        |
| Audio extraction       | ✅        |
| Material metadata      | ✅        |

------

# 5.4 Runtime Resource Types

| File        | Purpose                |
| ----------- | ---------------------- |
| `.ktx2`     | GPU texture atlas      |
| `.anim`     | timeline               |
| `.material` | shader/material config |
| `.audio`    | extracted audio        |
| `.json`     | metadata               |

------

# 5.5 Final Resource Structure

```text
assets/
 ├── clips/
 │    ├── idle/
 │    │    ├── atlas_0.ktx2
 │    │    ├── atlas_1.ktx2
 │    │    ├── timeline.anim
 │    │    ├── material.json
 │    │    └── idle.audio
 │    │
 │    └── walk/
 │
 ├── shaders/
 │    ├── feather.frag
 │    ├── composite.frag
 │    ├── blur.frag
 │    ├── lut.frag
 │    └── shadow.frag
 │
 └── configs/
```

------

# 6. Rendering Architecture

# 6.1 Rendering Backend

Mandatory:

```text
OpenGL ES 3.0
```

------

# 6.2 Rendering Thread Model

Dedicated GL thread only.

GL thread responsibilities:

- shader compile
- texture upload
- render pass
- framebuffer
- swap buffers

------

# 6.3 Forbidden on GL Thread

- disk IO
- transcoding
- asset parsing
- blocking synchronization

------

# 6.4 Render Pipeline

```text
Timeline Tick
      ↓
Collect Visible Layers
      ↓
Acquire Textures
      ↓
Apply Materials
      ↓
Execute Shader Passes
      ↓
Composite Layers
      ↓
Present Frame
```

------

# 7. GPU Texture System

# 7.1 Texture Format

Mandatory:

```text
KTX2 + BasisU
```

------

# 7.2 Texture Streaming Pipeline

```text
Asset IO
   ↓
KTX2 Stream
   ↓
Basis Transcode
   ↓
GPU Upload
   ↓
Texture Cache
```

------

# 7.3 Texture Cache Requirements

Must support:

- async upload
- LRU eviction
- reference count
- prefetch queue
- memory watermark

------

# 7.4 Texture Upload Rules

Must avoid:

```text
per-frame glTexImage2D
```

------

# 7.5 Correct Upload Strategy

```text
upload once
reuse many frames
```

------

# 8. Timeline Runtime System

# 8.1 Timeline Philosophy

Everything in runtime should be time-driven.

------

# 8.2 Timeline Responsibilities

| Item                | Required |
| ------------------- | -------- |
| Frame switching     | ✅        |
| Audio sync          | ✅        |
| FX parameter update | ✅        |
| Layer visibility    | ✅        |
| Transition control  | ✅        |

------

# 8.3 Scheduler

Mandatory:

```java
Choreographer.postFrameCallback()
```

------

# 8.4 Forbidden Scheduler

```java
Thread.sleep()
```

------

# 8.5 Future Timeline Expansion

Future support:

- keyframes
- easing
- parameter curves
- multi-track editing

------

# 9. Audio Runtime System

# 9.1 Audio Architecture

Audio becomes master clock source.

------

# 9.2 Synchronization Strategy

```text
Audio Time
    ↓
Timeline Time
    ↓
Frame Selection
```

------

# 9.3 Why Audio-Driven Timeline

Avoids:

- animation drift
- desync
- timing jitter

------

# 9.4 Audio Features

| Feature              | Required |
| -------------------- | -------- |
| loop                 | ✅        |
| seek                 | ✅        |
| pause/resume         | ✅        |
| latency compensation | ✅        |

------

# 10. Layer Composition System

# 10.1 Layer Types

| Layer      | Purpose         |
| ---------- | --------------- |
| Background | scene           |
| Character  | pet             |
| Shadow     | realtime shadow |
| FX         | effects         |
| Overlay    | UI              |

------

# 10.2 Composition Order

```text
Background
  ↓
Shadow
  ↓
Character
  ↓
FX
  ↓
Overlay
```

------

# 10.3 Layer Philosophy

Everything composited in GPU.

------

# 11. Shader System

# 11.1 Shader Philosophy

Shader becomes primary visual logic layer.

------

# 11.2 Mandatory Shader Features

## Phase 1

| Shader      | Purpose               |
| ----------- | --------------------- |
| Feather     | edge blending         |
| Alpha Blur  | soft edge             |
| Shadow      | contact shadow        |
| Color Match | background tone match |

------

# 11.3 Phase 2 Shaders

| Shader   | Purpose        |
| -------- | -------------- |
| Bloom    | glow           |
| LUT      | grading        |
| Sharpen  | detail enhance |
| Denoise  | cleanup        |
| RimLight | integration    |

------

# 11.4 Future Video FX Direction

Future:

- YUV shader
- AI shader
- temporal shader
- super resolution

------

# 12. AV Runtime Evolution

# 12.1 Current Stage

```text
Atlas Runtime
```

------

# 12.2 Next Stage

```text
GPU Texture Runtime
```

------

# 12.3 Future Stage

```text
Video Texture Runtime
```

------

# 12.4 Final Stage

```text
Realtime GPU AV Engine
```

------

# 13. Project Directory Structure

# 13.1 Final Recommended Structure

```text
app/
├── src/main/
│
├── java/com/petcat/runtime/
│   ├── app/
│   │    └── PetCatApplication.kt
│   │
│   ├── ui/
│   │    ├── MainActivity.kt
│   │    ├── RuntimeSurfaceView.kt
│   │    └── RuntimeRenderer.kt
│   │
│   ├── timeline/
│   │    ├── TimelinePlayer.kt
│   │    ├── Track.kt
│   │    ├── Clip.kt
│   │    └── Scheduler.kt
│   │
│   ├── audio/
│   │    ├── AudioPlayer.kt
│   │    └── AudioSync.kt
│   │
│   ├── resource/
│   │    ├── AssetManager.kt
│   │    ├── ClipLoader.kt
│   │    ├── TextureStream.kt
│   │    └── MaterialLoader.kt
│   │
│   ├── bridge/
│   │    └── NativeBridge.kt
│   │
│   └── config/
│
├── cpp/
│   ├── core/
│   │    ├── runtime/
│   │    ├── timeline/
│   │    ├── render/
│   │    ├── shader/
│   │    ├── texture/
│   │    ├── material/
│   │    ├── memory/
│   │    └── threading/
│   │
│   ├── renderer/
│   │    ├── gl_renderer.cpp
│   │    ├── framebuffer.cpp
│   │    └── shader_program.cpp
│   │
│   ├── texture/
│   │    ├── ktx2_loader.cpp
│   │    ├── basis_transcoder.cpp
│   │    └── texture_cache.cpp
│   │
│   ├── timeline/
│   │    ├── timeline_player.cpp
│   │    └── track_runtime.cpp
│   │
│   ├── audio/
│   │
│   ├── shader/
│   │
│   ├── third_party/
│   │    ├── basisu/
│   │    ├── ktx/
│   │    └── glm/
│   │
│   └── CMakeLists.txt
│
├── assets/
│   ├── clips/
│   ├── shaders/
│   ├── materials/
│   └── configs/
```

------

# 14. Why This Architecture Is Correct

Because this architecture:

| Requirement             | Solved |
| ----------------------- | ------ |
| 720P bottleneck         | ✅      |
| 1080P future            | ✅      |
| 4K future               | ✅      |
| Multi-layer composition | ✅      |
| AV synchronization      | ✅      |
| Shader FX               | ✅      |
| Video runtime evolution | ✅      |
| GPU-native pipeline     | ✅      |

------

# 15. Final Technical Conclusion

PetCat Runtime v2 should evolve into:

```text
GPU-native realtime AV composition runtime
```

with:

```text
KTX2/BasisU
+
OpenGL ES 3.0
+
GLSL shader pipeline
+
Timeline runtime
+
Audio synchronization
+
GPU composition
```

because:

- Canvas architecture has reached its limit
- Bitmap-centric rendering cannot scale
- Shader composition is mandatory for visual quality
- GPU texture streaming is mandatory for 1080P/4K
- Timeline-driven AV runtime is the correct long-term abstraction
- This architecture naturally evolves into a future video/FX engine.
