# okf-check concepts brief: second review

* **Target:** `/Users/oscar/code/github/oscar-io/okf-check/concepts_brief.md`, Decisions 1–32, Still open, Goals vs. Non-Goals and Staging (brief lines 203–367). The 2026-11-04 review record is context only, not re-reviewed.
* **Date:** 2026-11-05
* **Lenses:** design, codebase-details, zoom-out, comprehension, implementation-readiness, structural (one instance each).

## Overview

**The brief is not ready to draft a design document from.** The decisions do not resolve the first review's three blockers, although the Overview says they do (lines 15–17). Most of the new problems come from three decisions.

- **Discovery (decisions 4–8).** A bundle root is the topmost `index.md`, and a concept is a file with a `type`. This makes `docs/` io-insights' bundle root, so 440 `/src|tests|scripts/…` resources dangle. Repo-root concepts such as `AGENTS.md` stop being checked. A forgotten or unparseable `type` produces no output. And typed files that the exporter copies into `references/` become blocking concepts.
- **One blocking level (decision 14), applied to the exporter (decision 17).** At `block: house`, every export io-insights produces is refused: `kb/` and `sections/` have no `index.md`, and untidy findings withhold again, which `FATAL_CODES` was written to prevent. At `block: spec`, trust breaks such as `bad-instant` and `duplicate-source-id` get published.
- **The stdin protocol and wheel distribution (decisions 23–26).** Each is a sentence where a contract is needed. The protocol also does not match `write_bundle`'s render order.

Several stated facts are wrong:
- yz-docs is not clean under decision 7;
- it has 321 bare `generated.by` actors, not "about 200";
- it has a second front-matter index;
- it has no CI to gate.

The Stage 2 exam, "zero differences" from `validate_document`, contradicts eight decisions that change findings on purpose.

Every lens reached the same verdict independently. The one real disagreement is about remedy. Zoom-out would cut the exporter switch (decisions 17, 23–27) from v1. The other five would keep it and specify it.

## Prioritized findings

Severity is the highest any lens gave. Where lenses differed, the spread is noted.

### Blockers

#### 1. io-insights' generated bundles fail decisions 7 and 12, so every export is refused at `block: house`
- **Lenses:** structural, zoom-out, implementation-readiness; the synthesizer re-read the code.
- **Decisions:** 7, 12, 14, 17, 29.
- **Rule:** a checker that its first producer cannot pass is not ready to draft.
- **Evidence:**
  - Concepts are written to `kb/{name}.md` (`src/io_insights/pipeline/export_okf.py:150`) and `sections/{slug}.md` (`export_okf.py:535`).
  - Only the root `index.md` is rendered (`export_okf.py:1122`; `render_index` in `core/okf_export.py:912-927` groups every concept by type into that one file).
  - So `missing-index` fires for `kb/` and for `sections/`, and `unlisted-concept` fires for every concept in them.
  - io-insights runs at `house` (decision 14), and the exporter uses the same level (decision 17), so `write_bundle` raises on every run.
  - A first run produces no bundle at all. A later run leaves the stale bundle on disk, where it is read as current.
- **Recommendation:** choose one before drafting:
  - the exporter writes per-directory indexes, a layout change that touches `export_html.py` and the tests;
  - decision 12 accepts a root index that lists descendants;
  - the exporter blocks on a different set from CI (see finding 3).

#### 2. Typed files in `references/` are the normal case, and decision 6's remedy cannot be carried out
- **Lenses:** design, codebase-details, zoom-out, comprehension.
- **Decision:** 6, with 5, 7, 14 and 17.
- **Rule:** a file the producer copies verbatim cannot carry the producer's obligations.
- **Evidence:**
  - `embed_references` copies cited files whole (`export_okf.py:815-878`). It cannot slice them, because a slice would shift cited line numbers.
  - The repositories io-insights analyses are often OKF-documented: 226 of 228 non-reserved docs in io-insights are typed, and all 355 in yz-docs are.
  - Every cited `.md` therefore lands in `references/` as a concept, which triggers:
    - `missing-index` and `unlisted-concept`;
    - every field rule against the other repository's front matter. yz-docs' bare actors are `[spec]`, so they block even at `block: spec`.
  - Decision 6 says "if the exporter ever copies a typed file, the finding is the exporter's to fix". Every fix available to the exporter is worse than the problem:
    - stripping front matter shifts line numbers;
    - not embedding leaves citations dangling;
    - excluding the files is the exclude list decision 6 removed.
- **Reversal:** this reverses a documented decision without saying so.
  - `okf.py:65-73` and `:580-583` exclude `references/` on purpose.
  - `write_bundle` excludes it too (`export_okf.py:1023-1029`).
  - The guide says a repository "is entitled to its own frontmatter contract".
- **Recommendation:** make `references/` at a bundle root a documented carve-out, citing spec §6.3, that is never checked as concepts.

