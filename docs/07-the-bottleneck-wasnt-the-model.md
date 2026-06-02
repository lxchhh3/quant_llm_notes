# The bottleneck wasn't the model

After I ground the data and the universe (see [doc 06](06-ground-the-foundation.md)), I spent weeks trying to make the model better — different training objectives, rerankers, seeds, position sizing. Almost none of it moved the result, and it took me too long to understand why: the model was never the binding constraint. The constraint was everything that happens *between a good prediction and a filled position*, and I had been ignoring it because it was the boring part.

## Measure the idealized-to-realized gap first

The diagnostic that reframed the whole project was simple. I compared an *idealized* version of the strategy — form the target portfolio from the predictions, assume you get in at the reference price, hold for the intended period — against the *realized* version that actually trades. The idealized version captured a real edge over the benchmark. The realized version gave most of it back.

That gap is the answer to "what should I work on." If your idealized strategy already has the edge and your realized strategy doesn't, **more model work is wasted** — the signal is fine; you are losing it on the way to the book. The lever is participation, sizing, and execution, not the predictor.

In my case the realized losses came from three mundane places:

- On the days the winners actually moved, they were frequently untradeable — gapped away, or locked at the day's price limit — so the book couldn't get into the names that were carrying the edge.
- Being long-only at near-full exposure, it rode losers down with no mechanism to step aside.
- A minimum-holding rule that tested as a *feature* in backtest — it suppressed churn — behaved as *friction* live, locking positions I would rather have rotated out of.

None of these are model problems. All of them live in the layer I had spent the least time on.

## Match the training label to what the strategy realizes

The most expensive detour was a family of training objectives aimed at ordering the top of the cross-section "better." Pairwise and listwise rank-ordering losses, a two-stage reranker, even a reasoning model used as a reranker with hindsight. Every learned reordering lost badly to a plain, calibrated point predictor — by margins large enough to be unambiguous.

The reason, once I saw it, was obvious. A rank-ordering loss optimizes a *single-period* ordering. My strategy holds each pick for *multiple* periods. The objective and the realized P&L are computed over different horizons, and on this data they were anti-aligned — the better the model got at single-period ordering, the worse the held portfolio did. There is even a clean tell: the in-set ordering metric and the portfolio return moved inversely.

The fix that actually helped was not a cleverer loss. It was training the predictor on a label whose *horizon matches the holding period*. Match the target to what the strategy realizes, not to what is convenient to compute or what the literature defaults to. A loss or a metric computed at the wrong horizon is not a weak signal — it is a differently-aligned one, and optimizing it can actively hurt. (This is the same disease as [doc 03](03-ic-is-a-lying-compass.md): a metric that measures the wrong thing, confidently.)

## A fix that rescues a weak model can sink a strong one

One more trap worth naming. Early on, with a weaker model, an ablation had shown that variance-reduction sizing — down-weighting the more volatile names — beat naive equal weighting. I had filed that away as a settled win. Re-run on the stronger, grounded model, it *lost*, clearly. On a weak model, shrinking volatile positions removes noise you are better off without. On a strong model, those same positions are where the upside lives, and shrinking them just hands the return back.

The lesson: **an improvement is only an improvement relative to the model you tested it on.** When the model underneath changes, every prior result built on the old one is suspect. Re-test stale wins on the current stack before you trust them — a fix can invert its sign when the thing it was fixing gets better.

## The meta-lesson

I spent months on the picker — the legible, satisfying, intellectually fun part — while the actual binding constraint sat one layer down in execution and exposure, unexamined because it was tedious. The boring layer was the one that decided the outcome.

If I were starting over, the first question would not be "how good is my signal?" It would be "if my signal were perfect, how much of it would survive contact with the market?" The distance between those two numbers is your real to-do list.
