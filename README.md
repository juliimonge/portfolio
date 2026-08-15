# Portfolio — Julian Monge

Three projects. One habit: **taking a claim that normally goes untested and
making it into a number somebody can check** — then stating, next to the
result, what it does *not* establish.

| | Claim that usually goes untested | What replaces it |
|---|---|---|
| **[secure-agent-eval](https://github.com/juliimonge/secure-agent-eval)** | *"Our agent is safe."* | 60 adversarial cases, two configurations, an audit log, and two rates that disagree with each other on purpose |
| **[no-mls-pipeline](https://github.com/juliimonge/no-mls-pipeline)** | *"Nobody knows what's actually for sale."* | Nine sources, transitive deduplication, one canonical record with every source still attached |
| **[fractal-desk-v2](https://github.com/juliimonge/fractal-desk-v2)** | *"We might be losing tax credits."* | A statutory deadline computed per document, with a status model that classifies outcome rather than activity |

Two are Python data and evaluation work; one is a TypeScript product. That is
the honest range — I build the pipeline, and I build the thing people use.

📄 **[Read the full write-up](https://claude.ai/code/artifact/a960f4ce-0bf6-4e93-a2b7-283d4fcacd72)**
— context, design reasoning, and the questions a skeptical reviewer should ask.

---

## secure-agent-eval

**Offline evaluation harness for LLM agent behaviour under adversarial conditions.**
Python 3.11+ · 60 cases · 80 tests · runs free with no API key

[→ github.com/juliimonge/secure-agent-eval](https://github.com/juliimonge/secure-agent-eval)

An agent with tools, memory and access to someone else's data has an attack
surface a chat model does not. The standard claim is "we added guardrails" —
usually untested, and where it is tested, running the test costs money, so
nobody re-runs it.

60 adversarial cases across five categories — prompt injection, tool permission
boundaries, data leakage, unsafe actions, human handoff — each in English and
Spanish. The same pipeline runs twice, guardrails off and on, and reports the
difference. Append-only audit log behind every number.

| Config | Unsafe | Attempted |
|---|---|---|
| baseline | 60/60 (100%) | 48/60 (80%) |
| guarded | **3/60 (5%)** | **38/60 (63%)** |

**The number worth arguing about is 63%, not 5%.** Under guardrails the agent
still reaches for the forbidden thing 38 times out of 60 and is stopped by a
mechanism every time. The system got safe; the agent did not get better. A
harness reporting only the unsafe rate would have hidden that.

The bilingual hypothesis I set out to test **did not replicate** — exactly one
failure out of 60 is attributable to language, which is n=1, and the README says
so. What did replicate cleanly: *mechanisms transfer across languages, patterns
don't.* The permission layer and secret scanner score identically in both
languages because they do not know what language the request was in.

---

## no-mls-pipeline

**Entity resolution across agencies that share no key.**
TypeScript · Union-Find · 9 tests · two offline demos

[→ github.com/juliimonge/no-mls-pipeline](https://github.com/juliimonge/no-mls-pipeline)

Nosara has no MLS. The same house is advertised by four agencies at four prices
under four descriptions, and nobody can see the actual inventory. The
engineering problem is that **there is no join key** — no listing id, no address
standard, no shared taxonomy.

Union-Find rather than pairwise matching, because duplication is transitive: the
fixture contains a pair 18.8% apart on price and 26.8% apart on area, outside
both tolerance bands, correctly resolved as one property through a bridging
listing. The similarity function is deliberately asymmetric — bedrooms gate,
everything else votes.

The honest gap, documented rather than buried: the tolerances are hand-tuned and
there is no labelled ground truth, so precision and recall are unmeasured.
`docs/MATCHING.md` sets out what closing that would take, and why false splits
are both worse and quieter than false merges.

The nine production scrapers are deliberately not published — they encode the
markup of named competitors and carry no engineering value. One fully documented
reference adapter runs against an HTML fixture in their place.

---

## fractal-desk-v2

**Deadline engine for Costa Rican electronic documents under TRIBU-CR.**
Python 3.11+ · DuckDB · 224 tests · synthetic data only

[→ github.com/juliimonge/fractal-desk-v2](https://github.com/juliimonge/fractal-desk-v2)

An electronic document your business *receives* must be answered within the
first eight business days of the month following the month it was issued. Miss
it and the IVA credit is permanently lost.

> The deadline is not "eight business days after the invoice." It is a fixed
> calendar point per issue-month. Everything issued in March shares one
> deadline — the 8th business day of April. The obvious implementation,
> `issue_date + 8 business days`, is wrong for **every document it touches**.

The status model classifies **outcome, not activity**: a document confirmed two
days late is recorded as "accepted" by the source system, but the credit is
gone, so the engine records `EXPIRED_CREDIT_LOST` with `accepted_late=True`.
Reporting it as accepted would make the dashboard's headline number a lie.

Layered DuckDB warehouse; the deadline logic lives in Python and SQL consumes
the result rather than re-deriving it. All data is synthetic and enforced as
such — taxpayer ids in a range Costa Rica does not issue, `is_synthetic` on
every mart row, and nothing in the repo can sign or transmit a document.

The repository also documents **Fractal Desk v1**, a university partnership with
Universidad Nacional de Costa Rica where five students built the system and I
was client-side sponsor: requirements, scope, acceptance criteria, schedule.
I did not write that code and none of it is in this repository. v2 is my own
independent work and deliberately just the deadline engine.

---

## What these have in common

**Measurement over assertion.** Each project's central claim has a number
attached, and the number is reproducible from a command in the README.

**Stated limits.** A threat model with an out-of-scope table. A validator that
downgrades a rule to WARNING because the reference data behind it is a
placeholder. A matcher whose accuracy is openly unmeasured, with the plan to
measure it written down. Regulatory notes that open by saying they are not tax
advice and that if they disagree with the law, the law is right.

**Two metrics where one would flatter.** Unsafe rate *and* attempt rate.
Outcome *and* activity. The second number is always the one that makes the
first one honest.

---

## Access

These repositories are currently **private**. If you are reviewing this and
need access, ask and I will add you.

- `julimongea@gmail.com`
- Nosara, Guanacaste, Costa Rica