#### 3. One `block` level cannot express the exporter's publish policy
- **Lenses:** all 6. Blocker for design and zoom-out; Major for the other four.
- **Decisions:** 14, 16, 17; claims to resolve first-review theme 1 (third axis) and theme 4.
- **Rule:** do not replace a policy with an axis that cannot represent it. A failed CI gate costs an agent one edit. A refused export leaves the user with nothing.
- **Evidence:** `FATAL_CODES` (`core/okf.py:109-125`) withholds only untrue claims. Its comment records why: blocking on untidy findings "withheld an entire 23-concept bundle over a regex in one sentence." No level of decision 14 reproduces that policy:
  - **`house` withholds too much.** It blocks on:
    - `title-heading-mismatch` and `footnote-without-source`;
    - `dangling-resource`, which happens by design when a cited file is withheld as a secret or oversized (`export_okf.py:822-827`), and on every `--no-embed-references` export;
    - the new `unlisted-concept`.

    `okf.advisory` events (`export_okf.py:998-1009`) become dead code. `GateSeverityTests` in `tests/test_okf_export.py` and `tests/test_export_run_events.py` flip.
  - **`spec` withholds too little.** It publishes `bad-instant`, `duplicate-source-id`, `generated-missing-by` and `generated-not-mapping`, which decision 9 leaves in `[house]`.
  - **The second axis survives.** Decision 16 keeps `forged-human-verifier` as an exporter-side fatal check, so "applies the same way in CI, local runs and the exporter" (decision 14) is false by construction.
- **Recommendation:** choose one:
  - put a per-code trust or withhold class in the catalogue and have the exporter block on it;
  - state that the exporter keeps its own code list over okf-check's JSON and drop the parity claim;
  - (zoom-out) cut the switch, so `FATAL_CODES` stays the exporter's own policy.

#### 4. Decision 4 makes `docs/` io-insights' bundle root, so its `/`-prefixed resources dangle and root concepts drop out
- **Lenses:** design and codebase-details (resources); structural, implementation-readiness, comprehension and codebase-details (root concepts).
- **Decisions:** 4, 32.
- **Rule:** the bundle root decides how every `/`-prefixed resource resolves, so it must match the root the documents were written against.
- **Resources:**
  - Today the gate resolves resources against the repository root (`scripts/okf-validate:127`, `root = … else REPO_ROOT`; `core/okf.py:450-453`).
  - Under `docs/` there are 440 `resource: /src/…`, `/tests/…` or `/scripts/…` values across 163 files; codebase-details counts 729 across 201 when `/docs/…` is included.
  - io-insights has no root `index.md`, so decision 4 makes `docs/` the root. Every one of those values then resolves under `docs/` and becomes `dangling-resource`, which is a blocking finding at `house`.
- **Root concepts:**
  - `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md` and `CHANGELOG.md` are typed and checked today. `AGENTS.md` is human-verified.
  - Under decision 4 they sit in no bundle and silently stop being checked.
  - Two tests break: `test_the_sweep_actually_finds_the_headed_documents` (`tests/test_okf_frontmatter.py:285-293`) and `test_a_named_document_is_still_checked`.
  - Decision 32's "zero differences on `docs/`" scopes parity around exactly this loss.
- **Recommendation:** separate the resource-resolution root from the discovery root, for example a `resource_root` key or the VCS root. Report typed files outside any root (`outside-bundle`) rather than skipping them.

#### 5. Defining a concept by its `type` makes the most common agent mistakes silent
- **Lenses:** all 6. Blocker for comprehension and design; Major for the other four.
- **Decisions:** 5, 8, 9, 14, 28.
- **Rule:** the check must cover the mistakes its users most often make, and the brief names agents as its users.
- **What an agent sees:**
  - **No output.** A new doc with a missing, misspelled (`Type:`, `tpye:`) or empty `type` is an "ordinary file", so a run with no index link exits 0 with nothing printed.
  - **No block when it does fire.** When the doc is linked, only `listed-non-concept` fires. That is `[house]`, so it does not block at the default `spec`.
- **What the spec loses.** §11 rule 2 ("every frontmatter block contains a non-empty `type`", spec line 742) can no longer fail, but decision 8 admits only to weakening rule 1.
- **This reverses documented intent.** `okf.py:601-603` includes untyped front matter "so that a document missing its required `type` is reported instead of quietly skipped".
- **Decisions 5 and 9 contradict each other.** Decision 9 makes unparseable front matter `[spec]`, and decision 28 routes duplicate keys there. But a block that does not parse has no readable `type`, so under decision 5 the file is ordinary and never checked. BOM-prefixed files and unclosed blocks have no defined class or code.
- **Recommendation:** classify by the presence of a front-matter block, as okf.py (`_walk`) and okf-drift (`src/bundle.ts`) both do. Keep `missing-type` as `[spec]`. Define the predicate for:
  - BOM;
  - unclosed and unparseable blocks;
  - a null, empty or non-string `type`;
  - trailing whitespace on the opening `---`.

#### 6. The stdin protocol is one sentence and does not match how `write_bundle` builds a bundle
- **Lenses:** all 6. Blocker for implementation-readiness and codebase-details; Major for the other four.
- **Decision:** 23; claims to resolve first-review theme 1 (write order).
- **Rule:** a pre-write check guarantees something only if it checks the bundle that will actually be written.
- **Render order.** Today only rendered concepts are validated (`export_okf.py:985-991`). The rest is produced after that gate:
  - `index.md` is rendered after `embed_references` has copied files into the live bundle, and its note depends on which files were withheld (`export_okf.py:1100-1124`, confirmed);
  - `log.md` is merged from the copy on disk (`export_okf.py:1130-1144`);
  - `_refused_prune` keeps concepts on disk that are absent from `rendered` (`export_okf.py:1035-1041`). So the bundle checked on stdin differs from the bundle written, and a later on-disk run fails `unlisted-concept`.

  "Writes nothing until it passes" therefore needs a plan/apply split of `embed_references` that no stage lists.
