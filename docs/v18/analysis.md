# OMH v18: Re-Grounding on Native Hermes Capabilities

> **Status:** Analysis / Proposal
> **Date:** 2026-07-05
> **Target platform:** Hermes Agent v0.18.0 ("The Judgment Release", v2026.7.1)
> **Authors:** Forge ⚒️ + Donald
> **Supersedes:** the v1.0 → v2.0 → v3.0 roadmap in [`ROADMAP.md`](../../ROADMAP.md)

---

## Thesis

**OMH built a bespoke orchestration substrate — durable state, subagent-result
persistence, named-role injection, an evidence runner — because Hermes v0.7
lacked those primitives. Hermes v0.18.0 shipped all four natively. OMH v18
re-grounds onto the native primitives and becomes *only* what it uniquely is:
the deliberation patterns.**

The while-loop, the state machine, the fan-out plumbing, the role-dispatch hook
— that machinery is now upstream. What remains scarce and uniquely OMH's is the
*discipline*: adversarial consensus planning, fan-out-synthesize-verify research,
Socratic requirements interviewing, multi-role triage, and the pitfalls authored
from real runs. That is OMH's position in the world. No one else holds it, because
no one else has this lineage. Everything else was scaffolding we owned because we
had to — and every line of bespoke scaffolding is a liability to maintain, migrate,
test, and be wrong about. v0.18.0 lets us delete most of it.

---

## 1. Versioning realignment: OMH major tracks the Hermes capability line

OMH's capability surface is a direct function of what Hermes provides. A skill or
plugin feature that assumes Kanban goal-mode, background delegation, or the
verification evidence ledger simply does not work on a Hermes that predates them.
The dependency is real and load-bearing — so we make it legible in the version
string itself.

**Rule: OMH's major version equals the Hermes feature-line number it targets.**

| Hermes release | Feature line | OMH major |
|---|---|---|
| v0.18.x ("Judgment Release") | 18 | **v18.x** |
| v0.19.x (future) | 19 | v19.x |
| ... | ... | ... |

