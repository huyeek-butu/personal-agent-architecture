---
title: Agent System Architecture and Runtime Mechanics (an on-machine field record)
date: 2026-10-17
scope: C — runtime substrate + self-governed rule layer + where the two mesh
status: public version (redacted) · one-off document, not a rule
---

# Agent System Architecture & Runtime Mechanics · Field Record

**English** ｜ [中文](AGENT-SYSTEM-ARCHITECTURE.md)

> **Three-level annotation (used throughout)**
> **✅ Known** = attributable to a file, a command output, or measured behaviour
> **○ Inferred** = reverse-engineered from externally observable behaviour; the implementation is not visible; may be wrong
> **✗ Invisible** = platform internals; no observation channel from here
>
> **What is deliberately excluded**: any plaintext credential, server address / port, or account-bearing path. Where a reference is needed, it is written in the form `~/.workbuddy/`.

---

## 1. One-sentence summary

> **This is a two-layer machine — "platform runtime + a self-built rule-governance layer" — and the two layers mesh at exactly one point: *context assembly*. What the platform injects each turn determines what I can comply with.**

Concretely: the substrate is given (a desktop agent runtime; I cannot see its source), and the upper layer grew over several months (8 rule layers, shadow review, an ingestion pipeline, cross-layer navigation). The entire efficacy of the upper layer parasitizes one action of the substrate: **which files get packed into context each turn.** Whatever is packed in *will* be followed (and consumes budget). Whatever is not packed in survives only on "recall at the right moment" (no budget cost, but it erodes silently).

That single fact determines every trade-off that follows.

---

## 2. Overall shape: two layers, one seam

```
┌─────────────────────────────────────────────────────────┐
│  L0–L7 rule-governance layer (self-built, mutable)       │
│  identity → iron rules → process → topics → skills →     │
│  shadow → project → tools                                │
└───────────────────────┬─────────────────────────────────┘
                        │  ★ the only seam: context assembly
                        │    (run by the platform each turn;
                        │     not programmable from here)
┌───────────────────────┴─────────────────────────────────┐
│  Runtime substrate (given, immutable)                    │
│  context assembly · tool calls · state · scheduling ·    │
│  fault tolerance                                         │
└─────────────────────────────────────────────────────────┘
```

