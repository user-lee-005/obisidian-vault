# Overfitting and Generalization

> A model that memorizes the training data is like a student who memorizes exam answers without understanding the subject — they'll ace the practice test but fail a new one.

---

## The Goal: Generalization

The whole point of [[How Machines Learn|machine learning]] is to make predictions on **data the model has never seen**. We don't care how well it performs on training data — we care about performance on new, real-world data.

A model that generalizes well has learned the *underlying patterns*, not the specific noise in the training set.

---

## Overfitting — Too Complex, Fits Noise

**What happens:** The model is so powerful that it memorizes the training data — including random noise, outliers, and quirks that don't represent real patterns.

**Symptoms:**
- Training accuracy: 99.9%
- Validation/test accuracy: 72%
- The gap between training and validation performance is large

**Analogy:** Imagine memorizing every single question-answer pair from past exams. You'd score 100% on those exact questions. But if the new exam rephrases things or asks novel questions, you'd fail — because you never actually *understood* the material.

**When it happens:**
- Model is too complex for the amount of data (too many parameters)
- Training data is small or noisy
- Training for too many epochs

---

## Underfitting — Too Simple, Misses Patterns

**What happens:** The model is too simple to capture the real patterns in the data.

**Symptoms:**
- Training accuracy: 65%
- Validation accuracy: 63%
- Both are bad — the model hasn't learned much at all

**Analogy:** Trying to draw a curve with only a straight line. No matter how you position that line, it can't capture the shape.

**When it happens:**
- Model is too simple (not enough layers/neurons in a [[Neural Networks from Scratch|neural network]])
- Training stopped too early
- Features are insufficient or poorly engineered

**The fix:** Use a more complex model, train longer, or add better features.

---

## Train / Validation / Test Split

We split our data into three parts to properly evaluate [[Training a Model|the model]]:

| Set | Purpose | When Used |
|-----|---------|-----------|
| **Training set** (~70-80%) | Model learns from this | During training |
| **Validation set** (~10-15%) | Tune hyperparameters, check for overfitting | During development |
| **Test set** (~10-15%) | Final unbiased evaluation | Only once, at the very end |

**Why three and not two?**

If you only have train and test, and you keep tweaking your model based on test performance, you're indirectly fitting to the test set — it's no longer unbiased.

The **validation set** is your "practice test" — you check it often and adjust. The **test set** is the "final exam" — you only look at it once, when you're done.

**Cross-validation** (k-fold) is an alternative for small datasets — you rotate which portion serves as validation.

---

## Techniques to Prevent Overfitting

### Regularization (L1 and L2)

Add a **penalty** to the loss function that discourages large weights:

- **L2 (Ridge):** Adds sum of squared weights to loss. Pushes weights to be *small* but not zero. Like saying "keep it simple."
- **L1 (Lasso):** Adds sum of absolute weights to loss. Pushes some weights to *exactly zero*. Effectively removes unimportant features.

**Intuition:** The model must now balance two goals — fit the data well AND keep weights small. This prevents it from memorizing noise (which requires extreme weight values).

### Dropout

During training, **randomly deactivate** a percentage of neurons in each forward pass (typically 20-50%).

**Why it works:**
- Forces the network to not rely on any single neuron
- Creates redundancy — multiple neurons learn similar patterns
- Acts like training many different smaller networks and averaging their predictions

**Important:** Dropout is only active during training, *not* during inference.

### Early Stopping

Monitor validation loss during [[Training a Model|training]]:
- Save the model whenever validation loss improves
- If validation loss hasn't improved for N epochs (patience), stop training
- Use the saved "best" model

This is the simplest and most common regularization technique. It's almost always used.

### Data Augmentation

If you can't get more data, **create variations** of your existing data:

- **Images:** flip, rotate, crop, adjust brightness, add noise
- **Text:** synonym replacement, random insertion, back-translation
- **Audio:** change speed, add background noise, shift pitch

More diverse training data → harder to memorize → better generalization.

---

## Bias-Variance Tradeoff

This is the fundamental tension in ML:

- **Bias** — error from oversimplifying. High bias = underfitting. The model's assumptions prevent it from capturing the true pattern.
- **Variance** — error from being too sensitive to training data. High variance = overfitting. The model captures noise as if it were signal.

**Imagine a dartboard:**
- High bias, low variance → darts clustered together, but far from the bullseye (consistently wrong in the same way)
- Low bias, high variance → darts scattered everywhere, centered on the bullseye (right on average, but inconsistent)
- Low bias, low variance → darts clustered tightly on the bullseye ← **this is what we want**

**The tradeoff:** As you increase model complexity:
- Bias decreases (model can capture more complex patterns)
- Variance increases (model becomes more sensitive to specific training data)

**The sweet spot** is where total error (bias + variance) is minimized. This is what validation sets help you find.

---

## Putting It All Together

```
Underfitting ←————————————— Sweet Spot ——————————————→ Overfitting
High bias                  Low bias + Low variance              High variance
Simple model               Right complexity                     Too complex
```

Your workflow:
1. Start with a model complex enough to overfit (prove it can learn)
2. Then add regularization techniques to pull it back toward generalization
3. Use validation metrics to find the sweet spot

---

## **Key Takeaways**

1. The goal is **generalization** — performing well on unseen data, not memorizing training data.
2. **Overfitting** = model too complex, memorizes noise. **Underfitting** = model too simple, misses patterns.
3. Always split data into train/validation/test — the test set is your final, unbiased evaluation.
4. Combat overfitting with: regularization (L1/L2), dropout, early stopping, and data augmentation.
5. The **bias-variance tradeoff** is the fundamental tension — find the sweet spot between too simple and too complex.
6. To understand *why* these techniques work mathematically, see [[Math You Actually Need]].
