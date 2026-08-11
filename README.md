# NittiM benchmark — corpus, protocol, results

The public benchmark behind [nittim.com/benchmark](https://nittim.com/benchmark):
the corpora, the selection predicates, the measurement protocol, the results, and — deliberately —
every caveat and failure the measurement surfaced, including the unflattering ones.

> **Renamed.** This product shipped as "FIS" through mid-2026 and was renamed **NittiM** in
> August 2026. The repo slug (`fis-benchmark`) is kept for link stability; everything else below
> uses the current name.

**Why this repo exists.** Every tool in this space advertises an accuracy number. As far as we can
tell, none publishes its corpus or method. A number without a corpus is indistinguishable from a
poster. This repo is the corpus, so the numbers can be checked, criticized, and re-derived.

## What's here

| File | What it is |
|---|---|
| [`METHODOLOGY.md`](METHODOLOGY.md) | Selection predicates, protocol, results, adjudications, and every open caveat — including the held-out validation failure, cross-family adjudication, verdict variance, and the eval-doctrine limitation |
| [`corpus.clean.json`](corpus.clean.json) | 25 mature open-source libraries with no known issue (false-critical-rate corpus) |
| [`corpus.heldout.json`](corpus.heldout.json) | 9 libraries selected and frozen *after* the clean-corpus scanner fixes landed, to test whether those fixes generalized — they didn't. **Now burned**: the failures it surfaced were used to write the next round of fixes, so it is an in-sample record, not a reusable held-out set |
| [`corpus.vuln.json`](corpus.vuln.json) | 11 intentionally-vulnerable training apps (recall corpus) |
| [`corpus.app.json`](corpus.app.json) | 20 mature, widely-deployed open-source web applications (application-axis corpus) |
| [`corpus.vibe.json`](corpus.vibe.json) | 10 self-declared AI-built applications (Lovable, v0.dev, Claude Code, Bolt.new) — the first corpus made of NittiM's actual target class. Authorship label only, no vulnerability ground truth; field-mode reads only, no FCR/recall claim |
| [`corpus.messy.json`](corpus.messy.json) | 3 very large, complicated production codebases (behavioral field-test corpus — no ground truth by design) |

## What's deliberately not here

The audit engine, the deterministic scanners, the benchmark runner, and the exact guardrail
rules (invariant tables, severity cutoffs, path-classification rules) live in the private product
repo. `METHODOLOGY.md` describes their *behavior* and its measured consequences, not their
implementation. Where a passage was summarized rather than reproduced, it says so inline.

The comparison columns (Gitleaks, Trivy, Semgrep) use the exact public tool versions and
invocations listed in `METHODOLOGY.md`, so that half of the protocol is fully reproducible
by anyone.

## Reading the numbers honestly

Headline numbers without their qualifiers misrepresent this benchmark. The short version — leading
with the one that says the most about how seriously to take everything else:

- **A precision fix that looked complete, wasn't.** The clean-corpus scanner was tuned until it
  passed 25 repos at 100%. A held-out set of 9 repos never used in that tuning then failed at
  55.6% clean-pass, with a third of it driven by the secret scanner alone. The pure model stayed
  at 0% false criticals throughout — the failure was entirely in the deterministic layer, and it's
  published in full, including the correction of an earlier draft that had misread one of the four
  failures as a true positive. See "Held-out validation" in the doc.
- Small corpora — point estimates, not tight bounds (exact 95% CIs are in the doc).
- Verdict-level recall is **parity with a trivial aggregation of open-source scanners**, not
  detection superiority. The doc says this plainly.
- The library 0% false-critical rate is not an application number; the application axis is
  measured separately and has different failure modes (over-escalation on dependency drift,
  since retuned — the pre-fix numbers stay published). A rival-lab judge later re-checked the
  findings behind those verdicts and came back *stricter*, not friendlier.
- Single pass per repo — verdict variance is real, observed, and disclosed. A first N=3 measurement
  found zero flips on three repos, but none of them sat near a verdict boundary, which is exactly
  the condition the disclaimer was about — read the doc before citing "0% flip rate."
- The self-declared-AI-built corpus has **no vulnerability ground truth** — it supports qualitative
  field reads, not an FCR or recall number. Anyone citing a pass rate against it is misreading it.

If you find an error in the corpus or method, open an issue. Corrections get published the same
way the juice-shop-ctf removal was: in full, both numbers, no silent edits.