- **Undefined in the protocol:**
  - whether `references/` travels: files are up to `MAX_REFERENCE_BYTES`, may be non-UTF-8, and can be non-`.md`;
  - what "exists" means for `dangling-resource` when nothing is on disk;
  - the virtual root;
  - how stdin mode is selected;
  - the end-of-stream rule;
  - the response schema and stream;
  - exit codes that stay distinct from a Go panic, which also exits 2;
  - how output maps onto the `okf.validated`, `okf.advisory` and `okf.refused` events.
- **Second caller.** `pipeline/publish_okf.py`, named in first-review theme 1, renders a different index and log for the repo target (`publish_okf.py:87-131`) with no check. The brief never mentions it.
- **Recommendation:** write a protocol section with one request and response example. Require the payload to be byte-identical to what is written, for both callers.

#### 7. Wheel distribution leaves the build tool, platform tags and frozen-build lookup undecided
- **Lenses:** implementation-readiness (Blocker); design, codebase-details and zoom-out (Major).
- **Decisions:** 25, 26, 27.
- **Undecided items:**
  - **Wheel build and publishing.** GoReleaser does not build Python wheels. No wheel tool is named, and neither are a PyPI publishing workflow, the platform tags, an sdist, or the mapping from tag to version.
  - **Install failures.** A hard runtime dependency with no wheel for a platform (musl, others) makes `pip install io-insights` fail for everyone on it, not just for exporters. Decision 24's "binary missing" case then cannot occur.
  - **Frozen-build lookup.** "Resolves it from the installed wheel" has no branch for PyInstaller onefile, where the binary lives in `sys._MEIPASS`. That branch has the same shape as `core/bootstrap.py:32-45`, which decision 25 says is not reused.
  - **Universal builds.** `UNIVERSAL=1` (`scripts/build-macos:89-93`) would ship a single-architecture binary.
  - **The frozen-build test.** `tests/test_frozen_build_contents.py` parses the spec statically, and since `5a1357a` it skips sources outside the package. It runs on Linux CI, so it cannot "assert the binary is there" for macOS.
  - **Corpus pin.** Nothing pins the fixture corpus the Python reader is tested against (see finding 13).
- **Conflicting records.**
  - PyPI contradicts okf-drift's `docs/design/distribution.md:37,76`, which rejects an unscoped public registry name ("a permanent, global claim"). PyPI has no scopes. The brief does not mention this record (zoom-out; confirmed).
  - AGENTS.md §1 says "Everything else is stdlib", and `test_agents_runtime_dependency_table_matches_pyproject` enforces the table.
- **Recommendation:** name the wheel tool, the tags and the sdist policy. Specify the frozen-build lookup. Ship the corpus inside the wheel so one `uv.lock` pin covers binary, corpus and catalogue. Supersede `distribution.md` explicitly.

#### 8. The exporter switch still names no failure it prevents
- **Lenses:** zoom-out (Blocker); structural and design (Major).
- **Decisions:** 17, 23–27; first-review theme 1, "no payoff named".
- **Rule:** an expensive change needs a failure it prevents.
- **Evidence:**
  - Decision 17's reason is "keep dev and CI the same". That is a goal, not a failure. Findings 3 and 6 show the parity does not hold anyway.
  - The switch brings most of v1's cost: wheels, a second release pipeline, a new runtime dependency, a binary locator, the frozen-build changes, a second input mode, a `write_bundle` restructure, and a versioned JSON schema whose only v1 consumer is the exporter.
  - Shared fixtures plus the Stage 2 side-by-side run already cover reader drift.
- **Recommendation:** name the failure. Otherwise cut decisions 17 and 23–27 from v1, keep `validate_document` in the exporter held to the corpus, and reopen the switch when a failure appears. Five lenses specify the switch rather than cut it, so this is the author's call.

### Major

#### 9. Discovery has no single definition, no boundaries, and disagrees with the Python reader
- **Lenses:** all 6.
- **Decisions:** 2, 4, 6.
- **The definition contradicts itself.** "Topmost directory with an `index.md`" and "its parent has none" give different roots when there is a gap, for example `a/index.md` and `a/b/c/index.md` with no index in `a/b/`. That gap is the forgotten-index case decision 7 exists to catch.
- **There is no skip set.** okf.py has `_SKIP_DIRS` (`core/okf.py:75-78`), and okf-drift skips dot-directories as well (`src/bundle.ts`). Without one:
  - `vendor/`, `node_modules/`, `testdata/` and okf-check's own `fail/` fixtures become bundles, with no way to silence them;
  - Hugo, MkDocs and Docusaurus `index.md` files with front matter become `[spec]` §8 failures on the first run.
- **Nested and published bundles break.**
  - `discover_write_root` publishes to `docs/knowledge/` with an `okf_version` root index (`core/okf_repo_write.py:251-276`; `templates/okf/index.md:3-5`).
  - The exporter's cache sits at `<repo>/.io-insights/…/bundle/markdown/`.
  - Under a repository-level `index.md`, as in io-insights' `docs/` or yz-docs' root, neither bundle is topmost. Its `okf_version` then becomes a `[spec]` finding, and its `/slug.md` links resolve against the outer root.
  - Spec §3 allows a bundle in "a subdirectory within a larger repository".
