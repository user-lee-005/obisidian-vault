# How Machines Learn

> Traditional programming: you write the rules. Machine learning: you give it data and it *discovers* the rules. That one flip changes everything.

---

## Learning from Data — The Core Idea

In classical programming, a developer says: *"If the email contains 'free money', mark it as spam."* You explicitly code every rule.

In machine learning, you say: *"Here are 10,000 emails labeled spam or not-spam. Figure out the patterns yourself."*

The machine doesn't understand language. It finds statistical patterns — certain words, certain structures, certain sender behaviors — that correlate with spam. It **learns a function** that maps inputs to outputs.

This is powerful because:
- Humans can't write rules for everything (how do you describe a cat in code?)
- Patterns in data can be too subtle for humans to notice
- The model improves as you feed it more data

---

## Supervised Learning

**The setup:** You have labeled data — inputs paired with correct answers.

The model learns the mapping: `input → correct output`

**Two main flavors:**

### Classification — predicting a category
- Is this email spam or not? → **Binary classification**
- Is this image a cat, dog, or bird? → **Multi-class classification**
- Sentiment analysis: positive, negative, neutral

### Regression — predicting a number
- What will this house sell for? → Predict a price
- How many shipments will we process tomorrow?
- What's the temperature going to be at 3pm?

**Examples in the wild:**
- Spam filters (classification)
- House price prediction (regression)
- Medical diagnosis — tumor malignant or benign? (classification)
- Credit scoring (regression/classification)

**The intuition:** It's like a student studying with an answer key. They see questions + correct answers, and learn to answer new questions they haven't seen before.

---

## Unsupervised Learning

**The setup:** You have data but *no labels*. The model must find structure on its own.

**Key techniques:**

### Clustering — finding natural groups
- Group customers by purchasing behavior (you don't know the groups ahead of time)
- Segment news articles by topic
- Example: K-means clustering

### Anomaly Detection — finding outliers
- Detect fraudulent transactions that "look different" from normal ones
- Find defective products in a manufacturing line
- Network intrusion detection

### Dimensionality Reduction — simplifying data
- Reduce 1000 features to 50 while keeping important patterns
- Useful for visualization and speeding up other algorithms
- Example: PCA (Principal Component Analysis)

**The intuition:** It's like sorting a pile of unsorted Lego bricks — nobody told you the categories, but you naturally group them by color, shape, or size.

---

## Reinforcement Learning

**The setup:** An agent interacts with an environment. It takes actions, receives rewards or penalties, and learns a strategy (policy) to maximize long-term reward.

**Key concepts:**
- **Agent** — the learner/decision-maker
- **Environment** — the world it interacts with
- **Action** — what the agent can do
- **State** — current situation
- **Reward** — feedback signal (positive or negative)

**The process:** Trial and error. The agent explores, sometimes fails, sometimes succeeds, and gradually learns what works.

**Classic example — game AI:**
- AlphaGo learned to play Go by playing millions of games against itself
- No human told it the "right" moves — it discovered winning strategies through reward signals

**Other examples:**
- Robotics (learning to walk, grasp objects)
- Autonomous driving (navigating without crashing)
- Resource allocation and scheduling

**The intuition:** It's like training a dog — you can't explain the rules in words, but you reward good behavior and the dog figures out what you want.

---

## Semi-supervised and Self-supervised Learning

A brief mention of two important middle-ground approaches:

- **Semi-supervised:** You have a small amount of labeled data and a large amount of unlabeled data. The model uses both. Useful when labeling is expensive (medical images).

- **Self-supervised:** The model creates its own labels from the data. For example, mask a word in a sentence and predict it — that's how models like BERT and GPT are pre-trained. This is how [[What is AI and ML|large language models]] learn from the entire internet without human labelers.

---

## When to Use Which?

| Situation | Approach |
|-----------|----------|
| You have labeled data with clear input→output pairs | **Supervised** |
| You want to find hidden patterns or groupings in data | **Unsupervised** |
| You need an agent to make sequential decisions | **Reinforcement** |
| You have some labels but mostly unlabeled data | **Semi-supervised** |
| You have massive unlabeled text/image data | **Self-supervised** |

The real world often combines these. A system might use self-supervised pre-training, then supervised fine-tuning — which is exactly how modern LLMs work.

---

## **Key Takeaways**

1. ML flips the script — instead of coding rules, you provide data and the machine discovers patterns.
2. **Supervised learning** needs labeled data and predicts categories (classification) or numbers (regression).
3. **Unsupervised learning** finds hidden structure — clusters, anomalies, or simpler representations — without labels.
4. **Reinforcement learning** learns through trial-and-error interaction with an environment, guided by rewards.
5. Self-supervised learning (powering modern LLMs) creates its own training signal from raw data.
6. Next, let's look at *what* actually does the learning — [[Neural Networks from Scratch|neural networks]].
