# Marketplace under message loss (`failures.message_drop`)

**Scenario run (Part A):** `marketplace` — 50 buyers and 50 sellers trading
over 10 rounds, Tier 1 state-machine agents, seed 42. I ran it once
(`nest run marketplace`) and inspected the trace before touching anything.

## The setting I changed and why (Part B)

I changed **exactly one** setting: `failures.message_drop`, from `0.0`
(baseline) to `0.15`. (`name` and `output.trace` are renamed only so the
run does not clobber the built-in trace; they are not experimental
variables.)

I chose message loss because it is the failure mode a decentralized agent
network actually lives with, and because the marketplace's baseline trace
showed the negotiation is a tight request/response exchange (a buyer sends
`buy`, a seller replies `sold` or `reject`). My hypothesis before running:

> Because a completed interaction depends on a message getting through,
> dropping a fraction *p* of messages should cost *more* than *p* of the
> interactions, and deal quality (`deal_rate`) should fall.

Half of that was right. Half was wrong in an instructive way.

## Before / after — concrete evidence

All numbers are from `nest report` on the run traces (seed 42, reproducible).
I swept the one setting across `0.05 / 0.15 / 0.30` to characterize it.

| `message_drop` | messages sent | delivery_rate | deal_rate | rejection_rate | unique_pairs |
|---:|---:|---:|---:|---:|---:|
| 0.00 (baseline) | 1000 | 1.000 | 0.532 | 0.468 | 467 |
| 0.05 | 593 | 0.941 | **0.572** | 0.378 | 286 |
| 0.15 | 300 | 0.850 | 0.524 | 0.283 | 157 |
| 0.30 | 174 | 0.718 | 0.439 | 0.187 | 104 |

`delivery_rate` tracks `1 - p` exactly, so the drop injection works as
documented. `success_rate` equaled `delivery_rate` at every level, i.e. it
measures *message delivery* success, not *business* success.

## What surprised me, and how I investigated it

Two things did not match the naive reading:

1. **A 5% drop cut total messages by ~41%** (1000 → 593), not ~5%.
2. **`deal_rate` barely moved and even *rose* at 5%** (0.532 → 0.572),
   the opposite of my hypothesis.

I investigated by parsing the raw JSONL trace (`kind`, `corr`, `msg`, `from`):

- **Message verbs fall together.** Baseline is 500 `buy` → 500 responses
  (1:1). Responses/`buy` drops with *p* (0.95 → 0.81 → 0.63), confirming a
  reply only exists when its `buy` was delivered — a one-hop cascade.
- **The real multiplier: agents freeze.** Buys-per-buyer at baseline is
  `{10 buys: all 50 buyers}` — every buyer acts in all 10 rounds. Under
  drops the distribution collapses: at `p=0.30`, **26 of 50 buyers send
  exactly one `buy` and then go silent for the rest of the run.** A buyer
  advances to the next round only after its current exchange resolves, so a
  single lost message strands that buyer for *all* remaining rounds. One
  drop costs far more than one message. That is why volume craters ~8× the
  drop rate.
- **Why `deal_rate` looks fine.** `deal_rate = sold / (sold + reject)` over
  interactions that *completed*. Dropped interactions leave the denominator
  entirely (rejections fall from 0.468 → 0.187), so the surviving ratio
  stays flat while the system does dramatically less business
  (`unique_pairs` 467 → 104).

**Takeaway / judgment:** in a decentralized agent system, ratio and quality
metrics (`deal_rate`, `success_rate`) can be blind to a liveness collapse.
A dashboard watching only `deal_rate` would call this market healthy at 5%
loss while it transacted 40% less. You have to watch absolute throughput
(`message_count`, `unique_pairs`) and, at the protocol level, a stateful
agent that blocks on a reply needs a timeout/retry to survive loss — which
is the design lever I would reach for next.

## How to reproduce

```bash
uv sync
# baseline
uv run nest run marketplace
uv run nest report ./traces/marketplace.jsonl -o baseline.html
# the one-setting change (message_drop 0.0 -> 0.15)
uv run nest run docs/hackathon/submissions/marketplace-message-drop/scenario.yaml
uv run nest report ./traces/marketplace_message_drop.jsonl -o experiment.html
```

Everything is deterministic under seed 42.

## Use of AI and other help

I used an AI assistant (Anthropic Claude) as a pair: it ran the sweep in a
sandbox, wrote the throwaway Python that parsed the trace verbs and the
buys-per-buyer distribution, and helped draft this README. The experiment
design (change `message_drop`, sweep it, look at *volume* not just
`deal_rate`), the hypothesis, and the interpretation are mine, and I re-ran
the baseline and the `0.15` scenario myself to confirm the numbers above.
Docs used: the NANDA Town quickstart and writing-a-scenario guide.

## Next service/agent I would build

A **retry/timeout wrapper** at the coordination layer: a buyer that has not
heard back within N ticks re-issues its `buy` to another seller instead of
blocking. I would then re-run this exact sweep to measure how much of the
throughput collapse a bounded retry budget recovers, and what it costs in
duplicate messages.