- **Two algorithms.** The Python reader keeps `bundle_roots` (`core/okf.py:503-595`), which uses `okf_version` (declared) or a typed concept (inferred) with `max_depth=3`. Decision 4 says "there is no marker". Reader and checker will disagree on the same tree, and `discover_write_root` and `read_okf.read_priors` depend on the reader's answer.
- **Two minor cases.**
  - `src/io_insights/templates/okf/` becomes a bundle root. It is harmless today, but unstated.
  - A repository with typed docs and no `index.md` passes with zero checks, where `okf-validate` exits 1 today (`scripts/okf-validate:137-138`).
- **Recommendation:**
  - define discovery once, as fixtures both implementations run;
  - allow nested declared roots;
  - list the fixed skip set and say whether `.gitignore` applies;
  - report "no bundle found" instead of passing.

#### 10. The index link rule is undefined, and decision 7's "what this flags today" is wrong for both repositories
- **Lenses:** all 6.
- **Decisions:** 7, 12, 22, 29.
- **Links the literal rule would flag:**
  - io-insights' indexes link `log.md` (reserved, not a concept);
  - `docs/superpowers/index.md` links `plans/log.md` and `specs/log.md`;
  - `docs/development/index.md` and `docs/memories/index.md` link `../../AGENTS.md`, which is outside the bundle;
  - every yz-docs index links `./X/index.md`.
- **Link forms not covered:**
  - directory links (`subdir/`), which is spec §8 style;
  - `/`-links and `../` links;
  - angle-bracket destinations with spaces and parentheses (yz-docs);
  - percent-encoding;
  - table rows;
  - links to missing files, which spec §6.1 tolerates.
- **okf-drift already settled these semantics.** `src/index-cmd.ts`: a listing is a list item or table row; `log.md` and `index.md` are navigation; directory entries; `decodeURI`.
- **Recommendation:** port those semantics as the contract. State whether `unlisted-concept` is suppressed where `missing-index` already fires.

#### 11. Stage 2's "zero differences" exam contradicts the decisions, and Stage 0's baseline is circular
- **Lenses:** all 6 (Blocker for structural).
- **Decision:** 32 and Staging.
- **Why "zero differences" cannot hold.** It contradicts decisions 4, 7, 8, 9, 12, 16, 18, 28 and 29, each of which changes findings on purpose:
  - `no-front-matter` and `missing-type` stop firing;
  - new codes;
  - `forged-human-verifier` removed;
  - a new root for `dangling-resource`;
  - duplicate keys and loose timestamps now fail, and `type: no` now passes.
- **Why Stage 0 is circular.** "Corpus passes against okf.py" can only cover the old semantics, because okf.py has none of the index rules.
- **Recommendation:** measure "zero differences outside a ledger of intended changes, each tied to a decision". Mark each fixture with the implementation it must pass at each stage.

#### 12. The yz-docs facts behind decisions 7, 19, 25 and 32 are wrong, and Stage 1 depends on work scheduled after it
- **Lenses:** all 6 (Blocker for codebase-details).
- **What the brief got wrong (synthesizer re-checked each):**
  - **Two `missing-index` findings.** `compiler/README.md` and `compiler/examples/multi_root/README.md` carry `type: readme` in directories with no `index.md`, under the root bundle. Decision 7's "nothing in yz-docs" is therefore false.
  - **A second §8 finding.** `spec/index.md:1-4` carries `type: spec` plus `generated`; decision 19 lists only the root index.
  - **More bare actors.** 321 `generated.by: oscarryz` values across 321 files, not "about 200". There are also two bare `verified.by` values.
  - **Log headings.** `docs/Implementation/log.md` uses headings like `## 2026-09-28 (6)`, and `docs/log.md` has no date heading at all. Whether these are §9 findings depends on an undefined rule.
  - **Two unlisted concepts** in `docs/Examples/` (codebase-details: 141 typed files, 139 entries).
- **Where the gate would run (all confirmed):**
  - yz-docs has no CI: there is no `.github/` directory.
  - The only `go.mod` is `compiler/go.mod`, with `go 1.23`. `go get -tool` needs 1.24, and the gate would run from `compiler/` while the bundle root is the repository root.
- **The fix arrives too late.** The actor fix is exactly decision 20's `--actor` mapping, which is Stage 3, after the 2026-12-18 date for yz-docs to be clean.
- **Recommendation:** dry-run the new rules on yz-docs with a throwaway script before setting the Stage 1 target. Say where its gate runs. Move the actor mapping into Stage 1 or use a one-off script.

#### 13. Contract versioning, version skew and rollback are still unresolved (first-review theme 8)
- **Lenses:** structural, implementation-readiness, zoom-out, design, comprehension.
- **Decisions:** 2, 11, 21, 25.
- **What is missing:**
  - **No stability policy.** There is no rule for codes, for moving a rule between `[spec]` and `[house]`, or for the JSON schema. `schema_version` appears only in Goals.
  - **The decisions make breaking moves themselves.** `bad-actor` moves from house to `[spec]`, two codes leave `CONFORMANCE_CODES`, and `forged-human-verifier` leaves the catalogue.
  - **New rules break routine bumps.** House rules are on by default and both consumers block at `house`, so a routine wheel bump that adds a rule blocks every export.
  - **No corpus pin.** Nothing says how io-insights' tests get the corpus, or at which version.
  - **No rollback.** After `validate_document` is deleted there is no rollback path, and end users of the frozen app have no override.
