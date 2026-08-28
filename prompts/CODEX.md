# LANE: PROSECUTION + DECK — prompt for Codex

Paste this whole file as your opening message. Read `prompts/00-SHARED-BRIEF.md` first,
in full.

---

You are the **prosecution and attack lane** of a two-agent team competing in COLOSSEUM
(Day 26, MCP/A2A Infrastructure & Agentic Routing). A second agent (Claude) works the
defense lane in parallel on disjoint files. Read `prompts/00-SHARED-BRIEF.md` before
anything else — it holds the rules, the rubric, the eight mechanisms, and the working
agreement. Then read `RULES.md`, `eval/README.md` and `deck/README.md` in full.

## Your territory

You own, and are the only agent who may edit:

```
eval/prosecute.py    eval/__init__.py    eval/README.md
deck/deck.json       deck/lineup.json
```

You must not touch `kit/`, `bots/`, `fixtures/`, or `agent/`. If you believe a `kit/` file
has a bug, **do not patch it** — the hash gate rejects the bundle. Record it in
`notes/FINDINGS.md` and design around it. `deck/README.md` already documents one such case:
`referee/ledger.py`'s `defense_event_confirmed()` string-compares a card's `defense_event`
against `enforced.verdict_applied` (`"forward"`/`"deny"`/`"rewrite"`), so read literally it
can never match any card's declared string. **Write `"gateway.denied"` on every card we
author** — it matches the frozen worked example and is the forward-compatible choice once
`ledger.py` is fixed. Do not "optimise" it to `"deny"`.

Note that `deck/README.md` claims every starter card already uses that string, and that is
not true: the shipped deck carries five distinct values (`gateway.denied` ×6, plus
`registry.pinned`, `guardrail.refused`, `gateway.budget_held`, `a2a.crosschecked`), and
`validate_deck.py` emits a `WARN R8-held-in-principle` for each of the four deviants. They
are warnings, not failures, so the deck is legal as shipped — but normalising all ten
attack cards to `"gateway.denied"` clears those warnings. Verify with `make validate`
rather than trusting either the README or this paragraph.

Work on branch `lane/prosecution`.

---

# PART A — `eval/prosecute.py` (Task 2, the bigger half of your lane)

## The contract

```python
def prosecute(trace: list[dict], answer: dict, card: dict) -> dict:
    """Return {"v": 1, "claims": [...]}. Synchronous, no I/O, no network, 5 s deadline."""
```

- **At most 4 claims per exchange, at most 1 per family.** Prosecution is a case, not a
  dragnet.
- Every claim needs `cls`, `evidence`, `expected`, `observed`, `argument` (≤400 chars).
  A claim naming several classes at once is **rejected**.
- **A false claim costs `0.8 × the weight of the class filed.`** The penalty scales with
  the class deliberately: break-even is **44.4% for every class**, so there is no heavy
  class worth shotgunning. File only what you can prove from the trace.
- You receive the opponent's **L1 events only**, with their `own_telemetry` stripped. You
  never see their source. Everything must be derived from events.

## What ships, and what does not

| Piece | State |
|---|---|
| `RUBRIC` / `family_of` / `weight_of` | reads `kit/referee/rubric.py` (fallback copy otherwise). Use it; never hardcode weights. |
| `evt_ref` / `span_ref` / `anchor_ref` | the three evidence grammars: `evt:0412`, `answer.span:3`, `anchor:Frame:…` |
| `group_calls(trace)` | buckets the L1 trace into per-`command` groups (`command`/`decision`/`enforced`/`tool_call`/`tool_result`/`mutations`) — the correlation nearly every detector needs |
| `split_sentences(text)` | the exact `answer.span:N` split the referee uses |
| `ProsecutionBudget.try_add(...)` | enforces ≤4 claims / ≤1 per family **by construction**; over-filing lands in `.dropped`, not an error |
| `detect_enforcement_failure` | **the one competently-implemented detector — this is your template.** Read it before writing anything. |
| 16 `_hook_*` stubs | one per remaining class, each `return []` with a docstring naming exactly what it needs |
| `score_prosecutor(fn, fixtures)` | measures any `prosecute`-shaped callable against the labelled fixtures |

`enforcement_failure` ships implemented because it is the most mechanical: *the card's
invariant was violated by a command AND the matching `enforced.verdict_applied != "deny"`.*
Pure function of the trace, no text to read, no judgement. Its shape — group calls → scan
for the predicate → cite the `command` and the `enforced` event **together** — is the shape
you reuse.

## The labelled fixtures are your scoreboard

```bash
python -m eval.prosecute        # runs score_prosecutor over fixtures/prosecution/labelled/
```

Baseline today: precision decent, recall near zero, because 16 of 17 classes are no-op
hooks. That is expected and correct. **Your entire lane is measured by moving recall up
without dropping precision below the break-even line.** Fixtures live in
`fixtures/prosecution/labelled/` (`clean.jsonl`, `family_a_infrastructure.jsonl`,
`family_b_truth.jsonl`, …) — read them, read `fixtures/prosecution/build_fixtures.py` to
understand how they were generated, and **never edit them** (`fixtures/` is read-only kit).
`clean.jsonl` is the false-positive trap: a detector that fires on it is costing us
`0.8 × weight` every time.

## Order of work

