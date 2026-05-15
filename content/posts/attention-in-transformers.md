+++
title = "Writing One Attention Head in NumPy"
date = "2026-05-15"
draft = false
tags = ["ai", "transformers", "llms", "numpy", "from-scratch"]
categories = ["machine-learning"]
description = "Implementing a single self-attention head in ~30 lines of NumPy. What surfaces when you write the math by hand: the role of √d_k, where the causal mask goes, and why the row/column convention is the thing that actually trips you up."
[cover]
  hidden = true
+++

I had read about attention more times than I can count. Watched the 3Blue1Brown video, skimmed the Illustrated Transformer, half-read the paper. I could recite the slogan *queries, keys, values* but when I tried to picture what a single forward pass actually looked like in memory, I got fuzzy. Specifically: I was never sure, in a given diagram, whether each row of the attention matrix corresponded to one query or one key. That tiny ambiguity told me I did not really understand it yet.

So I wrote one. A single self-attention head in NumPy. No training, no GPU, no transformer stack; just the forward pass on a toy sentence, and a print of the attention matrix at the end. The whole thing is around 30 lines.

This post is what surfaced while writing it.

## The setup

The input is a short sentence, tokenized at the word level:

```python
tokens = ["a", "fluffy", "blue", "creature",
          "roamed", "the", "verdant", "forest"]
```

Real models do not tokenize at word boundaries. They use byte-pair encoding or similar, which splits `creature` into something like `cre` + `ature`. For a forward pass that does not matter. The mechanism is the same; the units are just different.

Each token needs an embedding: a vector in some `d_model`-dimensional space. In a trained model these come from a learned lookup table. Here, I just assigned each token a fixed random vector:

```python
import numpy as np

np.random.seed(0)
d_model = 16
d_head  = 8
n = len(tokens)

vocab = {t: i for i, t in enumerate(tokens)}
E = np.random.randn(len(vocab), d_model) * 0.5
X = np.stack([E[vocab[t]] for t in tokens])   # (n, d_model)
```

`X` is the matrix of token embeddings, shape `(n, d_model)`. One row per token. This row-vector layout is what PyTorch uses, and it is the convention I'll stick to for the rest of the post.

## Q, K, V are three linear projections

The whole "queries, keys, values" framing collapses, in code, to three matrix multiplies:

```python
W_Q = np.random.randn(d_model, d_head) * 0.3
W_K = np.random.randn(d_model, d_head) * 0.3
W_V = np.random.randn(d_model, d_head) * 0.3

Q = X @ W_Q   # (n, d_head)
K = X @ W_K   # (n, d_head)
V = X @ W_V   # (n, d_head)
```

That is the entire "produce a query, key, and value for every token" step. Three learned matrices, one matmul each. The conceptual story about *what a token is asking for* versus *what it can offer* is real, but in the code it is just three different projections of the same input.

Writing it this way is when the row convention clicked for me. With row-vector embeddings, `q_i = x_i W_Q`, the embedding sits on the *left*. Every visual diagram I had seen drew the weight matrix on the left and the vector on the right, which silently assumed column vectors. The math is the same; the layout is transposed; and that transposition is exactly what gets confusing when you try to line up code with diagrams.

## Scores: one matmul, then scale

To get how much every query matches every key, you take dot products of every query-key pair. That is a single matrix multiply:

```python
scores = Q @ K.T / np.sqrt(d_head)   # (n, n)
```

Row `i` of `scores` is the query at position `i` dotted against every key. Column `j` is every query dotted against the key at position `j`. The matrix is `n × n`.

The `/ np.sqrt(d_head)` is the thing the paper calls scaled dot-product attention. Without it, dot products grow with `d_head`, the variance of `scores` blows up, and softmax saturates: one entry dominates, the rest go to zero, and the gradient through softmax dies. With small embeddings this is harmless, but at `d_head = 128` it is the difference between a model that trains and one that does not.

## The causal mask

For a decoder-only model, a query at position `i` must not attend to keys at positions `j > i`. Otherwise the model could cheat during training: predicting the next token while looking at it.

The trick is to set the scores for those positions to `-inf` *before* softmax. After softmax, `exp(-inf) = 0`, so those positions contribute nothing.

```python
mask = np.triu(np.ones((n, n)), k=1).astype(bool)
scores[mask] = -np.inf
```

`np.triu(..., k=1)` is the strict upper triangle: exactly the positions where the key index is greater than the query index. Writing this line is what finally pinned down, in my head, that the mask lives in the score grid before softmax, and that "row = query, column = key" is the convention that makes the upper triangle the future.

## Softmax over the right axis

```python
def softmax(x, axis=-1):
    x = x - x.max(axis=axis, keepdims=True)
    e = np.exp(x)
    return e / e.sum(axis=axis, keepdims=True)

A = softmax(scores, axis=-1)   # softmax along rows
```