- **Recommendation:** add a contract-versioning decision:
  - what is major, minor and patch for codes, levels and the schema;
  - where the corpus ships;
  - one pin covering binary, corpus and reader;
  - a rollback procedure.

#### 14. The YAML profile cannot be implemented as written, and a strict loader breaks the lenient reader
- **Lenses:** design, codebase-details, implementation-readiness, structural, zoom-out, comprehension.
- **Decisions:** 3, 18, 28.
- **"YAML 1.2 core, keep instants as datetimes" is a custom profile.** YAML 1.2 core has no timestamp type. The brief does not say:
  - which keys resolve to datetimes;
  - whether quoted instants do;
  - what a bare date becomes;
  - which other 1.1 resolvers go (on/off, sexagesimal, old octal).
- **The instant-key list is incomplete.** It omits `usage_window.from` and `usage_window.to` (spec §5.1) and the legacy `timestamp` key.
- **Decision 28's library rationale is partly wrong.** go.yaml.in/yaml/v3 rejects duplicate keys only inside `decoder.mapping()`. Decoding into `yaml.Node`, which decision 28 picks for line numbers, returns before that check (`decode.go:492-495` vs `:767-783` in v3.0.4, confirmed). That library also resolves YAML 1.1 timestamps.
- **Decisions 3 and 28 contradict each other.** Decision 3 calls reading "lenient", but decision 28 makes okf.py's loader reject duplicates. That loader serves nine importers and foreign bundles.
  - `okf_read` drops any document whose `split_front_matter` raises (`core/okf_read.py:87-90`, confirmed).
  - `_reconcile` then treats the missing prior as absent and overwrites it.
  - So a human-edited prior with a duplicate key loses custody silently.
- **`yaml_text` still launders bad timestamps.** It turns a bare date into midnight UTC (`core/okf_export.py`).
- **Recommendation:**
  - write the profile as a table: input text, parsed value, finding;
  - give checker and reader separate load modes;
  - name the PyYAML loader change;
  - decide what reconcile does with an unparseable prior.

#### 15. `[spec]` has no stated test, and `bad-actor` as `[spec]` exceeds the spec
- **Lenses:** design, structural, codebase-details.
- **Decision:** 9.
- **Rule:** `[spec]` should mean what the spec requires, because `spec` is the default blocking level.
- **Evidence:**
  - **What §7 actually requires.** Its only MUST is to use `human:` for hand-authored or human-confirmed content (spec lines 500-501). The actor forms are a convention, and §11 makes these families a producer SHOULD. Making actor form `[spec]` blocks `generated.by: claude-code` by default.
  - **`bad-instant` stays `[house]`,** although §5's offset rule has the same normative force as §7's actor form.
  - **REQUIRED fields are missing from the `[spec]` list:** `generated.by`, `sources[].resource`, and `runtime` for attested computations.
  - **Codes freeze before levels settle.** Decision 11 keeps a single `bad-actor` code, so the two cases cannot later be split without a breaking change.
- **Recommendation:** state the test for `[spec]`, cite a spec line for each rule, and split `bad-actor` before codes freeze.

#### 16. The code decisions contradict each other, and retiring okf-drift drops most of its index checks
- **Lenses:** all 6.
- **Decisions:** 8, 11, 22, 29; Goals.
- **Contradictions:**
  - **Dead codes.** "Keep today's codes" (decision 11, Goals) includes `no-front-matter` and `missing-type`, which decision 8 says no longer fire. Goals' "every cited code has a fail fixture" cannot pass for them.
  - **Rules with no code.** Decision 29 names no code for the `[spec]` §8 and §9 checks, for newest-first, or for BOM and unclosed blocks.
  - **Renames.** `missing-index` and `unlisted-concept` rename okf-drift's `no-index` and `unlisted-document`.
- **Checks lost.** "Its `index --check` becomes decision 12" is overstated. Decision 22 drops:
  - the codes `dangling-entry`, `undescribed-document`, `description-drift`, `invalid-description` and `unreadable`;
  - the flags `--write` and `--strict`.
- **Behaviour reversed.** okf-drift's design treats an unlisted document as "a question, not a defect" (`docs/design/index-command.md`). Decision 12 makes it a defect without giving a reason.
- **Recommendation:** add a code table: old code, new code, level, status (kept, renamed, retired, dropped).

#### 17. `.okf-check.yaml` lookup is wrong for the exporter and for path arguments, and its default fits no consumer
- **Lenses:** design, comprehension, implementation-readiness, codebase-details, zoom-out, structural.
- **Decisions:** 14, 15, 17.
- **The exporter's working directory is arbitrary.**
  - It may pick up the analysed repository's `.okf-check.yaml`. A `block: none` there would let unchecked bundles publish.
  - A stray `.okf-check.yml` is a usage error, which refuses the export.
  - With no file at all, the level is `spec`, not `house`.
- **Path arguments read the wrong config.** The walk has no stop condition, and `okf-check ../other` reads the caller's config rather than the target's.
- **The default fits no consumer.** Both named consumers use `house`, yet the default is `spec`, so the file exists only to override the default (zoom-out).
- **Typos fall back silently.** Unknown keys are unspecified, so `blcok: house` would revert to `spec`.
- **Recommendation:**
  - resolve config from the target and stop at the VCS root;
  - the exporter always passes `--block` and ignores config files;
  - make unknown keys a usage error;
  - (zoom-out) default to `house` and cut decision 15.