A **v18.x OMH** declares, in one number, *"this works against a v18 Hermes and
uses its native capabilities."* An operator reading `omh @ v18.2` knows
immediately which platform floor it assumes. When Hermes advances its feature
line and ships primitives OMH should absorb, OMH cuts a new major that tracks it —
and the absorption work (this document's kind of work) is what that major
*is for*.

This supersedes the old `v1.0 (skills only) → v2.0 (plugin) → v3.0 (upstream PR)`
progression. That scheme tracked OMH's *internal* maturity; the new scheme tracks
the *platform capability floor*, which is the thing that actually determines
whether OMH runs. The plugin's internal maturity is now just the minor/patch.

The current codebase (plugin `0.1.0`, skills tagged `2.0.0`) is the pre-realignment
state. The re-grounding described here ships as **OMH v18.0.0**.

---

## 2. The core reframe

`docs/hermes-constraints.md` opens with its own honest framing:

> *"How OMH works around the limits of Hermes's subagent and lifecycle model."*

Every bespoke surface in the plugin is a documented workaround for a Hermes gap.
The Judgment Release closed the gaps. The reframe is mechanical:

| What OMH worked *around* (v0.7) | What Hermes now provides (v0.18.0) |
|---|---|
| No durable state → file sidecar + advisory locks | **Kanban** — SQLite state machine, reclaim-on-crash, block/unblock |
| "Final summary only" from subagents → persist-to-file contract | **`collect_task`** (full result) + **`kanban_complete(metadata=…)`** structured handoff |
| No named-role subagents → `[omh-role:NAME]` marker + injection hook | **Kanban named profiles** + `kanban_create(skills=[…])` per-task pinning |
| No verification hook → bespoke evidence runner | **Verification evidence ledger** + `pre_verify` hook + `/goal` completion contracts |
| No durable loop → one-task-per-invocation + state files | **Kanban goal-mode** (§4.7: *"reuses the same Ralph-style `/goal` engine"*) |

The compass we hold for any project sitting on a platform: *maximize what the
project uniquely is; minimize the bespoke surface around it; treat upstream as
something to build on, not work around or duplicate.* v0.18.0 is a major
absorption opportunity, and this document is the alignment study that scopes it.

---

## 3. The subsume audit

Four bespoke surfaces, run through the subsume filter. LOC counts are current.

### 3.1 State management — `omh_state.py` (531 LOC) + `.omh/state/` sidecar

**The largest single collapse candidate.** `omh_state.py` implements atomic
writes (`tmp → fsync → os.replace`), per-instance state files
(`{mode}--{slug}.json`), advisory locks with stale-PID reclaim, cancel signals
with TTL, staleness detection, and `state_list_active`. This is a hand-rolled
durable state layer with a coordination protocol — built because `delegate_task`
is not durable across turns (§2.9) and there was no other place to keep
cross-invocation state.

**Kanban is this primitive, shipped.** SQLite-backed board (`~/.hermes/kanban.db`),
the `triage → todo → ready → running → blocked → done → archived` lifecycle,
crash-reclaim via heartbeat timeout (`dispatch_stale_timeout_seconds`), `kanban_block`
/ `kanban_unblock` for the pause/resume the cancel-signal approximated, and a
durable audit trail in SQLite rows "forever" (§4.2) rather than sidecar JSON that
is lost on cleanup.

The mapping is near-exact:

| `omh_state` concept | Kanban equivalent |
|---|---|
| per-instance state file | a task (or board) |
| advisory lock + stale-PID reclaim | task claim + heartbeat reclaim |
| cancel signal + TTL | `kanban_block` / task cancel |
| `state_list_active` | `kanban_list` |
| atomic write envelope | durable SQLite row |
| `phase` / `iteration` fields | task state + `kanban_comment` thread |

**Verdict: drop-with-migration for all durable-pipeline state.** The one nuance
(see §5): purely *intra-session* tracking — e.g. deep-interview's coverage bins,
which live and die inside one conversation — does not need a durable board and
may stay lightweight (or use native `todo`).

### 3.2 Subagent result persistence — `omh_delegate.py` (449 LOC)

The prepare/finalize wrapper exists for one reason, stated in its own docstring
and in `hermes-constraints.md`:

> *"Final summary only — intermediate tool calls never enter the parent's context.
> (This is *why* `omh_delegate` writes via subagent-persists contract.)"*

Because a subagent's work product was unreachable except through its final string,
OMH injects a brutal-prose `<<<EXPECTED_OUTPUT_PATH>>>` contract forcing the
subagent to `write_file` the deliverable, then verifies the file exists and writes
dispatched/completed breadcrumbs. It is an elaborate workaround for "I cannot get
the full result back."

v0.18.0 removes the premise. **`collect_task`** (§2.4) blocks until a background
task completes and *retrieves the full result*. **`kanban_complete(metadata={…})`**
(§7.2) gives a structured handoff envelope — `changed_files`, `verification`,
`residual_risk` — that is exactly the structured result the file-persist contract
was reconstructing by hand.

**Verdict: likely drop.** For Kanban-shaped work, `kanban_complete` metadata is
the handoff. For in-turn fan-out, `collect_task` returns the full result. The
file-persist contract's reason to exist evaporates in both paths. (Retain only if
a specific pattern genuinely needs a durable on-disk artifact *as the deliverable*
— e.g. a research report that is itself the product — in which case the subagent
writing a file is domain output, not plumbing.)

### 3.3 Role injection — `tool_hooks.py` (58) + `omh_roles.py` (66) + `pre_llm_call` hook + 15× `role-*.md`

OMH's `[omh-role:NAME]` marker is parsed from a `delegate_task` goal and the
matching `role-*.md` prompt is injected into the subagent's system prompt via a
`pre_llm_call` hook. This is bespoke because Hermes v0.7 subagents had no concept
of a named, pre-configured role — you got an anonymous worker and passed
everything through `goal`/`context`.

Here the discriminator bites cleanly, and it is the heart of the reframe:

- **The 15 role prompts are uniquely OMH.** The executor/verifier/architect/critic/
  planner/researcher/synthesist/skeptic disposition set, tuned by real runs — that
  is OMH's authored value. **These stay.**
- **The injection *mechanism* is bespoke plumbing.** Kanban named profiles *are*
  roles; `kanban_create(skills=[…])` pins specialist skills onto a task without
  editing the assignee (§4.8); per-task model override routes cheap models to leaf
  roles natively (§2.8). The marker-and-hook apparatus is replaceable.

