# Gating LLM calls

The LLM-as-regime-classifier idea (see [doc 01](01-when-llms-help-quant.md)) sounds simple: every period, ask the LLM what regime we're in, and adjust accordingly. My first implementation did exactly that. It was worse than not having the LLM at all.

The problem isn't that the LLM gives wrong answers. The problem is that *most of the time, the underlying model doesn't need help*. Every time the LLM speaks, there's a chance it changes a decision that didn't need changing — and the cost of those wrong interventions on already-good periods overwhelmed the benefit on the bad periods.

The fix was to gate the LLM. Only consult it when the underlying model is visibly failing.

## Defining "visibly failing"

You need a measurable failure signal. Mine compares what the model is *actually holding* to what a passive benchmark would have done over a short rolling window. If the model's holdings have underperformed the benchmark by enough, the gate opens.

A few things matter about how this is constructed:

- It compares **holdings**, not signal quality. A model can have a fine signal (high IC) and still be holding the wrong instruments — see [doc 03](03-ic-is-a-lying-compass.md). Holdings are what you actually have on the books, so they're what the gate should watch.
- The window is **short**. Long enough to filter single-day noise, short enough to react before the damage compounds.
- The default when there isn't enough history is **closed** (model assumed fine), not open. This was a bug in my first version: defaulting to "open" meant the gate fired every time it had no data, which was constantly. The right default is "if I don't know, assume the model is fine."

## Throttling

Even when the gate opens, you want to throttle the LLM call. If the gate fires on Monday and the LLM says "rotation," you don't need to ask again on Tuesday — you already have a recent answer. Cache the LLM's most recent diagnosis and reuse it for some period before asking again.

This matters less for cost than for stability. Asking every day means the LLM might flip-flop between diagnoses, generating churn in the portfolio. A weekly cadence smooths this out.

## Why this matters more than the LLM itself

Without the gate, my LLM layer was net-negative across the test set. With the gate, it was strongly net-positive. The same prompts, the same model, the same taxonomy — the difference was *when* the LLM spoke.

This is the lesson I'd most want someone to take from this writeup. **The LLM-in-the-loop pattern requires a gate. The gate is not optional. The LLM is the loud headline-grabbing component, but the gate is doing most of the actual work.**

If you skip the gate, the LLM will speak on periods where it shouldn't have, and its mistakes there will likely overwhelm its help where it should have spoken. You will conclude the LLM doesn't help, and you'll be right — for that implementation. The gate is what makes the LLM worth having.

## The general pattern

Distilled:

1. Define a measurable signal of when your underlying model has lost calibration. (Holdings vs. benchmark over a short window is one option among many.)
2. Default to "calibrated" when the signal can't be computed. Never default to "broken."
3. Open the gate when the signal exceeds a threshold you tune.
4. Throttle the LLM call so a persistent failure doesn't turn into daily LLM consultations.
5. When the gate is closed, your LLM should not be running. Verify this with logging.

## Failure modes I hit

A few things broke during development:

- **Default to broken**: the gate fired on every period where it had no history. Fixed by flipping the default. This single bug masked the value of the gate for several iterations.
- **Window too long**: at one point my window was long enough that it would only catch failures after they had already done most of their damage. Shortening it was the fix; the right length depends on your strategy's holding period.
- **No throttle**: in early versions I called the LLM every day the gate was open, and the LLM's diagnosis flickered. A weekly throttle stabilized the portfolio.

None of these are interesting algorithmic ideas. They're all "details you have to get right or the system doesn't work." That's most of what building the gate was.

## What this isn't

This isn't a circuit breaker for risk management. A risk circuit breaker stops trading when losses exceed a threshold. The gate is upstream of that — it's about deciding when to consult a slow advisor, not about cutting exposure when something has gone wrong. You probably want both.

## Postscript: what happened when it went live

I shipped this design. For the entire first month of live paper trading, the gate never opened once — and the portfolio bled the whole time, in a market that hadn't crashed.

The gate wasn't disagreeing with me. It was broken. The live path made a timing assumption the backtest never had to: it tried to read a return that wasn't materialized yet at the time of day the live job ran. That raised an error; the error was swallowed inside a per-day loop; the failure-detection fell through to "not enough history"; and the function returned "model is fine" — silently, every single session. The one component the whole design rests on was effectively absent, and nothing in the logs said so.

Three lessons fell out of this, and they're worth more than the original design:

- **A gate that fails silently fails to "everything is fine."** That's the most dangerous possible default — the failure is invisible *and* it disables your protection exactly when you can't see it. If a check can't be computed, it should fail loud, not fall through to the benign answer.
- **Log the verdict every time, including the no-op.** A line that says "gate closed, model healthy" on every quiet day is what tells you the gate is alive. Its *absence* is the only thing that would have caught this, and I wasn't emitting it. Silence read as health.
- **Test the live path, not just the backtest path.** The bug lived entirely in a timing assumption that the backtest — which has all the data available at once — could never exercise. The backtest and the live loop are different programs; the gate has to be tested in the one that actually runs.

The irony isn't lost on me. This doc argues the gate matters more than the LLM. Live trading proved it by removing the gate involuntarily and letting me watch what the system does without it. It bleeds. The gate was doing the work, exactly as claimed — I just got to confirm it in the most expensive way.
