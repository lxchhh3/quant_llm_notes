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

Concretely: the underlying model alone returned about +7.9% annualized over the walk-forward set. The full system with the LLM layer returned about +31.4%. The first number is genuinely OOS for the model weights. The second number is OOS for the model weights but in-sample for the LLM layer's design. The truthful read is: *the underlying model produces ~8% OOS, plus an unknown amount from the LLM layer that I'm currently estimating at ~+23 percentage points, but which is statistically in-sample*. Live trading is what discounts that +23pp toward its true value. I would not be surprised if the live LLM-layer contribution is half of that, or less.

The small caveat on the model itself: its hyperparameters were chosen using a tuning run on a subset of the same windows I later evaluated on. So even the +7.9% has mild hyperparameter look-ahead. This is much smaller than the LLM-layer issue, but it's not zero.

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