Softmax along `axis=-1` means each *row* sums to 1. Each row is the attention distribution for one query token: how much that token pulls from every other token. If you take softmax along the wrong axis here, the matrix still has shape `(n, n)` and the code still runs. It is just wrong, silently, and you will not notice until your loss does not go down.

This is the place where the row-vs-column convention has the most teeth. Pick one and stay there.

## The actual mixing

```python
out = A @ V   # (n, d_head)
```

That is it. The output is, for each query position, a weighted sum of value vectors, weighted by how much that query attended to each key. One matmul.

The full forward pass for one attention head, with masking, is:

```python
Q = X @ W_Q
K = X @ W_K
V = X @ W_V

scores = Q @ K.T / np.sqrt(d_head)
scores[np.triu(np.ones((n, n)), k=1).astype(bool)] = -np.inf
A = softmax(scores, axis=-1)
out = A @ V
```

Eight lines. That is the mechanism that drives every modern language model.

## What the attention matrix looks like (when nothing is trained)

I printed `A` rounded to two decimals:

```text
        a    fluffy  blue    creature roamed  the     verdant forest
a       1.00 0       0       0        0       0       0       0
fluffy  0.21 0.79    0       0        0       0       0       0
blue    0.08 0.55    0.37    0        0       0       0       0
creature 0.14 0.32   0.27    0.27     0       0       0       0
roamed  0.10 0.18    0.22    0.31     0.19    0       0       0
the     0.04 0.10    0.16    0.27     0.22    0.21    0       0
verdant 0.03 0.06    0.11    0.20     0.16    0.18    0.26    0
forest  0.02 0.06    0.07    0.14     0.13    0.16    0.20    0.22
```

The lower-triangular structure is real: that is the mask doing its job. But the *values* are nonsense. With random `W_Q`, `W_K`, `W_V`, there is no reason for `creature` to attend more to `fluffy` and `blue` than to `a`. It does not.

This was the most useful thing the implementation taught me. Every intuition-pump explanation of attention shows you a beautifully clean pattern where adjectives attend to the noun they modify. Those patterns are not free. They are what *training* produces, after the gradient signal from millions of next-token predictions pushes `W_Q`, `W_K`, `W_V` into configurations that align relevant tokens. The architecture only provides the *capacity* to learn such patterns. The patterns themselves are learned, not built in.

Untrained attention is just a fancy way of computing a random weighted average over previous tokens.

## What multi-head adds (briefly)

A real transformer block does not run one head. It runs many in parallel (96 in GPT-3's largest configuration), with `d_head = 128` and `d_model = 12288`. In code this is one more axis on each weight matrix:

```python
h = 4
W_Q = np.random.randn(h, d_model, d_head) * 0.3
# ...
Q = np.einsum("nd,hde->hne", X, W_Q)   # (h, n, d_head)
```

Each head runs the same eight-line forward pass independently. Their outputs are concatenated along the head dimension, giving `(n, h * d_head)`, and then a final learned projection `W_O` maps back to `d_model`.

The point of multi-head is not that one head is too small. It is that different heads can specialize: one tracks local syntax, one tracks long-range coreference, one does something nobody has a clean interpretation of. Splitting the parameter budget into multiple smaller projections lets the model learn multiple routing patterns at once, instead of forcing a single attention pattern to serve every contextual relationship.

## What I left out

This is one head. A real transformer has:

- many heads per layer, plus the output projection `W_O`;
- many layers stacked, each with its own attention block;
- a feed-forward network after attention, which is where most of the parameters in a large decoder-only model actually live;
- residual connections around each sub-block;
- layer normalization (or RMSNorm) inside the block;
- positional encoding, because attention itself is order-invariant — `Q @ K.T` does not know which token is which without it;
- a final linear layer back to vocabulary size and a softmax for next-token prediction;
- training: a loss, an optimizer, and a lot of data.

The forward pass of one head is eight lines. The rest of the transformer is the work.

## What writing it changed

Three things stuck after writing this that had not stuck after reading.

**The row/column convention is the actual hard part.** Not the matmul, not the softmax, not the conceptual story. Once you commit to "embeddings are rows, scores have queries on rows and keys on columns, softmax is over the last axis, mask is upper-triangular," everything else falls out mechanically. Get one of those wrong and the code runs and silently produces garbage.

**The scaling factor is not optional.** It is one character of code, and it is the difference between a working model and a dead one. I had read about it; I had not internalized it.

**The architecture is a substrate, not a behavior.** Attention does not "look at adjectives." Attention provides a parameterized way to mix information across positions. *Training* is what makes the mixing meaningful. Looking at the random attention matrix above was more clarifying than any animation of a trained one.

## Further reading

- [3Blue1Brown — Attention in transformers, visually explained](https://www.youtube.com/watch?v=eMlx5fFNoYc)
- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)
- [Andrej Karpathy — Let's build GPT, from scratch, in code](https://www.youtube.com/watch?v=kCc8FmEb1nY)
- [A Mathematical Framework for Transformer Circuits](https://transformer-circuits.pub/2021/framework/index.html)
