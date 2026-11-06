# okf-check concepts brief: fourth review

* **Target:** `/Users/oscar/code/github/oscar-io/okf-check/concepts_brief.md`. Only the changes made after `concepts_review_3.md` were reviewed: decisions 5, 6, 20, 23, 37, 39, 47 and 48, the two new Stage 2 tasks, and the rewritten Overview. Each was also checked against decisions 1–46, Goals vs. Non-Goals, Still open and Staging.
* **Date:** 2026-11-06
* **Lenses:** design and zoom-out, one instance each. No other lens was run.
* **Level:** concept only. Items a design document would naturally specify are listed under "Left for the draft", without severity.

## Verdict

**Not ready to draft yet, but there are no blockers, and the remaining decisions need no new investigation.** The new decisions answer their third-review findings soundly:

- Decision 6's `--exclude references/` resolves B1 and keeps okf-check generic. It matches io-insights' own `NON_OKF_PREFIXES` rationale.
- Decision 5 resolves B2.
- Decision 20 resolves M9.
- Decision 23 resolves B4.
- Decision 47 closes M11's vacuous pass.

Three things still leave the concept ambiguous or unworkable:

1. **The scope sections say the opposite of three revised decisions** (M1, M2). The brief's "later wins" rule covers decisions, not Goals, Still open or Staging.
2. **The Stage 2 path rewrite breaks two io-insights consumers that keep running until Stage 3:** the Python sweep in the gate (M3) and the reader's bundle root (M4). The new `../` reader task fixes neither.
3. **Decision 34 still has the exporter write `references/` indexes**, which decision 6 has made pointless (M5).

| Severity | Count |
|---|---|
| Blocker | 0 |
| Major | 5 |
| Nit | 4 |

## Major

### M1. Goals and Non-Goals contradict decisions 6 and 23
- **Lenses:** design, zoom-out.
- **Decisions:** 6, 23; Goals vs. Non-Goals.
- **Problem:** the scope section still states the old position on two revised decisions:
  - **`publish_okf.py`:** Goals (Consumers) lists "io-insights' exporter and `publish_okf.py` (17, 23, …)" as in scope. Decision 23 now takes `publish_okf.py` out of v1.
  - **`exclude`:** Out of scope lists "an `exclude` list". Revised decision 6 brings it back, and so does the Stage 1 scaffolding row.
- **Why it matters:** the drafter takes scope from Goals. As written, two items are both in and out of v1.
- **Evidence:** `publish_okf.py:211-226` confirms that `main()` is gated closed. Its dormant path goes through `export_okf.write_bundle` (`publish_okf.py:251-257`), so leaving it out costs nothing.
- **Recommendation:**
  - move `publish_okf.py` to Non-Goals, with decision 23's reopening note;
  - drop "an `exclude` list" from Non-Goals;
  - cite decision 6 in the Goals binary list.

### M2. Decision 48 leaves decision 1 both settled and open
- **Lenses:** design, zoom-out.
- **Decisions:** 1, 32, 48; Stage 0; Still open.
- **Problem:** decision 48 says the okfcli trial "informs, it does not gate". Three places still treat it as a gate:
  - the Stage 0 row says "Go/no-go: … revisit decision 1";
  - decision 32's Stage 0 measure is "written up against its go/no-go criterion";
  - Still open lists "Build vs adopt: the okfcli spike result", and the Overview repeats it.
- **The concept question (zoom-out):** if the trial cannot change decision 1, build vs adopt is closed, and a Stage 0 day has no decision attached unless the brief names what the trial produces.
- **Recommendation:** choose one:
  - close decision 1, empty Still open, and restate the trial as harvesting divergence cases for the corpus;
  - drop the trial.

