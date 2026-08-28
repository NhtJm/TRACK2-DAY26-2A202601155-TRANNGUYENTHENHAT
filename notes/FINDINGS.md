# FINDINGS — shared, append-only

Both lanes write here. Append a section; never rewrite or delete another lane's entry.

Format:

## [claude|codex] YYYY-MM-DD — topic
- what you observed, and the command/event that proves it (e.g. `evt:0217`, a spar seed)
- what it means for the other lane

---

## [codex] 2026-08-28 — prosecution mechanisms and executable-spec gaps

- Baseline: `make test` produced `4606 passed, 4 skipped`; `python -m eval.prosecute`
  produced precision `1.000`, recall `0.059`, F1 `0.111`, false-claim rate `0.000`.
- Implemented mechanism predicates, not fixture-id/string matches:
  - infrastructure: missing/reused write preconditions, lease/partial-result misuse,
    stale-replica self-contradiction, and unchanged non-retry-safe retries;
  - truth: returned-row disagreement, unreturned citations, omitted-mask use,
    unsourced precise atoms, and approximation upgraded to exact precision;
  - safety: executed cross-act writes, verbatim private-row leakage, and explicit
    compliance with instruction injection;
  - quality: unflagged returned contradiction pairs, unsolicited actions/advice,
    same-subject numeric contradictions, and answers addressing no ask field.
- Candidate selection uses `EV = w * (1.8p - 0.8)`, drops non-positive-EV candidates,
  sorts by EV, and enforces at most four claims / one per family before emission.
- Current labelled-fixture result: 34 filed, 34 verified, 0 unproven, 0 false,
  0 rejected; precision/recall/F1 `1.000`, false-claim rate `0.000`.
- Referee mismatch to remember on defense: prosecution can use
  `exchange_start.p.defender` as an observable proxy for `ctx.act`, but gate-1's
  deterministic authority detector is ultimately supplied the arena's real
  `VerifyContext.act`; defense must derive authority from `act`, never identity alone.
- Kit contradiction: `tests/test_prosecute.py::test_starter_end_to_end_against_the_full_fixture_set`
  hard-codes that recall remains below `0.15` and all 16 hooks remain silent. It is
  necessarily stale once Task 2 is implemented; protected `tests/` was not edited.
- Packaging contradiction: `.gitignore` contains the promised local-test entries, but
  `kit.submit` recursively bundles every file under `eval/` regardless of git ignore. Local
  prosecution tests must be removed before submission unless submit is fixed upstream;
  `kit/` was not edited.
- Bootstrap blocker: both documented release tag `world-df8c55dabb35` and the user-provided
  `world-df8c55dabb35p` return `release not found` from the current private origin. Therefore
  `make doctor`, `make validate`, and live spar cannot pass until the instructor corpus appears.

## [codex] 2026-08-28 — deck proposal for defense review

- Preserve every existing ask, anchor, mutation target, and drift `path_id` while the real
  corpus is unavailable; changing any would be unmeasurable and risks a lock-time rejection.
- Normalize every attack card's `defense_event` to the frozen worked-example string
  `"gateway.denied"`; do not replace it with `"deny"` even though current ledger comparison is
  buggy.
- Proposed 10-round lineup:
  `atk_05, blk_01, atk_04, atk_08, atk_03, atk_09, atk_06, atk_02, atk_07, blk_02`.
  It keeps exactly 3 MCP + 2 gateway + 3 A2A attacks, removes the duplicate second identity
  card and the easier shadow card, and places blanks immediately after schema/catalog pressure
  and after the late identity/replica sequence where deny-heavy defenses are most likely to
  overreact.
- Defense implications: bound registry/catalog reads (`atk_05`), treat retrieved notes as data
  (`atk_04`), verify Agent Cards and skills (`atk_08`), pin/diff replicas (`atk_03`, `atk_02`),
  cross-check peer answers (`atk_09`), require write preconditions (`atk_06`), and bind delegated
  authority to `ctx.act` (`atk_07`). Blanks require allowing clean, schema-valid asks rather than
  blanket denial.
- Claude coordination was attempted through local CLI artifact
  `.omx/artifacts/claude-colosseum-deck-review-only-do-not-edit-files-current-deck-ha-2026-08-28T04-31-35-485Z.md`,
  but the provider returned `Credit balance is too low`; no Claude review was available.

## [codex] 2026-08-28 — real-world validation and live prosecution

- The defense worktree later supplied the corpus through a symlink at
  `kit/world/df8c55dabb35`; manifest reports world `df8c55dabb35`, 12,375 pages and no
  `truth.json`. `make doctor` now reaches `ready to spar.`
- `make validate` on that real world passes with 0 failures and the validator's single honest
  lethality-band warning. `make validate-bots` reports PASS for rookie/operator/adversary.
- `make doctor` prints `24750 pages` because its command sums every `counts` value including the
  manifest's `__total__=12375` in addition to each namespace count. This display bug does not
  affect world loading or validation; do not patch `Makefile`/`kit/` from either lane.
- Live prosecution, seeds 1-3: operator loses 100-0 on all tested seeds; adversary draws seed 1
  and wins seeds 2-3. The MISSED report shows `protocol_misuse x3`, but those exchanges also
  produce a higher-EV family-A `enforcement_failure` claim. Because the contract permits only
  one claim per family, the selector correctly keeps the weight-10 enforcement claim instead of
  the weight-6 protocol claim; the remaining latent flag is not an additional legally fileable
  claim in that exchange.
- Final review aligned the third deterministic protocol predicate with the frozen referee: a
  `#L…`/`#s…` citation after every matching `get_frame` mask omitted `body` is family-A
  `protocol_misuse`, not the semantic `ungrounded` heuristic. It also lowered confidence to
  `0.75` when an authority claim must infer `ctx.act` from `exchange_start.defender`; explicit
  `exchange_start.act` remains `0.995`.

## [codex] 2026-08-28 — referee-vs-spar protocol correction

- `kit.referee.detectors.protocol_misuse` fires from the `command` event when
  `slides.get_frame.lease_id` is falsy, even if the gateway later denies the command and no
  `tool_call` occurs. The prior prosecution hook incorrectly suppressed this claim after a deny;
  it now matches the frozen detector and cites the command directly. A constructed denied trace
  produces one `protocol_misuse` in both implementations; the labelled scorer remains 34/34.
- `fabricated_citation` has two frozen-referee legs: never returned in this trace, or does not
  resolve in `pages.jsonl`. The prosecutor can prove and files the first. Its synchronous,
  trace-only API receives no World, so it conservatively leaves the resolution leg as a false
  negative rather than guessing from syntax. Spar's synthetic anchors therefore make its latent
  count unsuitable as a prosecution precision measurement.
