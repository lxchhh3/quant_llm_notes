# Ground the foundation before you trust a number

When I first published these notes, I led with a backtest number. I was proud of it. Then I put the system into live paper trading, and in the course of understanding why live diverged from backtest, I found something more unsettling than any modeling mistake: the *foundation* under that number — the price data and the definition of the universe — was wrong, in ways that moved the result by more than any model choice I had ever made.

This doc is about the layer beneath the model. It is the least glamorous part of a quant pipeline and, it turns out, the part most able to silently invalidate everything above it.

## The data was misaligned, and the model hid it

My per-instrument price series had drifted out of alignment with the master trading calendar. The cause was mundane: an incremental update process appended each instrument's new rows without reinserting placeholders for the days that instrument didn't trade — halts, suspensions, anything that took it out of the market for a session. Every missed day shifted that instrument's entire history by one position relative to the calendar. The drift was non-uniform across instruments, so no single global correction could undo it.

Here is the insidious part: a cross-sectional model is *shift-immune*. It learns relationships within each day's cross-section, so a per-instrument time shift mostly washes out — the model looked completely unaffected, its training metrics fine. But anything that anchors to a specific date — a performance gate, a risk check, any "what did this stock do on this day" computation — was reading garbage.

The lesson generalizes: **a bug your model is immune to is a bug you will not find by looking at your model.** You find it by looking at the data directly, on days where you already know what the answer should be. The check that finally caught it: on the largest index-move days, does the cross-sectional median of your per-instrument returns track the index? If the index is down hard and your cross-section looks flat, your data is misaligned — even though every metric on your model dashboard looks healthy.

## The universe was survivorship-biased

The second problem was worse, because it wasn't a bug — it was a definition I had never thought to question. The "universe" I backtested on was a membership list: which instruments belong to this segment of the market. I had been using *today's* list. Across a multi-year backtest, that quietly deletes everything that was demoted, delisted, merged, or relegated out of the segment over the test period. The losers disappear. The survivors remain. Every historical day gets reconstructed as if you had somehow known, in advance, which names would still be standing years later.

The fix is point-in-time membership: who was *actually* in the universe on each date, reconstructed from a historical membership feed rather than from today's snapshot. It is more work, the data is harder to source, and it makes your numbers worse — which is exactly the point.

When I re-ran on a point-in-time universe, the result moved by more than the entire edge I thought I had. Not a refinement — a different conclusion. The most recent window, which had looked like a *weakness* on the survivorship universe, turned into the model's *strongest* on the honest one. Both the magnitude and the shape of the result had been artifacts of the universe definition.

**Treat the universe file as data, not configuration.** It encodes as much hindsight as any feature, and a wrong one can flip your conclusions while every other number on your dashboard stays green.

## Deployed is not validated

A sibling failure, same family. The model running live had not been trained the same way as the model I had validated. The validated recipe trained on a rolling recent window; the deployed artifact had been produced by a different, one-shot procedure. The two shared a file format and an interface, so nothing broke — but the thing taking live decisions was not the thing that had passed the test.

This is embarrassingly easy to do and genuinely hard to notice, because both objects are "the model," and both load and produce predictions. Guard against it explicitly: pin the training recipe, checksum the artifact, and verify that the file in production is the exact one that came out of the validation run. "It loads and produces predictions" is not evidence that it is the right model.

## Why this matters more than the modeling

Walk-forward, cross-validation, held-out test sets — all the out-of-sample discipline in the literature operates *on top of* the data and the universe you hand it. None of it can see a misaligned series or a survivorship-biased membership list. If the foundation is wrong, your careful methodology is carefully measuring an artifact. You get a clean, rigorous, reproducible, wrong answer — which is the most dangerous kind, because every check you know how to run will pass.

The order of operations I would recommend, learned the hard way:

1. Reconcile every per-instrument series to a master calendar, with explicit gaps for non-trading days. Verify it on known big-move days.
2. Reconstruct point-in-time universe membership before you trust any cross-period aggregate.
3. Pin and checksum the deployed artifact against the validated one.
4. *Then* — and only then — argue about the model.

I had this order backwards for a long time. I tuned the model for months on a foundation I had never audited. The single highest-leverage thing I did in the whole project was not a modeling improvement; it was discovering that the ground I had built on wasn't level.