**Why only one seam** ✅: there is no interface for me to plug into the runtime. I can do exactly two things — ① modify files on disk (affecting what gets injected next turn) and ② call tools (affecting this turn's observation and writes). There is no third point of leverage.

---

## 3. The substrate: five runtime concerns

### 3.1 Context assembly (the one concern that constrains everything)

**What I actually receive each turn is roughly these blocks** ○ (their ordering and weights are invisible; the *existence* of the blocks is known):

| Block | Contents | Source | Who controls it |
|---|---|---|---|
| System instructions | Operating rules, tool contracts, content boundaries | Platform | ✗ invisible, immutable |
| Identity | Persona / addressing / user profile | `SOUL.md` `IDENTITY.md` `USER.md` under `~/.workbuddy/` | ✅ I can edit |
| User-level persistent memory | Iron rules + pointer index (12 entries) | `~/.workbuddy/user-<account>-personal/MEMORY.md` | ✅ I can edit (**tier B is the injected source**, see §4 L1) |
| Project persistent memory | Project discipline / mechanisms / counters / to-dos | `.workbuddy/memory/MEMORY.md` in the workspace | ✅ I can edit; 3,951 / 4,000 characters |
| Skill list | Which skills are callable (name + one line) | scan of `skills/` | ✅ installing/removing a skill changes it |
| Tool list | Which tools exist (including deferred-tool roster) | platform + connector state | ✗ (except connecting/disconnecting connectors) |
| Runtime state | Current time, theme, connector reachability | platform | ✗ |
| Conversation history | Prior turns (**possibly already compressed into a summary**) | session jsonl | ○ indirect influence only |
| Current input | Your message | you | — |

**Two hard facts, both measured:**

- **✅ Accumulation hits a wall.** The persisted file for the current session is **359,056,370 bytes (~342 MB)**, and it has been compressed and reassembled once mid-way — the UI message where a conversation "continues from earlier context". Compression is not loss, but **the precision of earlier wording degrades**; only the summary survives.
- **✅ The injection budget has real weight.** The project `MEMORY.md` carries a self-imposed 4,000-character line (the platform's write allowance is also 4,000), and the user-level file the same. At the line you do not "stop writing" — you **must move content out into on-demand files**.

### 3.2 Tool invocation (the only place I can act)

| Mechanism | Description | Tag |
|---|---|---|
| Multiple tools per response | Several calls in one message; independent calls run in parallel | ✅ four probes fired in one message tonight |
| Deferred tools | Not every tool is loaded: search by name/meaning to fetch the contract, then execute | ✅ used in this session |
| Sub-agents | Spawn an isolated exploration session for open-ended search; returns conclusions, not process | ○ the tool surface exists; the degree of isolation is invisible |
| Sandbox | Commands run in a restricted environment by default; escaping it requires a separate request | ✅ stated in the tool contract |
| Headless channel | Start a **stateless, tool-less, no-local-files** session via the desktop CLI, purely as a "second brain" | ✅ this is how shadow pre-checks run |
| Connectors / MCP | External systems arrive as tools | ✅ reachability state exists; connectable and disconnectable |

**One easy-to-miss point**: tools do not only give me file I/O. They also determine **what I can observe**. Where there is no tool, my judgement rests on priors alone — that is the root of trade-off 4 in §7.

### 3.3 State management (four layers, not one)

| Layer | Carrier | What it holds | Tag |
|---|---|---|---|
| Session | `projects/<workspace>/<session-id>.jsonl` | Every message, tool call and result | ✅ largest single file 342 MB |
| Structured store | `workbuddy.db` (SQLite) | Session metadata, scheduled tasks, run records | ✅ provable from migration scripts (`automation_runs` / `failure_code` / `context_window` / `reasoning_effort` / `workspace_scope`) |
| File memory | `MEMORY.md` + `topics/` + `memory/YYYY-MM-DD.md` | Long-term discipline / knowledge / logs | ✅ written daily |
| Rollback & snapshots | `file-history/`, `*.file-rollback.ndjson`, trash | Pre-change versions; deletions recoverable | ✅ two mechanisms coexist |

**Why four layers instead of one** ○: their time constants differ completely — the session is second-scale, the structured store is task-scale, file memory is day-scale, rollback is operation-scale. Merging them would let them drag each other down. This is not a design choice; it is **forced by constraints**.

### 3.4 Scheduling (three kinds, not one)

| Kind | Trigger | Character | Tag |
|---|---|---|---|
| **Turn-driven** | You speak | The main path. One task = one or more turns | ✅ |
| **Scheduled tasks** | Time (cron-style repetition supported) | Unattended; the prompt must be self-contained; run records and failure codes are queryable | ✅ provable from DB tables and backup directories |
| **Headless self-invocation** | I initiate it | Only for review/evidence gathering; **changes nothing**; discarded after use | ✅ the shadow layer's second gate |

**What does not exist**: I have no resident process that stays awake to watch things. Every form of proactivity must land on a **scheduled task**, or wait for you to speak. This is the hardest physical boundary of this "self-governance" setup ✗○.

### 3.5 Fault tolerance (five gates, outermost to innermost)

| Gate | Mechanism | Measured parameters / behaviour | Tag |
|---|---|---|---|
| 1 Network / service | Circuit breaker + multi-endpoint health + fallback addresses | 60 s failure window, 3 consecutive failures to trip, 15 s open, backoff 30/60/120 s; three trusted sources polled; fallback address list enabled (addresses not recorded) | ✅ `failover.json` |
| 2 Model | Multiple channels + primary/backup switching | A separate channel ledger and health readings exist | ✅ |
| 3 Output | Empty-response detection | Headless channel measured: evidence packs above ~3.2 KB occasionally return **0 bytes** → now fixed as a "pack compression threshold" | ✅ reproduced twice |
| 4 Execution | Sandbox + assertions-first | Script failure **happens before the write**, so a failed assertion means zero contamination; the sandbox occasionally reports a "false failure" (non-zero exit but the action completed) | ✅ now a standing rule |
| 5 Files | Backups + trash | Snapshot before change → change; deletions go to trash, verified by scanning `$I` metadata | ✅ `_preview/backup-NNN/` |

**The critical ordering**: from outside in, the deeper you go, the more it is "mine to control". Gates 1 and 2 are entirely beyond my influence. Gates 4 and 5 are disciplines I wrote myself, and they are **the only two that guarantee "a mistake leaves no garbage"**.

---

## 4. The rule layer: 8 layers and what they really do

✅ All attributable (`~/.workbuddy/playbook/00-索引.md` §cross-layer navigation)

| Layer | What it holds | How it loads | Volume |
|---|---|---|---|
| **L0 Identity** | Persona / addressing / profile | **injected every turn** | 3 files, ~4 KB |
| **L1 Iron rules** | 20 iron rules + 12 pointers | **injected every turn** | ~3.9 K characters (cap 4,000) |
| **L2 Scenario playbooks** | Kickoff / execution / delivery / edits / summary / scheduling / dual-machine | **on demand** (I judge the scenario) | 8 files, ~57 KB |
| **L3 Knowledge topics** | Pitfalls / full preference text / ledgers / billing | **on demand** (driven by "when to read" in the L1 pointers) | 12 files, ~205 KB |
| **L4 Skills** | Single-task operating procedures | **auto-loaded on trigger words** (platform behaviour) | 79 skills, ~11 MB (self-reported by the index) |
| **L5 Shadow layer** | Verification + metabolism (error correction for the rules themselves) | **when you ask, or when I judge it "significant"** | 9 files, ~290 KB |
| **L6 Project memory** | Project discipline / mechanisms / counters / logs | **`MEMORY.md` injected every turn**; logs on demand | main file 3.9 K characters + daily logs |
| **L7 Tool layer** | Scripts + usage | **on demand** (no unified index) | 7 scripts, ~87 KB |

**The layers are ordered by "how often it is needed", not by "how important it is".** This is the foundation of the whole design:

- **The three injected layers (L0 / L1 / L6) carry decisions** — whether to act, how far to go, what must not be touched.
- **The on-demand layers carry process and knowledge** — the specific route, looked up only when used.
- **The auto-triggered L4 carries specialised procedures** — loaded only on a matching scenario, keeping the always-on layer uncontaminated.
- **L5 carries the meta level** — it does not review the task; it reviews **whether the rules themselves still hold**.

---

## 5. Coupling: four points, each owning one question

This is the core section of the document — this is where the two layers close.

| # | Coupling point | Implemented by | Owns | Failure signature |
|---|---|---|---|---|
| 1 | **Injection point** | Platform reads L0 / L1 / L6 every turn | **Whether to act** (decision) | Rule written but not injected → silently not executed, no error |
| 2 | **Routing point** | L2 routing table + skill trigger words | **How to proceed** (process) | Table points at the wrong place → detour or wrong process |
| 3 | **Fact point** | L7 scripts (lint / trash / page landing) | **Whether it is true** (verification) | No script → "it looks fine" is all I have |
| 4 | **Review point** | L5 three gates (self-check → cloud reviewer → cross-machine reviewer) | **Whether the rules themselves still hold** (meta) | Reviewer absent → same-family self-confirmation, independence discounted |

> **In one line: injection decides "whether", routing decides "how", scripts decide "whether it is true", the shadow layer decides "whether the rules themselves still hold".**

**These four points are asymmetrically strong** ✗○:

- 1 is strongest (platform-enforced; it cannot be skipped)
- 3 is next (executable, reproducible, and it errors out when it fails)
- 2 is medium (I may not consult it, or may consult and ignore it)
- 4 is weakest (depends on an external channel, and the external channel is frequently unavailable)

So what I have actually been doing these months is **migrating rules from point 2 to point 3** — anything expressible as a lint check should not remain prose. Only what cannot be expressed that way stays at 2.

---

## 6. The full data flow of one task (tonight's turn as the instance)

```
23:32  you ask a question
        └→ platform assembles context (incl. the L1 iron rule
           "align automatically on a new task")              【coupling point 1】
             ↓
        I judge: this is a new task, and not exempt           【L2 kickoff alignment】
             ↓
        turn 1: alignment only — restate the goal + 4
        specification items + 1 fork
        (no deliverable; this is what "at least two turns" means)
             ↓
23:35  you reply "C"
             ↓
        turn 2: read L2 routing → read L6 project memory →
        probe the actual state in parallel                  【tools: 4 in parallel】
             ↓
        write the document (new file; an L0-level action:
        reversible, no external effect, report on completion)
             ↓
        self-check: characters / line endings / bad bytes /
        sensitive values removed                            【coupling point 3】
             ↓
        record: daily log (append) → deliver (file card)
```

**There are three gates in this flow, each able to stop an error:**

1. **Alignment gate** (no deliverable in turn 1) — stops "finished in the wrong direction".
2. **Pre-write gate** (assertions first / new file instead of editing an old one) — stops "broke something".
3. **Self-check gate** (bad bytes / sensitive values / volume) — stops "finished but unusable".

**Note step 1**: my act of judging "this is a new task" is itself driven by an injected rule. **Without that iron rule in L1, I would have started writing immediately this turn** — which is exactly the cause of "alignment gets skipped" that you described. It is fixed now, but the fix is "add an injected rule", not "make the platform intercept".

---

## 7. Design trade-offs: six, each one an exchange

> Format: **what was chosen ↔ what was sacrificed.** None of them is "all upside".

### Trade-off 1 · Always-on vs on-demand — sacrifice coverage, buy guaranteed execution

A 4,000-character injection budget cannot hold a 700 KB rule corpus (the rough sum of the 8 layers). So only rules that are needed *to make a decision every turn* are allowed to stay always-on.

- **Bought**: always-on rules **are always executed** (independent of whether I remember).
- **Sacrificed**: the large majority of detailed rules **may or may not be executed** (dependent on my judging "I should look this up").
- **The counter-intuitive part**: when moving a rule out, if the sentence left behind says only "see also X" and the **action** is moved away, you have downgraded "executed every turn" into "executed if recalled" — **this is silent erosion and it reports nothing**. So when externalising, the action sentence must remain self-sufficient.

### Trade-off 2 · Rules vs hooks — sacrifice enforcement, buy portability

**"An instruction is a request; enforcement is an interception."** A rule telling an agent to stop will never be reliably obeyed, however strongly it is worded.

- **Bought**: rules are plain text — synchronisable across machines, reviewable, editable by me.
- **Sacrificed**: real enforcement. So **"stop and wait for approval" is a greeting, not a safety mechanism**; the actual backstop is "reversibility" (snapshots + trash).
- **Corollary**: if something matters enough that it must not fail, the correct move is not to word it more severely, but to **turn it into a script or a config** (coupling point 3).

### Trade-off 3 · One long session vs segmentation — sacrifice continuity, buy runnability

A single 342 MB session says this machine chose "keeps running" over "remembers everything".

- **Bought**: it can keep working without being interrupted by "let's start a new window".
- **Sacrificed**: the **precision** of earlier detail (only a summary survives compression), and the traceability of verbatim wording.
- **Therefore**: anything meant to last must **land in a file**; the session cannot be trusted to remember. This is the fundamental reason file memory exists.

### Trade-off 4 · Self-review vs external review — sacrifice speed, buy independence (only partly)

The shadow system has three gates: self-check → cloud reviewer (**a different model family**) → cross-machine reviewer (a second machine).

- **Bought**: the ability to catch "errors I cannot see in myself". It has paid off — it caught a structural misreading, and it caught me establishing a rule and violating it in the same edit.
- **Sacrificed**: a few minutes to a dozen-plus minutes on every large change.
- **What was *not* bought** ⚠: **the cloud reviewer is still the same platform.** It catches logical errors; it cannot catch "this does not match the machine's actual state". And the cross-machine reviewer has been absent for **15 rounds**, so the third gate is effectively severed.

### Trade-off 5 · Abstraction vs verbatim record — sacrifice searchability, buy fidelity

During ingestion: **the knowledge page holds an abstract distillation, the raw material is kept verbatim**; the two layers never contaminate each other.

- **Bought**: the ability to return to the original words later, without being misled by my own second-hand processing.
- **Sacrificed**: the same content stored twice; retrieval has to cross layers (which is why cross-layer navigation is needed).
- **Discipline**: the distillation layer must never modify the evidence layer.

### Trade-off 6 · Single home vs local convenience — sacrifice convenience, buy no drift

Anything has exactly one authoritative home; everything else may only be a pointer. Historically this required "merging two directories" and "repairing 537 broken links".

- **Bought**: no more "maintained in two places, therefore inconsistent".
- **Sacrificed**: every maintenance action must first confirm "where is the authority" — one step more than editing the nearest copy.
- **★ The only exception**: frozen copies (original text retained but no longer tracked) — only when four conditions hold simultaneously does it count as **not** a duplicate: **points at the authority · marked frozen · no active external pointer · has an unfreeze exit**. Miss one and it is a new drift point.

---

## 8. Hard constraints (the things that cannot be changed)

| Class | Constraint | Consequence |
|---|---|---|
| Platform | Kernel invisible and not programmable | I can only influence the system through "files + tools"; there is no third knob |
| Platform | Context has a ceiling | A 342 MB session gets compressed; detail becomes untraceable |
| Model | Roughly 150–200 instructions can be followed reliably | The always-on layer must be tiny, otherwise the whole file is ignored |
| Model | The middle of a long context carries the least weight | The most important rules go at the **start or the end** |
| Model | No tool, no observation | The headless reviewer can only read the evidence pack; it cannot verify "does this match the machine" |
| Human | Time budget constraint | Any **additive** suggestion is void; only substitution (trade X for Y) is viable |
| Machine | The two machines are not online simultaneously | Cross-machine review has a window, and it breaks easily |
| Structure | 4,000 characters per layer | At the line you must externalise, and externalising itself has a cost |

---

## 9. Weak points of this system (an honest list)

| # | Weak point | Current state | Severity |
|---|---|---|---|
| 1 | **The third gate is unpowered** | The cross-machine reviewer has been absent for rounds 9–23 (15 rounds); nobody can answer "does this match the machine" | High |
| 2 | **Single-session volume** | 342 MB; earlier detail is untraceable after compression | Medium |
| 3 | **Injection budget hugging its cap** | User-level L1 has ~91 characters of headroom | Medium |
| 4 | **L7 has no unified index** | 7 scripts, only 4 documented in the README | Medium |
| 5 | **Duplicated persistent mirror** | Tier B is the injected source, tier A is a legacy mirror; writing the wrong one reports nothing | Medium |
| 6 | **Candidate rules never promoted** | "The three reality gates" and "the four criteria for frozen copies" are both n=1, unverified | Low |
| 7 | **Triggering rests on self-judgement** | "Is this a new task?", "Is this a large change?" — I decide; a wrong call goes unblocked | Medium |

> **Point 7 deserves the most attention**: a large share of this system's reliability rests on *me judging correctly*. Rules reduce the misjudgement rate but **cannot eliminate it** — because the executor of the judgement is the one being judged. This is not a design defect; it is **the ceiling of a single-agent architecture**.

---

## Appendix · Full source-annotation table

| Topic | Tag | Basis |
|---|---|---|
| The **block composition** of context injection | ✅ | actually received every turn |
| The **ordering and weights** of injection | ○ | inferred from "what tends to get ignored" |
| Full session persistence | ✅ | 342 MB file |
| Compression happens | ✅ | it happened once in this session |
| The compression algorithm and retention policy | ✗ | no observation channel |
| SQLite table structure | ✅ | provable from migration script filenames |
| The scheduling kernel of scheduled tasks | ○ | only run records are visible |
| Circuit-breaker parameters | ✅ | plaintext in `failover.json` |
| The sandbox boundary implementation | ○ | known that it blocks; the rules are unknown |
| The degree of sub-agent isolation | ✗ | only the "conclusions returned" layer is visible |
| The existence and division of the 8 layers | ✅ | cross-layer navigation table + full file set |
| How often each layer is actually read | ○ | no logs; recall only |
| How skill trigger words are matched | ✗ | platform behaviour |
| Shadow-system rounds and conclusions | ✅ | `review_summary.md`, recorded round by round |
| The cross-machine reviewer's 15-round absence | ✅ | provable from missing numbers in the review summary |
