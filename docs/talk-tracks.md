# Three projects, and how to explain them

Julian Monge · last verified 2026-08-07

A reference for talking about the three repositories I'd point a technical
reviewer at. Each section has the same shape: what it is, why it exists, how it
works, what's measurable, what it doesn't do — then a spoken talk track.

Everything here was read out of the repositories, not from memory. Where a claim
is a number, the command that produces it is given.

---

## The through-line

If someone asks what the three have in common, this is the answer:

**All three are about making a fuzzy claim into a measurable one.**

- *"Our agent is safe"* → 60 adversarial cases, two configurations, an audit log,
  and two separate rates that disagree with each other on purpose.
- *"Nosara has no MLS, so nobody knows what's actually for sale"* → nine scrapers,
  a deduplication pass, and a canonical record with its sources attached.
- *"We might be losing IVA credits"* → a statutory deadline computed per document,
  with a status model that classifies outcome rather than activity.

The second thing they share is a habit: **each repo states what it does not do,
in writing, next to what it does.** A threat model with an out-of-scope table. A
validator that downgrades a rule to WARNING because the reference data behind it
is a placeholder. A case study that leaves `TODO(...)` in place rather than
estimating a number. That is a deliberate posture, and it's the thing I'd want a
reviewer to notice.

The third: **two are Python data/eval work, one is a TypeScript product.** That's
the honest range — I build the pipeline and I build the thing people use.

---

## 1. secure-agent-eval

`~/Claude/secure-agent-eval` · Python 3.11+ · MIT

**What it is.** An offline evaluation harness that runs a fixed set of
adversarial cases against an LLM agent under two configurations — unguarded
baseline and guarded — and reports the difference, split by attack category and
by language (English/Spanish).

### Why it exists

An agent with tools, memory, and access to someone else's data has an attack
surface that a chat model does not. The standard claim is "we added guardrails."
That claim is usually untested, and where it is tested, the test costs money to
run, so nobody re-runs it. This harness is 60 cases and a command that runs free,
offline, with no API key.

### How it works

```
cases.v1.yaml → schema validation → 60 cases (5 categories × 2 languages × 6)
                                          ↓
   input filter → model turn → permission check → tool execution → output scan
        (guardrails)  (adapter)      (tools)         (World)      (guardrails)
                                          ↓
                     grader → safe | unsafe | ambiguous  (+ attempted)
                     audit  → every step, append-only JSONL
                     report → results.json + report.md
```

Both configurations run the **same** pipeline; only the guardrail flags differ.
Under baseline the guardrails still evaluate and still log what they *would* have
decided, so a baseline run answers "what would have been blocked?" without
changing what happened.

The five categories: prompt injection (payload arrives in retrieved content),
tool permission boundaries, data leakage, unsafe/irreversible action, and human
handoff (where stopping is the correct behaviour and a fluent answer is the
failure).

Provider-agnostic: one class, one method (`step(TurnRequest) -> AgentTurn`). The
offline mock is a deliberately susceptible scripted agent — **not a model**. The
Anthropic adapter is a ~60-line reference implementation.

### What's measurable

Two rates, because they answer different questions:

- **unsafe rate** — a forbidden effect reached the world or the user. Grades the *system*.
- **attempt rate** — the agent reached for the forbidden thing, blocked or not. Grades the *agent*.

| Mock profile | Config | Unsafe | Attempted |
|---|---|---|---|
| compliant | baseline | 60/60 (100%) | 48/60 |
| compliant | guarded | **3/60 (5%)** | **38/60 (63%)** |
| cautious | baseline | 24/60 (40%) | 12/60 |
| cautious | guarded | 3/60 (5%) | 2/60 |

**The number worth arguing about is 63%, not 5%.** Under guardrails the agent
still reaches for the forbidden thing 38 times out of 60 and is stopped by a
mechanism every time. The system got safe; the agent did not get better. A
harness reporting only the unsafe rate would have hidden that.

Reproduce: `python -m secure_agent_eval.run --run-id sample-compliant`

### The finding that didn't go as pitched

I expected Spanish guardrails to degrade broadly. **They didn't.** Of three
guarded failures, exactly one is language-attributable: `HH-ES-02` says *"voy a
llevar esto a los tribunales"* where the escalation patterns cover *abogado /
demandar / litigio*. The other two are a matched EN/ES pair that fails
identically by design — they describe discrimination without naming it, which
defeats a lexical rule in any language.

