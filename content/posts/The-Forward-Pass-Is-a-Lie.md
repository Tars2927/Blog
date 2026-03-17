# The Forward Pass Is a Lie

> *Modern inference engines have reconstructed what "running a model" means so thoroughly that the textbook forward pass barely resembles what executes on your GPU.*

**Tags:** AI/ML Internals · Systems · LLM Inference  
**Read time:** ~12 min

---

Every ML course teaches the same story. You have a model. You feed in tokens. Each layer transforms the hidden state. You get logits out. *That's the forward pass.* Clean, sequential, mathematically elegant.

That story is not wrong exactly. It's just not what happens when you serve 70B parameters to thousands of concurrent users. The gap between the textbook diagram and production reality is enormous — and understanding it is the difference between an ML practitioner and an ML *engineer*.

> "What inference engines actually do would be unrecognizable to the person who wrote the original training loop."

---

## 01 · The Textbook Version

In the standard mental model, a transformer forward pass looks like this: embed input tokens, pass through `N` attention + MLP layers sequentially, apply a final linear projection, softmax, sample. Each operation waits for the previous one. The whole graph executes once per generated token.

```
Tokenize → Embed → Layer 1…N (attn + MLP) → LM Head → Sample → repeat
```

This model is fine for research. For a single batch, offline, on one GPU. The moment you add latency budgets, concurrent users, and commodity hardware, it collapses.

---

## 02 · Problem 1: The KV Cache Blows Up Memory

Attention requires access to the keys and values of every previous token in the context. Recomputing them on every decoding step would be catastrophically slow, so engines cache them — the famous KV cache. But the cache isn't free.

For a 70B parameter model with **96 layers**, a context of 4096 tokens, and FP16 precision, the KV cache for a *single sequence* consumes roughly **~10 GB**. Serve 20 concurrent users and you've blown the entire GPU memory on cache alone — before a single parameter weight is loaded.

This is why engines like **vLLM** invented *paged attention*: rather than pre-allocating one giant contiguous block per sequence (which wastes memory on unused future context), memory is managed in fixed-size pages, allocated on demand, and shared across sequences where possible — exactly like virtual memory in an OS. The "forward pass" now includes a memory pager running alongside the compute graph.

---

## 03 · Problem 2: Sequential Decoding Is GPU Poison

GPUs are throughput machines. They are miserable at sequential, latency-bound workloads. Autoregressive decoding — generating one token, then the next, then the next — is about as sequential as it gets. While the GPU waits to know the previous token before computing the next, utilization craters.

The solution is *continuous batching* (sometimes called "in-flight batching"). Instead of processing one request at a time, the engine maintains a running batch that is constantly updated: finished sequences are evicted, new ones are added, and every forward pass step processes a heterogeneous mix of sequences at different positions in their generation.

```
Step t:
  Seq A (token 47 of 200)
+ Seq B (token 3 of 50)
+ Seq C (prefill step 1)
+ Seq D (token 198 — done → evicted)
─────────────────────────────────
→ One fused kernel, all in parallel
```

The "batch" of the textbook is now a dynamically reshuffled collection of states that may share nothing except a GPU call.

---

## 04 · Problem 3: The Attention Kernel Doesn't Run As Written

The attention formula — `softmax(QK^T / √d) · V` — looks like four operations. In practice it cannot be naively executed as four separate CUDA kernels without incurring catastrophic memory bandwidth overhead: the full `QK^T` matrix for a long sequence is enormous, and writing it to HBM then reading it back for the softmax and multiply is a bottleneck that dwarfs the actual compute.

*Flash Attention* (Dao et al., 2022 and subsequent versions) rewrites this as a single fused kernel that tiles the computation in SRAM, never materializing the full attention matrix in global memory. It produces numerically identical results but achieves **2–4× speedups** on long sequences, with memory scaling linearly rather than quadratically in context length.

The "forward pass" now runs a kernel that doesn't correspond to any single operation in the original PyTorch graph.

---

## 05 · Problem 4: Prefill and Decode Are Different Problems

When a user sends a prompt, the engine processes the full input context in one shot — this is *prefill*. Then it generates tokens one at a time — this is *decode*. These two phases have opposite GPU utilization profiles:

| Phase | Bottleneck | GPU Behavior |
|-------|-----------|--------------|
| Prefill | Compute-bound | Long prompt, big matmuls — GPU is happy |
| Decode | Memory-bandwidth-bound | One token at a time, streaming weights — GPU is mostly idle |

Treating them identically in a single batched loop means you're always suboptimal for one of them.

State-of-the-art engines are now exploring *disaggregated prefill-decode*: routing the two phases to different GPU pools, or even different machines, tuned for their respective bottlenecks. Your "single forward pass" is now a distributed system call.

---

## 06 · Why This Matters Beyond Performance

These aren't just engineering tricks. They change how you reason about the system.

**Speculative decoding** — where a small draft model proposes several tokens and a large verifier model accepts or rejects them in parallel — breaks the assumption that token `t+1` cannot be started before token `t` is finalized. The forward pass is now a probabilistic pipeline with rollback.

**Mixture-of-Experts (MoE)** models add another layer: not all parameters are active for every token. The router decides, at inference time, which experts to invoke — meaning the compute graph is dynamically determined by the input. There is no single static "forward pass" graph anymore; it's an execution that branches per token.

> "Speculative decoding, Flash Attention, paged KV, continuous batching, MoE routing — none of these appear in the training code. They are entirely an inference-time construction."

### The myth vs. the reality

| The Myth | The Reality |
|----------|-------------|
| One sequential forward pass | Fused kernels, paged memory, dynamic batch |
| Fixed, pre-allocated KV cache | On-demand paged KV allocation (vLLM) |
| Attention = four ops in order | Flash Attention: tiled, in SRAM |
| Batch = static set of sequences | Continuous batching: heterogeneous, live |
| Prefill and decode are the same | Often disaggregated to separate hardware |

---

## 07 · So What Is the Forward Pass?

It's still a useful abstraction. The mathematical semantics are preserved — input in, probability distribution out — and that's all you need for most purposes. But the moment you care about latency, throughput, or memory, you need to think at a different level.

The forward pass, in production, is a coordination protocol between a memory pager, a kernel scheduler, a batching engine, and a sampling process. Some of these components run on different devices. Some overlap with each other. Some rewrite the math entirely while preserving outputs.

The researchers who trained the model probably don't know most of this. The engineers who deployed it had to invent it. That gap — between training-time abstraction and inference-time reality — is where some of the most interesting systems work in ML is happening right now.

Understanding it isn't optional anymore. It's the difference between knowing how a model works and knowing how to *use* one at scale.

---

## Further Reading

- **Flash Attention** — Dao et al., 2022
- **vLLM Paged Attention** — Kwon et al., 2023
- **Speculative Decoding** — Chen et al., 2023
- **Orca Continuous Batching** — Yu et al., 2022
