# The Transformer Architecture

This is it. The paper that changed everything: **"Attention Is All You Need"** (Vaswani et al., 2017). If you understand one thing in modern AI deeply, it should be this. Every GPT, BERT, LLaMA, and Claude is built on this foundation.

The big idea? Throw away recurrence entirely. Use **only attention**.

---

## Why Ditch Recurrence?

We saw in [[Sequence Models and Attention]] that attention solved the information bottleneck. But the encoder and decoder were still RNNs — processing tokens **one at a time**, sequentially.

Problems with recurrence:
- **Can't parallelize**: Step $t$ depends on step $t-1$. GPUs sit idle.
- **Long paths**: Information from token 1 must pass through every intermediate state to reach token 100.
- **Slow training**: Sequential processing means training time scales linearly with sequence length.

The Transformer's answer: process all positions **simultaneously** using self-attention.

---

## Self-Attention: The Core Innovation

In encoder-decoder attention, the decoder attends to the encoder. In **self-attention**, each position in a sequence attends to **every other position in the same sequence**.

For the sentence "The cat sat on the mat":
- When processing "sat", the model can directly attend to "cat" (the subject) and "mat" (the location)
- Every word gets context from every other word — in a single operation
- No sequential processing required

This is massively parallelizable and captures long-range dependencies in one step.

---

## Architecture Overview

The Transformer has two main stacks:

**Encoder Stack** (e.g., 6 identical layers):
- Each layer has: Multi-Head Self-Attention → Add & Norm → Feed-Forward → Add & Norm

**Decoder Stack** (e.g., 6 identical layers):
- Each layer has: Masked Self-Attention → Add & Norm → Cross-Attention (to encoder) → Add & Norm → Feed-Forward → Add & Norm

The flow:
1. Input tokens → [[Word Embeddings]] + Positional Encoding
2. Pass through encoder stack → rich contextual representations
3. Decoder generates output tokens one at a time, attending to both its own previous outputs (masked self-attention) and the encoder output (cross-attention)

---

## Self-Attention Step by Step

Let's walk through what happens at one attention layer. For each input position, we compute three vectors:

- **Q (Query)**: "What am I looking for?"
- **K (Key)**: "What do I contain/represent?"
- **V (Value)**: "What information do I carry?"

These are created by multiplying the input embedding by three learned weight matrices: $W_Q$, $W_K$, $W_V$.

**The computation:**

1. **Score**: For each pair of positions, compute $Q_i \cdot K_j$ (dot product). High score = high relevance.
2. **Scale**: Divide by $\sqrt{d_k}$ (dimension of keys). Without this, large dimensions make dot products huge, pushing softmax into saturated regions where gradients vanish.
3. **Softmax**: Convert scores to probabilities (attention weights that sum to 1).
4. **Weighted sum**: Multiply each Value by its attention weight and sum them up.

The formula: $\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$

**Intuition**: Each word "asks a question" (Q), every word "advertises what it has" (K), and you retrieve a blend of "answers" (V) weighted by relevance.

---

## Multi-Head Attention: Multiple Perspectives

One attention head might learn syntactic relationships ("subject-verb agreement"). Another might learn semantic relationships ("cat-animal"). Running attention multiple times in parallel — with different learned projections — lets the model capture **different types of relationships simultaneously**.

- Split Q, K, V into $h$ heads (e.g., 8 or 12)
- Run attention independently on each head
- Concatenate results and project back to full dimension

This is like having 8 different "experts" each looking at the sentence from a different angle.

---

## Positional Encoding: Where Am I?

Since self-attention processes all positions in parallel (no sequential order), the model has **no inherent notion of position**. "The cat sat" and "sat cat the" would look identical without position information.

**Solution**: Add a positional encoding vector to each input embedding.

The original paper uses sinusoidal functions:
- $PE_{(pos, 2i)} = \sin(pos / 10000^{2i/d})$
- $PE_{(pos, 2i+1)} = \cos(pos / 10000^{2i/d})$

Each position gets a unique pattern. Nearby positions have similar encodings. The model can learn to attend to relative positions ("two words to the left") from these patterns.

---

## Feed-Forward Layers

After attention, each position passes independently through a two-layer feed-forward network:

$\text{FFN}(x) = \text{ReLU}(xW_1 + b_1)W_2 + b_2$

This processes each position's enriched representation **independently** — no interaction between positions here. Think of it as "digesting" the information gathered by attention.

---

## Residual Connections and Layer Normalization

Every sub-layer (attention, feed-forward) is wrapped with:
- **Residual connection**: $\text{output} = \text{sublayer}(x) + x$ — gradients flow freely, as we learned in [[Deep Neural Networks]]
- **Layer normalization**: Stabilizes training by normalizing across features

These enable training very deep stacks (6+ layers) without degradation.

---

## Why Transformers Win

| Property | RNNs | Transformers |
|----------|------|--------------|
| Parallelizable | ❌ Sequential | ✅ Fully parallel |
| Long-range deps | Degrades with distance | Direct connection (1 step) |
| Training speed | Slow (sequential) | Fast (parallel on GPUs) |
| Scalability | Limited | Scales to billions of parameters |

The Transformer's parallelism meant we could finally **scale** — throw more data, more compute, more parameters at it. This scalability is what enabled GPT-3, GPT-4, and all modern LLMs.

---

## The Foundation of Everything

- **BERT**: Encoder-only Transformer (bidirectional self-attention)
- **GPT**: Decoder-only Transformer (causal/left-to-right self-attention)
- **T5**: Full encoder-decoder Transformer

Every major language model since 2018 is a Transformer variant. Understanding this architecture is understanding the engine that powers modern AI.

Next, we need to understand how text gets fed into this architecture — that's [[Tokenization and Vocabulary]].

---

## **Key Takeaways**

1. Transformers replace recurrence with self-attention — every position attends to every other position in parallel.
2. Q/K/V projections let each word "search" for relevant context. Scaling by √d_k prevents gradient saturation.
3. Multi-head attention captures multiple relationship types simultaneously (syntax, semantics, etc.).
4. Positional encodings inject order information since self-attention is position-agnostic.
5. Residual connections + layer norm enable deep stacking without training degradation.
6. Parallelism and scalability are why Transformers enabled the leap to billion-parameter models — this architecture is the foundation of ALL modern LLMs.
