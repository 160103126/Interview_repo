# Machine Learning — Theory & Concepts

> Core classical Machine Learning concepts covering Bias-Variance, Evaluation Metrics, Regression vs Classification, and ensemble methods.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is the Bias-Variance Tradeoff?

**Answer:**

It is the fundamental problem in supervised learning regarding how well a model generalizes to new, unseen data.

- **Bias:** Error introduced by approximating a real-world problem with a simplified model.
  - *High Bias = Underfitting.* The model is too simple (e.g., trying to fit a straight line to a curve).
- **Variance:** Error introduced by the model being too sensitive to small fluctuations in the training data.
  - *High Variance = Overfitting.* The model memorizes the training data noise but fails on new data.

**The Tradeoff:**
Decreasing bias usually increases variance, and vice versa. The goal is to find the "sweet spot" where total error is minimized.

---

### Q2: Explain Supervised, Unsupervised, and Reinforcement Learning.

**Answer:**

- **Supervised Learning:** The model is trained on labeled data (input-output pairs). E.g., Predicting house prices (output) based on square footage (input).
  - *Tasks:* Regression, Classification.
- **Unsupervised Learning:** The model is trained on unlabeled data. It must find hidden patterns or structures on its own.
  - *Tasks:* Clustering (K-Means), Dimensionality Reduction (PCA), Anomaly Detection.
- **Reinforcement Learning (RL):** An agent learns to make decisions by performing actions in an environment and receiving rewards or penalties.
  - *Tasks:* Game playing (AlphaGo), Robotics, Autonomous driving.

---

### Q3: What is the difference between Classification and Regression?

**Answer:**

Both are Supervised Learning tasks, but their outputs differ:

- **Classification:** Predicts a discrete class label or category.
  - *Example:* Is this email Spam or Not Spam? (Binary Classification). Is this animal a cat, dog, or bird? (Multiclass Classification).
  - *Algorithms:* Logistic Regression, SVM, Random Forest Classifier.
- **Regression:** Predicts a continuous numerical value.
  - *Example:* Predicting the price of a house, forecasting tomorrow's temperature.
  - *Algorithms:* Linear Regression, Ridge Regression, Random Forest Regressor.

---

## 🟡 Medium (Intermediate)

### Q4: Explain the Evaluation Metrics for Classification (Precision, Recall, F1-Score).

**Answer:**

Accuracy (Total Correct / Total Predictions) is misleading for imbalanced datasets (e.g., 99% of transactions are non-fraud. A model that always guesses "non-fraud" is 99% accurate but useless).

We use the Confusion Matrix (True Positives, False Positives, True Negatives, False Negatives):

