---
title: "Making Full-Context AlphaGenome Fit on an RTX 3090"
date: 2026-08-19
draft: false
tags:
  - ai-research
  - performance-engineering
  - genomics
  - gpu
  - jax
---

# Making Full-Context AlphaGenome Fit on an RTX 3090

AlphaGenome is one of those models that makes you want to try it locally and
then immediately reminds you that "open weights" and "fits on my GPU" are not
the same thing.

The model takes up to one million DNA base pairs and predicts thousands of
functional genomic signals: RNA expression, chromatin accessibility, histone
marks, transcription-factor binding, splicing, and contact maps. That long
context is not ornamental. Regulatory elements can act over large genomic
distances, so cropping the sequence can remove exactly the interaction one is
trying to study. The [AlphaGenome paper](https://www.nature.com/articles/s41586-025-10014-0)
reports a 1 Mb input and outputs across 11 modalities; the
[released research code](https://github.com/google-deepmind/alphagenome_research)
recommends at least an NVIDIA H100.

I had an RTX 3090 with 24 GiB of memory.

This post is about how I got the released AlphaGenome checkpoint to execute its
full 1,048,576 bp trunk and RNA head on that card. The short version is that I
did not make the model smaller. I replaced one memory-hungry implementation of
attention with a tiled online-softmax kernel, written in JAX Pallas, that is
similar in spirit to FlashAttention and understands AlphaGenome's unusual
learned pair bias.

The final run used 12,530 MiB of sampled process VRAM and took a median 1.511
seconds for a synchronized warm RNA-head execution. There are important
qualifications behind that sentence, and I will get to all of them.

---

## The first idea was not the best idea

I did not begin by writing a GPU kernel. My first project was adaptive context:
run a cheap short-context prediction, escalate difficult examples to a longer
window, and avoid paying for 524 kb on every variant.

That experiment was useful precisely because the answer was not especially
flattering. On a held-out set of 500 GTEx eQTL variants, a fixed 131 kb-to-524
kb missingness policy recovered the 524 kb reference's coverage while invoking
the long context for 15.8% of variants. It saved 53.24% of the measured GPU time
relative to always running 524 kb. But it was 41.53% slower than simply always
running 131 kb, and its conditional biological rank correlation was lower than
the 524 kb reference. A three-stage 16/131/524 kb cascade was worse as a systems
idea: the mandatory early passes ate almost all the savings, leaving only 2.64%
versus always-524 kb.

So the adaptive policy bought coverage, not a clean general speedup. It also
left the awkward fact that my "teacher" was only 524 kb because the 1 Mb model
did not fit.

That failure changed the question. Instead of asking _which examples can avoid
the expensive model?_, I started asking _why is the exact model expensive in
the first place?_

## Profiling pointed at one very large object

AlphaGenome compresses the one-million-base input before its sequence
transformer. At full context, the transformer sees 8,192 positions. Its
sequence attention has eight query heads, with shared keys and values.

A dense FP32 attention tensor therefore has shape:

```text
[batch, heads, query positions, key positions]
[1,     8,     8192,            8192]  = 2 GiB
```

That is one tensor. The StableHLO contained this shape through the learned-bias,
logit, soft-cap, and softmax path in each of nine transformer blocks. The dense
pair bias was another costly surprise: a compact pair representation was
projected and then repeated 16 times along both sequence axes to make the full
attention bias.

The attention _arithmetic_ is quadratic, and this work does not change that.
The important observation is that attention does not need quadratic _working
memory_. We do not need to keep the complete logit or probability matrix if we
can process it a tile at a time.

This is the central idea behind
[FlashAttention](https://proceedings.neurips.cc/paper_files/paper/2022/hash/67d57c32e20fd0a7a302cb81d36e40d5-Abstract-Conference.html):
reorder exact attention around the GPU memory hierarchy so that blocks of the
matrix pass through fast on-chip memory, rather than repeatedly writing the
whole matrix to high-bandwidth memory. FlashAttention itself is established
work. My problem was making the idea fit AlphaGenome's attention equation.

## Why stock FlashAttention was not a drop-in replacement

AlphaGenome's sequence attention is not the usual `softmax(QK^T)V` call.
For the released checkpoint, the relevant tensors are:

```text
Q: [B, S, 8, 128]
K: [B, S, 1, 128]   shared across query heads
V: [B, S, 1, 192]   shared, and wider than K
```

It also has a learned bias from pair features. The compact bias has one cell per
16 sequence positions on each axis. In the original dense implementation, each
cell is repeated over a 16 x 16 square. Finally, AlphaGenome soft-caps logits
before softmax:

$$
L_{ij} = 5\tanh\left(\frac{Q_iK_j^T/\sqrt{128} + b_{\lfloor i/16\rfloor,\lfloor j/16\rfloor}}{5}\right).
$$

Any replacement had to preserve all of that:

1. multi-query attention with eight query heads but one K/V head;
2. a 128-wide Q/K and a 192-wide value;
3. the checkpoint's learned pair bias;
4. the 16-position nearest-neighbour bias expansion;
5. the tanh soft cap before softmax; and
6. BF16 matrix products with FP32 accumulation.

Ignoring the bias would have made a wonderfully small program that computed the
wrong model.

## Online softmax, one block at a time

The kernel assigns one GPU program to a batch, head, and 64-position query
tile. It holds that query tile fixed and walks over the keys and values in
64-position tiles.

For each row it maintains only three pieces of FP32 state:

- the largest logit seen so far, $m$;
- the running softmax denominator, $d$; and
- the running weighted-value numerator, $n$.

Suppose a new key tile has row maximum $m_t$. The stable update is:

$$
m' = \max(m, m_t)
$$

$$
d' = d\,e^{m-m'} + \sum_j e^{L_j-m'}
$$

$$
n' = n\,e^{m-m'} + \sum_j e^{L_j-m'}V_j.
$$

After the final key tile, the output is $n/d$. This is the same softmax, apart
from normal floating-point reordering; it is not a sparse or approximate
attention method.

The bias is never expanded globally. For query position $i$ and key position
$j$, the kernel loads the compact cell at
`bias[i // 16, j // 16]` for the current 64 x 64 tile, adds it to the dot
product, and applies the soft cap. The largest sequence-sized temporary inside
the kernel is gone.

The development path was deliberately boring:

```text
dense implementation
        |
        v
pure-JAX tiled oracle
        |
        v
Pallas/Triton 64 x 64 kernel
        |
        v
opt-in AlphaGenome backend
        |
        v
16 kb -> 131 kb -> 524 kb -> guarded 1 Mb run
```

The original dense backend remains the default. The tiled backend uses the same
Haiku parameter and state tree, so the released checkpoint does not need to be
converted. It is currently inference-only and specialized to AlphaGenome's
actual 128/192 dimensions and full 64-position tiles.

## Correctness before the heroic run

Memory optimization is not useful if it quietly changes the biology. I used a
progressive validation ladder rather than jumping directly to a result that the
dense path could not check on this GPU.

On randomized standalone attention at sequence length 512, the Pallas kernel's
maximum absolute error against dense attention was 0.001953125, the mean error
was roughly $2.4\times10^{-7}$, and cosine similarity was effectively one.

Then I loaded one checkpoint state and compared the dense and tiled full model
on identical deterministic inputs:

|      Input | Output | Elements outside tolerance | Maximum absolute error |
| ---------: | ------ | -------------------------: | ---------------------: |
| 131,072 bp | RNA    |            0 / 100,663,296 |              0.0078125 |
| 131,072 bp | trunk  |            7 / 204,996,608 |                  0.125 |
| 524,288 bp | RNA    |            0 / 402,653,184 |            0.009765625 |
| 524,288 bp | trunk  |           15 / 826,277,888 |                  0.125 |

The tolerance was `atol=0.02, rtol=0.02`, with a predeclared hard maximum error
of 0.1. This means the 131 kb and 524 kb source reports formally failed because
of the rare 0.125 trunk outliers. I kept those reports failed. Separate,
hash-bound engineering adjudications allowed the memory experiment to proceed
because the outlier fractions were below $4\times10^{-8}$ and the RNA outputs
passed strictly.

That distinction matters. The tiled backend is numerically near-equivalent in
BF16, not bitwise identical. Different matrix tilings change floating-point
reduction order. The prediction outputs we inspected agreed within the declared
tolerance; a tiny number of intermediate trunk elements crossed the harder
ceiling.

## The NaNs that were supposed to be there

The first 1 Mb RNA run looked bad: 105,906,176 outputs were NaN. That exact
number turned out to be the clue.

The human RNA head has 768 physical tracks, but only 667 are biological tracks.
The other 101 are padding used for the model's track layout. Their metadata
means are NaN. During the normal output-unscaling step, every position in those
padding tracks becomes NaN:

```text
1,048,576 positions x 101 padding tracks = 105,906,176 expected NaNs
```

Before metadata unscaling, all 805,306,368 head values were finite. After
unscaling, every value in all 667 biological tracks was finite, and every value
in the 101 padding tracks was NaN—exactly the metadata-defined pattern.

This was a useful reminder that `isfinite().all()` can be a bad test when a
model has structured padding. The corrected harness checks the official track
mask and reports biological and padding tracks separately.

## What ran on the RTX 3090

Here is the final measured 1 Mb RNA-head result:

| Measurement                 |                          Result |
| --------------------------- | ------------------------------: |
| Input length                |                    1,048,576 bp |
| Transformer length          |                           8,192 |
| Biological RNA tracks       |                             667 |
| Finite biological values    |       699,400,192 / 699,400,192 |
| Expected padding tracks     |                             101 |
| Expected padding NaNs       |       105,906,176 / 105,906,176 |
| Warmups / repeats           |                          2 / 10 |
| Median synchronized latency |                       1.51096 s |
| p95 synchronized latency    |                       1.51511 s |
| Compilation time            |                         42.50 s |
| Sampled peak process VRAM   |                      12,530 MiB |
| GPU                         | NVIDIA GeForce RTX 3090, 24 GiB |

JAX's usual BFC allocator initially failed on a large allocation. The successful
run used JAX's experimental `address` allocator, which talks directly to the
PJRT allocator; the options and trade-offs are documented in the
[JAX GPU memory guide](https://docs.jax.dev/en/latest/gpu_memory_allocation.html).
Allocator choice did not remove the attention matrices—the tiled kernel did
that—but it mattered for fitting the remaining large activations into fragmented
device memory.

One subtle but important qualification: the compiled benchmark reduces the
enormous RNA output to finite-count, min, max, mean, and RMS scalars _on the
GPU_. It does this so that measuring feasibility does not retain and transfer a
roughly 700-million-value result to the host. The trunk and RNA head really did
execute at full context; this is not yet a demonstration of the complete public
multimodal output API or an end-to-end REF/ALT variant-scoring workflow at 1 Mb.

## Has anyone else done this?

"AlphaGenome on consumer hardware" covers several different claims. Shorter
contexts, fewer output heads, framework ports, and frozen-head workflows can all
reduce the footprint. Those are useful, but they are not the same as preserving
the full one-million-base context.

The best public comparison I found is the independent
[GenomicsXAI GPU benchmark](https://genomicsxai.github.io/blogs/2026-005/).
It reports approximately 34.6--36.1 GB for official JAX and 40.7--40.8 GB for
the community PyTorch path at 1 Mb. Its 24 GB RTX 6000 completed the PyTorch
path at 524 kb with 12.5 GB peak memory; the 1 Mb entry is an out-of-memory dash.

Those measurements are not directly comparable to mine. The GPU models,
software, allocators, requested outputs, and measurement methods differ, and my
executable returns scalar summaries instead of the entire output tensor. I
would not divide the two numbers and announce a universal memory reduction.
What the comparison does establish is the motivation: public stock-implementation
benchmarks put 1 Mb outside a 24 GB card, while this specialized RNA-head run
completed on one.

FlashAttention is not new, and neither is using it in genomics. The potentially
new part is adapting exact tiled attention to AlphaGenome's multi-query layout,
unequal value width, learned binned pair bias, and soft cap while preserving the
released checkpoint.

The careful novelty statement is:

> To our knowledge, this is the first publicly documented adaptation of
> AlphaGenome's learned-bias sequence attention to a FlashAttention-style tiled
> kernel, demonstrating full 1 Mb trunk and RNA-head computation on a 24 GB RTX 3090.

That is a "to our knowledge," not a claim to have searched every private
repository or unpublished project. A public code release and independent
replication would make it much stronger.

## What this does—and does not—mean

We did not shrink AlphaGenome. There was no pruning, distillation,
quantization, smaller checkpoint, shorter crop, or retraining. The optimization
changes _how_ the same attention equation is evaluated, not what weights or
context it uses.

What has been demonstrated:

- the released checkpoint and full 1,048,576 bp context;
- all nine sequence-attention blocks using the tiled custom call;
- the full trunk and RNA prediction head on an RTX 3090;
- finite biological RNA outputs; and
- direct dense comparisons through 524 kb.

What has not yet been demonstrated:

- a complete 1 Mb dense-versus-tiled comparison on larger hardware;
- transfer of all full-resolution outputs through the ordinary public API;
- every AlphaGenome modality at 1 Mb;
- a complete REF/ALT variant-effect scoring run at 1 Mb;
- backward propagation or fine-tuning with this kernel; or
- support for arbitrary dimensions, masks, tails, or GPUs beyond this tested
  configuration.

The model is also still quadratic in compute. Tiling removes the quadratic
attention storage, not the dot products. At longer sequences or larger batches,
runtime will eventually become the next wall.

## Reproducing the experiment

The experimental tree keeps the dense backend as the default and enables the
new path with `attention_backend="pallas_tiled"`. The staged 1 Mb harness runs
four fresh processes: trunk compile preflight, trunk execution, RNA compile
preflight, and RNA execution. Each stage is guarded by the hashes of the
shorter-context validation and adjudication reports.

The key implementation pieces are:

```text
alphagenome_research/model/tiled_attention.py
    pure-JAX reference implementation

alphagenome_research/model/pallas_tiled_attention.py
    forward-only 64 x 64 Pallas/Triton kernel

alphagenome_flash_fyp/scripts/validate_end_to_end.py
    dense-versus-tiled validation ladder

alphagenome_flash_fyp/scripts/run_1mb_staged.sh
    guarded full-context experiment

alphagenome_flash_fyp/results/execution_1mb_rna_prediction_masked.json
    primary machine-readable result
```

The experiment used JAX 0.11.0 and included a narrow workaround for that
version's missing RTX 3090 name in its Triton GPU registry—the A10 and RTX 3090
share compute capability 8.6. This is prototype engineering, not a reason to
fork a compiler forever.

## Where I would take it next

The most valuable next run is not another synthetic benchmark. It is a
scorer-specific REF/ALT experiment that performs two full-context predictions
sequentially and returns a compact biological variant score. That would test the
workflow people actually care about without requiring the host to hold every
raw track.

After that:

1. run a direct 1 Mb dense comparison on a 48/80 GB GPU;
2. test the remaining output heads and the normal public output wrappers;
3. add tail handling and broader GPU coverage;
4. profile whether AlphaGenome's pair row-attention becomes the next bottleneck;
5. tune tiles and pipeline stages across Ampere and newer GPUs; and
6. investigate a backward kernel only if fine-tuning becomes a real use case.

The larger lesson is pleasantly mundane. Before changing a model, inspect the
objects it allocates. My first approach tried to avoid full-context inference by
making a decision around the model. Profiling showed a better seam: keep the
model and change the implementation of its worst operation.

That is what made one million bases fit on the gaming GPU under my desk.
