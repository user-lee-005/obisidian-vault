# Training a Model

> Training a neural network is like tuning a radio — you adjust the dials (weights) until the static (error) goes away and the signal (predictions) comes through clearly.

---

## What Does "Training" Actually Mean?

When we say a model is "training," we mean:

1. The model makes a prediction (forward pass through [[Neural Networks from Scratch|the network]])
2. We measure how wrong the prediction is (loss)
3. We figure out which weights caused the error (backpropagation)
4. We nudge those weights slightly to reduce the error (gradient descent)
5. **Repeat** thousands or millions of times

That's the entire training loop. Everything else is details.

Before training, weights are random → predictions are garbage. After training, weights are tuned → predictions are (hopefully) accurate.

---

## Loss/Cost Function — Measuring How Wrong You Are

The **loss function** takes the model's prediction and the true answer, and outputs a single number — how bad the prediction was. Lower loss = better model.

### Mean Squared Error (MSE) — for regression
```
MSE = average of (predicted - actual)²
```
- Punishes big errors heavily (squaring amplifies them)
- Used when predicting continuous numbers (prices, temperatures)

### Cross-Entropy Loss — for classification
```
Loss = -[y·log(ŷ) + (1-y)·log(1-ŷ)]
```
- Penalizes confident *wrong* predictions very harshly
- Used when predicting categories (spam/not-spam, cat/dog/bird)

**Intuition:** The loss function is your "grade" — it tells the model exactly how far off it is. The whole goal of training is to make this number as small as possible.

---

## Gradient Descent — Walking Downhill

Imagine you're blindfolded on a mountainous landscape. You want to find the lowest valley (minimum loss). You can't see, but you can feel the slope under your feet.

**Gradient descent** is the strategy: *feel which direction goes downhill, take a step that way, repeat.*

Mathematically:
- The **gradient** tells you the direction of steepest *ascent* (uphill)
- So you go the *opposite* direction (downhill)
- New weight = old weight - learning_rate × gradient

This is how every weight in the network gets updated — each one gets nudged in the direction that reduces the loss.

The [[Math You Actually Need|math]] behind this is just calculus (derivatives), but the intuition is simple: walk downhill.

---

## Learning Rate — How Big Are Your Steps?

The **learning rate** (often called `α` or `lr`) controls how much we adjust weights in each step.

- **Too large** → you overshoot the minimum, bounce around, never converge (like taking giant leaps on the mountain and jumping over valleys)
- **Too small** → you'll get there eventually, but training takes forever (like taking baby steps)
- **Just right** → you converge smoothly and efficiently

Typical values: 0.001 to 0.01 (but it varies wildly by problem)

**In practice**, people use learning rate schedulers that start large (explore quickly) and shrink over time (fine-tune carefully).

---

## Backpropagation — Blame Flows Backward

Backpropagation answers: *"Which weights are most responsible for the error, and how should we change them?"*

**The chain rule intuition:**

Imagine a long assembly line producing a product. The final product has a defect. You need to trace back through the line to find which station caused the problem and how much each station contributed.

Backpropagation does exactly this:
1. Start at the output (where we measured the loss)
2. Ask: "How much did each weight in the last layer contribute to this loss?"
3. Move to the previous layer: "How much did *these* weights contribute?"
4. Continue backward through every layer until you reach the inputs

This uses the **chain rule** from calculus — the derivative of a composition of functions. If layer 3's output depends on layer 2, and layer 2's output depends on layer 1, then we can compute how layer 1's weights affect the final loss by chaining the derivatives together.

**In practice:** Frameworks like PyTorch and TensorFlow compute this automatically. You don't manually derive gradients — but understanding the concept helps you debug training issues.

---

## Training Loop Terminology

Let's clarify terms that often confuse beginners:

- **Epoch** — one complete pass through the *entire* training dataset. If you have 10,000 samples, one epoch means the model has seen all 10,000.

- **Batch** — a subset of the training data processed together. If batch size = 32, you process 32 samples before updating weights.

- **Iteration (step)** — one weight update. With 10,000 samples and batch size 32, one epoch = ~312 iterations.

**Example:** Training for 10 epochs with batch size 32 on 10,000 samples = 10 × 312 = 3,120 weight updates total.

---

## Flavors of Gradient Descent

| Type | Batch Size | Pros | Cons |
|------|-----------|------|------|
| **Batch GD** | Entire dataset | Stable, exact gradient | Slow, memory-heavy |
| **Stochastic GD (SGD)** | 1 sample | Fast updates, can escape local minima | Noisy, unstable |
| **Mini-batch GD** | 16-512 samples | Best of both worlds | Need to pick batch size |

**In practice**, everyone uses **mini-batch gradient descent** (typically batch size 32, 64, 128, or 256). When people say "SGD" today, they usually mean mini-batch SGD with momentum.

---

## When to Stop Training

Training too little → [[Overfitting and Generalization|underfitting]] (model hasn't learned enough)
Training too long → [[Overfitting and Generalization|overfitting]] (model memorizes training data)

**How to know when to stop:**

1. **Monitor validation loss** — when it starts going *up* while training loss goes *down*, you're overfitting
2. **Early stopping** — save the model whenever validation loss improves; stop after N epochs without improvement
3. **Set a target** — if you know what "good enough" accuracy looks like for your use case

The loss curve typically looks like:
- Training loss: steadily decreases
- Validation loss: decreases, then hits a floor, then starts increasing ← **stop here**

---

## **Key Takeaways**

1. Training = repeatedly making predictions, measuring error, and adjusting weights to reduce that error.
2. The **loss function** quantifies "how wrong" — MSE for regression, cross-entropy for classification.
3. **Gradient descent** adjusts weights in the direction that reduces loss — "walk downhill."
4. **Learning rate** controls step size — too big overshoots, too small is painfully slow.
5. **Backpropagation** uses the chain rule to compute how much each weight contributes to the error, flowing blame backward through the network.
6. In practice, we use mini-batch gradient descent and stop training when validation loss stops improving.
7. Next up: understanding why models fail to generalize → [[Overfitting and Generalization]].
