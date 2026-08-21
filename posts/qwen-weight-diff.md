# What Actually Changed Between Two Generations of the Same Model

### Qwen3.6-27B and Qwen3.8-27B have byte-identical architectures. So the entire difference is in the weights — and you can read it without downloading either one.

---

> **The question:** two releases of a 27B model, same config in every field, one clearly better than the other. Which weights moved, and are the ones that moved the ones that matter?
>
> **The short answer:** the feed-forward layers moved most and the norms barely moved at all — but transplanting the moved weights into the older model makes it *worse*. The improvement isn't located in a component. It's co-adaptive, and three of my own intermediate conclusions were measurement artefacts before I caught them.

**[→ Interactive report: every tensor, every layer](https://mohitburkule.github.io/blogs/posts/assets/qwen-weight-diff/index.html)**

---

## Table of contents

- [1. The setup](#1-the-setup)
- [2. Reading a checkpoint you haven't downloaded](#2-reading-a-checkpoint-you-havent-downloaded)
- [3. What moved](#3-what-moved)
- [4. Three findings that were really just division](#4-three-findings-that-were-really-just-division)
- [5. The one that survived: updates share a subspace](#5-the-one-that-survived-updates-share-a-subspace)
- [6. Does the quantiser agree about what matters?](#6-does-the-quantiser-agree-about-what-matters)
- [7. The transplant](#7-the-transplant)
- [8. Do the two models compute the same things?](#8-do-the-two-models-compute-the-same-things)
- [9. Did 3.8 continue from 3.6, or restart from 3.5?](#9-did-38-continue-from-36-or-restart-from-35)
- [10. What I'd take from this](#10-what-id-take-from-this)

---

## 1. The setup

Qwen3.5-27B, Qwen3.6-27B and Qwen3.8-27B report the same `config.json` in every field that
describes the network:

| | value |
|---|---|
| architecture | `Qwen3_5ForConditionalGeneration` |
| layers | 64 — as **48 Gated DeltaNet** + **16 full attention**, `full_attention_interval: 4` |
| hidden / intermediate | 5120 / 17408 |
| query / KV heads | 24 / 4, `head_dim` 256 |
| RoPE | theta 1e7, `partial_rotary_factor` 0.25, mrope `[11, 11, 10]` |
| context | 262,144 native |

Same tensor names, all 1,199 of them. Three generations, one architecture. Whatever makes 3.8
better than 3.6 is entirely in the numbers.

That hybrid layer mix is worth pausing on, because it explains a lot of this model's behaviour.
Only 16 of 64 layers keep a growing KV cache; the other 48 are Gated DeltaNet, carrying a
fixed-size recurrent state regardless of sequence length. Working it through from the config —
4 KV heads × 256 head_dim × 2 × 16 layers — gives about 16 KiB per token at 4-bit KV, which
matches what I measured running it. It's why a 27B model holds 131k of context on a 24 GB card
and still decodes at 96 tokens/second at 120k.

## 2. Reading a checkpoint you haven't downloaded

Both checkpoints in bf16 are 112 GB. I didn't want them, and didn't need them.

A safetensors file is an 8-byte header length, a JSON header giving every tensor's dtype, shape
and byte offsets, then one contiguous buffer. So one HTTP range request fetches a slice of a
single tensor, and the same slice can be pulled from both checkpoints and compared directly:

```python
n = struct.unpack("<Q", get(url, 0, 7))[0]        # header length
header = json.loads(get(url, 8, 8 + n - 1))      # dtype, shape, offsets
meta = header["model.language_model.layers.4.mlp.gate_proj.weight"]
start, end = meta["data_offsets"]
raw = get(url, 8 + n + start, 8 + n + start + take * 2 - 1)   # just this tensor
```

65,536 elements per tensor, 1,199 tensors, both models: a few hundred megabytes instead of 112 GB.
The same trick reads GGUF headers, which is how the quantisation comparison in section 6 works
without downloading two more 18 GB files.

## 3. What moved

Across all 64 layers, 1,182 tensors compared:

| Component | Tensors | Mean relative change | Mean cosine |
|---|---|---|---|
| Norms | 241 | **0.039** | 0.9986 |
| Vision tower | 112 | 0.100 | 0.9865 |
| Embedding | 5 | 0.159 | 0.9797 |
| Gated DeltaNet | 430 | 0.279 | 0.9367 |
| Full attention | 96 | 0.319 | 0.9333 |
| **MLP** | 297 | **0.424** | 0.8780 |

Two things stand out. The feed-forward layers moved by 0.42 of their own norm — light instruction
tuning moves weights by a few percent, and published analyses put a reinforcement-learning pass an
order of magnitude below a supervised one. A 0.42 relative change is not a post-training pass on
top of the previous checkpoint. It's a different training run.

And the vision tower barely moved (0.100) while the language MLPs were rebuilt. The visual encoder
was carried over; the language model was retrained.

## 4. Three findings that were really just division

This is the part worth writing down, because I got the same thing wrong three times in a row and
each one looked like a result.

**Claim one: "the changes are concentrated near the input, contradicting the literature."**
Published checkpoint-delta studies report weight updates *growing* with depth, largest in the upper
third. My numbers showed the opposite for every component. I wrote it up as a contradiction.

It wasn't. Those studies compute **absolute** ‖ΔW‖. I was computing ‖ΔW‖/‖W‖ — and attention's
baseline weight magnitude itself grows with depth, 0.151 → 0.205 from the lower third to the upper.
Dividing by a denominator that rises along the axis you're plotting inverts the trend. On the
published metric:

| Component | lower third | middle | upper third | |
|---|---|---|---|---|
| Full attention | 0.0116 | 0.0132 | **0.0268** | rises, +132% |
| Gated DeltaNet | 0.0065 | 0.0065 | 0.0071 | rises |
| MLP | 0.0076 | 0.0064 | 0.0062 | falls |

Attention updates more than double towards the output, exactly as published. The real departure is
narrow — only the MLP moved most near the input.

**Claim two: "training moved the unimportant weights, Spearman −0.89."** Unsloth's dynamic quants
spend an uneven bit budget, which makes the per-tensor quant type an independent estimate of
importance. Bits granted against relative change correlated at −0.89 across 598 tensors, monotone
across every tier. Clean story: training preserved what matters.

Also division. Quantisers grant more bits to tensors with large outliers, and those tensors have
larger ‖W‖ — bits against weight RMS correlates at **+0.83**. Against *absolute* movement the
correlation collapses to **−0.05**. It survives in one place only: restricted to feed-forward
tensors, where the norm confound is weaker, bits against absolute movement is −0.55.

**Claim three: "importance dropped about 2.7 bits across the whole model."** Two families move
uniformly across all 48 layers with zero variance — `ssm_alpha` and `ssm_beta`, both F32 → Q8_0,
−23.5 bits each. That's a decision in unsloth's tooling, not a judgement about those weights, and
it swamped every average I took.

The pattern in all three: aggregate over a systematically varying term, then report the ratio as a
property of the numerator. The measures that held up are the scale-invariant ones — cosine
similarity and principal angles — which is not a coincidence. Weight magnitude in an RMSNorm
transformer is partly gauge: you can scale a matrix and compensate in the following norm without
changing the function at all.

## 5. The one that survived: updates share a subspace

If successive generations pushed the weights the same way, you could extrapolate:
`W' = W(3.8) + α·Δ`. Testing that needs three checkpoints, which is why 3.5 is in this study.

Element-wise, the answer is no. The cosine between the two update vectors is **−0.31** — mildly
*anti*-correlated. Individual weights get pushed back and forth, and naive extrapolation points
somewhere the next generation didn't go.

But element-wise cosine asks whether the same *coordinates* moved the same way, which coordinate
noise and any sign slack destroy. The functional question is whether the updates act on the same
*directions*. So: SVD each update matrix, compare the top-64 principal subspaces.

| Comparison | Top-64 subspace overlap |
|---|---|
| Same tensor, its own Δ₁ and Δ₂ | **0.066 – 0.078** |
| Same tensor's Δ₁ against a *different* layer's Δ₂ | 0.0125 – 0.0130 |
| Random matrices, matched shape | 0.0036 |

The cross-tensor row is the control that matters, and it isn't decoration: deltas do share some
generic structure, 3.5× the random baseline, so comparing against random alone would have
overstated this. But the same-tensor overlap is roughly **6× the cross-tensor level**. Each weight
matrix has its own preferred low-dimensional subspace, and both generations of training rewrote
that matrix's own directions while reversing sign within them.

Which changes what "nudging the weights" would even mean. Not `W + αΔ` — that direction reverses.
A low-rank edit inside the subspace the training itself keeps returning to.

## 6. Does the quantiser agree about what matters?

Unsloth publishes calibrated quants for both versions at the same tier, so the bits granted to each
tensor is an importance judgement made twice, independently. Reading only the two GGUF headers:
**53% of tensors keep the same type.** 131 promoted, 269 demoted.

Excluding the uniform families from section 4, the per-layer signal is directional:

| Family | mean Δ bits | |
|---|---|---|
| `attn_k` | **+1.78** | promoted |
| `attn_output` | +1.47 | promoted |
| `attn_v` | +1.05 | promoted |
| `ffn_up` | +0.39 | promoted |
| `attn_gate` | −0.33 | demoted |
| `ffn_down` | −0.37 | demoted |
| `attn_qkv` | −0.45 | demoted |
| `ssm_out` | **−2.66** | demoted |

Importance shifted **from the linear-attention output path towards the attention projections**.
`attn_k` gained nearly two bits per weight; `ssm_out` lost 2.7. Hold that thought for the next
section.

## 7. The transplant

Everything above is weight space, and weight space is a proxy. A large parameter change can be
functionally inert; a small one can be decisive. The only way to attribute the improvement is to
move a component and measure.

Both checkpoints quantised under one policy give matched per-tensor types, which makes a component
transplant a byte-level substitution: take 3.6 as the body, substitute 3.8's tensors for one
component, run it. Perplexity over 16 chunks of 4096 tokens of real Python:

| Model | PPL | vs 3.6 | Transplant |
|---|---|---|---|
| 3.6 base | 1.7569 ± .015 | — | — |
| 3.8 base | **1.6936 ± .014** | −0.063 | — |
| 3.6 + 3.8's Gated DeltaNet | 1.7603 ± .015 | **+0.003** | 336/336 complete |
| 3.6 + 3.8's MLP | 1.9483 ± .018 | +0.191 | 192/192 complete |
| 3.6 + 3.8's attention | 2.8166 ± .049 | +1.060 | 176/192 — partial |

**No component carries the gain.** Every transplant is neutral or worse; none moves 3.6 towards
3.8's 1.6936. The improvement is co-adaptive — distributed across components trained together. So
"the MLPs moved most" does not mean the MLPs are where the improvement lives: the clean 192/192 MLP
transplant costs +0.191, three times the entire generational gain. 3.8's feed-forward layers only
work in 3.8's body.

The Gated DeltaNet row is the sharpest result here. All 336 of those tensors swapped and perplexity
did not move — +0.003, inside the error bar — even though the same tensors show a relative weight
change of 0.279. They changed substantially and it doesn't matter functionally.

And two independent methods agree on that. Unsloth's calibration demoted `ssm_out` by 2.66 bits in
the newer model; the transplant shows the whole linear-attention path is interchangeable between
versions. Weight movement in those layers overstates their functional weight.

The attention row is confounded and I won't read anything into it: llama.cpp's own Q4_K_M policy
assigned `attn_output` different types in the two files, so 16 blocks ended up with 3.8's q/k/v
against 3.6's output projection — a mismatch *inside* a single attention block. An earlier attempt
at the whole experiment was invalid for a bigger version of the same reason: mismatched
quantisation policies meant only 34 of 192 MLP tensors could be swapped, and a one-sixth transplant
attributes nothing.

## 8. Do the two models compute the same things?

Everything so far is parameters and perplexity. Neither says what the network actually computes.
`llama-eval-callback` dumps every intermediate tensor of a forward pass, so the same tokens can be
run through both models and the activations compared directly — 513 tensors, identical input.

| Activation | n | mean cosine | mean relative difference |
|---|---|---|---|
| `norm` | 64 | 0.882 | 0.462 |
| `Kcur_normed` | 16 | 0.876 | 0.512 |
| `Qcur_normed` | 16 | 0.866 | 0.531 |
| `attn_output` | 64 | 0.826 | 0.615 |
| **`l_out`** (residual stream) | 64 | **0.739** | 0.746 |
| `linear_attn_out` | 48 | 0.716 | 0.810 |
| `final_output` | 48 | 0.582 | 0.975 |
| **`ffn_out`** | 64 | **0.526** | 0.996 |

They are not the same. The residual stream sits at cosine 0.74, and feed-forward outputs at 0.53 —
a relative difference near 1.0, meaning the discrepancy is about as large as the signal. Third
independent method, same conclusion: the feed-forward path is where the change lives.

The shape is the interesting part, and it is not a drift:

| Activation | layer 0 | lower ⅓ | middle ⅓ | upper ⅓ | last layer |
|---|---|---|---|---|---|
| `l_out` | 0.900 | 0.783 | **0.681** | 0.751 | 0.755 |
| `ffn_out` | 0.802 | 0.552 | **0.429** | 0.593 | 0.714 |
| `attn_output` | 0.957 | 0.754 | 0.842 | 0.880 | 0.602 |

A U-shape. The two models start nearly aligned, diverge hardest through the middle of the stack —
`ffn_out` down to 0.43, close to orthogonal — then partially re-converge towards the output. They
share a vocabulary and have to produce a distribution over it, so the ends are pinned; the middle
is where they are free to differ, and that is where the retraining went.

Which explains the transplant result rather than merely agreeing with it. Middle-layer
representations being nearly orthogonal is *why* dropping 3.8's MLPs into 3.6 hurts: those layers
expect an input distribution 3.6's body does not produce. Co-adaptation, visible in the activations
instead of inferred from perplexity.

## 9. Did 3.8 continue from 3.6, or restart from 3.5?

Three checkpoints allow a question that two cannot: is this a sequential lineage? If
3.5 → 3.6 → 3.8 is one continuing run with roughly independent steps, the distance from 3.5 to 3.8
should be about the Pythagorean sum of the two steps, and certainly larger than either.

Absolute per-element delta magnitudes, with base weight magnitudes drifting under 0.1% (1.75% for
attention) so the comparison holds:

| Component | ‖3.5→3.6‖ | ‖3.6→3.8‖ | ‖3.5→3.8‖ | orthogonal prediction |
|---|---|---|---|---|
| MLP | 0.00284 | 0.00489 | **0.00479** | 0.00565 |
| Gated DeltaNet | 0.00350 | 0.00669 | **0.00661** | 0.00754 |
| Norms | 0.00344 | 0.00605 | **0.00586** | 0.00696 |
| Vision | 0.00126 | 0.00227 | **0.00210** | 0.00259 |
| **Full attention** | 0.00657 | 0.01777 | **0.02143** | 0.01894 |

Two different behaviours in one model.

**Attention accumulates.** ‖3.5→3.8‖ exceeds both ‖3.6→3.8‖ and the orthogonal prediction, so
those updates point roughly the same way across both generations and 3.8 is the furthest point from
3.5.

**Everything else partially reverts.** For MLP, Gated DeltaNet, norms and the vision tower, 3.8 is
*closer to 3.5 than 3.6 is*, and below the orthogonal prediction — the second update partly undid
the first. That matches the delta cosines exactly: attention −0.015 (orthogonal, distances add),
DeltaNet −0.23 and MLP −0.33 (anti-correlated, distances partly cancel).

So "3.8 is 3.6 plus more training" is inconsistent with this data for four of five components. What
the weights are consistent with is a branch from an earlier checkpoint, a partial revert, or a
merge with something upstream — and weights alone cannot distinguish those three. Worth stating
plainly rather than picking the most interesting one.

There is also a methodological trap here worth flagging, because I nearly published the wrong
version. Computed as relative change, attention looked like it reverted along with everything else.
Only the absolute distances show it accumulating. Same lesson as section 4, for the fourth time.

## 10. What I'd take from this

**Weight-space distance is a weak proxy for functional change, and it's worse than weak in an
RMSNorm transformer**, where magnitude is partly gauge. Every claim I made from a ratio needed
retracting; the ones from angles and from transplants held.

**Component transplants are cheap and they answer the question the diffs can't.** Two matched GGUFs
and `gguf-py` is an afternoon, and it turned "the MLPs moved most" into "the MLPs are the least
transferable part", which is nearly the opposite conclusion.

**Three independent methods agreed, which is the only reason I trust any of them.** The weight
diff, the transplant and the activation comparison all put the feed-forward path at the centre of
the change, from completely different evidence. Every claim that rested on one method and a ratio
had to be retracted.

**Generational gains in this family are not localised.** If you were hoping to graft an improvement
from one release onto another, or to extrapolate the update direction forward, neither survives
contact with measurement. What does survive is a subspace — each matrix has directions the training
keeps returning to — and that's where a deliberate intervention would have to live.

---

*Method and every number above: [the interactive report](https://mohitburkule.github.io/blogs/posts/assets/qwen-weight-diff/index.html) has
per-layer, per-tensor detail for all 1,182 tensors. Scripts are in
[qwen38-27b-4090-findings](https://github.com/MohitBurkule/qwen38-27b-4090-findings).*
