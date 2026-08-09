# FIS Benchmark — Methodology

_The reproducible spec for the FIS Benchmark: how each corpus is built, how the comparison is
run, what ground truth is, and every known weakness. The point is that a hostile reader can
re-derive the corpus, verify any entry, or flag one that fails a predicate. Results + narrative
live in `reports/fis/fis-benchmark-summary-*.md`; this file is the method._

---

## Benchmark Corpus — Selection Criteria (hardened)

This section documents the predicates used to construct each corpus and absorbs the substantive objections raised in adversarial review. Every predicate is checkable against public data (GitHub API, OSV, npm/PyPI/pkg.go.dev/crates.io release history, and a second independent secret scanner). The goal is that a hostile reader can re-derive the corpus, verify each entry, or flag one that fails a predicate.

Source files: `benchmark/corpus.clean.json` (n=25 repos), `benchmark/corpus.vuln.json` (n=11 repos).

> **Sample size, stated up front and without spin.** Both corpora are **small**: n=25 clean, n=11 vulnerable. This is a **v0 pilot**, sufficient to detect *gross* false-alarm and false-negative behavior, **not** to bound either rate tightly. At n=25 with zero false criticals, the exact (Clopper-Pearson) 95% confidence interval for the true False Critical Rate is **0%–13.8%** — i.e. a single false critical would move the observed rate to 4%, and the data cannot rule out a true rate as high as ~14%. The headline "0% FCR" is a point estimate on a small pilot, not a precision claim. The v1 target is ≥100 clean repos (which would put the 95% upper bound near 3% at the same confidence).

---

### Clean corpus — 25 mature libraries

**Purpose.** Measure the False Critical Rate (FCR): how often the FIS auditor raises a Critical on code with no committed secrets and no known dependency CVEs. The metric isolates the model's own false-positive tendency from the deterministic scanners.

> **Scope limitation — read this before the 0% number.** This corpus's 0% FCR is measured on **library / framework / CLI code only.** Predicate C6 below excludes web applications from this clean set by design — yet web applications are the exact code type FIS markets itself to audit. The application false-critical rate was therefore measured **separately**, on the 20-app application corpus documented later in this file (2026-07-01) — with its own, different failure modes and caveats. Never read the library 0% as an application number: the headline must always be *"0% false-critical rate on mature library code; the application axis is measured separately, below."*

#### Inclusion predicates (all must hold at corpus-freeze date)

