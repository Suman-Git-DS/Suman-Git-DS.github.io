---
layout: default
title: "KV Cache in Transformers — Explained with Simple Math (No Jargon)"
date: 2026-06-24
---

{% include nav.html %}

# KV Cache in Transformers — Explained with Simple Math (No Jargon)

> **The reason LLMs like ChatGPT are fast is not just because they have better models.**
>
> **It's because they avoid recomputing work they've already done.**
>
> One of the biggest ideas behind this is **KV Cache**.

---

## The Problem

Let's take a very simple sentence:

> **Ram is a good boy**

When a Transformer reads this sentence, it doesn't understand words directly.

Instead, every word is converted into numbers called **vectors**.

For every token, the model creates three vectors:

- **Query (Q)** → What information am I looking for?
- **Key (K)** → How should I be matched?
- **Value (V)** → What information do I contain?

---

## How Attention Works

Suppose the model has already processed:

```
Ram | is | a
```

Now it receives the next word:

```
good
```

The model asks:

> Which of the previous words should I pay attention to?

To answer this, it compares the **Query** of the current word with the **Keys** of every previous word.

```
           Query ("good")
                 │
                 ▼

      Compare with every Key

K₁ ("Ram")
K₂ ("is")
K₃ ("a")

                 │
                 ▼

         Attention Scores

                 │
                 ▼

      Weighted Sum of Values

                 │
                 ▼

      Context-aware representation
```

---

# The Inefficiency

Now imagine a document with **10,000 tokens**.

Every time a new token arrives, the model would have to recompute

- K₁
- K₂
- K₃
- ...
- K₁₀₀₀₀

again.

That is extremely expensive.

---

# The Key Observation

Past tokens never change.

Once we've computed

```
K₁
V₁
```

there is no reason to compute them again.

Instead, we simply **store them**.

This stored memory is called the **KV Cache**.

---

# How KV Cache Grows

Processing:

```
Ram
```

```
K Cache
[K₁]

V Cache
[V₁]
```

---

Processing:

```
Ram is
```

```
K Cache
[K₁ K₂]

V Cache
[V₁ V₂]
```

---

Processing:

```
Ram is a
```

```
K Cache
[K₁ K₂ K₃]

V Cache
[V₁ V₂ V₃]
```

---

Processing:

```
Ram is a good boy
```

```
K Cache
[K₁ K₂ K₃ K₄ K₅]

V Cache
[V₁ V₂ V₃ V₄ V₅]
```

The cache simply grows as new tokens arrive.

---

# What Changes?

Without KV Cache:

```
New Token

↓

Recompute ALL previous K and V

↓

Run Attention
```

---

With KV Cache:

```
New Token

↓

Compute ONLY new K and V

↓

Reuse everything already stored

↓

Run Attention
```

Nothing changes mathematically.

The model simply avoids repeating work.

---

# Why This Matters

Without KV Cache

- Recompute previous tokens every step
- Higher latency
- Higher GPU cost
- Doesn't scale well

With KV Cache

- Compute each token once
- Reuse previous computation
- Faster inference
- Lower compute cost
- Essential for long conversations

---

# A Simple Analogy

Imagine you're reading a book.

Without KV Cache:

Every time you read a new page, you go back and reread the entire book.

With KV Cache:

You remember what you've already read and continue from there.

That's exactly what the model is doing.

---

# Real-World Impact

KV Cache is one of the core ideas behind modern LLM serving systems such as:

- ChatGPT
- Claude
- Llama
- DeepSeek
- vLLM
- TensorRT-LLM
- SGLang

Without KV Cache, interactive LLMs would be significantly slower and much more expensive to operate.

---

# Try It Yourself

I've created a minimal PyTorch implementation that shows exactly how the cache grows token by token.

👉 **GitHub Repository**

https://github.com/Suman-Git-DS/kv-cache-from-scratch

You can run it in less than a minute and inspect the tensors yourself.

---

# What's Next?

In the next article, we'll build on this foundation and explore:

- Multi-Head Attention
- Real KV Cache tensor shapes
- Why KV Cache becomes the biggest memory bottleneck
- PagedAttention (the idea behind vLLM)
- FlashAttention
- Long-context inference

---

## References

- Transformer (Attention Is All You Need)
- FlashAttention
- PagedAttention (vLLM)
- SmoothQuant
- Mooncake
