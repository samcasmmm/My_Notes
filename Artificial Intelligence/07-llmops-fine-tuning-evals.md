[🏠 Back to AI Index](./README.md) • [⬅️ Prev: AI Agents](./06-autonomous-agents-tool-use.md) • [Next: Interview Mastery ➡️](./08-ai-interview-mastery.md)

<div align="center">
  <h1>07. Fine-Tuning, Alignment & LLMOps</h1>
  <p><b>LoRA, QLoRA, DPO, Quantization (AWQ/FP8/GGUF), vLLM Serving, & RAGAS Evaluations</b></p>
</div>

---

## 📑 Module Index
- [1. Parameter-Efficient Fine-Tuning (PEFT)](#1-parameter-efficient-fine-tuning-peft)
  - [LoRA (Low-Rank Adaptation) Mathematical Formulation](#lora-internals)
  - [QLoRA: 4-Bit NormalFloat (NF4) & Double Quantization](#qlora)
- [2. Model Alignment: SFT, RLHF & DPO](#2-model-alignment)
  - [Supervised Fine-Tuning (SFT)](#sft)
  - [Direct Preference Optimization (DPO) vs RLHF / PPO](#dpo-vs-rlhf)
- [3. Quantization Mechanics (FP16 to FP8 & INT4)](#3-quantization-mechanics)
  - [AWQ vs GPTQ vs GGUF Formats](#quantization-formats)
- [4. High-Throughput Production Serving](#4-high-throughput-production-serving)
  - [vLLM & PagedAttention Mechanics](#vllm--pagedattention)
  - [Continuous Batching vs Static Batching](#continuous-batching)
- [5. LLM Evaluation Frameworks (LLMOps)](#5-llm-evaluation-frameworks)
  - [RAGAS Metrics: Faithfulness, Precision, Recall, & Relevance](#ragas-metrics)
  - [LLM-as-a-Judge & Safety Guardrails (Llama Guard)](#guardrails)

---

## 1. Parameter-Efficient Fine-Tuning (PEFT)

Full fine-tuning of a 70-Billion parameter model requires updating all weights, demanding **hundreds of gigabytes of GPU VRAM**. **PEFT** freezes the base model and trains only a tiny fraction ($<1\%$) of parameters.

### LoRA (Low-Rank Adaptation)

LoRA freezes original pre-trained weight matrix $W_0 \in \mathbb{R}^{d \times k}$ and decomposes the update $\Delta W$ into two low-rank matrices $B$ and $A$:

$$W = W_0 + \Delta W = W_0 + \frac{\alpha}{r} (B \cdot A)$$

```
     Original Pre-Trained Weight Matrix W₀                LoRA Adapter Matrices
                     (Frozen)                                   (Trainable)
                 ┌──────────────┐                             ┌─────────┐
                 │              │                             │  B      │ (d x r)
                 │   d x k      │                             │         │
                 │              │                             └────┬────┘
                 └──────────────┘                                  ▼
                                                              ┌─────────┐
                                                              │  A      │ (r x k)
                                                              └─────────┘
                                                       where rank r << min(d, k) (e.g. r=16)
```

- **Rank ($r$):** Typically set to $8, 16, \text{ or } 32$.
- **Alpha ($\alpha$):** Scaling factor (typically set to $2 \times r$).
- **VRAM Savings:** Reduces memory footprint by up to **$80\%$**. Adapters weigh only $20–100\text{ MB}$ and can be swapped dynamically at runtime!

---

### QLoRA (Quantized LoRA)

Enables fine-tuning a **70B parameter model on a single 48GB GPU (or 7B model on a consumer 16GB GPU)** using 3 innovations:
1. **4-bit NormalFloat (NF4):** Information-theoretically optimal quantile quantization for normally distributed weights.
2. **Double Quantization:** Quantizes quantization constants themselves, saving $0.37$ bits per parameter.
3. **Paged Optimizers:** Uses CUDA unified memory to page optimizer memory spills to CPU RAM, preventing out-of-memory (OOM) crashes.

---

## 2. Model Alignment

```
Pre-Trained Base Model ──► SFT (Supervised Fine-Tuning) ──► DPO / RLHF (Human Alignment)
```

### DPO (Direct Preference Optimization)

Standard RLHF requires training a complex separate **Reward Model** followed by unstable PPO reinforcement learning loops.

**DPO (Rafailov et al.) derives an exact mathematical shortcut:**
- Re-parameterizes reward function in terms of optimal policy $\pi_\theta$ and reference model $\pi_{\text{ref}}$.
- Optimizes preference loss directly on prompt $x$ and pair $(y_w \text{ [chosen]}, y_l \text{ [rejected]})$:

$$\mathcal{L}_{\text{DPO}}(\theta) = -\mathbb{E} \left[ \log \sigma \left( \beta \log \frac{\pi_\theta(y_w \mid x)}{\pi_{\text{ref}}(y_w \mid x)} - \beta \log \frac{\pi_\theta(y_l \mid x)}{\pi_{\text{ref}}(y_l \mid x)} \right) \right]$$

- **Advantage:** Faster training, highly stable, and requires $50\%$ less GPU memory than RLHF.

---

## 3. Quantization Mechanics

Quantization maps continuous 32-bit/16-bit floating points into lower bit-depth representations:

```
FP32 (32-bit float)  ──►  FP16 / BF16 (16-bit)  ──►  FP8 (8-bit)  ──►  INT4 (4-bit)
   (4 Bytes/param)           (2 Bytes/param)          (1 Byte/param)      (0.5 Byte/param)
   70B Model = 280 GB        70B Model = 140 GB       70B Model = 70 GB   70B Model = ~35 GB
```

| Format | Quantization Method | Best Deployment Target |
| :--- | :--- | :--- |
| **AWQ** (Activation-aware) | Protects salient $1\%$ weights during INT4 quantization. | Production GPU inference (vLLM, TensorRT-LLM). |
| **GGUF** (llama.cpp) | Flexible quantized format supporting multi-level quantization (Q4_K_M, Q8_0). | CPU + GPU offloading, local devices, edge apps. |
| **FP8 (E4M3 / E5M2)** | Native 8-bit floating point supported by NVIDIA Ada Lovelace / Hopper GPUs. | Maximum throughput without precision loss. |

---

## 4. High-Throughput Production Serving

### vLLM & PagedAttention

Traditional inference engines allocate contiguous memory for KV Caches per request, causing **$60–80\%$ VRAM fragmentation waste**.

```
Traditional Serving (Contiguous):  [ Key-Value Memory ][ Wasted Space (Fragmentation) ]
vLLM PagedAttention:               [ Page 1 ][ Page 2 ][ Page 3 ] (Allocated on-demand in non-contiguous blocks)
```

- **PagedAttention:** Inspired by Virtual Memory OS paging; partitions KV cache into fixed-size physical blocks.
- **Continuous (Iteration-Level) Batching:** Batches incoming requests token-by-token immediately rather than waiting for an entire batch of requests to finish generating.
- **Throughput:** Delivers **$5\times–15\times$ higher requests-per-second (RPS)** than naive Hugging Face pipelines.

---

## 5. LLM Evaluation Frameworks (LLMOps)

### RAGAS Metrics for RAG Pipelines

Evaluating RAG requires decomposing performance across 4 independent dimensions:

```
             ┌───────────────────────── RETRIEVAL ─────────────────────────┐
             │                                                             │
      Context Precision                                              Context Recall
"Are relevant chunks ranked at top?"                          "Did we find all needed facts?"
             │                                                             │
             └───────────────────────── GENERATION ────────────────────────┘
             │                                                             │
        Faithfulness                                                Answer Relevance
"Is output derived ONLY from context?"                         "Does output answer user query?"
   (Detects Hallucinations)                                        (Detects Off-Topic replies)
```

---

### LLM-as-a-Judge & Safety Guardrails

- **LLM-as-a-Judge:** Uses a high-capability frontier model (GPT-4o / Claude 3.5) with strict rubrics to score output correctness, tone, and formatting consistency on test suites.
- **Llama Guard / Guardrails:** Fast classifier models placed in front of user inputs and model outputs to detect prompt injections, jailbreaks, toxicity, and PII leaks before reaching users.

---

[➡️ Continue to Module 08: AI & GenAI Interview Mastery](./08-ai-interview-mastery.md)