#### 18. The v2 log merge check reverses a stable io-insights decision without citing it
- **Lenses:** zoom-out (decision record; confirmed); design, structural, implementation-readiness and comprehension (mechanics).
- **Decisions:** 13, 29 (`missing-log-entry`), 31.
- **The reversed record.** `io-insights/docs/design/log-entry-checks-not-gated.md` is `status: "stable"` and says: "Do not propose a CI check that requires a `log.md` entry whenever a document changes." Its reason is that such a check forces the vague trap. Decision 13 rewrites "Earn the entry", and decision 31 gates exactly what that record forbids. Neither cites it.
- **Mechanics:**
  - **Rebase.** The merge-base date window breaks on rebase: an entry dated when the branch began fails once the base moves.
  - **Timezone.** None is stated.
  - **Exit 2.** It is reused for "history unavailable", although v1 defines it as a usage error.
  - **CI depth.** io-insights CI checks out at depth 1 (`ci.yml`).
  - **Drift anchor.** Decision 31's "newest `verified`" anchor still conflicts with `okf_travel`'s first-event anchor. Decision 2 keeps that Python drift engine.
- **Recommendation:** cut decision 31 and `missing-log-entry` from this brief, or supersede the design record explicitly with what changed since 2026-09-21. Theme 11's advice to make v2 a separate decision was not followed.

#### 19. The CLI surface is never stated, so the Stage 2 "thin wrapper" is undefined
- **Lenses:** comprehension, structural, codebase-details, implementation-readiness.
- **Decisions:** Goals, Staging, 15, 16, 20, 23.
- **What callers rely on today.** The finalize-branch skill (`.agents/skills/finalize-branch/SKILL.md:33`), the guide and `NamedTargetTests` all call `okf-validate <changed files>`, and some pass `--generated`.
- **What the brief leaves open:**
  - what a named file means when the index rules need the whole bundle;
  - what happens to a named reserved or ordinary file;
  - the default target of a bare `okf-check`.
- **No synopsis.** Stdin mode, `--block`, the JSON flag, `explain`, `--fix` and `--actor` are scattered across decisions.
- **Recommendation:** add a CLI synopsis and one paragraph on named-file mode.

#### 20. The output still cannot tell an agent what failed, where, or under which inputs
- **Lenses:** comprehension; structural (theme 9 mapping).
- **Decisions:** 9, 14; Goals lines 327-329.
- **No line number.** The text format `[spec]|[house] path: code: message` has no line, although the next Goals line promises file, line and key. Agents and editors parse `path:line:`.
- **No blocking marker.** At the default `spec`, `[house]` findings print and the run exits 0. That is first-review theme 4's "reads as optional", still unresolved.
- **The `[house]` label is unresolved.** Decision 9 defines it as "one of these rules", with no referent. In yz-docs it still reads as yz-docs' own style (theme 5).
- **Inputs are not printed.** Theme 9's printing of active inputs (config path, level, discovered roots) and its `skipped[]` list were dropped without comment.
- **Recommendation:** use `path:line: [spec] code: message (key=…; got …; expected …)`, mark each finding blocking or advisory, and print `config: <path> (block: house); roots: …` first.

#### 21. Richer findings leak analysed-repository text into run events
- **Lenses:** design, implementation-readiness.
- **Decision:** 23; Goals (theme 9 fields).
- **Evidence:** `REDACTED_FIELDS` is `{"error","issues","rule","message","detail"}` (`core/run_events.py:52`). The new `got`, `expected`, `fix` and `key` fields quote the analysed repository's text. If the exporter forwards them under those names, they reach the event sink unredacted, against AGENTS.md §5.
- **Recommendation:** state the mapping from okf-check JSON to run events, and add the new fields to the redaction set.

#### 22. The decision 32 exam is not runnable, and it can reward forging `human:`
- **Lens:** comprehension.
- **Decisions:** 16, 20, 32.
- **Not runnable as written:**
  - The Stage 1 exam seeds only `[spec]` rules, yet yz-docs gates at `house`.
  - The agent, the model, the number of trials and the pass criterion (exit 0, or a correct fix) are undefined.
  - Silent cases (finding 5) cannot fail the exam, because they produce no output.
- **Can reward forgery.** An agent told that `bad-actor` expects `human:<id>` will invent one. With `forged-human-verifier` moved into io-insights, nothing in a consuming project catches it.
- **Recommendation:**
  - seed house rules and the "new doc without `type`" case;
  - define a pass as a correct fix;
  - make `explain bad-actor` say that only a person may supply `human:`.

#### 23. Default house rules still carry io-insights' vocabulary and fail the spec's own examples (theme 5 not resolved)
- **Lenses:** structural (Major); design, zoom-out and comprehension (Nit).
- **Decisions:** 3, 10.
- **Evidence:**
  - `bad-implementation` checks io-insights' `implementation` key (`core/okf.py:386-390`), yet decision 3's io-insights binding omits it.
  - `non-bundle-relative-resource` still fires on Appendix A's `resource:`, which §6.2 allows.
  - `title-heading-mismatch` still fires on §4.3 and §4.4.
  - With both consumers at `house` and no per-rule switch, the spec's own examples fail the gate, and they cannot serve as `pass/` fixtures.
- **Recommendation:** move `bad-implementation` into the binding, and fix the two rules that misfire on the spec's examples.

