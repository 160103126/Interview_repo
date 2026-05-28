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

*End of ML Coding Patterns — Deep mathematical breakdowns of foundational algorithms.*
