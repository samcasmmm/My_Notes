[🏠 Back to AI Index](./README.md) • [⬅️ Prev: Math & Basics](./01-math-foundations-ai-basics.md) • [Next: Deep Learning ➡️](./03-deep-learning-neural-networks.md)

<div align="center">
  <h1>02. Machine Learning Core</h1>
  <p><b>Supervised, Unsupervised, Ensembles, Feature Engineering, & Diagnostic Metrics</b></p>
</div>

---

## 📑 Module Index
- [1. Supervised Learning Algorithms](#1-supervised-learning-algorithms)
  - [Linear Regression & Normal Equation](#linear-regression)
  - [Logistic Regression & Decision Boundary](#logistic-regression)
  - [Decision Trees & Splitting Criteria (Gini, Entropy)](#decision-trees)
  - [Ensemble Methods: Bagging vs Boosting (Random Forest vs XGBoost)](#ensemble-methods)
  - [Support Vector Machines (SVM) & Kernel Trick](#support-vector-machines)
- [2. Unsupervised Learning Algorithms](#2-unsupervised-learning-algorithms)
  - [K-Means & Elbow Method](#k-means-clustering)
  - [DBSCAN (Density-Based Spatial Clustering)](#dbscan)
  - [PCA (Principal Component Analysis)](#pca-dimensionality-reduction)
- [3. Regularization & Loss Functions](#3-regularization--loss-functions)
  - [L1 Lasso vs L2 Ridge vs ElasticNet](#regularization-techniques)
  - [Core Loss Functions (MSE, Cross-Entropy, Huber, Hinge)](#loss-functions)
- [4. Feature Engineering & Preprocessing](#4-feature-engineering--preprocessing)
  - [Scaling, One-Hot Encoding, & Handling Missing Values](#data-preprocessing)
  - [Handling Imbalanced Datasets (SMOTE & Class Weights)](#imbalanced-data)
- [5. Model Diagnostics & Validation](#5-model-diagnostics--validation)
  - [K-Fold Stratified Cross-Validation](#cross-validation)
  - [Confusion Matrix, Precision, Recall, F1, ROC-AUC](#evaluation-metrics)
  - [Preventing Data Leakage in Production](#data-leakage)

---

## 1. Supervised Learning Algorithms

### Linear Regression

Models a linear relationship between input vector $\mathbf{x} \in \mathbb{R}^d$ and continuous target $y \in \mathbb{R}$:

$$\hat{y} = \mathbf{w}^T \mathbf{x} + b = w_0 + w_1 x_1 + w_2 x_2 + \dots + w_d x_d$$

#### Optimization via Mean Squared Error (MSE):
$$J(\mathbf{w}, b) = \frac{1}{2N} \sum_{i=1}^N (\hat{y}^{(i)} - y^{(i)})^2$$

#### Analytical Solution (Normal Equation):
$$\mathbf{w}^* = (\mathbf{X}^T \mathbf{X})^{-1} \mathbf{X}^T \mathbf{y}$$

---

### Logistic Regression

Used for **Binary Classification**. It maps any real-valued number into a probability between $0$ and $1$ using the **Sigmoid / Logistic Function**:

$$\sigma(z) = \frac{1}{1 + e^{-z}}, \quad \text{where } z = \mathbf{w}^T \mathbf{x} + b$$

$$P(y = 1 \mid \mathbf{x}) = \sigma(\mathbf{w}^T \mathbf{x} + b)$$

```
     1.0 ┼───────────────────────╭────────
         │                      / 
     0.5 ┼ - - - - - - - - - - ╭┴- - - - - - Threshold = 0.5
         │                    /
     0.0 ┼─────────╮─────────╯────────────
                  -4   -2   0    2    4
```

- **Decision Rule:** Predict $\hat{y} = 1$ if $P(y=1 \mid \mathbf{x}) \ge 0.5$, else $\hat{y} = 0$.
- **Cost Function (Binary Cross-Entropy / Log Loss):**
  $$J(\mathbf{w}) = -\frac{1}{N} \sum_{i=1}^N \left[ y^{(i)} \log(\hat{y}^{(i)}) + (1 - y^{(i)}) \log(1 - \hat{y}^{(i)}) \right]$$

---

### Decision Trees

Non-parametric models that recursively split feature space into rectangular decision regions based on purity metrics.

```
                  [ Credit Score > 650? ]
                       /           \
                    YES             NO
                   /                 \
        [ Income > $50k? ]       [ REJECT ]
            /        \
          YES         NO
         /             \
    [ APPROVE ]    [ REVIEW ]
```

#### Splitting Criteria:
1. **Gini Impurity (CART Algorithm - default in Scikit-Learn):**
   $$Gini(S) = 1 - \sum_{k=1}^C p_k^2$$
   ($0.0 = \text{pure node where all samples belong to 1 class}$).
2. **Entropy & Information Gain (ID3 / C4.5):**
   $$H(S) = -\sum_{k=1}^C p_k \log_2(p_k)$$
   $$\text{Gain}(S, A) = H(S) - \sum_{v \in \text{Values}(A)} \frac{|S_v|}{|S|} H(S_v)$$

---

### Ensemble Methods: Bagging vs Boosting

```
┌───────────────────────────────────────────────┬───────────────────────────────────────────────┐
│              BAGGING (Parallel)               │             BOOSTING (Sequential)             │
│        (Example: Random Forest)               │     (Example: XGBoost, LightGBM, CatBoost)    │
├───────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ • Trains multiple trees in **parallel**       │ • Trains trees in **sequence**                │
│ • Each tree trained on a random bootstrap     │ • Each new tree focuses on errors/residuals   │
│   sample of data + random subset of features. │   made by previous trees.                     │
│ • **Reduces Variance** (prevents overfitting).│ • **Reduces Bias** (creates powerful models). │
│ • Final output = Majority vote / Average.     │ • Final output = Weighted sum of all trees.   │
└───────────────────────────────────────────────┴───────────────────────────────────────────────┘
```

#### Code Example (Scikit-Learn vs XGBoost):
```python
from sklearn.ensemble import RandomForestClassifier
from xgboost import XGBClassifier

# Random Forest (Bagging)
rf_model = RandomForestClassifier(n_estimators=100, max_depth=10, random_state=42)
rf_model.fit(X_train, y_train)

# XGBoost (Gradient Boosting)
xgb_model = XGBClassifier(n_estimators=150, learning_rate=0.05, max_depth=6, eval_metric='logloss')
xgb_model.fit(X_train, y_train)
```

---

### Support Vector Machines (SVM)

Finds the maximum-margin hyperplane separating classes.

- **Hard Margin:** Linear separation without allowing misclassifications.
- **Soft Margin:** Allows some misclassifications using slack variables $\xi_i$ controlled by penalty parameter $C$.
- **The Kernel Trick:** Projects non-linearly separable points into higher-dimensional space where they become linearly separable without computing higher coordinates explicitly:
  - **RBF (Radial Basis Function / Gaussian):** $K(\mathbf{x}, \mathbf{z}) = \exp(-\gamma \|\mathbf{x} - \mathbf{z}\|^2)$
  - **Polynomial Kernel:** $K(\mathbf{x}, \mathbf{z}) = (\mathbf{x}^T \mathbf{z} + c)^d$

---

## 2. Unsupervised Learning Algorithms

### K-Means Clustering

Partitions $N$ unlabeled points into $K$ distinct non-overlapping clusters:

1. Randomly initialize $K$ centroids $\mu_1, \dots, \mu_K$.
2. **Assignment Step:** Assign each sample to its nearest centroid:
   $$c^{(i)} = \arg\min_j \|\mathbf{x}^{(i)} - \mu_j\|^2$$
3. **Update Step:** Recalculate centroids as the mean of all points in that cluster:
   $$\mu_j = \frac{1}{|S_j|} \sum_{i \in S_j} \mathbf{x}^{(i)}$$
4. Repeat until convergence.

- **Elbow Method:** Plot Inertia (Within-Cluster Sum of Squares) against $K$ to pick the optimal inflection point.

---

### DBSCAN

**Density-Based Spatial Clustering of Applications with Noise:**
- Does **not** require specifying number of clusters $K$ beforehand.
- Can discover arbitrary non-linear shapes (circles, crescents).
- Identifies **outliers / noise** automatically.
- **Parameters:** $\epsilon$ (neighborhood radius) and `minPts` (minimum samples to form dense region).

---

### PCA (Dimensionality Reduction)

1. Standardize the data ($z$-score).
2. Compute the $(d \times d)$ **Covariance Matrix** $\mathbf{\Sigma} = \frac{1}{N} \mathbf{X}^T \mathbf{X}$.
3. Compute **Eigenvectors** and **Eigenvalues** of $\mathbf{\Sigma}$.
4. Sort eigenvectors in descending order of eigenvalues.
5. Project original dataset $\mathbf{X}$ onto top $k$ principal components.

---

## 3. Regularization & Loss Functions

### Regularization Techniques

Adds a penalty to the loss function to shrink weights and prevent overfitting:

$$\text{Total Loss} = \text{Data Loss} + \lambda \cdot \Omega(\mathbf{w})$$

| Method | Penalty $\Omega(\mathbf{w})$ | Effect | When to Use |
| :--- | :--- | :--- | :--- |
| **L2 Ridge** | $\sum w_j^2$ | Shrinks all weights smoothly towards zero (never exactly 0). | When many features are correlated. |
| **L1 Lasso** | $\sum \|w_j\|$ | Forces coefficients of unimportant features to **exact 0** (Feature Selection). | When data is sparse with many irrelevant features. |
| **ElasticNet** | $\alpha L_1 + (1-\alpha) L_2$ | Combines both $L_1$ and $L_2$ penalties. | Best default when features are numerous & correlated. |

---

## 4. Feature Engineering & Preprocessing

### Preprocessing Checklist
1. **Handling Missing Values:**
   - Numerical: Impute with Median (robust to outliers) or KNNImputer.
   - Categorical: Impute with Mode or mark as `"Unknown"`.
2. **Feature Scaling:**
   - `StandardScaler`: $z = \frac{x - \mu}{\sigma}$ (Preferred for linear models, SVMs, Neural Networks).
   - `MinMaxScaler`: $x_{scaled} = \frac{x - x_{min}}{x_{max} - x_{min}} \in [0, 1]$ (Good for bounded ranges/images).
   - *Note:* Tree models (Random Forest, XGBoost) do not require feature scaling!
3. **Categorical Encoding:**
   - Nominal (no order): `OneHotEncoder(drop='first')`.
   - High Cardinality (>50 categories): Target Encoding with smoothing or Embeddings.

### Imbalanced Datasets

When one class dominates ($99\%$ Normal vs $1\%$ Fraud):
- **DO NOT rely on Accuracy.**
- **Techniques:**
  - **SMOTE (Synthetic Minority Over-sampling Technique):** Generates synthetic minority examples along lines connecting nearest neighbors.
  - **Class Weighting (`class_weight='balanced'`):** Multiplies loss of minority class errors by $\frac{N}{2 \cdot N_{\text{minority}}}$.

---

## 5. Model Diagnostics & Validation

### Confusion Matrix & Classification Metrics

```
                     Actual Positive (1)     Actual Negative (0)
Predicted Pos (1)  [ True Positive (TP)  ] [ False Positive (FP) (Type I Error)  ]
Predicted Neg (0)  [ False Negative (FN) ] [ True Negative (TN) (Type II Error) ]
```

- **Accuracy:** $\frac{TP + TN}{\text{Total}}$
- **Precision:** $\frac{TP}{TP + FP}$ *(High cost of False Alarm, e.g. Spam filter)*
- **Recall:** $\frac{TP}{TP + FN}$ *(High cost of Missing a positive, e.g. Cancer detection)*
- **F1-Score:** $2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$
- **ROC-AUC (Receiver Operating Characteristic):** Area under the TPR vs FPR curve across all probability thresholds ($1.0 = \text{ideal}$, $0.5 = \text{random coin toss}$).

---

### Preventing Data Leakage

> [!CAUTION]
> **Data Leakage** occurs when information from outside the training dataset (test data or future time) is used to create the model.

- **Rule 1:** Always split data into `train_test_split` **BEFORE** applying `StandardScaler`, `Imputer`, or `PCA`.
- **Rule 2:** Fit transformers on training set only (`scaler.fit_transform(X_train)`), and only transform on test set (`scaler.transform(X_test)`).
- **Rule 3 (Time-Series):** Never use random K-Fold cross-validation on time series. Always use `TimeSeriesSplit` (Walk-forward validation).

---

[➡️ Continue to Module 03: Deep Learning & Neural Networks](./03-deep-learning-neural-networks.md)
