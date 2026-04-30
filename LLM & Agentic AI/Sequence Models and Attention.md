# Sequence Models and Attention

We've seen how [[Deep Neural Networks]] handle sequences with RNNs and LSTMs. They work — but they hit a wall. The attention mechanism broke through that wall and changed everything. Let's see why.

---

## The Sequence-to-Sequence Problem

Many important tasks map one sequence to another:

- **Translation**: "I love cats" → "J'aime les chats"
- **Summarization**: Long article → short summary
- **Question Answering**: (context + question) → answer

The input and output can be **different lengths**. We need an architecture that handles this flexibly.

---

## Encoder-Decoder Architecture

The elegant solution (Sutskever et al., 2014):

1. **Encoder**: Process the entire input sequence with an RNN/LSTM. The final hidden state becomes a fixed-length "context vector" — a compressed representation of the entire input.
2. **Decoder**: Another RNN/LSTM that takes the context vector and generates the output sequence, one token at a time.

This works! It powered early neural machine translation. But there's a serious flaw...

---

## The Bottleneck Problem

Imagine compressing an entire paragraph into a single vector of 512 numbers. That's what the encoder does. For short sentences, it's manageable. But for long sequences:

- Critical information gets lost
- The encoder must somehow "prioritize" what to keep
- Performance degrades sharply with input length

We're asking one vector to do too much. What if the decoder could look back at the **entire input** instead of just one compressed vector?

---

## Attention: The Breakthrough

**Attention** (Bahdanau et al., 2015) is beautifully simple: instead of relying on a single context vector, let the decoder **look at all encoder hidden states** at every decoding step, and **learn which ones are relevant**.

**How it works, step by step:**

1. The encoder produces a hidden state for each input position (not just the final one)
2. At each decoder step, compute a **relevance score** between the current decoder state and every encoder state
3. Convert scores to **attention weights** (softmax — they sum to 1)
4. Compute a **weighted sum** of encoder states — this is the context for this specific output step
5. Use this focused context (alongside the decoder state) to predict the next output token

Each output word gets its own custom "view" of the input — focusing on what matters most for that particular prediction.

---

## Query, Key, Value: The Library Analogy

The modern formulation of attention uses three concepts. Imagine searching a library:

- **Query (Q)**: What you're looking for — "I need information about cats"
- **Key (K)**: The index card of each book — helps you determine relevance
- **Value (V)**: The actual content of the book — what you retrieve once you find a match

The process:
1. Compare your **query** to each **key** to get a relevance score
2. Use those scores to take a weighted combination of **values**

In the encoder-decoder setup:
- Query = current decoder state ("what do I need right now?")
- Keys = encoder hidden states ("what does each input position represent?")
- Values = encoder hidden states ("what information is there?")

This abstraction becomes crucial in [[The Transformer Architecture]].

---

## Visualizing Attention: Alignment Matrices

One beautiful property of attention: it's **interpretable**. We can visualize the attention weights as a matrix:

- Rows = output positions (decoder steps)
- Columns = input positions (encoder states)
- Each cell = how much the decoder "looked at" that input position

For translation, you'll see a roughly diagonal pattern (word order is similar) with interesting deviations (reordering, many-to-one mappings). This confirms the model is learning meaningful alignments.

---

## Why Attention Changed Everything

Before attention:
- Long sequences degraded badly
- No interpretability
- Fixed-size bottleneck

After attention:
- Performance holds across long sequences
- We can see what the model focuses on
- Information flows directly from relevant input positions

But we still have recurrence — the encoder and decoder are still LSTMs, processing one step at a time. What if we could use attention **without** any recurrence?

That's exactly what [[The Transformer Architecture]] does — and it's the foundation of every modern large language model.

---

## **Key Takeaways**

1. Encoder-decoder architectures map variable-length inputs to variable-length outputs, but compress everything into a single bottleneck vector.
2. Attention allows the decoder to look at ALL encoder states at each step, learning which input positions are relevant.
3. The Query-Key-Value framework: queries search against keys to produce weights, which select from values.
4. Attention weights are interpretable — you can visualize exactly what the model is "looking at."
5. Attention removed the information bottleneck but still relied on slow sequential (recurrent) processing — motivating the fully attention-based Transformer.
