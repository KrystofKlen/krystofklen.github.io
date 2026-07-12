+++
date = '2026-05-01T16:59:17+02:00'
draft = false
title = 'Context window'
categories = ['AI']
tags = ['context window', 'KV cache', 'why LLM forgets information']
+++

# What is a context window?
A context window is the maximum number of tokens a language model can process and attend to during a single forward pass.

It represents the entire set of tokens visible to the model at a given moment:

- System prompt (often hidden from the user)
- User prompts
- Previous assistant responses
- Current generated output
- In reasoning models, additional internal reasoning tokens may also be generated. These are usually counted as output tokens.

# Why models forget information in long chats
As conversations grow longer, models may appear to ignore earlier information. This usually happens because the context window limit has been reached.

Once the limit is approached, the system must remove or compress older content to make room for new tokens.

Common techniques include:

## Sliding window
The oldest tokens are removed from context first. The model effectively “forgets” earlier parts of the conversation.

## Context packing
Previous conversation history may be summarized and replaced with a shorter version to save tokens.

# What happens when the context window is exceeded?
Context failures are dangerous because they are often invisible.

There is usually no explicit error message indicating that earlier information was dropped. API calls still appear successful, but the model may silently lose important instructions or details, ehich will cause hallucinations.

![my image](/images/context-window/context-window.png)

# Context window vs token limit
Context window size and token generation limits are different concepts.

The context window defines how many total tokens the model can see at once, including both input and output tokens.

A token limit usually refers specifically to the maximum number of output tokens the model can generate.

If the generation limit is reached, the response simply stops, often producing an incomplete answer.

# Why large context windows are computationally expensive
In long texts, any token can theoretically influence the meaning of any other token. Transformer models handle this using the self-attention mechanism.

Self-attention computes relationships between tokens by calculating attention scores across the entire input.

This creates quadratic complexity: O(n²).

As the number of tokens grows, computation and memory requirements increase rapidly.

## KV cache
To improve performance, models commonly use a KV (Key-Value) cache.

Previously computed token representations are stored and reused for future prompts within the same conversation.

Instead of recomputing attention for the entire chat history, the model only computes KV values for newly added tokens.

This significantly reduces latency and improves time-to-first-token, but it requires additional GPU memory.

KV caching speeds up computation but does not reduce the number of input tokens inside the context window.

# Context window paradox
Increasing the context window does not automatically improve model quality.

In practice, model performance often degrades as the context grows.

## Lost in the middle problem
Models tend to pay the most attention to information located at the beginning and end of the prompt.

Accuracy frequently drops for information placed in the middle of very large contexts, creating a U-shaped performance curve.

# Security implications
## Small context windows
If a prompt contains large amounts of distracting or deceptive data, important safety instructions may be pushed out of the context window.

This can increase hallucinations or weaken prompt-based guardrails.

Systems that rely only on textual instructions instead of architectural enforcement become less reliable when context is limited.

## Very large context windows
As context grows, models may pay less attention to information in the middle of the prompt.

Sensitive data or critical instructions may therefore be ignored even though they technically remain inside the context window.

Because of the context window paradox, prompt injections placed near the beginning or end of a prompt often have a higher probability of success.

# Useful tips
- Regularly clear long coding chats
- Keep prompts concise whenever possible
- Place the most important information near the beginning or end
- Large contexts require significantly more compute resources
- Longer prompts increase latency and memory usage
