# Tokenization and Vocabulary

We've seen how [[The Transformer Architecture]] processes sequences and how [[Word Embeddings]] turn words into vectors. But there's a crucial step we've glossed over: **how do we break raw text into the units the model actually sees?**

This is tokenization — and the choices here have profound effects on model behavior.

---

## Why Not Just Use Words?

The obvious approach: split on spaces, treat each word as a token. But this fails badly:

- **Vocabulary explosion**: English has hundreds of thousands of word forms. "run", "runs", "running", "runner" — each is a separate entry. Many languages are worse (German compound words, Turkish agglutination).
- **Unknown words**: Any word not in your fixed vocabulary becomes `[UNK]`. New names, technical terms, typos — all invisible to the model.
- **No morphological sharing**: The model can't see that "running" and "runner" share the root "run" — they're completely unrelated indices.

---

## Character-Level: Too Far the Other Way

What if each character is a token? "Hello" becomes `['H', 'e', 'l', 'l', 'o']`.

- ✅ Tiny vocabulary (~256 characters)
- ✅ No unknown tokens ever
- ❌ Sequences become extremely long (5x or more)
- ❌ Model must learn spelling, word boundaries, and meaning all from scratch
- ❌ Long-range dependencies become even harder

We need a middle ground.

---

## Subword Tokenization: The Sweet Spot

The insight: **split common words into single tokens, but break rare words into meaningful subword pieces.**

- "cat" → `["cat"]` (common, kept whole)
- "unhappiness" → `["un", "happiness"]` or `["un", "happy", "ness"]`
- "transformers" → `["transform", "ers"]`

Common words stay intact (efficient), rare words decompose into familiar parts (generalizable). This is how all modern LLMs work.

---

## Byte-Pair Encoding (BPE)

**BPE** (Sennrich et al., 2016) is the most widely used subword algorithm. It's elegantly simple:

**Training the tokenizer:**
1. Start with a vocabulary of individual characters
2. Count all adjacent pairs in the training corpus
3. Merge the most frequent pair into a new token
4. Repeat steps 2–3 for a desired number of merges (e.g., 50,000 times)

**Walkthrough example:**

Starting corpus tokens: `l, o, w, e, s, t, n, ew, w`

- Most frequent pair: `(l, o)` → merge into `lo`
- Next: `(lo, w)` → merge into `low`
- Next: `(e, s)` → merge into `es`
- Next: `(es, t)` → merge into `est`
- Now "lowest" tokenizes as: `["low", "est"]`

**At inference time:**
- Apply learned merges greedily to new text
- Unknown words get broken into known subwords: "lowest" → `["low", "est"]`
- Even totally novel words decompose: "chatgptify" → `["chat", "g", "pt", "ify"]`

---

## WordPiece (Used by BERT)

Similar to BPE, but instead of merging the most frequent pair, it merges the pair that **maximizes the likelihood of the training data**. This is a subtle but meaningful difference — it considers how useful the merge is for the language model, not just how common it is.

WordPiece marks subwords that continue a word with `##`:
- "playing" → `["play", "##ing"]`
- "unbelievable" → `["un", "##believable"]` or `["un", "##believ", "##able"]`

---

## SentencePiece: Language-Agnostic

**SentencePiece** (Kudo & Richardson, 2018) doesn't assume spaces separate words. It treats the raw input as a stream of bytes/characters and learns segmentation from scratch.

- Works for any language (Japanese, Chinese — no spaces!)
- Treats whitespace as a regular character (often represented as `▁`)
- "Hello world" → `["▁Hello", "▁world"]`
- Used by T5, LLaMA, and many multilingual models

---

## Special Tokens

Models need tokens with special meaning beyond regular text:

| Token | Purpose |
|-------|---------|
| `[CLS]` | Classification token (BERT) — aggregate representation |
| `[SEP]` | Separates segments (sentence A from sentence B) |
| `[PAD]` | Padding for batch alignment (ignored by attention mask) |
| `[MASK]` | Masked token for BERT pre-training |
| `<\|endoftext\|>` | End of document (GPT family) |
| `<s>`, `</s>` | Beginning/end of sequence |

These are learned like any other token — the model discovers their function during training.

---

## Token Limits and Context Windows

Every Transformer has a **maximum context length** — the most tokens it can process at once. This is a hard architectural limit tied to the positional encoding and attention computation (which scales quadratically with sequence length).

- GPT-3: 2,048 tokens
- GPT-4: 8,192 / 32,768 / 128,000 tokens
- Claude: 100,000+ tokens

**Why it matters**: If your prompt + response exceeds the context window, the model literally cannot see the overflow. Long documents must be chunked, summarized, or handled with retrieval techniques.

A rough rule: **1 token ≈ ¾ of a word** in English (or about 4 characters).

---

## Practical Example: How Text Becomes Numbers

Let's trace "Hello, world!" through a tokenizer (GPT-2 style BPE):

```
"Hello, world!" → ["Hello", ",", " world", "!"] → [15496, 11, 995, 0]
```

Each token maps to an index in the vocabulary. That index selects a row from the [[Word Embeddings]] matrix. Those embeddings (plus positional encodings) become the input to [[The Transformer Architecture]].

The entire pipeline: **Raw text → Tokenizer → Token IDs → Embedding lookup → Transformer input**

---

## **Key Takeaways**

1. Word-level tokenization fails due to vocabulary explosion and inability to handle unknown words.
2. Subword tokenization (BPE, WordPiece, SentencePiece) is the sweet spot — common words stay whole, rare words decompose into meaningful pieces.
3. BPE iteratively merges the most frequent character pairs, building a vocabulary of subwords from the bottom up.
4. Special tokens ([CLS], [PAD], [MASK], etc.) serve structural roles that the model learns during training.
5. Context window limits are hard constraints — exceeding them means the model cannot see the extra tokens.
6. The full pipeline is: raw text → tokenizer → integer IDs → embedding vectors → Transformer.
