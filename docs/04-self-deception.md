# Self-deception with an LLM in the loop

This is the doc I least want to write and the one most worth writing.

Walk-forward backtesting is the standard way to evaluate a quant strategy. You train your model on a window, test it on the next window, retrain, test, repeat. The model's weights are genuinely out-of-sample for each test segment — they were never exposed to that data during training. This protects against the most common form of look-ahead bias.

It does *not* protect against the kind of look-ahead bias that creeps in when you put an LLM in the loop.

## The bias

When I added the LLM layer to my system, I tuned several things:

- The prompts that drive the LLM
- The taxonomy the LLM picks from
- The macro context corpus passed in with each query
- The gate threshold and lookback
- The cadence of LLM calls

I tuned all of these against the same walk-forward dataset I was evaluating on. I knew which periods needed intervention before I designed the LLM layer. I knew which categories had worked in which periods before I built the taxonomy. I knew the gate had to fire on the bad periods and stay closed on the good ones, because I'd already seen which periods were which.

This is data-snooping. Politely, in-sample tuning. The LLM layer's reported performance on my walk-forward set is closer to in-sample than to out-of-sample.

## Why I'm saying this out loud

Most published work in this area doesn't acknowledge it. The standard pattern in LLM-augmented trading writeups is: *here's the strategy, here's the LLM layer, here's the walk-forward backtest, here are the great numbers*. The walk-forward part covers the model. It does not cover the LLM-layer design choices, which were almost always made with knowledge of the test set.

I think this is one of the bigger silent issues in the LLM-quant intersection right now. The numbers being reported are not what they appear to be. The model is OOS. The wrapper around the model usually isn't.

## What this means for my own numbers

The headline performance of my system is real in the sense that the underlying model genuinely was OOS, but the LLM-layer contribution is overstated. If I were to be honest about which parts of the result are walk-forward-clean:

- The underlying model's solo performance is OOS (with a small caveat — see below).
- The LLM layer's incremental contribution is approximately in-sample.
- The total system performance is somewhere in between, weighted by how much of the result comes from the LLM layer (which in my case was a lot).

Concretely: the underlying model alone produced a modest single-digit annualized return over the walk-forward set. The full system, with the LLM layer, reported several times that. The first number is genuinely OOS for the model weights. The second is OOS for the weights but in-sample for the LLM layer's design — most of the headline came from the part of the system that had seen the answers. The truthful read: the model produces a small real edge, plus a much larger increment from the LLM layer that is statistically in-sample and will discount toward its true value the moment it meets data it hasn't seen. I would not be surprised if the live contribution is a fraction of the backtest's.

The small caveat on the model itself: its hyperparameters were chosen using a tuning run on a subset of the same windows I later evaluated on. So even that small edge carries mild hyperparameter look-ahead. This is much smaller than the LLM-layer issue, but it's not zero.

## What to do about it

A few things I'd recommend, even though I haven't fully done them myself:

- **Hold out a true test set.** Pick a chunk of history (or, ideally, a chunk of the future via paper trading) that the LLM-layer design has never seen. Don't iterate against it. Don't peek. Treat numbers from that segment as the only ones you're allowed to publish honestly.

- **Treat live performance as the first real OOS test.** Backtest numbers on a system with an LLM layer are an *upper bound* on what live will do, not a forecast.

- **Be specific in writeups about which parts are walk-forward-clean and which aren't.** "Walk-forward backtest" without qualification implies the whole system is OOS. If only the model is, say so.

- **Be skeptical of any LLM-quant result that omits this distinction.** It's not necessarily dishonest — most authors haven't thought it through — but the numbers are very likely overstating what to expect live.

## Why this is hard to admit

The LLM-layer contribution is the headline finding of my project. Saying that the headline number is in-sample feels like undermining the work. It isn't — the design pattern is real, the underlying mechanism makes sense, and live trading will validate or refute it. But the number itself is a ceiling, not a forecast, and presenting it as a forecast would be misleading.

The tradeoff I'm making by saying this is: a smaller, less impressive-sounding result, but one that's accurate. Over a long enough horizon, accuracy is the only thing worth optimizing for. Inflated numbers are a form of debt that has to be repaid the moment someone tries to verify your claims.

## The minimal version

If you remember one thing: walk-forward protects model weights, not prompts, taxonomies, gating thresholds, or anything else you tuned by looking at the walk-forward results. If your LLM layer was iterated against the same windows you're reporting numbers on, the LLM-layer numbers are in-sample. Say so.

## Update: the live test came back

I ended the original version of this doc saying that live trading would be the first honest out-of-sample test of the LLM layer, and that I expected it to discount the backtest. It came back. Two things happened, both more humbling than I'd braced for.

First, it discounted hard. The system spent its first live month in a drawdown — a double-digit one — in a market that had not crashed. Direction as predicted; magnitude worse than I'd hoped. The ceiling was a ceiling.

Second, and more instructive: the live test wasn't even clean, and neither was the backtest I'd been comparing it against.

- The protective gate was silently broken for the entire live month (see [doc 02](02-gating-llm-calls.md)), so "live" was never testing the system I designed — it was testing the model alone, stripped of its safety layer, plus execution friction the backtest had assumed away.
- When I went back to re-examine the backtest itself, the data and the universe underneath it were both wrong (see [doc 06](06-ground-the-foundation.md)). The number I'd published as an honest "ceiling" had been computed on a foundation that didn't hold.

So the real shape of my self-deception was worse than this doc originally confessed. I'd owned up to one layer — look-ahead bias on the LLM-layer design. Underneath it sat two more layers I hadn't even suspected: broken data, and a deployed model that wasn't the one I'd validated. **Self-deception isn't a single error you confess once and clear. It's layered. Every level you've validated can be sitting on a level you haven't.**

What I actually believe now, having ground the foundation and watched the live tape: the picker has a modest, real edge, strongest in the most recent regime. Most of the original headline was foundation, look-ahead, and frictionless assumptions — not alpha. That is a much smaller claim, and a much truer one, and I'm more confident in it than I ever was in the big number.

I've kept this doc's original argument intact — I've only stripped the specific backtest figures it once quoted, because doc 06 is why I no longer trust them. Editing the *reasoning* to look prescient would be precisely the failure mode this doc is about; removing numbers I've since learned were measured on bad ground is just honesty catching up.