| # | Predicate | Rationale |
|---|---|---|
| C1 | GitHub stars ≥ 1,000 | Proxy for community review surface; a latent issue would more likely have surfaced publicly. |
| C2 | First public release ≥ 4 years before corpus construction | Eliminates projects whose security posture is unproven. |
| C3 | Last commit within 12 months | Confirms active maintenance. |
| C4 | Zero open critical/high CVEs in OSV at freeze date (verified via `osv-scanner` over each manifest) | A repo with an open high CVE is not clean; including it would conflate a correct FIS warning with a false positive. |
| C5 | Zero committed secrets per **two independent scanners** (FIS's own scanner **and** Gitleaks) | Breaks the circularity of testing the tool against its own secret-scan output — see note below. |
| C6 | Library, framework, or CLI tool — **not** a web application | Scopes the FCR to library-style code; application attack surface is the domain of the vuln set. **This is a scope boundary, acknowledged as a limitation, not a strength** (see scope box above). |
| C7 | Language in {JavaScript, TypeScript, Python, Go, Rust} | The languages FIS's analysis engine currently supports. |

**C5 circularity fix.** The original C5 checked "no secrets" using only FIS's own scanner — meaning a secret type FIS systematically misses would mark a repo "clean," and if the model also misses it, the benchmark records a false 0% from a shared blind spot. C5 now requires **both** FIS's scanner and **Gitleaks** (an independent engine, already in the comparison table) to agree there are no committed secrets. This does not eliminate the possibility of a secret type both miss, but it removes the single-tool self-reference.

#### Selection bias: disclosed, not denied

Three concentration biases survive in the v0 corpus and we name them rather than bury them:

1. **Minimal-dependency bias removes the hardest false-positive condition.** 18 of 25 (72%) entries are flagged `minimal_dep: true`. The original rationale — "isolate the model from the dep-CVE guardrail" — is methodologically backwards for measuring real-world precision: a model's *hardest* false-positive challenge is a complex, dependency-heavy repo where it must reason "library X has security associations but is used only in a non-production path here." That reasoning failure is invisible in a near-zero-dep corpus. **The 0% FCR therefore does not bound the false-positive rate for repos with non-trivial dependency trees.** v1 commitment: a `complex-dep clean` sub-corpus (e.g. sequelize, passport, socket.io, httpx) and **FCR reported separately by `minimal_dep` bucket**.

2. **Ecosystem / maintainer concentration.** JavaScript/TypeScript is 13 of 25 (52%), and that slice leans on a single maintainer's zero-dep utility family (the sindresorhus packages) plus the Pallets cluster (flask/click/jinja) in Python. The earlier draft's "five distinct maintainer lineages" claim **overstated diversity and is retracted**: three repos from one author in one micro-utility niche is not lineage diversity. v1 commitment: cap any single ecosystem at ≤35% of the corpus and replace several pure-JS utilities with different code-type categories (an ORM, an auth library, a serialization library, a networking library).

3. **No security-primitive or async/network libraries.** None of the 25 entries implement authentication, password hashing, JWT/crypto, or session management — i.e. the exact surface where a model that pattern-matches "sees `hash`/`salt`/`secret` → Critical" would false-positive. Their absence means the FCR has **not** been tested against the model's most likely false-positive failure mode. v1 commitment: add CVE-clean security libraries (argon2, Python `cryptography`, `golang.org/x/crypto`) and re-run.

#### Anti-cherry-picking guarantee — and its honest limit

The selection question was fixed before any audit ran: *"What are the most widely-recognized, most-downloaded, security-mature libraries in each supported language?"* — a question with public answers (download stats, star rankings) independent of FIS. **No repo was added because FIS passed it; no repo was removed from the clean set because FIS flagged it.** The strongest supporting evidence: when four repos initially produced scanner false alarms (ky's test-fixture token; lodash and zod devDependency CVEs; flask's example-app pinned requirements), the response was to **fix the scanner's precision** (commit `a669e04`) and keep all four repos — not to drop the inconvenient data.

**The honest limit of this guarantee:** there is **no timestamped pre-registration** for v0. No public commit of `corpus.clean.json` predates the first audit run; the integrity claim rests on the author's assertion plus the retained-problematic-repos evidence, which is suggestive but not proof. v1 commitment: push a `corpus.clean.lock.json` (resolved commit SHA per repo) to a dated public release/tag **before** running, so timestamps are independently auditable. Until then, treat the guarantee as "asserted and circumstantially supported," not "cryptographically established."

#### A note on the 84% → 100% clean-pass figure

The improvement from 84% to 100% clean-pass was produced by **fixing the scanner in response to the benchmark** (commit `a669e04`). Each fix was individually correct, but the 100% is **in-sample**: the scanner was tuned to the same 25 repos it is now scored on, which is a mild form of overfitting. The 100% should be read as *"post-fix, in-sample."* A held-out validation set of 5–10 repos not in the original 25, audited after the fix, is an **outstanding v1 item** to confirm the fix generalized rather than solving only the four observed cases.

---

### Application corpus — 20 mature, deployed web applications

Source file: `benchmark/corpus.app.json` (n=20). **This corpus exists to close the single biggest caveat above:** the clean-library 0% was measured on library/CLI code only (predicate C6), leaving the **application** false-critical rate — the code type FIS actually markets to — unmeasured. This set measures it.

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

**Sample.** 20 mature, deployed OSS web applications (Medusa, Redash, PostHog, Metabase, +16). A small, non-random convenience sample of *well-maintained* projects — **not** representative of the younger, thinner-reviewed, vibe-coded code FIS actually markets to. Read every number below as a **lower bound** on real-world error, not a point estimate.

**Headline.** 16/20 (80%) passed green; 4/20 (20%) not-green. Of the four: three dep-CVE-driven (Medusa, Redash, PostHog — all `not_safe`), one model-only (Metabase, `high_risk`) that **flipped to green on an independent re-audit**.

#### (a) Strict false-critical rate — 0%, but nearly definitional

Zero uncorroborated criticals across all 20 (no critical on model opinion alone). But this is close to definitional given the architecture: the guardrail only lets a critical stand when a deterministic scanner fires, so the model is *structurally* prevented from inventing one. This metric confirms the guardrail holds — it does **not** measure whether the corroborated criticals are *right*, which is the worse story below.

#### (b) Finding-level soundness — 2 SOUND / 2 OVERSTATED / 1 FALSE

Five crit/high dependency findings drove the four not-green verdicts, each adjudicated twice adversarially against OSV + the real lockfile:

| # | Finding | Sev | Verdict |
|---|---|---|---|
| 1 | Medusa — `form-data` CRLF (GHSA-hmw2-7cc7-3qxx) | high | **OVERSTATED** — real advisory, but transitive-only, conditional (the advisory *author* calls it integrity-only, "Moderate-defensible"), impact not reachable as stated |
| 2 | Redash — `elliptic` private-key-extraction | critical | **FALSE** — the lockfile pins `elliptic@6.6.1`, the **patched** release; FIS read the `^6.6.0` spec string and ignored the lock; also never imported |
| 3 | Redash — `axios@0.27.2` prototype-pollution / DoS | high | **SOUND** — real, pinned, runtime, reachable |
| 4 | Redash — DOMPurify / Bootstrap XSS | high | **SOUND** (risk-sound) — reachable stored-XSS on user-rendered markdown; but the *specific* GHSAs cited are misattributed to the actual sanitizer config |
| 5 | PostHog — `python-multipart` / `requests` / `pytest` | high | **OVERSTATED** — real runtime `python-multipart` DoS, but FIS fabricated version strings absent from the lockfile and counted **test-only** `pytest` as production risk |

**Only two of five findings were clean; one was flatly false; two materially overstated.** The recurring, concrete, fixable failure mode: **FIS reads declared version ranges / spec strings instead of the resolved lockfile, and repeats an advisory's headline severity without reading the advisory's own scoping caveats.** Two-of-five is the most damning number in this section and must not be buried under the 0% headline.

#### (c) Verdict-level honest read — over-escalation dominates

All four not-green verdicts adjudicated **OVER_ESCALATED**; none pointed at an app-specific, reachable defect in the app's *own* code — all penalize the normal background rate of dependency drift every mature app carries.

- **Metabase** — `high_risk` on **zero** deterministic evidence; model-only, **flipped to green** on re-audit. A clean soft false alarm.
- **Medusa** — `not_safe` on a single transitive, OVERSTATED `form-data` CVE. Over-escalated.
- **PostHog** — `not_safe` on ubiquitous Python CVEs including a **test-only** `pytest` — the scanner did not separate production runtime from dev tooling here.
- **Redash** — *the most defensible of the four:* its `not_safe` is corroborated by **two SOUND, reachable HIGH findings** (axios prototype-pollution + stored-XSS via DOMPurify on user-rendered markdown), which a security-conservative operator would reasonably treat as ship-relevant. The caveat is that the verdict was *reached the wrong way* — the **FALSE `elliptic` critical deterministically forced the verdict every run**, so FIS is "right for a wrong reason," and the reasoning chain it prints is not trustworthy even where the bottom line is.

**The tension between (a) and (c) is the real story:** the strict model-FCR is 0%, yet the not-green *verdict* precision on mature apps is poor. The pure model behaved; the **deterministic dependency-CVE layer over-escalated** — at the time of this run it forced the harshest verdict on any runtime CVE match, including a false one. The user-visible failure is a **scanner/guardrail-tuning problem, not a hallucinating-model problem** — which is the more fixable of the two. ("Soft false alarm" is itself a position: on mature deployed software dep drift is expected and non-blocking, but a conservative operator could call a real low-reachability CVE *useful noise*.)

#### (d) The single most important caveat

**A not-green from FIS on a mature codebase is far more likely to be dependency drift than a genuine ship-blocker**, and the burden is on FIS — not the reviewer — to prove reachability before it is treated as one. The zero-uncorroborated-criticals headline must **not** be read as "FIS's criticals are trustworthy": one of the two corroborated criticals in this run was flatly false.

#### Actionable defects this measurement surfaced

1. **Resolve the installed version, not the declared range. — ✅ FIXED (commit `8fe8bbb`).** The FALSE `elliptic` critical was a real dep-scanner bug: the scanner matched the vulnerability database against the *declared version range* (which resolves pessimistically to `6.6.0`, vulnerable) instead of the *installed version recorded in the lockfile* (`6.6.1`, patched). The scanner now resolves installed versions from the repo's lockfile across the package managers the corpus uses. Verified end-to-end: elliptic now resolves `6.6.1`, the false critical is gone, and real resolved-version CVEs the range-fallback missed (lodash, `@babel`, documenso) now surface — recall *up*, false-critical source *closed*.
2. **Separate dev/test deps from runtime for Python. — ✅ FIXED (commit `8fe8bbb`).** `pytest` was counted toward a production-risk finding; the Python path now classifies dev/test-only dependencies as advisory rather than production risk, matching the Node dev flag.
3. **Verdict-floor tuning — don't force the harshest verdict on ubiquitous transitive dep drift. — ✅ FIXED (commit `5a883d3`).** The deterministic floor was split by severity: a genuinely critical runtime vulnerability still forces `not_safe` (full teeth), while a high-severity-only runtime vulnerability now forces an elevated-but-lesser verdict — never green (so recall holds), never the trust-destroying `not_safe` for background dependency drift. This loosens a *deterministic floor*, not the *model's ceiling* — the model still cannot rate a repo safer than the deterministic evidence allows, so model-over-rating remains impossible. A companion fix aligned the GitHub Action's default CI gate with this split, so a high-severity runtime vulnerability still **blocks CI** by default — the report label is now honest without trading away gate protection. The safety invariants are regression-tested. The motivating over-escalations (medusa, posthog, redash) now resolve at the deterministic layer to an elevated verdict rather than `not_safe`. *(The exact invariant table and severity cutoffs are part of the product's server-side guardrail and are described here at behavior level only.)*

