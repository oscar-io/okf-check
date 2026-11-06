# okf-check concepts brief: third review

* **Target:** `/Users/oscar/code/github/oscar-io/okf-check/concepts_brief.md`: Overview, Decisions 1–46, Still open, Goals vs. Non-Goals and Staging (brief lines 5–17 and 205–499). The 2026-11-04 review record was read for context only and not reviewed.
* **Date:** 2026-11-06
* **Lenses:** design, codebase-details, zoom-out, comprehension, implementation-readiness, structural (one instance each).
* **Question:** is the brief ready to hand to `/egm:egm_draft`?

## Verdict

**Not ready to draft. It is four answers away.** The second review's distribution and exporter-mechanics blockers are resolved. So are most of its Majors. Four problems remain that a drafter cannot work around, and the author must answer each one:

1. **Copied files in `references/` still refuse whole exports** (decisions 6, 14, 17, 34, 38). All six lenses found this. Decision 34 fixes only the index findings, which are `[house]` and do not block at `integrity` anyway.
2. **The concept predicate still hides the commonest agent mistakes, and four decisions contradict each other about it** (5, 8, 38, 39). The second review's blocker 5 has no answering decision, although the Overview says every blocker is answered.
3. **Decision 33 changes io-insights' documents to suit the checker and breaks io-insights' own reader and gate.** It also covers only 440 of about 755 `/`-prefixed resources, and no stage builds the `--fix` that Stage 2 depends on.
4. **v1 schedules `publish_okf.py`, which a stable io-insights decision has gated off** (23, 32, Stage 3).

Eighteen Majors follow. Most of them are one-sentence decisions or contradictions a drafter can settle with the owner while drafting. The ones that change scope or cost are M1, M7, M13 and M18: the version pin, the description codes, the Staging total and the okfcli trial.

| Severity | Count |
|---|---|
| Blocker | 4 |
| Major | 18 |
| Nit | 14 |

Lens agreement: B1 was raised by all 6 lenses, B2 by 5, B3 by 5 and B4 by 3. Where lenses disagreed on severity, the reasoning is given under the finding.

## Resolution of the second review's findings

A finding the author declined counts as resolved when the brief records the decision and its reason.

| # | Second-review finding | Status | Answered by | What remains |
|---|---|---|---|---|
| 1 | Exporter's `kb/`/`sections/` refused at `house` | Resolved | 14, 17, 34 | Decision 34 brings a new bug (M5) |
| 2 | Typed files in `references/` | **Not resolved** | 6, 34 | B1 |
| 3 | One `block` level vs the publish policy | Resolved | 14 | `integrity` is not equivalent to `FATAL_CODES` (M10) |
| 4 | `docs/` root: resources dangle, root concepts drop out | Partly | 4, 33 | Dropping root concepts is a recorded choice, so that half is resolved. The resources half is B3 |
| 5 | Concept-by-`type` hides mistakes | **Not resolved** | none | B2 |
| 6 | Stdin protocol | Resolved | 23 | Replaced by a staging directory. New problems in M3 and B4 |
| 7 | Wheel distribution | Resolved | 25, 26 | No wheels. New problems in M1 and M2 |
| 8 | Exporter switch names no payoff | Resolved (declined, with reason) | 17 | none |
| 9 | Discovery undefined | Partly | 4 | Skip set and nested roots are settled. The claim of parity with `bundle_roots` is false (M12). The checked tree and the no-bundle case are open (M11) |
| 10 | Index link rule undefined | Partly | 35 | Decision 35 misdescribes okf-drift (M6) |
| 11 | "Zero differences" exam | Resolved | 36 | The ledger format is open (M16) |
| 12 | yz-docs facts; Stage 1 depends on later work | Partly | 19, 20 | Gate location and the Stage 1 actor fix are resolved. The first-run list is still incomplete (M8) |
| 13 | Versioning, skew, rollback | Partly | 37 | Pins conflict and "absent means latest" cannot be implemented (M1). No JSON-schema policy |
| 14 | YAML profile not implementable; strict loader vs lenient reader | Partly | 18, 28, 46 | The profile table is scheduled for Stage 0. The Python loader change is not named. Which parser defines "unparseable" is open (M4). The v4 `WithUniqueKeys` claim is unverified |
| 15 | `[spec]` has no test | Partly | 38 | A test now exists and `bad-actor` moved. The list fails the brief's own test (B2, M10) |
| 16 | Code decisions contradict each other | Partly | 39 | The table exists only as a Stage 0 task. Renames, `--strict` and the `unreadable` label are open (M10) |
| 17 | Config lookup | Resolved | 40 | Residual Nit N2 |
| 18 | v2 log check reverses a stable record | Resolved (recorded, with reason) | 13, 31 | The amendment is unscheduled (M13, N5) |
| 19 | No CLI surface | Partly | 41 | Named-file mode is settled. The default target and the meaning of a path argument are open (M11) |
| 20 | Output lacks location and inputs | Partly | 42 | Line numbers and severity are added, and label-first is declined with a reason. The input header and `skipped[]` were dropped without a reason (M14) |
| 21 | Run-event redaction | Resolved | 43 | The event mapping is open (M15) |
| 22 | Agent exam not runnable | Partly | 44 | Seeds and the pass criterion are defined. The protocol is not, and the untyped seed produces no finding (M14, B2) |
| 23 | House vocabulary; spec examples fail | Resolved (declined, with reason) | 45 | none |
| 24 | Brief contradicts itself on open, in scope and staged | Partly | Staging, Still open | M17 |
| 25 | Scale vs pace; no cost line | Partly | 32, Staging | Estimates exist. Work is uncosted and there is no buffer (M13) |
| Nit | Undefined terms | Partly | 4 | The `# OKF v0.2` comment is resolved. The others are in N9 |
| Nit | Decision 13 reads as v1 | Resolved by "later wins" | 29, 31, Goals | Amendment timing (N5) |
| Nit | Decision 20's fix wrong for non-root indexes | **Not resolved, now Major** | none | M9 |
| Nit | `--actor` raises the trust tier | Resolved | 19 | none |
| Nit | Decision 16 narrower than the code | Not resolved | none | N6 |
| Nit | `fail/<rule-code>/` allows one case | Not resolved | none | M16 |
| Nit | Duplicate date groups undecided | Not resolved, now triggered | none | M8 |
| Nit | `missing-index` is a producer-side rule | Not resolved | none | N9 |
| Nit | Decision 30 hides the folder guides | Not resolved, no reason recorded | none | N9 |
| Nit | okf-drift retirement date | Partly | 22, Staging "Later" | N7 |
| Nit | Speed baseline measures nothing reported | Not resolved | none | N8 |
| Nit | Document structure | Not resolved | none | N13 |