**Phase 0 — ground truth.** `make install`, download the world, `make doctor` until it says
`ready to spar.` Run `make test` and `python -m eval.prosecute`; record both baselines
verbatim. Read `detect_enforcement_failure`, then `kit/referee/detectors.py` (1648 lines —
**this is the actual referee that will judge every claim**; the kit ships it hash-synced
precisely so prosecution is not guesswork) and `kit/referee/verify.py`,
`kit/referee/ledger.py`, `kit/referee/adjudicate.py`. Note which classes are *deterministic*
(9 of them, decidable from the trace) and which are *adjudicated* (8, requiring the
referee's oracle) — the deterministic nine are where recall is cheap and safe.

**Phase 1 — the deterministic nine, weight-first.** Implement in this order, one at a time,
re-running `python -m eval.prosecute` after each and keeping the precision/recall numbers
in the commit message: `write_violation` (8) → `stale_read` (8) → `authority_exceeded` (10,
if trace-decidable in these fixtures) → `protocol_misuse` (6) → `wasteful` (3). Each of the
eight mechanisms in the shared brief is a detectable event signature: a `get_frame` with an
expired lease, a write retried after a 409 without an intervening `registry.provenance`
read, a duplicate write with no idempotency key, a `partial: true` result treated as
complete, a `slides.search` call when `slides.query` exists, a mask carrying fields that
are then cited (or citing fields that were masked out).

**Phase 2 — the answer-text classes.** `fabricated_citation` (8), `ungrounded` (5),
`hallucination` (7), `unsupported_precision` (4), `unflagged_conflict` (6), `overreach` (5),
`incoherent` (4), `non_responsive` (4). These read `answer` and cite `answer.span:N` —
`split_sentences` gives the exact split, so an off-by-one in the span index is a false
claim. Cross-check every cited anchor against what the trace shows was actually retrieved.

**Phase 3 — claim selection, which is where the points actually are.** With more detectors
than claim slots, *choosing* becomes the skill. Build an explicit selector: for each
candidate, estimate confidence, multiply by class weight, subtract `0.8 × weight ×
(1 − confidence)`, and file the top 4 subject to the one-per-family constraint. **Never
file a claim whose expected value is negative.** Filing 2 certain claims beats filing 4
where two are guesses. Make this selector a named, tested function — it is the single
highest-leverage piece of code in your lane.

**Phase 4 — argument quality.** `expected` / `observed` / `argument` are read by the
referee. Argument ≤400 chars, naming the invariant, the event, and the causal link.
"No claim, no damage" cuts both ways: a real defect argued vaguely earns nothing.

**Phase 5 — live.** `make spar BOT=operator AS=prosecutor` and `BOT=adversary AS=prosecutor`
across several `--seed` values. `operator` is the richest target — it pins and diffs but
forwards `traceparent` without verifying, and confuses identity with authority. Confirm the
fixture-measured detectors actually fire on live traces, and that nothing fires on a clean
exchange.

**Guard against overfitting the fixtures.** A detector that pattern-matches a fixture's
literal strings will score well offline and zero live. Detect the *mechanism* (an event
predicate over `group_calls` output), never a substring.

---

# PART B — `deck/` (Task 1)

14 cards: **10 attacks + 4 blanks.** We play 10 in a locked order (`lineup.json`), no
repeats. Legality, all checked offline by `make validate`:

- ≥3 MCP-layer · ≥3 A2A-layer · ≥2 gateway-layer
- ≥6 distinct classes of the nine
- **every `replica_flip` card must name a `path_id` in the measured drift set** — roughly a
  third of days do not drift at all, and a card aimed at one of those finds nothing
- lethality band: the deck must fall to `rookie` and be held by `adversary`

**Invalid cards never enter the tournament — they are rejected at lock, not at play. Run
`make validate` after every single edit.** Also run `make validate-bots` to see how the
three shipped decks score on the same checks.

A **blank** is `{"id", "kind": "blank", "ask": {...}}` — no class/invariant/mutation/trigger.
It plays a clean ask over an unmutated stack. It deals nothing by itself, but a defender who
quarantines it, refuses it, or gets prosecuted into a false positive on one takes **8**.
Blanks are how a deny-everything defence is punished; place them where the opponent is most
likely to be twitchy — right after our heaviest attacks.

The interesting decision in a card is **not the JSON syntax — it is which `ask` makes this
particular lie hardest to catch.** An `ask` whose `require` list forces the opponent onto a
drifting path, or into a language-negotiated tool, or into a frame fetch far from its
query (lease expiry), turns a survivable mutation into an unavoidable one. Study the three
bot decks (`bots/*/deck.json` + `bots/adversary/STRATEGY.md`) for what a graded attack looks
like, then rewrite ours — the starter deck is legal, verified, and deliberately unambitious.

`lineup.json` order matters: triggers fire on `call_index`, so a card's position changes
whether its mutation lands during discovery or mid-retrieval. Reason about the sequence
explicitly and write the reasoning down.

**Coordinate with Claude before finalising the deck.** Every attack we author is a defence
Claude must also survive from the other side — the ask types and mutation ops we choose tell
Claude exactly what to harden. Post the deck rationale in `notes/FINDINGS.md`.

---

## Local tests

`tests/` is kit — read-only, and must stay green (`make test`). Write your own under
`eval/tests_local/` (git-ignored, never bundled). Test at minimum: `prosecute()` never
raises on malformed traces; never returns >4 claims or >1 per family; every claim passes
`_schema_errors`; the whole thing finishes well inside 5 s on the largest fixture.

## Reporting

After each phase report: the commands run, their real output, and the
precision/recall/score numbers from `python -m eval.prosecute` before and after. Never
report a number you did not produce. Append every mechanism you nail down to
`notes/FINDINGS.md` under `## [codex] <date> — <topic>` — Claude needs the same facts from
the defence side, and a mechanism proven once should never be re-derived.

If something in the kit contradicts this prompt, **the kit wins** — it is the executable
spec. Say so, and adapt.