### M3. The Stage 2 path rewrite turns io-insights' gate red until Stage 3
- **Lenses:** design, zoom-out.
- **Decisions:** 32, 33, 36; Stages 2 and 3.
- **Problem:** Stage 2 rewrites `docs/` paths: `/src/…` becomes `../src/…`, and `/docs/…` becomes `/…`. The Python checker is deleted only in Stage 3. In between, `RepositorySweepTests` still runs `validate_document(..., root=ROOT)` on every OKF document in the repository and fails on any finding (`tests/test_okf_frontmatter.py:276-293`). After the rewrite:
  - every `../` path is `non-bundle-relative-resource` (`core/okf.py:454-457`);
  - every `/x` that was `/docs/x` is `dangling-resource` against the repository root (`core/okf.py:450-453`).
- **Why it matters:** Stage 2's exit in decision 32 ("gated at `house` locally and in CI") cannot coexist with a green `scripts/check`. Third-review B3 raised this "gate" half, and no new task answers it.
- **Scope note (zoom-out):** the sweep also covers `AGENTS.md`, `CLAUDE.md` and `CONTRIBUTING.md`. Retiring it is where decision 4's "files outside `docs/` are not checked, by choice" takes effect, so the brief should record that at that step.
- **Recommendation:** choose one:
  - add a Stage 2 row that retires or re-roots the sweep (and `NamedTargetTests`) when okf-check starts gating `docs/`, keeping `validate_document` for the exporter until Stage 3;
  - move the path rewrite to Stage 3.

### M4. After the rewrite, the reader resolves `/` against a different bundle root
- **Lens:** design. The synthesizer re-read the code.
- **Decisions:** 2, 4, 33; Stage 2 reader task.
- **Problem:** decision 33 makes `/` mean okf-check's bundle root, which is `docs/` (decision 4). io-insights' reader finds its roots with `okf.bundle_roots`:
  - it stops at the first directory that holds a typed `.md` (`core/okf.py:546-548`);
  - io-insights' `AGENTS.md` carries `type: "Agent Instructions"` (line 3);
  - so for io-insights the reader's root is the repository, not `docs/`.

  After Stage 2 turns `/docs/design/x.md` into `/design/x.md`, `resolve_repo_path` tries `<repo>/design/x.md` under both the bundle root and the repository root (`core/okf_read.py:127-139`). Neither exists, so it returns `""`, and the citation is dropped without any message. Teaching the reader the `../` form does not help, because these paths keep the `/` form.
- **Why it is concept-level:** the Overview leaves third-review M12 (aligning `bundle_roots` with decision 4) for the draft, but the Stage 2 rewrite depends on it. A migration that silently drops the reader's citations does not work as staged.
- **Scope of impact:** it applies when io-insights reads its own docs as priors, and to any repository whose `docs/` bundle sits beside a typed file at the repository root. yz-docs is not affected, because its bundle root is its repository root.
- **Recommendation:** bring M12 into Stage 2, before the rewrite. Either make the reader use decision 4's root, or record why the roots differ and resolve `/` against okf-check's root.

### M5. Decision 34's `references/` indexes serve no check and keep a known defect
- **Lenses:** zoom-out (Major), design (Nit). Graded Major because it is in-scope work that revised decision 6 made pointless, and it carries a defect already on record.
- **Decisions:** 6, 34; Stage 3 row.
- **Problem:** decision 34 still has the exporter write an `index.md` in every directory "including `references/`". Under decision 6 the exporter's check never walks `references/`, so neither `missing-index` nor `unlisted-concept` can fire there. The spec also says consumers "MUST NOT reject a bundle because of missing `index.md` files".
- **The defect:** a cited `docs/index.md` lands on the same path as a generated `references/docs/index.md`. `_prune_references` deletes every file under `references/` that the current run did not embed, generated indexes included (`pipeline/export_okf.py:881-898`; third review B1 and M5).
- **Recommendation:** limit decision 34 to `kb/` and `sections/`, and shrink the Stage 3 row to match.

## Nit