#### 24. The brief contradicts itself on what is open, in scope and staged
- **Lenses:** structural, comprehension, design, implementation-readiness.
- **Sections:** Overview lines 15-17, Still open, Goals, Staging.
- **What is open:**
  - Decisions 9, 25 and 29 are "(proposed)", yet "Still open" lists only the okfcli spike.
  - Decision 17 says theme 1's mechanics "stay open below", and nothing below is open.
  - Q12 (staleness in the reader contract) is answered nowhere.
- **What is in scope.** Goals' "today's rules and codes, minus `forged-human-verifier`" and "the two new house rules" contradict decision 8 (two more codes retired) and decision 29 (three v1 codes).
- **What is staged.** Staging never schedules:
  - the catalogue;
  - `explain`, which the Stage 1 exam depends on;
  - JSON output;
  - the GoReleaser and wheel pipeline;
  - the guide, the skill and their test;
  - the rewrites of tests that import `CONFORMANCE_CODES` and `FATAL_CODES`;
  - the `write_bundle` restructure.
- **Stage 0 misreads the test files.**
  - It converts the contract test's reachability checks, which decision 12 rejects.
  - It treats `LogMergeTests`, which tests the `merge_log` writer, as checker cases.
  - Stage 2 "runs alongside" before the wheel exists in CI.
- **The okfcli spike has no go/no-go criterion.** Decisions 4–31 proceed as if build-vs-adopt were settled.
- **Recommendation:** list every "(proposed)" item and Q12 under Still open. Cite a decision for each Goals line and a Goals line for each stage task. Name the spike result that would reverse decision 1.

#### 25. The scale does not fit the stated side-project pace, and there is no cost line
- **Lenses:** zoom-out, design, structural.
- **Decision:** 32 (theme 12 not resolved).
- **Stage 1's four weeks must deliver:**
  - discovery;
  - about 25 rules plus new codes;
  - block levels and config;
  - text and JSON output with locations;
  - `explain`;
  - the catalogue, the guide, the skill and the cited-code test;
  - the corpus with `expected.json`;
  - GoReleaser for six targets;
  - `go tool` support.
- **Stage 2 spans the year-end holidays** and adds the wheels, the stdin protocol, the `write_bundle` restructure and the frozen build.
- **No effort estimates.** None exist for any stage.
- **Recommendation:**
  - add an effort line per stage;
  - (zoom-out) cut the exporter switch, the config file, v2 and `--fix` from v1;
  - drop the windows and arm64 builds until someone consumes them.

### Nits

- **Undefined terms and dangling references:**
  - "carried forward" (decision 16);
  - "reserved file";
  - "the `# OKF v0.2` comment";
  - "Still open item 1 of the first pass" (decision 23 heading), which has no target;
  - guide §8 and §10, cited without links.
  (comprehension, structural)
- **Decision 13 reads as a v1 rule,** but its check is v2. (comprehension)
- **Decision 20's index fix is wrong for non-root indexes.** "Index front matter becomes `okf_version`" fails for an index such as yz-docs' `spec/index.md`, which must lose its front matter entirely. (comprehension, design, codebase-details)
- **Decision 20's `--actor` mapping can raise a trust tier.** Applied to `verified[].by`, it raises the document from machine-confirmed to human-reviewed. Say so. (codebase-details)
- **Decision 16 is narrower than the code.** `scripts/okf-validate --generated` and `tests/test_okf_bundle.py` also use generated mode. (codebase-details)
- **Decision 21's fixture layout allows one case per code.** Use `fail/<code>/<case>/`, and give an `expected.json` schema with one checker example and one reader example. (structural, comprehension, design)
- **Duplicate date groups are undecided.** `docs/evaluations/log.md` has two `## 2026-10-01` groups. Say whether that is a finding. (codebase-details)
- **`missing-index` is a producer-side rule.** Its catalogue entry should say so, because §11 says consumers "MUST NOT reject a bundle because of missing `index.md` files". (comprehension)
- **Decision 30 removes the folder guides.** Dropping the README links from the indexes hides the guides that `test_docs_landing_files_each_keep_one_job` protects. Consider exempting navigation READMEs. (design)
- **okf-drift's retirement date is unclear.** Decision 22 says retire it now, Staging archives it at v2, and there is no deprecation notice for its external `npm install -g github:` users. (structural, implementation-readiness)
- **The decision 32 speed baseline measures nothing reported.** No one has reported a speed problem, so drop "no slower than baseline". (zoom-out)
- **The document structure is broken in places:**
  - line 1 starts with a tab, so the title renders as a code block;
  - the decision groups are bold text, not headings;
  - the YAML profile is split across decisions 18 and 28.
  (structural)

## Where the lenses agree

| Finding | Lenses |
|---|---|
| One `block` level cannot express the exporter's publish policy (3) | all 6 |
| Concept-by-`type` makes missing or unparseable `type` silent (5) | all 6 |
| Stdin protocol is underspecified and mismatches the write order (6) | all 6 |
| Discovery: ambiguous, no skip set, disagrees with `bundle_roots` (9) | all 6 |
| Index link semantics undefined; decision 7's impact claim wrong (10) | all 6 |
| Stage 2 "zero differences" contradicts the decisions (11) | all 6 |
| yz-docs facts wrong; Stage 1 depends on later work (12) | all 6 |
| Code decisions contradict each other; okf-drift checks dropped (16) | all 6 |
| Config lookup wrong for exporter and path arguments (17) | all 6 |
| YAML profile not implementable; strict loader breaks lenient reader (14) | 6 (comprehension minor) |
| Contract versioning and version skew unresolved (13) | structural, implementation-readiness, zoom-out, design, comprehension |
| Exporter's own `kb/`/`sections/` layout refused at `house` (1) | structural, zoom-out, implementation-readiness (+ synthesizer) |
| `references/` typed copies are routine and unfixable by the exporter (2) | design, codebase-details, zoom-out, comprehension |
| Repo-root concepts drop out of the gate (4) | design, structural, implementation-readiness, codebase-details, comprehension |
| Wheel distribution mechanics undecided (7) | implementation-readiness, design, codebase-details, zoom-out |
| Brief inconsistent on open/in-scope/staged (24) | structural, comprehension, design, implementation-readiness |

