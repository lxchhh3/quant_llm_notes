# quant-llm-notes

Notes from building an LLM-augmented stat-arb system. This repo is a writeup, not a release — it documents the design patterns and lessons, not the production configuration. See [`NOT-INCLUDED.md`](NOT-INCLUDED.md) for what's deliberately omitted and why.

## What this is about

A daily-rebalanced equity strategy. A deep learning model produces per-stock scores from technical features, a small concentrated portfolio is held with low turnover, and an LLM sits in a parallel "advisory" loop that watches for periods when the underlying model has lost calibration with the market and intervenes when it has.

The interesting part is the LLM layer. The interesting result is that it was useful in a narrow, specific way that wasn't the way I expected.

## Performance, with caveats

Across nine non-overlapping walk-forward test windows spanning roughly thirteen years:

|                       | Underlying model alone | Full system (model + LLM layer) |
|-----------------------|------------------------|---------------------------------|
| Annualized return     | +7.9%                  | **+31.4%**                      |
| Information ratio     | 0.37                   | **1.08**                        |
| Win rate              | 56%                    | **67%**                         |
| Max drawdown          | −17.9%                 | −17.2%                          |

The improvement was not evenly spread. The LLM layer was approximately neutral on most windows and decisive on two — where it turned what would have been catastrophic losses into the largest gains in the test set. One of those windows went from roughly a two-thirds loss to a ~70% gain. The other roughly doubled an already-strong gain.

Methodology: each window trains on its own segment, validates on the next, tests on a held-out segment that follows. No overlap between test segments. The model is retrained per window from scratch.

**Important caveat.** These numbers reflect look-ahead bias on the LLM layer that walk-forward does not protect against. The underlying model's weights are genuinely out-of-sample for each test segment, but the prompts, taxonomy, gating threshold, and macro context corpus were all designed with knowledge of which windows needed intervention. Treat the headline number as an *upper bound* on what live trading would produce, not a forecast. See [`docs/04-self-deception.md`](docs/04-self-deception.md). I'm running paper trading now; that's the first real OOS test.

## The four lessons

I already knew how to build the deep-learning piece — the techniques are well documented elsewhere. The new ground was figuring out where an LLM helps and where it doesn't. Four findings stood out, each written up in its own doc.

1. **[When LLMs help quant](docs/01-when-llms-help-quant.md)** — A typology of where an LLM is and isn't useful in a trading pipeline. Short version: not for picking instruments, sometimes for classifying regimes.

2. **[Gating LLM calls](docs/02-gating-llm-calls.md)** — Calling an LLM on every decision damages many decisions it shouldn't have spoken on. A simple gate that detects model failure and only then consults the LLM is more important than the LLM itself.

3. **[IC is a lying compass](docs/03-ic-is-a-lying-compass.md)** — The Information Coefficient is the standard factor-evaluation metric. For my strategy, it correlated almost zero with realized P&L. Two of my best periods had near-zero IC; one of my worst had positive IC.

4. **[Self-deception with an LLM in the loop](docs/04-self-deception.md)** — Walk-forward backtesting protects you from look-ahead bias on model weights. It does not protect you from look-ahead bias on the prompts you wrote, the taxonomy you defined, or the gate threshold you tuned. Uncomfortable to admit; worth saying out loud.

## Why no code

Two reasons.

The first is straightforward IP protection. The production configuration was tuned against a specific universe and dataset over months of experiments, and copy-paste reproduction is something I'd rather not enable. See [`NOT-INCLUDED.md`](NOT-INCLUDED.md) for the explicit list.

The second is that the lessons travel; the code mostly doesn't. The exact gate threshold, the specific taxonomy, the model hyperparameters — all of those are calibrated to *my* dataset and *my* universe. They would mostly mislead someone applying them elsewhere. The patterns generalize. The values don't.

## What you can take from this

If you're building a similar system, the docs are written so you can implement your own version after reading them. The choices that took me months to converge on are described as patterns, not as recipes. Your numbers will be different. The shape of the system probably shouldn't be.

## Longer essays

I'm publishing some of this material in narrative form on Substack — same ideas, more storytelling, less reference-doc structure.

- [The compass that lied to me](https://kevinlee412770.substack.com/p/60c) — on why the Information Coefficient mostly didn't correlate with my strategy's P&L

## About

I work on quantitative trading systems, with an interest in LLM augmentation of statistical strategies. Reach me through GitHub.
