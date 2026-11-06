	# okf-check
* **Author:** Oscar Reyes
* **Date:** 2026-11-04, decisions added 2026-11-05

## Overview

okf-check is a standalone Go binary (`oscar-io/okf-check`) that checks Open Knowledge Format v0.2 documents. Its purpose is to let io-insights' OKF rules, fixtures and authoring guidance be reused in other projects, mostly Go, where AI agents write the documents and the checker's findings are how they learn what to fix. It replaces io-insights' in-process validator (`validate_document` in `src/io_insights/core/okf.py`) and okf-drift's TypeScript `index --check`. io-insights keeps its Python reader, held to the same fixtures.

Where it stands:
- **[Decisions](#decisions-2026-11-05)** 1–52 record what was settled across four reviews. Where decisions conflict, the later one wins.
  - **Bundles and concepts:** a bundle is the outermost `index.md` with a concept beneath it, and a concept is a `.md` with a `type`.
  - **Labels and blocking:** findings are labelled `[spec]`, `[integrity]` or `[house]`, and each bundle sets one blocking level and its `exclude` paths in `.okf-check.yaml`.
  - **YAML:** a YAML 1.2 profile on `go.yaml.in/yaml/v4`.
  - **Distribution:** GitHub Releases only.
  - **Consumers:** yz-docs first, then io-insights' docs, then its exporter in v1, which checks a staging directory.
- **Reviews:** `concepts_review_2.md`, `concepts_review_3.md`, and `concepts_review_4.md` (a concept-level review of the changes after the third; answered by decisions 49 and 50 and by text fixes). The third review's resolution table maps each second-review finding to its decision.
  - **Concept-level findings** from the third review are answered by decisions 5, 6, 20, 23, 37, 39, 47 and 48, and by the Stage 2 path and reader tasks.
  - **Design-level Major findings** (M2–M6, M10, M14–M16; M12 moved into Stage 2 by decision 50) are deliberately left for `/egm:egm_draft` and its review. `concepts_review_3.md` goes along as input.
  - **The full yz-docs first-run list** (M8) comes from a dry run in Stage 1.
- **Still open:** the okfcli trial, a Stage 0 task that informs and does not gate.

The review record below is unchanged. It found three blockers: the exporter switch, the missing YAML profile, and spec rule 1's scope. The first pass of decisions claimed to resolve all three. The second review showed they did not.

Lenses consulted: design (architecture and cost), codebase-details (grounding and reuse), zoom-out (YAGNI), comprehension (agent and developer ergonomics), implementation-readiness (migration and compatibility), structural (maintainability).

Line references come from the reviewers, who read the files. The synthesizer re-read `core/okf.py:100-194`, `core/okf.py:596-622` and `pipeline/export_okf.py:980-1021` to check the claims everything else depends on.

## Analysis (review record, 2026-11-04)

### Objections and risks, by theme

**1. Switching io-insights' exporter to a subprocess breaks guarantees the exporter relies on. Blocker. Raised by design, codebase-details, structural, implementation-readiness and zoom-out (5 of 6).**
- **Write order.** `write_bundle` validates rendered text in memory before anything is written (`pipeline/export_okf.py:985-1021`, confirmed). On a fatal finding it raises, and the previous bundle stays intact. A binary needs files on disk first, so the change needs either a staging directory with an atomic swap or a stdin protocol.
- **Per-document mode.** The generated-output rules are switched per document (`generated_output=path not in carried_forward`, line 990, confirmed). The proposed CLI and config have no per-document mode. `scripts/okf-validate`'s `--generated` flag is global.
- **Third axis.** Publishing is blocked by `FATAL_CODES` (`okf.py:122-125`, confirmed), which is independent of [spec]/[house]. `duplicate-source-id` is a house rule and still fatal. The proposed catalogue (code, level, meaning) has nowhere to record this.
- **Other dependents.** A second caller, `pipeline/publish_okf.py:254`. References are embedded after the write. Run events depend on exact finding codes and send `message` as a redacted field (lines 1003-1009, confirmed).
- **Runtime dependency.** io-insights ships a PyInstaller build, so the exporter would need okf-check on end users' machines.
  - The existing native-binary pattern (`core/bootstrap.py`, `scripts/build-macos`) prefers a copy on PATH, bundles whatever version the build machine has, and builds for the host architecture only.
  - `tests/test_frozen_build_contents.py` skips `bin/`, so the gate would not notice a missing or wrong binary.
  - If the binary is missing, the exporter must either fail closed (export unusable) or fail open (publishes the unchecked trust claims that `FATAL_CODES` exists to stop). Nobody has chosen.
- **No install route.** CI sets up uv and Node, not Go (`.github/workflows/ci.yml`). `scripts/check` silently skips the JS checks when Node is missing; copying that pattern would let the conformance gate pass without running.
- **Dependency rule.** AGENTS.md §1 treats a new dependency as "a real decision".
- **Cost of the PyInstaller rejection.** It was rejected because Go CI would need per-platform builds. This switch puts the same burden on Python CI and on io-insights' frozen build.
- **No payoff named.** Zoom-out: nothing in the concept says what failure this switch prevents that shared reader fixtures would not.

**2. Without a defined YAML profile, the fixture corpus cannot stop the two parsers diverging. Blocker (design), Major (structural, codebase-details, implementation-readiness). All 6 lenses raised a parser concern.**
- **Two parsers remain.** The Python reader stays (`split_front_matter`, `trust_tier`, `verification_events`, `bundle_roots`, plus `claim_blocks`, `differs_only_by_generated_at`, `RESERVED_FILENAMES` and `okf_export.yaml_text`). Only about 130 lines of `validate_document` move to Go.
- **Undecided cases** where PyYAML 1.1 and a Go YAML 1.2 library give different findings:
  - Duplicate keys: PyYAML keeps the last value silently; yaml.v3 fails, which becomes [spec] `unparseable-front-matter`.
  - `type: no`: YAML 1.1 reads it as `False`, so `str(meta.get("type") or "")` gives `missing-type` in Python. Go reads the string "no" and reports nothing.
  - Loose timestamps (`2026-8-5 9:00:00 +2`): PyYAML resolves them to an aware datetime, so `is_instant` passes (`okf.py:167-168`, confirmed). A lexical check in Go fails them as `bad-instant`, which is fatal. The same value passes unquoted and fails quoted in Python today.
  - `okf_export.yaml_text` turns a bare date into midnight UTC, which quietly makes an invalid instant valid.
- **Trust tier already differs between okf.py and frontmatter.ts.**
  - Python counts any `by` starting with `human:`, even when malformed. TS drops events whose actor is invalid. So `human:Oscar Reyes` is human-reviewed in Python and unverified in TS, and `okf_reconcile.py` decides overwrite custody from this tier.
  - TS flags an empty `verified` list; Python does not.
  - TS accepts trailing whitespace on the opening `---`; Python does not.
- **Correction to the problem statement.**
  - CRLF: the CRLF divergence holds only for raw strings. The sweep reads files with `Path.read_text(encoding="utf-8")` (`okf.py:617`, confirmed), which converts CRLF to LF, so CRLF files on disk are parsed and checked. Codebase-details said CRLF files are never validated; that is wrong for the sweep.
  - BOM: a BOM-prefixed file fails `startswith("---\n")` (line 620) and is skipped without any report. Fixtures must work on bytes in files, not on strings.
- **The reader/checker split is in the wrong place (design).**
  - "Staleness" is in the reader contract, but okf.py has no staleness function (comprehension), and staleness needs instant parsing, which the concept files under checker rules.
  - Bundle discovery and file classification are in neither level, though both sides need them.

**3. Spec rule 1 conflicts with io-insights' opt-in discovery and its exported bundles. Blocker. Raised by design, structural, implementation-readiness and codebase-details (4).**
- **Rule 1 vs the sweep.** Spec §11 rule 1 requires front matter on every non-reserved `.md`. okf.py sweeps only files that start with `---\n` (`okf.py:620`, confirmed). The guide says the repo is "deliberately not a conformant bundle".
- **What fails on day one.** Under "config cannot disable spec rules", these become [spec] failures:
  - `README.md`;
  - `docs/README.md`;
  - the templates under `src/io_insights/templates/okf/`;
  - in a Go repo, its README.
- **Exported bundles.** The exporter copies cited files whole into `references/`, which `NON_OKF_PREFIXES` excludes today (`okf.py:73`), yet spec §6.3 calls its contents first-class concepts. Checking the whole bundle directory would turn every copied `.md` without front matter into a fatal [spec] finding and withhold the bundle.
- **`exclude` already disables rule 1.** It is how io-insights avoids a repo-wide rule 1 today, so the claim that config cannot disable spec rules does not hold as stated.

**4. One rule axis is not enough, and context-dependent rules have no inputs. Major. Raised by comprehension, design, structural, codebase-details and implementation-readiness (5).**
- **okf.py has three independent axes:** conformance vs house (`CONFORMANCE_CODES`, line 104), fatal vs advisory (`FATAL_CODES`, line 122), and generated vs authored (`generated_output`).
- **Deferring per-rule severity does not work.** The first consumer, io-insights' exporter, already needs two severity levels.
- **"Forged human verifier on by default" needs a mode.** Without a generated/authored mode, it either flags every real human verification, such as the one in `DRIFT-0007.md`, or never fires.
- **`dangling-resource` needs a bundle root**, and in a Go repo the root is undefined.
- **The exit code reads as "optional".** `scripts/okf-validate` prints "all house rules; the documents remain OKF-conformant" and still exits 1, so an agent reads house findings as optional (comprehension).

**5. House rules on by default export one project's taste and fail the spec's own examples. Major. Raised by design, structural, comprehension and zoom-out (4).**
- **The spec's own examples fail.** `title-heading-mismatch` fires on spec §4.3 and §4.4. `non-bundle-relative-resource` fires on Appendix A's `resource: dashboards/exec-revenue`, which §6.2 allows. DRIFT-0007 already records this defect.
- **Conflicting rule.** "Index lists every concept" contradicts okf-drift's own "curation is legitimate" rule (`docs/design/index-command.md`). io-insights' gate checks reachability from the index (`tests/test_documentation_contract.py`), not completeness.
- **Project vocabulary.** The `implementation` key is io-insights vocabulary.
- **The label misleads.** In another project, "[house]" reads as that project's rules (comprehension).
- **Unneeded config (zoom-out).** The `spec` preset has no consumer, and a config file is not needed in v1: repeatable `--exclude`, `--generated` and `--root` flags cover it.
- **Suggested split (design, structural):**
  - default: spec rules plus portable trust rules (actor and instant forms, duplicate source id, footnote without source, forged verifier in generated mode);
  - opt-in: style rules (title vs heading, bundle-relative paths, index completeness, newest-first);
  - an io-insights preset or extension: `implementation`.

**6. The [spec] index and log checks are narrower than the concept implies, and existing docs would trip stricter ones. Major. Raised by zoom-out, comprehension, structural, codebase-details and implementation-readiness (5).**
- **The actual MUSTs are two:** no front matter in index files except `okf_version` in the bundle-root index (§8), and `YYYY-MM-DD` log date headings (§9).
  - Newest-first is not a MUST.
  - Whether rule 3's "follows the structure" means more than these two is unresolved.
- **Already partly enforced elsewhere.**
  - `tests/test_documentation_contract.py` checks that indexes carry no front matter and that every document is reachable from an index.
  - `tests/test_okf_export.py` covers date headings and ordering for generated logs.
  - So "okf.py skips §8/§9" is true of okf.py and false of the gate.
- **Formats that a stricter structure check would reject:**
  - `docs/design/log.md` writes entries as paragraphs, not bullets;
  - `docs/index.md` has prose before its first section and uses `##`/`-`;
  - `src/io_insights/templates/okf/index.md` is a Jinja template named `index.md`.
- **A test flips.** `NamedTargetTests` in `tests/test_okf_frontmatter.py` asserts that index and log files are skipped, so it must be rewritten.

**7. The rejection of okfcli/okf does not hold for the chosen design. Major. Raised by zoom-out and design (2), with structural asking whether a Go port is justified before a Go consumer adopts it.**
- **Two parsers either way.** okfcli was rejected for "subprocess only, giving two parsers". The proposal gives io-insights the same thing: a Python reader plus a Go subprocess. Go projects would also use okf-check as a subprocess or via `go tool`.
- **Same gaps covered.** By its README, okfcli already has stable rule IDs, SARIF, trust tiers and the §8/§9 checks, which are the gaps the concept cites in okf.py.
- **Same YAML library.** Both would sit on yaml.v3 or its fork.
- **The defensible reasons are different:** owning the house rules and the log merge check. Spend about a day running okfcli/okf on io-insights' docs and one Go repo before writing a parser.
- **The landscape is incomplete.** It omits okf-kit, the npm validator at v0.10 that frontmatter.ts already defers to (codebase-details). Sudhakaran88/okf-conformance could supply the [spec] fixtures (design, zoom-out).

**8. The contract is premature in some parts and underspecified in others. Major. Raised by zoom-out, design, structural, implementation-readiness and comprehension. The lenses disagree on remedy.**
- **Zoom-out: too much.** A two-level, language-neutral contract with a JSON schema has no second implementer. Keep fixtures as Go testdata and put a rule-code table in the README.
- **The other four: not enough.**
  - Directory-name expectations cannot say "exactly these findings", which level, or which preset. Use an `expected.json` per fixture (design).
  - Codes need a stability policy. okf.py uses `bad-actor` and `bad-instant` where okf-drift uses `malformed-trust`, and io-insights tests import `CONFORMANCE_CODES` (structural).
  - The JSON output needs `schema_version`, the tool version and the spec version (structural, implementation-readiness).
  - Moving a rule between [spec] and [house] is a breaking change (structural).
- **Version skew.** The binary, the corpus and the Python reader can drift apart unless one pin covers all three (implementation-readiness).
- **Reconciliation.** Both sides agree the reader-level cross-language cases are the real need. The disagreement is how much versioned surface to publish now.

**9. Findings do not tell an agent where the problem is or how to fix it. Major. Raised by comprehension, codebase-details and design.**
- **No location.** `Finding` holds only `code`, `message` and `path` (`okf.py:128-135`, confirmed): no line, no key. `bad-instant` comes from four keys, and `bad-actor` from `generated.by` and every `verified[].by`.
- **No fix hint.** Messages do not show the valid forms or the fix.
- **Proposed fields:**
  - `level`, `blocking`, `file`, `line`, `key` (for example `verified[1].at`);
  - `got`, `expected` and a one-line `fix`;
  - a catalogue anchor;
  - `okf-check explain <code>`, readable offline because agents often cannot open URLs.
- **Visible inputs.** Print the active root, the generated mode and the excludes.
- **Skipped files.** List them with reasons (`skipped[]`), and give BOM and unclosed blocks their own codes rather than skipping them or reporting `no-front-matter`.
- **Token cost (design).** Agents rerun the checker often, so keep a terse one-line-per-finding text mode.

**10. "Each rule stated once" is not enforced, and the guide is already inaccurate. Major. Raised by comprehension, structural and codebase-details.**
- **Rules are stated in five places today:**
  - io-insights' guide;
  - okf-drift's `docs/design/authoring-guide.md`;
  - AGENTS.md §16;
  - SKILL.md;
  - the proposed tool guide and skill.
- **Consuming projects will copy rules too**, into their own AGENTS.md.
- **Proposed jobs:**
  - catalogue: every checkable rule, self-sufficient, with spec quote, bad and good snippets, and fix;
  - guide: judgement only;
  - skill: procedure only.
  - Add a test that fails when a doc cites an unknown code or a code has no fail fixture.
- **Guide errors** (codebase-details, implementation-readiness):
  - lines 616-624 claim okf.py implements all three conformance rules;
  - line 42 cites "spec section 9" for attested computations, which are §10;
  - `/finalize-branch` cites §7.1 for repo logs, where the rules are in §4.
- **The guide split is coarser than "§8 and §10".** §8's first table and §10 item 2 are generic. Parts of §4, §7, §7.1, §11 and the status header are io-insights-specific.

**11. v2 is scope creep, and part of it already leaks into v1. Major. Raised by zoom-out, structural, codebase-details, design and comprehension.**
- **No trigger for a rewrite.** okf-drift works. Its own decision record revisits the language only if it is adopted outside JS-shaped projects (`language-choice.md:49-51`).
- **Index completeness is already in v1.** The v1 house rule "index lists every concept" is okf-drift's `index --check` (structural).
- **okf_travel and okf-drift `check` do different jobs (codebase-details).** okf_travel compares content span by span, runs in-process, and anchors on the first `verified` event. okf-drift compares commit times and uses the newest event. A Go `check` will not replace okf_travel, so v2 adds a third drift engine.
- **The log merge check has unresolved problems:**
  - It encodes io-insights conventions (`(#<PR>)`, implementation plans).
  - It needs a PR number and a landing date that CI does not have before merge.
  - It inherits a contradiction: guide §4 says routine additions do not earn an entry, while AGENTS.md §16 requires one for every new document.
  - It fails on shallow clones.
  - Zoom-out asks whether it belongs in an OKF tool at all.

**12. No success measure, timeline or cost, and a solo maintainer. Major (design), Nit (structural).**
- **Missing:** a count of target repos, a date, the cost of the repo, release pipeline, schema and cross-language harness, and a success exam.
- **Wrappers (structural).** Make npm, pip and brew wrappers an explicit non-goal.
- **The pip wheel (implementation-readiness).** If io-insights consumes the binary in v1, a pip wheel locked in `uv.lock` is needed in v1, not "later on demand".
- **Contradiction.** These two positions conflict only if io-insights switches; theme 1's recommendation removes the conflict.

**13. Factual corrections to the concept. Nit. Raised by codebase-details and implementation-readiness.**
- **Importers.** Nine modules in `src` import the reader functions, not about 7: `okf_read`, `okf_html`, `export_html`, `okf_priors`, `okf_reconcile`, `okf_travel`, `export_okf`, `pipeline/read_okf` and `okf_repo_write`.
- **Hot path.** `okf_read._CACHE` keys the parsed bundle on file size and mtime, so a bundle is reparsed when it changes, not on every request. The comment at `okf.py:176-177` is stale. Validation is not on that path at all.

### Where the lenses agree

| Finding | Lenses |
|---|---|
| Some front-matter parsing divergence must be decided and fixed in fixtures before porting | all 6 |
| Exporter subprocess switch is unsound or unjustified in v1 | design, codebase-details, structural, implementation-readiness, zoom-out |
| Catalogue needs blocking/fatal and applicability, not just level | comprehension, design, structural, codebase-details, implementation-readiness |
| [spec] index/log checks must be limited to the two actual MUSTs | zoom-out, comprehension, structural, codebase-details, implementation-readiness |
| Spec rule 1 scope conflicts with `exclude` and opt-in discovery | design, structural, implementation-readiness, codebase-details |
| House defaults should split trust rules from style rules | design, structural, comprehension, zoom-out |
| v2 should be separate decisions, not a staged commitment | zoom-out, structural, codebase-details, design |

The one real disagreement is how much contract to publish now (theme 8): zoom-out says almost none, the other four say more than the concept specifies.

### Open questions

1. **Exporter.** Is switching io-insights' exporter to okf-check a goal in itself, and what failure does it prevent that shared reader fixtures would not? If it switches and the binary is missing, does export refuse or publish?
2. **Severity.** Does the catalogue carry the blocking (publish-withholding) class, or does io-insights keep `FATAL_CODES` as its own exporter policy? Are they the same thing as "house findings fail CI"?
3. **YAML profile.** Is it normative for every reader (YAML 1.2 core, duplicate keys as error, timestamps as strings, instants judged on source text)? Will io-insights change its loader, `is_instant` and `yaml_text` to match? Which Go library: go.yaml.in/yaml/v3 or goccy/go-yaml?
4. **Scope.** Which folders are bundles in a mixed repo, and who declares them? Does `exclude` remove files from rule 1's scope? Is "headed documents only" a legitimate mode, or should io-insights become a conformant bundle?
5. **`references/`.** Is it excluded by a documented rule, or does the exporter change what it embeds?
6. **Rule 3.** Does it mean only the explicit MUSTs, or the §11 wording "follows the structure"? Are newest-first and the §7 MUST at spec line 501 [spec] or [house]?
7. **Index rule.** Is it reachability (what io-insights' gate checks) or completeness (what the concept lists)?
8. **okfcli spike.** Has okfcli/okf, or any listed validator, been run on io-insights' docs, and where does it disagree?
9. **First consumer.** Which Go project is first, by when, and do its agents write `implementation`, footnoted sources and generated bundles, or a subset?
10. **Generated output.** How does okf-check know a document counts as "generated" in a project with no exporter?
11. **Codes.** Do they keep okf.py's names (`bad-instant`) or become namespaced IDs?
12. **Staleness.** Should the reader contract include it, given nothing implements it today?
13. **Drift anchor.** Which `verified` event anchors drift: the first (okf_travel) or the newest (okf-drift)?
14. **Repo logs.** Which wins: "earn the entry" (guide §4) or "every new document gets an entry" (AGENTS.md §16)?
15. **Log merge check.** Does it belong in okf-check, or in the finalize-branch skill's own script?
16. **Signing.** Does a binary inside the macOS frozen build need its own signing? This was not checkable from local files.

## Decisions (2026-11-05)

Settled in discussion after the review. Where they differ from the review record above, these win. "Q" numbers refer to the review's open questions.

**Shape and home**

1. **Build okf-check as a Go single binary in its own repository, `oscar-io/okf-check`.** It owns its conformance layer instead of wrapping okfcli/okf; the one-day okfcli spike stays in Stage 0 as a check on that choice. A single binary runs as easily as `grep` in any project, whatever its language.
2. **Keep the Python reader in io-insights, held to the shared fixtures.** io-insights generates OKF knowledge, reads its front matter heavily, and uses it to detect drift, so `split_front_matter`, `trust_tier`, `verification_events` and the rest stay in `core/okf.py`. Other languages can implement against the same fixtures later; none is planned.
3. **Split the concerns into five pieces:**
   - format reading (lenient);
   - format checking (okf-check);
   - change policy against `main` (the log merge check, v2);
   - agent guidance (a generic guide and a procedure-only skill);
   - the io-insights binding (guide §8 evidence mapping, §10 privacy, `forged-human-verifier`), which stays in io-insights.

**Which files are checked** (resolves Q4, Q5 and theme 3)

4. **A bundle root is the outermost directory with an `index.md` whose tree holds at least one concept.** No ancestor within the checked tree qualifies. An `index.md` with no concept beneath it, such as an MkDocs, Hugo or Docusaurus site, makes no bundle, so its front matter is never a §8 finding. A fixed set of directories is always skipped: dot-directories, `node_modules/` and `vendor/`. `.gitignore` is respected.
   - **A subdirectory's `index.md` is a listing, not a new bundle.** The spec's own layout (§3) and the OKF announcement's example both put one in every directory.
   - **Exception:** an `index.md` that carries `okf_version` always starts its own bundle. A bundle embedded in another, such as an exported or vendored one, then keeps its own `/` paths. This is `core/okf.py`'s existing declared-root rule (`_is_declared_root`). Otherwise `okf_version` stays optional (§12 MAY).
   - **Markers:** the `# OKF v0.2` comment is a house habit the spec never mentions, and the tool ignores it.
   - **In io-insights the root is `docs/`.** Spec §3 allows "a subdirectory within a larger repository", and the announcement describes a bundle that "lives in version control alongside the code it describes". Files outside it, such as `AGENTS.md`, are not checked, by choice.
5. **A concept is a non-reserved `.md` file under a bundle root whose front matter has a `type`.** A file with no front matter, or with front matter but no `type`, is an ordinary file and is not checked. The exception is front matter that looks like OKF: if it carries `generated`, `verified` or `sources` but no `type`, the tool reports `missing-type` (`[spec]`, because §4 makes `type` REQUIRED). Agents always write `generated`, while Obsidian, Hugo and skill files never carry those keys (third review, B2). Checklists and instructions can live in a bundle as ordinary files, or be concepts with a free-text `type` such as `"Checklist"`.
6. **`exclude` exists, both as a `.okf-check.yaml` key and as a repeatable `--exclude` flag** (revised 2026-11-06, superseding "no exclude list").
   - **Paths** are relative to the bundle root, and an excluded path is never walked. Its files stay on disk, so `dangling-resource` still resolves citations into it.
   - **The flag adds to the config,** and the report header prints every exclusion.
   - **Why it came back:** io-insights' exporter checks only the files it writes, not the analysed repository's files it copies into `references/` (third review, B1). It passes `--exclude references/`, so a generated bundle never needs a config file of its own.
   - **okf-check stays generic:** it knows nothing about `references/` or any producer. Deciding by `generated.by` was rejected because it would make the tool know producer identities.
7. **Two house rules close the gaps this leaves:**
   - a concept in a directory without an `index.md` is a finding, which catches a new subdirectory whose index was forgotten;
   - an `.md` file linked from an `index.md` must be a concept, which catches a listed document that lost its `type` or never had front matter.

   What this flags today: io-insights' `docs/README.md` and `docs/product/README.md`, both linked from their indexes and neither with front matter. In yz-docs, `missing-index` would flag `compiler/README.md` and `compiler/examples/multi_root/README.md`, both `type: readme`. They lose their whole front matter and become ordinary files (decision 19).
8. **okf-check checks less than spec rule 1 says, and the catalogue says so.** A strict OKF consumer treats every `.md` file in a bundle as a concept; okf-check does not. `no-front-matter` no longer fires, and the link rule in decision 7 takes its place. `missing-type` fires only for front matter that looks like OKF (decision 5).

**Rules and labels** (resolves Q6, Q7, Q11, Q14 and themes 5–6)

9. **Every finding gets one of three mutually exclusive labels.** This is the narrow reading of rule 3.
   - **`[spec]`, the file breaks OKF itself:**
     - front matter that opens but does not parse;
     - front matter in an `index.md` other than the root's `okf_version` (§8);
     - a log heading that starts with a date but is not exactly `YYYY-MM-DD`, such as `## 2026-09-28 (6)` (§9);
     - `generated-missing-by` (spec line 377, `generated.by` is REQUIRED) and `source-missing-resource` (line 305, `sources[].resource` is REQUIRED);
     - `missing-type`, for front matter that looks like OKF but has no `type` (decision 5; line 165, `type` is REQUIRED);
     - `unreadable`, a `.md` file that opens front matter but cannot be decoded as UTF-8, or that cannot be opened at all (§4, line 155: "Every concept is a UTF-8 markdown file"; §11 rule 1). Decision 51 defines the scope.
   - **`[integrity]`, the file publishes a malformed trust signal:** `bad-actor`, `bad-instant`, `duplicate-source-id`, `generated-not-mapping`, `verified-not-mapping`, `invalid-description`. These are the trust half of today's `FATAL_CODES`. The word "trust" was rejected because the spec already uses it 23 times, for trust signals and the trust tier.
   - **`[house]`, the file is untidy by the author's rules:** everything else, including a log with entries but no date heading at all, newest-first logs, `title-heading-mismatch`, `dangling-resource` and `footnote-without-source`.
10. **House rules are on by default.** They are the author's own rules, wanted in every project, so there is no `spec` preset.
11. **Keep today's finding codes.**
12. **List every concept in its own directory's `index.md`.** The rule is completeness, not reachability.
13. **Gate log entries for new documents only.** `missing-log-entry` fires only for a document that is new relative to `main`. Changes to existing documents are never gated.
    - **The reversal:** this partly reverses io-insights' stable `docs/design/log-entry-checks-not-gated.md`. Its title becomes scoped to changes, and the amendment carries its own log entry. The reason: a creation always has a non-routine answer to "why does this exist?", which is AGENTS.md §16's rule.
    - **Guide §4:** "Earn the entry" becomes "Make the entry worthwhile", and "Routine additions do not" becomes "Routine changes do not".
    - **The residual risk:** a gate cannot stop a vague `Added X.` entry. Only review catches it.

**Blocking and config** (resolves Q2, Q10 and theme 4)

14. **Set one blocking level per bundle: `block: spec | integrity | house | none`, default `spec`.** The levels nest: `spec` ⊂ `integrity` ⊂ `house`.
    - `spec` fails on `[spec]` findings.
    - `integrity` fails on `[spec]` and `[integrity]` findings.
    - `house` fails on all three.
    - `none` reports everything and exits 0, for a first run on a new repository.

    Findings below the chosen level are still reported. Repository docs use `house` in `.okf-check.yaml`, the same locally and in CI. io-insights' exporter passes `--block=integrity` for the bundle it generates. That bundle has a different author, an LLM, and no one present to fix an untidy finding, so refusing it over one would bring back the 23-concept refusal that `FATAL_CODES` was written to stop. This replaces `FATAL_CODES` with a single scale.

    Rejected alternatives:
    - withholding only the failing concepts, which breaks the links and index rows that point at them;
    - running the exporter at `spec` with trust checks kept in Python, which brings back two checkers.
15. **Store the level in `.okf-check.yaml` at the repository root.** The tool finds it by walking up from the bundle being checked to the repository root (decision 40), and `--block` overrides it for one run.
    - **Why a file:** an agent running bare `okf-check` gets the same result as CI, which a flag in each gate script cannot guarantee.
    - **Why YAML:** the binary already parses YAML under the agreed profile, and YAML has comments. JSON has no comments, and TOML adds a second parser.
    - **Why not `.okf-check.md`:** it would put a config file inside the bundle it configures, and invite rule explanations that belong in the catalogue.
    - **One name only:** a `.okf-check.yml` is a usage error, not something the tool silently ignores.
16. **Move `forged-human-verifier` into io-insights' exporter.** The exporter is the only caller that knows which documents are carried forward, and §7 does not forbid a program from writing `human:`, so the rule belongs to io-insights. okf-check has no `--generated` flag.
17. **Switch the exporter to okf-check in v1.** The exporter runs okf-check on its staging directory at `--block=integrity` (decisions 14 and 23). One checker, not a Python copy of the rules, is the payoff. The cost is shipping the binary inside the io-insights product.

**YAML profile** (resolves most of Q3)

18. **Use the YAML 1.2 core schema, and keep instants as datetimes.** `yes` and `no` are strings. For the instant keys (`generated.at`, `verified[].at`, `sources[].last_modified`, `stale_after`, `usage_window.from`, `usage_window.to`, and the legacy `timestamp`), the tool checks the source text for an explicit offset first, then converts the value to a datetime. Readers still get datetimes, and a loose timestamp that PyYAML resolves no longer passes.

**Consumers, fixes and contract**

19. **The first consumer is yz-docs.** Its findings on the first run were re-checked on 2026-11-06:
    - **`missing-index`, 2:** `compiler/README.md` and `compiler/examples/multi_root/README.md`. They lose their whole front matter and become ordinary files. Removing only `type` would leave `generated` without `type`, which triggers `missing-type` (decision 5).
    - **Index front matter (§8), 2:** root `index.md` (`type: index`) and `spec/index.md` (`type: spec`).
    - **Bare actors (§7):** 321 `generated.by: "oscarryz"` and 2 `verified.by: "oscarryz"`. The two verified ones become human-reviewed under the mapping, which is correct only because the author did review them.
    - **Log headings (§9):** `docs/Implementation/log.md` has `## 2026-09-28 (6)` (`[spec]`), and `docs/log.md` uses `2026-09-10:` lines with no headings (`[house]`). Both are fixed by hand.
    - **`unlisted-concept`, 2:** in `docs/Examples/`, which has 141 docs and 139 index links.

    **Where its gate runs:** yz-docs has no CI, and its only `go.mod` is `compiler/go.mod` at `go 1.23`, while the bundle root is the repository root. A new GitHub Actions workflow and local runs both use the same pinned-download script as io-insights (decision 25). No Go and no `compiler/` change are needed.
20. **Add `--fix` for fixes with one correct answer only.** The index front matter and actor-mapping cases ship in Stage 1, because yz-docs cannot reach clean without them, and the rest after v1's checks.
    - Index front matter becomes `okf_version` in the bundle-root index only. A nested index just loses its front matter, so `--fix` never splits a bundle (third review, M9).
    - Actors change only through an explicit mapping (`--actor <name>=human:<name>`), never an invented `human:`.
    - Fixes are targeted text edits, never re-serialised YAML.
    - Ambiguous cases, such as a timestamp without an offset, are left to the author.
21. **The contract is the fixtures, the rule catalogue and the JSON output schema.** Fixtures live in `pass/<name>/` and `fail/<rule-code>/`, each with an `expected.json` that exists only in the corpus, never in a bundle.
22. **Retire okf-drift.** Its `index --check` becomes decision 12. Its `check` command and the log merge check are v2.

**Exporter and distribution** (resolves Still open item 1 of the first pass)

23. **The exporter writes the complete new bundle to a staging directory, runs okf-check on it, and replaces the old bundle only if it passes.**
    - **What "complete" means:** concepts, the old concepts `_refused_prune` keeps, `references/`, `index.md` and `log.md`. okf-check then checks exactly what ships, on disk, the same way CI does.
    - **What it replaces:** a stdin protocol was rejected. `write_bundle` builds `index.md`, `log.md` and `references/` after its current check point, so a stdin copy would be a partial bundle. Making it whole would also need a protocol contract.
    - **The second caller:** `publish_okf.py` is out of v1. Its `main()` is gated closed (`docs/design/repo-publish-is-gated.md`), and it adopts the staging step when that path reopens (third review, B4).
    - **Unverified:** whether the final replacement can be an atomic rename for every target. That holds only within one filesystem.
24. **If the binary is missing, the export is refused.**
25. **Distribute okf-check only through GitHub Releases on `oscar-io/okf-check`. There is no PyPI or npm package.**
    - **Releases:** GoReleaser attaches binaries for each platform, plus a checksums file, to each tag.
    - **Go projects:** `go install` or `go tool` builds from source through the module proxy, pinned in `go.mod`.
    - **io-insights:** a script holds the pinned version and checksum, and downloads the release on first use. That keeps okf-check out of AGENTS.md §1's runtime dependencies, needs no Go toolchain, and stays consistent with okf-drift's `distribution.md`, which rejects claiming a global registry name.
    - **The frozen build** bundles the downloaded binary at build time, the way `scripts/build-macos` does for `codebase-memory-mcp`. `uv sync` installs fetch it on first export. Neither ever prefers a copy on `PATH`.
    - **macOS quarantine:** binaries fetched by a script (curl, urllib, Go's HTTP client) get no quarantine attribute, so no approval is needed. The Go linker ad-hoc signs `darwin/arm64` binaries, which Apple Silicon requires. Inside the frozen app, okf-check behaves as `codebase-memory-mcp` does today. Only a manual browser download is quarantined.
26. **The gate fails when the binary is missing.** `scripts/check` runs the pinned download and fails if it cannot get the binary. Copying how it skips the JS checks when Node is absent would let the gate pass without running.
27. **Signing adds no new requirement.** `scripts/build-macos` does not sign the frozen build today. Signing becomes a question for okf-check only when the app is signed.

**YAML profile, continued**

28. **A duplicate key is a `[spec]` `unparseable-front-matter` finding.** YAML 1.2 requires mapping keys to be unique. PyYAML accepts duplicates silently, so okf.py's checking path must reject them. The Go library is `go.yaml.in/yaml/v4`, whose `WithUniqueKeys` is on by default. v4.0.0 was released on 2026-11-04. Whether the option also applies when decoding into `yaml.Node`, which findings need for line numbers, is unverified, and a fixture covers it.

**Codes, docs and v2**

29. **New codes (proposed):**
    - `missing-index`: a directory holds concepts but no `index.md`. Reported once per directory.
    - `listed-non-concept`: an index links an `.md` file that is not a concept.
    - `unlisted-concept`: a concept is missing from its directory's `index.md`.
    - `missing-log-entry`: v2, for the log merge check.
30. **Remove the links to io-insights' two READMEs from their indexes.** `test_docs_landing_files_each_keep_one_job` in `tests/test_documentation_contract.py` currently requires every non-reserved `.md` under `docs/` to be reachable from an index, READMEs included. Its set must narrow to concepts, or the removal fails the gate.
31. **v2 drift and log rules.**
    - Drift anchors on the newest `verified` event.
    - The log merge check lives in okf-check, and `(#<PR>)` is optional.
    - An entry's date must fall between the merge-base commit date and the day of the check, since the landing date is unknown before merge.
    - When the merge base is unavailable, as in a shallow clone, the check exits 2 and names `fetch-depth: 0`. It never passes.
32. **Success measures, with dates from the effort estimates in Staging.**
    - **Stage 0, by 2026-11-27:**
      - the okfcli trial is written up;
      - the corpus runs against okf.py, with every expected difference in the ledger (decision 36);
      - today's `scripts/okf-validate` time on `docs/` is recorded as a baseline.
    - **Stage 1, by 2027-02-19:**
      - okf-check is released;
      - yz-docs is gated at `block: house` and clean;
      - the agent test in decision 44 passes.
    - **Stage 2, by 2027-03-19:**
      - io-insights' docs are gated at `house` locally and in CI;
      - the only differences from `validate_document` are ledger entries;
      - a full check of `docs/` is no slower than the baseline.
    - **Stage 3, by 2027-04-16:**
      - io-insights' exporter checks its staging directory with okf-check;
      - the Python checker is deleted (the reader stays).

**Paths and exported bundles** (from the second review, 2026-11-06)

33. **A path that leaves the bundle is relative.**
    - **The problem:** spec §6.1–6.2 says `/` means the bundle root. io-insights' 440 `resource:` values starting `/src/`, `/tests/` or `/scripts/` (in 163 files) therefore dangle once `docs/` is the root.
    - **The fix:** they become `../`-relative paths, which is one of the three §6.2 forms.
    - **The rule changes:** `non-bundle-relative-resource` (`core/okf.py`) becomes "a path inside the bundle uses `/`; a path leaving it is relative". `dangling-resource` checks both forms on disk.
    - **The rewrite:** `--fix` rewrites them. okf-check exists to bring non-compliant files into line.
    - **Rejected:** commit-pinned URLs, which cannot be resolved or checked offline; and making the repository root the bundle root, which makes the repository the bundle rather than its docs.
34. **io-insights' exporter writes an `index.md` in every directory that holds concepts,** including `kb/` and `sections/`. `references/` gets none, because it is excluded (decision 6). Today it writes only the root one (`export_okf.py`). The author is filing a ticket.

**Second review, Major findings** (2026-11-06)

35. **Index entries follow okf-drift's semantics** (`src/index-cmd.ts`).
    - **What counts:** an entry is a link in a list item, or in a table row with any number of columns, under any heading.
    - **Navigation is ignored:** links to `log.md` and `index.md` are not entries.
    - **How targets resolve:** a leading `/` resolves from the bundle root, anything else from the index's directory. Fragments and queries are stripped, and percent-encoding is decoded.
    - **Subdirectory entries:** `subdir/` is one, and so is `subdir/index.md`. It is checked only for existence.
    - **Outside the bundle:** a target such as `../../AGENTS.md` is exempt from `listed-non-concept`.
36. **Measure parity with a ledger, not "zero differences".** Every difference between okf-check and the Python checker must trace to a decision that changes findings on purpose, and anything else is a bug. Each fixture records which implementation it must pass at each stage.
37. **`.okf-check.yaml` carries `version`.**
    - **It is required.** A missing `version` is a usage error, because the latest release cannot be known offline (third review, M1).
    - **The binary refuses to run** when its own version differs from the pin.
    - **Rollback** is lowering the pin.
    - **The corpus** ships as a release asset at the same version, and io-insights' reader tests fetch it at that version.
    - **Integrity of the binary is the consuming project's job:** `go.sum` for Go projects, the lockfile's hashes for Python, and a committed SHA-256 in any download script. A version alone can be hijacked by re-publishing.
    - **Stability policy:**
      - new rules and level moves are minor, because the pin protects consumers;
      - removing or renaming a code is major;
      - patch releases never change findings.
38. **The `[spec]` test is a spec MUST or REQUIRED sentence, or one of §11's conformance conditions, cited in the catalogue.** `bad-actor` is therefore `[integrity]`, not `[spec]`. The actor forms are a convention, §7's only MUST is that hand-authored content uses `human:`, and the tool cannot tell whether a bare name is a person.
39. **Add okf-drift's index codes:**
    - `dangling-entry` (`[house]`; the spec tolerates broken links);
    - `description-drift` (`[house]`);
    - `invalid-description` (`[integrity]`);
    - `unreadable` (`[spec]`): the file opens front matter but cannot be decoded as UTF-8, or cannot be opened at all, so it cannot be classified (decision 51). It is distinct from `unparseable-front-matter`, where the text decodes but its YAML does not parse;
    - `undescribed-document` (`[house]`).

    `write-refused` waits for `--write`. `no-front-matter` is retired, and `listed-non-concept` replaces it. `missing-type` stays, as `[spec]`, for OKF-looking front matter (decision 5). The two description codes add about 300 findings in yz-docs, and the author accepts that cleanup. A code table records every kept, renamed, retired and added code.
40. **Config lookup starts at the bundle being checked and walks up to the repository root,** the directory holding `.git`. The exporter always passes `--block` and ignores config files. An unknown key is a usage error.
41. **okf-check checks bundles, never loose files.** `/finalize-branch` checks the whole `docs/` bundle instead of changed files, so existing findings block a branch until they are fixed. A lighter per-file mode is a later enhancement.
42. **Text output puts the label first, padded to the widest label, then location, severity, code and message:**
    ```
    [house]     docs/a.md:12  warning  title-heading-mismatch: …
    [integrity] docs/b.md:3   error    bad-instant: verified.at needs an explicit offset (got 2026-08-05 09:00)
    [spec]      docs/c.md:1   error    unparseable-front-matter: …
    ```
    - **Severity:** `error` blocks at the current level, and `warning` is reported only.
    - **Editors:** most problem matchers expect `path:line:` at the start of a line, so editor integration reads the JSON output.
43. **Add the new finding fields (`got`, `expected`, `key`, `fix`) to `REDACTED_FIELDS`** (`core/run_events.py`) when the exporter forwards them. Run events go to a local `events.jsonl` in a 0700 output directory, never to git, so redaction guards against secrets in free text, not against publication.
44. **The agent success test seeds failures at all three levels,** including a new document without a `type`.
    - **A pass** is a correct fix, not exit 0.
    - **`explain bad-actor`** says only a person may supply `human:`, and an invented `human:` fails the test.
45. **House rules may fail the spec's own examples.** That is why they are house rules. In the fixtures, the spec's examples are pass cases at `block: spec`. `bad-implementation` stays, and fires only when an `implementation` key is present.

46. **A prior the reader cannot parse is read-only and reported, never overwritten.** `okf_read.py` skips such a document today, so it escapes custody (`okf_reconcile.py`). That applies to any unparseable front matter, not just duplicate keys. A scan on 2026-11-06 found no duplicate keys in 543 io-insights or 357 yz-docs front-matter blocks.

47. **A run that finds no bundle is an error** (exit 2), never a pass. That includes the exporter's check of an empty staging directory (third review, M11).
48. **The okfcli trial informs, it does not gate** (third review, M18).

49. **In Stage 2, okf-check takes over io-insights' docs check, and `RepositorySweepTests` (`tests/test_okf_frontmatter.py`) is retired.** That test runs the Python checker over `docs/`, and that checker flags the `../` paths decision 33 introduces. After Stage 2, the Python checker serves only the exporter, until Stage 3 deletes it (fourth review, M3).
50. **In Stage 2, io-insights' reader adopts decision 4's bundle-root rule, held to shared discovery fixtures** (third review M12, moved out of the draft by the fourth review, M4). Today `okf.bundle_roots` infers the repository root `.` for io-insights, because typed files such as `AGENTS.md` sit there, while okf-check's root is `docs/`. After the path rewrite the reader would silently drop every rewritten `/…` citation.

51. **Classify a file from its raw first line, and decode only files that open front matter.**
    - **How:** okf-check reads the first line without decoding it. If that line is `---` (after an optional BOM), the file opens front matter, and the rest is decoded as UTF-8. Otherwise the file is ordinary and is never decoded, so a Latin-1 README is ignored exactly like a UTF-8 one (decision 8).
    - **`unreadable` fires only** when the rest of a file that opens front matter fails to decode, or when a `.md` file cannot be opened at all, for example because of permissions. That file cannot be classified, and skipping it silently is the failure this design avoids.
52. **Accept the harmless byte-level variations, and report the one that loses a document.** The shared fixtures hold the Python reader to the same results. The reader changes land in Stage 2, and until then the ledger records the difference (decision 36).

    | Case | Result |
    |---|---|
    | UTF-8 BOM before `---` | Stripped, no finding. Today the Python reader skips such a file silently |
    | CRLF line endings | Normalised, no finding |
    | Spaces or tabs after the opening or the closing `---` | Accepted as the delimiter, no finding. okf-drift already does this; Python requires exactly `---\n` today |
    | Front matter opened but never closed | `unparseable-front-matter` (`[spec]`), with a message saying it was never closed. Today Python treats it as having no front matter, so the document silently becomes an ordinary file |

    No new code is minted for the unclosed case. The message tells the agent what to fix.

### Still open

1. **The okfcli trial** (Stage 0). It informs decision 1 and does not gate it (48).

## Goals vs. Non-Goals

* **In scope for v1:**
  * **The binary:**
    * discovery (decisions 4–8);
    * the three labels and the `[spec]` test (9, 38);
    * `block` levels (14);
    * `.okf-check.yaml` with `version` and `exclude`, looked up from the bundle, plus `--exclude` (6, 15, 37, 40);
    * bundles only (41);
    * padded text output and JSON output (42);
    * `explain`;
    * exit codes 0 clean, 1 blocking, 2 usage error.
  * **Rules:**
    * today's rules minus `forged-human-verifier` (16);
    * the reserved-file checks (9);
    * the index rules and codes (7, 12, 29, 35, 39);
    * the out-of-bundle path rule (33).
  * **The YAML profile** on `go.yaml.in/yaml/v4` (18, 28).
  * **`--fix`** for index front matter, `--actor` mapping and out-of-bundle paths (20, 33).
  * **Contract and docs:**
    * the fixture corpus and the ledger (21, 36);
    * the rule catalogue and the code table (39);
    * a generic authoring guide and a procedure-only skill that cite codes, with a test that every cited code exists and has a fail fixture.
  * **Distribution:** GoReleaser binaries on GitHub Releases, and a download script that checks a committed checksum (25, 26).
  * **Consumers:**
    * yz-docs (19);
    * io-insights' docs (30);
    * io-insights' exporter (17, 23, 34, 43, 46). `publish_okf.py` stays gated and out of v1 (23).
* **Out of scope for v1:**
  * The remaining `--fix` cases, and `--write`.
  * npm, brew and PyPI packages.
  * Windows builds (proposed: none until someone asks).
  * Native readers in other languages.
  * A per-file mode (41).
  * okf-drift's `check`, replacing `core/okf_travel.py`, the log merge check and `missing-log-entry` (all v2).
  * Per-rule severity beyond `block`, a `spec` preset and `--generated`.

## Staging

Effort is in focused days: the author's own time, including reviewing agent-written code. Calendar dates assume about **2 focused days a week** and two weeks off over the year-end holidays. Treat each estimate as ±50%. The riskiest items are marked ⚠.

**Stage 0, no Go: about 6 days, by 2026-11-27.**

| Task | Days |
|---|---|
| okfcli/okf trial on io-insights' `docs/` and yz-docs. It informs decision 1 and does not gate it (48) | 1 |
| Corpus layout, converting `tests/test_okf_frontmatter.py` and the index and log checks in `tests/test_documentation_contract.py` (not the `merge_log` writer tests), plus the ledger | 2 |
| The YAML profile written as a table (input text, parsed value, finding), plus the divergence fixtures | 1 |
| The code table and catalogue skeleton, with spec citations | 1 |
| io-insights prep: move `forged-human-verifier` into the exporter, and apply the read-only rule for unparseable priors (46) | 1 |

**Stage 1, the binary and yz-docs: about 19.5 days, by 2027-02-19.**

| Task | Days |
|---|---|
| Repository scaffolding, CLI synopsis, config lookup, `block`, `exclude`, exit codes | 2 |
| YAML v4 loader, the profile, line numbers | 1.5 |
| Discovery: skip set, `.gitignore`, `okf_version` boundaries | 1.5 |
| Port about 25 existing rules | 3 |
| Index rules with okf-drift's semantics, and the reserved-file checks ⚠ | 2.5 |
| Text and JSON output, `explain` | 1.5 |
| `--fix` for index front matter and `--actor`, as targeted text edits | 1.5 |
| GoReleaser (darwin and linux, amd64 and arm64), checksums, corpus asset, download script | 1.5 |
| The generic guide, the skill, the cited-code test | 2 |
| yz-docs: CI workflow, run at `none`, apply decision 19's fixes and the description-code cleanup (39), gate at `house` | 1.5 |
| The agent test (44) | 1 |

**Stage 2, io-insights' docs: about 8 days, by 2027-03-19.**

| Task | Days |
|---|---|
| Download script wired into `scripts/check`, failing when the binary is missing | 0.5 |
| `--fix` for paths: rewrite out-of-bundle `/src`, `/tests` and `/scripts` paths to `../` form, and in-bundle `/docs/…` paths (289) to `/…` | 1 |
| io-insights' reader learns the `../` form (`okf_read.resolve_repo_path` returns `""` for it today) | 1 |
| Clean `docs/`: rewrite the 440 out-of-bundle paths with `--fix`, add index rows, remove the README links and narrow the contract test (30) | 2 |
| Ledger comparison against `validate_document` | 1 |
| okf-check takes over the docs check, and `RepositorySweepTests` is retired (49) | 0.5 |
| The reader adopts decision 4's root rule, held to shared discovery fixtures (50) | 1 |
| The reader strips a BOM, accepts CRLF and whitespace after either delimiter, and reports an unclosed block instead of treating the file as ordinary (52) | 0.5 |
| Turn `scripts/okf-validate` into a thin wrapper and update `/finalize-branch` | 0.5 |

**Stage 3, the exporter switch: about 8.5 days, by 2027-04-16.**

| Task | Days |
|---|---|
| `write_bundle` writes the complete bundle to staging, checks it, then replaces the old one ⚠ | 3 |
| An `index.md` in every exported directory (34, the author's ticket) | 1.5 |
| Binary in the frozen build: download in `scripts/build-macos`, universal macOS binary, runtime lookup, frozen-build test | 2 |
| Run-event mapping and redaction (43); `--exclude references/` | 0.5 |
| Delete the Python checker, and rewrite the tests that import `CONFORMANCE_CODES` and `FATAL_CODES` | 1.5 |

**Totals:** about 42 focused days for v1. The exporter switch is about a quarter of that. The author accepts that there is no buffer (third review, M13).

**Later.** The remaining `--fix` cases, then v2 (okf-drift's `check` and the log merge check, then archiving okf-drift).

**Success exam:** decision 32.