### N1. Decisions 5, 8, 9 and 39 still disagree about `missing-type`
- **Lenses:** design, zoom-out.
- **Problem:**
  - Decision 8 still says `no-front-matter` and `missing-type` "no longer fire on their own".
  - Decision 9's catch-all would make an unlisted code `[house]`.
  - Only decisions 5 and 39 keep `missing-type` as `[spec]`.
  - Decision 8 records only the narrowing of spec rule 1. Decision 5 also narrows §11 rule 2 ("every frontmatter block contains a non-empty `type`", `/tmp/okf0.2-spec:740-742`): front matter with neither `type` nor the OKF keys is silently ordinary.
- **Recommendation:** rewrite decision 8 to record both narrowings and drop its `missing-type` clause. List `missing-type` under `[spec]` in decision 9.

### N2. Decision 19's README fix now trips `missing-type`
- **Lens:** synthesizer, found while checking decision 5 against 1–46.
- **Problem:** decisions 7 and 19 say yz-docs' `compiler/README.md` and `compiler/examples/multi_root/README.md` "lose their `type` and become ordinary files". Both carry `generated: { by: "oscarryz", … }` (lines 1-4 of each). Under decision 5, removing only `type` leaves OKF-looking front matter, which produces `[spec]` `missing-type`. That blocks at every level.
- **Recommendation:** the fix is to drop the whole front-matter block. Say so in decision 19.

### N3. The accepted description-code cleanup is not in any stage exit
- **Lenses:** design, zoom-out.
- **Decisions:** 19, 32, 39; Stages 1 and 2.
- **Problem:** decision 39 accepts about 300 yz-docs findings from the two description codes. They are not in:
  - decision 19's first-run list;
  - the Stage 1 yz-docs row.

  io-insights' `description-drift` findings (third review, M7) are not mentioned for Stage 2's `house` gate either. Neither consumer runs okf-drift today, so the codes serve only okf-drift's retirement, which comes after v2.
- **Recommendation:** choose one:
  - name the cleanup in the yz-docs row and the "Clean `docs/`" row;
  - move the two codes to the okf-drift retirement point.

### N4. The `exclude` config key has no v1 consumer
- **Lens:** zoom-out.
- **Decisions:** 6, 40.
- **Problem:** the only stated user of `exclude` is the exporter. It passes the flag, and under decision 40 it ignores config files. Neither io-insights' `docs/` nor yz-docs' root needs an exclusion. io-insights' other `NON_OKF_PREFIXES` lie outside `docs/` or in dot-directories, which decision 4 already skips.
- **Recommendation:** ship the repeatable `--exclude` flag in v1, and add the config key when a repository needs it.

## Left for the draft

- Whether the dot-directory and `.gitignore` skips apply to the target's ancestors. With decision 47, this decides whether an exporter staging directory under `.io-insights` or `~/.cache` is refused on every run.
- Whether a block that fails to parse is checked under decision 5's `type`-based predicate.
- Which keys count as OKF-looking for `missing-type`: only `generated`, `verified` and `sources`, or also `stale_after` and `status`.
- Decision 37's leftover "If present" wording next to "It is required".
- The two pins per consumer (download script or `go.mod`, plus `.okf-check.yaml`), and how a `go install` build knows its own version.
- How `--exclude` paths resolve when one run covers several bundles.
- Recording that excluded `references/` copies fall outside the bundle's conformance claim (spec §6.3 calls them first-class concepts).
- Decision 23's "checks exactly what ships", given that `references/` is excluded and serves only `[house]` `dangling-resource`.
- The rule that makes a path `--fix` one-answer: rewrite only when a path dangles from the bundle root and exists from the repository root.
- The base for `../` paths in `resolve_repo_path`, which is not given the document path today.
- Goals lists only out-of-bundle paths for `--fix`, while Stage 2 also rewrites the in-bundle `/docs/…` paths.
- A kept unparseable prior in staging (third review, M4), which now depends on the parse rule above.
- `NamedTargetTests` importing `scripts/okf-validate` once it becomes a thin wrapper.
