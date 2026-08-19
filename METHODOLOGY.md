# NittiM Benchmark — Methodology

_The reproducible spec for the NittiM Benchmark: how each corpus is built, how the comparison is
run, what ground truth is, and every known weakness. The point is that a hostile reader can
re-derive the corpus, verify any entry, or flag one that fails a predicate. Results + narrative
live in `reports/fis/fis-benchmark-summary-*.md`; this file is the method._

> **Renamed.** This product shipped as "FIS" through mid-2026 and was renamed **NittiM** in
> August 2026. Historical filenames and artifact paths below (e.g. `reports/fis/…`) predate the
> rename and are left as-is for traceability; all prose uses the current name.

---

## Benchmark Corpus — Selection Criteria (hardened)

This section documents the predicates used to construct each corpus and absorbs the substantive objections raised in adversarial review. Every predicate is checkable against public data (GitHub API, OSV, npm/PyPI/pkg.go.dev/crates.io release history, and a second independent secret scanner). The goal is that a hostile reader can re-derive the corpus, verify each entry, or flag one that fails a predicate.

Source files: `benchmark/corpus.clean.json` (n=25 repos), `benchmark/corpus.vuln.json` (n=11 repos).

> **Sample size, stated up front and without spin.** Both corpora are **small**: n=25 clean, n=11 vulnerable. This is a **v0 pilot**, sufficient to detect *gross* false-alarm and false-negative behavior, **not** to bound either rate tightly. At n=25 with zero false criticals, the exact (Clopper-Pearson) 95% confidence interval for the true False Critical Rate is **0%–13.8%** — i.e. a single false critical would move the observed rate to 4%, and the data cannot rule out a true rate as high as ~14%. The headline "0% FCR" is a point estimate on a small pilot, not a precision claim. The v1 target is ≥100 clean repos (which would put the 95% upper bound near 3% at the same confidence).

---

### Clean corpus — 25 mature libraries

**Purpose.** Measure the False Critical Rate (FCR): how often the NittiM auditor raises a Critical on code with no committed secrets and no known dependency CVEs. The metric isolates the model's own false-positive tendency from the deterministic scanners.

> **Scope limitation — read this before the 0% number.** This corpus's 0% FCR is measured on **library / framework / CLI code only.** Predicate C6 below excludes web applications from this clean set by design — yet web applications are the exact code type NittiM markets itself to audit. The application false-critical rate was therefore measured **separately**, on the 20-app application corpus documented later in this file (2026-07-01) — with its own, different failure modes and caveats. Never read the library 0% as an application number: the headline must always be *"0% false-critical rate on mature library code; the application axis is measured separately, below."*

#### Inclusion predicates (all must hold at corpus-freeze date)

