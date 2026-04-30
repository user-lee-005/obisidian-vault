# Neural Networks from Scratch

> A neural network is just a function — a very flexible one — that takes numbers in and produces numbers out. The "magic" is in how it adjusts millions of knobs to get the right answers.

---

## Biological Inspiration

Yes, neural networks are loosely inspired by the brain. A biological neuron:
- Receives electrical signals from other neurons
- If the combined signal exceeds a threshold, it "fires" and sends a signal forward

But don't over-analogize. Artificial neural networks are **math**, not biology. They share a vague structural idea — connected units passing signals — but the details are completely different. Think of it as *inspiration*, not imitation.

---

## The Artificial Neuron (Perceptron)

A single artificial neuron does this:

1. **Receives inputs** — a list of numbers (x₁, x₂, x₃, ...)
2. **Multiplies each by a weight** — (w₁·x₁, w₂·x₂, w₃·x₃, ...)
3. **Sums them up + adds a bias** — z = w₁·x₁ + w₂·x₂ + ... + b
4. **Applies an activation function** — output = f(z)

**Weights** control how important each input is. **Bias** shifts the decision boundary. These are the parameters the model *learns* during [[Training a Model|training]].

Think of it like a voting system — each input gets a vote, weights determine how much that vote counts, and the activation function decides the final answer.

---

## Activation Functions

Without activation functions, a neural network would just be a linear equation — no matter how many layers, it could only draw straight lines. Activation functions introduce **non-linearity**, allowing the network to learn complex patterns.

### Sigmoid
- Squishes any number into the range (0, 1)
- Formula: σ(z) = 1 / (1 + e⁻ᶻ)
- **Good for:** output layer in binary classification (probability-like output)
- **Problem:** gradients vanish for very large/small inputs (slow learning)

### ReLU (Rectified Linear Unit)
- If input > 0, output = input. If input ≤ 0, output = 0
- Formula: f(z) = max(0, z)
- **Good for:** hidden layers — simple, fast, works well in practice
- **Problem:** "dead neurons" — if a neuron always gets negative input, it never activates

### Tanh
- Squishes numbers into (-1, 1) — centered around zero
- Formula: tanh(z) = (eᶻ - e⁻ᶻ) / (eᶻ + e⁻ᶻ)
- **Good for:** when you want outputs centered at zero
- **Problem:** also suffers from vanishing gradients (less than sigmoid)

**Rule of thumb:** Use **ReLU** for hidden layers, **sigmoid** for binary output, **softmax** for multi-class output.

---

## Layers — Building the Network

A neural network is organized in layers:

- **Input layer** — receives raw data (one neuron per feature). If your data has 10 features, you have 10 input neurons.
- **Hidden layers** — the "thinking" layers. Each neuron connects to all neurons in the previous layer. *This is where patterns get learned.*
- **Output layer** — produces the final prediction. One neuron for regression, one per class for classification.

**Architecture example:**
```
Input (4 features) → Hidden (8 neurons) → Hidden (8 neurons) → Output (1 neuron)
```

The number and size of hidden layers is a design choice. More/bigger layers = more capacity to learn complex patterns, but also more risk of [[Overfitting and Generalization|overfitting]].

---

## Forward Pass — How Data Flows

The forward pass is simply the computation from input to output:

1. Feed input values into the input layer
2. Each hidden layer computes: `output = activation(weights · inputs + bias)`
3. Pass the output to the next layer
4. Repeat until you reach the output layer
5. The output layer gives your prediction

That's it. Data flows **forward** — one layer at a time, left to right. No loops, no going back (in a standard feedforward network).

Every single step is just multiplication, addition, and applying a function. A computer can do billions of these per second.

---

## A Simple Example

**Task:** Classify if a number is positive or negative (simplified binary classification)

- **Input:** one number (e.g., -3, 7, 0.5)
- **Network:** Input(1) → Hidden(2 neurons, ReLU) → Output(1, sigmoid)
- **Output:** a number between 0 and 1 (probability of being positive)

Before [[Training a Model|training]], the weights are random — the network guesses randomly. After training on examples like (-5→0, 3→1, -1→0, 7→1), it adjusts weights so that:
- Positive inputs → output close to 1
- Negative inputs → output close to 0

The weights encode the "knowledge" the network has learned.

---

## Why Depth Matters

A network with *one* hidden layer can theoretically approximate any function — but might need an impossibly large layer. **Deep** networks (many layers) are more efficient because:

- **Early layers** learn simple features (edges, basic patterns)
- **Middle layers** combine simple features into complex ones (shapes, textures)
- **Later layers** combine complex features into high-level concepts (faces, objects)

This hierarchical learning is why "deep learning" works so well for complex tasks like image recognition, language understanding, and game playing.

Each layer builds on the previous one — like how you learn letters, then words, then sentences, then meaning.

---

## **Key Takeaways**

1. A neuron is just: `output = activation(weights · inputs + bias)` — multiply, sum, squish.
2. Activation functions (ReLU, sigmoid, tanh) introduce non-linearity — without them, the network can only learn linear relationships.
3. Networks are organized in layers: input → hidden(s) → output. Data flows forward.
4. **Weights and biases** are the learnable parameters — they start random and get adjusted during [[Training a Model|training]].
5. Deeper networks learn hierarchical features — simple patterns combine into complex concepts.
6. The [[Math You Actually Need|math]] behind all of this is just linear algebra (matrix multiplication) and calculus (for learning).
