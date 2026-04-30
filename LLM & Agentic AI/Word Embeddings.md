# Word Embeddings

We've trained neural networks on numbers — inputs, weights, activations, gradients. But language is made of **words**. How do we bridge that gap? This is one of the most elegant ideas in all of NLP.

---

## The Problem: Computers Don't Understand Words

A neural network needs numerical input. We can't feed it the string "cat" — we need a vector of numbers that *represents* "cat". But how?

---

## One-Hot Encoding: Simple but Terrible

The naive approach: assign each word in your vocabulary an index, create a vector of all zeros with a 1 at that index.

- Vocabulary of 50,000 words → each word is a 50,000-dimensional vector with a single 1
- "cat" = [0, 0, 1, 0, 0, ... 0]
- "dog" = [0, 0, 0, 1, 0, ... 0]

**Why this fails:**
- **No relationships**: The distance between "cat" and "dog" is the same as between "cat" and "refrigerator". The representation encodes zero semantic information.
- **Huge and sparse**: 50,000 dimensions, almost all zeros. Computationally wasteful.
- **No generalization**: Learning something about "cat" tells the model nothing about "kitten".

We need something better — vectors where **similar words are close together**.

---

## The Insight: Distributional Hypothesis

> "You shall know a word by the company it keeps." — J.R. Firth (1957)

Words that appear in similar contexts tend to have similar meanings. "Dog" and "cat" both appear near "pet", "feed", "vet". This is the **distributional hypothesis** — and it's the foundation of all embedding methods.

---

## Word2Vec: Learning Meaning from Context

Word2Vec (Mikolov et al., 2013) trains a shallow neural network on a simple task: **predict words from their context** (or vice versa). The magic is that the learned weights become meaningful word representations.

**Two variants:**

- **CBOW (Continuous Bag of Words)**: Given surrounding context words, predict the center word. ("The ___ sat on the mat" → predict "cat")
- **Skip-gram**: Given a center word, predict the surrounding context words. ("cat" → predict "the", "sat", "on", "mat")

**How training works:**
- Slide a window across a massive text corpus (billions of words)
- For each window, create training examples (center word ↔ context words)
- Train the network to maximize prediction accuracy
- The hidden layer weights become your word vectors (typically 100–300 dimensions)

**The magic — vector arithmetic captures relationships:**

```
king - man + woman ≈ queen
paris - france + italy ≈ rome
walking - walk + swim ≈ swimming
```

The network was never told about gender or geography — it learned these relationships purely from patterns of co-occurrence. The dimensions encode latent semantic features.

---

## GloVe: Global Vectors

**GloVe** (Pennington et al., 2014) takes a different approach. Instead of training on local context windows, it:

1. Builds a giant **co-occurrence matrix** (how often does word A appear near word B across the whole corpus?)
2. Factorizes this matrix to produce dense vectors

The result is similar to Word2Vec, but GloVe explicitly leverages **global statistics** rather than local windows. In practice, both produce high-quality embeddings.

---

## Properties of Good Embeddings

Once trained, word embeddings have remarkable properties:

- **Similarity**: cos("happy", "joyful") is high; cos("happy", "table") is low
- **Analogies**: Consistent vector offsets capture relationships (gender, tense, plurality)
- **Clustering**: Plot embeddings in 2D (using t-SNE) and you'll see countries cluster together, verbs cluster together, etc.

---

## Limitations: One Vector Per Word

Here's the critical flaw: each word gets **exactly one vector**, regardless of context.

The word "bank" means different things in:
- "I sat by the river **bank**"
- "I went to the **bank** to deposit money"

Static embeddings give "bank" the same vector in both cases. This is the **polysemy problem** — and solving it requires **contextual embeddings** (which we'll meet in [[The Transformer Architecture]]).

---

## Connection to Neural Networks

How do embeddings fit into a model? They're simply the first layer — a lookup table (weight matrix) that maps word indices to dense vectors. As described in [[Neural Networks from Scratch]], this is just a learned weight matrix where row $i$ contains the embedding for word $i$.

During training, these embeddings get updated via backpropagation just like any other weights. We can also use **pre-trained embeddings** (trained on massive corpora) and fine-tune them for a specific task — a form of transfer learning.

---

## **Key Takeaways**

1. Words must be represented as numbers for neural networks — one-hot encoding fails because it captures no semantic relationships.
2. The distributional hypothesis ("words known by their context") is the foundation of learned embeddings.
3. Word2Vec learns dense vectors by predicting words from context (CBOW) or context from words (Skip-gram).
4. Vector arithmetic on embeddings captures semantic relationships (king - man + woman ≈ queen).
5. Static embeddings have a fatal flaw: one vector per word can't handle multiple meanings (polysemy).
6. Embeddings are just learned weight matrices — they connect directly to everything in [[Neural Networks from Scratch]].