| # | Predicate | Rationale |
|---|---|---|
| C1 | GitHub stars ≥ 1,000 | Proxy for community review surface; a latent issue would more likely have surfaced publicly. |
| C2 | First public release ≥ 4 years before corpus construction | Eliminates projects whose security posture is unproven. |
| C3 | Last commit within 12 months | Confirms active maintenance. |
| C4 | Zero open critical/high CVEs in OSV at freeze date (verified via `osv-scanner` over each manifest) | A repo with an open high CVE is not clean; including it would conflate a correct NittiM warning with a false positive. |
| C5 | Zero committed secrets per **two independent scanners** (NittiM's own scanner **and** Gitleaks) | Breaks the circularity of testing the tool against its own secret-scan output — see note below. |
| C6 | Library, framework, or CLI tool — **not** a web application | Scopes the FCR to library-style code; application attack surface is the domain of the vuln set. **This is a scope boundary, acknowledged as a limitation, not a strength** (see scope box above). |
| C7 | Language in {JavaScript, TypeScript, Python, Go, Rust} | The languages NittiM's analysis engine currently supports. |

**C5 circularity fix.** The original C5 checked "no secrets" using only NittiM's own scanner — meaning a secret type NittiM systematically misses would mark a repo "clean," and if the model also misses it, the benchmark records a false 0% from a shared blind spot. C5 now requires **both** NittiM's scanner and **Gitleaks** (an independent engine, already in the comparison table) to agree there are no committed secrets. This does not eliminate the possibility of a secret type both miss, but it removes the single-tool self-reference.

#### Selection bias: disclosed, not denied

Three concentration biases survive in the v0 corpus and we name them rather than bury them:

1. **Minimal-dependency bias removes the hardest false-positive condition.** 18 of 25 (72%) entries are flagged `minimal_dep: true`. The original rationale — "isolate the model from the dep-CVE guardrail" — is methodologically backwards for measuring real-world precision: a model's *hardest* false-positive challenge is a complex, dependency-heavy repo where it must reason "library X has security associations but is used only in a non-production path here." That reasoning failure is invisible in a near-zero-dep corpus. **The 0% FCR therefore does not bound the false-positive rate for repos with non-trivial dependency trees.** v1 commitment: a `complex-dep clean` sub-corpus (e.g. sequelize, passport, socket.io, httpx) and **FCR reported separately by `minimal_dep` bucket**.

2. **Ecosystem / maintainer concentration.** JavaScript/TypeScript is 13 of 25 (52%), and that slice leans on a single maintainer's zero-dep utility family (the sindresorhus packages) plus the Pallets cluster (flask/click/jinja) in Python. The earlier draft's "five distinct maintainer lineages" claim **overstated diversity and is retracted**: three repos from one author in one micro-utility niche is not lineage diversity. v1 commitment: cap any single ecosystem at ≤35% of the corpus and replace several pure-JS utilities with different code-type categories (an ORM, an auth library, a serialization library, a networking library).

3. **No security-primitive or async/network libraries.** None of the 25 entries implement authentication, password hashing, JWT/crypto, or session management — i.e. the exact surface where a model that pattern-matches "sees `hash`/`salt`/`secret` → Critical" would false-positive. Their absence means the FCR has **not** been tested against the model's most likely false-positive failure mode. v1 commitment: add CVE-clean security libraries (argon2, Python `cryptography`, `golang.org/x/crypto`) and re-run.

#### Anti-cherry-picking guarantee — and its honest limit

The selection question was fixed before any audit ran: *"What are the most widely-recognized, most-downloaded, security-mature libraries in each supported language?"* — a question with public answers (download stats, star rankings) independent of NittiM. **No repo was added because NittiM passed it; no repo was removed from the clean set because NittiM flagged it.** The strongest supporting evidence: when four repos initially produced scanner false alarms (ky's test-fixture token; lodash and zod devDependency CVEs; flask's example-app pinned requirements), the response was to **fix the scanner's precision** (commit `a669e04`) and keep all four repos — not to drop the inconvenient data.

**The honest limit of this guarantee:** there is **no timestamped pre-registration** for v0. No public commit of `corpus.clean.json` predates the first audit run; the integrity claim rests on the author's assertion plus the retained-problematic-repos evidence, which is suggestive but not proof. v1 commitment: push a `corpus.clean.lock.json` (resolved commit SHA per repo) to a dated public release/tag **before** running, so timestamps are independently auditable. Until then, treat the guarantee as "asserted and circumstantially supported," not "cryptographically established."

#### A note on the 84% → 100% clean-pass figure

The improvement from 84% to 100% clean-pass was produced by **fixing the scanner in response to the benchmark** (commit `a669e04`). Each fix was individually correct, but the 100% is **in-sample**: the scanner was tuned to the same 25 repos it is now scored on, which is a mild form of overfitting. The 100% should be read as *"post-fix, in-sample."* A held-out validation set of 5–10 repos not in the original 25, audited after the fix, was an **outstanding v1 item** to confirm the fix generalized rather than solving only the four observed cases. **It was run on 2026-08-11. The fix did not generalize.** The result is the next section.

---

### Held-out validation — the v1 generalization test, and its failure

Source file: [`corpus.heldout.json`](corpus.heldout.json) (n=9). **This corpus exists to answer one question and it answered it badly for us.** The 100% clean-pass above is in-sample; these 9 repos were selected *after* the scanner-precision fixes had landed, were never run, inspected, or used to tune anything before being frozen, satisfy the clean corpus's inclusion predicates (with one disclosed exception below), and are absent from every other corpus in this benchmark — checked against all of them before freezing. Every entry was verified live against the GitHub API (exists, not archived, language, stars, push recency), OSV.dev, and an independent Gitleaks pass over a fresh shallow clone.

> **One predicate was verified at a weaker level than it should have been, and it produced a wrong headline.** The clean corpus's dependency-CVE predicate is a *manifest*-level check — no open CVE anywhere the repo's own manifest pins, not just in the library's own release. This corpus was verified with a *package*-level check instead (does the library itself, at its own latest version, have an advisory) — a strictly weaker test. `encode/httpx` fails the manifest-level check: one of its pinned tooling dependencies carries three HIGH advisories. It was admitted anyway because the weaker check passed it, and that admission produced the fourth not-green verdict below — which an earlier draft of this document then miscredited as a true positive. Both the admission and the miscrediting are disclosed rather than quietly fixed after the fact; a manifest-level re-verification of all nine entries is outstanding before this corpus can be cited as a full held-out result.

The set also deliberately closes selection-bias gaps disclosed above: it adds **security-primitive / crypto libraries** (a JOSE/JWT library, a widely-used general-purpose cryptography library, a Go JWT implementation) the original 25 had none of, adds **non-minimal-dependency** libraries (an async HTTP client, a web framework, an async runtime, an HTTP implementation, a Go web framework), and caps **JS/TS at 2 of 9 (22%)** against the original's 52%, across nine unrelated maintainer organizations with zero overlap with the clusters already in the clean corpus.

> **2026-08-19 — the outstanding manifest-level re-verification was run.** `osv-scanner` 2.5.1,
> `scan source -r`, over a fresh clone of all nine. An independent tool on purpose: verifying this
> predicate with NittiM's own dependency scanner would be the scanner grading its own corpus.
> **Result: 3 pass · 2 fail · 4 not evaluable.**
>
> 1. **`encode/httpx` fails, as already disclosed** — 6 CRITICAL/HIGH advisories reachable from its
>    pinned tooling requirements.
> 2. **`pyca/cryptography` also fails, which was NOT previously disclosed** — 2 HIGH advisories via a
>    pin in its CI constraints file. It is one of the three *secret-scanner* not-greens, so this does
>    not excuse that false positive, but a second entry does not satisfy the admission predicate.
> 3. **The predicate is not evaluable for 4 of 9** — `expressjs/express`, `hyperium/hyper`,
>    `rust-lang/regex` and `tokio-rs/tokio` commit no lockfile, so a manifest-level CVE check has
>    nothing resolved to read. That is a defect in the predicate, not in the repos: libraries
>    deliberately do not commit lockfiles. C4 needs restating — manifest-level where a lockfile
>    exists, explicitly N/A where none does — rather than being treated as quietly satisfied.
>
> The corpus therefore does not meet its own C4 for at least two of nine and cannot be shown to meet
> it for four more. **This does not exonerate the dependency scanner**: the `encode/httpx` not-green
> was already traced to a dev/runtime misclassification, a real defect recorded below. Both hold at
> once. What the re-verification adds is only that the advisories themselves were real — the error
> was which dependency set they were attributed to. **Untouched:** this is a dependency predicate, so
> the 33.3% secret-scanner rate and the 0% pure-model rate stand exactly as measured.

> **Read the comparison honestly in both directions.** This corpus is *harder by construction* — it was loaded with precisely the conditions the original disclosure named as untested, so 55.6% here is not a like-for-like regression against the original 25's 100%. But that cuts the other way too: those conditions were named as untested because they are where a false positive was *expected*, and the expectation was correct. A precision fix that holds only on the code shapes it was written against is exactly what "in-sample" means.

#### Result — deep tier (Opus) and free tier, same 9 repos

| Metric | Held-out (n=9) | In-sample library corpus (n=25) |
|---|---|---|
| **Clean-pass rate** (verdict green on clean code) | **55.6%** (5/9) · 95% CI 21.2%–86.3% | 100% (25/25) |
| Not-green rate (candidate false alarms) | 44.4% (4/9) · 95% CI 13.7%–78.8% | 0% |
| — secret-scanner driven | 3 repos | 0 |
| — dep-CVE driven | 1 repo | 0 |
| — model-only driven | **0** | 0 |
| **Pure model false-critical rate** | **0%** (0 findings) · 95% CI 0%–33.6% | 0% · CI 0%–13.7% |
| Mean readiness score (higher = safer) | 88.6 | — |
| **Free-tier secret-scanner false-critical rate** | **33.3%** (3/9) · 95% CI 7.5%–70.1% | 0% (application corpus) |
| Free-tier dep-CVE hard evidence | 1 repo (adjudicated a false alarm below, not a catch) | — |
| **All-cause not-green rate after adjudication** | **44.4%** (4/9) — 3 secret-driven + 1 dep-driven, **zero of the four defensible** | 0% |

#### The verdict on the commitment: failed at both deterministic axes, held at the model layer

- **The model is exonerated again, out of sample.** Zero criticals across all 9 repos, zero model-only not-green verdicts, mean readiness 88.6 — on a corpus deliberately stocked with the class of code (JWT/JOSE/crypto/TLS) most likely to trigger a model that pattern-matches security-sounding tokens into a Critical. That was the named gap in the original clean corpus, and it is now tested and clean. At n=9 the true model FCR could still be as high as ~33.6%, so this closes the *gap*, not the *question*.
- **The secret scanner failed to generalize, and that is the whole headline.** The precision rules that took the in-sample application secret-FCR from 35% to 0% (see the application-corpus section below) were written as *general classes of non-production content* rather than per-repo allowlists — genuinely the right move, and it bought real generalization: one library produced **47 raw secret hits and still verdicted green**, every one landing in a shape the rules already recognized as a test fixture. But three repos in this held-out set put their non-production key material in shapes the rules had never seen, and the rules did not read them.
- **The dep scanner also failed to generalize — and an earlier draft of this document got that wrong, in NittiM's favour.** The fourth repo's not-green was originally recorded here as the run's one *true positive* ("the dep axis passed its held-out test"). **Adversarial re-check refuted that.** All three verdict-forcing advisories trace to a single package that is **not a runtime dependency** of the audited project at all — it appears only in a tooling/requirements file whose own first line says it is pinning tooling, listed beside a linter, a docs generator, and a test runner. The scanner classified every other package in that same file as non-production and misclassified only this one as production — the identical dev/runtime misclassification defect from the July application-corpus run (see "Actionable defects" below), recurring out of sample. The severity-split verdict floor behaved exactly as designed given a runtime-High input — severity handling differs by evidence class, and specifics are not published — but it was fed a misclassification, so this is the floor working correctly on a wrong input, not a held-out pass.
- **Consequence for the headline:** all four not-green verdicts on this corpus are candidate false alarms — three secret-driven, one dep-driven. The earlier "3 false + 1 true" split is withdrawn.

#### What actually fired, and a correction along the way

The run artifact recorded secret-hit counts, not paths, so an initial attempt to attribute the three secret-driven failures used an independent scanner's hit locations on the same commits as a proxy. **That inferred attribution was wrong on all three repos** — two of the three locations it named had never been read by the audit engine at all (the file types were outside what gets fetched), so it would have pointed a fix at code the engine never sees. It is kept in the private working notes as a record of what an inferred, non-measured attribution costs, but the numbers here are the corrected, directly-measured causes:

- One library: 22 of 23 verdict-forcing hits sat in two non-source directories used for runnable documentation examples and for the project's own test suite — a directory-naming convention the classification rules had not previously recognized; the 23rd hit was an unrelated format-check string in shipped source.
- One library: both verdict-forcing hits were **in shipped source**, not a fixture directory at all — an OID label string and a format-marker constant, neither a credential. No directory rule could have reached either; this is a token-shape false positive.
- One framework: the single verdict-forcing hit was a comment-annotated constant in shipped source, not a test fixture — also a token-shape problem, not a path-classification one.

Two of the three are a **naming-vocabulary gap** (non-production directory conventions the rules had never seen), one is a **token-shape problem** (ordinary source strings that happen to match a credential-shaped pattern). Neither is a deep defect and both are fixable — but "fixable once observed" is the definition of the in-sample failure this corpus was built to detect, and the next held-out set will find the next few spellings.

#### The fix — general rules, and the trade each one pays

Four general precision rules landed the same day. Every rule targets a **class** of non-production or non-credential content, is documented with the case that motivated it, and **demotes rather than drops** a finding — it stays in the report as evidence, it just loses the power to force the verdict. No per-repo allowlist exists anywhere in the scanners.

| # | Class targeted | What it trades away |
|---|---|---|
| Idiomatic non-production locations | Additional non-production directory-naming conventions beyond the ones already recognized | Broadens what gets demoted, at real cost: one newly-recognized convention carries a wider blast radius than the others; the trade is documented internally. Disclosed as the widest-blast-radius trade of the four |
| PEM-boundary-as-complete-literal | Demotes format labels, not key material | None measured — real key material, whether unquoted or continued across an escaped line break, still fails this test and stays verdict-forcing |
| Self-referential constant | Demotes hits that are self-descriptive labels rather than credentials | One-directional by design — we have found no realistic credential the demotion could absorb; anything credential-shaped stays verdict-forcing |
| Non-runtime tooling declarations (Python) | Demotes tooling-only pins when a runtime manifest contradicts them | Narrow on purpose — any package the project *does* declare as a runtime dependency is never demoted, and a project with no parseable runtime manifest is unaffected |

**A candidate class was considered and explicitly rejected.** One held-out crypto library's own large known-answer test-vector directory produced tens of thousands of raw scanner hits — the obvious next candidate for the first rule above — but direct measurement showed **zero** of those hits were ever verdict-forcing to begin with. Adding a rule for it would have bought no measured precision while silently broadening an entire class of demotion. A regression test pins that this class stays excluded.

**A recall hole found while measuring, and fixed in the same pass.** A comment-detection bug suppressed a class of key material; fixed and regression-tested. This is a recall fix, not a precision one, and it is disclosed here because it was found by this work, not because this work caused it.

#### Adversarial re-check of the four rules — three held, one broke

The rules above were re-attacked with purpose-built fixtures rather than corpus repos.

- **The tooling-declaration rule's core promise was breakable, and was broken.** It promises that a package the project's own manifest declares as a runtime dependency is *never* demoted. The code that reads that declared runtime set had a bug: a common syntactic pattern in how a dependency is sometimes qualified (an optional-extras marker) could cause the reader to treat the declared list as ending early, silently dropping every package declared after that point — so a genuinely runtime-declared package could still be misclassified as tooling-only, and a real runtime CVE on it would be wrongly demoted. Fixed by parsing the declaration more carefully; the case that motivated the rule is unaffected. Pinned by a regression test.
- **The idiomatic-location rule held, at the wider cost already disclosed above** for its short-common-word member.
- **The PEM-boundary rule held** under attack across multiple key-material shapes — a service-account credential file, an escaped-continuation literal, and a raw unquoted key blob all stayed verdict-forcing; it demotes format labels, not key material.
- **The self-referential-constant rule held** at the boundary that matters — an ordinary credential-shaped value stays forcing. It does absorb one adjacent case involving a dictionary-word placeholder value; the qualifying shape is one no realistic credential takes.
- **The comment-detection fix works and carries one edge**, erring toward more content staying verdict-forcing rather than less, so it is left as-is.

**What this does NOT establish.** These four rules were written in response to the four failures this corpus produced — precisely the in-sample condition this corpus was built to expose. Regression runs across every other corpus in this benchmark (below) confirm the rules did not *cost* anything measurable elsewhere; they cannot prove the vocabulary is now complete. It is not: the next held-out set will find the next spelling.

#### Verification — the regression floor, re-measured after the fix

Every corpus in this benchmark was re-run after the fix landed. **Nothing was dropped anywhere** — total secret findings per repo are identical before and after on every corpus; only whether a hit forces the verdict moved.

> 🔥 **`corpus.heldout.json` is now BURNED.** Its numbers below are **in-sample**: the rules above were written *against the four failures these nine repos produced*, so re-scoring the same nine measures that the fix addressed what it was written for — nothing more. That is the identical shape of error the original 84%→100% in-sample clean-pass made, and the reason this corpus was built in the first place. **The held-out FCR of this scanner is currently unmeasured and unmeasurable** — it needs a second held-out set, frozen *after* these fixes landed, never run or inspected before freezing (see "Deferred" below). Until that set is run, the honest public claim is *"0% in-sample on every corpus in this benchmark; held-out generalization untested since the fixes landed."*

| Corpus | Metric | Before | After |
|---|---|---|---|
| `corpus.heldout.json` (n=9) — **BURNED, in-sample** | secret-scanner false-critical rate | 33.3% | **0%** |
| `corpus.heldout.json` — **BURNED, in-sample** | runtime-classified dep CVEs | 3 | **0** (reclassified advisory, none dropped) |
| `corpus.clean.json` (n=25) | secret-scanner false-critical rate | 0% | **0%** |
| `corpus.app.json` (n=20) | secret-scanner false-critical rate | 0% | **0%** |
| `corpus.vuln.json` (n=11) | free-tier detection rate | 63.6% (7/11) | **63.6% (7/11)** — unchanged |
| `corpus.vibe.json` (n=10) | production secrets | 2 | **2, both still verdict-forcing** |
| `corpus.messy.json` (n=3) | all counts | unchanged | **unchanged** |

#### Two further things this run exposes

1. **Severity handling differs by evidence class, and specifics are not published — but a secret-scanner misclassification still costs the same as any other secret hit, which makes secret-scanner precision the single highest-leverage trust variable in the product.** The consequence, visible here: one misclassified string in the wrong shape takes a mature, CVE-clean web framework straight to the harshest verdict NittiM can issue.
2. **Score and verdict visibly disagree on the report.** Two of the three secret-driven false alarms rendered a readiness score in the high 80s / low 90s beside the harshest verdict — the architecture working as specified (the score is the model's, the verdict is deterministically clamped), but a user reading a 92 next to "not safe to ship" is reading an internal contradiction the report does not currently explain. Disclosed here as a user-visible artifact, unresolved.

**Sample size, again without spin.** n=9. The 33.3% secret-driven FCR carries a 95% CI of 7.5%–70.1%; the point estimate is nearly meaningless as a *rate*. What the run establishes is not a number but an **existence proof**: the fix does not cover code shapes outside the ones it was written against, demonstrated three times on nine repos.

---

### Application corpus — 20 mature, deployed web applications

Source file: `benchmark/corpus.app.json` (n=20). **This corpus exists to close the single biggest caveat above:** the clean-library 0% was measured on library/CLI code only (predicate C6), leaving the **application** false-critical rate — the code type NittiM actually markets to — unmeasured. This set measures it.

**Purpose.** Measure the deterministic **application false-critical rate**: how often the free-tier scanners raise verdict-forcing hard evidence on a real, mature, widely-deployed open-source web app that should audit roughly green. These are not libraries and not intentionally-vulnerable training apps — they are production codebases across diverse stacks (Node/Next.js, Go, Python/Django, Ruby/Rails, PHP, Elixir, Clojure: Ghost, Strapi, n8n, cal.com, Directus, Medusa, Documenso, Formbricks, Twenty, Gitea, Mattermost, Saleor, Redash, PostHog, Discourse, Mastodon, Appwrite, Nextcloud, Plausible, Metabase).

#### The honest caveat that makes this corpus different

A full deployed application **legitimately carries more surface than a zero-dep library** — it ships a real dependency tree that, at any given HEAD, genuinely contains outdated packages with real CVEs. So unlike the library corpus, **a not-clean verdict here is not automatically a false alarm.** We therefore do not report one conflated "FCR." We bucket each hard-evidence repo by its **forcing driver** and report two numbers:

- **Secret-scanner false-critical rate** (the trust metric) — repos whose ONLY hard evidence is a committed-secret hit with no corroborating CVE. These are the candidate false positives.
- **Real runtime-CVE catches** (true positives, reported separately, **not** counted as false alarms) — repos with a real High/Critical CVE in a runtime dependency. A real app shipping a vulnerable dep version is the scanner working, not crying wolf.

A repo with a real CVE is attributed to the CVE bucket even if a (possibly false-positive) secret hit also fired — the CVE is the independently-legitimate forcing signal, so it takes attribution priority. (This forcing-driver bucketing was a deliberate fix; the prior code bucketed by raw secret hits, which would have mislabeled a real-CVE catch as a secret false positive.)

#### Current result (deterministic free tier — no Opus in the trust path)

| Metric | Value |
|---|---|
| **Secret-scanner false-critical rate** | **0%** (0/20) |
| Real runtime-CVE catches (true positives) | 3/20 — `medusajs/medusa` (form-data HIGH), `getredash/redash` (axios 0.27.2 HIGH + elliptic CRITICAL), `PostHog/posthog` (python-multipart HIGH) |
| Library-corpus FCR (unchanged, regression guard) | 0% (0/25) |
| Vuln-corpus recall (unchanged, regression guard) | 63.6% (7/11) |

> **The "true positives" row above has not survived later adjudication and is left standing only as the contemporaneous record.** Of the three, `elliptic` was adjudicated **FALSE** by both judges (the lockfile resolves the patched release) and `python-multipart` **FALSE** by the cross-family judge (the resolved version is above the advisory's affected range) — see the finding-level soundness table and the cross-family re-check in the Deep-tier section below. **The general lesson: a runtime-classified CVE is not a true positive until someone checks the resolved version and the dev/runtime split.**

The **0% secret false-critical rate** was reached by curating the four residual secret false positives an earlier run surfaced (which had put the raw, conflated number at 35%): n8n, cal.com, twenty, and appwrite, each a documented example/template credential rather than a real one. Each was fixed with a **general precision rule keyed on the path- or content-class of the hit, never a per-repo allowlist** — the same rule fires on any repo with that class of non-production credential. *(The exact rule definitions are part of the scanner implementation in the private product repo and are summarized rather than reproduced here.)*

In three of the four cases (cal.com, twenty, appwrite) the finding is **still surfaced as evidence** — it loses only its *verdict-forcing* teeth, exactly as a test-fixture secret does. The n8n hit is a documented placeholder stand-in rather than a credential at all, so it is dropped outright.

#### Caveats (read before the 0%)

1. **Free-tier deterministic only.** This measures the two scanners, not the Opus audit. The paid-tier model FCR on applications is a separate, still-open number.
2. **Small n, in-sample.** n=20, and the four curation rules were written in response to these same 20 repos — a mild overfit. A held-out application set (apps not in these 20, audited after the fix) is an outstanding v1 item, mirroring the library-corpus commitment.
3. **The curation lowers recall on a real edge case.** Treating example/template/documentation credentials as non-verdict-forcing means a genuinely real secret committed in a location the rules classify as non-production would be surfaced but not force the verdict. This is the accepted precision/recall trade-off; the regression guards confirm it did not cost the one path-based catch the vuln corpus depends on (railsgoat's seed-file password stays verdict-forcing).
4. **The 3 CVE catches are true positives, not the headline.** Do not read "15% of apps flagged" as a false-alarm rate — 15% is the raw conflated number; the false-alarm rate is the 0% secret line, and the 3 CVE repos are correct catches on real outdated deps.

---

### Deep-tier application FCR (Opus audit over the 20-app corpus)

The section above measures the **free deterministic tier**. This one measures the **paid Deep tier (Opus 4.8)** on the same 20 apps — the number that actually matters, since the Deep audit is the product customers buy. Each not-green app's crit/high findings were then **independently adjudicated against OSV + the repo's real manifest/lockfile** via a two-adjudicator adversarial pass (each verdict attacked in both directions), and each not-green verdict was judged for proportionality. Full run + adjudication artifacts live in the reports drawer (`reports/fis/`), not the repo.

**Sample.** 20 mature, deployed OSS web applications (Medusa, Redash, PostHog, Metabase, +16). A small, non-random convenience sample of *well-maintained* projects — **not** representative of the younger, thinner-reviewed, vibe-coded code NittiM actually markets to. Read every number below as a **lower bound** on real-world error, not a point estimate.

**Headline.** 16/20 (80%) passed green; 4/20 (20%) not-green. Of the four: three dep-CVE-driven (Medusa, Redash, PostHog — all `not_safe`), one model-only (Metabase, `high_risk`) that **flipped to green on an independent re-audit**.

#### (a) Strict false-critical rate — 0%, but nearly definitional

Zero uncorroborated criticals across all 20 (no critical on model opinion alone). But this is close to definitional given the architecture: the guardrail only lets a critical stand when a deterministic scanner fires, so the model is *structurally* prevented from inventing one. This metric confirms the guardrail holds — it does **not** measure whether the corroborated criticals are *right*, which is the worse story below.

#### (b) Finding-level soundness — 2 SOUND / 2 OVERSTATED / 1 FALSE

Five crit/high dependency findings drove the four not-green verdicts, each adjudicated twice adversarially against OSV + the real lockfile:

| # | Finding | Sev | Verdict |
|---|---|---|---|
| 1 | Medusa — `form-data` CRLF (GHSA-hmw2-7cc7-3qxx) | high | **OVERSTATED** — real advisory, but transitive-only, conditional (the advisory *author* calls it integrity-only, "Moderate-defensible"), impact not reachable as stated |
| 2 | Redash — `elliptic` private-key-extraction | critical | **FALSE** — the lockfile pins `elliptic@6.6.1`, the **patched** release; NittiM read the `^6.6.0` spec string and ignored the lock; also never imported |
| 3 | Redash — `axios@0.27.2` prototype-pollution / DoS | high | **SOUND** — real, pinned, runtime, reachable |
| 4 | Redash — DOMPurify / Bootstrap XSS | high | **SOUND** (risk-sound) — reachable stored-XSS on user-rendered markdown; but the *specific* GHSAs cited are misattributed to the actual sanitizer config |
| 5 | PostHog — `python-multipart` / `requests` / `pytest` | high | **OVERSTATED** — real runtime `python-multipart` DoS, but NittiM fabricated version strings absent from the lockfile and counted **test-only** `pytest` as production risk |

**Only two of five findings were clean; one was flatly false; two materially overstated.** The recurring, concrete, fixable failure mode: **NittiM reads declared version ranges / spec strings instead of the resolved lockfile, and repeats an advisory's headline severity without reading the advisory's own scoping caveats.** Two-of-five is the most damning number in this section and must not be buried under the 0% headline.

#### (c) Verdict-level honest read — over-escalation dominates

All four not-green verdicts adjudicated **OVER_ESCALATED**; none pointed at an app-specific, reachable defect in the app's *own* code — all penalize the normal background rate of dependency drift every mature app carries.

- **Metabase** — `high_risk` on **zero** deterministic evidence; model-only, **flipped to green** on re-audit. A clean soft false alarm.
- **Medusa** — `not_safe` on a single transitive, OVERSTATED `form-data` CVE. Over-escalated.
- **PostHog** — `not_safe` on ubiquitous Python CVEs including a **test-only** `pytest` — the scanner did not separate production runtime from dev tooling here.
- **Redash** — *the most defensible of the four:* its `not_safe` is corroborated by **two SOUND, reachable HIGH findings** (axios prototype-pollution + stored-XSS via DOMPurify on user-rendered markdown), which a security-conservative operator would reasonably treat as ship-relevant. The caveat is that the verdict was *reached the wrong way* — the **FALSE `elliptic` critical deterministically forced the verdict every run**, so NittiM is "right for a wrong reason," and the reasoning chain it prints is not trustworthy even where the bottom line is.

**The tension between (a) and (c) is the real story:** the strict model-FCR is 0%, yet the not-green *verdict* precision on mature apps is poor. The pure model behaved; the **deterministic dependency-CVE layer over-escalated** — at the time of this run it forced the harshest verdict on any runtime CVE match, including a false one. The user-visible failure is a **scanner/guardrail-tuning problem, not a hallucinating-model problem** — which is the more fixable of the two. ("Soft false alarm" is itself a position: on mature deployed software dep drift is expected and non-blocking, but a conservative operator could call a real low-reachability CVE *useful noise*.)

#### (d) The single most important caveat

**A not-green from NittiM on a mature codebase is far more likely to be dependency drift than a genuine ship-blocker**, and the burden is on NittiM — not the reviewer — to prove reachability before it is treated as one. The zero-uncorroborated-criticals headline must **not** be read as "NittiM's criticals are trustworthy": one of the two corroborated criticals in this run was flatly false.

#### Actionable defects this measurement surfaced

1. **Resolve the installed version, not the declared range. — ✅ FIXED (commit `8fe8bbb`).** The FALSE `elliptic` critical was a real dep-scanner bug: the scanner matched the vulnerability database against the *declared version range* (which resolves pessimistically to `6.6.0`, vulnerable) instead of the *installed version recorded in the lockfile* (`6.6.1`, patched). The scanner now resolves installed versions from the repo's lockfile across the package managers the corpus uses. Verified end-to-end: elliptic now resolves `6.6.1`, the false critical is gone, and real resolved-version CVEs the range-fallback missed (lodash, `@babel`, documenso) now surface — recall *up*, false-critical source *closed*.
2. **Separate dev/test deps from runtime for Python. — ✅ FIXED (commit `8fe8bbb`).** `pytest` was counted toward a production-risk finding; the Python path now classifies dev/test-only dependencies as advisory rather than production risk, matching the Node dev flag.
3. **Verdict-floor tuning — don't force the harshest verdict on ubiquitous transitive dep drift. — ✅ FIXED (commit `5a883d3`).** The deterministic floor was split by severity: a genuinely critical runtime vulnerability still forces `not_safe` (full teeth), while a high-severity-only runtime vulnerability now forces an elevated-but-lesser verdict — never green (so recall holds), never the trust-destroying `not_safe` for background dependency drift. This loosens a *deterministic floor*, not the *model's ceiling* — the model still cannot rate a repo safer than the deterministic evidence allows, so model-over-rating remains impossible. A companion fix aligned the GitHub Action's default CI gate with this split, so a high-severity runtime vulnerability still **blocks CI** by default — the report label is now honest without trading away gate protection. The safety invariants are regression-tested. The motivating over-escalations (medusa, posthog, redash) now resolve at the deterministic layer to an elevated verdict rather than `not_safe`. *(The exact invariant table and severity cutoffs are part of the product's server-side guardrail and are described here at behavior level only.)*

> **Note on the FALSE finding above:** with defect #1 fixed, the `elliptic` critical (Finding 2) no longer fires — the single flatly-false critical in this run is resolved at the source, not suppressed. The finding-level soundness table reflects the *pre-fix* run that motivated the fix; a re-run would show 2 SOUND / 2 OVERSTATED / 0 FALSE.

#### Eval-doctrine limitation — cross-family re-check added

> **Update: the five findings behind the four not-green verdicts above were re-adjudicated by a
> judge from a rival model family** — a version-pinned, dated GPT snapshot (`gpt-5.5-2026-04-23`)
> — against independently fetched evidence (live OSV advisory records + manifest/lockfile
> excerpts at the exact audited commits). Offline benchmark only; production user code never
> routes to a third-party model.
>
> | # | Finding | Original (Opus-family) | Cross-family (GPT) | Read |
> |---|---|---|---|---|
> | 1 | Medusa form-data | OVERSTATED | **SOUND** | GPT: resolved version is below the fix version → version-applicable; original judge weighed reachability. Judgment-shaped disagreement. |
> | 2 | Redash elliptic | FALSE | **FALSE** | **Agreement on the headline correction** — the lockfile resolves the patched release. Not friendly-judge bias. |
> | 3 | Redash axios | SOUND | **OVERSTATED** | GPT: one of the two cited advisories doesn't apply to the resolved version. Partly a harness artifact (original findings' full prose wasn't preserved verbatim, so the cited-advisory pair was partly reconstructed). |
> | 4 | Redash dompurify/bootstrap | SOUND (risk-sound) | **OVERSTATED** | GPT: one of the two libraries resolves to a fixed version; only the other is advisory-affected. Sharpens the original's own "misattributed" note. |
> | 5 | PostHog python-multipart | OVERSTATED | **FALSE** | **The friendly-judge effect, caught:** the resolved version is above the advisory's affected range. The original judge had called the underlying issue "real." |
>
> Net: the rival-family judge was *stricter* with NittiM (2 FALSE / 2 OVERSTATED / 1 SOUND vs
> the original's 1 / 2 / 2), agreed on the elliptic correction, and exposed one case (#5) where
> the same-family judge had been too generous. Honest caveats of the re-run: single judge, single
> pass, evidence is excerpt-based not full manifests, and the disagreements (#1, #3, #4) are
> precisely reachability-vs-version-applicability judgment calls — a third-family judge with
> disagreements routed to a human is the designed next step, not yet built.

_The original disclosure below is kept as written — the limitation is real; the re-check above
narrows it, it doesn't erase it._

#### (original) Eval-doctrine limitation (disclosed, not solved)

Every adjudication was performed by an **Opus-family model — the same family as the model under test** — because NittiM ships Opus 4.8 and no strictly-stronger independent judge exists. This violates the "never let a model grade its own family" ideal; we state it plainly. Two partial mitigations, neither sufficient: (1) each verdict was decided against **OSV + real lockfile ground truth**, not model opinion — shared-family blind spots bite hardest on judgment, least on "did OSV return this advisory for this resolved version"; (2) the two most consequential corrections (Finding 2 → FALSE, Finding 5 → OVERSTATED) turned on lockfile facts an independent linter would also catch. **Residual risk:** a "reachable/exploitable" blind spot shared by both sides passes undetected. A truly independent, at-least-as-strong judge is the correct fix and is unavailable; discount these soundness numbers for same-family bias accordingly.

#### Verdict instability (a finding in its own right)

Metabase's model-only `high_risk` **flipped to green on re-audit**, and the re-audit did not perfectly reproduce the benchmark's verdicts. All figures above are from a **single pass per app**, so they understate run-to-run variance; the true instability rate is unmeasured. **v1 commitment:** report **verdict variance across N≥3 runs** per audit, surfacing the distribution rather than one sampled verdict, so a flip like Metabase's shows as declared variance instead of a silent coin-flip. Until then, treat any single Deep-tier verdict near a band boundary as potentially unstable.

#### Verdict variance, measured — delivered, and narrower than the disclaimer implied

The commitment above is delivered as a harness capability: an N-runs mode audits every repo N times and publishes the full distribution — verdicts, executive-score min/mean/max/σ, per-headline-score spread, and an explicit flip list — alongside the single-pass headline, which is always labelled as run 1 of N. A first measurement was run at **N=3 on a 3-repo corpus** chosen for contrast: the historical flipper (Metabase), a clean library, and a self-declared AI-built app.

| Metric | Value |
|---|---|
| Runs per repo | 3 (9 deep audits total) |
| **Verdict flips** (not identical across runs) | **0/3 (0%)** · 95% CI 0%–70.8% |
| Mean executive-score σ | 1.79 |
| Max executive-score spread on one repo | 9 points (Metabase, 70→79) |

**What this does and does not settle — three readings, in descending order of how much we like them.**

1. **The verdict was stable within a sitting.** Nine audits, three repos, zero flips. On this evidence a Deep-tier verdict is not a coin-flip.
2. **But the sample contains no near-boundary repo, which is the exact condition the disclaimer was about.** All three verdict distributions are degenerate, so the run says nothing about a repo sitting at a band edge — it measures stability where stability was never in doubt. 0/3 with a 95% upper bound of 70.8% is a floor on our ignorance, not a stability guarantee.
3. **Metabase did not flip within the sitting, but it is still the across-time flipper.** It was included *because* it is the historical flipper, and across three fresh runs it returned the same mid-band verdict all three times — which is one of the two values it has already produced across separate sittings, not a new one. **The same repo, on the same engine lineage, has returned two different verdicts across sittings while being perfectly stable within one.** Within-sitting stability is not across-time stability, and this run measures only the first. (Metabase also audits at a small fraction of its file count — it is a very large repo — so its verdict is a stable read of an unstable sample, which is its own caveat.)
4. **The scores move more than the verdicts do.** Metabase's production-readiness score and privacy score each swing double digits across three identical inputs while the verdict a customer sees holds. A single-run headline score is a sample, and the report does not currently say so anywhere the reader will look.

**Status of the commitment:** the *capability* is delivered and the *disclaimer is retired* — run-to-run variance is now a measured, published quantity rather than an admitted unknown. The **full-corpus** version (N≥3 across the 20-app corpus, which is what would actually bound the flip rate) is deferred on cost; see "Deferred" below.

---

### Vibe-coded corpus — 10 self-declared AI-built applications

Source file: [`corpus.vibe.json`](corpus.vibe.json) (n=10, frozen 2026-08-11). **This corpus exists because every other corpus in this benchmark is the wrong code.** The clean, application, vulnerable, and messy corpora are all mature, professionally-maintained open-source software or purpose-built training apps. None of them are the young, AI-scaffolded, often first-time-builder applications NittiM is actually marketed to audit — a gap the deep-tier application section above names explicitly. This set is the first corpus made of that code.

#### Definition and selection predicates

Each entry: (1) **self-declares AI-tool authorship** in its own README, badge, description, or topics, with the exact quoted evidence recorded per entry (Lovable, v0.dev, Claude Code, Bolt.new, or a bare "vibe coded" claim); (2) is **application-shaped** — a runnable app with real code, not a library/framework/CLI (that is the clean corpus's job) and not an awesome-list, prompt collection, or AI-tool-building tool (a large number of these were filtered out during search); (3) has more than 30 files in its git tree; (4) was created within roughly the last 18 months; (5) is publicly fetchable, not a fork, not archived; (6) does not appear in any of the other four corpora. All verified live via the GitHub API on freeze date.

Language skew (9 TypeScript, 1 JavaScript) is **disclosed, not curated away** — it reflects what the AI-scaffolding tools in this corpus actually emit (React/Next.js scaffolds), not a selection choice.

#### The caveat that defines this corpus: the label is self-declared authorship, not security ground truth

> **There is no vulnerability ground truth here and no expected verdict.** "Self-declared AI-built" is an **authorship** label taken from each repo's own README — not a security label, not independently verified authorship (a README badge is a claim, not provenance), and not a random sample of AI-built code. The self-selection is severe in a specific direction: a repo that *keeps* the boilerplate README, is public, and has accumulated stars is by construction more finished and more looked-at than the median AI-generated project. **Do not compute or cite an FCR, a recall rate, or a catch/miss count against this corpus.** Those metrics require a known-clean or known-vulnerable label, which this corpus does not have and does not claim.

What the corpus **does** support: (a) qualitative field-behaviour reads — are verdicts, scores, and findings *proportionate* on genuinely AI-scaffolded code, the same way the messy corpus checks proportionality on large accreted legacy code; and (b) a validation surface for the **AI-code-confidence signal** specifically, since every entry carries an independent out-of-band authorship label the model's score can be checked against without NittiM having produced that label itself.

#### Status: field-mode only, free tier run, deep tier deferred

**Free tier (deterministic scanners, no Opus), all 10 repos:**

| Metric | Value |
|---|---|
| Repos completed / errored | 10 / 0 |
| Signal distribution | 8 hard evidence · 2 no deterministic findings |
| Forcing-driver distribution | **8 dependency-CVE · 0 secret · 2 none** |
| Mean hard-evidence findings per repo | 9.1 |

The shape of that result is itself the finding worth recording: on ten AI-scaffolded apps the deterministic signal is **almost entirely dependency drift**, not committed credentials — only two repos produced any production-classified secret hit at all, and in both the CVE load took forcing attribution anyway. Read with the caveat the application corpus already established: on real deployed code a dep-CVE hit is often the scanner working rather than crying wolf, and none of this is a false-alarm rate because there is no label to be false against.

**Deep tier: 1 of 10.** Only one repo has been Opus-audited so far (it was carried into the N=3 variance run above): the same elevated verdict on all three runs, dependency-CVE forcing driver, one high finding located in the app's own code rather than its dependencies. The specific defect class is withheld here pending upstream disclosure to the repo's maintainer. That is a proportionate, located, non-label finding on a self-declared AI build, which is encouraging and is **n=1**.

On the AI-code-confidence axis, the only comparison currently available: that one repo scored in the high-90s across three runs against a README that says it was built entirely with an AI coding agent, while the nine mature held-out libraries scored single digits — with one unexplained outlier in the low 40s. One positive example and nine negatives is an anecdote pointing the right way, not a validated classifier. The full deep-tier pass over this corpus is deferred on cost; see "Deferred" below.

---

### Vulnerable corpus — 11 intentionally-vulnerable apps

**Purpose.** Measure recall: how often NittiM catches code *known* to be vulnerable. A tool with zero false positives and high false negatives is dangerous to rely on.

#### Inclusion predicates (all must hold)

| # | Predicate | Rationale |
|---|---|---|
| V1 | The repo's README/project page identifies it as an intentionally-vulnerable training app with known, deliberate vulnerabilities | Ground-truth label. |
| V2 | OWASP-affiliated or cited in published security-training literature | Independent corroboration beyond the author's own claim. |
| V3 | Web application | Application-shaped, to match NittiM's primary use case. |
| V4 | Publicly accessible on GitHub at freeze date | Required for the harness to fetch and re-audit. |

#### Recall is **verdict-level**, not finding-level — and that matters

The reported "100% detection" means **100% of the 11 apps were verdicted `not_safe`** (none verdicted green). It does **not** mean NittiM located each app's documented vulnerabilities. Two honest deficiencies:

1. **Some criticals are label recognition, not code analysis.** Inspecting the raw findings, several titles restate the README rather than point at code: dvna critical #8 ("Application is intentionally vulnerable and must never run in production"), NodeGoat's "(intentional A1)/(intentional A2 demo)" tags, DSVW #1 ("intentionally and comprehensively vulnerable by design"), dvws-node #1 ("intentionally vulnerable across the entire OWASP API/Web Service Top 10"). Strip the README and repo name and those findings would not exist. A tool that reads "OWASP Juice Shop" and says `not_safe` has perfect recall here and unknown utility on an unlabeled real-world repo. v1 commitment: a **finding-level spot-check** — for ≥3 corpus entries, manually verify that ≥2 of NittiM's criticals correspond to a documented vulnerability at a specific code location independent of name/README signal — published as part of the methodology.

2. **Verdict-level recall here is parity with a trivial aggregator.** A one-line rule over the scanner outputs ("`not_safe` if any Trivy high CVE OR any Semgrep ERROR OR any Gitleaks secret") also catches all 11. So the 100% does not demonstrate detection *superiority* over the scanners — only that NittiM does not return a false *safe* on grossly broken apps.

#### Finding-level spot-check — the instrument is built, the measurement is not made

The commitment above was to turn the label-recognition caveat into a number. The **instrument now exists** and is designed to be honestly blinded: for each critical it fetches the cited file at the audited commit, replaces every repository and organization name with a neutral placeholder in **both** the finding text and the fetched code, **withholds any README or markdown the finding cited** (documentation is label signal, not a code location), and asks a single question — *does this finding identify a concrete issue at a real code location, independent of any repository-name or README signal?* Three verdicts: **LOCATED** (names a specific place in code, consistent with the fetched code — survives name/README removal), **LABEL_ONLY** (restates what the project *is*, or states generic risk with no code location — would not exist without the label), **NOT_FOUND** (cites a location, but the fetched code doesn't contain what it describes). The judge is a version-pinned dated rival-lab model, never the family under test, per the never-let-a-model-grade-its-own-family rule. Offline benchmark only.

> **The blinding is real but not airtight, and the limit is named before any number is published on it.** Verified with a constructed test case: the audited repo's own organization and name are replaced in both the finding text and the fetched code, including common naming-variant forms, and cited markdown is withheld. Two residual leaks survive by construction: an organization name that is **not** the audited repo's own owner is not a redaction target, and **fetched source can carry its own identifying strings** (a copyright header naming the project's author, for instance) that the redaction doesn't touch. A judge inclined to guess the project can therefore sometimes still guess it. This limit didn't affect the run below (it judged zero evidence-bearing findings, so the redaction was never load-bearing), but it must be closed before a LOCATED rate measured with this instrument is published as evidence.

**The result is that there is no result. The v1 commitment (≥3 repos with ≥2 located criticals) is NOT MET — and not because NittiM failed it, because the measurement could not be taken.**

| Metric | Value |
|---|---|
| Repos checked · criticals checked | 1 · 1 |
| **Evidence-bearing criticals judged (headline denominator)** | **0** |
| LOCATED / LABEL_ONLY / NOT_FOUND | 0 / 0 / 0 |
| Criticals citing a code path | 0/1 |
| Criticals citing no location at all | 1 |

**The blocker, stated plainly: the archived run artifacts that predate this instrument recorded finding *titles* only, with no cited evidence.** With no cited file, there is nothing to fetch, nothing to blind, and nothing a finding could fairly be judged against — so title-only findings are reported separately and **excluded from the headline** rather than scored as failures. The single critical available for this run came from the application-corpus artifact (the `elliptic` finding already adjudicated FALSE elsewhere in this document), not the vulnerable-corpus run whose findings the original caveat actually names.

**What this kills, what it confirms, and what it leaves standing:**

- It **confirms a citation-hygiene defect independent of any judge**: 0 of 1 criticals cited a code path and 1 of 1 cited no location whatsoever. That is measured arithmetic on the artifact, not a model's opinion, and it is the weaker cousin of the original caveat — a finding that cites nothing cannot be acted on by a reader regardless of whether the underlying issue is real.
- It **kills nothing.** The label-recognition caveat — that several criticals restate the README rather than locate code, and would not exist if the repo name were stripped — stands **entirely undischarged**. Nothing in this run supports it and nothing refutes it.
- It leaves a clean, cheap next step: re-run the deep-tier vulnerable corpus capturing full finding evidence, and re-point the spot-check at that artifact. The instrument is the expensive half and it is done; the corpus is 11 repos.

#### The juice-shop-ctf removal — disclosed in full

The original run included a 12th entry, `bkimminich/juice-shop-ctf`, which NittiM passed — the sole miss, giving **91.7% (11/12)** verdict recall. It was then removed on the grounds that it is a CTF challenge-*generation* tool, not a vulnerable application (it matches the vuln-set exclusion rule for CTF tooling). **We disclose plainly that this removal happened after the run, not before it.** The removal rationale is defensible on its merits, but because it was decided post-result it cannot claim the same anti-cherry-picking standing as a pre-registered exclusion. Therefore we publish **both** numbers: **100% on the re-frozen 11-app corpus**, and **91.7% (11/12) under a strict pre-registration standard** where the 12th entry stays in and is argued to be mislabeled. v1 commitment: the corpus is frozen via lock file before the run, so this ambiguity cannot recur.

#### Language comparability gap

The clean FCR was measured only on C7-supported languages (JS/TS/Python/Go/Rust). The recall corpus includes **Ruby** (railsgoat) and **PHP** (DVWA, OWASP/Vulnerable-Web-Application) — 3 of 11 entries, in languages NittiM does not formally support. **Precision and recall are therefore measured on partly different language sets and are not strictly comparable.** DVWA in particular is the corpus's weakest evidence: unsupported language, analysis truncated at 220 files, **zero** specific findings, "detected" only by a collapsed holistic score. We flag it as such rather than letting it pad the recall count.

#### Score-label correction

The separation metric was previously labeled "executive-risk score" with clean repos at 89.3 and vulnerable at 30.3 — which, read as a *risk* score, inverts the intended meaning (it would call clean code high-risk). It is a **readiness/safety score (higher = safer)** and is relabeled accordingly. The ~59-point separation is real, but two caveats attach: (a) the two populations are maximally contrasted *by construction* (zero-dep audited libraries vs. intentionally-broken apps), so the separation says little about discriminating a real vibe-coded app with 2–3 embedded issues from clean code — the actual use case, absent from this benchmark; and (b) we now report the **min/max** of each population, not only the mean, so readers can see the distribution around the verdict boundary (one vulnerable-corpus entry, Juice Shop, scores close to the safe band — detection there depends on a verdict cutoff that is a product-design choice, not an empirically validated threshold, disclosed as such).

---

## Comparison protocol — NittiM vs the field (reproducible)

The full 11-app comparison table and honest reading live in the published summary
(`reports/fis/fis-benchmark-summary-2026-06-30.md`, §4). This is the exact protocol so a
skeptic can re-derive every number.

**Ground truth.** The deterministic slices (committed secrets, dependency CVEs) are checked
against independent engines, not NittiM's own scanners. The vulnerable corpus is labeled by the
upstream projects themselves (intentionally-vulnerable OWASP/training apps). The clean corpus is
"no *known* issue" (OSV + two secret scanners agree), which is weaker than "provably correct" — a
clean pass means no corroborated finding, not a proof of safety.

**Tools & exact invocation** (per app, over the same shallow-cloned commit; SHAs recorded per run):

| Tool | Version | Command | Counted as |
|---|---|---|---|
| Gitleaks | 8.30.1 | `gitleaks detect --source <dir> --no-git --report-format json` | # findings = secrets |
| Trivy | 0.72.0 | `trivy fs --scanners vuln,secret,misconfig --format json <dir>` | Σ Vulnerabilities / Secrets / FAIL-Misconfigurations |
| Semgrep | 1.168.0 | `semgrep --config auto --json <dir>` (metrics on — auto-config requires it) | total results / ERROR-severity |
| NittiM | Opus 4.8 (`AUDIT_MODEL`) | `runAudit` (production engine) | verdict + critical-severity findings |

**Known measurement artifacts** (disclosed in the table footnotes): Trivy `vuln=0` on apps with no
lockfile / no installed `node_modules` (dvna, juice-shop) *understates* Trivy — the dep-CVE surface
is unseen, not absent. Trivy `vuln=0` on apps with no dependency manifest (DSVW, OWASP/VWA) is
genuine. Semgrep MUST be run with metrics on; `--metrics off` silently disables `--config auto` and
falls back to per-repo rulesets, making counts non-comparable across rows (a bug we hit and fixed).

**The honest headline:** NittiM's 100% verdict-level recall on this corpus is **parity with a one-line
threshold rule** over the three scanners (every app carries ≥1 scanner signal), *not* detection
superiority. NittiM's distinct contribution is the single prioritized verdict plus non-scanner
categories (architecture, AI-code-confidence), not vulnerability coverage.

---

## Open caveats — published, not hidden

- Application FCR is unmeasured: the 0% false-critical rate is on library/framework/CLI code only (predicate C6 excludes web apps), yet web apps are NittiM's marketed target class. The clean-application false-positive rate is an open, unproven quantity.
- Small sample: n=25 clean (95% CI for the true FCR is 0%-13.8%, Clopper-Pearson exact) and n=11 vulnerable. v0 is a pilot adequate for gross-error detection, not for tight rate bounds. A single false critical would move the observed FCR to 4%.
- Minimal-dependency bias: 72% of the clean corpus is near-zero-dep, removing the complex-dependency reasoning condition where a model is most likely to false-positive. FCR does not bound precision on dependency-heavy repos. **— PARTIALLY CLOSED:** the held-out corpus adds six non-minimal-dependency libraries. FCR is still not reported by dependency-weight bucket on the original 25.
- Ecosystem concentration: JS/TS is 52% of the clean set and leans on one maintainer's utility family plus the Pallets cluster; the earlier 'five distinct lineages' diversity claim is retracted as overstated. **— PARTIALLY CLOSED:** the held-out set caps JS/TS at 22% across nine unrelated maintainer orgs with zero overlap with the original clusters. The original 25 are unchanged.
- No security-primitive, crypto, auth, or async/network libraries in the clean set, so the model's likeliest false-positive trigger (pattern-matching security keywords in correct code) is untested. **— CLOSED:** the held-out set is stocked with JOSE/JWT/crypto/async-network code. The model raised **zero** criticals across all nine. The trigger is now tested and clean at n=9 (true rate could still be ~34%).
- No timestamped pre-registration for v0: the anti-cherry-pick guarantee rests on the author's assertion plus retained-problematic-repos evidence, not an independently auditable dated artifact. Lock-file pre-registration is a v1 commitment. **— STILL OPEN for every v0 corpus; DELIVERED for `corpus.heldout2`,** which was frozen on 2026-08-19 and published to the dated public tag `corpus-heldout2-frozen-2026-08-19` (`fis-benchmark`), with a resolved commit SHA per entry in `corpus.heldout2.lock.json`, **before NittiM was run against a single one of its repos**. For that corpus the anti-cherry-pick guarantee is independently timestamped rather than asserted. The held-out and vibe corpora were frozen before their first run, but neither was published to a dated public tag beforehand, so the same "asserted, not cryptographically established" standing applies to them too.
- In-sample tuning: the 84%->100% clean-pass came from fixing the scanner against the same 25 repos now being scored (commit a669e04); a held-out validation set to confirm generalization is outstanding. **— DELIVERED, AND IT FAILED.** On 9 never-before-run repos the clean-pass rate is 55.6%, with a 33.3% secret-scanner-driven false-critical rate at both free and deep tier — and, after adversarial re-check, all four not-green verdicts turned out to be candidate false alarms. The pure model false-critical rate stayed at 0%. The precision fix generalized to the code shapes it was written against and not to shapes spelled differently. See "Held-out validation" above.
- Verdict variance was unmeasured: all deep-tier figures were single-pass, and Metabase's flip to green on re-audit showed the instability is real. **— DELIVERED:** an N-runs mode publishes the full distribution; a first measurement at N=3 on 3 repos returned **0/3 flips** (95% CI 0%–70.8%), mean executive-score σ 1.79, max spread 9 points. Two honest limits: no repo in that sample sits near a band boundary (the exact condition at issue), and within-sitting stability is not across-time stability — Metabase was perfectly stable across these three runs yet has returned different verdicts across earlier sittings. Headline scores vary materially more than verdicts do. See "Verdict variance, measured" above.
- juice-shop-ctf was removed from the vuln set after the run cured the only miss; the honest recall is 100% on the re-frozen 11-app corpus and 91.7% (11/12) under strict pre-registration. The removal rationale (CTF tooling, not a vulnerable app) is defensible but was decided post-result.
- Recall is verdict-level, not finding-level: several NittiM criticals restate the README/repo name rather than locate code (dvna #8, NodeGoat intentional tags, DSVW #1, dvws-node #1); a finding-level spot-check is owed. **— INSTRUMENT DELIVERED, MEASUREMENT NOT MET.** The spot-check harness blinds repo/org names in both finding text and code, withholds cited documentation, fetches the cited file at the audited commit, and judges LOCATED / LABEL_ONLY / NOT_FOUND with a pinned rival-family judge. The headline denominator came back **0**: the archived artifacts predate finding-evidence capture and recorded titles without evidence strings, so there was nothing to fetch. The caveat stands entirely undischarged. The one measurable thing — citation hygiene, computed without a judge — is bad: 0/1 criticals cited a code path, 1/1 cited no location at all.
- Verdict recall equals a trivial threshold aggregator over Trivy/Semgrep/Gitleaks, so 100% detection is not evidence of detection superiority over the scanners.
- Language gap: clean FCR is on JS/TS/Python/Go/Rust; recall adds Ruby and PHP (unsupported), so precision and recall are measured on partly different language sets and are not strictly comparable. Java, C/C++, C# are absent entirely. **— PARTIALLY ADDRESSED:** dependency-CVE scanning has since been extended to Go, Rust, and Maven/Gradle (declared-only for Java, which has no universal lockfile). The clean-pass numbers reported earlier in this document are unchanged and are **not** re-measured under the newer coverage — the corpus's Go and Rust entries at the time those numbers were produced carried zero dependency-scan evidence, so their clean-pass verdicts rested on secret-scan + model-audit alone. Ruby and PHP remain genuinely unsupported for dependency CVEs.
- DVWA is the weakest single data point: unsupported language, analysis truncated at 220 files, zero specific findings, detection only via a collapsed holistic score whose threshold is a product-design choice, not an empirically validated cutoff.
- Comparison-table inputs are uneven: Trivy vuln=0 on dvna and Juice Shop are artifacts of missing lockfiles/node_modules (real dep-CVE surface unseen), and Semgrep used a different ruleset per repo (68 to 1,059 rules), so Semgrep counts are not comparable across rows.
- Tuned-Semgrep not benchmarked: the 'noise' framing is only fair against unfocused --config-style runs; NittiM vs. a focused Semgrep config on signal-to-noise is untested.
- No grey-zone corpus: the benchmark contrasts maximally-clean libraries against maximally-broken training apps. NittiM's ability to discriminate a real web app with 2-3 embedded issues from clean code (the actual use case) is unmeasured. **— PARTIALLY ADDRESSED:** the vibe-coded corpus (n=10 self-declared AI-built applications) is the first corpus made of NittiM's actual target class. It carries no vulnerability ground truth by design, so it cannot fully close this caveat — a grey-zone corpus needs *labels*, and a self-declared authorship badge is not one. The free tier has been run over all 10 (8/10 hard evidence, all dependency-CVE-driven, zero secret-driven forcing); the deep tier has covered 1 of 10.
- No adversarial examples: clean code with security-pattern-like naming, or a real credential formatted for plausible deniability, are not represented; adversarial robustness is untested.
- HEAD-based snapshots, not SHA-pinned in v0: upstream drift means re-runs are distinct snapshots, not replications. The runner records commitSha per run; a committed lock file is a v1 pre-condition for citability.

### Deferred — what is not done, and what it would cost

Everything in this table is deferred on **API cost alone**, not on doubt about its value — each row is a bounded model-inference spend at current list prices, tracked against an internal benchmark budget (exact per-run spends are internal). Each one is named because we would rather publish the gap than the impression that it isn't there.

| Deferred item | Scope | Why it matters |
|---|---|---|
| **Score-separation re-run** | Deep tier over the 25 clean + 11 vulnerable repos, 36 audits | The ~59-point separation (clean 89.3 / vuln 30.3) predates the lockfile-resolution, dev/test-classification, and guardrail-severity fixes, and is still reported as a mean without the min/max the score-label correction promised. Until re-run, the separation figure describes an engine that no longer ships. |
| **Full vibe-corpus deep tier** | Deep tier over all 10 self-declared AI-built apps, single pass (ideally N=3) | This is the only corpus made of NittiM's actual target class, and 9 of its 10 repos have never seen the model. Every deep-tier behavioural claim in this document rests on mature, well-maintained code. |
| Full-corpus verdict variance | N≥3 across the 20-app corpus, 60 audits | The N=3 run bounds nothing (0/3 flips, CI 0%–70.8%) and contained no near-boundary repo, which is the condition the instability caveat is actually about. |
| Finding-level spot-check, for real | Re-run the deep vulnerable corpus capturing full finding evidence, 11 audits, then re-point the existing spot-check harness | The cheapest outstanding item and the one that discharges the oldest caveat. The instrument is built; only the evidence-bearing artifact is missing. |
| Held-out re-test after the secret-scanner fix — **now blocking, not optional** | A *second* held-out set, frozen after the fixes above landed | The current held-out corpus **was spent** generalization-testing the fixes it motivated, so its post-fix 0% is an in-sample number and the product currently has **no held-out precision measurement at all**. A fix is only validated by a set it has never seen. Bias the new set toward the classes the fixes traded away. |

_One measurement-hygiene note carried over from the internal record: the benchmark runner records duration but not per-audit token or cost fields, so spend attribution is currently read from the API provider's usage console rather than run artifacts. Fixing that (cost on the result row) is itself an outstanding item._