## Blockers

### B1. A file the exporter only copied can refuse the whole export
- **Lenses:** all 6.
- **Decisions:** 4, 6, 9, 14, 17, 34, 38.
- **Rule:** a gate must not hold the producer responsible for content it is not allowed to change.
- **Evidence:**
  - **Copies keep their names and front matter.** `embed_references` copies every cited file whole to `references/<repo path>` (`pipeline/export_okf.py:12, 857`). It filters only absolute or climbing paths, secret names, missing or oversized files and symlinks (`export_okf.py:836-856`). There is no filter for `.md` files or reserved names, so a cited `index.md`, `log.md` or typed `.md` arrives verbatim.
  - **Typed copies are concepts.** Decision 6 says so (brief line 228) and claims they "are covered by decision 34". Decision 34 (line 356) only adds `index.md` files. Every field rule still runs on the other repository's front matter.
  - **One foreign field refuses the export.** At the exporter's `--block=integrity`:
    - a bare actor is `[integrity]` `bad-actor` (decision 38). yz-docs has 327 bare `generated.by` values and 2 bare `verified.by` values;
    - a cited `docs/Implementation/log.md` carries `## 2026-09-28 (6)` and 17 more suffixed headings, each a `[spec]` finding;
    - a cited index with front matter is a `[spec]` §8 finding. yz-docs' root `index.md` has `type: index` (lines 1-4);
    - unparseable YAML is a `[spec]` finding.
  - **Nested bundles.** A cited `index.md` that carries `okf_version` starts a nested bundle inside `references/` (decision 4's exception).
  - **Collisions.** A cited `docs/index.md` lands on the same path as the `references/docs/index.md` that decision 34 must generate. `_prune_references` deletes every file under `references/` that this run did not embed, generated indexes included (`export_okf.py:881-898`).
- **What it reverses without saying so:** io-insights excludes `references/` on purpose, everywhere:
  - `NON_OKF_PREFIXES` and its rationale (`core/okf.py:65-73`): "validating it as one of our concepts reports findings against a file we only copied";
  - `write_bundle` (`export_okf.py:1023-1029`);
  - `read_bundle` (`core/okf_read.py:78-79`);
  - `publish_okf._existing_concept_paths` (`pipeline/publish_okf.py:62-68`).

  The checker and io-insights' own reader would disagree about which files in one bundle are concepts.
- **Unverified:** how often real exports cite `.md` files, or `index.md` and `log.md` in particular. Decision 6 itself treats typed copies as normal.
- **Recommendation:** choose one, and record it against okf.py's rationale:
  - field and reserved-file rules skip `references/` at a bundle root (index rules may still apply), recorded as a deviation from §6.3 the way decision 8 records one;
  - the exporter blocks only on findings in files it authored, and findings under `references/` are advisory;
  - accept the refusal, and state what the user sees.

### B2. The concept predicate hides the commonest agent mistakes, and decisions 5, 8, 38 and 39 contradict each other
- **Lenses:** comprehension (Blocker), structural, codebase-details, design (Nit), implementation-readiness.
- **Decisions:** 5, 7, 8, 9, 28, 38, 39, 42, 44.
- **Rule:** a checker whose users are agents must report the mistakes agents make most often.
- **What an agent gets at `block: house`:**
  - **No output.** It writes a new document with `tpye:`, `Type:` or `type: ""`, not yet linked from an index. That is no concept (decision 5), so no `unlisted-concept` fires. The run exits 0 with nothing printed. The same happens in a directory with no `index.md`, because `missing-index` counts only concepts.
  - **One typo hides everything else.** Because field rules run only on concepts, a missing `type` also hides a bad `generated.at`, a bad actor or a duplicate source id.
  - **The wrong repair.** If the document is linked, only `listed-non-concept` fires. Deleting the link clears it, and decision 30 prescribes exactly that fix for the READMEs.
  - **No defined code.** An unclosed block is "no front matter" today (`core/okf.py:190-192`). A BOM fails `startswith("---\n")` (`okf.py:187`). No decision defines either.
- **The contradictions:**
  - **Is unparseable front matter checked?** Decision 9 makes it `[spec]`. Decision 28 routes duplicate keys there, and decision 42's sample prints `unparseable-front-matter`. But under decision 5 a block that does not parse has no readable `type`, so the file is ordinary and never checked.
  - **`type` passes decision 38's own test.** The test is "a spec MUST or REQUIRED sentence". Spec §11 rule 2 requires "a non-empty `type` field" in every front-matter block (`/tmp/okf0.2-spec:742`), and codebase-details cites §4.1's REQUIRED at line 165. Yet decision 39 retires `missing-type`, and decision 8 admits only to weakening rule 1. Decision 39 is later than 38, so "later wins" retires the code. It does not explain why the brief's own test fails, so a drafter cannot tell which to trust.
  - **Two codes for one condition.** `unreadable` (decision 39, no label) is okf-drift's code for front matter that fails to parse (`okf-drift/src/bundle.ts:100-107`). That is the same condition as `unparseable-front-matter`.
  - **The exporter loses a check.** Today an exported concept without a `type` is fatal (`missing-type` is in `FATAL_CODES`, `okf.py:104-106, 122-125`). Under decisions 5 and 39 it becomes an ordinary file and is published.
  - **The agent exam cannot test it.** Decision 44 seeds "a new document without a `type`", which produces no finding or one that deleting the link clears.
- **Recommendation:** define one predicate, as a fixture table:
  - which files are parsed: every non-reserved `.md` under a root whose first line is `---`, or only concepts;
  - what each of these produces: a BOM, CRLF, a trailing-space `---`, an unclosed block, and a null, empty or non-string `type`;
  - keep `missing-type` as `[spec]` for a parsed block without a `type`, as decision 38 requires, or amend 38's test;
  - fold `unreadable` into `unparseable-front-matter`.

### B3. Decision 33 breaks io-insights' reader and gate, undercounts the rewrite, and its `--fix` is never built
- **Lenses:** design and implementation-readiness (Blocker); codebase-details, structural, zoom-out (Major).
- **Decisions:** 2, 4, 20, 33, 36; Stages 1 and 2.
- **Rule:** a path convention is only adopted when every reader that keeps running understands it.
- **The reader:**
  - `okf_read.resolve_repo_path` treats any resource without a leading `/` as a scope descriptor and returns `""` (`core/okf_read.py:119-120`). Its third branch exists for exactly the in-repo `/src/…` form that decision 33 rewrites (`okf_read.py:112-114, 133-138`).
  - Prior reading and reconcile depend on it (`pipeline/read_okf.py`, `core/okf_reconcile.py`; design and implementation-readiness give the call sites), as does the source list in `core/okf_html.py` (codebase-details).
  - So once any analysed repository follows the new `../` form, io-insights silently loses those citations. Design traced this through `core/okf_priors.py` to a demoted trust verdict. That trace is not re-checked here.
  - Decision 2 keeps this reader, and no stage changes it.
- **The gate:**
  - `RepositorySweepTests.test_every_okf_document_in_the_repository_is_conformant` sweeps every OKF document with `validate_document(..., root=ROOT)` (`tests/test_okf_frontmatter.py:277-283`).
  - Python flags every relative path as `non-bundle-relative-resource` (`okf.py:454-457`), so the Stage 2 rewrite turns the sweep red.
  - The Python checker is deleted only in Stage 3.
  - `NamedTargetTests` import `scripts/okf-validate` and break when it becomes a thin wrapper that checks bundles only (decision 41).
- **The count:** under `docs/` there are about 755 `/`-prefixed `resource:` values in 202 files:
  - 440 under `/src/`, `/tests/` or `/scripts/` (163 files), which decision 33 covers;
  - 289 under `/docs/` (153 files). They are inside the bundle but written against the repository root, so under a `docs/` root they resolve to `docs/docs/…` and dangle. Their fix is to strip the prefix and keep `/`, not to make them relative;
  - about 26 others, such as `/AGENTS.md` and `/.github/workflows/ci.yml`, which also leave the bundle.
- **The guide:** `docs/design/okf-authoring-guide.md:643-644` says "every path is bundle-relative rather than relative to the document". No row amends it.
- **The fixer:** Stage 2 rewrites the paths "with `--fix`" (brief line 480), and Goals puts path `--fix` in v1 (line 426). But Stage 1's `--fix` row covers only index front matter and `--actor` (line 469), and decision 20 says "the rest after v1's checks" (line 289).
- **What the spec says:** §6.2 lists the relative form without saying what it is relative to, or whether it may leave the bundle (`/tmp/okf0.2-spec:468-477`). A `docs/` bundle shipped on its own would dangle.
- **Recommendation:** before drafting, decide:
  - the base: relative to the document's directory;
  - the reader change: `resolve_repo_path`, which needs the document path it is not given today;
  - the two rewrite rules and their true scope;
  - whether the Python sweep and okf.py's path rules retire in Stage 2 or are ported;
  - whether path rewriting is an okf-check `--fix` (add about 2 Stage 1 days) or a one-off Stage 2 script. Its only consumer is io-insights' one-time migration; yz-docs' root is its repository root.

### B4. v1 schedules `publish_okf.py`, which is gated off by a stable io-insights decision
- **Lenses:** implementation-readiness (Blocker); design, codebase-details (Major).
- **Decisions:** 23 (third bullet), 32 (Stage 3), Goals (brief line 435), Stage 3 row.
- **Rule:** a plan cannot count as done the work on a path the project has deliberately closed.
- **Evidence:**
  - `publish_okf.main()` prints a gate message and returns 1, "Gated closed, not fixed (docs/design/repo-publish-is-gated.md)" (`pipeline/publish_okf.py:211-226`). The comment says re-opening it means settling how much of the write belongs inside the token-bound plan, and whether a repository publish may ever delete. Both are deferred contract questions.
  - Its module docstring says "No file is ever deleted here" (`publish_okf.py:39-44`).
  - Decision 23's "replaces the old bundle" deletes inside someone else's working tree, which is exactly the deferred question.
  - Decision 32 makes it part of the Stage 3 exit.
- **Recommendation:** drop `publish_okf` from v1 and record what it does when it is un-gated: check a temporary copy, then apply through the existing plan. Otherwise answer `repo-publish-is-gated.md`'s two questions and add the un-gating work to Stage 3.

## Major

### M1. The version pin has three sources and an "absent means latest" default that cannot exist
- **Lenses:** all 6.
- **Decisions:** 25, 37, 40.
- **The default:** "If absent, the tool uses the latest release" (line 368). A binary is one release. If the sentence means the script fetches "latest", that contradicts the committed SHA-256 that decisions 25 and 37 require (lines 308, 372).
- **Two pins per consumer:** the download script or `go.mod` pins one version, and `.okf-check.yaml` pins another. A mismatch refuses every run. A routine `go.mod` bump alone breaks the gate.
- **The exporter skips the pin:** it ignores config (decision 40), so the frozen build follows a third pin, and the corpus that the reader tests fetch can differ from it.
- **The policy has no force:** with exact-match refusal, "new rules are minor" and "patch releases never change findings" protect nobody.
- **Not defined:** where a `go install` build gets its version (design notes it reports build info, not a GoReleaser stamp; general Go knowledge, not verified here), the mismatch exit code, and a JSON-schema policy.
- **Recommendation:**
  - make `.okf-check.yaml` `version` the one pin, read by every download script next to per-platform checksums;
  - treat absent as "no check, with a warning";
  - say whether matching is exact or by minor range, and label the policy informational if exact.

### M2. The runtime binary location contradicts itself
- **Lenses:** design, implementation-readiness, codebase-details.
- **Decisions:** 24, 25; Stage 3 frozen-build row.
- **The contradiction:** decision 25 says "Neither ever prefers a copy on PATH" (line 309), then says okf-check "behaves as `codebase-memory-mcp` does today" (line 310). That pattern checks PATH first, then `~/.local/bin`, then copies the bundled binary into `~/.local/bin` (`core/bootstrap.py:19-45`), and `scripts/build-macos` also prefers a PATH copy (lines 36-46). Copying the pinned okf-check onto users' PATH would shadow other repositories' pins and trigger decision 37's refusal there.
- **Undefined:**
  - where non-frozen installs store the binary;
  - what happens on an offline first export, which decision 24 refuses;
  - concurrent first downloads;
  - where the universal macOS binary comes from: Stage 1 builds per-architecture binaries (line 470) and Stage 3 wants a universal one;
  - what the "frozen-build test" checks, since `tests/test_frozen_build_contents.py` skips sources outside the package and CI runs on Linux.
- **Recommendation:** lookup order `sys._MEIPASS`, then a versioned cache outside PATH, never PATH. State the offline behaviour.

### M3. The staging directory's contents and replacement step are undefined, and a directory swap is not atomic
- **Lenses:** design, implementation-readiness.
- **Decision:** 23.
- **What staging must contain beyond decision 23's list:**
  - old files that `write_bundle` deliberately does not rewrite: unchanged and restamp-only concepts (`export_okf.py:1067-1080`), and unchanged reference copies (`:867-871`);
  - `README.md`, which is written outside `markdown/` (`:1125-1128`).
- **Reads and events:** the claim-survival diff and the log merge read the live bundle (`:1055-1057`, `:1133-1136`). Events such as `okf.references` and `okf.prune_refused` are emitted for a bundle that may then be refused.
- **The swap:** POSIX `rename` cannot replace a non-empty directory. A swap takes two renames, with a window where `markdown/` is missing while the web app may be reading it. Symlink swaps are ruled out by `_assert_non_symlink_ancestors` (`:864`). (General POSIX knowledge, not verified in the repository.)
- **Recommendation:** stage only to check, then apply today's per-file writes, so no swap is needed. Otherwise state the staging location, the state readers may observe, and crash recovery.

### M4. One unparseable or old prior can refuse every later export
- **Lenses:** design, implementation-readiness, codebase-details.
- **Decisions:** 9, 14, 23, 28, 46.
- **"Unparseable" by whose parser?** After Stage 3 the only Python parser is PyYAML's `CSafeLoader` (`okf.py:178, 194`), which keeps the last duplicate key silently. A duplicate-key prior is readable to Python, carried forward, then refused by okf-check as `[spec]` `unparseable-front-matter`.
- **A kept prior refuses every run.** Decision 46 keeps a genuinely unparseable prior read-only, so it ships in staging and refuses the export on every run until a person edits it. That is the whole-bundle refusal decision 14 exists to avoid.
- **Decision 46 also misses deletion.** `read_bundle` drops such a file (`okf_read.py:87-90`). `write_bundle` then counts it in `previous` and unlinks it when it is not re-rendered (`export_okf.py:1027-1031, 1097-1098`).
- **Existing caches.** The first okf-check-gated export checks old kept files that passed the PyYAML-era rules: loose timestamps, duplicate keys, a non-string `description`. Restamp suppression stops them from being rewritten.
- **Recommendation:**
  - name the checker's parser as the definition of "unparseable", and give reconcile a strict load mode;
  - decide whether a read-only bad prior is left out of staging, advisory, or blocking;
  - name the event it emits;
  - add a first-run rule (for example, rewrite everything once) and a ledger case for it.

### M5. Decision 34's per-directory indexes break `write_bundle`'s diff and pruning
- **Lenses:** implementation-readiness, design.
- **Decisions:** 34; Stage 3 row (1.5 days).
- **Evidence:**
  - `previous` subtracts only the root `{"index.md", "log.md"}` (`export_okf.py:1027-1029`). So a generated `kb/index.md` counts as a concept, lands in `removed`, is unlinked and logged as removed on every run, and joins the `kb` family in `_refused_prune`. `publish_okf._existing_concept_paths` has the same filter (`publish_okf.py:62-68`).
  - `_prune_references` deletes generated `references/**/index.md` files (B1).
  - The brief does not say whether the root index still lists `kb/` concepts, or how `export_html` uses it.
- **Recommendation:**
  - filter reserved names by basename;
  - decide what the root index lists;
  - limit decision 34 to `kb/` and `sections/` unless B1 keeps index rules for `references/`.

  The change does not need okf-check, so it can land before Stage 3.

### M6. Decision 35 misdescribes okf-drift, and a faithful port flags yz-docs and the exporter's own index
- **Lenses:** codebase-details, structural, comprehension, zoom-out.
- **Decisions:** 19, 35, 39.
- **What `okf-drift/src/index-cmd.ts` actually does:**
  - **Leading link only.** A list entry is a `-` or `*` item whose text begins with the link (line 15), so later links in the same line are not entries.
  - **Exact-string navigation.** Only targets that are exactly `log.md` or `index.md` are treated as navigation (line 29), so `plans/log.md` counts as an entry.
  - **No outside-bundle detection.** `..` past the root is clamped by `path.pop()` (lines 50-53), so the tool cannot tell a target is outside, and decision 35's exemption has no implementation to port.
  - **`dangling-entry` is narrower than "missing".** It means "not a concept in this directory" (lines 317, 352-358), so a link to an existing file elsewhere is dangling.
  - **No angle brackets.** Destinations in `<…>` are not unwrapped (lines 15, 41-42). yz-docs has 255 of them in 10 indexes (codebase-details); for example, `docs/Features/index.md` links `<./Pending features/index.md>`.
- **Internal contradiction:** decision 35 says "links to `index.md` are not entries", and also "`subdir/index.md` is" an entry.
- **Recommendation:** write "port `index-cmd.ts` exactly, except …" and list each exception as a fixture:
  - CommonMark destinations;
  - navigation matched by basename;
  - out-of-bundle detection;
  - `dangling-entry` meaning "target does not exist";
  - precedence over `listed-non-concept`.

### M7. Two okf-drift description codes block both v1 consumers at `house`, and nothing costs the cleanup
- **Lenses:** zoom-out (Blocker); others did not test it. Graded Major here, because the remedy is a one-line scope choice and it does not stop drafting.
- **Decisions:** 10, 19, 22, 39; Stage 1 and Stage 2 exams.
- **What they fire on:**
  - **`undescribed-document`** fires when an index entry has a description and the concept has none (`index-cmd.ts:364-375`). Only 32 yz-docs files have a `description:`, and its indexes describe most entries. For example, `docs/Features/index.md:13` describes `Annotations.md`, which has none. Expect about 300 findings that decision 19's recount does not include.
  - **`description-drift`** compares index text with front matter (`index-cmd.ts:378-391`). In io-insights, `docs/design/index.md:11` describes `okf-authoring-guide.md` differently from its front matter (`docs/design/okf-authoring-guide.md:5`). Zoom-out found 6 of 6 sampled rows differ.
- **No consumer needs them in v1:** neither consumer runs okf-drift today. The only reason to port them is retiring okf-drift, which Staging puts after v2.
- **Recommendation:** move both codes to the okf-drift retirement point. If they stay, recount, add the remediation days, and consider them for `--fix`.

### M8. The yz-docs first-run list is still incomplete, so "clean at `house`" is not costed
- **Lenses:** codebase-details, comprehension, zoom-out.
- **Decision:** 19.
- **Corrections, beyond M6 and M7:**
  - **Actors.** There are 327 bare `generated.by` values (321 flow style, 6 block style, for example `README.md:3-5`), not 321.
  - **Logs.** `spec/log.md` also has no date headings (`2026-10-06:` at line 1). `docs/Implementation/log.md` has 18 suffixed headings, not one. Fixing them creates duplicate date groups, for example six `## 2026-09-28` headings from line 21 on, and the rule for duplicate groups is still undecided.
  - **A fix that makes a finding.** Removing `type` from `compiler/README.md` makes the root index's link to it fire `listed-non-concept` (decision 7), which blocks at `house`.
- **Recommendation:** rerun decision 19 with decisions 35 and 39 as written, then decide the duplicate-group rule.

### M9. Decision 20's index fix turns yz-docs' `spec/` into a separate bundle
- **Lenses:** all 6.
- **Decisions:** 4, 19, 20.
- **Evidence:** "Index front matter becomes `okf_version`" (line 290) applied to `spec/index.md`, a non-root index with `type: spec` (lines 1-4), makes `spec/` its own bundle under decision 4's exception (line 224). That re-roots every `/` path inside it.
- **Recommendation:** only the bundle-root index gets `okf_version`; a subdirectory index loses its front matter entirely.

### M10. Labels are assigned by a catch-all, and the result has gaps, duplicates and new export refusals
- **Lenses:** structural, codebase-details, design, implementation-readiness.
- **Decisions:** 9, 14, 29, 38, 39.
- **No code table in the brief.** It is only a Stage 0 task. The renames (`no-index`→`missing-index`, `unlisted-document`→`unlisted-concept`) and the dropped `--strict` are not recorded.
- **`invalid-description` is mislabelled.** Decision 9 calls `[integrity]` "the trust half of today's `FATAL_CODES`" (line 244). But `invalid-description` is okf-drift's "description must be a string" (`index-cmd.ts:283-291`). It is not in `FATAL_CODES` (`okf.py:122-125`) and is not a trust signal. At `--block=integrity` it newly withholds exports.
- **Inconsistent siblings.** `sources-not-list` and `source-not-mapping` fall into `[house]` through the catch-all. Their siblings `generated-not-mapping` and `verified-not-mapping` are `[integrity]`, and `source-missing-resource` is `[spec]`. So `sources: "x"` publishes, but one entry without `resource` withholds.
- **Decision 38's test is incomplete.**
  - §10.2 `runtime` is REQUIRED for attested computations and has no code.
  - §8 "Index files contain no frontmatter" (`/tmp/okf0.2-spec:512`) has no MUST or REQUIRED word. It is `[spec]` only through §11 rule 3 (line 743), so the test must admit §11's conditions.
- **`integrity` is not equivalent to `FATAL_CODES`.**
  - Newly blocking: `source-missing-resource` (advisory today, `okf.py:443-446`), the §8 and §9 checks, and `invalid-description`.
  - No longer blocking: `no-front-matter` and `missing-type`.
  - Each needs a ledger entry.
- **Codes without a code.** The §8, §9, missing-heading and newest-first checks, and BOM and unclosed blocks, have no code.
- **Recommendation:** put the code table in the brief, replacing decision 9's catch-all. Columns: code, label, status, old name, spec citation.

### M11. "The checked tree" is undefined, and the exporter's check can pass vacuously
- **Lenses:** comprehension, design, zoom-out, codebase-details, structural.
- **Decisions:** 4, 23, 24, 41.
- **Undefined targets:** decision 4's "no ancestor within the checked tree qualifies" never defines the tree. If `okf-check docs/design` makes `docs/design` the root, `/` resolution and decision 33's inside/outside test both change. A bare `okf-check` has no stated target.
- **A vacuous pass:** the exporter's bundle lives under a dot-directory: `~/.cache/io-insights` for served runs (`serve.py:159`), or `<repo>/.io-insights` (`core/artifacts.py:97-101`). If the dot-directory skip or `.gitignore` applies to the target's ancestors, okf-check sees zero concepts. Nothing says that is an error, so it passes and publishes unchecked trust claims, which decision 24 exists to prevent.
- **Its own fixtures:** the skip set has no `testdata/` or corpus marker, so okf-check's own `fail/` fixtures would fail its own gate.
- **Recommendation:**
  - skips apply only below the target;
  - a path argument resolves to the bundles containing it or beneath it, with the repository as the ceiling;
  - finding zero bundles is a non-zero exit;
  - drop `.gitignore` from v1 (zoom-out: no current consumer needs it), or define its scope.

### M12. The reader and the checker still find different bundle roots
- **Lenses:** codebase-details (Blocker), design.
- **Decisions:** 2, 4.
- **Evidence:** decision 4 calls "an `okf_version` index always starts its own bundle" `core/okf.py`'s "existing declared-root rule" (line 224). The code does the opposite:
  - `_walk_for_roots` stops at the first root, declared or inferred: "a root suppresses every candidate beneath it" (`okf.py:545`);
  - it infers a root from any directory holding a typed `.md`, with or without an `index.md` (`okf.py:519-534, 546-548`), only to depth 3;
  - its docstring says `references/` must never be read as a nested bundle (`okf.py:579-583`).

  `discover_write_root` and `read_priors` depend on this answer.
- **Severity:** graded Major, not Blocker. There is one defensible fix, and a drafter can apply it.
- **Recommendation:** align `bundle_roots` with decision 4 under discovery fixtures that both implementations run, or record the divergence and its reason. Correct decision 4's claim.

### M13. Staging omits v1 work, and the dates carry no buffer
- **Lenses:** zoom-out, implementation-readiness, structural, design.
- **Decisions:** 13, 32, 37; Goals; Staging.
- **The arithmetic is right:** stage sums 6, 19, 4 and 10, total 39, at 2 days a week plus a two-week holiday.
- **Not costed:**
  - path `--fix` or a script (B3);
  - the reader's relative-path change and retiring the Python sweep and named-target tests (B3);
  - the Python YAML-profile loader change (decisions 2, 18, 28);
  - decision 46's `read_bundle` and reconcile changes, and the cache migration (M4);
  - the basename filter (M5);
  - full catalogue text, beyond Stage 0's skeleton, which `explain` and decision 44 need;
  - the corpus-fetch harness in io-insights (37);
  - the description-code cleanup if kept (M7);
  - decision 13's amendments to `docs/design/log-entry-checks-not-gated.md` and guide §4;
  - decision 33's guide amendment;
  - an okf-drift deprecation notice.

  Reviewer estimates for this work range from 5 to 9 days, plus B3's 2.
- **No slack:** the dates are exact midpoints. The stated ±50% means decision 32's exam fails on any slip; at +50%, v1 lands around June 2027.
- **Recommendation:** add the rows, re-total after B1–B4 and M7 are decided, and give each stage a midpoint and a late date.

### M14. The agent-facing contract is under-specified: output inputs, JSON schema and the agent test
- **Lenses:** comprehension, structural.
- **Decisions:** 21, 37, 42, 44; Goals.
- **Output:**
  - Severity depends on the active level, which is never printed. Decision 42's sample shows `[house]` as `warning`, which implies `integrity`, the exporter's level rather than the repository's.
  - The input header and `skipped[]` were dropped without a reason.
  - Directory-level findings (`missing-index`) have no defined location.
- **JSON schema:** decision 21 makes it part of the contract, yet no field list exists. `got`, `expected`, `key` and `fix` appear only in decision 43's redaction rule, and `schema_version` left Goals without a record.
- **Agent test (44):** no agent, model, number of trials, threshold, judge, starting tree or case table. Its untyped seed is silent (B2).
- **Recommendation:**
  - print `config: <path> block: <level> roots: …` first and a count line last;
  - list the JSON fields with `schema_version`;
  - write the agent test as a table: case, expected finding, correct fix, and a wrong fix that must fail.

### M15. Run-event mapping and new failure modes are undefined
- **Lens:** implementation-readiness.
- **Decisions:** 24, 43.
- **Undefined:**
  - how okf-check's JSON fills `okf.validated` (concepts, advisory, fatal, `by_code`), the per-finding `okf.advisory` events and `okf.refused` (`export_okf.py:995-1021`);
  - whether "concepts" counts kept or reference files;
  - which event records a missing binary, a version mismatch or a crash. A Go panic also exits 2.

  `tests/test_export_run_events.py` asserts these shapes.
- **Recommendation:** add a mapping table, and say whether unit tests run the real pinned binary.

### M16. The fixture and ledger formats have no home and cannot carry what the decisions require
- **Lenses:** structural, implementation-readiness.
- **Decisions:** 2, 21, 36, 37, 41, 45.
- **What the decisions require of a fixture:**
  - several cases per code (21's `fail/<rule-code>/` allows one);
  - per-stage implementation markers (36);
  - a block level, because a `pass/` fixture can carry `[house]` findings (45);
  - reader-level expectations such as the trust tier and parsed values (2).
- **Two problems with the markers:** they are io-insights rollout data, yet decision 37 ships them to every consumer. The corpus also cannot be a release asset in Stage 0, because there is no release until Stage 1.
- **Recommendation:**
  - use `fail/<code>/<case>/`;
  - give `expected.json` the level, the exit code, the exact findings, and an optional reader block;
  - keep stage markers and the ledger in io-insights, keyed by fixture id and decision id.

### M17. The Overview's status claims cannot be traced, and "Still open" is incomplete
- **Lenses:** structural, comprehension.
- **Sections:** Overview (lines 16-17), Still open (406), Goals, Staging.
- **Wrong claims:** the Overview says blockers are answered by twelve decisions, with no mapping. Blockers 2 and 5 are not answered (B1, B2), and decisions 13 and 19 answered Majors 18 and 12, not blockers.
- **"Still open" omits:**
  - the "(proposed)" codes in 29 and the proposed Windows policy;
  - decision 23's unverified rename and decision 28's unverified `yaml.Node` check;
  - the ticket decision 34 depends on;
  - Q12 (staleness);
  - whether `bundle_roots` changes (M12).
- **Traceability gaps:**
  - Goals lines with no decision: `explain`, exit codes, the guide/skill/test;
  - Staging rows with no Goal: deleting the Python checker, the universal binary, the agent test, the CLI synopsis.
- **Recommendation:** add a table of second-review finding, decision, and resolved or declined with the reason. Make "Still open" complete.

### M18. The okfcli go/no-go cannot change the outcome
- **Lenses:** zoom-out, comprehension.
- **Decisions:** 1; Still open; Stage 0.
- **Evidence:**
  - The criterion asks whether okfcli covers "the `[spec]` and `[integrity]` rules with stable codes". `[integrity]` is this brief's own taxonomy, so no external tool can match it.
  - Its outcome is "revisit decision 1", with no named alternative.
  - The draft would be written before the trial, so the trial could not change the draft.
  - okfcli was not available locally to check.
- **Recommendation:** run the one-day trial before drafting, against criteria in okfcli's own terms and a named adopt design, or close decision 1 with its existing reason and empty "Still open".

## Nits

- **N1. Exit codes.** None is stated for a version mismatch (37) or an unknown config key (40). v2 reuses 2 for "history unavailable" (31), and a Go panic also exits 2. (structural, design)
- **N2. Config lookup leftovers** (15, 40):
  - decision 15 still says the walk starts "from the working directory", and its argument against a config inside the bundle conflicts with decision 40 starting at the bundle;
  - unstated: whether the nearest file wins, behaviour when `.git` is a file (worktrees) or absent (a tarball bundle), and one run covering bundles with different `version` values.
- **N3. `../` paths are not portable.** Say so in the catalogue entry. (implementation-readiness)
- **N4. `--fix` file scope.** Some `/`-paths sit in fenced examples in `okf-authoring-guide.md`, so `--fix` must edit front matter only. (implementation-readiness)
- **N5. Decisions 13 and 31 are v2 material in a v1 brief.** Amending a stable io-insights record now would describe a gate that does not exist yet. Move them to a "v2 notes" section and amend the record when the check ships. (zoom-out, implementation-readiness)
- **N6. Generated mode survives elsewhere.** Decision 16 is narrower than the code: `scripts/okf-validate --generated` and `tests/test_okf_bundle.py` still use generated mode, and nothing schedules their removal. (codebase-details)
- **N7. okf-drift's retirement is unclear.** Decision 22 says "retire", Staging archives it "later", and there is no deprecation notice for `npm install -g github:` users. (zoom-out, structural)
- **N8. The speed baseline has no motivation.** Decision 32 keeps a baseline and "no slower" with no reported speed problem. (zoom-out)
- **N9. Undefined or unlinked references:**
  - "carried forward", "reserved file", `_refused_prune`, "the 23-concept refusal", "the author's ticket";
  - guide §8 and §10;
  - decision 23's heading reference to "Still open item 1 of the first pass".

  Also: `missing-index`'s catalogue entry should say it is a producer-side rule (spec §11, `/tmp/okf0.2-spec:756-762`), and decision 30 should give its reason for hiding the folder READMEs. (comprehension, structural)
- **N10. Decision 12 gives no reason.** It reverses okf-drift's "curation is legitimate" (`okf-drift/docs/design/index-command.md`) without saying why. (structural, codebase-details)
- **N11. The guide's log-scope test depends on the old tool.** It uses "whether `scripts/okf-validate` lists it", and okf-check lists no checked documents. (comprehension)
- **N12. Reader tests need the network.** Fetching the corpus as a release asset makes `python -m unittest` network-dependent, and the asset has no checksum. (implementation-readiness, codebase-details)
- **N13. Structure:**
  - line 1 starts with a tab, so there is no H1;
  - decision groups and stage titles are bold text, not headings;
  - the YAML profile is spread over decisions 18, 28 and 46;
  - the 2026-11-06 groups are organised by which review prompted them, not by topic. (structural)
- **N14. Dates:**
  - the Overview dates the second review 2026-11-06, but `concepts_review_2.md` line 4 says 2026-11-05;
  - the header and the "Decisions (2026-11-05)" heading predate decisions 33–46. (structural, comprehension)

## What a drafter would still have to invent

**Must decide before drafting.** These are the Blockers:
- the rules that apply to `references/` copies (B1);
- the parse and concept predicate (B2);
- the reader's handling of relative paths, the rewrite scope and owner, and the Stage 2 test sequencing (B3);
- whether `publish_okf` is in v1 (B4).

**Strongly recommended before drafting, because they change scope or cost:**
- one version pin (M1);
- whether the description codes are v1 (M7);
- the okfcli trial (M18).

**Can be decided during drafting,** with the owner choosing between options where there is more than one defensible answer:
- binary lookup and offline behaviour (M2);
- check-only staging vs a swap (M3);
- the disposition of a bad prior and cache migration (M4);
- the basename filter and root-index content (M5);
- the index grammar exceptions (M6);
- the yz-docs recount and the duplicate-group rule (M8);
- decision 20's non-root fix (M9);
- the code table (M10);
- the checked-tree semantics and the zero-bundle exit (M11);
- `bundle_roots` alignment (M12);
- the re-totalled Staging (M13);
- the output header, JSON fields and agent-test table (M14);
- the run-event table (M15);
- the fixture and ledger schema (M16);
- the status table and "Still open" (M17);
- all Nits.

## Questions the author must answer before drafting

1. **`references/`.** When the exporter checks its staging bundle, do field and reserved-file rules apply to files it only copied? If they do, what does the user see when one foreign file refuses the export?
2. **What gets checked.** Is every non-reserved `.md` with a `---` block parsed and checked, or only files with a `type`? Is a block without a `type` `[spec]` `missing-type`, as decision 38's own test requires?
3. **Paths.** What is a relative `resource` relative to? Does io-insights' reader learn that form before Stage 2 rewrites anything? Do the 289 `/docs/…` values become `/…`? Is the rewrite an okf-check `--fix` or a one-off script?
4. **`publish_okf`.** Is it out of v1? If not, what are the answers to `docs/design/repo-publish-is-gated.md`'s two open questions?

## What the synthesizer re-checked

The synthesizer read these directly, rather than relying on the lens reports:
- **io-insights code:** `core/okf.py:60-199` and `:290-619`; `pipeline/export_okf.py:1-40` and `:815-1154`; `core/okf_read.py:70-144`; `pipeline/publish_okf.py:30-74` and `:205-234`; `core/bootstrap.py:1-51`; `core/okf_travel.py:1-130`; `core/run_events.py:52`; `core/artifacts.py:97-101`; `serve.py:159`; `scripts/build-macos`'s PATH lookup.
- **io-insights tests and docs:** `tests/test_okf_frontmatter.py:274-298`; `docs/design/okf-authoring-guide.md:5` and `:640-645`; `docs/design/index.md:11`; `docs/status.md:3`.
- **The spec:** `/tmp/okf0.2-spec:100-159`, `:286-555` and `:736-767`.
- **okf-drift:** `src/index-cmd.ts:1-100` and `:265-394`; `src/bundle.ts:60-119`.
- **yz-docs:** `index.md`, `spec/index.md`, `README.md`, `spec/log.md`, the headings of `docs/Implementation/log.md`, the `Annotations.md` row of `docs/Features/index.md`.

The synthesizer's own searches:
- 440 `/src|tests|scripts` resources in 163 files;
- 289 `/docs/` resources in 153 files;
- about 755 `/`-prefixed resources in 202 files;
- 32 yz-docs files with `description:`;
- 329 bare-actor occurrences in yz-docs, which matches 327 `generated.by` plus 2 `verified.by`.

Not verified by anyone: go.yaml.in/yaml/v4's `WithUniqueKeys` behaviour on `yaml.Node`, decision 46's duplicate-key scan, the 0700 output directory in decision 43, okfcli itself, and the POSIX and Go build-info points in M1 and M3. Line numbers attributed to a lens and not listed above come from that lens's report.