- **Precision:** Out of all cases the model *predicted* as Positive, how many were actually Positive? `TP / (TP + FP)`
  - *Use when:* False Positives are expensive (e.g., Spam filter. You don't want to send a crucial email to the spam folder).
- **Recall (Sensitivity):** Out of all *actual* Positive cases, how many did the model find? `TP / (TP + FN)`
  - *Use when:* False Negatives are expensive (e.g., Cancer detection. It's better to false-alarm than to miss a tumor).
- **F1-Score:** The harmonic mean of Precision and Recall. `2 * (Precision * Recall) / (Precision + Recall)`
  - *Use when:* You need a single metric that balances both Precision and Recall, especially on imbalanced datasets.

---

### Q5: What is Cross-Validation (K-Fold)? Why use it?

**Answer:**

Instead of a single Train/Test split (e.g., 80/20), K-Fold Cross-Validation splits the dataset into `K` equally sized "folds".

**Process:**
1. Train the model on `K-1` folds.
2. Test the model on the 1 remaining fold.
3. Repeat this process `K` times, using a different fold as the test set each time.
4. Average the performance metrics across all `K` trials.

**Why use it?**
It provides a much more robust and reliable estimate of model performance. It ensures the model's accuracy isn't just a lucky result of a particularly favorable train/test split.

---

### Q6: Explain L1 (Lasso) and L2 (Ridge) Regularization.

**Answer:**

Regularization prevents overfitting by adding a penalty term to the loss function, discouraging the model from assigning large weights to features.

- **L1 Regularization (Lasso):** Adds the *absolute value* of the weights to the loss function.
  - *Effect:* It can shrink some weights exactly to zero.
  - *Use case:* **Feature Selection.** It naturally removes unimportant features, creating a sparse model.
- **L2 Regularization (Ridge):** Adds the *squared magnitude* of the weights to the loss function.
  - *Effect:* It shrinks weights towards zero, but rarely exactly to zero.
  - *Use case:* Handling multicollinearity (highly correlated features) and keeping all features but reducing their individual impact.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: Explain Bagging vs Boosting (Ensemble Methods). Give examples.

**Answer:**

Ensemble methods combine multiple weak models to create one strong model.

**Bagging (Bootstrap Aggregating):**
- **How it works:** Trains multiple independent models in parallel on random subsets of the data (with replacement). The final prediction is an average (Regression) or majority vote (Classification).
- **Goal:** Reduces **Variance** (prevents overfitting).
- **Classic Algorithm:** **Random Forest** (an ensemble of Decision Trees).

**Boosting:**
- **How it works:** Trains models *sequentially*. Each new model focuses specifically on the errors (residuals) made by the previous models.
- **Goal:** Reduces **Bias** (creates a highly accurate model).
- **Classic Algorithms:** AdaBoost, Gradient Boosting, **XGBoost**, LightGBM.

---

### Q8: How does Gradient Descent work? What are the variants (Batch, SGD, Mini-batch)?

**Answer:**

Gradient Descent is an optimization algorithm used to minimize the Loss Function by iteratively moving in the direction of steepest descent (the negative gradient).

**Formula:** `new_weight = old_weight - (learning_rate * gradient)`

**Variants:**
1. **Batch Gradient Descent:** Calculates the error for the *entire* dataset before updating weights. (Slow, memory-heavy, but stable convergence).
2. **Stochastic Gradient Descent (SGD):** Calculates error and updates weights for *one single data point* at a time. (Very fast, low memory, but highly erratic/noisy convergence).
3. **Mini-batch Gradient Descent:** The standard compromise. Updates weights after evaluating a small batch of data (e.g., 32 or 64 samples). Utilizes matrix multiplication optimizations on GPUs.

---

### Q9: What is the "Curse of Dimensionality" and how do you solve it?

**Answer:**

As the number of features (dimensions) increases, the volume of the feature space increases exponentially.
Data becomes sparse. To maintain the same statistical significance, the amount of training data needed grows exponentially. Distance metrics (like Euclidean distance used in K-Means or KNN) lose meaning because the distance between *any* two points approaches the same value.

**Solutions (Dimensionality Reduction):**
1. **Feature Selection:** Removing irrelevant/redundant features (using L1 Lasso, Information Gain, or recursive feature elimination).
2. **Feature Extraction:** Projecting the data into a lower-dimensional space while retaining the most important information.
   - **PCA (Principal Component Analysis):** Linear transformation finding directions of maximum variance.
   - **t-SNE / UMAP:** Non-linear transformations, primarily used for visualizing high-dimensional data in 2D/3D.

---

### Q10: How do Support Vector Machines (SVM) handle non-linear data? (The Kernel Trick)

**Answer:**

A standard SVM finds a linear hyperplane that maximizes the margin between two classes. However, many real-world datasets are not linearly separable.

**The Kernel Trick:**
Instead of manually creating complex polynomial features, the Kernel function implicitly maps the input data into a much higher-dimensional space where it *becomes* linearly separable.

It computes the dot products between data points in this higher-dimensional space *without ever explicitly computing the coordinates* in that space, making it computationally efficient.

**Common Kernels:**
- **Polynomial:** Maps to polynomial space.
- **RBF (Radial Basis Function):** Maps to an infinite-dimensional space. (The default and most powerful).

---

### Q11: What is Feature Engineering? Give common techniques.

**Answer:**

Feature Engineering is the process of creating new input variables from raw data to improve model performance. It is often the single biggest factor in winning Kaggle competitions and building production ML systems.

**Common Techniques:**
1. **One-Hot Encoding:** Convert categorical variables (e.g., "Red", "Blue", "Green") into binary columns.
2. **Binning/Bucketing:** Convert continuous variables into categories (e.g., Age → "Young", "Middle", "Senior").
3. **Log/Power Transforms:** Apply `log(x)` to heavily skewed features (e.g., income) to make them more normally distributed.
4. **Interaction Features:** Create new features by combining existing ones (e.g., `price_per_sqft = price / sqft`).
5. **Time-based Features:** Extract `day_of_week`, `hour`, `is_weekend`, `month` from timestamps.
6. **Rolling/Lag Features:** For time series: `avg_revenue_last_7_days`, `revenue_yesterday`.
7. **Target Encoding:** Replace a category with the mean of the target variable for that category (careful: causes leakage if not done with cross-validation).

---

### Q12: How do you handle Imbalanced Datasets?

**Answer:**

If 99% of transactions are legitimate and 1% are fraud, a model that always predicts "not fraud" achieves 99% accuracy but is completely useless.

**Data-Level Techniques:**
1. **Oversampling (SMOTE):** Synthetically generates new minority class examples by interpolating between existing ones. (Not recommended for image/NLP data).
2. **Undersampling:** Randomly removes majority class examples. (Risk: loses valuable data).
3. **Combined:** SMOTE + Tomek Links (remove borderline majority examples).

**Algorithm-Level Techniques:**
1. **Class Weights:** Most classifiers (Logistic Regression, XGBoost, PyTorch) support a `class_weight="balanced"` parameter that penalizes misclassifying the minority class more heavily.
2. **Focal Loss:** Used in object detection (RetinaNet). It down-weights easy, well-classified examples and focuses training on hard, misclassified ones.

**Evaluation:**
- **Never use Accuracy.** Use Precision, Recall, F1-Score, or AUC-ROC.
- Use **Stratified K-Fold Cross-Validation** to ensure each fold has the same class ratio.

---

### Q13: Explain Hyperparameter Tuning Strategies.

**Answer:**

Hyperparameters (learning rate, number of trees, regularization strength) are NOT learned during training. You must search for the best values.

1. **Grid Search:** Exhaustively tries every combination of hyperparameters from a predefined grid. Guaranteed to find the best combo in the grid but extremely slow. (10 params × 5 values each = 10^5 combinations).
2. **Random Search:** Randomly samples hyperparameter combinations. Surprisingly effective — it explores the space more efficiently than Grid Search because many hyperparameters are irrelevant (random search doesn't waste time varying them).
3. **Bayesian Optimization (Optuna, HyperOpt):** Uses a probabilistic model (Gaussian Process) to predict which hyperparameters will yield the best score. It intelligently focuses on promising regions of the search space, converging much faster.

**Best Practice:** Use Random Search for initial exploration, then Bayesian Optimization to refine.

---

### Q14: Explain Decision Trees. What are Gini Impurity and Information Gain?

**Answer:**

A Decision Tree is a flowchart-like structure where each internal node tests a feature, each branch represents a decision, and each leaf node is a prediction.

**How splits are chosen:**
The algorithm iterates through every feature and every threshold, selecting the split that maximizes "purity" (homogeneity) of the child nodes.

- **Gini Impurity:** `Gini = 1 - Σ(p_i)²` where `p_i` is the proportion of class `i` in the node. A pure node (all one class) has Gini = 0. A 50/50 node has Gini = 0.5.
- **Information Gain (Entropy-based):** `IG = Entropy(parent) - Weighted_Avg(Entropy(children))`. Entropy = `-Σ p_i * log2(p_i)`. Measures the reduction in uncertainty after the split.

**Drawback:** Single Decision Trees overfit massively on noisy data. This is why we use ensembles (Random Forest, XGBoost).

---

### Q15: How does Random Forest improve over a single Decision Tree?

**Answer:**

Random Forest is a **Bagging** ensemble of Decision Trees.

**How it works:**
1. Create `N` Decision Trees (e.g., 100).
2. Each tree is trained on a **random bootstrap sample** of the data (sampling with replacement — each tree sees a different ~63% of the data).
3. At each split, each tree only considers a **random subset of features** (not all features). This decorrelates the trees.
4. **Prediction:** Average (regression) or majority vote (classification) across all 100 trees.

**Why it works:**
- Individual trees overfit (high variance, low bias).
- Averaging 100 noisy, decorrelated trees cancels out the individual errors, dramatically reducing variance.
- It is extremely resistant to overfitting — one of the few algorithms where adding more trees almost never hurts.

---

### Q16: What makes XGBoost so powerful? (Gradient Boosting internals)

**Answer:**

XGBoost (eXtreme Gradient Boosting) is the most dominant algorithm for structured/tabular data (Kaggle, industry).

**Core Idea:**
Unlike Random Forest (parallel trees), XGBoost trains trees **sequentially**. Each new tree learns to predict the **residual errors** (mistakes) of the combined previous trees.

**Why XGBoost wins:**
1. **Regularization:** L1 and L2 penalties on the tree leaf weights prevent overfitting (regular Gradient Boosting has no regularization).
2. **Approximate Split Finding:** Uses histogram-based bucketing to find optimal splits efficiently on massive datasets.
3. **Missing Value Handling:** Natively handles missing values. During training, it learns the optimal direction (left or right) for missing values at each split.
4. **System Optimization:** Parallelized tree construction, cache-aware access patterns, and out-of-core computing for datasets larger than RAM.

---

### Q17: Explain K-Nearest Neighbors (KNN). What are its weaknesses?

**Answer:**

KNN is a non-parametric, instance-based algorithm. It stores the entire training dataset and makes predictions by finding the K closest data points to the new input.

**How it works:**
1. Calculate the distance (Euclidean, Manhattan, or Cosine) between the new data point and ALL training points.
2. Select the K nearest neighbors.
3. Classification: Majority vote. Regression: Average of K neighbors' values.

**Weaknesses:**
1. **Curse of Dimensionality:** In high-dimensional spaces (100+ features), all distances converge, making KNN useless. (Must use PCA/dimensionality reduction first).
2. **Computational Cost:** O(N*D) per prediction — must compute distance to every training point. Not scalable for large datasets.
3. **Feature Scaling Required:** Features with larger ranges dominate the distance calculation. Must normalize/standardize features first.
4. **Sensitive to K:** Too small K (K=1) → overfitting. Too large K → underfitting. Must tune via cross-validation.

---

### Q18: Explain Naive Bayes. Why is it "Naive"?

**Answer:**

Naive Bayes is a probabilistic classifier based on Bayes' Theorem:
`P(Class | Features) = P(Features | Class) * P(Class) / P(Features)`

**The "Naive" Assumption:**
It assumes all features are **conditionally independent** given the class. This is almost never true in reality (e.g., in email spam detection, the words "free" and "winner" are highly correlated). Despite this unrealistic assumption, Naive Bayes works surprisingly well in practice.

**Variants:**
- **Gaussian NB:** For continuous features (assumes normal distribution).
- **Multinomial NB:** For count data (text classification, word counts). The standard for NLP with Bag-of-Words.
- **Bernoulli NB:** For binary features.

**Why it's still used:**
Extremely fast (O(N*D) training), works well with small datasets, serves as a strong baseline for text classification, and handles high-dimensional data (many features) gracefully.

---

### Q19: How do you choose between different ML models? (Model Selection)

**Answer:**

There is no single "best" model. Use this decision framework:

| Data Characteristics | Recommended Models |
|---|---|
| **Small dataset (< 10K rows)** | Logistic Regression, SVM, Random Forest |
| **Large tabular data** | XGBoost, LightGBM, CatBoost |
| **Text data** | BERT, DistilBERT (fine-tuned), or TF-IDF + Logistic Regression as baseline |
| **Image data** | CNN (ResNet, EfficientNet) with Transfer Learning |
| **Time series** | ARIMA (simple), LSTM/Transformer (complex) |
| **Need interpretability** | Decision Tree, Logistic Regression, SHAP on any model |
| **Need a quick baseline** | Always start with Logistic Regression (classification) or Linear Regression (regression) |

**The Process:**
1. Start with a simple baseline (Logistic Regression).
2. Try a tree-based ensemble (XGBoost).
3. If still underperforming, try Deep Learning.
4. Compare using cross-validated metrics on a held-out test set.

---

### Q20: What is SHAP? How do you explain model predictions?

**Answer:**

SHAP (SHapley Additive exPlanations) is the gold standard for model interpretability. Based on game theory, it assigns each feature a contribution score for every individual prediction.

**How it works:**
For a single prediction (e.g., "Why was this loan application rejected?"), SHAP computes how much each feature pushed the prediction away from the baseline (average) prediction.

**Example output:**
- `income = -$50K` → pushed prediction toward "Reject" (most important factor)
- `credit_score = 720` → pushed prediction toward "Approve"
- `debt_ratio = 0.6` → pushed prediction toward "Reject"

**Why it matters for MAANG interviews:**
- **Regulatory Compliance:** Laws (GDPR, ECOA) require explainability for automated decisions.
- **Model Debugging:** SHAP reveals if the model is using spurious features (e.g., relying on "zip code" as a proxy for race).
- **Stakeholder Trust:** Non-technical stakeholders need to understand why the model makes certain decisions.

---

*End of Machine Learning Theory — 20 comprehensive questions covering Bias-Variance, Evaluation, Cross-Validation, Regularization, Ensembles, Dimensionality Reduction, Feature Engineering, Imbalanced Data, Hyperparameter Tuning, Decision Trees, Random Forest, XGBoost, KNN, Naive Bayes, Model Selection, and SHAP.*

