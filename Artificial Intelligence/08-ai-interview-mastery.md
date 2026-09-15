[🏠 Back to AI Index](./README.md) • [⬅️ Prev: Fine-Tuning & LLMOps](./07-llmops-fine-tuning-evals.md)

<div align="center">
  <h1>08. AI & GenAI Interview Mastery</h1>
  <p><b>50+ High-Yield Production & Theoretical Interview Scenarios with Detailed Answers</b></p>
</div>

---

## 📑 Interview Sections
1. [Part 1: Core Machine Learning & Math](#part-1-core-machine-learning--math)
2. [Part 2: Deep Learning & Neural Architectures](#part-2-deep-learning--neural-architectures)
3. [Part 3: Transformers & LLM Internals](#part-3-transformers--llm-internals)
4. [Part 4: Modern RAG & Vector Search](#part-4-modern-rag--vector-search)
5. [Part 5: Fine-Tuning, Alignment & LLMOps](#part-5-fine-tuning-alignment--llmops)
6. [Part 6: AI System Design & Agent Architecture](#part-6-ai-system-design--agent-architecture)

---

## Part 1: Core Machine Learning & Math

#### Q1: What is the Bias-Variance Tradeoff and how do you diagnose it in production?
- **High Bias (Underfitting):** Model makes strong incorrect assumptions. Fails on both train and validation data. *Fix: Increase model complexity, add feature interactions.*
- **High Variance (Overfitting):** Model memorizes training noise. Near-perfect training score, but poor test score. *Fix: Regularization ($L_1/L_2$, Dropout), add more data, prune trees.*

#### Q2: When would you choose Precision over Recall, and vice-versa?
- **High Precision:** When the cost of a **False Positive (Type I)** is catastrophic (e.g., spam filter moving important emails to spam, false fraud accusations).
- **High Recall:** When the cost of a **False Negative (Type II)** is catastrophic (e.g., cancer detection, finding defective aircraft parts).

#### Q3: Why is Cross-Entropy preferred over MSE for Classification?
- MSE with Sigmoid produces non-convex error surfaces with plateaus where gradients vanish when predictions are confidently wrong ($y=1$, $\hat{y}=0.001$).
- Cross-Entropy produces a strictly convex loss whose gradient $\frac{\partial L}{\partial z} = \hat{y} - y$ is steep when the model is wrong, guaranteeing rapid convergence.

---

## Part 2: Deep Learning & Neural Architectures

#### Q4: Why does Batch Normalization behave differently during training vs evaluation?
- **Training:** Normalizes activations using the **mini-batch mean and variance**.
- **Evaluation/Inference:** Mini-batch statistics cannot be computed reliably for single-sample inference ($B=1$). It uses pre-computed **running exponential moving averages** of mean and variance accumulated during training.

#### Q5: How do Skip Connections in ResNet solve the degradation problem?
- Allows gradients to flow directly through the identity pathway $\mathcal{F}(x) + x$ back to early layers without shrinking, since $\frac{\partial (\mathcal{F}(x) + x)}{\partial x} = \frac{\partial \mathcal{F}(x)}{\partial x} + 1$. The $+1$ prevents gradients from vanishing.

---

## Part 3: Transformers & LLM Internals

#### Q6: What is the computational and memory complexity of Self-Attention with respect to sequence length $N$?
- **Complexity:** $O(N^2 \cdot d)$ because every token computes a dot-product with every other token ($Q K^T$ results in an $N \times N$ matrix).
- **Solution for long context:** FlashAttention (avoids writing $N \times N$ matrix to GPU memory) and Grouped-Query Attention (GQA).

#### Q7: Why is Grouped-Query Attention (GQA) used in modern models like Llama 3 instead of standard Multi-Head Attention (MHA)?
- During autoregressive generation, loading Key-Value (KV) caches from GPU VRAM is memory-bandwidth bound.
- GQA groups 8 Query heads to share a single Key and Value head, cutting **KV Cache memory consumption by $4\times–8\times$** while maintaining $99\%+$ model capability.

#### Q8: What does Temperature do mathematically during token generation?
- It divides logits by scalar $T$ before the Softmax: $P(w_i) = \frac{e^{z_i / T}}{\sum_j e^{z_j / T}}$.
  - When $T < 1.0$: Sharpens probability distribution toward top tokens (deterministic).
  - When $T > 1.0$: Flattens distribution, increasing probability of lower-ranked tokens (creative/random).

---

## Part 4: Modern RAG & Vector Search

#### Q9: What is the difference between Dense Vector Search and Sparse Lexical Search (BM25)?
- **Dense:** Captures latent semantic meaning and synonyms ("car" $\rightarrow$ "automobile").
- **Sparse (BM25):** Matches exact keywords, rare alphanumeric codes, and SKUs.
- **Production Best Practice:** **Hybrid Search** combining both using **Reciprocal Rank Fusion (RRF)**.

#### Q10: How does a Cross-Encoder Reranker improve RAG precision?
- Bi-Encoders encode query and documents independently (fast, but misses nuanced token-level interactions).
- Cross-Encoders pass the concatenated `[Query, Document]` through joint multi-head attention layers, accurately scoring the top 20 candidate documents before passing them to the LLM.

#### Q11: What are the 4 core metrics in the RAGAS evaluation framework?
1. **Faithfulness:** Measures if response is grounded solely in retrieved context (0 hallucinations).
2. **Answer Relevance:** Measures if response directly addresses user question.
3. **Context Precision:** Measures if signal-to-noise ratio in retrieved chunks is high.
4. **Context Recall:** Measures if all facts needed to answer query were successfully retrieved.

---

## Part 5: Fine-Tuning, Alignment & LLMOps

#### Q12: How does LoRA (Low-Rank Adaptation) reduce trainable parameters?
- It assumes weight updates have a low intrinsic dimension. It decomposes update $\Delta W \in \mathbb{R}^{d \times k}$ into $B \cdot A$ where $B \in \mathbb{R}^{d \times r}$ and $A \in \mathbb{R}^{r \times k}$ with rank $r \ll \min(d, k)$.
- For $d=4096, r=16$, parameters drop from $16.7\text{M}$ to just $131\text{k}$ per matrix ($>99\%$ reduction).

#### Q13: Why is DPO (Direct Preference Optimization) replacing RLHF with PPO?
- RLHF requires training an auxiliary Reward Model and running complex PPO reinforcement learning with unstable actor-critic loops.
- DPO derives an exact closed-form loss directly on preference pairs $(y_{\text{chosen}}, y_{\text{rejected}})$, achieving superior stability with $50\%$ less GPU VRAM.

#### Q14: How does vLLM achieve $10\times$ throughput compared to standard Hugging Face pipelines?
- **PagedAttention:** Allocates non-contiguous physical memory blocks for KV Caches like Virtual Memory OS paging, eliminating memory fragmentation waste ($>70\%$).
- **Continuous Batching:** Schedules new requests dynamically at token-level iteration boundaries.

---

## Part 6: AI System Design & Agent Architecture

#### Q15: How do you architect an Autonomous AI Agent with Tools and Safety Guardrails?

```
                      ┌──────────────────────────────────────────────┐
                      │                 USER CLIENT                  │
                      └──────────────────────┬───────────────────────┘
                                             │ User Prompt
                                             ▼
                      ┌──────────────────────────────────────────────┐
                      │        INPUT GUARDRAIL (Llama Guard)         │
                      │   • Prompt Injection Check • PII Masking     │
                      └──────────────────────┬───────────────────────┘
                                             │ Safe Prompt
                                             ▼
                      ┌──────────────────────────────────────────────┐
                      │             AGENT REASONING LOOP             │
                      │               (ReAct Pattern)                │
                      │  • Thought: Decide next step                 │
                      │  • Tool Call: Output JSON function schema    │
                      └──────────────┬───────────────────────────────┘
                                     │
                     ┌───────────────┴───────────────┐
                     ▼                               ▼
          [ Read-Only Tools ]             [ State-Changing Tools ]
          (Vector Search, SQL Read)       (Payments, Send Email, Delete)
                     │                               │
                     │                     ┌─────────▼──────────┐
                     │                     │ HUMAN-IN-THE-LOOP  │
                     │                     │ User Approval Modal│
                     │                     └─────────┬──────────┘
                     │                               │ Approved
                     └───────────────┬───────────────┘
                                     ▼
                      ┌──────────────────────────────┐
                      │    TOOL EXECUTION BACKEND    │
                      └──────────────┬───────────────┘
                                     │ Observation JSON
                                     ▼
                      ┌──────────────────────────────┐
                      │      LLM FINAL SYNTHESIS     │
                      └──────────────┬───────────────┘
                                     │ Generated Response
                                     ▼
                      ┌──────────────────────────────┐
                      │       OUTPUT GUARDRAIL       │
                      │   • Hallucination & Fact Check│
                      └──────────────┬───────────────┘
                                     │ Verified Stream
                                     ▼
                                [ USER UI ]
```

---

<div align="center">
  <sub><b>My_Notes</b> • Continuous Learning & Interview Mastery</sub><br/>
  <a href="./README.md"><b>Back to AI Curriculum Index 🏠</b></a>
</div>
