[🏠 Back to AI Index](./README.md) • [Next: Machine Learning Core ➡️](./02-machine-learning-core.md)

<div align="center">
  <h1>01. Math Foundations & AI Basics</h1>
  <p><b>The Mathematical Backbone of Machine Learning and Classical AI Algorithms</b></p>
</div>

---

## 📑 Module Index
- [1. Linear Algebra for AI](#1-linear-algebra-for-ai)
  - [Vectors, Matrices & Tensors](#vectors-matrices--tensors)
  - [Dot Product & Geometric Intuition](#dot-product--geometric-intuition)
  - [Cosine Similarity (The Engine of Vector Search)](#cosine-similarity)
  - [Matrix Multiplication & Dimensional Transformation](#matrix-multiplication)
  - [Eigenvalues & Eigenvectors](#eigenvalues--eigenvectors)
- [2. Calculus & Optimization](#2-calculus--optimization)
  - [Derivatives & Partial Derivatives](#derivatives--partial-derivatives)
  - [The Gradient Vector & Steepest Descent](#the-gradient-vector)
  - [The Chain Rule (Foundation of Backpropagation)](#the-chain-rule)
- [3. Probability & Statistics for Machine Learning](#3-probability--statistics)
  - [Random Variables & Probability Distributions](#random-variables--distributions)
  - [Bayes' Theorem & Conditional Probability](#bayes-theorem)
  - [Expectation, Variance & Standard Deviation](#expectation-variance--std)
  - [Maximum Likelihood Estimation (MLE)](#maximum-likelihood-estimation-mle)
- [4. Classical AI & Search Algorithms](#4-classical-ai--search-algorithms)
  - [State Space Representation & Search Problems](#state-space-search)
  - [Uninformed vs Informed Search (BFS, DFS, A*)](#informed-search--a-algorithm)
  - [Adversarial Search (Minimax & Alpha-Beta Pruning)](#adversarial-search)

---

## 1. Linear Algebra for AI

Linear algebra is the language of machine learning. Everything in modern AI—from tabular features to image pixels and word embeddings—is represented as vectors and matrices.

### Vectors, Matrices & Tensors

```
Scalar (0D Tensor)   Vector (1D Tensor)      Matrix (2D Tensor)       Tensor (3D+ Tensor)
     [ 42 ]              [ 1.2 ]               [ 1  2  3 ]               [ [ [1, 2],
                         [ 3.4 ]               [ 4  5  6 ]                   [3, 4] ],
                         [ 5.6 ]                                           [ [5, 6],
                                                                             [7, 8] ] ]
```

- **Scalar ($x \in \mathbb{R}$):** A single real number (e.g., Temperature = $24.5$).
- **Vector ($\mathbf{v} \in \mathbb{R}^n$):** A 1D array of numbers representing a point in $n$-dimensional space or a list of features (e.g., `[Age, Income, CreditScore]`).
- **Matrix ($\mathbf{A} \in \mathbb{R}^{m \times n}$):** A 2D grid with $m$ rows (samples) and $n$ columns (features).
- **Tensor ($\mathbf{T} \in \mathbb{R}^{B \times C \times H \times W}$):** Multi-dimensional array (e.g., Batch of RGB Images: $\text{Batch} \times \text{Channels} \times \text{Height} \times \text{Width}$).

---

### Dot Product & Geometric Intuition

The **dot product** between two vectors $\mathbf{a} = [a_1, \dots, a_n]$ and $\mathbf{b} = [b_1, \dots, b_n]$ is:

$$\mathbf{a} \cdot \mathbf{b} = \sum_{i=1}^n a_i b_i = \|\mathbf{a}\| \|\mathbf{b}\| \cos(\theta)$$

```
       b
       ▲
       │ \
       │   \
       │  θ  \
       └───────► a
     Projection
```

- If $\theta = 0^\circ$ ($\cos \theta = 1$): Vectors point in the **exact same direction** (Maximum similarity).
- If $\theta = 90^\circ$ ($\cos \theta = 0$): Vectors are **orthogonal / independent** (Zero correlation).
- If $\theta = 180^\circ$ ($\cos \theta = -1$): Vectors point in **opposite directions**.

---

### Cosine Similarity

Used ubiquitously in **Vector Databases, Search Engines, and LLM Embeddings** to calculate how close two pieces of text are in semantic space regardless of magnitude:

$$\text{Cosine Similarity}(\mathbf{u}, \mathbf{v}) = \frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\|_2 \|\mathbf{v}\|_2} = \frac{\sum u_i v_i}{\sqrt{\sum u_i^2} \sqrt{\sum v_i^2}}$$

```python
import numpy as np

def cosine_similarity(u: np.ndarray, v: np.ndarray) -> float:
    return np.dot(u, v) / (np.linalg.norm(u) * np.linalg.norm(v))

vec_king = np.array([0.9, 0.2, 0.8])
vec_queen = np.array([0.88, 0.22, 0.82])
print(f"Similarity: {cosine_similarity(vec_king, vec_queen):.4f}") # ~0.9998
```

---

### Matrix Multiplication

If $\mathbf{A}$ has shape $(m \times k)$ and $\mathbf{B}$ has shape $(k \times n)$, the product $\mathbf{C} = \mathbf{A}\mathbf{B}$ has shape $(m \times n)$:

$$C_{ij} = \sum_{r=1}^k A_{ir} B_{rj}$$

In Neural Networks, a dense layer transformation is:
$$\mathbf{y} = \sigma(\mathbf{X}\mathbf{W} + \mathbf{b})$$
where $\mathbf{X}$ is the input batch $(B \times d_{in})$, $\mathbf{W}$ is the weight matrix $(d_{in} \times d_{out})$, and $\mathbf{b}$ is the bias vector $(1 \times d_{out})$.

---

### Eigenvalues & Eigenvectors

For a square matrix $\mathbf{A}$, if multiplying by vector $\mathbf{v}$ only scales the vector by scalar $\lambda$ without changing its direction:

$$\mathbf{A}\mathbf{v} = \lambda \mathbf{v}$$

- $\mathbf{v}$ is the **Eigenvector** (Principal direction of variance).
- $\lambda$ is the **Eigenvalue** (Magnitude of stretch along that direction).
- **Application:** **Principal Component Analysis (PCA)** uses eigenvectors of the covariance matrix to compress data from 1,000 dimensions down to 2 or 3 essential axes while retaining maximum information.

---

## 2. Calculus & Optimization

### Derivatives & Partial Derivatives

- **Derivative ($\frac{df}{dx}$):** Instantaneous rate of change of function $f(x)$ with respect to $x$.
- **Partial Derivative ($\frac{\partial f}{\partial x_i}$):** Derivative of multi-variable function with respect to one variable while holding all other variables constant.

$$\text{For } f(x, y) = 3x^2 y + 5y^3 \implies \frac{\partial f}{\partial x} = 6xy, \quad \frac{\partial f}{\partial y} = 3x^2 + 15y^2$$

---

### The Gradient Vector

The **Gradient** $\nabla f$ is the vector of all partial derivatives pointing in the direction of **steepest ascent**:

$$\nabla f(\mathbf{x}) = \begin{bmatrix} \frac{\partial f}{\partial x_1}, & \frac{\partial f}{\partial x_2}, & \dots, & \frac{\partial f}{\partial x_n} \end{bmatrix}^T$$

**Gradient Descent Rule:** To minimize a Loss function $L(\mathbf{w})$, take steps in the *opposite* direction of the gradient:
$$\mathbf{w}_{t+1} = \mathbf{w}_t - \alpha \nabla L(\mathbf{w}_t)$$
where $\alpha$ is the **learning rate**.

---

### The Chain Rule

If $y = g(u)$ and $u = h(x)$, then:

$$\frac{dy}{dx} = \frac{dy}{du} \cdot \frac{du}{dx}$$

In Deep Neural Networks with 50 layers, **Backpropagation** repeatedly applies the multivariate chain rule from the final loss output back to every single weight:

$$\frac{\partial L}{\partial w_1} = \frac{\partial L}{\partial a_3} \cdot \frac{\partial a_3}{\partial z_3} \cdot \frac{\partial z_3}{\partial a_2} \cdot \frac{\partial a_2}{\partial z_2} \cdot \frac{\partial z_2}{\partial a_1} \cdot \frac{\partial a_1}{\partial z_1} \cdot \frac{\partial z_1}{\partial w_1}$$

---

## 3. Probability & Statistics

### Bayes' Theorem

Calculates the **posterior probability** of an event given prior beliefs and new evidence:

$$P(A \mid B) = \frac{P(B \mid A) \cdot P(A)}{P(B)}$$

- $P(A \mid B)$: **Posterior probability** (Probability of disease given positive test).
- $P(B \mid A)$: **Likelihood** (Probability test is positive given patient actually has disease).
- $P(A)$: **Prior probability** (General incidence of disease in population).
- $P(B)$: **Evidence / Marginal probability** (Total probability of testing positive).

---

### Expectation & Variance

- **Expected Value (Mean $\mu$):**
  $$\mathbb{E}[X] = \sum x_i P(x_i) \quad \text{or} \quad \int x f(x) dx$$
- **Variance ($\sigma^2$):** Measures the spread or dispersion around the mean:
  $$\text{Var}(X) = \mathbb{E}[(X - \mu)^2] = \mathbb{E}[X^2] - (\mathbb{E}[X])^2$$
- **Standard Deviation ($\sigma$):** $\sqrt{\text{Var}(X)}$ (in original units).

---

### Maximum Likelihood Estimation (MLE)

MLE seeks parameter values $\hat{\theta}$ that maximize the likelihood of observing the collected dataset $\mathcal{D}$:

$$\hat{\theta}_{\text{MLE}} = \arg\max_\theta \sum_{i=1}^N \log P(x_i \mid \theta)$$

- Minimizing **Mean Squared Error (MSE)** in Linear Regression is mathematically identical to MLE under Gaussian noise assumption!
- Minimizing **Binary Cross-Entropy Loss** in Logistic Regression is mathematically identical to MLE for a Bernoulli distribution!

---

## 4. Classical AI & Search Algorithms

### State Space Search

An AI Agent formulates problems as:
1. **Initial State ($S_0$):** Starting situation.
2. **Actions ($A(s)$):** Valid moves from state $s$.
3. **Transition Model ($Result(s, a)$):** Outcome state after action.
4. **Goal Test ($IsGoal(s)$):** Check if target reached.
5. **Path Cost ($c(s, a, s')$):** Step cost (e.g., fuel, distance, time).

---

### Informed Search: $A^*$ Algorithm

The **$A^*$ (A-Star) search** is the gold-standard shortest-path finding algorithm combining actual cost and heuristic estimate:

$$f(n) = g(n) + h(n)$$

- $g(n)$: Exact cost incurred so far from start node to node $n$.
- $h(n)$: Estimated heuristic cost from node $n$ to the goal.
- **Admissibility Condition:** $h(n)$ must **never overestimate** the true remaining cost ($h(n) \le h^*(n)$) to guarantee an optimal path.

```
Start (0) ──► Node A [g=2, h=5 -> f=7] ──► Goal [g=8, h=0 -> f=8]
           └─► Node B [g=4, h=2 -> f=6] ──► Goal [g=7, h=0 -> f=7]  <-- A* explores B first!
```

---

### Adversarial Search: Minimax & Alpha-Beta Pruning

Used in turn-based 2-player games (Chess, Tic-Tac-Toe, Checkers):

```
       MAX (Player)         [ 5 ]
                          /       \
       MIN (Opponent)   [ 5 ]     [ 2 ]
                        /   \     /   \
       LEAF NODES      5     9   2     1
```

- **Minimax:** MAX player tries to maximize score; MIN player tries to minimize score.
- **Alpha-Beta Pruning:** Eliminates entire subtrees that cannot influence the final decision, reducing search complexity from $O(b^d)$ to $O(b^{d/2})$:
  - $\alpha$: Highest score MAX is guaranteed so far.
  - $\beta$: Lowest score MIN is guaranteed so far.
  - Prune when $\beta \le \alpha$.

---

[➡️ Continue to Module 02: Machine Learning Core](./02-machine-learning-core.md)
