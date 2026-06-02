# quant-llm-notes

Notes from building an LLM-augmented stat-arb system. This repo is a writeup, not a release — it documents the design patterns and lessons, not the production configuration. See [`NOT-INCLUDED.md`](NOT-INCLUDED.md) for what's deliberately omitted and why.

## What this is about

A daily-rebalanced equity strategy. A deep-learning model produces per-stock scores from technical features, a small concentrated portfolio is held with low turnover, and an LLM sits in a parallel "advisory" loop that watches for periods when the underlying model has lost calibration with the market and intervenes when it has.

The interesting part is the LLM layer. The interesting result is that it helped in a narrow, specific way that wasn't the way I expected — and that figuring out whether it helped *at all* turned out to depend on getting several much more boring things right first.

## The arc this repo documents

I first published these notes leading with a backtest headline. I was proud of the number. Since then I put the system into live paper trading, and spent the following months discovering how much of that number was *foundation* rather than *alpha*. That journey is now the spine of this repo, because it's more useful than the headline ever was.

Roughly in the order I learned what the number depended on:

- The **data** under the backtest was misaligned, and the **universe** was survivorship-biased. Correcting both moved the result by more than the entire edge I thought I had. ([doc 06](docs/06-ground-the-foundation.md))
- The model I had **deployed** wasn't trained the same way as the model I had **validated**. ([doc 06](docs/06-ground-the-foundation.md))
- The protective LLM **gate** was silently broken for the entire first live month, so the live test wasn't even testing the system I'd designed. ([doc 02](docs/02-gating-llm-calls.md))
- The binding constraint was never the model — it was **execution and exposure**, the gap between a good prediction and a filled position. ([doc 07](docs/07-the-bottleneck-wasnt-the-model.md))

What survived all of that is a model with a **modest, real edge** — strongest in the most recent regime — wrapped in an LLM layer whose contribution I now hold far more humbly than I did. I deliberately no longer lead with a single performance number, because I no longer believe a single number is an honest summary of this system. The patterns below are what I'd actually stand behind.

For the part of the evaluation that *is* clean: the model is assessed walk-forward — each segment trains on its own window and is tested on a held-out segment that follows, with no overlap, retrained from scratch per segment. That protects the model's weights. It does **not** protect the things I tuned by looking at the results, which is the subject of [doc 04](docs/04-self-deception.md) — the most important doc here, and the one I most want you to read.

## What I learned

I already knew how to build the deep-learning piece — those techniques are well documented elsewhere. The new ground was everything around it: where an LLM helps and where it hurts, and then the harder lesson of how much has to be right beneath a model before any number it produces means anything. Seven findings:

1. **[When LLMs help quant](docs/01-when-llms-help-quant.md)** — A typology of where an LLM is and isn't useful in a trading pipeline. Short version: not for picking instruments, sometimes for classifying regimes.

2. **[Gating LLM calls](docs/02-gating-llm-calls.md)** — Calling an LLM on every decision damages decisions it shouldn't have spoken on. A gate that detects model failure and only then consults the LLM matters more than the LLM itself — a claim live trading proved by breaking the gate and letting me watch.

3. **[IC is a lying compass](docs/03-ic-is-a-lying-compass.md)** — The standard factor-evaluation metric correlated almost zero with realized P&L for my strategy. Two of my best periods had near-zero IC; one of my worst had positive IC.

4. **[Self-deception with an LLM in the loop](docs/04-self-deception.md)** — Walk-forward protects model weights. It does not protect the prompts, taxonomy, or gate threshold you tuned by looking at the results. Updated with what happened when the live test came back — and with how much further down the self-deception went than I first admitted.

5. **[Dead ends](docs/05-dead-ends.md)** — A structured list of what I tried and cut. The negative results are often more transferable than the positive ones.

6. **[Ground the foundation before you trust a number](docs/06-ground-the-foundation.md)** — Data alignment, survivorship in the universe definition, and deploy hygiene. The least glamorous layer in the stack, and the one most able to silently invalidate everything above it.

7. **[The bottleneck wasn't the model](docs/07-the-bottleneck-wasnt-the-model.md)** — The idealized-to-realized gap, matching the training label to the holding period, and why a fix that rescues a weak model can sink a strong one.

## Why no code

Two reasons.

The first is straightforward IP protection. The production configuration was tuned against a specific universe and dataset over months of experiments, and copy-paste reproduction is something I'd rather not enable. See [`NOT-INCLUDED.md`](NOT-INCLUDED.md) for the explicit list.

The second is that the lessons travel; the code mostly doesn't. The exact gate threshold, the specific taxonomy, the model hyperparameters — all of those are calibrated to *my* dataset and *my* universe. They would mostly mislead someone applying them elsewhere. The patterns generalize. The values don't.

## What you can take from this

If you're building a similar system, the docs are written so you can implement your own version after reading them. The choices that took me months to converge on are described as patterns, not as recipes. Your numbers will be different. The shape of the system probably shouldn't be — and the order I'd now insist on is *foundation first, model last*, which is the reverse of how I actually did it.

## Longer essays

I'm publishing some of this material in narrative form on Substack — same ideas, more storytelling, less reference-doc structure.

- [The compass that lied to me](https://kevinlee412770.substack.com/p/the-compass-that-lied-to-me) — on why the Information Coefficient mostly didn't correlate with my strategy's P&L

## About

I work on quantitative trading systems, with an interest in LLM augmentation of statistical strategies. Reach me through GitHub.
