# Math You Actually Need

> Don't panic. You don't need a math degree to understand ML. You need *intuition* for three areas: linear algebra, probability, and calculus. Let's build that intuition without drowning in notation.

---

## The Honest Truth

Here's what you actually need:
- **Linear algebra** — because data and weights are vectors/matrices
- **Probability** — because predictions are uncertain
- **Calculus** — because [[Training a Model|training]] uses gradients

You don't need to prove theorems. You need to understand *what* these tools do and *why* ML uses them. Frameworks like PyTorch handle the actual computation.

---

## Linear Algebra — The Language of Data

### Vectors — Lists of Numbers

A vector is just an ordered list of numbers:
```
v = [3, 1, 4, 1, 5]
```

**In ML context:**
- A single data point with 5 features is a vector of length 5
- A word embedding is a vector (e.g., 768 numbers representing a word's meaning)
- The weights connecting to one neuron form a vector

**Geometric intuition:** A vector has direction and magnitude. Think of an arrow pointing somewhere in space.

### Matrices — Grids of Numbers

A matrix is a 2D grid (rows × columns):
```
M = [[1, 2, 3],
     [4, 5, 6]]    ← 2 rows, 3 columns (2×3 matrix)
```

**In ML context:**
- A batch of training data: each row = one sample, each column = one feature
- The weights between two [[Neural Networks from Scratch|layers]] form a matrix
- An image is a matrix of pixel values

### Dot Product — How Similar Are Two Vectors?

The dot product multiplies corresponding elements and sums them:
```
[1, 2, 3] · [4, 5, 6] = 1×4 + 2×5 + 3×6 = 32
```

**Why it matters in ML:**
- A neuron's computation is literally a dot product: `weights · inputs + bias`
- Cosine similarity (measuring how similar two vectors are) is a normalized dot product
- Attention mechanisms in Transformers use dot products to measure relevance

**Intuition:** If two vectors point in the same direction, their dot product is large and positive. If perpendicular, it's zero. If opposite, it's negative. It measures *alignment*.

### Matrix Multiplication — Transforming Data

When we multiply a matrix by a vector, we're *transforming* that vector — rotating, scaling, or projecting it into a new space.

```
[w₁₁ w₁₂] × [x₁] = [w₁₁·x₁ + w₁₂·x₂]
[w₂₁ w₂₂]   [x₂]   [w₂₁·x₁ + w₂₂·x₂]
```

**In ML context:**
- A forward pass through a layer is: `output = W × input + bias`
- That's just matrix multiplication + vector addition
- The entire [[Neural Networks from Scratch|neural network]] is a series of matrix multiplications with activation functions between them

**This is why GPUs matter** — they're built for parallel matrix math.

---

## Probability — Dealing with Uncertainty

### Probability Distributions

A probability distribution tells you how likely different outcomes are:
- **Uniform:** all outcomes equally likely (fair die)
- **Normal (Gaussian):** bell curve — most values near the mean (heights, measurement errors)
- **Bernoulli:** binary outcome (coin flip, spam/not-spam)

**In ML:** Model outputs are often probabilities. "80% chance this email is spam" is more useful than just "spam" — it tells you how confident the model is.

### Bayes' Theorem — Updating Beliefs

**Intuition only:** Given new evidence, how should we update our beliefs?

```
P(hypothesis | evidence) ∝ P(evidence | hypothesis) × P(hypothesis)
```

In plain English: *"What I now believe = How well the evidence fits my hypothesis × What I believed before."*

**Example:** If 1% of emails are spam (prior), but an email contains "free money" (evidence), and "free money" appears in 90% of spam but only 0.1% of legitimate email — Bayes' theorem tells you this email is almost certainly spam.

**In ML:** Naive Bayes classifiers use this directly. More broadly, the concept of updating beliefs with evidence is fundamental to how models learn.

### Expected Value

The average outcome if you repeated an experiment infinitely:
```
E[X] = Σ (value × probability of that value)
```

**In ML:** Loss functions compute an expected value over the training data. Reinforcement learning maximizes expected cumulative reward.

---

## Calculus — The Engine of Learning

### Derivatives = Rate of Change = Slope

The derivative of a function at a point tells you: *"If I nudge the input slightly, how much does the output change?"*

- If the derivative is positive → output increases as input increases (going uphill)
- If the derivative is negative → output decreases as input increases (going downhill)
- If the derivative is zero → you're at a flat spot (possibly a minimum!)

**In ML:** We want to know "if I change this weight slightly, how does the loss change?" That's a derivative.

### Partial Derivatives — One Variable at a Time

When a function has multiple inputs (and neural networks have millions!), a partial derivative tells you how the output changes when you change *just one* input, holding all others constant.

```
∂Loss/∂w₁ → "How does loss change if I nudge weight w₁?"
∂Loss/∂w₂ → "How does loss change if I nudge weight w₂?"
```

Each weight gets its own update based on its own partial derivative.

### Chain Rule — Why Backpropagation Works

If `y = f(g(x))` — a function of a function — then:
```
dy/dx = dy/dg × dg/dx
```

**Why this matters for [[Training a Model|backpropagation]]:**

A neural network is a chain of functions: Layer 1 → Layer 2 → Layer 3 → Loss

To find how a weight in Layer 1 affects the Loss, you chain together:
```
∂Loss/∂w₁ = ∂Loss/∂Layer3 × ∂Layer3/∂Layer2 × ∂Layer2/∂w₁
```

The chain rule lets us propagate blame backward through arbitrarily deep networks. Without it, deep learning wouldn't work.

### Gradient = Direction of Steepest Ascent

The **gradient** is a vector of all partial derivatives — it points in the direction of steepest increase.

```
∇Loss = [∂Loss/∂w₁, ∂Loss/∂w₂, ..., ∂Loss/∂wₙ]
```

**In [[Training a Model|gradient descent]]:** We go the *opposite* direction of the gradient (steepest *descent*) to minimize loss.

---

## How It All Connects to ML

| Math Concept | ML Usage |
|-------------|----------|
| Vectors | Data points, weights, embeddings |
| Matrices | Weight matrices between layers, batches of data |
| Dot product | Neuron computation, attention scores |
| Matrix multiplication | Forward pass through a layer |
| Probability | Model confidence, output interpretation |
| Bayes' theorem | Classification, belief updating |
| Derivatives | How loss changes with weights |
| Chain rule | Backpropagation through [[Neural Networks from Scratch|deep networks]] |
| Gradient | Direction to adjust weights during training |

**The whole training process in math terms:**
1. Forward pass: matrix multiplications + activations (linear algebra)
2. Loss computation: compare prediction to truth (probability/statistics)
3. Backward pass: chain rule to get gradients (calculus)
4. Weight update: subtract gradient × learning rate (calculus + linear algebra)

---

## What You Can Safely Ignore (For Now)

- Eigenvectors and eigenvalues (useful later for PCA, not needed initially)
- Information theory (useful but advanced)
- Measure theory (only needed if doing ML research)
- Differential geometry (manifold learning — very advanced)

Focus on the intuition first. The formal notation will become readable once you understand *what* the operations mean.

---

## **Key Takeaways**

1. You need intuition in three areas: **linear algebra** (data as vectors/matrices), **probability** (uncertainty in predictions), and **calculus** (how to adjust weights).
2. A neuron's computation is a **dot product** — weights · inputs — which is why linear algebra is everywhere in ML.
3. The **chain rule** is why we can train deep networks — it lets us compute how any weight affects the final loss, no matter how deep.
4. The **gradient** points uphill; we go the opposite direction to minimize loss — that's [[Training a Model|gradient descent]].
5. Frameworks compute all this automatically — your job is understanding *what* the math does, not computing it by hand.
6. With this foundation, you're ready to explore how these principles scale up to [[What is AI and ML|modern AI systems]] like large language models.
