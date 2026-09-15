[🏠 Back to AI Index](./README.md) • [⬅️ Prev: Deep Learning](./03-deep-learning-neural-networks.md) • [Next: Modern RAG ➡️](./05-modern-rag-vector-databases.md)

<div align="center">
  <h1>04. LLMs, Transformers & Generative AI</h1>
  <p><b>Decoder Architectures, BPE Tokenization, RoPE, Grouped-Query Attention, FlashAttention & Alignment</b></p>
</div>

---

## 📑 Module Index
- [1. The Transformer Architecture Deep Dive](#1-the-transformer-architecture-deep-dive)
  - [Encoder-Only (BERT) vs Encoder-Decoder (T5) vs Decoder-Only (GPT, Llama)](#transformer-flavors)
  - [The Autoregressive Decoder Block](#the-autoregressive-decoder-block)
- [2. Tokenization & High-Dimensional Embeddings](#2-tokenization--high-dimensional-embeddings)
  - [Byte-Pair Encoding (BPE) & Tiktoken Internals](#bpe-tokenization)
  - [Rotary Position Embeddings (RoPE)](#rotary-position-embeddings-rope)
- [3. Modern Attention Optimizations](#3-modern-attention-optimizations)
  - [Multi-Head Attention (MHA) vs Multi-Query (MQA) vs Grouped-Query Attention (GQA)](#mha-mqa-gqa)
  - [FlashAttention & Memory Bandwidth Bottlenecks](#flashattention)
  - [The KV Cache (Key-Value Cache)](#the-kv-cache)
- [4. Training & Alignment Lifecycle](#4-training--alignment-lifecycle)
  - [Phase 1: Self-Supervised Pre-Training](#pre-training)
  - [Phase 2: Supervised Fine-Tuning (SFT)](#sft)
  - [Phase 3: Alignment (RLHF vs DPO - Direct Preference Optimization)](#alignment-rlhf-vs-dpo)
- [5. Inference Sampling Controls](#5-inference-sampling-controls)
  - [Temperature, Top-P (Nucleus), Top-K & Structured Outputs](#sampling-parameters)
  - [Context Caching (Prompt Caching in Claude / Gemini)](#context-caching)

---

## 1. The Transformer Architecture Deep Dive

### Transformer Flavors

```
┌────────────────────────┬────────────────────────┬────────────────────────┐
│   ENCODER-ONLY (BERT)  │  ENCODER-DECODER (T5)  │  DECODER-ONLY (GPT/LLM)│
├────────────────────────┼────────────────────────┼────────────────────────┤
│ • Bidirectional        │ • Encodes full input;  │ • Causal / Masked      │
│   attention (looks     │   decodes output       │   attention (looks only│
│   left & right).       │   autoregressively.    │   at past tokens).     │
│ • Best for: Search,    │ • Best for: Language   │ • **De-facto Standard**│
│   Embeddings, Classify │   Translation, Abstract│   for GPT-4o, Claude,  │
│   (e.g., `text-embed`) │   summarization.       │   Llama 3, DeepSeek.   │
└────────────────────────┴────────────────────────┴────────────────────────┘
```

---

### The Autoregressive Decoder Block

Modern LLMs stack 32 to 128 identical Decoder layers:

```
[ Input Tokens ] ──► [ Token Embedding + RoPE Positional Encoding ]
                           │
                           ▼
                  ┌─────────────────────────────────┐
                  │ ──► RMSNorm                     │
                  │ ──► Causal Multi-Head (GQA)    │ ──⊕ (Residual Connection)
                  │ ──► RMSNorm                     │
                  │ ──► SwiGLU Feed-Forward Network │ ──⊕ (Residual Connection)
                  └─────────────────────────────────┘ (Repeated N Layers)
                           │
                           ▼
                      [ Final RMSNorm ]
                           │
                           ▼
             [ Linear Projection (Unembed) ] ──► [ Softmax Logits over Vocab ] ──► Next Token
```

- **Causal Masking:** Ensures token $t$ cannot look ahead at future tokens $t+1, t+2$ during training by masking future positions with $-\infty$.

---

## 2. Tokenization & High-Dimensional Embeddings

### Byte-Pair Encoding (BPE)

LLMs do not read letters or whole words; they read **tokens** (statistical sub-word units):

```python
import tiktoken

enc = tiktoken.get_encoding("cl100k_base") # OpenAI tokenizer
tokens = enc.encode("Artificial Intelligence & Agents")
print(tokens) # [26221, 10793, 1111, 23773]
```

- **Algorithm:**
  1. Start with individual bytes/characters as base vocabulary.
  2. Iteratively count the most frequent adjacent byte-pair across the corpus and merge them into a new single token.
  3. Repeat until target vocabulary size is reached (typically 32,000 to 128,000 tokens).

---

### Rotary Position Embeddings (RoPE)

Instead of adding static vectors to token embeddings, **RoPE** rotates the Query and Key vectors in the complex plane by an angle proportional to their position:

$$\mathbf{R}_{\Theta, m}^d = \text{diag}\left( R_{\theta_1, m}, R_{\theta_2, m}, \dots, R_{\theta_{d/2}, m} \right)$$

- **Why RoPE is in all 2026 models:** Preserves relative distance naturally ($q_m^T k_n$ depends purely on $m - n$), allowing models to scale context windows from 8k up to **128k–1M+ tokens** with RoPE scaling interpolation (YaRN).

---

## 3. Modern Attention Optimizations

### MHA vs MQA vs GQA (Grouped-Query Attention)

During inference, loading the **Key-Value (KV) Cache** from GPU VRAM is the primary bottleneck.

```
       MHA (Standard)                      GQA (Llama 3 / Mistral)                   MQA
  8 Query heads : 8 KV heads             8 Query heads : 2 KV heads           8 Query heads : 1 KV head
    Q Q Q Q Q Q Q Q                         Q Q Q Q   Q Q Q Q                     Q Q Q Q Q Q Q Q
    │ │ │ │ │ │ │ │                         └──┬──┘   └──┬──┘                     └──────┬──────┘
    K K K K K K K K                            K         K                               K
    V V V V V V V V                            V         V                               V
  (Full VRAM footprint)                  (4x-8x Less VRAM, Same Quality)           (Extreme VRAM savings)
```

- **GQA (Grouped-Query Attention):** Groups multiple Query heads to share single Key and Value heads. Delivers **$4\times$ to $8\times$ reduction in KV cache memory** with zero loss in output quality.

---

### FlashAttention & GPU IO Bottlenecks

Standard PyTorch attention writes intermediate $(N \times N)$ attention matrix to High-Bandwidth GPU Memory (HBM) and reads it back, which is memory-bound.

**FlashAttention (Dao et al.):**
- Uses **Tiling** (splitting $Q, K, V$ into blocks that fit in fast GPU SRAM).
- Computes Softmax incrementally using online normalization without ever storing the large $N \times N$ matrix in HBM.
- **Result:** $3\times–8\times$ faster attention computation with $O(N)$ memory instead of $O(N^2)$.

---

## 4. Training & Alignment Lifecycle

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. PRE-TRAINING (Self-Supervised)                                                      │
│    • Ingest 15–20 Trillion tokens (Common Crawl, GitHub, Books, Papers).               │
│    • Cost: Millions of GPU hours.                                                      │
│    • Output: Base Foundation Model (Completes text, but uncontrollable).                │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. SUPERVISED FINE-TUNING (SFT)                                                        │
│    • 100k–1M curated (Instruction, Response) dialogue pairs.                           │
│    • Teaches model to act as a helpful conversational assistant.                       │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. ALIGNMENT (RLHF vs DPO)                                                             │
│    • RLHF: Uses Reward Model + PPO reinforcement learning.                             │
│    • DPO (Direct Preference Optimization): Eliminates reward model! Optimizes          │
│      closed-form cross-entropy directly on preference pairs (Chosen vs Rejected).      │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Inference Sampling Controls

When an LLM generates the next token, the output layer produces a probability distribution over the entire vocabulary:

```
Token:       "the"     "a"      "my"     "pizza"
Probability:  0.65     0.25     0.08      0.02
```

### Sampling Parameters

1. **Temperature ($T$):** Scales logits before Softmax ($z_i / T$):
   - $T \rightarrow 0$ (Argmax / Greedy): Always picks token with highest probability. Purely deterministic.
   - $T = 0.7$: Balanced creativity and coherence.
   - $T > 1.2$: Flattens probabilities $\rightarrow$ high randomness and potential hallucinations.
2. **Top-P (Nucleus Sampling):** Dynamically sums tokens with highest probabilities until cumulative probability exceeds $P$ (e.g., $P = 0.9$), discarding the tail.
3. **Top-K:** Restricts candidate pool to top $K$ highest tokens (e.g., $K = 50$).

---

### Context Caching (Prompt Caching)

In LLM API systems (Claude 3.5, Gemini 1.5), system prompts and large document attachments (100k+ tokens) are stored in GPU memory after the first request:
- **Cost Reduction:** Cached input tokens cost up to **$90\%$ less**.
- **Latency:** Time-to-first-token (TTFT) drops from seconds to **milliseconds**.

---

[➡️ Continue to Module 05: Modern RAG & Vector Databases](./05-modern-rag-vector-databases.md)