> **Note on the FALSE finding above:** with defect #1 fixed, the `elliptic` critical (Finding 2) no longer fires — the single flatly-false critical in this run is resolved at the source, not suppressed. The finding-level soundness table reflects the *pre-fix* run that motivated the fix; a re-run would show 2 SOUND / 2 OVERSTATED / 0 FALSE.

#### Eval-doctrine limitation (disclosed, not solved)

Every adjudication was performed by an **Opus-family model — the same family as the model under test** — because FIS ships Opus 4.8 and no strictly-stronger independent judge exists. This violates the "never let a model grade its own family" ideal; we state it plainly. Two partial mitigations, neither sufficient: (1) each verdict was decided against **OSV + real lockfile ground truth**, not model opinion — shared-family blind spots bite hardest on judgment, least on "did OSV return this advisory for this resolved version"; (2) the two most consequential corrections (Finding 2 → FALSE, Finding 5 → OVERSTATED) turned on lockfile facts an independent linter would also catch. **Residual risk:** a "reachable/exploitable" blind spot shared by both sides passes undetected. A truly independent, at-least-as-strong judge is the correct fix and is unavailable; discount these soundness numbers for same-family bias accordingly.

#### Verdict instability (a finding in its own right)

Metabase's model-only `high_risk` **flipped to green on re-audit**, and the re-audit did not perfectly reproduce the benchmark's verdicts. All figures above are from a **single pass per app**, so they understate run-to-run variance; the true instability rate is unmeasured. **v1 commitment:** report **verdict variance across N≥3 runs** per audit, surfacing the distribution rather than one sampled verdict, so a flip like Metabase's shows as declared variance instead of a silent coin-flip. Until then, treat any single Deep-tier verdict near a band boundary as potentially unstable.

