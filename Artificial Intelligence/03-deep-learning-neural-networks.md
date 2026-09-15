[🏠 Back to AI Index](./README.md) • [⬅️ Prev: Machine Learning Core](./02-machine-learning-core.md) • [Next: LLMs & GenAI ➡️](./04-llms-transformers-genai.md)

<div align="center">
  <h1>03. Deep Learning & Neural Networks</h1>
  <p><b>Neural Architecture, PyTorch Internals, Optimizers, CNNs, ResNets, LSTMs, & Attention</b></p>
</div>

---

## 📑 Module Index
- [1. Neural Network Architecture](#1-neural-network-architecture)
  - [The Artificial Neuron & Multi-Layer Perceptron (MLP)](#the-artificial-neuron)
  - [Modern Activation Functions (ReLU, GELU, Swish, Softmax)](#activation-functions)
- [2. Training Mechanics & Optimization](#2-training-mechanics--optimization)
  - [Computational Graph & Autograd in PyTorch](#pytorch-autograd)
  - [Backpropagation & The Chain Rule](#backpropagation)
  - [Optimizers: SGD, Momentum, RMSprop, Adam vs AdamW](#optimizers)
  - [Loss Functions (Cross-Entropy, MSE, Focal Loss)](#loss-functions)
- [3. Deep Learning Regularization & Normalization](#3-regularization--normalization)
  - [Dropout, Weight Decay ($L_2$), & Data Augmentation](#regularization)
  - [Batch Normalization (BatchNorm) vs Layer Normalization (LayerNorm)](#batchnorm-vs-layernorm)
- [4. Computer Vision & CNNs](#4-computer-vision--cnns)
  - [Convolutions, Filters/Kernels, Stride & Padding](#convolution-operations)
  - [ResNet & Residual Skip Connections](#resnet--skip-connections)
- [5. Sequence Models & Recurrence](#5-sequence-models--recurrence)
  - [Recurrent Neural Networks (RNN) & The Vanishing Gradient Problem](#rnn-limitations)
  - [LSTM (Long Short-Term Memory) & GRU Internal Gates](#lstm--gru)
- [6. The Foundation of Transformers: Self-Attention](#6-the-self-attention-mechanism)
  - [Queries, Keys, & Values (Q, K, V)](#q-k-v-intuition)
  - [Scaled Dot-Product Attention & Multi-Head Attention](#scaled-dot-product-attention)

---

## 1. Neural Network Architecture

### The Artificial Neuron

A single neuron receives an input vector $\mathbf{x} = [x_1, \dots, x_d]$, calculates a weighted sum with bias $b$, and passes it through an activation function $\sigma$:

$$z = \mathbf{w}^T \mathbf{x} + b = \sum_{i=1}^d w_i x_i + b$$
$$a = \sigma(z)$$

```
  x₁ ──► [ w₁ ] ──┐
  x₂ ──► [ w₂ ] ──┼──► [ ∑ (wᵢ xᵢ) + b ] ──► [ Activation σ(z) ] ──► Output (a)
  x₃ ──► [ w₃ ] ──┘
```

---

### Modern Activation Functions

```
       ReLU                 GELU (Used in LLMs/BERT)             Swish / SiLU
   f(x) = max(0, x)         f(x) = x · Φ(x)                      f(x) = x · σ(βx)
         /                       /                                   /
        /                       /                                   /
    ───┘                   ────╯                                ───╯
```

| Function | Formula | Characteristics & Usage |
| :--- | :--- | :--- |
| **ReLU** | $\max(0, x)$ | Fast to compute; fixes vanishing gradient for positive values; prone to "Dying ReLU" if neurons output 0 permanently. |
| **GELU** | $x \cdot \Phi(x)$ | Gaussian Error Linear Unit; smooth non-linearity; standard in **BERT, GPT, Claude, Llama**. |
| **Swish / SiLU**| $x \cdot \sigma(x)$ | Discovered by Google; smooth gradient; standard in **Modern LLMs (Llama 3 SwiGLU)**. |
| **Softmax** | $\frac{e^{z_i}}{\sum_j e^{z_j}}$ | Converts raw logits vector into a probability distribution summing to $1.0$ at the final multi-class output layer. |

---

## 2. Training Mechanics & Optimization

### PyTorch Autograd & Model Training Loop

```python
import torch
import torch.nn as nn
import torch.optim as optim

# 1. Define Model Architecture
class MLP(nn.Module):
    def __init__(self, in_features=128, hidden=64, num_classes=10):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_features, hidden),
            nn.LayerNorm(hidden),
            nn.GELU(),
            nn.Dropout(0.1),
            nn.Linear(hidden, num_classes)
        )

    def forward(self, x):
        return self.net(x)

# 2. Setup Loss & Optimizer
model = MLP()
criterion = nn.CrossEntropyLoss()
optimizer = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-2)

# 3. Complete Training Step
optimizer.zero_grad()            # 1. Clear previous gradients
outputs = model(inputs)          # 2. Forward pass
loss = criterion(outputs, labels)# 3. Compute loss
loss.backward()                  # 4. Backward pass (Autograd computes gradients)
optimizer.step()                 # 5. Update weights
```

---

### Optimizers: SGD to AdamW

```
SGD  ──►  SGD + Momentum  ──►  RMSprop  ──►  Adam  ──►  AdamW (Decoupled Weight Decay)
```

1. **SGD with Momentum:** Adds a fraction $\beta$ of the past update vector to avoid getting stuck in shallow local minima:
   $$v_t = \beta v_{t-1} + (1-\beta) \nabla L(w_t), \quad w_{t+1} = w_t - \alpha v_t$$
2. **RMSprop:** Scales learning rate adaptively per parameter by dividing by moving average of squared gradients.
3. **Adam (Adaptive Moment Estimation):** Combines Momentum (1st moment $m_t$) + RMSprop (2nd moment $v_t$).
4. **AdamW (Standard in all 2026 LLMs & Vision models):** Fixes standard Adam by decoupling $L_2$ weight decay regularization directly from gradient updates, preventing weight blowup.

---

## 3. Deep Learning Regularization & Normalization

### BatchNorm vs LayerNorm

```
         BATCH NORMALIZATION (CV)                    LAYER NORMALIZATION (NLP & LLMs)
   Normalizes across all samples in Batch      Normalizes across all features for a single sample
            [ Sample 1: f₁, f₂, f₃ ]                     [ Sample 1: f₁, f₂, f₃ ]  <── Normalized alone!
            [ Sample 2: f₁, f₂, f₃ ]                     [ Sample 2: f₁, f₂, f₃ ]  <── Normalized alone!
            [ Sample 3: f₁, f₂, f₃ ]                     [ Sample 3: f₁, f₂, f₃ ]  <── Normalized alone!
              ▲   ▲   ▲
              └───┴───┴── Normalized per feature column
```

- **Batch Normalization:** Dependent on batch size (fails on small batches and variable sequence lengths). Perfect for **Computer Vision (CNNs)**.
- **Layer Normalization:** Independent of batch size; normalizes across the feature dimension for each token. Standard in **Transformers & LLMs**.
- **RMSNorm (Root Mean Square Normalization):** A faster variant of LayerNorm that skips mean calculation, used in **Llama 3 and Mistral**.

---

## 4. Computer Vision & CNNs

### Convolution Operations

Instead of fully connecting every pixel, a **Convolutional Layer** slides small learnable filters (e.g., $3 \times 3$ matrices) across the input image:

```
  Input Image (5x5)        Kernel/Filter (3x3)        Feature Map (3x3)
  [ 1  0  1  2  1 ]           [ 1  0 -1 ]
  [ 0  3  2  1  0 ]     *     [ 1  0 -1 ]      ──►    [ 4   2   1 ]
  [ 2  1  0  3  2 ]           [ 1  0 -1 ]             [ ... ... ... ]
```

- **Stride ($S$):** Step size the filter moves (Stride 2 halves output resolution).
- **Padding ($P$):** Adding zeros around borders to preserve spatial dimensions.
- **Output Size:**
  $$O = \left\lfloor \frac{W - K + 2P}{S} \right\rfloor + 1$$

---

### ResNet & Skip Connections

Deep neural networks (>20 layers) suffer from the **Degradation Problem** (vanishing gradients making deeper models perform worse than shallow ones).

**He et al. (2015) introduced Residual Learning:**

```
           x ─────────────── Identity Shortcut ──────────────┐
           │                                                 │
           ▼                                                 ▼
      [ Layer 1 ] ──► ReLU ──► [ Layer 2 ] ────────► [ ⊕ Add ] ──► ReLU ──► Output
                                                        F(x) + x
```

- Instead of learning direct mapping $\mathcal{H}(\mathbf{x})$, the network learns residual mapping $\mathcal{F}(\mathbf{x}) = \mathcal{H}(\mathbf{x}) - \mathbf{x}$.
- Gradients flow unobstructed backwards through the shortcut $+1$ connection, enabling training of networks with **100–1000+ layers**.

---

## 5. Sequence Models & Recurrence

### The Vanishing Gradient Problem in RNNs

Standard Recurrent Neural Networks (RNNs) share weights across time steps $t=1 \dots T$:

$$h_t = \tanh(W_{hh} h_{t-1} + W_{xh} x_t + b)$$

When calculating gradients across 100 time steps, the repeated multiplication of $W_{hh}^T$ causes gradients to vanish to $0$ or explode to $\infty$, making RNNs unable to remember long-term context.

---

### LSTM (Long Short-Term Memory)

LSTMs solve vanishing gradients by maintaining a constant **Cell State ($C_t$)** regulated by 3 gates:

```
                  ┌───────────────── Cell State (Cₜ) ────────────────┐
                  │                                                 │
                  ▼                                                 ▼
   xₜ ──► [ Forget Gate (fₜ) ] ──► [ Input Gate (iₜ) ] ──► [ Output Gate (oₜ) ] ──► Hidden State (hₜ)
          "What to erase"          "What new info to save" "What to emit"
```

1. **Forget Gate:** $f_t = \sigma(W_f [h_{t-1}, x_t] + b_f) \in [0, 1]$
2. **Input Gate:** $i_t = \sigma(W_i [h_{t-1}, x_t] + b_i)$ and $\tilde{C}_t = \tanh(W_c [h_{t-1}, x_t] + b_c)$
3. **Update Cell State:** $C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$
4. **Output Gate:** $o_t = \sigma(W_o [h_{t-1}, x_t] + b_o)$ and $h_t = o_t \odot \tanh(C_t)$

---

## 6. The Foundation of Transformers: Self-Attention

### Queries, Keys, and Values (Q, K, V)

The Attention mechanism models relationships dynamically. Think of it like a database retrieval system:
- **Query ($Q$):** What I am looking for (the current token).
- **Key ($K$):** The index tag of each item (all tokens in the sequence).
- **Value ($V$):** The actual content of each item.

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V$$

```
    Queries (Q) ──┐
                  ├──► [ Matrix Multiply (Q · Kᵀ) ] ──► [ Scale by 1/√dₖ ] ──► [ Softmax (Weights) ]
    Keys (K) ────┘                                                                    │
                                                                                      ▼
    Values (V) ─────────────────────────────────────────────────────────────► [ Weighted Sum ]
                                                                                      │
                                                                                      ▼
                                                                               Context Vectors
```

- **Why divide by $\sqrt{d_k}$?** Without scaling, for high dimensions $d_k = 128$, dot products grow large, pushing the Softmax into regions with extremely tiny gradients (vanishing gradient during attention).

---

[➡️ Continue to Module 04: LLMs, Transformers & GenAI](./04-llms-transformers-genai.md)
