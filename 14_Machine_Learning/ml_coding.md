# Machine Learning — Coding Patterns

> Pure NumPy implementations of foundational Machine Learning algorithms. MAANG interviews frequently require building these from scratch to prove you understand the underlying calculus and linear algebra, not just the scikit-learn API.

---

## Table of Contents

- [Q1: Linear Regression (Gradient Descent)](#q1-linear-regression-gradient-descent)
- [Q2: Logistic Regression](#q2-logistic-regression)
- [Q3: K-Nearest Neighbors (KNN)](#q3-k-nearest-neighbors-knn)
- [Q4: K-Means Clustering](#q4-k-means-clustering)
- [Q5: Decision Tree Split (Gini Impurity)](#q5-decision-tree-split-gini-impurity)
- [Q6: Principal Component Analysis (PCA)](#q6-principal-component-analysis-pca)

---

### Q1: Linear Regression (Gradient Descent)

**Problem:** Implement simple linear regression ($y = wx + b$) using Gradient Descent.

```python
import numpy as np

class LinearRegression:
    def __init__(self, learning_rate=0.01, n_iters=1000):
        self.lr = learning_rate
        self.n_iters = n_iters
        self.weights = None
        self.bias = None

    def fit(self, X, y):
        n_samples, n_features = X.shape
        self.weights = np.zeros(n_features)
        self.bias = 0

        for _ in range(self.n_iters):
            y_predicted = np.dot(X, self.weights) + self.bias
            
            # Calculate gradients (Derivatives of Mean Squared Error)
            dw = (1 / n_samples) * np.dot(X.T, (y_predicted - y))
            db = (1 / n_samples) * np.sum(y_predicted - y)

            # Update weights
            self.weights -= self.lr * dw
            self.bias -= self.lr * db

    def predict(self, X):
        return np.dot(X, self.weights) + self.bias
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Gradients:** The loss function is Mean Squared Error (MSE): $\frac{1}{N} \sum (y_{pred} - y)^2$. 
- Taking the partial derivative with respect to $w$ yields: $\frac{2}{N} \sum X(y_{pred} - y)$. (We drop the `2` in code as it scales into the learning rate).
- **Matrix Multiplication:** `np.dot(X.T, error)` calculates the gradient for every feature simultaneously without a `for` loop, utilizing highly optimized BLAS/LAPACK C libraries under the hood.

#### ⏱️ Complexity
- **Time Complexity:** $O(\text{iters} \times N \times F)$ where N is samples, F is features.
- **Space Complexity:** $O(F)$ to store the weights.

---

### Q2: Logistic Regression

**Problem:** Implement Logistic Regression for binary classification.

```python
import numpy as np

def sigmoid(x):
    return 1 / (1 + np.exp(-x))

class LogisticRegression:
    def __init__(self, learning_rate=0.01, n_iters=1000):
        self.lr = learning_rate
        self.n_iters = n_iters
        self.weights = None
        self.bias = None

    def fit(self, X, y):
        n_samples, n_features = X.shape
        self.weights = np.zeros(n_features)
        self.bias = 0

        for _ in range(self.n_iters):
            linear_model = np.dot(X, self.weights) + self.bias
            y_predicted = sigmoid(linear_model)

            # Gradients of Binary Cross Entropy loss
            dw = (1 / n_samples) * np.dot(X.T, (y_predicted - y))
            db = (1 / n_samples) * np.sum(y_predicted - y)

            self.weights -= self.lr * dw
            self.bias -= self.lr * db

    def predict(self, X):
        linear_model = np.dot(X, self.weights) + self.bias
        y_predicted = sigmoid(linear_model)
        return [1 if i > 0.5 else 0 for i in y_predicted]
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Sigmoid Function:** Maps any real number into a probability between $0$ and $1$. 
- **The Mathematical Miracle:** Despite the loss function changing from MSE to Binary Cross-Entropy (Log Loss), the resulting gradient equations for $dw$ and $db$ are mathematically *identical* to Linear Regression! The only difference is that $y_{predicted}$ is now passed through the sigmoid function first.

#### ⏱️ Complexity
- **Time Complexity:** $O(\text{iters} \times N \times F)$
- **Space Complexity:** $O(F)$

---

### Q3: K-Nearest Neighbors (KNN)

**Problem:** Implement the KNN classification algorithm.

```python
import numpy as np
from collections import Counter

def euclidean_distance(x1, x2):
    return np.sqrt(np.sum((x1 - x2) ** 2))

class KNN:
    def __init__(self, k=3):
        self.k = k

    def fit(self, X, y):
        self.X_train = X
        self.y_train = y

    def predict(self, X):
        return np.array([self._predict(x) for x in X])

    def _predict(self, x):
        # 1. Compute distances between x and all examples in the training set
        distances = [euclidean_distance(x, x_train) for x_train in self.X_train]
        
        # 2. Sort by distance and return indices of the first k neighbors
        k_indices = np.argsort(distances)[:self.k]
        
        # 3. Extract the labels of the k nearest neighbor training samples
        k_nearest_labels = [self.y_train[i] for i in k_indices]
        
        # 4. Return the most common class label
        most_common = Counter(k_nearest_labels).most_common(1)
        return most_common[0][0]
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Lazy Learning:** Notice that `fit()` does absolutely no math. It just saves the dataset to RAM. KNN is a "lazy learner." All computation happens during inference (`predict()`).
- **Distance Metric:** We use Pythagorean theorem (Euclidean distance) generalized to N-dimensions.
- **Majority Vote:** `collections.Counter` tallies the labels of the $K$ closest points and returns the mode.

#### ⏱️ Complexity
- **Time Complexity:** $O(1)$ for `fit`. $O(N \log N)$ per prediction (due to `argsort`).
- **Space Complexity:** $O(N \times F)$ to store the entire training set.

---

### Q4: K-Means Clustering

**Problem:** Implement K-Means clustering algorithm.

```python
import numpy as np

class KMeans:
    def __init__(self, K=5, max_iters=100):
        self.K = K
        self.max_iters = max_iters

    def predict(self, X):
        self.X = X
        self.n_samples, self.n_features = X.shape
        
        # 1. Initialize centroids randomly
        random_sample_idxs = np.random.choice(self.n_samples, self.K, replace=False)
        self.centroids = [self.X[idx] for idx in random_sample_idxs]

        for _ in range(self.max_iters):
            # 2. Assign clusters
            self.clusters = self._create_clusters(self.centroids)
            
            # 3. Calculate new centroids
            old_centroids = self.centroids
            self.centroids = self._get_centroids(self.clusters)
            
            # 4. Check for convergence
            if self._is_converged(old_centroids, self.centroids):
                break
                
        return self._get_cluster_labels(self.clusters)

    def _create_clusters(self, centroids):
        clusters = [[] for _ in range(self.K)]
        for idx, sample in enumerate(self.X):
            centroid_idx = self._closest_centroid(sample, centroids)
            clusters[centroid_idx].append(idx)
        return clusters

    def _closest_centroid(self, sample, centroids):
        distances = [np.sqrt(np.sum((sample - point)**2)) for point in centroids]
        return np.argmin(distances)

    def _get_centroids(self, clusters):
        centroids = np.zeros((self.K, self.n_features))
        for cluster_idx, cluster in enumerate(clusters):
            cluster_mean = np.mean(self.X[cluster], axis=0)
            centroids[cluster_idx] = cluster_mean
        return centroids

    def _is_converged(self, old_centroids, centroids):
        distances = [np.sqrt(np.sum((old_centroids[i] - centroids[i])**2)) for i in range(self.K)]
        return sum(distances) == 0

    def _get_cluster_labels(self, clusters):
        labels = np.empty(self.n_samples)
        for cluster_idx, cluster in enumerate(clusters):
            for sample_index in cluster:
                labels[sample_index] = cluster_idx
        return labels
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Expectation-Maximization (EM):** K-Means is a classic EM algorithm. 
  - **Expectation Step (`_create_clusters`):** Assume the centroids are correct, and assign every point to its closest centroid.
  - **Maximization Step (`_get_centroids`):** Assume the cluster assignments are correct, and recalculate the physical center of mass (mean) for each cluster to find the *new* optimal centroids.
- **Convergence:** When the centroids stop moving between iterations (distance = 0), the algorithm has converged into a local minimum.

#### ⏱️ Complexity
- **Time Complexity:** $O(\text{iters} \times N \times K \times F)$
- **Space Complexity:** $O(K \times F)$ for the centroids.

---

### Q5: Decision Tree Split (Gini Impurity)

**Problem:** Calculate the Gini Impurity of a dataset to determine the best split for a Decision Tree.

```python
import numpy as np

def gini_impurity(y):
    """Calculate Gini Impurity of labels y."""
    m = len(y)
    if m == 0: return 0
    
    # Get probability of each class
    _, counts = np.unique(y, return_counts=True)
    probabilities = counts / m
    
    # Gini = 1 - sum(p_i^2)
    return 1.0 - np.sum(probabilities ** 2)

def information_gain(y, y_left, y_right):
    """Calculate how much Gini impurity decreased after a split."""
    p = len(y_left) / len(y)
    return gini_impurity(y) - (p * gini_impurity(y_left) + (1 - p) * gini_impurity(y_right))

# Example:
y_parent = np.array([0, 0, 1, 1])          # Gini = 1 - (0.5^2 + 0.5^2) = 0.5
y_left = np.array([0, 0])                  # Gini = 1 - (1^2) = 0
y_right = np.array([1, 1])                 # Gini = 1 - (1^2) = 0

print("Gain:", information_gain(y_parent, y_left, y_right)) # Gain = 0.5
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Gini Impurity:** A measure of how often a randomly chosen element from the set would be incorrectly labeled if it was randomly labeled according to the distribution of labels in the subset. A pure node (all same class) has a Gini of 0.
- **Information Gain:** We calculate the Gini of the parent node, and subtract the weighted average Gini of the two child nodes. The algorithm loops through every possible feature threshold and chooses the split that maximizes this gain.

#### ⏱️ Complexity
- **Time Complexity:** $O(N)$ to calculate counts.
- **Space Complexity:** $O(C)$ where C is number of unique classes.

---

### Q6: Principal Component Analysis (PCA)

**Problem:** Implement PCA to reduce the dimensionality of a dataset.

```python
import numpy as np

class PCA:
    def __init__(self, n_components):
        self.n_components = n_components
        self.components = None
        self.mean = None

    def fit(self, X):
        # 1. Mean centering
        self.mean = np.mean(X, axis=0)
        X_centered = X - self.mean
        
        # 2. Covariance matrix
        # rowvar=False indicates that each column represents a variable
        cov_matrix = np.cov(X_centered, rowvar=False)
        
        # 3. Eigenvalues and Eigenvectors
        eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
        
        # 4. Sort eigenvectors by decreasing eigenvalues
        idxs = np.argsort(eigenvalues)[::-1]
        eigenvalues = eigenvalues[idxs]
        eigenvectors = eigenvectors[:, idxs]
        
        # 5. Store first n_components
        self.components = eigenvectors[:, :self.n_components]

    def transform(self, X):
        # Project data
        X_centered = X - self.mean
        return np.dot(X_centered, self.components)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Goal:** Find the axes (Principal Components) that maximize the variance (spread) of the data.
- **Covariance & Eigenvectors:** The eigenvectors of the covariance matrix point in the directions of maximum variance. The eigenvalues represent the magnitude of that variance.
- **Projection:** We sort the eigenvectors by their eigenvalues to find the most "important" directions, keep the top `n_components`, and use matrix dot product to project our original data onto this new, smaller dimensional plane.

#### ⏱️ Complexity
- **Time Complexity:** $O(F^3)$ due to Eigen Decomposition of the $F \times F$ covariance matrix.
- **Space Complexity:** $O(F^2)$ to store the covariance matrix.

---

### Q7: Ridge Regression (L2 Regularization)

**Problem:** Implement Ridge Regression using closed-form solution (Normal Equation).

```python
import numpy as np

class RidgeRegression:
    def __init__(self, alpha=1.0):
        self.alpha = alpha
        self.weights = None

    def fit(self, X, y):
        n_samples, n_features = X.shape
        
        # Add a column of ones for the bias term (intercept)
        X_b = np.c_[np.ones((n_samples, 1)), X]
        
        # Identity matrix for regularization, size (n_features + 1) x (n_features + 1)
        I = np.eye(n_features + 1)
        
        # Do not regularize the bias term (first element)
        I[0, 0] = 0
        
        # Normal Equation with L2 penalty: w = (X^T * X + alpha * I)^-1 * X^T * y
        matrix_inverse = np.linalg.inv(X_b.T.dot(X_b) + self.alpha * I)
        self.weights = matrix_inverse.dot(X_b.T).dot(y)

    def predict(self, X):
        X_b = np.c_[np.ones((X.shape[0], 1)), X]
        return X_b.dot(self.weights)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **L2 Regularization:** Ridge regression adds a penalty term $\alpha \sum w_i^2$ to the loss function. This prevents weights from becoming too large, reducing overfitting (high variance).
- **Closed-Form Solution:** Instead of gradient descent, we use the Normal Equation. By adding $\alpha I$ to $X^T X$, we guarantee the matrix is invertible, solving multicollinearity issues.
- **Bias Term Trick:** We prepend a column of 1s to $X$ to compute the bias inside the weight vector, but we explicitly *don't* regularize the bias (by setting `I[0, 0] = 0`).

#### ⏱️ Complexity
- **Time Complexity:** $O(F^3)$ due to matrix inversion where $F$ is features.
- **Space Complexity:** $O(F^2)$ to store intermediate matrices.

---

### Q8: Softmax Function (Numerically Stable)

**Problem:** Implement a numerically stable Softmax function for multi-class classification.

```python
import numpy as np

def softmax(logits):
    """
    Computes numerically stable softmax for a 2D array of logits.
    logits shape: (batch_size, num_classes)
    """
    # Shift logits by subtracting the max value for numerical stability
    # keepdims=True ensures the shape allows broadcasting (batch_size, 1)
    shifted_logits = logits - np.max(logits, axis=1, keepdims=True)
    
    # Exponentiate
    exp_logits = np.exp(shifted_logits)
    
    # Normalize by dividing by the sum of exponents
    probabilities = exp_logits / np.sum(exp_logits, axis=1, keepdims=True)
    
    return probabilities

# Example:
logits = np.array([[1000, 1001, 1002]]) 
# standard np.exp(1000) causes overflow (inf), but stable softmax handles it gracefully:
print(softmax(logits)) # [[0.09 0.24 0.66]]
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Problem:** `np.exp(1000)` causes a float overflow (returns `inf`). If your neural network outputs large logits, standard softmax breaks.
- **The Solution:** $e^{x_i} / \sum e^{x_j} = e^{x_i - c} / \sum e^{x_j - c}$. Subtracting the maximum logit $c$ from all logits makes the largest exponent $e^0 = 1$. The math remains mathematically identical, but guarantees no overflow.

#### ⏱️ Complexity
- **Time Complexity:** $O(N \times C)$ where $C$ is number of classes.
- **Space Complexity:** $O(N \times C)$ for output probabilities.

---

### Q9: K-Fold Cross Validation Splitter

**Problem:** Write a generator function that yields train/test indices for K-Fold Cross Validation.

```python
import numpy as np

def k_fold_split(n_samples, k=5, shuffle=True, random_state=None):
    indices = np.arange(n_samples)
    
    if shuffle:
        if random_state is not None:
            np.random.seed(random_state)
        np.random.shuffle(indices)
        
    # Calculate sizes of each fold
    fold_sizes = np.full(k, n_samples // k, dtype=int)
    # Distribute remainders
    fold_sizes[:n_samples % k] += 1
    
    current_idx = 0
    for fold_size in fold_sizes:
        start, stop = current_idx, current_idx + fold_size
        
        # Test indices for this fold
        test_indices = indices[start:stop]
        
        # Train indices (everything else)
        train_indices = np.concatenate((indices[:start], indices[stop:]))
        
        yield train_indices, test_indices
        
        current_idx = stop

# Usage:
# for train_idx, test_idx in k_fold_split(10, k=3):
#     print(f"Train: {train_idx}, Test: {test_idx}")
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Math:** If $N=10$ and $K=3$, fold sizes should be `[4, 3, 3]`. We initialize all to $10 // 3 = 3$, and add $1$ to the first $10 \% 3 = 1$ folds.
- **Generators:** Using `yield` saves memory compared to returning a massive list of arrays.

---

### Q10: One-Hot Encoding from Scratch

**Problem:** Implement One-Hot Encoding for a 1D array of categorical integer labels.

```python
import numpy as np

def one_hot_encode(labels, num_classes=None):
    if num_classes is None:
        num_classes = np.max(labels) + 1
        
    n_samples = len(labels)
    
    # Initialize a matrix of zeros
    encoded = np.zeros((n_samples, num_classes))
    
    # Advanced indexing: 
    # np.arange(n_samples) generates row indices [0, 1, 2...]
    # labels provides column indices
    encoded[np.arange(n_samples), labels] = 1.0
    
    return encoded

# Example:
labels = np.array([0, 2, 1, 0])
# Output:
# [[1. 0. 0.]
#  [0. 0. 1.]
#  [0. 1. 0.]
#  [1. 0. 0.]]
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Advanced Indexing:** Instead of using a slow `for` loop, we use NumPy advanced indexing `matrix[rows, cols] = 1`. This runs at C-speed and is essential for performant data pipelines.

---

### Q11: TF-IDF (Term Frequency - Inverse Document Frequency)

**Problem:** Calculate TF-IDF matrix for a list of documents.

```python
import numpy as np
from collections import Counter
import math

def compute_tfidf(corpus):
    # 1. Tokenize and build vocabulary
    tokenized_docs = [doc.lower().split() for doc in corpus]
    vocab = list(set([word for doc in tokenized_docs for word in doc]))
    vocab_size = len(vocab)
    n_docs = len(corpus)
    
    # Map words to indices
    word_to_idx = {word: i for i, word in enumerate(vocab)}
    
    tf = np.zeros((n_docs, vocab_size))
    df = np.zeros(vocab_size)
    
    # 2. Compute TF (Term Frequency) and DF (Document Frequency)
    for doc_idx, doc in enumerate(tokenized_docs):
        doc_len = len(doc)
        word_counts = Counter(doc)
        
        for word, count in word_counts.items():
            word_idx = word_to_idx[word]
            # TF: (word count in doc) / (total words in doc)
            tf[doc_idx, word_idx] = count / doc_len
            # DF: +1 if word appears in this doc
            df[word_idx] += 1
            
    # 3. Compute IDF (Inverse Document Frequency)
    # Adding +1 for smoothing (prevents division by zero)
    idf = np.log((1 + n_docs) / (1 + df)) + 1
    
    # 4. Compute TF-IDF
    tfidf = tf * idf
    
    # Optional: L2 Normalize each document vector
    norms = np.linalg.norm(tfidf, axis=1, keepdims=True)
    tfidf = tfidf / np.where(norms == 0, 1, norms)
    
    return tfidf, vocab

# Usage:
corpus = [
    "the cat sat on the mat",
    "the dog barked at the cat",
    "the birds are flying"
]
tfidf_matrix, vocab = compute_tfidf(corpus)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **TF:** Measures how often a word appears in a specific document.
- **IDF:** Penalizes words that appear in *every* document (like "the", "and"). If a word is rare across the corpus, its IDF is high.
- **L2 Normalization:** Ensures that longer documents don't artificially get higher scores just because they have more words.

---

### Q12: Neural Network Forward Pass (NumPy)

**Problem:** Implement a simple 2-layer neural network forward pass (Linear -> ReLU -> Linear -> Sigmoid).

```python
import numpy as np

def relu(x):
    return np.maximum(0, x)

def sigmoid(x):
    return 1 / (1 + np.exp(-x))

def neural_network_forward(X, W1, b1, W2, b2):
    # Layer 1: Linear + ReLU
    z1 = np.dot(X, W1) + b1
    a1 = relu(z1)
    
    # Layer 2: Linear + Sigmoid
    z2 = np.dot(a1, W2) + b2
    a2 = sigmoid(z2)
    
    return a2

# Dimensions check:
# X:  (batch_size, input_dim) -> (32, 10)
# W1: (input_dim, hidden_dim) -> (10, 64)
# b1: (hidden_dim,)           -> (64,)
# W2: (hidden_dim, output_dim)-> (64, 1)
# b2: (output_dim,)           -> (1,)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Matrix Multiplication:** Shows deep understanding of tensor dimensions. `X` (32x10) dot `W1` (10x64) results in a (32x64) matrix.
- **Broadcasting:** When `b1` (64,) is added to the (32x64) matrix, NumPy automatically broadcasts it across all 32 rows.
- **Non-Linearities:** ReLU and Sigmoid allow the network to learn non-linear decision boundaries. Without them, the entire network collapses into a single linear transformation $X(W_1W_2) + (b_1W_2 + b_2)$.

---

*End of ML Coding Patterns — Deep mathematical breakdowns of foundational algorithms.*