**Where they disagree:**
- **The exporter switch (finding 8).** Zoom-out would cut it from v1. The other five would specify it.
- **The config file (finding 17).** Zoom-out would default to `house` and cut the config file. Design would extend the file to declare bundle roots.

Either resolution of the first disagreement removes or reshapes findings 3, 6, 7 and 21.

## Status of the first review's blockers

| First-review blocker | Claimed resolution | Status |
|---|---|---|
| 1. Exporter switch | 14, 16, 17, 23–27 | Not resolved: no payoff (8), publish policy lost (3), protocol and order (6), `publish_okf` unaddressed (6), distribution undecided (7) |
| 2. YAML profile | 18, 28 | Partly resolved: the profile is not specified (14), and the Python reader gets no reader-level fixtures once `validate_document` is deleted (design) |
| 3. Spec rule 1 scope | 4–8 | Replaced by new failures: resources dangle and root concepts drop out (4), `references/` blocks again (2), rule 2 silently dropped (5) |

Q12 (staleness) is unanswered. Q9 is only partly answered: do yz-docs' agents write `implementation`, footnoted sources or generated bundles? Theme 9's `skipped[]` list and visible inputs were dropped without comment.

## Questions the author must answer before drafting

1. **Exporter switch.** What failure does it prevent that the corpus plus the Stage 2 side-by-side run would not? If none, is it cut from v1?
2. **Withholding a bundle.** Which findings withhold one: a per-code trust class, the exporter's own list, or the repository's `block` level? Is withholding over `title-heading-mismatch` acceptable again?
3. **The exporter's own layout.** How do `kb/` and `sections/` pass decisions 7 and 12: per-directory indexes, or a root index that may list descendants?
4. **`references/`.** Is it exempt by rule? If not, how does the exporter embed a typed `.md` without shifting line numbers?
5. **Bundle roots.**
   - Who declares a root: topmost, or parent-has-none?
   - Are nested `okf_version` roots legal?
   - What is the skip set, and is `.gitignore` honoured?
   - Does `core/okf.bundle_roots` change to the same rule?
6. **Resource root.** What root do `/`-prefixed resources resolve against? Are `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md` and `CHANGELOG.md` meant to leave the gate?
7. **Untyped and unparseable files.** Is front matter without a non-empty `type` a `[spec]` finding? What do an unparseable block, a BOM, an unclosed block and a trailing-space `---` each produce?
8. **Stdin protocol.** What is the contract (mode flag, virtual root, references, response schema, exit codes)? How is the payload kept identical to what is written? Does `publish_okf` use it?
9. **Wheels.** Which tool builds them, for which tags, and is there an sdist? How does the frozen build locate the binary? Does the corpus ship in the wheel under one pin?
10. **Versioning and rollback.** What is the policy for codes, `[spec]`/`[house]` moves and the schema? What is the rollback path once `validate_document` is gone?
11. **The `[spec]` test.** What puts a rule in `[spec]`, and is `bad-actor` split before codes freeze?
12. **Codes.** What is the full code table, including the §8 and §9 checks and okf-drift's codes, with kept, renamed or dropped for each?
13. **YAML profile.** What is the exact profile table? Does the lenient reader reject duplicate keys? What does reconcile do with an unparseable prior?
14. **Index links.** What is the link grammar for the index rules (reserved targets, sub-indexes, out-of-root links, missing targets)?
15. **yz-docs.** What is the recounted first-run list, where does the gate run, which Go version applies, and how are 321 actors fixed before `--fix` exists?
16. **Log merge check.** Do decisions 13 and 31 supersede `docs/design/log-entry-checks-not-gated.md`? If so, what changed?
17. **Parity and the spike.** What is the ledger of intended differences that replaces "zero differences"? Which okfcli spike result would reverse decision 1?

## What the synthesizer re-checked

The synthesizer re-read the claims the verdict rests on, rather than relying on the lens reports:
- `core/okf.py:95-134` and `:440-459`;
- `pipeline/export_okf.py:820-860` and `:958-1145`;
- `core/okf_export.py:905-944`;
- `core/okf_read.py:80-93`;
- `scripts/okf-validate`'s root default;
- `docs/design/log-entry-checks-not-gated.md`;
- okf-drift's `docs/design/distribution.md`;
- yz-docs' `index.md`, `spec/index.md`, `compiler/go.mod`, `compiler/` READMEs and `docs/Implementation/log.md`;
- the absence of yz-docs' `.github/`;
- go.yaml.in/yaml/v3 v3.0.4 `decode.go:485-499` and `:767-783`.

The resource count (440 across 163 files) and the bare-actor count (321) come from the synthesizer's own searches. Other line numbers come from the lens reports.
