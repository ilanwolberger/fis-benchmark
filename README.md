# fis benchmark — corpus, protocol, results

The public benchmark behind [fis.noproduct.com/benchmark](https://fis.noproduct.com/benchmark):
the corpora, the selection predicates, the measurement protocol, the results, and — deliberately —
every caveat and failure the measurement surfaced, including the unflattering ones.

**Why this repo exists.** Every tool in this space advertises an accuracy number. As far as we can
tell, none publishes its corpus or method. A number without a corpus is indistinguishable from a
poster. This repo is the corpus, so the numbers can be checked, criticized, and re-derived.

## What's here

| File | What it is |
|---|---|
| [`METHODOLOGY.md`](METHODOLOGY.md) | Selection predicates, protocol, results, adjudications, and every open caveat — including verdict instability, the post-hoc corpus removal (disclosed both ways), and the eval-doctrine limitation |
| [`corpus.clean.json`](corpus.clean.json) | 25 mature open-source libraries with no known issue (false-critical-rate corpus) |
| [`corpus.vuln.json`](corpus.vuln.json) | 11 intentionally-vulnerable training apps (recall corpus) |
| [`corpus.app.json`](corpus.app.json) | 20 mature, widely-deployed open-source web applications (application-axis corpus) |
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

Headline numbers without their qualifiers misrepresent this benchmark. The short version:

- Small corpora — point estimates, not tight bounds (exact 95% CIs are in the doc).
- Verdict-level recall is **parity with a trivial aggregation of open-source scanners**, not
  detection superiority. The doc says this plainly.
- The library 0% false-critical rate is not an application number; the application axis is
  measured separately and has different failure modes (over-escalation on dependency drift,
  since retuned — the pre-fix numbers stay published).
- Single pass per repo — verdict variance is real, observed, and disclosed as unmeasured.

If you find an error in the corpus or method, open an issue. Corrections get published the same
way the juice-shop-ctf removal was: in full, both numbers, no silent edits.