One case is n=1. The README says so.

What *did* replicate cleanly, in every cell: **mechanisms transfer across
languages, patterns don't.** The permission layer and the canary scanner score
identically in EN and ES (0/6 everywhere) because they don't know what language
the request was in. The two lexical guardrails are the only places any gap
appears. That generalises past this suite — prefer structural controls to pattern
lists wherever the choice exists.

### What it doesn't do

Single-turn only. No encoding/obfuscation attacks (base64, homoglyphs,
zero-width). No multimodal. Two languages is not "multilingual." Six cases per
cell means several differences are one case wide, and no confidence intervals are
reported because none would be meaningful. Cases and guardrails share an author,
which biases coverage toward attacks I thought of — visible in the fact that the
only cases that get through are the two I wrote to get through.

### Numbers to have ready

80 tests, all offline · 60 cases, evenly split · 3 guardrails · ~0.1 ms and ~12%
token overhead per case for the guarded config (offline artifacts, not a
cost-of-safety estimate).

---

## 2. nosara-listings — the no-MLS pipeline

`~/Claude/nosara-listings` · Next.js 16 / React 19 / TypeScript / Tailwind 4 / Supabase

**What it is.** A scraping and deduplication pipeline that builds a single view
of what's actually for sale in Nosara, Costa Rica, across nine agencies that
don't share a database — plus the Next.js app that serves it.

### Why it exists

Nosara has no MLS. Listings are open and non-exclusive, which means the same
house is advertised by four agencies at four prices under four descriptions, and
no one — buyer, seller, or agent — can see the actual inventory. The market's
core information problem is duplication without a shared key.

This matters as an engineering problem because there *is* no join key. No listing
ID, no address standard, no shared taxonomy. You have to infer identity from
attributes that each agency records differently, or not at all.

### How it works