---

### Vulnerable corpus — 11 intentionally-vulnerable apps

**Purpose.** Measure recall: how often FIS catches code *known* to be vulnerable. A tool with zero false positives and high false negatives is dangerous to rely on.

#### Inclusion predicates (all must hold)

| # | Predicate | Rationale |
|---|---|---|
| V1 | The repo's README/project page identifies it as an intentionally-vulnerable training app with known, deliberate vulnerabilities | Ground-truth label. |
| V2 | OWASP-affiliated or cited in published security-training literature | Independent corroboration beyond the author's own claim. |
| V3 | Web application | Application-shaped, to match FIS's primary use case. |
| V4 | Publicly accessible on GitHub at freeze date | Required for the harness to fetch and re-audit. |

#### Recall is **verdict-level**, not finding-level — and that matters

The reported "100% detection" means **100% of the 11 apps were verdicted `not_safe`** (none verdicted green). It does **not** mean FIS located each app's documented vulnerabilities. Two honest deficiencies:

1. **Some criticals are label recognition, not code analysis.** Inspecting the raw findings, several titles restate the README rather than point at code: dvna critical #8 ("Application is intentionally vulnerable and must never run in production"), NodeGoat's "(intentional A1)/(intentional A2 demo)" tags, DSVW #1 ("intentionally and comprehensively vulnerable by design"), dvws-node #1 ("intentionally vulnerable across the entire OWASP API/Web Service Top 10"). Strip the README and repo name and those findings would not exist. A tool that reads "OWASP Juice Shop" and says `not_safe` has perfect recall here and unknown utility on an unlabeled real-world repo. v1 commitment: a **finding-level spot-check** — for ≥3 corpus entries, manually verify that ≥2 of FIS's criticals correspond to a documented vulnerability at a specific code location independent of name/README signal — published as part of the methodology.

