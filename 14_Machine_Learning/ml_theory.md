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

*End of Machine Learning Theory — 10 questions covering Bias-Variance, Evaluation, Cross-Validation, Regularization, Ensembles, and Dimensionality Reduction.*
