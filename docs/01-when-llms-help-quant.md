# When LLMs help quant

When I started this project, my prior was that an LLM was a general-purpose intelligence I could insert at almost any step in a trading pipeline and get a marginal improvement. That prior was wrong. The LLM helped in one narrow place, hurt or did nothing in several others, and the work was mostly in figuring out which was which.

Here's the typology I ended up with.

## Where it didn't help: picking instruments

The most natural-seeming application — show the LLM the top names from your model and ask it which to keep, drop, or boost — did not work. I tried several variants:

- Asking the LLM to filter the top scored names down to a smaller subset
- Showing the LLM all candidate stocks with their company names and sectors and letting it pick a portfolio from scratch
- Letting the LLM see the full ranked output and re-rank within the top tier

All of these were worse than just trusting the underlying model's ranking. The reason, as best I can tell: even when the LLM has the *right macro thesis*, it cannot map that thesis onto specific tickers reliably. It picks the famous names rather than the ones that will actually move. It picks the "textbook" beneficiaries of a policy shift rather than the actual beneficiaries, which are usually idiosyncratic and not findable from a name and a sector tag.

This was uncomfortable to confront because it implies LLMs do something different from what their interface invites you to think they do. They sound knowledgeable about which companies are good, but their knowledge is an aggregation of consensus narratives, not the kind of edge a stockpicker needs.

## Where it didn't help: feature engineering at runtime

I considered using an LLM to generate features at inference time — for example, summarizing news around a stock and turning that into a sentiment score. I built none of this in the production system, but I want to flag the reason: an LLM in the inner loop is slow, expensive, and non-deterministic. It defeats the entire premise of a quant strategy, which is that the rules are stable enough to be backtested.

The general principle: don't put an LLM anywhere that gets called once per stock per day. The combinatorics are bad even if the latency isn't.

## Where it helped: classification at low frequency

The thing the LLM is genuinely good at is taking a textual description of the world and turning it into one of a small number of named categories. *Given these macro headlines and policy signals, is the regime A, B, or C?* That's a question LLMs answer well, and it's the question I ended up using mine for.

The shape of the working application is:

- The LLM is called rarely (think weekly, not daily).
- It picks from a small fixed taxonomy I defined ahead of time, not from an open-ended set.
- Its decision modifies a *coarse* parameter of the strategy — a risk knob, an inclusion list, a tilt — not a per-stock score.
- The mapping from its decision to actual stocks is deterministic code I wrote, not the LLM.

That last point is the most important one. The LLM picks a label. The label maps to a basket of names through code I control. The LLM never has to know which specific tickers are in the basket. This bypasses the "LLMs can't pick names" problem entirely.

## The principle

LLMs are good where the answer is naturally a category and bad where the answer is naturally a ranking. Most quant tasks are rankings. A few are categories. Find the category-shaped problems in your pipeline and put the LLM there.

## What this looks like in practice

The application that worked for me was a regime classifier. The strategy mostly runs on its own. Periodically — at a cadence I tuned, see [the gating doc](02-gating-llm-calls.md) — the LLM is asked: *given the current macro environment, which named regime are we in?* It picks from a short list. The strategy adjusts its parameters according to which regime was named.

Crucially, the LLM does *not* tell me which stocks to hold. It tells me which **kind** of stocks to favor. The stocks are then chosen by code. This split is what makes the application robust.

## What I'd try next

The same pattern applies in places I haven't built yet:

- **Calibration check**: *given recent performance and recent news, is the model still calibrated?* — yes/no, weekly.
- **Catastrophe detection**: *given current headlines, is there an active black-swan event we should flatten exposure for?* — yes/no, daily-but-cheap.
- **Regime-conditional feature emphasis**: *in regime X, which features should be weighted more?* — chosen from a small fixed feature set.

In all of these, the LLM picks a category, the code does the rest. That's the only shape I've seen work.
