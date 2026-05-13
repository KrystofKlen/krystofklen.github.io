+++
date = '2026-05-07T16:59:17+02:00'
draft = false
title = 'Word Embeddings'
categories = ['AI']
tags = ['embeddings','semantic vector', 'euclidean distance', 'cosine similarity']
+++

# What Are Word Embeddings?
Everything inside a computer is ultimately represented as numbers. The reason large language models (LLMs) appear to “understand” words is because they transform language through many layers of abstraction into numerical representations that computers can process.

Word embeddings are numerical representations of words designed to preserve as much meaning and context as possible. They allow computers to work with language in a mathematically meaningful way.

# Semantic Vectors
Embeddings are typically represented as n-dimensional vectors, where each vector points in a specific direction within a high-dimensional vector space.

The key idea behind semantic vectors is that words with similar meanings should have vectors pointing in similar directions. This allows language models to capture relationships between words mathematically.

For example, imagine a language model with the following simplified vectors:

$$
\begin{aligned}
King   = [0.81, 0.95]\\
Queen = [0.78, 0.93]\\
Man   = [0.95, 0.25]\\
Woman = [0.92, 0.28]
\end{aligned}
$$

![my image](/images/word-embeddings/without-context.png)

Now consider the question:

What do we get if we take a king and replace the man with a woman?
The expected answer is queen.

$$
King - Man + Woman ≈ Queen
$$

Substituting with the vectors:

$$
[0.81, 0.95] - [0.95, 0.25] + [0.92, 0.28]
= [0.78, 0.98]
$$

The resulting vector [0.78, 0.98] is very close to the vector for Queen [0.78, 0.93].

This is why it is important for a language model to have vectors represnting words with similar meaning pointing into similar direction.

# Contextual Semantic Vectors
A major challenge in natural language processing is that the meaning of a word can change depending on context.

Take the word “King” as an example. In many situations, it is semantically related to words like “Queen” or “Royalty.” However, consider the sentence:

“I ordered a Whopper and Pepsi at Burger King.”
In this context, “King” should be associated more closely with words like “McDonald’s,” “Restaurant,” or “Fastfood” rather than "Queen". Therefore, the vector "King" should point to similar direction as "McDonald's" and "Fastfood"

![my image](/images/word-embeddings/without-context-2.png)

Modern language models solve this problem using the self-attention mechanism. Self-attention dynamically adjusts a numbers in the vector based on the surrounding words in the sentence. As a result, the vector points into a direction which respects the context in which the word is used.

![my image](/images/word-embeddings/with-context.png)

This context-sensitive representation is called a contextual embedding (or contextual semantic vector), and it enables language models to make much more accurate next-token predictions.

# How Is Similarity Between Vectors Measured?
To determine how similar two vectors are, language models commonly use one of the following mathematical measures.

## Euclidean Distance
Euclidean distance measures how far apart two vectors are in space.

In simple terms, it answers the question:

“How many units apart are these points?”
Smaller distance means the vectors are more similar.

$$
d(a, b) = \sqrt{\sum_i (a_i - b_i)^2}
$$

However, Euclidean distance is sensitive to vector magnitude. For example:

$$
\begin{aligned}
[0.01, 0.01]\\
[1, 1]
\end{aligned}
$$    
Both vectors point in the same direction, but they are far apart in magnitude, resulting in a large Euclidean distance.

Because of this limitation, Euclidean distance is not always ideal for measuring semantic similarity.

## Cosine Similarity
Cosine similarity measures how closely two vectors align in direction.

Unlike Euclidean distance, it ignores vector magnitude and focuses only on the angle between vectors.

In other words, it answers:

“Do these vectors point in the same direction?”

$$
\cos(\theta) = \frac{\mathbf{a} \cdot \mathbf{b}}{\|\mathbf{a}\|\;\|\mathbf{b}\|}
= \frac{\sum_{i=1}^{n} a_i b_i}{\sqrt{\sum_{i=1}^{n} a_i^2}\;\sqrt{\sum_{i=1}^{n} b_i^2}}
$$
    
- Small angle → high similarity
- Large angle → low similarity

Because semantic meaning is often better represented by direction rather than magnitude, cosine similarity is widely used in language models and vector databases.