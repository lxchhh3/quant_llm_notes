# Dead ends

A partial list of approaches I tried, kept for some period, and eventually cut. Each is summarized — the original internal version contains specifics that don't transfer cleanly outside the system they were tested on, and I've left those out.

This doc exists for two reasons. First, the *act* of cutting is more interesting than the act of building, because it's harder. Most engineering work accumulates; removing complexity that "kind of works" requires admitting it doesn't pay for itself. Second, negative results are often more transferable than positive ones — knowing what *didn't* work narrows the search space for someone else more efficiently than knowing what did.

## Defensive responses to alpha failure

When the model entered a period of alpha failure (see [doc 02](02-gating-llm-calls.md)), the obvious first instinct is to run defense. Several variants were tried; none of them helped.

**Cutting exposure to a low risk weight.** The portfolio still held the wrong stocks, just at lower weight. Net absolute loss was about as bad as before, and benchmark-relative performance was *worse* because the position was now too small to participate in the rally that was happening around it.

**Going fully to cash.** Strictly worse than the low-exposure variant. Selling costs were paid in full, and then the position sat in cash while the index continued upward. The benchmark-relative drag was the largest of any defense tried.

**Inverting the signal.** If the top picks are losing, surely the bottom picks are winning? They were not. The signal during alpha failure isn't *backwards* — it's *uninformative*. Inverting added noise without recovering edge.

**Excluding the top picks.** Reasoning: if the top of the ranking is failing, maybe the names ranked just below the top are better. They weren't. During alpha failure, the ranks *within* the top tier are no more informative than the top names themselves; the entire ranking has been calibrated against features that the new regime no longer rewards.

The lesson from all four: defensive responses treat the problem as a market problem (cut risk, hedge, hold less) or a signal problem (invert, exclude). Alpha failure is a **universe** problem — the universe of stocks the model is choosing from has rotated, and the model doesn't know. None of the defenses fix this; only changing the universe does.

## Detection mechanisms that didn't work

Before settling on a holdings-vs-benchmark gate (see [doc 02](02-gating-llm-calls.md)), several other signals for "is the model failing?" were tried.

**Rolling Information Coefficient.** The standard factor-evaluation metric, used as a regime detector. Didn't work, for reasons covered in [doc 03](03-ic-is-a-lying-compass.md). IC and head-of-distribution accuracy are different signals; using IC as a detector produced exactly the wrong calls.

**Signal-health summaries inside the LLM prompt.** I tried passing summaries of recent signal quality (IC, spread between top and bottom predictions, hit rate) to the LLM as context for its diagnosis. The LLM was *more* prone to false alarms with these inputs, not less — it weighted the noisy signal-quality metrics heavily and overrode otherwise-clean macro reasoning. Removed; the LLM now sees only macro context.

**Long-window detection.** A first version of the gate used a long lookback before declaring the model "failing." It only caught failures after most of the damage was done. Shortened until the gate caught failures while there was still room to recover.

## LLM uses that didn't work

Covered in detail in [doc 01](01-when-llms-help-quant.md). Briefly:

**LLM as stock picker — multiple variants.** Filtering top picks, picking from the full universe, full-context reasoning to specific names — all worse than trusting the underlying model's ranking. The LLM picks plausible but wrong names; the gap between "plausible" and "operationally correct" is the difference between losing and winning.

**LLM in the inner loop** (called per-stock, per-day). Considered for sentiment-style features. Never built. Latency, cost, and non-determinism are all wrong for that role.

## Model and feature choices that didn't work

**A larger feature set, hand-engineered on top of a standard published one.** The added features were technical decompositions (illiquidity proxies, intraday/overnight decomposition, volume-price divergence, autocorrelation summaries, and similar). Each was well-motivated individually. Stacking them onto the standard set reduced returns — the model spent capacity learning to ignore the noise rather than gaining from the signal. The standard set alone outperformed.

**Higher-turnover variants.** More aggressive rotation per rebalance, or shorter holding thresholds, both added cost without adding edge. The minimal-turnover configuration won on both gross fitness and net P&L.

**Larger portfolio sizes.** Holding more names diluted the model's edge into noise. There was a clear sweet spot for portfolio size; both directions away from it were worse.

**Alternative model families.** Several different model architectures were tried in earlier iterations. The production choice was selected by walk-forward performance on the validation segments. The choice is contingent on the universe — a different market may favor a different architecture, and the production choice should not be transferred without re-testing.

**A different market segment within the same universe family.** Tested with the same architecture; underperformed. The market choice matters independently of the model — it's not a hyperparameter you can transfer.

## What this list isn't

This isn't every dead end. The internal version contains experiments that were specific enough to the underlying data that summarizing them would either leak the system or mislead readers about what the result *means*. The cuts above are the ones whose lessons generalize.

It also isn't a list of bad ideas. Several of these were the obvious first thing to try, and trying them was correct — that's how you know they don't work. The list is structured this way mainly to save someone else the months it took to walk the same paths.

## Why I think this matters

Most published quant work shows what worked. Showing what *didn't* work — and why — is more useful for someone in the same situation, because the search space of "things that look reasonable but fail" is enormous, and every entry on a dead-ends list is one fewer dead end the reader has to walk into themselves.

It's also more honest about the shape of research. The system as it exists now is the residue of a much larger set of experiments. Reading only the description of the final design makes it look like a clean line from idea to result. It wasn't. Most of the work was finding things to remove.
