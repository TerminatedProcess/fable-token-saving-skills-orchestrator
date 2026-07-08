# Keepalive Economics

Anthropic's prompt caching docs say the default cache lifetime is 5 minutes
and that the cache is refreshed when cached content is used. Anthropic pricing
docs say 5-minute cache writes are 1.25x base input and cache reads are 0.1x
base input.

Sources:

- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)

## Rule Of Thumb

If a lane is likely to return soon, a short keepalive can be cheaper than
letting a large Fable context go cold. If the wait is genuinely long, let the
cache die once.

## Where The Break-Even Is

Make the rule of thumb concrete. Let `C` be the cached context size in tokens.
Letting the cache die and rewriting it later costs one write at `1.25x * C`. A
keepalive instead pays one cache read at `0.1x * C` every tick for the whole
wait. With the default 250s tick under the 5-minute lifetime, that is roughly
`0.1x * C` every ~4 minutes.

So `N` ticks cost `0.1x * C * N` versus a single cold rewrite at `1.25x * C`.
They break even at `N = 12.5` ticks — about **50 minutes** of waiting. Under
that horizon the keepalive is cheaper; past it, a single cold rewrite wins, so
let the cache die once.

Two caveats this model ignores, both pushing the real break-even *earlier*:

- Each wake does "one small status check" — real output tokens per tick on top
  of the cache read.
- If the context keeps growing while you wait, later reads cost more than
  earlier ones.

Treat ~50 minutes as an upper bound, not a target. Measure your own `C` and
tick cadence before trusting the number.

Use keepalives for:

- short waits where no useful inline work remains
- CI or agent lanes expected back within the next few cycles
- sessions where cold context is large and expensive

Do not use keepalives for:

- overnight waits
- quota windows
- human-response waits
- tasks where the next useful event is far away

## Sleep-Tick Pattern

The reliable pattern is a self-extinguishing background sleep-tick:

```bash
scripts/waiting_phase_on.sh
scripts/keepalive_tick.sh
```

When the tick wakes the agent, it should:

1. do one small status check
2. process any completed lane work densely
3. re-arm the next tick only if still waiting
4. run `scripts/waiting_phase_off.sh` when the waiting phase is over

The Stop hook guards against forgetting to re-arm a tick while a waiting
sentinel exists.