**Verdict: prompts stay; mechanism drops (or thins).** Under Kanban, a role
becomes a profile or a pinned-skill attribute. Where in-turn `delegate_task` is
retained (see fork §4), the role prompt travels in `context` — which the skills
already document as the no-plugin fallback. Either way the hook + marker parser +
`pre_tool_call` validator retire.

### 3.4 Evidence gathering — `evidence_tool.py` (204 LOC)

`omh_gather_evidence` runs allowlisted build/test/lint commands and returns
`{results, all_pass, summary}` — a pass/fail verification harness, built because
there was no native verification surface.

v0.18.0 ships the **verification evidence ledger** (§7.2, `agent.coding_context`),
the **`pre_verify` hook**, and **`/goal` completion contracts** (§7.1) that judge
against specified evidence rather than model feeling.

**Verdict: reshape, do not drop.** This is the one surface where a bespoke tool
may still earn its slot — but in a *changed shape*. The native ledger is
coding-check-scoped (pass/fail on canonical project checks). OMH's forthcoming
loop-engineering work (§6) needs a *metric-returning* variant — a command that
yields a **number** (build time, allocation count, test count), not a boolean.
That is not something the native pass/fail ledger provides. So `evidence_tool`
should be **absorbed for the coding pass/fail case** (defer to the native ledger)
and **reshaped into a metric primitive** for the optimization case. This makes the
re-grounding and the eventual capability extension share a seam rather than fight.

---

## 4. The architectural fork: how far does Kanban reach?

This is the one genuine decision the analysis surfaces. The rest is mechanical
absorption; this is a real trade-off.

The OMH skills are not monolithic. Some are **durable, multi-session** pipelines;
some are **in-turn, single-context** deliberations. The question is whether both
kinds re-ground onto Kanban, or only the durable ones.

### α — Kanban-native everything

Every pipeline becomes a board. Maximum bespoke deletion. **Risk:** it bends
in-turn deliberations onto a durable-queue model they don't want. An adversarial
debate (ralplan) wants all voices reasoning in relation to each other; a research
fan-out wants the orchestrator to hold the synthesis context. Marshaling those
through SQLite tasks adds latency and indirection to patterns that are *in-turn by
nature*.

### β — Kanban for durable, `delegate_task` for in-turn (recommended)

Split by the honest discriminator — **durability**:

| Pattern | Nature | Native mechanism |
|---|---|---|
| **ralph** | durable, multi-session execution | Kanban **goal-mode** (§4.7) |
| **autopilot** | durable phase pipeline | Kanban dependency graph + lifecycle |
| **triage** | durable backlog grooming | Kanban tasks |
| **deep-research** | in-turn fan-out → synthesize | `delegate_task` batch + **`collect_task`** (or Kanban **swarm** §4.5) |
| **ralplan** | in-turn adversarial debate | `delegate_task` fan-out (+ **MoA** §6 as an option for the debate itself) |
| **deep-interview** | in-turn, interactive, single-session | stays closest to current shape (Kanban ill-fit; it needs the user) |

β deletes almost as much bespoke as α (the whole 531-LOC state sidecar and the
449-LOC persist contract go either way), keeps each pattern in its natural
register, and matches the compass exactly: *adopt upstream where it fits the
invariant; keep the native split (durable vs in-turn) that Hermes itself draws
between Kanban and `delegate_task`.*

**Note the near-misses that make β compelling:** Kanban goal-mode is *named* as
the Ralph engine (§4.7). `hermes kanban swarm` (§4.5) creates "root orchestrator,
parallel workers, gated verifier, gated synthesizer, shared blackboard" — which is
**deep-research's exact topology** as a one-command primitive. So even under β,
deep-research has a native Kanban option (swarm) *and* an in-turn option
(delegate_task + collect_task); the choice there is a sub-decision within the fork.

**Lean: β.** It is the structurally honest split. But this is the decision to
ratify before any code moves.

---

## 5. What stays uniquely OMH

The subsume audit deletes plumbing. It must not touch identity. After the
re-grounding, OMH is:

1. **The deliberation patterns.** Adversarial consensus (ralplan), fan-out-
   synthesize-verify (deep-research), Socratic requirements (deep-interview),
   multi-role consensus triage — expressed now as native topologies rather than
   bespoke choreography, but the *patterns* are OMH's.
