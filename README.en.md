# Agent System Architecture & Runtime Mechanics · Field Notes

> Field notes on treating an AI agent as a **maintainable system**. The runtime is given (a desktop agent runtime); the upper layer is self-built (an 8-layer rule-governance stack). The two halves meet at exactly one seam — and every design trade-off in here grows out of that seam.

This is not a theory paper, and it is not product documentation. It records **a machine that actually runs**: after several hundred conversation turns, dozens of tasks, and more than twenty rounds of self-review, here is what it has grown into, where it is broken, and what was traded away for what.

---

## Three conclusions that are not obvious

If you only read three sentences, read these three.

1. **Where a rule lives matters more than what it says.** Rules injected into context every turn *will* be followed — and they eat the context budget. Rules loaded on demand only get checked when the agent happens to think of it, and **they fail silently**. So moving a rule out of the always-on layer while leaving only a "see also X" pointer downgrades an enforced rule into an optional habit. That is silent erosion, and nothing reports an error.

2. **The four coupling points are asymmetrically strong.** Injection (platform-enforced) > script verification (reproducible) > routing table (the agent may not consult it) > external review (the channel is often absent). The real mainline action is therefore a single move: **migrate rules from routing toward scripts** — anything expressible as a lint check should not remain prose.

3. **Single-agent architectures have a ceiling.** A substantial share of the system's reliability rests on *the model judging correctly* — is this a new task, is this a large change. Rules reduce the error rate; they do not remove it, because the judge is the same agent being judged.

---

## Document structure

| Section | What it covers |
|---|---|
| 1. One-sentence summary | Two layers, one seam |
| 2. Overall shape | Why there is only one point of leverage |
| 3. Five runtime concerns | Context assembly / tool invocation / state management / scheduling / fault tolerance |
| 4. The 8-layer rule stack | What each layer holds, how it loads, how large it is |
| 5. Coupling: four points | Injection decides *whether*, routing decides *how*, scripts decide *whether it is true*, the shadow layer decides *whether the rules themselves still hold* |
| 6. Data flow of one task | A real turn walked through step by step |
| 7. Six design trade-offs | Each written as "what was sacrificed for what" |
| 8. Hard constraints | The things that cannot be changed |
| 9. Weak points | An honest failure list |
| Appendix | Full source-annotation table |

---

## Source annotation

Every factual claim in the document carries a provenance level, so that inference and observation never blur together:

| Tag | Meaning |
|---|---|
| **✅ Known** | Directly attributable to a file, command output, or measured behaviour |
| **○ Inferred** | Reverse-engineered from externally observable behaviour; the implementation is not visible and may differ |
| **✗ Invisible** | Inside the platform; no observation channel exists |

Read the tags first. The most valuable part of this document is arguably where it says `✗` — **an explicit admission of what cannot be seen.**

---

## What this is not

- **Not official documentation.** It is unrelated to the platform vendor; it is a user-side field record.
- **Not best practice.** All six trade-offs are "under these specific constraints, chose A and sacrificed B". Change the constraints — team size, cost budget, compliance requirements — and the conclusions change.
- **Not a copyable configuration.** The contents of the 8 layers are tightly bound to one situation. What transfers is the **structure and the method**, not the individual entries.

---

## Who this is for

If you are doing any of the following, this may be worth your time:

- Collaborating with the same agent over months, watching rules pile up until they start contradicting each other
- Wanting an agent's behaviour to be **stable and predictable** rather than a coin flip
- Needing the agent to **review itself**, while knowing self-review is unreliable
- Caring about meta-questions such as how a rule system avoids rotting

---

## License

MIT — see [LICENSE](LICENSE).

---

**中文** → [README.md](README.md) · **Full text** → [AGENT-SYSTEM-ARCHITECTURE.en.md](AGENT-SYSTEM-ARCHITECTURE.en.md)
