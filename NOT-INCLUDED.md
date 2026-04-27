# What you won't find here, and why

This repo documents the design and lessons from an LLM-augmented stat-arb system. It deliberately omits the parts that would let someone reproduce the production configuration.

## Specifically not included

- The market and universe
- The deep-learning model architecture, hyperparameters, training schedule, and seed
- The gating rule's exact shape (lookback length, threshold, comparison set)
- The taxonomy that the LLM picks from, including its size and how it was constructed
- The prompts used to drive the LLM (system prompt, user prompt, response schema)
- Walk-forward configuration: train length, valid length, test length, step size, window count
- Per-period performance numbers
- The macro context corpus passed to the LLM
- Cost and turnover assumptions
- Production code

## Why

Two reasons.

First, the system's behavior depends on the values, not just the architecture. Several of them were chosen empirically against this dataset, and they encode information from the data that I'd rather not redistribute. The LLM prompts in particular were iterated against observed model failure modes, so they carry quite a bit of latent knowledge about what works on this universe.

Second, those values mostly do not transfer. Copying the gate threshold or the taxonomy onto a different universe would likely produce worse results than the reader designing their own from scratch using the patterns in [`docs/`](docs/). So publishing them would mostly help people misapply them, while still removing whatever edge they represent here. The patterns are the thing worth sharing. The numbers aren't.

## What is shared

Architecture, intent, the failure modes I encountered, and a structured account of which design choices mattered. Anyone with the relevant background should be able to design their own version of each component.

If you have questions that the docs don't answer and that don't ask for the values above, GitHub DMs are open.