2. **The role-prompt catalog.** The 15 tuned dispositions. These are authored
   value, refined by real runs. They stay verbatim; only their injection mechanism
   changes.
3. **The pitfalls and driver playbooks.** `omh-ralph-driver`, `omh-ralplan-driver`,
   `omh-triage-driver` — the "how to actually run this without it going wrong"
   discipline. This is the hardest-won, most uniquely-OMH content, and it is
   *entirely* prose. Nothing about the re-grounding touches it except to update
   the mechanical references (e.g. "state lives in Kanban now, not `.omh/state/`").
4. **The composition.** `omh-deep-research → omh-deep-interview → omh-ralplan →
   omh-ralph` as a coherent pipeline for unfamiliar domains. The composition is a
   pattern, not plumbing.

The discriminator, held explicitly: **the mechanism is Kanban's; the discipline is
OMH's.** Deleting the mechanism sharpens the discipline — it stops being buried
under 1,200 lines of state/dispatch/injection code and stands as what it is.

---

## 6. What's next (not this document): loop engineering

Once OMH v18 stands cleanly on native primitives, the next arc **extends** OMH's
capability surface with the loop-engineering patterns that crystallized across the
field in mid-2026 — in particular the **metric-optimization loop** (the Karpathy /
Shopify autoresearch shape): specify a goal + a numeric metric, iterate
hypothesis → apply → measure → keep-or-revert, run until externally stopped.

That work is deliberately *out of scope here*. It is a capability **extension**,
not part of the re-grounding. But it is why §3.4 reshapes `evidence_tool` into a
metric primitive rather than dropping it outright: the optimization loop's
verifier returns a *number*, not a pass/fail, and native Hermes verification does
not (yet) provide that shape. The re-grounding leaves that seam ready.

The sequencing is deliberate and matches Donald's stated instinct:
**re-architect onto native capabilities first; extend to new loop patterns
second.** Autoresearch then lands as the clean worked-example of the new grain,
built against native primitives from the start rather than layered on bespoke
scaffolding we were about to delete.

---

## 7. Sequencing sketch (analysis-level)

Not a PR plan — a shape. Each step is independently scope-able against this map.

1. **Ratify the fork (§4).** α or β. Everything downstream depends on it.
2. **State → Kanban.** Migrate durable-pipeline state off `.omh/state/` onto
   Kanban. Retire `omh_state.py`'s durable surface. This is the largest deletion
   and the clearest win.
3. **Persistence → `collect_task` / `kanban_complete`.** Retire the
   `omh_delegate` persist-contract where the native result path covers it.
4. **Role mechanism → profiles / pinned skills.** Keep the 15 prompts; retire the
   marker + hook + validator.
5. **Evidence → native ledger (coding) + reshaped metric primitive (optimization
   seam).**
6. **Update the driver playbooks and skill references** to name native mechanisms.
   (This is the "doc-sweep is part of the fix" step — mechanical references across
   every skill must move in lockstep, not as follow-up.)
7. **Cut OMH v18.0.0.**
8. *(Next arc)* Loop-engineering extension — autoresearch as worked example.

---

## Appendix: bespoke surface inventory (current)

| File | LOC | Role | Post-v18 disposition |
|---|---|---|---|
| `omh_state.py` | 531 | durable state + locks + cancel | drop-with-migration → Kanban |
| `omh_delegate.py` | 449 | subagent persist contract | drop → `collect_task`/`kanban_complete` |
| `evidence_tool.py` | 204 | verification runner | reshape → native ledger + metric primitive |
| `omh_roles.py` | 66 | role catalog loader | thin/retire; prompts stay |
| `tool_hooks.py` | 58 | role-marker validator | retire |
| `hooks/llm_hooks.py` | — | `pre_llm_call` role injection | retire (mechanism) |
| `references/role-*.md` | 15 files | **role prompts** | **keep — uniquely OMH** |
| `skills/**` | — | **deliberation patterns + drivers** | **keep — uniquely OMH** (update mechanical refs) |

The line is clean: roughly **1,300 lines of bespoke plumbing** become deletion or
absorption candidates; the **role prompts and the deliberation discipline** — the
thing OMH uniquely is — stay and get sharper.

⚒️
