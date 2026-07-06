# OMH v18: Loop Engineering — Where We Go Next

> **Status:** Forward map / speculative design
> **Date:** 2026-07-05
> **Depends on:** [`analysis.md`](analysis.md) — the v18 re-grounding onto native
> Hermes primitives. **Nothing here is scoped or scheduled until the re-platforming
> lands and the α/β fork is ratified.**
> **Companion source material:** `docs/research/Loop Engineering — The Comprehensive
> 2026 Guide.md`, `docs/research/Hermes Agent Sub-Agent Orchestration … (v2026.7.1).md`

---

## Framing

`analysis.md` deletes bespoke plumbing and leaves OMH standing as *only* its
deliberation patterns, on native Hermes primitives. This document asks the next
question: **once OMH is a clean layer of deliberation discipline on top of Kanban +
`delegate_task` + goal-mode, what new capability does it uniquely add?**

The answer the field converged on in mid-2026 is **loop engineering** — but the
guide's central argument reframes what that means, and the reframe is load-bearing
for OMH:

> *"Defining the loop is the hard part. The while loop, the orchestration
> framework, the tool plumbing — that machinery is now largely solved and almost
> commoditized. What remains scarce and difficult is the design work that happens
> before the first iteration fires."*

That sentence is OMH's whole opportunity. The execution machinery is Hermes's now
(that's what `analysis.md` establishes). The **design work before the loop runs** —
specifying a verifiable goal, choosing a verification strategy that can't be gamed,
naming terminal states, drawing the blast radius, deciding where a human stays in
the loop — is exactly the class of adversarial-deliberation work OMH already does
for *plans*. OMH's next act is to do it for **loops**.

Three arcs, in dependency order:

1. **Metrics** — the measurement primitive that makes non-boolean loops possible.
2. **Autoresearch** — the metric-optimization loop itself (the worked example).
3. **Goal definition** — the deliberation layer that helps a user *specify* a
   loop worth running, before a single iteration fires.

The ordering is deliberate: (1) is the primitive (2) needs; (2) is the concrete
loop that proves the primitive; (3) is the layer that makes (1) and (2) safe and
usable at multi-day, multi-agent scale. And (3) is the one that is *most* uniquely
OMH — it is deliberation applied to loop design.

---

## Arc 1 — Metrics: the measurement primitive

### The gap

Every verification surface OMH and Hermes have — `omh_gather_evidence`, the native
verification evidence ledger (§7.2), `/goal` completion contracts (§7.1) — answers
one question: **did it pass?** Boolean. Converge-and-stop.

A metric-optimization loop asks a different question: **what is the number?** Build
time in seconds. Object allocations. Test count. p95 latency. Bundle size. The
verifier returns a *scalar*, and the loop's decision rule is `metric < baseline`,
not `tests == green`.

This is the single genuinely-new primitive. `analysis.md` §3.4 already anticipates
it: `evidence_tool` is reshaped rather than dropped precisely because the native
pass/fail ledger cannot return a number.

### The shape

`omh_measure` — a sibling to the native evidence ledger, not a replacement:

```
omh_measure(
    command="npm run build",           # the thing that produces a number
    extract="duration_seconds",        # how to pull the scalar from the output
    direction="minimize",              # minimize | maximize
) -> {value: 19.1, unit: "s", raw: "<captured output>", exit_code: 0}
```

Design questions this arc has to answer (each is a real fork, not a foregone
conclusion):

- **Extraction strategy.** Regex against stdout? A named line? Wall-clock timing of
  the command itself (the common case — "how long did this take")? A JSON field
  from a benchmark harness? Likely a small set of named `extract` modes rather than
  one general mechanism. The guide's discipline applies: *the smallest thing that
  works, then add modes when a real case needs one.*
- **Crash / regression semantics.** A hypothesis that crashes is not "infinitely
  slow" — it is *discard*. A hypothesis that is slower is *discard*. Only strictly
  better keeps. The primitive must distinguish {improved, regressed, crashed,
  unchanged} cleanly, because the loop's git keep-or-revert (Arc 2) keys off it.
- **Noise and honesty.** A single build-time measurement is noisy. Does `omh_measure`
  support N-run averaging? A stability check before accepting an improvement? This
  is the difference between "real 5% win" and "measurement noise the loop banked as
  progress." The guide's *cost-per-accepted-change* health metric depends on the
  measurement being trustworthy.

### Verification honesty (the guide's five-level ladder)

The guide's most useful framework is the **verification ladder** (L1 deterministic
→ L5 human), and its discipline is *honesty about which level you're actually at*.
`omh_measure` is a **Level 1 deterministic** verifier when the metric is a real
number from a real command — which is exactly why the metric loop is *safe to run
unattended*. It cannot be gamed the way an LLM-judge (L4) can, because the number
is the number. This is the property that makes autoresearch autonomous: **the
reward signal is deterministic.**

