The new frontier model for AgTech. 

Build the engineers at Irrigant.

# Helios Neural Network (HNN)

## Technical README

HNN is the V2 forecasting core of Helios, replacing the XGBoost Helios V1 It is a recurrent network in the LSTM
family, not a transformer. It carries a fixed-size hidden state forward
through time. Its compute cost per timestep is constant, no matter how many
days of field history it has already processed. It does not carry a
growing key-value cache.

---

## 1 · Why not a transformer

A transformer's cost per step grows with context length. The reason is the
key-value cache: one entry per past token, held in memory for the life of
the sequence. A field's full irrigation history should not force a growing
cache just to answer today's question.

An LSTM's hidden state is fixed size. It does not grow with history. That
fixed size is what lets it sit in SRAM instead of DRAM or HBM.

This is a deliberate architecture bet. Recurrent architectures with a
constant hidden state and constant FLOPs per step will outperform
transformers on tasks shaped like this one. That holds once external
memory replaces the KV cache as the way to handle long context. The bet is
specific to niche, narrow-fact domains like farming. It is not a general
claim against transformers everywhere.

## 2 · The core design: weights hold functions, not facts

HNN's weights do not memorize field-specific facts: this field's soil
texture, last season's yield, the calibration offset of probe 4. The
weights learn only the functions to read, write, and manipulate facts held
outside the model.

Think of a notebook, not a memorized script. The model looks a fact up when
it needs it, instead of carrying every fact in its parameters.

Facts live in an external memory store, reached through a tool-use
interface: the same shape as Mem0, RAG, or RLM-style read and write memory.
The weights encode how to search and use that store. They do not encode
what the store contains.

For a narrow domain like irrigation, this fits well. The model does not
need to know who Kevin Costner is or how many people live in Paris. It
needs to manipulate a bounded set of field records, weather series, and
probe readings. General world knowledge inside the weight file is waste for
this problem.

## 3 · Memory tiers

HNN reads and writes across three tiers:

| Tier | What it holds | Where it lives |
| --- | --- | --- |
| Short-term | The hidden state carried between adjacent timesteps | SRAM, fixed size |
| Long-term (weights) | The read, write, and retrieve functions, learned once in training | On-chip, fixed size |
| External memory | Field history, probe calibrations, irrigation events, weather archives | CPU RAM or disk, addressed in small amounts per query |

External memory access is sparse and targeted: a lookup or a small
regex-shaped query, not a dense similarity scan over a large matrix.

## 4 · Saliency and forgetting are learned, not labeled

HNN learns its own saliency function. It decides what to keep in the
hidden state, what to write out to external memory, and what to forget
entirely. No human demonstration traces teach this split. It is trained
end to end with reinforcement learning, rewarded on downstream forecast
accuracy.

## 5 · Hardware implication

The hidden state is fixed size and lives in SRAM. External memory is
addressed sparsely, not scanned densely. HNN therefore prefers chips with
large SRAM, a high count of streaming multiprocessors, and comparatively
little or no DRAM. This shapes which accelerators are worth targeting as
the architecture matures.

## 6 · Scope note

This document describes HNN's architecture direction for one narrow-fact
domain: field-scale soil water and irrigation forecasting. It is not a
general claim that LSTM or RNN architectures beat transformers everywhere.
It is a claim that they win for niche industries like farming, where the
model needs few general facts and a lot of fast, constant-cost sequential
updating over structured external memory.
