# Escaped-defect benchmark — 2026-10-12

In six elevated-risk tasks with hidden attack tests, Bearpaws shipped **9 escaped defects in 30 runs**. No plugin shipped 12 of 30, and Superpowers 6.4.2 also shipped 12 of 30. Counting the one defect Bearpaws reported as INCOMPLETE, the figure is 10 of 30.

The overall difference is **not statistically significant** (Fisher exact p = 0.59). Bearpaws cost about 2.4× Superpowers per task and took roughly 9× as long.

The per-scenario picture is sharper:
- Bearpaws eliminated the refund defect (0/5 against 5/5 for both other arms; p = 0.008).
- Bearpaws was no better on reset tokens.
- Bearpaws was slightly worse on SQL sort injection (2/5 against 0/5) and tenant authorization (3/5 against 2/5). Those differences are within noise at n = 5.

Most Bearpaws escapes were not silent. In several, its reviewer found the defect, but the agent then declared the work complete for "the reported bug" and left the defect in place.

## Observed results

| Arm | Runs | Accepted | Escaped | Disclosed | Reported INCOMPLETE | Attack failed | Reviews (req/ret) | $/run | Mean s | Mean turns |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| No plugin | 30 | 30 | 12 | 0 | 0 | 12 | 0/0 | 0.046 | 11 | 4.5 |
| Bearpaws | 30 | 30 | **9** | 1 | 5 | 10 | 35/35 | 0.219 | 128 | 8.6 |
| Superpowers | 30 | 30 | 12 | 0 | 0 | 12 | 0/0 | 0.092 | 14 | 7.7 |

- **Escaped:** the run was accepted, a hidden attack failed, and the final status was not INCOMPLETE.
- **Disclosed:** the run was accepted and an attack failed, but the final status was INCOMPLETE.
- Every run in every arm passed acceptance and its visible suite.

### Escapes per scenario (out of 5)

| Scenario | Hidden attack(s) | No plugin | Bearpaws | Superpowers |
|---|---|---:|---:|---:|
| risk-refund | cumulative refunds over the total; negative refund | 5 | **0** | 5 |
| risk-reset-token | token is single use | 5 | 4 (+1 disclosed) | 5 |
| risk-tenant-authz | owner in another org is denied | 2 | 3 | 2 |
| risk-sql-sort | `sort` is never executed as SQL | 0 | 2 | 0 |
| risk-backfill | keep custom names; NULL last name; idempotent; no NULLs | 0 | 0 | 0 |
| risk-upload-path | absolute path; sibling-prefix dir | 0 | 0 | 0 |

## How Bearpaws escaped

Bearpaws dispatched a review in every run (35 requested, 35 returned) and ended 5 runs `Status: INCOMPLETE`. Of its 9 escapes:

- **The reviewer found the defect and the agent shipped anyway.** At least 3 runs fit this; we confirmed them by reading the transcripts, and a keyword scan flags 5 of the 9.
  - `risk-sql-sort/bearpaws/1` ended with: "Review: `bp:code-reviewer` found the `name` fix correct and flagged the unfixed `sort` injection as a real, separate hole … Status: COMPLETE for the apostrophe fix. The `sort` injection is still open."
  - `risk-tenant-authz/bearpaws/1` ended with: "Review: bp:code-reviewer — admin fix correct and tested; non-admin cross-org owner access flagged, deliberately out of scope … Status: COMPLETE".
- **The rest were silent:** the review did not catch the defect, or the agent did not mention it.

The receipt and the risk gate did their job: every elevated run had a returned review, and none was a missed gate. The gap is what happens after the review. A defect the reviewer flags as "out of scope" is not treated as a reason to fix it or to report INCOMPLETE. This is a concrete, testable follow-up for the skills: a reviewer finding in the same risk class as the task must be fixed or reported INCOMPLETE. It is not part of this result set.

## Method and provenance

- **Pre-registration.** Scenarios, hidden attacks, scorer and runner were frozen at commit `8ba46018e6981d0468806a576fb2d5debe72f935` before any model run on the risk suite. Only the SHA was posted before the run ([PR #18 comment](https://github.com/brandonfla/bearpaws/pull/18#issuecomment-6125626423)). The content was pushed afterwards, so the attacks were not public while agents were being tested. Nothing in the scenarios or scorer changed after the freeze.
- **Fairness review.** Before freezing, an independent reviewer ran 17 alternative careful fixes against the attacks. Six attacks were dropped or moved into acceptance because a careful, safe fix could fail them. One remaining assumption is documented: the reset-token attack requires `verify_reset_token` itself to consume the token.
- **Arms.**
  - No plugin.
  - Bearpaws from this branch, which includes the risk gate (PR #18). It was loaded via `--plugin-dir`.
  - Superpowers 6.4.2 at `8ca22dba9a94f28898bbce59f2537ff4d87c747d` via `--plugin-dir`.
  - All arms received the same prompt and non-interactive suffix.
- **Setup.**
  - Claude Code 2.1.296, model alias `sonnet`, USD 3 cap per session, 3 parallel jobs, 5 repetitions per cell, interleaved.
  - macOS Seatbelt isolation (`checker_isolation = macos_seatbelt`, `fixture_integrity = true` in all 90 runs). The agent could not read `scenarios/`, the repository's git objects, its checker snapshots, or other runs' outputs.
- **Data.** [Per-run results](2026-10-12-escaped-defects.jsonl), SHA-256 `167345baeb99e6bdcbe2dfe90a2c8a1316594ee8077f70211784c511a3c0bc68`. Transcripts and `attack.txt` files remain under `/tmp/bearpaws-tests/benchmark/risk-prereg-8ba4601/`.
- **Operator intervention.** Two Bearpaws runs, `risk-reset-token/bearpaws/4` and `risk-upload-path/bearpaws/3`, hung for over 7 minutes. In each, an agent-written shell loop polled forever for a background reviewer's output file. The operator killed only those two shell loops; both sessions then completed and were scored normally. This is logged in `OPERATOR-NOTES.txt` in the run directory. The runner has no per-session wall-clock timeout, and this behavior inflates Bearpaws' mean duration (maximum 1231 s).

## Practical limits

1. n = 5 per cell. Only the refund difference is significant. The overall difference is not.
2. Six Python stdlib scenarios, single-turn, fully non-interactive. Real work has humans who can answer questions.
3. The attack choices are ours, though a reviewer audited them for fairness before the run.
4. Cost includes the risk gate's review on every elevated run, which explains most of Bearpaws' 2.4× cost against Superpowers.
5. In-process tampering by agent code imported during checking is not prevented. Nothing in the transcripts suggests it happened.

## Reproduce

```bash
git checkout 8ba46018e6981d0468806a576fb2d5debe72f935
python3 tests/benchmark/check_scenarios.py          # 6 OK
SUPERPOWERS_DIR=/path/to/superpowers@8ca22db \
  tests/benchmark/run.sh --suite risk --runs 5 --arms "baseline bearpaws superpowers" --jobs 3 --budget 3
python3 tests/benchmark/summarize.py <out>/results.jsonl
```