The one reward-hacking risk to design against (§ anti-pattern 3, "Specification
Gaming"): a hypothesis that makes the *metric* better by breaking the *thing*
(deletes the test that was slow; hardcodes the benchmark's expected output). The
mitigation is the guide's "red-before / green-after on a hold-out": pair
`omh_measure` with a **correctness gate the hypothesis never edits against** — the
existing test suite must still pass (native ledger, Level 1) *before* the metric
improvement counts. Metric-better + tests-broken = discard, not keep.

---

## Arc 2 — Autoresearch: the metric-optimization loop

### The pattern (Karpathy / Shopify shape)

```
define metric + measure baseline
loop (until externally stopped):
    generate hypothesis        (delegate_task → optimizer role)
    apply tentatively          (git stash / temp commit)
    measure                    (omh_measure)
    correctness gate           (native evidence ledger — tests still green?)
    better AND correct?        → commit, advance baseline, record
    else                       → revert, record the dead end
```

This is **not ralph.** Ralph is task-completion against a plan — it converges and
stops when tasks pass. Autoresearch is hypothesis-driven search over an improvement
space that **never converges on success** — "NEVER STOP LOOPING" is the literal
Shopify system prompt. It exits on budget, human stop, or an optional target
threshold. The two loops share almost no control flow; what they share is the
native substrate underneath.

### What it rides on (post-re-platforming)

Because `analysis.md` already re-grounded OMH, autoresearch is built on native grain
from day one — this is the payoff of sequencing the re-platforming first:

| Autoresearch need | Native mechanism (post-v18) |
|---|---|
| durable state across a multi-day run | **Kanban** — the run is a board/task; survives restarts |
| one iteration per invocation (context hygiene over 50+ iterations) | **Kanban goal-mode** re-invoke discipline, or cron-driven tick |
| hypothesis generation | **`delegate_task`** → optimizer role, **cheap model pinned** (§2.8) — the cost lever that makes a 50-iteration overnight run viable |
| numeric feedback | **`omh_measure`** (Arc 1) |
| correctness gate | native **verification evidence ledger** (§7.2) |
| human-in-the-loop pause/inspect | **`kanban_block` / `kanban_comment` / `kanban_unblock`** |
| audit trail of every hypothesis tried | durable **Kanban rows** + `kanban_comment` thread |

The genuinely-new code is small and well-bounded:

1. **`omh_measure`** (Arc 1) — the metric primitive.
2. **The keep-or-revert git mechanic** — tentative apply → measure → commit or
   restore. This is the one piece neither ralph nor any native primitive provides;
   it is the *blast-radius containment* that lets the loop "try crazy things"
   safely (the Shopify insight: an agent with no competing priorities will attempt
   what no human sprint plan would, and the revert makes that safe).
3. **The `omh-autoresearch` skill** — the loop specification: the disposition, the
   optimizer role prompt, the terminal-state definitions, the guardrails.

### The driver question (echoes analysis.md §4)

*What re-invokes the loop body?* Same fork-family as the re-platforming:

- **Cron tick, one iteration per fire** — philosophically truest to the "runs in
  the cracks of the day, no boredom, no deadline" character. Durable by
  construction; naturally rate-limited; "NEVER STOP" becomes "runs until you pause
  the cron job."
- **Kanban goal-mode card** — goal-mode "continues until the judge agrees or the
  turn budget runs out" (§4.7); for autoresearch the judge never agrees, so it runs
  to budget, which is *exactly right*, and you get HITL + reclaim for free.

**Design the loop body driver-agnostic** (the same discipline ralph already holds —
the body doesn't know what re-invokes it). Ship one driver; keep the other a config
swap. This is the "separate the body from the driver" split the guide names: *the
harness supplies the engine; loop engineering writes the pilot.*

### The blast radius (the guide's Part VIII)

An unattended optimization loop is a **faster loop scaling risk at machine speed.**
The guide's documented incidents (a $16k–$50k weekend from an unbounded loop) are
the cautionary floor. Autoresearch must ship with all four budget controls as
first-class, non-optional design elements:

- **iteration cap** (`max_turns`) — hard ceiling
- **budget ceiling** — token/dollar limit → escalate, don't continue
- **wall-clock limit** — for the multi-day case, a real "stop at Monday 9am"
- **stagnation detector** — N iterations with no accepted improvement → stop and
  report, don't burn budget circling a plateau

And the guide's named terminal states, every one explicit: *success* (target hit,
if one was set), *no-op* (nothing improved this tick), *blocked* (dependency it
can't resolve), *stalled* (plateau), *exhausted* (budget hit). Naming them is what
prevents the loop from calling "I got tired of iterating" a success.

This is where OMH's existing discipline transfers directly: ralph's 3-strike
circuit breaker is *already* a stagnation detector. The pattern is authored; it
re-points from "same error 3×" to "no accepted improvement in N."

---

## Arc 3 — Goal definition: deliberation applied to loop design

**This is the most uniquely-OMH arc, and the one the field is worst at.**

The guide's corpus analysis (50 real loops) found the maturity mismatch that names
OMH's opening:

> *"Mature: 70% verify in the autonomous zone, 74% name their terminal states, 66%
> set a verifiable goal. Immature: only 22% use an automated trigger, only 20% call
> named reusable skills, only 32% develop persistent memory. The practice has
> largely solved 'how do I know it's done.' It has barely started solving 'how does
> this run without me.'"*

And the deeper failure the guide keeps returning to: **most people specify the goal
wrong the first time.** A goal that can't be verified mechanically — "improve this
code," "make the tests better" — gives the loop no convergence criterion. The
Shopify piece proves it: *"asking the agent 'improve Polaris build time' didn't
work with the one-shot solution"* — the loop only succeeded once the goal was a
**targeted metric**, not a vague aspiration.

### The insight

**Turning a vague human intention into a loop-ready specification is a
deliberation problem — and deliberation is exactly what OMH does.** OMH already
has:

- **deep-interview** — Socratic requirements extraction, coverage tracking across
  dimensions, user-confirmed readiness gates.
- **ralplan** — adversarial consensus, where a Critic's job is to *break* the
  proposal before it ships.

Point those two capabilities at **loop design** and you get the thing the field is
missing: a structured conversation + adversarial check that produces a
*loop specification worth running* before any tokens burn. The guide even names the
deliverable: the `<name>-loop.md` document with a fixed skeleton (goal + verification
level, terminal states, guardrails, memory location, sub-loops with cost ceilings,
a "why it works" section). **OMH is positioned to be the thing that produces that
document well.**

### The shape: `omh-loop-design` (a deliberation skill, not a loop)

A pre-flight deliberation that runs *before* any loop, adapting deep-interview's
Socratic method and ralplan's adversarial check to the guide's four pre-flight
decisions:

1. **"Is this actually a loop?"** — the guide's first gate: *does the result of one
   turn change the next action?* If no, it's a scheduled one-shot; build that
   instead. An interview question, not a code path. Catches the most expensive
   mistake (running a loop where a prompt-chain would do) before it costs anything.

2. **"What does 'done' actually mean?"** — drive the goal from vague → verifiable.
   Classify it against the guide's goal types (verifiable / model-judged / mixed /
   pure-judgment). **Pure-judgment goals are not loopable** — surface that verdict
   honestly rather than letting the user launch a loop that can never stop. For
   metric loops specifically: *what is the exact number, measured by what command?*

3. **"What is the verification strategy?"** — place it honestly on the five-level
   ladder. The guide's discipline: *most practitioners want to claim Level 1 while
   relying on Level 4.* A Critic role's job here is to catch that self-deception —
   "you called this deterministic, but the real check is an LLM's opinion; harden
   it or admit the level."

4. **"Terminal states + budget?"** — force every terminal state to be named and
   every budget control to be set *before* the loop runs. Non-negotiable design
   elements, per the guide, not afterthoughts.

The **adversarial layer** is where OMH's ralplan lineage earns its slot: a
**loop-critic** role whose entire job is to attack the proposed loop the way the
guide's anti-patterns describe —

- *"Is this a while-true around a stranger?"* (no named skills, no real check)
- *"Is the maker also the checker?"* (reward-hacking risk)
- *"Can the agent game the spec?"* (delete the test, hardcode the output)
- *"Is Level 4 masquerading as Level 1?"*
- *"Is this an unattended runaway waiting to happen?"* (no stagnation detector, no
  budget ceiling)

If the loop specification survives the critic, it's stronger for it — the exact
value proposition ralplan already delivers for plans, re-pointed at loops.

### Output

A `<name>-loop.md` specification (the guide's format) that any of OMH's execution
paths — ralph, autoresearch, or a native goal-mode card — can then run. The
deliberation is decoupled from the execution: `omh-loop-design` produces the spec;
the loop skills consume it. Same separation OMH already draws between deep-interview
(produces spec) and ralph (consumes plan).

### Why this is the crown of the arc

Metrics (Arc 1) and autoresearch (Arc 2) give OMH the ability to *run* a
non-boolean, non-converging loop. But the guide's whole thesis is that **running
the loop was never the hard part** — designing a loop worth running is. Arc 3 is
OMH doing the hard part, using the deliberation muscle it already has. It is the
clearest expression of the compass: OMH's unique position is *adversarial,
iterative, multi-agent deliberation*, and loop-design is that discipline applied to
the highest-leverage question in the field — *what should the loop even be?*

---

## Multi-day, multi-agent scale (the through-line)

The user's framing named "multi-agent and multi-day loops" specifically. All three
arcs are built for that scale, and it is worth making the through-line explicit
because it is *why the re-platforming had to come first*:

- **Multi-day durability** is Kanban's, not bespoke. A loop that runs for three days
  across restarts, migrations, and human interruptions needs the durable board —
  the exact thing `analysis.md` re-grounds OMH onto. Autoresearch's overnight (or
  over-weekend) run *is* a long-lived Kanban task. This would have been
  near-impossible to hold reliably on the old `.omh/state/` sidecar; it is native
  now.
- **Multi-agent** is the orchestrator-subagent + swarm topology (§4.5), cheap-model
  routing for leaf roles (§2.8), and the maker/checker separation the guide names as
  the single most important loop-design decision. OMH's role catalog already encodes
  maker/checker separation (executor vs verifier vs critic); loops inherit it.
- **Human position on the autonomy spectrum** (in-the-loop / on-the-loop /
  out-of-the-loop) becomes a *configured* property of the Kanban board — HITL gates
  via `kanban_block` on irreversible steps, on-the-loop monitoring via the durable
  audit thread. Arc 3's goal-design deliberation is where that position gets
  *decided* honestly, before the loop runs.

The guide's closing line is the standard to hold: *"Build the loop. But build it
like someone who intends to stay the engineer."* Arc 3 is the mechanism that keeps
the human the engineer — comprehension and judgment stay upstream of the loop,
because OMH made the user design it deliberately before it ran.

---

## Dependency summary

```
analysis.md  (re-platform onto native Hermes v0.18 primitives)
      │
      ├── ratify α/β fork  ← gate: nothing below moves until this lands
      │
      ▼
Arc 1: omh_measure         (metric primitive — reshape evidence_tool)
      │
      ▼
Arc 2: omh-autoresearch    (metric-optimization loop; rides Kanban + delegate_task
      │                     + omh_measure; new: keep-or-revert git mechanic)
      │
      ▼
Arc 3: omh-loop-design     (deliberation applied to loop specification;
                            adapts deep-interview + ralplan; produces <name>-loop.md;
                            the most uniquely-OMH capability of the three)
```

None of it is scoped or scheduled here. This is the map for *after* the
re-platforming stands clean. The order is the dependency order; the value order is
arguably the reverse — Arc 3 is the highest-leverage, because designing the loop is
the hard part, and that is precisely what OMH is for.

⚒️
