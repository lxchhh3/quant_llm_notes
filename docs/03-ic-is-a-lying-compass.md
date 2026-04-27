# IC is a lying compass

The Information Coefficient (IC) — typically the rank correlation between your model's predicted returns and realized returns across a cross-section of stocks — is the most popular evaluation metric in factor research. Every textbook recommends it. Every quant pipeline computes it.

For my strategy, IC was almost uncorrelated with realized P&L. Two of my best periods had near-zero or slightly negative IC. One of my worst periods had a marginally *positive* IC. Treating IC as a regime indicator or as a feature for the LLM gate would have produced exactly the wrong calls.

This isn't a deep finding. It follows from how IC is computed and what my strategy does with the predictions.

## Why IC misleads a top-K strategy

IC is a cross-sectional metric. It rewards a model whose predictions are correctly ordered across the *whole* universe — including the names you're never going to hold.

My strategy holds a small concentrated top-K portfolio. It only cares about the head of the distribution. A model can have a high IC because it correctly ranks the bottom 80% of the universe (which is easier — bad stocks really are bad) while getting the top names mostly wrong. The IC looks fine. The portfolio bleeds money.

The reverse also happens. A model can have negative IC because it gets the bottom of the distribution slightly inverted — but if its top picks are the right names, the strategy makes money. This was the case for one of my best periods.

## Why IC misleads a regime detector

I considered using rolling IC as an input to the LLM gate. The hypothesis: when IC drops, the model is failing, so consult the LLM.

This failed for the same reason as above: IC and head-of-distribution accuracy are not the same signal. IC was up when the head was wrong; IC was down when the head was right. Worse, IC moves slowly — by the time rolling IC clearly drops, the strategy has often already taken most of the damage. Even when the signal would have been correct, it was too late.

I removed IC from the gate entirely and switched to a metric based on what the strategy was actually holding versus the benchmark. That worked.

## The general lesson

Validate metrics against what you actually optimize.

IC is a research metric. It's useful for comparing two factors, for understanding whether a feature has any predictive content. It is not a strategy metric. If your strategy is top-K, your evaluation metric should be about the top-K, not the whole cross-section.

This sounds obvious in writing. It is not obvious when you've spent two years computing IC because every published paper computes IC.

## What I use instead

For evaluating predictions, I look at the realized returns of a hypothetical top-K portfolio formed from the predictions. That's it. It's the metric that corresponds to what I actually do with the signal.

For evaluating regime — the gate — I look at how the actual portfolio is performing relative to a benchmark over a short window. Same logic: measure the thing you act on, not a proxy for it.

## A small recommendation

If you're building a quant pipeline and you compute IC, also compute the realized return of a top-K portfolio formed from your predictions, on the same data. Plot them against each other across periods. See whether they're correlated. For a top-K strategy, they often won't be, and that disconnect is worth knowing about before you build any more tooling on top of IC.

For me, the disconnect was so large that the metric I'd been using had been actively misleading me for months. I don't think I'm unique in this. The metric is too well-established for very many people to question whether it actually points at the thing they care about.