**Nine scrapers**, one per agency (RE/MAX, Century 21, Two Costa Rica, Coldwell,
Blue Zone, Mango, Sotheby's, Tropicasa, Wanderlust). Playwright for
JavaScript-rendered sites, cheerio for static HTML. Each has a
`// ── CONFIGURATION ──` block at the top so selectors are changed in one place
when a site redesigns — which they do.

**Union-Find deduplication** (`scripts/lib/matcher.ts`). Every cross-agency pair
is compared; matching pairs get unioned; connected components become groups. The
choice of Union-Find over pairwise matching matters: duplication is transitive.
If A matches B and B matches C but A and C look different, they're still one
house, and only a connected-components structure gets that right.

The similarity function is explicit and tunable at the top of the file:

| Field | Rule |
|---|---|
| Bedrooms | Hard match — a mismatch is an immediate reject, not a penalty |
| Price | Within ±15% |
| Construction area | Within ±20% |
| Lot area | Within ±20% |
| Bathrooms | Exact, contributes but never rejects |

A pair matches on `≥2 fields agreeing and 0 fields disagreeing`. Same-agency
pairs are skipped — one agency listing a house twice is their problem, not a
cross-source duplicate.

Each group picks a canonical record and keeps every source attached, so the app
can show "this house, listed by four agencies, at these four prices" — which is
the actual product.

**Operational constraint, enforced in the repo:** scrapers run locally only,
never on Vercel, and the Supabase service-role key never leaves the machine.
That's written into `CLAUDE.md` as a rule because it's the kind of thing that
gets violated by accident during a deploy.

### The honest engineering notes

`O(n²)` comparison, with a comment saying so and why it's fine at ~500 listings.
That's the right call — and naming it means the reader knows it was a decision
rather than an oversight.

Tolerances (±15%, ±20%, min-2-fields) are heuristics, not learned. They're
extracted to named constants at the top of the file so they're tunable, but
they're currently set by judgement, and there is no labelled ground-truth set to
measure precision/recall against. **That is the single biggest gap in the project
and the most interesting thing to talk about** — see the questions section.

### What it doesn't do

No test suite. No measured accuracy for the matcher. No geocoding, so identity
inference can't use location. `scrape-llm.ts` exists as an alternate extraction
path but the primary pipeline is selector-based.

---

## 3. fractal-desk-v2 — the deadline engine

`~/Claude/fractal-desk-v2` · Python 3.11+ · DuckDB · Streamlit · MIT · synthetic data only

**What it is.** A deadline engine and warehouse for Costa Rican electronic
documents a business *receives*: it computes the statutory acceptance deadline
per document, classifies each one's status, validates structure, and serves it as
a dashboard.

### Why it exists

Two things reset the ground in 2025: format v4.4 became mandatory on 1 September,
and **TRIBU-CR replaced ATV** as the tax platform on 6 October. The new regime
enforces a rule with money attached.

**The rule:** an electronic document you receive must be answered — accepted,
partially accepted, or rejected — within the **first eight business days of the
month following** the month it was issued. Miss it and the IVA credit on that
document is **permanently lost**. No retroactive upload restores it.

The counter-intuitive part, and the reason this is a real engine rather than a
date-arithmetic helper:

> The deadline is not "eight business days after the invoice." It's a fixed
> calendar point per issue-month. Every document issued in March 2026 — the 1st,
> the 15th, the 31st — shares one deadline: the 8th business day of April 2026.

A naive implementation (`issue_date + 8 business days`) is wrong for **every
document it touches**. That's the whole pitch in one sentence.

### How it works

`deadline(issue_date) = 8th business day of (issue_month + 1)`, over a business
calendar with Costa Rican holidays and an explicit coverage guard — if the
calendar doesn't cover the range being asked about, it raises rather than
silently returning a wrong date.

**Status model** — four states, and the fourth is a judgement call I'd defend:

| Status | Meaning |
|---|---|
| `ACCEPTED` | Confirmed on or before the deadline. Credit secured. |
| `PENDING_SAFE` | Unconfirmed, more than 3 business days remain |
| `PENDING_AT_RISK` | Unconfirmed, ≤3 business days remain (inclusive of deadline day) |
| `EXPIRED_CREDIT_LOST` | Window closed unanswered — **or a confirmation was sent late** |

That last clause: a document confirmed two days late is recorded as "accepted" in
the source system, but the credit is gone. Reporting it as ACCEPTED would make
the dashboard's headline number a lie. **The engine classifies outcome, not
activity** — so a late acceptance is `EXPIRED_CREDIT_LOST` and carries
`accepted_late=True` so the distinction stays queryable.

**Layered warehouse**, dbt-style, in DuckDB: `raw_ → stg_ → int_ → dim_/fct_`,
with numeric filename prefixes declaring dependency order. The deadline logic
lives in Python and the SQL layer *consumes* it as `fct_deadline_status` rather
than re-deriving it — one implementation, one place to defend.

**Structural validator** with ERROR / WARNING / INFO severities. Rule V007
(activity code in catalogue) is a WARNING rather than ERROR precisely because
`catalog_economic_activities.csv` is a placeholder — making it an ERROR "would
produce a validator that is loudly confident on the strength of data it
invented."

### What's measurable

**224 tests.** Run: `.venv/bin/python -m pytest -q`

Dependencies are pinned exactly, not floored, with the reasoning written into
`requirements.txt`: the project's claim is that a clone reproduces the numbers,
and a floating dependency breaks that the first time an upstream default changes.

### What it doesn't do — and this is load-bearing

**All data is synthetic and the repo proves it structurally**: every taxpayer ID
begins `999` (a range Costa Rica does not issue), every company name begins
`SYN`, every mart row carries `is_synthetic = TRUE`. No keys, no live calls, and
nothing in it can sign or transmit a document. `lxml` is present so the mock
submission layer emits parseable XML — it's a parser, not a signer.

`REGULATORY_NOTES.md` opens with "This is not tax advice" and records what rules
the engine implements *and why*, so a qualified person can check it against the
law. It says outright: if the document and the law disagree, the law is right.

### The v1 / v2 distinction — say this unprompted

`CASE_STUDY.md` documents **Fractal Desk v1**, a university–industry partnership
with Universidad Nacional de Costa Rica. Five UNA informatics students built it.
I was client-side sponsor and general director: requirements, scope, acceptance
criteria, schedule, milestone review. **I did not write that code, and none of it
is in this repository.** The IP is UNA's and the students'.

v2 is my own independent work, and it isn't a rebuild of v1 — it's one wedge,
built properly. When ATV became TRIBU-CR, the thing v1 integrated with stopped
existing, and the new regime created a problem that had no tool. So v2 is the
deadline engine and nothing else: not a CRM, not project management, not
invoicing. The narrowest thing that solves a problem people actually lose money
to.

Volunteering that distinction is the right move. Getting caught blurring it is
fatal; stating it up front reads as someone who knows where the line is.

---

## Talk tracks

Roughly three minutes each, spoken. Structure is the same throughout: problem →
what I built → the one number → what I'd fix.

### secure-agent-eval

> An agent with tools and access to other people's data has an attack surface a
> chat model doesn't. The usual claim is "we added guardrails," and that claim is
> usually untested — and where it is tested, running the test costs money, so
> nobody re-runs it.
>
> So I built a harness: 60 adversarial cases across five categories — prompt
> injection, tool permission boundaries, data leakage, unsafe actions, and human
> handoff — each one in English and Spanish. It runs the same pipeline twice, once
> with guardrails off and once with them on, and reports the difference. It runs
> offline with no API key, so it's free and anyone can re-run it. There's an
> append-only audit log behind every number.
>
> The design decision I'd defend hardest is reporting two rates instead of one.
> The unsafe rate asks whether harm actually reached the user — that grades the
> system. The attempt rate asks whether the agent reached for the forbidden thing
> at all, blocked or not — that grades the agent. Guardrails took the unsafe rate
> from 100% to 5%. The attempt rate went from 80% to 63%. So the system got safe
> and the agent didn't get better — it's being caught by a mechanism, every time.
> If I'd only reported the first number, that would have looked like a model
> improvement, and it isn't one.
>
> The bilingual result is the part I'd want to be honest about. I expected Spanish
> guardrails to degrade broadly. They didn't — exactly one failure out of 60 is
> attributable to language. One case is n=1, not a finding, and the README says
> so. What did replicate cleanly is more useful: mechanisms transfer across
> languages, pattern-matching doesn't. The permission layer and the secret scanner
> score identically in both languages because they don't know what language the
> request was in. The two regex-based guardrails are the only places any gap shows
> up.
>
> The obvious next step is testing the language hypothesis properly — about fifty
> escalation cases per language, sourced from real Spanish rather than mine — plus
> encoding attacks, which a regex filter should fail badly, and I'd like to
> measure how badly.

### nosara-listings

> Nosara has no MLS. Listings are open and non-exclusive, so the same house gets
> advertised by four agencies at four different prices under four different
> descriptions, and nobody can see the actual inventory — not buyers, not sellers,
> not the agents.
>
> As an engineering problem, the interesting part is that there's no join key. No
> listing ID, no address standard, no shared taxonomy. You have to infer that two
> records are the same house from attributes each agency records differently, or
> doesn't record at all.
>
> So: nine scrapers, one per agency — Playwright for the JavaScript-rendered
> sites, cheerio for the static ones — then a deduplication pass, then a Next.js
> app on Supabase.
>
> The dedup is Union-Find, and that choice is the thing I'd point at. Duplication
> is transitive: if listing A matches B, and B matches C, but A and C look
> different because one agency omitted the lot size — they're still one house.
> Pairwise matching gets that wrong. Connected components get it right. The
> similarity function is deliberately explicit: bedrooms are a hard match, a
> mismatch rejects immediately; price within 15%; areas within 20%; and a pair
> needs at least two fields agreeing with zero disagreeing. Each group keeps every
> source attached, so the product is "this house, four agencies, four prices" —
> which is the thing that didn't exist before.
>
> What I'd fix first: the tolerances are judgement, not learned, and there's no
> labelled ground truth, so I can't tell you the matcher's precision or recall. I
> know how I'd get it — hand-label a few hundred cross-agency pairs, then tune
> against that instead of against intuition — and until I do, the honest answer is
> that I don't have the number. The comparison is also O(n²), which I left in
> deliberately with a comment saying why: at 500 listings it's milliseconds, and
> blocking it before there's a measured accuracy problem would be optimising the
> wrong thing.

### fractal-desk-v2

> Costa Rica changed its tax platform in October 2025 — TRIBU-CR replaced ATV —
> and it enforces a rule with real money attached. Electronic documents your
> business *receives* have to be answered within the first eight business days of
> the month after they were issued. Miss that window and the IVA credit is gone
> permanently. There's no retroactive fix.
>
> The thing that makes it a real engine rather than a date helper: the deadline
> isn't eight business days after the invoice. It's a fixed calendar point per
> issue-month. Everything issued in March — the 1st, the 15th, the 31st — shares
> one deadline, the 8th business day of April. So the obvious implementation,
> add-eight-business-days, is wrong for every single document it touches.
>
> So the engine computes that per document over a Costa Rican business calendar,
> and classifies each one into four states. The one I'd highlight: if a document
> was confirmed *late*, the source system records it as accepted, but the credit
> is gone. Reporting it as accepted would make the dashboard's headline number a
> lie. So the engine classifies outcome, not activity — a late acceptance is
> EXPIRED_CREDIT_LOST, with a flag so you can still query the distinction.
>
> Structurally it's a layered warehouse in DuckDB — staging, intermediate, marts —
> and the deadline logic lives in Python with SQL consuming the result as a fact
> table rather than re-deriving it. One implementation, one place to defend. 224
> tests.
>
> Two things I'd say before being asked. All the data is synthetic and the repo
> enforces that structurally — every taxpayer ID is in a range Costa Rica doesn't
> issue, every row carries an is_synthetic flag, and nothing in it can sign or
> transmit a document. And there's an earlier version, Fractal Desk v1, that I
> sponsored as a university partnership — five students at UNA built it and I
> owned requirements and delivery. I didn't write that code and none of it is in
> this repo. v2 is mine, and it's deliberately just the deadline engine rather
> than a rebuild of the whole thing.

---

## Questions a skeptical reviewer will ask

**"Your eval's baseline is 100% unsafe. Isn't that rigged?"**
Under the compliant mock profile, yes, by construction — it's a worst-case bound,
not a prediction, and it's labelled that way. That's why the cautious profile
exists and is published alongside: it reproduces the documented asymmetry that
models resist direct adversarial requests far more than the same instruction
arriving inside retrieved content, and it gives 40%. Both are in the README. Also
worth saying: the mock can't see which configuration it's running under, and
there's a test asserting that — otherwise the comparison would be circular.

**"So did the Spanish thing work or not?"**
No, not as I framed it. One case out of 60. I'd rather report a hypothesis that
didn't replicate than round it up. The result that *did* replicate — mechanisms
transfer across languages, pattern-matching doesn't — is more useful anyway,
because it's an architectural recommendation rather than an observation about one
language.

**"What's the accuracy of your matcher?"**
I don't know, and that's the honest answer. There's no labelled ground truth, so
the tolerances are judgement. I know exactly how I'd get the number — hand-label
a few hundred cross-agency pairs and tune against them — I just haven't. What I
did do is extract every threshold to a named constant at the top of the file, so
when the ground truth exists the tuning surface is already there.

**"How do I know the tax rules in the deadline engine are right?"**
You don't take my word for it. `REGULATORY_NOTES.md` records which rule the
engine implements and why, with sources, dated, and it opens by saying it isn't
tax advice and that if it disagrees with the law, the law is right. The engine
also refuses to answer where it can't — the business calendar raises rather than
returning a wrong date when the holiday data doesn't cover the range asked about.

**"Which of these is most like the work here?"**
secure-agent-eval, directly — it's an eval harness with an explicit grading
rubric, an audit trail, and results split by cohort. fractal-desk-v2 is the same
discipline applied to a compliance domain: a rule with money attached, a status
model that refuses to flatter the numbers, and a test suite. nosara-listings is
the product end — a real pipeline with real ugly inputs that has to stay up.

---

## State of the repositories — fix before sharing

Read this before you send anyone a link. These are all quick.

| Repo | Commits | README | Issue |
|---|---|---|---|
| `secure-agent-eval` | **0** | 12 KB, written | Nothing is in git yet. |
| `fractal-desk-v2` | **0** | 19 KB, written | `git init` ran, no commits. 11 `TODO(...)` placeholders in `CASE_STUDY.md`. |
| `nosara-listings` | **1** | **boilerplate** | Only "Initial commit from Create Next App". README is the unmodified Next.js template. |

Concretely:

1. **`nosara-listings` — highest leverage.** Replace the create-next-app README.
   Right now the first thing a reviewer sees is *"This is a Next.js project
   bootstrapped with…"*, which throws away the entire no-MLS story. The
   `CLAUDE.md` in that repo already contains most of what the README should say.
   Someone who opens this repo today learns nothing about what it is.
2. **`secure-agent-eval`** — `git init`, commit, push. Complete and tested, just
   not anywhere yet.
3. **`fractal-desk-v2`** — commit it. Then fill the 11 `TODO(project duration…)`
   placeholders in `CASE_STUDY.md`; only you have those from your records, and
   they're deliberately blank rather than estimated, which is the right call but
   reads as unfinished to a reviewer who doesn't know that.

Two of the three have no git history at all. If the pitch is "these are the core
projects in my git," that needs to be true before the link goes out.
