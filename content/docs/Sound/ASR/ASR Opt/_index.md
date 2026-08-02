# ASR 端到端延迟优化指南

> "In real-time speech processing, latency is not just a metric — it is the user experience. Every millisecond of delay breaks the illusion of a natural conversation."

## 一、核心指标定义

在优化之前，明确我们要优化的目标指标：

1. **首字延迟 (First Token Latency / Time-To-First-Word)**
   - **定义**：用户说完一句话，到第一个文字上屏的时间。
   - **意义**：决定交互的"跟手"感，直接影响用户感知的响应速度。
   - **目标**：< 200ms (极致), < 500ms (优秀), < 1s (可接受)。

2. **尾字延迟 (Last Token Latency)**
   - **定义**：用户说完一句话，到整句结果完整输出并确定的时间。
   - **意义**：决定最终结果产出的速度，影响后续处理（如 TTS 或 Agent 执行）的开始时间。
   - **目标**：< 1s (极致), < 2s (优秀)。

---

## 二、算法与架构层（决定性因素）

架构选型是降低延迟的物理上限。如果模型本身是非流式的，后端的优化空间非常有限。

### 2.1 流式 vs 非流式架构对比

| 架构类型 | 代表模型 | 延迟表现 | 精度表现 | 适用场景 |
|---------|---------|---------|---------|---------|
| **全注意力 (Full Attention)** | Whisper (原版), Wav2Vec2 | 🔴 极高 (需整句结束) | 🟢 极高 | 录音转写、离线分析、高容忍度场景 |
| **CTC (Connectionist Temporal Classification)** | CTC-Transformer, Paraformer-CTC | 🟡 中 (需 Beam Search 解码) | 🟡 中高 | 需要快速首字，允许后续修正 |
| **RNN-T (Transducer)** | Conformer-RNNT, U2++ | 🟢 极低 (逐字输出) | 🟢 高 | **实时对话、同声传译、语音控制** |
| **Chunk-based Attention** | WeNet, Paraformer-stream | 🟡 低 (固定 Chunk 延迟) | 🟡 高 | **平衡精度与延迟的主流方案** |

**核心结论**：
- 要实现低延迟，必须使用**流式架构 (Streaming Architecture)**。
- **RNN-T** 是目前延迟最低的架构，支持 Symbol-by-symbol 输出，无需等待 Chunk 结束。
- **Chunk-based (分块)** 是平衡方案，将长音频切分为固定大小的块（如 320ms），模型只看当前块和有限上下文。

### 2.2 解码策略优化

- **CTC Prefix Beam Search**：比传统 Beam Search 快，利用 CTC 的合并特性减少搜索空间。
- **CTC Prefix Rescoring**：先用 CTC 快速解码，再用语言模型 (LM) 局部打分。
- **投机解码 (Speculative Decoding)**：小模型快速生成草稿，大模型并行验证。ASR 中可用轻量级 CTC 做 Draft，大模型做 Verification。

---

## 三、推理工程层（算力榨取）

### 3.1 推理引擎加速

| 方案 | 加速比 | 说明 | 推荐框架 |
|------|-------|------|---------|
| **TensorRT / TensorRT-LLM** | 2x - 5x | 算子融合、内核调优，NVIDIA GPU 首选 | TensorRT |
| **CTranslate2 (CT2)** | 2x - 4x | CPU/GPU 双优化，Whisper 加速事实标准 | Faster-Whisper |
| **ONNX Runtime** | 1.5x - 2x | 跨平台兼容性好 | ORT |
| **Sherpa-ONNX / K2** | 2x - 3x | 专为 ASR 设计，C++ 核心，流式支持极佳 | Sherpa-ONNX |

### 3.2 量化 (Quantization)

- **INT8 量化**：ASR 模型 (Encoder) 量化后精度损失极小 (WER 增加 < 0.5%)，速度提升 2-3 倍。
- **FP8 量化**：H100 架构支持，精度优于 INT8，速度接近 FP16。
- **AWQ / GPTQ**：针对 Speech-LLM 类大模型进行权重量化。

### 3.3 显存与 KV Cache 优化

- **PagedAttention**：类似 vLLM，避免显存碎片，提升并发吞吐。
- **State Management**：流式推理的核心是**复用**。每次只计算新 Chunk，复用上一步的 Hidden States/KV Cache，避免重复计算。

---

## 四、管线与系统层（隐藏延迟）

### 4.1 VAD (Voice Activity Detection) 策略

VAD 是 ASR 的"门卫"，决定了音频何时送入模型。

- **激进的 VAD**：检测到语音立即启动，降低首字延迟，但可能引入噪声。
- **延迟截断 (Tail Trimming)**：不要等用户完全沉默才发送。可以在 300ms 无语音时就发送已录制音频，让结果"追"着语音出来。
- **预启动 (Pre-roll)**：VAD 触发前保留一小段静音或起音，避免吞字。

### 4.2 流水线并行 (Pipeline Parallelism)

```
Audio Capture -> Feature Extraction -> Model Encoder -> Model Decoder -> Post-Processing
      (Thread 1)        (Thread 2)         (Thread 3)       (Thread 4)       (Thread 5)
```
- **双缓冲 (Double Buffering)**：计算当前 Chunk 时，预处理下一个 Chunk。
- **异步 I/O**：使用 WebSocket/gRPC 异步流传输，避免阻塞。

### 4.3 中间结果 (Partial Hypotheses)

- **机制**：在最终确定前，输出临时结果。用户看到文字不断修正和追加。
- **效果**：感知延迟大幅降低，用户感觉"系统听懂了"。
- **实现**：框架通常输出 `is_final=True/False` 字段。

---

## 五、主流框架优化策略

### 5.1 Whisper 优化
Whisper 本身是非流式的，必须通过工程手段实现流式。
- **Faster-Whisper**：底层 CTranslate2，支持 INT8 量化，速度比原版快 4 倍。
- **流式 Hack**：使用 `whisper-streaming` 或滑动窗口机制，配合状态缓存。
- **VAD 配合**：结合 Silero VAD，仅有人说话时送推理，减少无效计算。

### 5.2 极致低延迟推荐：Sherpa-ONNX
- **K2 团队出品**：C++ 核心，无 Python 开销。
- **支持**：Transducer, Paraformer, Whisper 流式版。
- **特性**：边缘设备友好 (Android/iOS/RK3588)，流式 API 原生支持。

### 5.3 工业级方案：Paraformer (阿里)
- **架构**：Non-autoregressive (NAR)，推理速度天生比自回归模型快。
- **流式**：支持 Chunk-based streaming，精度与延迟平衡极佳。

---

## 六、优化 Checklist

| 优化项 | 具体动作 | 预期收益 |
|--------|------|---------|
| **架构** | 切换至 Streaming/RNN-T/Chunk-based | 延迟从 10s+ 降至 < 1s |
| **VAD** | 启用 Silero VAD，配置 Tail Trimming | 减少 30% 无效推理，首字更快 |
| **Chunk** | Chunk size 设为 0.32s - 0.64s | 延迟降低 80% |
| **引擎** | 使用 TensorRT 或 CTranslate2 | 速度提升 2-4 倍 |
| **量化** | INT8 / FP8 | 显存减半，速度提升 2 倍 |
| **解码** | Beam Size 降至 4，CTC Prefix | 解码时间减少 50% |
| **并发** | Continuous Batching | P99 延迟显著降低 |

---

*文档版本：v1.0 | 作者：小伟 | 日期：2026-07-29*
