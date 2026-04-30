# Deep Neural Networks

In [[Neural Networks from Scratch]], we built a simple network with one or two layers. That's enough to learn basic patterns — but the real world is messy, hierarchical, and deeply structured. Let's go deeper.

---

## From Shallow to Deep: Why More Layers Help

Imagine trying to recognize a face. A single layer might detect edges. But you need **layers of abstraction**:

- **Layer 1**: Edges and simple textures
- **Layer 2**: Combinations — eyes, noses, curves
- **Layer 3**: Face parts arranged in spatial patterns
- **Layer 4**: Entire faces, expressions, identities

This is **hierarchical feature learning**. Each layer builds on the previous one, composing simple features into complex concepts. A shallow network would need an impossibly wide single layer to capture what a deep network learns naturally through composition.

---

## The Vanishing/Exploding Gradient Problem

Going deep introduces a nasty problem. Remember from [[Training a Model]] — we use backpropagation to compute gradients. In a deep network, gradients get **multiplied** through each layer during the backward pass.

- **Vanishing gradients**: If multipliers are < 1, gradients shrink exponentially. Early layers barely learn.
- **Exploding gradients**: If multipliers are > 1, gradients blow up. Training becomes unstable.

**Solutions that made deep learning possible:**

- **ReLU activation**: Unlike sigmoid (which squashes to 0–1), ReLU is simply `max(0, x)`. Gradients are either 0 or 1 — no shrinking.
- **Batch Normalization**: Normalize activations within each mini-batch. Keeps values in a healthy range throughout the network.
- **Residual connections (skip connections)**: Add the input of a layer directly to its output (`y = F(x) + x`). Gradients can flow straight through the skip path. This is what made networks with 100+ layers trainable.

---

## CNNs: Built for Images

**Convolutional Neural Networks** exploit a key insight: images have local spatial structure.

- **Convolution operation**: Slide a small filter (kernel) across the image. At each position, compute a dot product. The filter detects a specific pattern (edge, corner, texture).
- **Multiple filters**: Each filter learns a different feature. Stack them to get a rich feature map.
- **Pooling**: Downsample feature maps (e.g., max pooling takes the maximum value in a region). Reduces computation and adds translation invariance.
- **Visual hierarchy**: Early layers detect edges → middle layers detect parts → deep layers detect objects.

CNNs are powerful for images, but what about **sequences** — like text?

---

## RNNs: Processing Sequential Data

Text is inherently sequential. The meaning of a word depends on what came before. **Recurrent Neural Networks** handle this by maintaining a **hidden state** — a vector that carries information from previous time steps.

At each step $t$:
- Take the current input (a word) and the previous hidden state
- Produce a new hidden state and (optionally) an output

We can "unroll" an RNN through time — it's like having a copy of the network at each time step, connected by the hidden state.

**The problem**: RNNs have terrible memory. After 10-20 steps, the hidden state has essentially "forgotten" the beginning of the sequence. This is the vanishing gradient problem again, but across time steps.

---

## LSTMs and GRUs: Gated Memory

**Long Short-Term Memory (LSTM)** networks solve the memory problem with **gates** — learned mechanisms that control information flow:

- **Forget gate**: "What should I throw away from memory?" (sigmoid → 0 means forget, 1 means keep)
- **Input gate**: "What new information should I store?" (selects which values to update)
- **Output gate**: "What part of memory should I reveal right now?"

The key innovation is the **cell state** — a highway that runs through time, allowing gradients (and information) to flow uninterrupted.

**GRUs (Gated Recurrent Units)** are a simplified variant with just two gates (reset and update). Fewer parameters, often similar performance.

---

## Why This Matters for NLP

Text is sequential data. Language has long-range dependencies ("The cat, which sat on the mat that was in the kitchen, **was** sleeping" — "was" agrees with "cat", many words back).

LSTMs powered the first wave of modern NLP: machine translation, text generation, sentiment analysis. But they process words **one at a time** — which is slow and still struggles with very long dependencies.

The next revolution? [[Sequence Models and Attention]] — and eventually [[The Transformer Architecture]], which threw away recurrence entirely.

---

## **Key Takeaways**

1. Deeper networks learn hierarchical features — simple patterns compose into complex concepts.
2. Vanishing/exploding gradients are solved by ReLU, batch normalization, and residual connections.
3. CNNs use local filters and pooling for spatial data (images); RNNs use hidden states for sequential data (text).
4. Vanilla RNNs forget quickly — LSTMs and GRUs add gates to control long-term memory.
5. Sequential processing is inherently slow — this limitation motivates the move to attention and transformers.