2. **Verdict-level recall here is parity with a trivial aggregator.** A one-line rule over the scanner outputs ("`not_safe` if any Trivy high CVE OR any Semgrep ERROR OR any Gitleaks secret") also catches all 11. So the 100% does not demonstrate detection *superiority* over the scanners — only that FIS does not return a false *safe* on grossly broken apps.

#### The juice-shop-ctf removal — disclosed in full

The original run included a 12th entry, `bkimminich/juice-shop-ctf`, which FIS passed — the sole miss, giving **91.7% (11/12)** verdict recall. It was then removed on the grounds that it is a CTF challenge-*generation* tool, not a vulnerable application (it matches the vuln-set exclusion rule for CTF tooling). **We disclose plainly that this removal happened after the run, not before it.** The removal rationale is defensible on its merits, but because it was decided post-result it cannot claim the same anti-cherry-picking standing as a pre-registered exclusion. Therefore we publish **both** numbers: **100% on the re-frozen 11-app corpus**, and **91.7% (11/12) under a strict pre-registration standard** where the 12th entry stays in and is argued to be mislabeled. v1 commitment: the corpus is frozen via lock file before the run, so this ambiguity cannot recur.

#### Language comparability gap

The clean FCR was measured only on C7-supported languages (JS/TS/Python/Go/Rust). The recall corpus includes **Ruby** (railsgoat) and **PHP** (DVWA, OWASP/Vulnerable-Web-Application) — 3 of 11 entries, in languages FIS does not formally support. **Precision and recall are therefore measured on partly different language sets and are not strictly comparable.** DVWA in particular is the corpus's weakest evidence: unsupported language, analysis truncated at 220 files, **zero** specific findings, "detected" only by a collapsed holistic score. We flag it as such rather than letting it pad the recall count.

#### Score-label correction

The separation metric was previously labeled "executive-risk score" with clean repos at 89.3 and vulnerable at 30.3 — which, read as a *risk* score, inverts the intended meaning (it would call clean code high-risk). It is a **readiness/safety score (higher = safer)** and is relabeled accordingly. The ~59-point separation is real, but two caveats attach: (a) the two populations are maximally contrasted *by construction* (zero-dep audited libraries vs. intentionally-broken apps), so the separation says little about discriminating a real vibe-coded app with 2–3 embedded issues from clean code — the actual use case, absent from this benchmark; and (b) we now report the **min/max** of each population, not only the mean, so readers can see the distribution around the verdict boundary (one vulnerable-corpus entry, Juice Shop, scores close to the safe band — detection there depends on a verdict cutoff that is a product-design choice, not an empirically validated threshold, disclosed as such).

---

## Comparison protocol — FIS vs the field (reproducible)

The full 11-app comparison table and honest reading live in the published summary
(`reports/fis/fis-benchmark-summary-2026-06-30.md`, §4). This is the exact protocol so a
skeptic can re-derive every number.

**Ground truth.** The deterministic slices (committed secrets, dependency CVEs) are checked
against independent engines, not FIS's own scanners. The vulnerable corpus is labeled by the
upstream projects themselves (intentionally-vulnerable OWASP/training apps). The clean corpus is
"no *known* issue" (OSV + two secret scanners agree), which is weaker than "provably correct" — a
clean pass means no corroborated finding, not a proof of safety.

**Tools & exact invocation** (per app, over the same shallow-cloned commit; SHAs recorded per run):

| Tool | Version | Command | Counted as |
|---|---|---|---|
| Gitleaks | 8.30.1 | `gitleaks detect --source <dir> --no-git --report-format json` | # findings = secrets |
| Trivy | 0.72.0 | `trivy fs --scanners vuln,secret,misconfig --format json <dir>` | Σ Vulnerabilities / Secrets / FAIL-Misconfigurations |
| Semgrep | 1.168.0 | `semgrep --config auto --json <dir>` (metrics on — auto-config requires it) | total results / ERROR-severity |
| FIS | Opus 4.8 (`AUDIT_MODEL`) | `runAudit` (production engine) | verdict + critical-severity findings |

**Known measurement artifacts** (disclosed in the table footnotes): Trivy `vuln=0` on apps with no
lockfile / no installed `node_modules` (dvna, juice-shop) *understates* Trivy — the dep-CVE surface
is unseen, not absent. Trivy `vuln=0` on apps with no dependency manifest (DSVW, OWASP/VWA) is
genuine. Semgrep MUST be run with metrics on; `--metrics off` silently disables `--config auto` and
falls back to per-repo rulesets, making counts non-comparable across rows (a bug we hit and fixed).

**The honest headline:** FIS's 100% verdict-level recall on this corpus is **parity with a one-line
threshold rule** over the three scanners (every app carries ≥1 scanner signal), *not* detection
superiority. FIS's distinct contribution is the single prioritized verdict plus non-scanner
categories (architecture, AI-code-confidence), not vulnerability coverage.

---

## Open caveats — published, not hidden

- Application FCR is unmeasured: the 0% false-critical rate is on library/framework/CLI code only (predicate C6 excludes web apps), yet web apps are FIS's marketed target class. The clean-application false-positive rate is an open, unproven quantity.
- Small sample: n=25 clean (95% CI for the true FCR is 0%-13.8%, Clopper-Pearson exact) and n=11 vulnerable. v0 is a pilot adequate for gross-error detection, not for tight rate bounds. A single false critical would move the observed FCR to 4%.
- Minimal-dependency bias: 72% of the clean corpus is near-zero-dep, removing the complex-dependency reasoning condition where a model is most likely to false-positive. FCR does not bound precision on dependency-heavy repos.
- Ecosystem concentration: JS/TS is 52% of the clean set and leans on one maintainer's utility family plus the Pallets cluster; the earlier 'five distinct lineages' diversity claim is retracted as overstated.
- No security-primitive, crypto, auth, or async/network libraries in the clean set, so the model's likeliest false-positive trigger (pattern-matching security keywords in correct code) is untested.
- No timestamped pre-registration for v0: the anti-cherry-pick guarantee rests on the author's assertion plus retained-problematic-repos evidence, not an independently auditable dated artifact. Lock-file pre-registration is a v1 commitment.
- In-sample tuning: the 84%->100% clean-pass came from fixing the scanner against the same 25 repos now being scored (commit a669e04); a held-out validation set to confirm generalization is outstanding.
- juice-shop-ctf was removed from the vuln set after the run cured the only miss; the honest recall is 100% on the re-frozen 11-app corpus and 91.7% (11/12) under strict pre-registration. The removal rationale (CTF tooling, not a vulnerable app) is defensible but was decided post-result.
- Recall is verdict-level, not finding-level: several FIS criticals restate the README/repo name rather than locate code (dvna #8, NodeGoat intentional tags, DSVW #1, dvws-node #1); a finding-level spot-check is owed.
- Verdict recall equals a trivial threshold aggregator over Trivy/Semgrep/Gitleaks, so 100% detection is not evidence of detection superiority over the scanners.
- Language gap: clean FCR is on JS/TS/Python/Go/Rust; recall adds Ruby and PHP (unsupported), so precision and recall are measured on partly different language sets and are not strictly comparable. Java, C/C++, C# are absent entirely.
- DVWA is the weakest single data point: unsupported language, analysis truncated at 220 files, zero specific findings, detection only via a collapsed holistic score whose threshold is a product-design choice, not an empirically validated cutoff.
- Comparison-table inputs are uneven: Trivy vuln=0 on dvna and Juice Shop are artifacts of missing lockfiles/node_modules (real dep-CVE surface unseen), and Semgrep used a different ruleset per repo (68 to 1,059 rules), so Semgrep counts are not comparable across rows.
- Tuned-Semgrep not benchmarked: the 'noise' framing is only fair against unfocused --config-style runs; FIS vs. a focused Semgrep config on signal-to-noise is untested.
- No grey-zone corpus: the benchmark contrasts maximally-clean libraries against maximally-broken training apps. FIS's ability to discriminate a real web app with 2-3 embedded issues from clean code (the actual use case) is unmeasured.
- No adversarial examples: clean code with security-pattern-like naming, or a real credential formatted for plausible deniability, are not represented; adversarial robustness is untested.
- HEAD-based snapshots, not SHA-pinned in v0: upstream drift means re-runs are distinct snapshots, not replications. The runner records commitSha per run; a committed lock file is a v1 pre-condition for citability.

