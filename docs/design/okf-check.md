---
type: Design
title: okf-check
description: "Design for okf-check, a Go binary that checks OKF v0.2 bundles, and its adoption by yz-docs and io-insights"
status: draft
generated:
  by: claude-code/claude-opus-5-5
  at: 2026-11-06T15:37:45Z
verified: { by: "human:oscar-io", at: 2026-11-06T16:40:00+01:00 }
---

# okf-check

*   **Author:** Oscar Reyes

--------------------------------------------------------------------------------

## Overview

okf-check is a single Go binary, published from `oscar-io/okf-check`, that
checks documents written in Open Knowledge Format (OKF) v0.2. It reads a
documentation bundle from disk, reports each problem with a label, a location
and a suggested fix, and exits non-zero when a problem at or above the bundle's
blocking level is found.

It exists so that the OKF rules, test fixtures and authoring guidance built up
inside io-insights can be used in other projects, most of them Go, where AI
agents write the documents. In those projects the checker's output is the main
way an agent learns what it got wrong, so the output is designed for an agent
to act on.

v1 has three consumers, adopted in this order:

1.  **yz-docs**, a Go project with no CI today, gated at `block: house`.
2.  **io-insights' own `docs/` bundle**, where okf-check takes over the docs
    check from the Python validator.
3.  **io-insights' exporter**, which checks each generated bundle in a staging
    directory at `--block=integrity` before replacing the published one.

At the end of v1, io-insights' Python checker (`validate_document` in
`src/io_insights/core/okf.py`) is deleted. Its Python reader (front-matter
parsing, trust tier, verification events, bundle roots) stays, and is held to
the same fixtures as okf-check. okf-drift's `index --check` is superseded by
okf-check's index rules. okf-drift itself is retired after v2.

This document covers all three repositories because v1's success is measured in
the consumers, not in the binary alone. The okf-check repository is specified
in full. The io-insights and yz-docs changes are specified to the level needed
to adopt it.

The decisions behind this design are recorded in `concepts_brief.md`
(decisions 1–52) and its four reviews. Where this document cites "decision N",
it refers to that brief.

## The Problem

**OKF checking is locked inside one Python project, and the two checkers that
exist disagree.**

*   **No checker for other projects.** io-insights' rules live in
    `core/okf.py`, called in-process by its exporter and its test suite. A Go
    project such as yz-docs cannot run them without installing io-insights.
    yz-docs has 357 front-matter blocks and no gate at all, so its first run
    surfaces more than 320 bare `generated.by` actors, front matter in two
    index files, and malformed log headings that nothing caught.
*   **Two checkers, two answers.** io-insights' Python checker and okf-drift's
    TypeScript `index --check` cover different rules and disagree where they
    overlap:
    *   Python counts any `by` starting with `human:` as a human verifier, even
        when malformed. TypeScript drops it. `human:Oscar Reyes` is
        human-reviewed in one and unverified in the other, and io-insights'
        reconcile step decides overwrite custody from that answer.
    *   They name the same problem differently (`bad-actor` vs
        `malformed-trust`).
*   **The YAML parser decides some results by accident.** PyYAML implements
    YAML 1.1, so:
    *   `type: no` is read as the boolean `false` and reported as a missing
        type;
    *   a loose timestamp such as `2026-8-5 9:00:00 +2` passes when unquoted
        and fails when quoted;
    *   a duplicate key keeps the last value without any report.

    The Python reader also loses whole documents without saying so: a file
    that starts with a UTF-8 BOM, or whose front matter is never closed, is
    treated as having no front matter and is never checked.

    None of these is a rule anyone chose. A second implementation would choose
    differently unless the profile is written down and tested.
*   **Findings do not tell an agent where or how to fix the problem.** A
    finding today holds only a code, a message and a path. `bad-instant` can
    come from four different keys, and `bad-actor` from `generated.by` or any
    `verified[].by`, with no line, key, expected form or fix. An agent fixing
    the wrong key, or deleting an index link to silence a finding, is the
    predictable result.
*   **Blocking is a separate, hidden axis.** The exporter withholds a bundle
    on `FATAL_CODES`, a set that cuts across the spec-vs-house split.
    `duplicate-source-id` is a house rule and still fatal. Nothing in a finding
    says whether it blocks, so the test script prints "the documents remain
    OKF-conformant" and still exits 1.
*   **Each rule is written down in up to five places.** io-insights' authoring
    guide, okf-drift's authoring guide, io-insights' AGENTS.md, its skill file,
    and each consuming project's own instructions. The io-insights guide is
    already wrong in places (it claims `okf.py` implements all three
    conformance rules, and cites the wrong spec sections).

## Goals

Each goal is met when the Stage exit it names in decision 32 is met. Dates
assume about two focused days a week and are ±50%.

1.  **One checker that runs in any project without a language runtime.**
    okf-check is a single static binary for darwin and linux on amd64 and
    arm64, installed from GitHub Releases by a pinned download script with a
    committed checksum, or with `go install` / `go tool` in Go projects.
    *Measured by:* Stage 1, by 2027-02-19. okf-check is released and yz-docs
    runs it in a new GitHub Actions workflow and locally.
2.  **Every finding says what is wrong, where, and how to fix it.** Each
    finding carries a label, file and line, the key path (for example
    `verified[1].at`), the value found, the expected form and a one-line fix.
    `okf-check explain <code>` prints the rule entry offline.
    *Measured by:* the agent test (decision 44). Failures are seeded at all
    three labels, including a new document without a `type`, and the test
    passes only on a correct fix, not on exit 0. An invented `human:` actor
    fails it.
3.  **One label per finding and one blocking level per bundle.** Every finding
    is `[spec]`, `[integrity]` or `[house]`. Each bundle sets
    `block: spec | integrity | house | none` in `.okf-check.yaml`. This
    replaces `FATAL_CODES` with a single nested scale.
    *Measured by:* the code table assigns every code exactly one label, and
    each `[spec]` row cites the spec MUST or REQUIRED sentence, or the §11
    conformance condition, behind it (decision 38).
4.  **A written YAML profile that okf-check and io-insights' reader both
    follow.** YAML 1.2 core schema, duplicate keys rejected, instants checked
    on their source text for an explicit offset.
    *Measured by:* Stage 0, by 2026-11-27. The fixtures run against `okf.py`,
    and every difference is a ledger entry traced to a decision (decision 36).
5.  **io-insights' docs and exporter use okf-check, and its Python checker is
    deleted.**
    *Measured by:*
    *   Stage 2, by 2027-03-19. `docs/` is gated at `house` locally and in
        CI. The only differences from `validate_document` are ledger entries.
        A full check is no slower than the Stage 0 baseline.
    *   Stage 3, by 2027-04-16. The exporter checks its staging directory with
        okf-check, and the Python checker is deleted (the reader stays).
6.  **Each checkable rule is written down once.** The rule docs
    (`docs/rules/`) hold every rule with its spec citation, a bad and a good
    example, and the fix.
    The authoring guide holds judgement only, and the skill holds procedure
    only. Both cite codes rather than restating rules.
    *Measured by:* a test that fails when any doc cites an unknown code, or
    when a code has no fail fixture.

## Non-Goals

*   **Full spec rule 1 conformance.** Spec §11 rule 1 treats every
    non-reserved `.md` in a bundle as a concept. okf-check checks only files
    whose front matter has a `type`, or that look like OKF (decisions 5 and 8),
    and the rule docs say so.
*   **Checking files outside a bundle,** such as io-insights' `AGENTS.md`,
    and checking loose files. okf-check checks bundles only (decision 41).
*   **Knowing who produced a document.** okf-check has no `--generated` mode
    and never decides by `generated.by`. `forged-human-verifier` moves into
    io-insights' exporter (decision 16), and the exporter excludes
    `references/` by flag (decision 6).
*   **`publish_okf.py`.** It stays gated closed (`docs/design/repo-publish-is-gated.md`).
    It adopts the staging check when that path reopens (decision 23).
*   **Native readers in languages other than Python.** The fixtures make them
    possible, but none is planned.
*   **Packages other than GitHub Releases and the Go module.** No npm, PyPI or
    Homebrew package.
*   **Windows builds,** until someone asks.
*   **A per-file mode** for checking only changed files (decision 41).
*   **`--fix` beyond the cases one correct answer covers in v1:** index front
    matter, `--actor` mapping, out-of-bundle and repository-rooted paths, and
    merging duplicate log dates. Also no `--write`.
*   **Per-rule severity beyond `block`,** and a `spec` preset.
*   **v2 work:** okf-drift's `check` command, replacing `core/okf_travel.py`,
    the log merge check and `missing-log-entry`.
*   **Making OKF checking faster.** No speed problem is reported. The Stage 2
    baseline only guards against a regression.

## Open Questions

All thirteen design choices are settled. They are listed in
[Owner choices](#owner-choices). What remains:

**Open and not blocking**

*   **The okfcli trial** (Stage 0). It informs decision 1 and does not gate it
    (decision 48).

**Unverified facts the design relies on**

*   **`go.yaml.in/yaml/v4`'s `WithUniqueKeys` applies when decoding into
    `yaml.Node`,** which line numbers require (decision 28). If it does not,
    okf-check walks mappings itself (section 4). A fixture covers it either
    way.
*   **A strict PyYAML loader can stay on the C parser** (item 44). If it
    cannot, the pure-Python loader is about 7× slower on large front matter
    (`core/okf.py`'s comment on `_LOADER`), and the Stage 2 baseline decides
    whether that matters.
*   **`.claude/skills/finalize-branch/SKILL.md` needs the same edit** as the
    `.agents` copy (item 49). The two files differ, and only the `.agents`
    one calls `scripts/okf-validate`.

**Known limits, accepted**

*   **A crash midway through the exporter's write pass** leaves a partly
    written `markdown/`, as today (C11).
*   **An unparseable prior that no index lists** ships without being checked
    (C12).
*   **The frozen build's binary is checked only by `scripts/build-macos`.**
    No test covers the frozen build today (item 55).
*   **A `../` path is not portable** to OKF tools that resolve relative paths
    against the document's directory (C6).

## Critical User Journeys (CUJs)

### CUJ 1: An agent fixes its own document in yz-docs

*   **Given**: yz-docs has `.okf-check.yaml` with `version`, `block: house`,
    and the pinned download script. An agent has just written
    `docs/Features/Tuples.md` with `generated.at: 2026-11-06 10:00` (no
    offset) and has not added it to `docs/Features/index.md`.
*   **When**: the agent runs `okf-check` from the repository root.
*   **Then**: two lines are printed, one per finding, each starting with its
    label and `file:line`:
    *   `[integrity] … error bad-instant`, naming `generated.at`, the value
        found, the expected form (`YYYY-MM-DDTHH:MM:SS` plus `Z` or
        `±HH:MM`), and the fix "add the offset the time was written in";
    *   `[house] … error unlisted-concept`, located at
        `docs/Features/index.md`.

    The run exits 1. After the agent adds an offset and an index row, a rerun
    prints only the header and the count line, and exits 0.

### CUJ 2: An agent asks what a code means, offline

*   **Given**: the agent got `bad-actor` and cannot open URLs.
*   **When**: it runs `okf-check explain bad-actor`.
*   **Then**: the rule entry prints: label, spec citation, valid forms,
    a bad and a good example, and the fix. The entry states that only a person
    may supply a `human:` actor, and that the fix for a bare name the agent
    cannot attribute is `<producer>/<version>`.

### CUJ 3: First run on a repository with many findings

*   **Given**: a repository with an OKF bundle and no `.okf-check.yaml`.
*   **When**: the author runs `okf-check --block=none`.
*   **Then**: every finding is printed as a `warning`, a count line gives
    totals per label, and the run exits 0. The author then writes
    `.okf-check.yaml` and fixes findings until `block: house` passes.
    `okf-check --fix --actor oscarryz=human:oscarryz` rewrites the bare actors
    and the index front matter as text edits, and leaves everything else
    alone.

### CUJ 4: CI gates a pull request

*   **Given**: the repository's workflow runs the pinned download script, then
    `okf-check`.
*   **When**: a pull request adds a document whose front matter opens but
    never closes.
*   **Then**: `[spec] … unparseable-front-matter` reports that the block was
    never closed. The job fails because `[spec]` blocks at every level except
    `none`. A missing binary or a checksum mismatch also fails the job, and
    never skips the check.

### CUJ 5: io-insights' exporter checks a generated bundle

*   **Given**: an export has written its complete new bundle to a staging
    directory under `<repo>/.io-insights/`. That directory is gitignored and
    sits inside a dot-directory.
*   **When**: the exporter runs
    `okf-check --no-config --block=integrity --exclude references/ --format=json <staging>`.
*   **Then**: okf-check finds the staging bundle even though it is under a
    dot-directory (see [Discovery](#2-discovery)). Files under `references/`
    are not checked, but citations into them still resolve. `[house]`
    findings come back as warnings and the run exits 0, so the exporter
    replaces the published bundle and forwards the warnings as run events. A
    `bad-actor` would exit 1, and the exporter would keep the old bundle.

## Technical Plan (Architecture)

### Overview

okf-check reads a directory tree and runs in one pass:

```text
 argv ──► 1. Resolve target, config and block level
              │   .okf-check.yaml (bundle root → repository root), flags win
              ▼
          2. Discovery: find bundle roots, walk each bundle
              │   skip set below the target, exclude, nested okf_version roots
              ▼
          3. Classification: per .md file, from the raw first line
              │   ordinary (never decoded) │ opens front matter │ unreadable
              ▼
          4. Parse front matter under the YAML profile
              │   yaml.Node tree with line numbers; unique keys; core schema
              ▼
          5. Rules
              │   document rules ─ reserved-file rules ─ index rules ─ path rules
              ▼
          6. Label and severity
              │   code table gives the label; block level gives error/warning
              ▼
          7. Report (text | JSON), exit code
              │
              └─► --fix: targeted byte edits to front matter, then re-run 2–7
```

The binary keeps no state between runs and uses no network. Every run reads
the files again. Whether that is fast enough is measured against the Stage 2
baseline, not assumed.

### 1. CLI, config and version pin

**Synopsis.**

```text
okf-check [flags] [path ...]        check the bundles at or under each path
okf-check explain <code>            print one rule entry
okf-check version                   print tool version, spec version, schema version

flags:
  --block spec|integrity|house|none   override the config's level for this run
  --exclude <path>                    repeatable; relative to each bundle root
  --no-config                         ignore .okf-check.yaml (the exporter uses this)
  --format text|json                  default text
  --fix                               apply one-answer fixes, then report what remains
  --actor <name>=human:<name>         repeatable; only with --fix
```

With no path, the target is the working directory. Flags use Go's standard
`flag` package. Repeatable flags are `flag.Value` types. No CLI library is
added.

**Exit codes.**

| Code | Meaning |
|---|---|
| 0 | No finding at or above the block level |
| 1 | At least one blocking finding |
| 2 | The check could not run: bad flags, bad or missing config key, `.okf-check.yml` present, version mismatch, no bundle found (decision 47), a target that does not exist, or an internal error |

An internal error (a recovered panic) also exits 2, with
`okf-check: internal error:` and the stack on stderr. Both mean "no verdict",
and every caller must treat 2 as a refusal, so a separate code adds nothing.

**Config file.** `.okf-check.yaml`, parsed with the same YAML profile:

```yaml
version: 1.4.0        # required; must equal the binary's version exactly
block: house          # optional; default spec
exclude:              # optional; each path relative to each bundle root
  - drafts/
```

*   **Lookup** (decisions 15 and 40): for each bundle, start at the bundle
    root and walk up to the repository root, meaning the nearest ancestor
    holding `.git` as a directory or a file (worktrees). The nearest
    `.okf-check.yaml` wins. Without `.git`, only the bundle root itself is
    searched. A `.okf-check.yml` anywhere on that walk is exit 2.
*   **Unknown keys** are exit 2 (decision 40).
*   **Several bundles in one run** each use their own nearest config. The
    report header prints the config path and level for each bundle.
*   **Version pin** (decision 37): the binary's version comes from
    `-ldflags -X` (GoReleaser), else from
    `runtime/debug.ReadBuildInfo().Main.Version`, which `go install …@v1.4.0`
    and `go tool` both set to the module version. A mismatch is exit 2, with
    both versions and the config path in the message. A development build
    reports `(devel)`, skips the pin check, and prints a warning to stderr, so
    okf-check's own CI can run `go run ./cmd/okf-check`.
*   **`--no-config`** skips the lookup and the pin check. Decision 40 says the
    exporter ignores config files, and without this flag a staging directory
    inside an analysed repository would pick up that repository's config and
    its pin. The exporter's binary is pinned by its own download script.

### 2. Discovery

**Bundle roots** (decisions 4 and 47).

1.  **Find the ceiling.** Walk up from the target. Stop at the repository
    root or at the first ancestor that is itself a skipped directory (a
    dot-directory, `node_modules/`, `vendor/`, or one matched by `.gitignore`),
    whichever comes first. A skipped ancestor bounds the search but is never
    applied to the target or its parents. That is why the exporter's staging
    directory under `.io-insights/` or `~/.cache/` is found. It also stops a
    staging directory inside yz-docs from being read as part of yz-docs'
    root bundle.
2.  **Look for an enclosing root.** From the ceiling down to the target, the
    outermost directory with an `index.md` whose tree holds at least one
    concept is the bundle root. A target inside a bundle checks the whole
    bundle (decision 41).
3.  **Otherwise look below.** Walk the target's subtree, applying the skip
    set. Each outermost qualifying `index.md` starts a bundle.
4.  **Nested roots.** Inside any bundle, an `index.md` that carries
    `okf_version` starts its own bundle. Its subtree is removed from the
    parent bundle and checked as a separate bundle with its own `/`.
5.  **No bundle** at any target is exit 2 (decision 47).

**`.gitignore`** (C4: ask git).

*   **When it applies:** only when the ceiling is the repository root, so
    that the bundle is reachable from the root without crossing a skipped
    directory. A bundle inside an ignored or dot-directory, such as the
    exporter's staging directory, is outside the repository's tree, and
    git's view of it would ignore every file in it. So no `.gitignore`
    applies there, and outside any repository nothing is ignored.
*   **The call:** once per repository root,
    `git -C <root> ls-files -z --others --ignored --directory --exclude-per-directory=.gitignore`.
    Paths are matched against its output.
*   **Only the repository's own `.gitignore` files count.** The call does
    not use `--exclude-standard`, which would add `.git/info/exclude` and the
    user's global excludes file. Those differ between machines, and decision
    15 requires a bare local run to match CI.
*   **Tracked files are always checked,** even when a pattern matches them.
    That is git's own rule: `.gitignore` affects only untracked files.
*   **git missing, or the call failing inside a repository, is exit 2,**
    with a message naming the repository. It never silently stops ignoring,
    because that would change findings without saying so.

**Walking a bundle.** Inside each bundle the walk skips dot-directories,
`node_modules/`, `vendor/`, `.gitignore` matches and `exclude` paths.
An excluded path is never walked, but its files stay on disk, so path rules
still resolve citations into it (decision 6). An `exclude` entry that matches
nothing in any bundle prints a warning, which catches typos. Symlinks are not
followed. A symlinked `.md` is listed with the ordinary files in the JSON
output, so it is visible without being checked.

### 3. Classification (decisions 5, 51, 52)

For every `.md` file in a bundle:

| Raw first line, after an optional UTF-8 BOM | Result |
|---|---|
| Cannot be opened | `unreadable` (`[spec]`) |
| `---` followed only by spaces or tabs, then LF or CRLF | Opens front matter: decode the rest as UTF-8 |
| Anything else | Ordinary file, never decoded |

A file that opens front matter is then:

*   **Not valid UTF-8:** `unreadable`.
*   **Never closed:** `unparseable-front-matter`, with the message "front
    matter opened on line 1 is never closed by a `---` line".
*   **Closed** by a line that is `---` followed only by spaces or tabs:
    *   CRLF is normalised to LF before parsing. Byte offsets for `--fix` are
        kept against the original bytes.

The parsed front matter of a non-reserved file decides what kind of document
it is:

| Front matter | Kind | Checked by |
|---|---|---|
| Fails to parse | Broken | `unparseable-front-matter` only |
| `type` is a non-empty string | Concept | all document, index and path rules |
| `type` key present but empty, null or not a string | OKF-looking | `missing-type` only (message names what was found) |
| No `type`, but has `generated`, `verified` or `sources` | OKF-looking | `missing-type` only |
| None of these keys | Ordinary | nothing |

The OKF-looking keys are exactly decision 5's three. `stale_after` and
`status` are left out because other tools' front matter uses `status`.

An OKF-looking file gets `missing-type` and nothing else (C2). No other
rule reads its front matter, and the index rules treat it as neither a
concept nor a non-concept. So an index row pointing at it raises no
`listed-non-concept`, whose fix would be to delete the link: the wrong
repair for a document that only lacks `type`. Once `type` is added, the
next run applies every rule. The agent test's untyped seed (decision 44)
therefore produces exactly one finding, and only adding `type` clears it.

`index.md` and `log.md` are never concepts. They get only the reserved-file
rules (section 5).

### 4. YAML profile (decisions 18, 28)

*   **Library:** `go.yaml.in/yaml/v4`, decoding into `yaml.Node` so that each
    finding can carry a line and a key path.
*   **Duplicate keys:** rejected as `unparseable-front-matter`, naming the key
    and both lines. If v4's `WithUniqueKeys` turns out not to apply to
    `yaml.Node` decoding (unverified), okf-check walks each mapping node and
    checks for duplicates itself. A fixture case covers this either way.
*   **Scalars:** YAML 1.2 core schema. `yes`, `no`, `on` and `off` are strings.
    Rules read values from nodes and never decode into `interface{}`, so the
    library's resolution of implicit timestamps never applies.
*   **Instants:** for `generated.at`, `verified[].at`,
    `sources[].last_modified`, `stale_after`, `usage_window.from`,
    `usage_window.to` and the legacy `timestamp`, the node's source text must
    be an RFC 3339 date-time with an explicit offset (`Z` or `±HH:MM`). A space
    is accepted in place of `T`, as RFC 3339 permits. This is stricter than
    §5's "ISO 8601 with an explicit UTC offset", and the rule docs say so.
    `bad-instant` reports the value found and the expected form. When the
    value already carries an offset, it also shows the normalised value, for
    example `2026-08-05T09:00:00+02:00` for `2026-8-5 9:00:00 +2`. It never
    suggests an offset the author did not write.
*   **The profile table.** The Stage 0 profile table (input text, parsed
    value, finding) is the normative statement. Each row is a fixture case
    that both okf-check and the Python reader run.

### 5. Rules and the code table

**Rule kinds.** Each rule has a code and one of four scopes:

| Scope | Input | Examples |
|---|---|---|
| Document | one concept's front matter and body | `bad-actor`, `bad-instant`, `duplicate-source-id`, `title-heading-mismatch`, `footnote-without-source` |
| Reserved file | one `index.md` or `log.md` | index front matter (§8), log date headings (§9) |
| Directory | one directory's concepts and its `index.md` | `missing-index`, `unlisted-concept`, `listed-non-concept`, `dangling-entry`, the description codes |
| Path | a path-valued field and the files on disk | `dangling-resource`, the out-of-bundle path rule (decision 33) |

A rule is a Go function that receives its input and returns findings. Rules
never read files themselves. Discovery and classification hand them parsed
inputs, so every rule can be tested on in-memory values.

**The finding.**

```text
Finding {
  code      string   // e.g. "bad-instant"
  label     string   // "spec" | "integrity" | "house", from the code table
  severity  string   // "error" | "warning", from label and block level
  file      string   // path relative to the working directory
  line      int      // 1-based; 0 for directory-level findings
  key       string   // front-matter key path, e.g. "verified[1].at"; "" if none
  got       string   // the value found, truncated to 120 characters
  expected  string   // the valid form(s)
  fix       string   // one line, imperative
  message   string   // one sentence, without the code
}
```

Directory-level findings (`missing-index`, `unlisted-concept`) are located
at the `index.md` path the agent must create or edit, with line 0.

**Severity.** `error` when the finding's label is inside the block level
(`spec` ⊂ `integrity` ⊂ `house`), otherwise `warning`. At `none`, everything
is a warning.

**Codes the brief names** (the complete table, with old names and spec
citations, is written in Stage 0 from `core/okf.py`):

| Label | Codes |
|---|---|
| `[spec]` | `unparseable-front-matter`, `unreadable`, `missing-type`, `generated-missing-by`, `source-missing-resource`, `computation-missing-runtime` (new, C9), index front matter (§8), log date heading not `YYYY-MM-DD` (§9) |
| `[integrity]` | `bad-actor`, `bad-instant`, `duplicate-source-id`, `generated-not-mapping`, `verified-not-mapping`, `sources-not-list`, `source-not-mapping` (C8), `invalid-description` |
| `[house]` | `title-heading-mismatch`, `dangling-resource`, `footnote-without-source`, `bad-implementation`, the out-of-bundle path rule, `missing-index`, `listed-non-concept`, `unlisted-concept`, `dangling-entry`, `description-drift`, `undescribed-document`, `duplicate-log-date` (new, C10), log without any date heading, log not newest-first |
| Retired | `no-front-matter` (replaced by `listed-non-concept`), `forged-human-verifier` (moved to io-insights' exporter) |
| Renamed | `no-index` → `missing-index`, `unlisted-document` → `unlisted-concept` (okf-drift names) |

`computation-missing-runtime` (C9) fires on a concept whose `type` is
`Attested Computation` (compared case-insensitively, with surrounding
whitespace trimmed) and whose `runtime` is missing, empty or not a string.
§10.2 marks `runtime` "REQUIRED for this type". It is the only REQUIRED
field in §10, so this one code completes decision 38's `[spec]` set for
§10.

`sources-not-list` and `source-not-mapping` are `[integrity]` (C8), because
a malformed `sources` value leaves the trust tier and drift detection
nothing to read, the same as a malformed `generated` or `verified`. If
either is missing from today's `FATAL_CODES`, it gets a ledger entry, and the
exporter can newly refuse on it. The Stage 0 code table records which.

**Index entries** (decision 35, with okf-drift's `src/index-cmd.ts` as the
base). The parts settled here:

*   **Destinations** are CommonMark: `[text](dest)` and `[text](<dest with spaces>)`.
    okf-drift does not unwrap angle brackets, and yz-docs has about 255 of
    them (third review, M6).
*   **Navigation**, which is never an entry: a target that resolves to the
    index itself, to any `log.md`, or to an `index.md` that is not in a
    direct subdirectory (such as `../index.md`). okf-drift matches only the
    exact strings `log.md` and `index.md`, so `plans/log.md` becomes a
    dangling entry there.
*   **Subdirectory entries:** a target ending in `/` or resolving to
    `<direct subdirectory>/index.md`. These are checked only for existence.
    This removes decision 35's apparent contradiction: `index.md` in the same
    directory is navigation, and one in a direct subdirectory is an entry.
*   **Out of the bundle:** resolution keeps `..` segments that climb above
    the root instead of clamping them as okf-drift does. Such a target is
    exempt from every index rule.
*   **Precedence:** a target that does not exist is `dangling-entry`. One that
    exists but is not a concept is `listed-non-concept`. A concept in another
    directory is a valid cross-link and produces no finding.
*   **Which links count** (C3): in a list item (`-`, `*` or `+`), only a
    link that begins the item's text, as in okf-drift. Text after it, links
    included, is the entry's description. In a table row, the first link in
    the first cell that holds one. Other cells supply the description only
    under a column headed `Description`, as okf-drift does. Link text may
    contain backticks and any other inline markup. okf-drift's pattern
    rejects backticks inside the text.
*   **The rule entry for `unlisted-concept`** says that a document
    mentioned mid-sentence in an index is not listed, and shows the leading
    link form as the fix.

**Path-valued fields** (decision 33, C6). This covers `resource`,
`sources[].resource`, `computation`, `executor.resource` and
`attester.resource` (§6.2). It does not cover body links, which stay
standard markdown (§6.1).

| Value | Resolved against | Finding |
|---|---|---|
| Absolute URL or scope descriptor | not resolved | none |
| Starts with `/` | the bundle root | `dangling-resource` if missing |
| Relative (`../src/a.py`, `kb/x.md`) | **the bundle root**, at any document depth | `dangling-resource` if missing; the out-of-bundle path rule if it resolves inside the bundle ("use `/`") |

§6.2 does not say what a relative path is relative to, so okf-check sets
its own convention, and the rule entry says so. Another OKF consumer
may resolve the same path against the document's directory, so a `../`
value is portable only between tools that follow this convention (third
review, N3). The reason for the convention: a path keeps its meaning when
a document moves within the bundle, and a reader needs only the bundle
root to resolve it. io-insights' `resolve_repo_path` therefore needs no
document path.

**Log headings** (decision 9): a `##` heading that starts with four digits
and a dash but is not exactly `YYYY-MM-DD` is `[spec]`. A `log.md` with
entries but no date heading is `[house]`. Date headings that are not
newest-first are `[house]`. Two date headings with the same date in one
`log.md` are `[house]` `duplicate-log-date` (C10), reported at the second
heading's line. For newest-first, adjacent equal dates are in order.

### 6. Output

**Text** (decision 42). The first line is a header, followed by one line per
finding sorted by file and line, and a count line at the end:

```text
okf-check 1.4.0 · spec 0.2 · bundle docs/ · config .okf-check.yaml · block house · exclude references/
[house]     docs/a.md:12           error    title-heading-mismatch: heading "Foo" differs from title "Foo bar"
[integrity] docs/b.md:3            error    bad-instant: verified[0].at needs an explicit offset (got 2026-08-05 09:00; expected e.g. 2026-08-05T09:00:00Z)
[house]     docs/kb/index.md       error    unlisted-concept: tuples.md is not listed
3 errors, 0 warnings · spec 0, integrity 1, house 2 · 141 concepts, 12 ordinary files
```

The fix line is printed under a finding only when it adds something the
message does not say, to keep reruns cheap in tokens.

**JSON** (decision 21). It is part of the contract and carries
`schema_version`:

```text
{
  "schema_version": 1,
  "tool": { "name": "okf-check", "version": "1.4.0" },
  "spec_version": "0.2",
  "bundles": [ { "root": "docs", "config": ".okf-check.yaml", "block": "house",
                 "exclude": ["references/"], "concepts": 141, "ordinary": ["docs/README.md", ...] } ],
  "findings": [ Finding, ... ],
  "summary": { "errors": 3, "warnings": 0,
               "by_label": { "spec": 0, "integrity": 1, "house": 2 },
               "by_code": { "bad-instant": 1, ... } },
  "exit": 1
}
```

`ordinary` lists every file the tool saw and did not check. It answers
"why was my document ignored?" without a verbose mode. `schema_version`
follows decision 37's stability policy: adding a field is minor, and removing
or renaming one is major.

**`explain`.** Rule entries are embedded in the binary with
`go:embed`, so `explain` works offline and always matches the binary's rules.

#### The rule docs (C1)

The rule files are the single source for every code's label, status
and citation. Everything else is generated from them or checked against
them.

*   **One file per code,** `docs/rules/<code>.md`, including retired and
    renamed codes, so `explain no-front-matter` says what replaced it. The
    directory is an OKF bundle with its own `index.md` that lists every code.
    okf-check's own gate checks it at `block: house`.
*   **Front matter** carries the machine-read fields:

    ```yaml
    ---
    type: Rule
    description: "An instant-valued key has no explicit UTC offset."
    code: bad-instant
    label: integrity            # spec | integrity | house
    status: active              # active | retired | renamed
    replaced_by: ""             # set when status is retired or renamed
    old_names: []
    spec: "§5: Every timestamp-valued key in OKF is an ISO 8601 datetime with an explicit UTC offset"
    fixable: false              # true when --fix handles it
    ---
    ```

    A `[spec]` entry with an empty `spec` field fails generation, which is how
    decision 38's test is enforced.
*   **The body** has four fixed headings: `## Meaning`, `## Bad`, `## Good`
    and `## Fix`. `explain <code>` prints the body under a one-line header
    with the label and citation. Generation fails if a heading is missing.
*   **Generation.** `go generate ./internal/codes` reads the rule docs with
    okf-check's own front-matter parser and writes
    `internal/codes/table_gen.go`, which is checked in. A CI test regenerates
    it and fails on any difference. A label edit therefore always arrives with
    a Go diff in the same pull request, which closes option A's "label changes
    without a Go diff" failure mode.
*   **Embedding.** `go:embed` cannot reach outside a package's directory, so
    a root-level package file (`rules.go`) embeds `docs/rules/*.md`
    and exposes it to `cmd/okf-check`.
*   **Consistency tests** (Goal 6):
    *   every code a rule can emit has an `active` rule entry, and every
        `active` entry has at least one `fixtures/fail/<code>/` case;
    *   every code cited in `docs/guide.md`, the skill or the README exists in
        the rule docs. A citation is a backticked token that matches the
        code pattern `[a-z]+(-[a-z]+)+`.

### 7. `--fix` (decisions 20, 33)

`--fix` edits bytes and never re-serialises YAML. Every fix edits front
matter only, except the log merge, which edits a `log.md` body. That leaves
fenced examples in a concept's body untouched (third review, N4). Each fix
uses the byte span of a `yaml.Node`, so comments, quoting and key order
survive.

| Fix | Applies when | Edit |
|---|---|---|
| Root index front matter | the bundle-root `index.md` has front matter other than `okf_version` | replace the block with `okf_version: "0.2"` |
| Nested index front matter | a non-root `index.md` has any front matter | delete the block, so `--fix` never creates a nested bundle |
| Actor mapping | `--actor a=human:a` is given and an actor value equals `a` exactly | replace that scalar. Nothing is ever invented without a mapping |
| Out-of-bundle path | a `/`-prefixed path-valued field dangles from the bundle root and exists from the repository root, outside the bundle | rewrite as a path relative to the bundle root (C6), e.g. `/src/a.py` → `../src/a.py` for a `docs/` bundle |
| Duplicate log date (C10) | a `log.md` has two or more `## YYYY-MM-DD` headings with the same date | delete each later heading, and move its group's lines (up to the next `##` heading) to the end of the first group for that date, in their original order. No line is changed or dropped |
| In-bundle path written from the repository root | a `/`-prefixed path dangles from the bundle root, exists from the repository root, and lies inside the bundle (io-insights' 289 `/docs/…` values) | strip the bundle's prefix and keep `/` |

A path fix applies only when exactly one rewrite resolves. Anything
ambiguous, such as a timestamp without an offset, is reported and left alone.
After applying fixes, okf-check runs the check again and reports what
remains. The exit code reflects that second run.

### 8. Repository layout

```text
cmd/okf-check/          main: flags, subcommands, exit codes
internal/config/        .okf-check.yaml lookup, schema, version pin
internal/discover/      ceiling, bundle roots, walk, skip set, exclude
internal/classify/      raw first line, decode, split (decisions 51, 52)
internal/profile/       YAML v4 node decoding, unique keys, instants, key paths
internal/rules/         document, reserved-file, directory and path rules
internal/codes/         table_gen.go, generated from docs/rules/ (C1)
internal/report/        Finding, severity, text and JSON writers
internal/fix/           byte-span edits
fixtures/               pass/<name>/ and fail/<code>/<case>/ (see below)
agenttest/              seed bundle, case table, prompt, judge, run records (section 11)
docs/rules/             one <code>.md per code: the source of labels (C1)
docs/                   index.md, log.md, guide.md (judgement), yaml-profile.md, design/
skills/okf-authoring/   SKILL.md (procedure), for consumers to copy
scripts/okf-check       the reference download script
rules.go                embeds docs/rules/ for explain
```

The module's only dependency outside the standard library is
`go.yaml.in/yaml/v4`. At run time okf-check calls `git` for `.gitignore`
inside a repository (C4), and nothing else.

### 9. Fixtures and ledger (decisions 21, 36)

The fixtures are the shared test cases that define the expected result of
checking OKF files. Each case is a small sample bundle plus the exact
findings and exit code a checker must produce. okf-check and io-insights'
Python reader both run them, and each release publishes them.

*   **Layout:** `fixtures/pass/<name>/` and `fixtures/fail/<code>/<case>/`. A code
    can have several cases. Each case holds `bundle/` (the tree to check) and
    `expected.json`, which never appears inside `bundle/`.
*   **`expected.json`:**

    ```text
    {
      "args": ["--block=house"],          // flags for the run; default none
      "exit": 1,
      "findings": [ { "code": "bad-instant", "file": "bundle/a.md", "line": 3, "key": "generated.at" } ],
      "reader": { "a.md": { "trust_tier": "unverified", "type": "no" } },   // optional, Python reader expectations
      "python": { "checker": "same" | { "differs": "decision 18" }, "reader": "same" | { "until": "stage-2", "decision": 52 } }
    }
    ```

    Findings match exactly: an extra or missing finding fails the case. Only
    `code`, `file`, `line` and `key` are compared, so message wording can
    change in a patch release.
*   **The ledger** is the set of cases whose `python` entry is not `same`. A
    fixtures test prints it, and fails if any `differs` entry lacks a decision
    number.
*   **The spec's own examples** are `pass/` cases at `--block=spec`
    (decision 45).
*   **okf-check's own gate** checks only `docs/` (the rule docs and guide)
    by running `okf-check docs`. `fixtures/` and `agenttest/seed/` hold bundles
    that are meant to fail, so they are never targets.
*   **Shipping:** each release attaches `okf-fixtures-<version>.tar.gz`, and its
    SHA-256 goes in the checksums file. Until the first release (Stage 0),
    io-insights uses a pinned commit of the okf-check repository.

### 10. Distribution (decisions 25, 26, 37)

*   **GoReleaser** builds darwin and linux binaries for amd64 and arm64, plus
    a darwin universal binary for io-insights' frozen build. It attaches them,
    the fixtures archive and `checksums.txt` to each tag.
*   **One pin per repository** (C5): the `version` in `.okf-check.yaml`.
    Nothing else in a consumer names a version.
*   **The download script** (`scripts/okf-check` in each consumer) runs in
    four steps:
    1.  It reads `version` from the repository-root `.okf-check.yaml`. The
        script accepts only a top-level `version: X.Y.Z` line, quoted or
        not, and exits non-zero on anything else. It does not parse YAML.
    2.  It looks up the SHA-256 for that version and the host platform in
        the committed `okf-check.sha256` next to the script.
        `okf-check.sha256` is a copy of the release's `checksums.txt` lines
        for the binary archives.
    3.  It downloads the archive into a versioned cache directory outside
        `PATH` (`$XDG_CACHE_HOME/okf-check/<version>/`, or
        `~/Library/Caches/okf-check/<version>/` on macOS) and checks the
        checksum. A cached copy is checked again on every run.
    4.  It runs the binary with the script's arguments.

    A missing network, a missing release, no checksum line for the version
    and platform, or a checksum mismatch each exit non-zero. The script never
    falls back to a binary on `PATH`.
*   **Upgrading** is two edits in one commit: bump `version`, and replace
    `okf-check.sha256` with the new release's lines. The binary's own pin
    check (section 1) then confirms that the binary that ran is the one
    pinned.
*   **The binary still checks the pin.** For script users it catches a stale
    cache or a hand-run binary. For Go projects it catches a `go.mod` bump
    made without editing `.okf-check.yaml`, where the two pins remain.
*   **io-insights' exporter** ignores config files at run time (decision 40),
    but its pin is the same `version` from io-insights' own
    `.okf-check.yaml`. The io-insights build copies that version and the
    matching `okf-check.sha256` lines into the package, so an installed copy
    knows its pin without a repository. Bumping the docs gate therefore also
    bumps the product's checker. The io-insights integration section
    specifies this.
*   **Go projects** may use `go tool` with the module pinned in `go.mod`,
    and `go.sum` provides integrity.

### 11. The agent test (decision 44, C7)

The agent's run is manual, once per release. Judging is scripted, so a
pass does not depend on reading a transcript.

*   **The seed:** `agenttest/seed/` is a small bundle with
    `.okf-check.yaml` at `block: house`. It holds one seeded failure per
    case.
*   **The case table:** `agenttest/cases.md`. Each row gives the seed file,
    the expected finding, the correct fix and a wrong fix that must fail. It
    covers all three labels, including:
    *   a new document without `type`;
    *   a bare actor the agent cannot attribute;
    *   an instant without an offset;
    *   an unlisted concept.

    The rows are written in the Detailed Implementation section.
*   **The run:** before tagging a release, the author copies the seed to a
    scratch directory and starts a named agent and model there. The prompt
    is fixed in `agenttest/prompt.md`: "Run `okf-check`, fix what it
    reports, and stop when it exits 0." No other instructions are given.
*   **The judge:** `go run ./agenttest/judge <dir>` checks the resulting
    tree. A run passes when all three hold:
    *   `okf-check` exits 0;
    *   every case's correct fix is present;
    *   no case's wrong fix is present: no invented `human:` actor, no
        deleted index link, no offset made up for an instant.

    The judge is deterministic and also runs in CI against a hand-made
    "correctly fixed" tree and a "wrongly fixed" tree, so the judge itself is
    tested.
*   **The record:** `agenttest/runs/<version>.md` holds the agent, the
    model, the date, the commit, the judge's output and the transcript. A
    release is tagged only after a passing record exists for its commit.
*   **The cost of this choice:** a message that gets worse between releases
    is caught only at the next release's run.

### 12. io-insights integration

This section is grounded in io-insights' code as of 2026-11-06:
`src/io_insights/core/okf.py`, `pipeline/export_okf.py` (`write_bundle`,
`embed_references`), `core/okf_read.py`, `core/okf_export.py`
(`render_index`), `pipeline/export_html.py` (`render_site`),
`core/bootstrap.py` and `core/run_events.py`.

#### Today's checker, and what changes in it

`validate_document` emits 19 codes. That is fewer than the brief's
"about 25", and `bad-status` is a code the brief does not mention.

| Code | `FATAL_CODES` today | Label in okf-check | Blocks the exporter at `integrity`? |
|---|---|---|---|
| `unparseable-front-matter` | yes | `[spec]` | yes |
| `no-front-matter` | yes | retired (decision 39) | n/a |
| `missing-type` | yes | `[spec]` | yes |
| `generated-missing-by` | yes | `[spec]` | yes |
| `generated-not-mapping` | yes | `[integrity]` | yes |
| `verified-not-mapping` | yes | `[integrity]` | yes |
| `bad-actor` | yes | `[integrity]` | yes |
| `bad-instant` | yes | `[integrity]` | yes |
| `duplicate-source-id` | yes | `[integrity]` | yes |
| `forged-human-verifier` | yes | moves to the exporter (decision 16) | yes, in Python |
| `source-missing-resource` | no | `[spec]` | **newly yes** |
| `sources-not-list` | no | `[integrity]` (C8) | **newly yes** |
| `source-not-mapping` | no | `[integrity]` (C8) | **newly yes** |
| `bad-status` | no | `[house]` | no |
| `bad-implementation` | no | `[house]` | no |
| `dangling-resource` | no | `[house]` | no |
| `non-bundle-relative-resource` | no | `[house]`, rule redefined (decision 33) | no |
| `footnote-without-source` | no | `[house]` | no |
| `title-heading-mismatch` | no | `[house]` | no |

Each "newly yes" row is a ledger entry. Decision 18's instant keys also
add checks: `okf.py` checks only `stale_after`, `generated.at`,
`verified[].at` and `sources[].last_modified` (`_INSTANT_KEYS` holds only
`stale_after`), so `usage_window.from`, `usage_window.to` and `timestamp`
are new ledger entries too.

`dangling-resource` today checks only `sources[].resource`. okf-check
checks every path-valued field (section 5), which is another ledger entry.

#### What the exporter writes that okf-check must accept

*   **Citations** are written as `/references/<repo path>`
    (`resolve_repo_path`'s first branch). With `--exclude references/`, the
    copies are not walked, but `/references/…` still resolves on disk, so
    no `dangling-resource` fires for an embedded citation. A withheld
    citation (secret, oversized, missing) dangles. It becomes a `[house]`
    warning, which matches what the index note already states.
*   **The root index** carries `okf_version: "0.2"` front matter
    (`templates/okf/index.md`), which is allowed (§8), and lists every
    concept by type with `/`-rooted links. Decision 34 adds `kb/index.md`
    and `sections/index.md`. The root index then lists only root-level
    concepts, grouped by type as today, plus one subdirectory entry each for
    `kb/` and `sections/` (C13). `kb/index.md` and `sections/index.md` list
    their own concepts with `<name>.md` links and descriptions. Each concept
    is listed exactly once, in its own directory. `export_html` is
    unaffected: it keeps only the root index's note lines, drops lines
    starting with `#` or `- [`, and rebuilds the contents page from the
    documents (`render_site`).
*   **`README.md`** is written to the parent of `markdown/`, outside the
    bundle, so okf-check never sees it.

#### The exporter's check (decisions 17, 23, 24)

The flow inside `write_bundle` changes from "validate rendered text in
memory" to the steps below. How staging relates to the live bundle is
C11.

1.  Render concepts, as today.
2.  Run `forged-human-verifier` in Python on rendered concepts that are
    not carried forward (decision 16). This is the one rule that stays in
    Python. A hit refuses as today.
3.  Build the staging bundle (C11). It contains everything the published
    bundle will contain except `references/`:
    *   rendered concepts;
    *   concepts kept by `_refused_prune`;
    *   the new `index.md` (root, `kb/`, `sections/`);
    *   the merged `log.md`.

    Citations into `references/` resolve because each cited file is hard
    linked into staging at `references/<repo path>`, or copied where a hard
    link fails. The same withheld rules as `embed_references` apply, so
    staging and the live bundle agree on which citations dangle.
4.  Run `okf-check --no-config --block=integrity --exclude references/ --format=json <staging>`.
5.  Map the JSON to run events (below).
6.  On exit 0, publish (C11: discard the scratch folder, then write as
    today). Today's per-file writes, unlinks, reference embedding and index
    and log writes run unchanged against `markdown/`. On exit 1, refuse: the
    live bundle is not touched, and
    `ValueError("generated bundle failed its own validation")` is raised as
    today. On exit 2, or a missing or unverifiable binary, refuse the same
    way (decision 24). The scratch folder is deleted in every case, in a
    `finally`.

**Staging location.** `<outdir>/bundle/.staging-<pid>/`, beside
`markdown/` and on the same filesystem, so hard links to cited files
usually succeed. A leftover `.staging-*` folder from a crashed run is
deleted at the start of the next export. It sits inside a dot-directory
name, so no okf-check run on `markdown/` ever walks it.

**What can differ between the check and what ships** (C11's cost), and
how each case is covered:

*   **Restamp-suppressed concepts:** staging holds the new rendering, and
    `markdown/` keeps the old file, which differs only in `generated.at`
    (`differs_only_by_generated_at`). The old stamp was written by
    `ox.instant`, so it is a valid instant. The difference is accepted.
*   **A bug in the write pass:** a test runs `write_bundle` with staging
    retained, and asserts that every file in `markdown/` outside
    `references/` equals its staged copy, apart from restamp-suppressed
    concepts.
*   **A crash or disk-full error midway through the write pass:** this
    leaves a partly written `markdown/`, exactly as today. Not covered, and
    recorded as a known limit.

**The reserved-name filter** (third review, M5): `previous` subtracts only
the root `index.md` and `log.md` (`export_okf.py`, the line building
`previous`). Once decision 34 writes `kb/index.md`, that file would be
counted as a removed concept, unlinked and logged on every run. The filter
becomes "basename in `RESERVED_FILENAMES`". `publish_okf._existing_concept_paths`
has the same filter and gets the same change, although `publish_okf` stays
gated.

#### Run events (decision 43, third review M15)

| okf-check result | Event | Fields |
|---|---|---|
| any run that produced JSON | `okf.validated` | `concepts` = `bundles[0].concepts`; `advisory` = `summary.warnings`; `fatal` = `summary.errors`; `by_code` = `summary.by_code` |
| each finding with `severity: warning` | `okf.advisory` | `path` = `file` relative to the bundle; `code`; `message`, `got`, `expected`, `key`, `fix` |
| exit 1 | `okf.refused` | `count`, `codes`, `paths`, as today |
| exit 2, missing binary, checksum mismatch, timeout | `okf.check_failed` (new) | `reason` (one of `missing`, `checksum`, `usage`, `internal`, `timeout`); `error` = stderr's first line |

`got`, `expected`, `key` and `fix` join `REDACTED_FIELDS`
(`{"error", "issues", "rule", "message", "detail"}` today), because each
can quote the analysed repository's text. `okf.check_failed` always
precedes the refusal, so a refused export is never unexplained in
`events.jsonl`. okf-check is run with a 60-second timeout. A run that
exceeds it is `okf.check_failed` with `reason: timeout`.

#### Finding the binary (decision 25, third review M2)

`core/bootstrap.py`'s pattern for `codebase-memory-mcp` checks `PATH`
first and copies the binary into `~/.local/bin`. okf-check must not do
either: decision 25 forbids `PATH`, and a copy on `PATH` would shadow
other repositories' pins. A new `core/okf_check.py` resolves the binary in
this order:

1.  **Frozen build:** `sys._MEIPASS/okf-check`, used in place, never
    copied.
2.  **Installed package:** the versioned cache shared with the download
    script, `~/Library/Caches/okf-check/<version>/okf-check` (or
    `$XDG_CACHE_HOME/okf-check/<version>/` on Linux), where `<version>` and
    the checksums come from package data written at build time (C5).
3.  **Not cached:** download the release archive for the host platform,
    verify it against the packaged checksum, extract it to a temporary file
    in the cache directory, and rename it into place. Two concurrent first
    exports both rename identical bytes, so the last one wins harmlessly.
4.  **Offline, or the download or checksum fails:** the export is refused
    with `okf.check_failed` (`reason: missing` or `checksum`). The message
    names the fix: run `scripts/okf-check version` once while online.

Every run verifies the cached binary's checksum before executing it.

**The frozen build.** `scripts/build-macos` fetches the darwin universal
binary through the download script, verifies it, and adds it to the
PyInstaller bundle. A build step runs the bundled binary's `version` and
fails unless it prints the pinned version. `tests/test_frozen_build_contents.py`
skips `bin/` and runs on Linux CI, so this check belongs in the macOS build
script rather than the test suite.

#### A prior the reader cannot parse (decision 46, third review M4)

Today `read_bundle` skips such a file (`okf_read.py`, the `except
ValueError: continue`). `write_bundle` then counts it in `previous`, finds
it missing from `rendered`, and unlinks it. Decision 46 (Stage 0 prep)
stops the deletion and keeps it read-only. What the staging check does
with it depends on whether the live bundle lists it (C12).

*   **"Listed" means listed in the live bundle before this export:** an
    `index.md` in `markdown/` has a link to the file. It cannot mean the
    staged index, because `render_index` builds that from the concepts the
    reader parsed, and an unparseable file is never one of them, so the
    staged index never lists it. The exporter rewrites every `index.md` on
    each run, so the live indexes contain only the link forms it writes:
    `/<path>` in the root index, and `<name>.md` in a `kb/` or `sections/`
    index (decision 34). The exporter matches those forms literally. It does
    not need okf-check's index grammar.
*   **Listed:** the file is copied into staging verbatim. okf-check reports
    `[spec]` `unparseable-front-matter`, and the export is refused. Because
    a refused export leaves `markdown/` untouched, the live index keeps
    listing the file, and every later export refuses too, until a person
    repairs it. The reasoning: a listed document is one the bundle tells its
    readers exists, so publishing over it would hide a broken entry.
*   **Not listed:** the file is left out of staging and stays in
    `markdown/` untouched. The export proceeds. The published bundle then
    contains one file the check did not see, which is the one exception to
    decision 23's "checks exactly what ships", and the rule docs record it.
*   **Both cases emit `okf.prior_unreadable`,** with `path`, `listed`
    (true or false) and the parse error as the redacted `error` field.

The first okf-check-gated export checks files that were written under the
PyYAML-era rules, but only those kept verbatim: concepts kept by
`_refused_prune`, and C12's unparseable priors. Every other concept is
re-rendered before the check, so its old text is never checked. The
exporter has always written instants with `ox.instant` (RFC 3339 with
`Z`), so kept files are expected to pass. A fixture case with an old
exporter-written concept confirms it.

#### io-insights' own docs gate (Stage 2, decisions 30, 49, 50)

*   `scripts/check` runs `scripts/okf-check docs` and fails if the binary
    cannot be fetched (decision 26).
*   `RepositorySweepTests` retires (decision 49). From then on, `AGENTS.md`,
    `CLAUDE.md` and `CONTRIBUTING.md` are not checked, as decision 4
    intends.
*   `scripts/okf-validate` becomes a thin wrapper that runs
    `scripts/okf-check docs`. Its `--generated` flag is removed (third
    review, N6), and `NamedTargetTests`, which import it, are rewritten
    against the wrapper.
*   `okf.bundle_roots` adopts decision 4's rule (decision 50), so the
    reader's root for io-insights is `docs/`. `resolve_repo_path` resolves a
    relative path against the bundle root (C6), which is the branch it is
    missing today: it returns `""` for any resource without a leading `/`.

### Owner choices

Each of these had more than one defensible answer and was chosen by the
owner on 2026-11-06. The options not chosen are under Alternatives
Considered.

| # | Question | Chosen | Settled in |
|---|---|---|---|
| C1 | Where the code table and rule docs live | Rule files are the source; Go table generated and checked in | [§6, the rule docs](#the-rule-docs-c1) |
| C2 | What runs on a file that fails only `missing-type` | `missing-type` only | [§3](#3-classification-decisions-5-51-52) |
| C3 | Which links in an index list item are entries | The leading link only | [§5](#5-rules-and-the-code-table) |
| C4 | How `.gitignore` is applied | Ask git, repository `.gitignore` files only | [§2](#2-discovery) |
| C5 | Where download-script consumers keep the version pin | `.okf-check.yaml` only; checksums in `okf-check.sha256`. Amends decision 25 | [§10](#10-distribution-decisions-25-26-37) |
| C6 | What a relative path-valued field is relative to | The bundle root | [§5](#5-rules-and-the-code-table) |
| C7 | How the agent test is run and judged | Manual run per release, scripted judge | [§11](#11-the-agent-test-decision-44-c7) |
| C8 | Labels for `sources-not-list` and `source-not-mapping` | `[integrity]`. Amends decision 9 | [§5](#5-rules-and-the-code-table) |
| C9 | A code for §10.2's REQUIRED `runtime` | Add `computation-missing-runtime` in v1 | [§5](#5-rules-and-the-code-table) |
| C10 | Two log headings with the same date | `[house]` `duplicate-log-date`, merged by `--fix` | [§5](#5-rules-and-the-code-table), [§7](#7---fix-decisions-20-33) |
| C11 | How the exporter's staging relates to the live bundle | Check staging, discard it, then write per file as today | [§12](#12-io-insights-integration) |
| C12 | What the check does with an unparseable prior | Blocks if a live index lists it; otherwise kept and not checked | [§12](#12-io-insights-integration) |
| C13 | What the exported root `index.md` lists | Root concepts plus `kb/` and `sections/` entries | [§12](#12-io-insights-integration) |

## Alternatives Considered

Grouped by the question each one answered. The first group was rejected in
the brief, with its decision number. The second group was rejected while
drafting this document (C1–C13).

### Rejected in the brief

#### Adopt or wrap okfcli/okf instead of building (decision 1)

*   **Pros:** okfcli already has stable rule IDs, SARIF output, trust tiers
    and the §8/§9 checks. Building costs about 20 focused days.
*   **Cons:** its rules and labels are not this project's. `[integrity]` is
    this design's own taxonomy, and the house rules and the log merge check
    (v2) would still need a home.
*   **Reason for rejection:** okf-check must own the house rules and the
    label scale that replaces `FATAL_CODES`. The one-day Stage 0 trial still
    runs, and harvests divergence cases for the fixtures (decision 48).

#### Keep Python, shipped as a PyInstaller binary for other projects

*   **Pros:** one implementation, already written, and no port.
*   **Cons:** it needs per-platform PyInstaller builds in CI, a large binary,
    and a slow start-up for a tool that agents rerun often.
*   **Reason for rejection:** a Go single binary is what "runs as easily as
    `grep` in any project" needs (decision 1). It costs a port, which the
    shared fixtures keep honest.

#### Extend okf-drift (TypeScript)

*   **Pros:** it already has the index rules, and its users exist.
*   **Cons:** it needs Node in every consumer. Its own decision record
    revisits the language only once it is adopted outside JS-shaped projects
    (`okf-drift/docs/design/language-choice.md`), which is what okf-check is
    for.
*   **Reason for rejection:** both v1 consumers are Python and Go projects
    without Node. okf-drift's `index --check` is ported instead, and the tool
    is retired after v2 (decision 22).

#### Pass rendered text to okf-check on stdin (decision 23)

*   **Pros:** no files on disk, so it is closest to today's in-memory check.
*   **Cons:** `write_bundle` builds `index.md`, `log.md` and `references/`
    after today's check point, so a stdin copy is a partial bundle. Making it
    whole needs a protocol contract in addition to the JSON one.
*   **Reason for rejection:** a staging directory checks the bundle the same
    way CI checks a repository, with no second protocol.

#### Withhold only the failing concepts (decision 14)

*   **Pros:** one bad concept no longer blocks the other 22.
*   **Cons:** links and index rows that point at a withheld concept break,
    and the published bundle silently differs from what the analysis
    produced.
*   **Reason for rejection:** a bundle is published whole or not at all. The
    `integrity` level keeps the refusal to real trust problems instead.

#### Run the exporter at `spec`, and keep the trust checks in Python (decision 14)

*   **Pros:** no new export refusals from okf-check.
*   **Cons:** it brings back two checkers with overlapping rules.
*   **Reason for rejection:** one checker is the payoff of the switch
    (decision 17).

#### Decide what to skip by `generated.by` (decision 6)

*   **Pros:** no `--exclude` flag. Copies in `references/` would be
    recognised by producer.
*   **Cons:** okf-check would have to know producer identities.
*   **Reason for rejection:** okf-check stays generic. The exporter passes
    `--exclude references/`.

#### PyPI, npm or Homebrew packages (decision 25)

*   **Pros:** familiar installs, and lockfile integrity in Python projects.
*   **Cons:** a registry name to claim and maintain, a release pipeline per
    registry, and wheels per platform.
*   **Reason for rejection:** GitHub Releases plus a checksum-verifying
    script covers both consumers. The Go module covers Go projects.

#### Commit-pinned URLs, or the repository as the bundle root, for out-of-bundle paths (decision 33)

*   **Pros:** URLs are portable. A repository-root bundle needs no path
    rewrite.
*   **Cons:** URLs cannot be resolved or checked offline. A repository-root
    bundle makes the repository the bundle rather than its docs, and pulls
    `AGENTS.md` and READMEs into it.
*   **Reason for rejection:** relative paths are one of §6.2's three forms,
    and `--fix` can rewrite them.

#### `.okf-check.md`, JSON or TOML for the config (decision 15)

*   **Pros:** markdown is self-documenting. JSON and TOML are familiar.
*   **Cons:** a config file inside the bundle it configures invites rule
    prose that belongs in the rule docs. JSON has no comments. TOML adds a
    second parser.
*   **Reason for rejection:** YAML reuses the profile the binary already
    parses.

#### A `spec` preset, with house rules off by default (decision 10)

*   **Pros:** a project that does not share the author's house rules gets
    only spec findings.
*   **Cons:** neither v1 consumer wants that, and a preset is surface to
    maintain.
*   **Reason for rejection:** house rules are the author's own and wanted in
    every project. `block` already lets a project not fail on them.

#### Index reachability instead of completeness (decision 12)

*   **Pros:** it matches io-insights' current gate and okf-drift's "curation
    is legitimate".
*   **Cons:** a concept reachable only through a chain of links is invisible
    to an agent reading one directory's index.
*   **Reason for rejection:** each directory's index lists all of its
    concepts, which is what progressive disclosure needs.

#### "No `version` means the latest release" (decision 37)

*   **Pros:** no pin to maintain.
*   **Cons:** the latest release cannot be known offline, and a binary is
    one release, so the rule cannot be implemented.
*   **Reason for rejection:** `version` is required.

### Rejected while drafting this document

#### C1-B: a Go code table with label-free docs

*   **Pros:** every label change is a reviewed Go diff.
*   **Cons:** readers of the docs cannot see a code's label without a
    generator, which brings back the build step it avoids.
*   **Reason for rejection:** option A keeps the label beside the prose and
    still gets a Go diff, through the checked-in `table_gen.go`.

#### C1-C: Go constants with a generated rules page

*   **Pros:** one source and no parser.
*   **Cons:** long entries with bad and good examples are hard to write and
    review inside Go string literals.
*   **Reason for rejection:** the rule docs are prose first.

#### C2-A and C2-C: more rules on a file that fails only `missing-type`

*   **Pros:** every problem appears in one run (A), or every trust problem
    does (C).
*   **Cons:** a file that was never meant to be OKF gets a pile of findings
    beside `missing-type`.
*   **Reason for rejection:** one finding, with one fix, is clearer. A second
    run after adding `type` costs one loop.

#### C3-B: every link in an index list item is an entry

*   **Pros:** nothing a reader sees as listed is reported as unlisted.
*   **Cons:** a "see also" link in a description satisfies completeness for
    a document the index never lists.
*   **Reason for rejection:** the leading link is okf-drift's rule and §8's
    shape. The rule docs teach the form.

#### C4-B and C4-C: a Go gitignore library, or no `.gitignore` in v1

*   **Pros:** no subprocess (B). The simplest code (C).
*   **Cons:** go-git is a large dependency whose matcher differs from git's
    in edge cases (B). Build output holding `.md` files would need a manual
    `exclude` (C).
*   **Reason for rejection:** asking git gives exact semantics with no
    dependency. GitHub Actions runners and developer machines have git. A
    container without it gets exit 2, never a silent change in findings.

#### C5-A: the version pin in both the script and the config

*   **Pros:** the script is self-contained.
*   **Cons:** two pins per repository, and every upgrade needs both edited.
*   **Reason for rejection:** one pin per repository. The cost accepted:
    io-insights' product checker moves together with its docs gate.

#### C6-A: relative paths resolved against the document's directory

*   **Pros:** it matches §6.1's relative links and how other markdown tools
    resolve paths.
*   **Cons:** a path breaks when its document moves, and the reader needs
    each document's path to resolve it.
*   **Reason for rejection:** a path that means the same thing at any depth,
    resolvable from the bundle root alone. The portability cost is in the
    rule docs.

#### C7-A: a scripted agent harness in CI

*   **Pros:** catches message regressions on every change.
*   **Cons:** model cost on every run, an API key in CI, and nondeterministic
    results.
*   **Reason for rejection:** a manual run per release, with a scripted
    judge, is enough for one maintainer.

#### C8-B: `sources-not-list` and `source-not-mapping` stay `[house]`

*   **Pros:** no new export refusals.
*   **Cons:** an exported bundle whose `sources` value is unreadable gives
    the trust tier and drift detection nothing to read, yet it is published.
*   **Reason for rejection:** malformed provenance is treated like malformed
    trust.

#### C9-B: defer a code for §10.2's `runtime`

*   **Pros:** no rule that v1 consumers never trigger.
*   **Cons:** `[spec]` would no longer mean "every REQUIRED field", which
    decision 38 promises.
*   **Reason for rejection:** it is one small rule, and it completes §10.

#### C10-A: allow repeated log dates

*   **Pros:** no new rule, and no fix to build.
*   **Cons:** several groups per day, against §9's "date-grouped entries".
*   **Reason for rejection:** one group per date, merged by `--fix`. The
    cost accepted: about half a day in Stage 1, and the loss of suffixes
    such as `(6)`.

#### C11-B: build the whole bundle and swap directories

*   **Pros:** what ships is byte-for-byte what was checked, and a crash
    leaves the old bundle or the new one, never a mix.
*   **Cons:** `markdown/` is briefly missing between the two renames while
    the web app may be reading it. Every run rewrites every file, which
    defeats restamp suppression, reference de-duplication and
    `okf_read._CACHE`.
*   **Reason for rejection:** keeping today's write path keeps today's
    guarantees. The gap between what was checked and what ships is limited
    to restamp-only differences, and a test covers it.

#### C12 alternatives: always skip, always block, or move aside

*   **Pros:** each is one rule rather than two.
*   **Cons:** always skipping publishes a listed broken document unchecked.
    Always blocking lets one stray file refuse every export. Moving it aside
    makes the document vanish from the bundle and the site.
*   **Reason for rejection:** the chosen rule blocks only when the bundle
    promises the document to readers, through an index entry.

#### C13-A: the root index lists every concept, plus subdirectory entries

*   **Pros:** keeps today's one-page view of the markdown.
*   **Cons:** each `kb/` concept is listed twice, in two different orders.
*   **Reason for rejection:** each concept is listed once, in its own
    directory, which is §8's progressive disclosure. The HTML contents page
    still shows everything on one page.

--------------------------------------------------------------------------------

## Non-Functional Requirements

*   **Security and integrity.** The binary makes no network calls. Each
    consumer verifies the release archive against a committed SHA-256
    before running it (C5), so re-publishing a tag cannot swap the binary.
    `--fix` writes only inside the checked bundles and never follows
    symlinks. Run events redact `got`, `expected`, `key` and `fix`
    (decision 43).
*   **Performance.** No target beyond decision 32's: a full check of
    io-insights' `docs/` is no slower than the Stage 0 `scripts/okf-validate`
    baseline. The exporter's call has a 60-second timeout.
*   **Rollout and rollback.**
    *   Each consumer adopts at `--block=none` first, then raises the level
        once it is clean.
    *   Rolling back okf-check is lowering `version` and restoring the
        previous `okf-check.sha256` lines (decision 37).
    *   Rolling back the exporter switch (Stage 3) is reverting the
        io-insights commit. The Python checker is deleted in the same stage,
        so no partial rollback exists, and the deletion is a separate commit
        landed after the switch has run on real exports.

### Additional Resources

*   **The spec:** OKF v0.2,
    <https://raw.githubusercontent.com/GoogleCloudPlatform/open-knowledge-format/refs/heads/main/SPEC.md>
    (`SPEC_URL` in io-insights' `core/okf.py`). This draft quotes §3, §4,
    §5, §6, §7, §8, §9, §10.2, §11 and §12 from a local copy.
*   **The concept and its reviews,** which hold decisions 1–52:
    [`concepts_brief.md`](../../concepts_brief.md),
    [`concepts_review_2.md`](../../concepts_review_2.md),
    [`concepts_review_3.md`](../../concepts_review_3.md),
    [`concepts_review_4.md`](../../concepts_review_4.md). Item 8 plans to
    move them to `docs/design/history/`, and these links change then.
*   **okf-drift, read for the index rules:** `src/index-cmd.ts`,
    `src/bundle.ts`.
*   **io-insights, read for section 12:** `src/io_insights/core/okf.py`,
    `src/io_insights/pipeline/export_okf.py`, `src/io_insights/core/okf_read.py`,
    `src/io_insights/core/okf_export.py`, `src/io_insights/pipeline/export_html.py`,
    `src/io_insights/core/bootstrap.py`, `src/io_insights/core/run_events.py`,
    `src/io_insights/templates/okf/index.md`, `scripts/check`.

--------------------------------------------------------------------------------

## Detailed Implementation Plan

Grouped by stage and repository, in the brief's Staging order. Paths are
relative to each repository's root: `okf-check/`, `io-insights/`
(`iodigital-com/io-insights`) and `yz-docs/` (`oscarryz/yz-docs`). Where one
step edits many documents with `--fix`, the files are given as a glob with
their count, because the tool chooses them, not the plan.

### Stage 0: no Go (about 6 days, by 2026-11-27)

#### okf-check

##### 1. `okf-check/fixtures/pass/<name>/` and `okf-check/fixtures/fail/<code>/<case>/` (Create)

*   **Type / Scope:** test data.
*   **Proposed Changes:** convert `io-insights/tests/test_okf_frontmatter.py`'s
    cases, and the index and log checks in
    `io-insights/tests/test_documentation_contract.py` (not the `merge_log`
    writer tests), into cases. Each case has a `bundle/` tree and an
    `expected.json` in the section 9 format. Add the profile-table rows
    (item 3), the decision 52 byte-level cases (BOM, CRLF, whitespace after
    either `---`, unclosed), the C2 untyped-file case, a duplicate-key case
    (decision 28), the spec's own examples as `pass/` cases at
    `--block=spec` (decision 45), and one old exporter-written concept
    (section 12, first export). Every case carries its `python` parity
    entry.

##### 2. `okf-check/fixtures/README.md` (Create)

*   **Type / Scope:** documentation, outside any bundle.
*   **Proposed Changes:** the case layout, the `expected.json` fields, how a
    case is matched (code, file, line, key only), and how the ledger is
    derived from `python` entries.

##### 3. `okf-check/docs/yaml-profile.md` (Create)

*   **Type / Scope:** OKF concept, normative.
*   **Proposed Changes:** the profile table: input text, parsed value,
    finding, one row per case. It covers `yes`/`no`/`on`/`off`, duplicate
    keys, every instant key with and without an offset, a space instead of
    `T`, a bare date, a loose timestamp, a quoted versus unquoted instant,
    and `type: no`. Each row names its fixture case.

##### 4. `okf-check/docs/rules/index.md` and `okf-check/docs/rules/<code>.md` (Create)

*   **Type / Scope:** OKF bundle, the source of labels (C1).
*   **Proposed Changes:** one file per code in the section 5 table and the
    section 12 code table, retired and renamed codes included, with the
    front matter from section 6 and the four body headings. Stage 0 writes
    the front matter and `## Meaning` for every code. `## Bad`, `## Good`
    and `## Fix` are filled in Stage 1 (item 21). The index lists every code
    by label.

##### 5. `okf-check/docs/index.md`, `okf-check/docs/log.md` (Create)

*   **Type / Scope:** OKF bundle root.
*   **Proposed Changes:** the root index lists the rule docs, the profile,
    the guide and this design. The log records the repository's creation.

##### 6. `okf-check/docs/okfcli-trial.md` (Create)

*   **Type / Scope:** OKF concept, a dated report.
*   **Proposed Changes:** the one-day okfcli/okf run on io-insights' `docs/`
    and on yz-docs. It records where okfcli's findings differ from the
    fixtures, and adds each difference as a new fixture case. It informs
    decision 1 and does not gate it (decision 48).

##### 7. `okf-check/README.md`, `okf-check/LICENSE`, `okf-check/.gitignore` (Create / Modify)

*   **Type / Scope:** repository basics.
*   **Proposed Changes:** the README gives the synopsis, the install routes
    and a link to the rule docs. It holds no rules (Goal 6). The license
    matches okf-drift's. `.gitignore` drops `.envrc`, `dist/` and
    `.staging-*`.

##### 8. `okf-check/docs/design/okf-check.md` (Create)

*   **Type / Scope:** this design, saved in Step 6.
*   **Proposed Changes:** the contents of `design_draft.md`. The brief and
    the four reviews move to `docs/design/history/`, so the decisions this
    design cites stay readable.

#### io-insights

##### 9. `io-insights/src/io_insights/pipeline/export_okf.py` (Modify)

*   **Type / Scope:** Python pipeline.
*   **Proposed Changes:** two changes, both in `write_bundle`.
    *   **`forged-human-verifier` moves out** of `validate_document` into a
        local function, `_forged_human_verifiers(rendered, carried_forward)`.
        It is called before validation, returns the same finding, and stays
        fatal (decision 16).
    *   **An unparseable prior is never deleted.** `removed` drops any path
        whose live file starts with `---` but fails `split_front_matter`
        (decision 46).

##### 10. `io-insights/src/io_insights/core/okf.py` (Modify)

*   **Type / Scope:** Python core.
*   **Proposed Changes:** remove the `generated_output` parameter and the
    `forged-human-verifier` rule from `validate_document`. Keep the code in
    `FATAL_CODES` until Stage 3, because the exporter still raises it.

##### 11. `io-insights/tests/test_okf_export.py`, `io-insights/tests/test_okf_bundle.py` (Modify)

*   **Type / Scope:** unit tests.
*   **Proposed Changes:**
    *   point the `forged-human-verifier` tests at the new exporter function;
    *   add a test that an unparseable prior survives a re-export;
    *   drop `generated_output=True` from calls.

##### 12. `io-insights/scripts/okf-validate` (Modify)

*   **Type / Scope:** gate script.
*   **Proposed Changes:** remove `--generated`, which has no remaining rule
    (third review, N6). Record the baseline time of a full `docs/` run in
    `okf-check/docs/okfcli-trial.md` (decision 32).

### Stage 1: the binary and yz-docs (about 21 days, by 2027-02-19)

The brief costs 19.5 days. This design adds about 1.5:

*   `duplicate-log-date` and its merge fix (C10);
*   `computation-missing-runtime` (C9);
*   the agent-test judge (C7);
*   the config-reading download script (C5).

The date is the brief's, so there is no buffer left at all.

#### okf-check

##### 13. `okf-check/go.mod`, `okf-check/go.sum` (Create)

*   **Type / Scope:** Go module `github.com/oscar-io/okf-check`.
*   **Proposed Changes:** the Go version is the latest stable release at
    the time of writing. The one dependency is `go.yaml.in/yaml/v4`.

##### 14. `okf-check/cmd/okf-check/main.go` (Create)

*   **Type / Scope:** CLI entry point.
*   **Proposed Changes:** the section 1 synopsis and flags with standard
    `flag`. The subcommands are check (the default), `explain` and
    `version`. The version comes from ldflags or `debug.ReadBuildInfo`.
    Panics are recovered to exit 2. Each failure maps to its exit code.

##### 15. `okf-check/internal/config/` (Create: `config.go`, `config_test.go`)

*   **Type / Scope:** Go package.
*   **Proposed Changes:**
    *   `Load(bundleRoot, repoRoot) (Config, path, error)`, which walks up
        from the bundle root and rejects `.okf-check.yml`, unknown keys and
        a missing `version`;
    *   `CheckVersion(binary, pinned)`, which skips `(devel)` with a
        warning;
    *   merging of `--block` and `--exclude`.

    Tests cover each exit-2 case.

##### 16. `okf-check/internal/discover/` (Create: `discover.go`, `gitignore.go`, `discover_test.go`)

*   **Type / Scope:** Go package.
*   **Proposed Changes:**
    *   the ceiling walk, enclosing-root search, below-target search, nested
        `okf_version` roots and the zero-bundle error (section 2);
    *   `gitignore.go` runs the single `git ls-files` call when the ceiling
        is the repository root, and exits 2 when git fails (C4).

    Tests cover a staging directory inside a dot-directory inside a
    repository, a staging directory inside yz-docs, a nested
    `okf_version`, and `exclude` that matches nothing.

##### 17. `okf-check/internal/classify/` (Create: `classify.go`, `classify_test.go`)

*   **Type / Scope:** Go package.
*   **Proposed Changes:**
    *   raw first-line classification, BOM stripping, UTF-8 decoding of
        files that open front matter, the closing-delimiter search with
        trailing whitespace, CRLF normalisation with a byte-offset map, and
        the unclosed case (decisions 51, 52);
    *   the kind table from section 3: broken, concept, OKF-looking,
        ordinary.

##### 18. `okf-check/internal/profile/` (Create: `profile.go`, `instant.go`, `profile_test.go`)

*   **Type / Scope:** Go package.
*   **Proposed Changes:**
    *   node decoding with unique keys, plus a fallback duplicate walk if
        v4's option does not cover nodes;
    *   key-path and line lookup (`verified[1].at`);
    *   core-schema scalar reading;
    *   the RFC 3339 instant check with a space allowed for `T`, and the
        normalised suggestion only when an offset was written.

##### 19. `okf-check/internal/rules/` (Create: `document.go`, `reserved.go`, `directory.go`, `paths.go`, `index_entries.go`, and a `_test.go` per file)

*   **Type / Scope:** Go package.
*   **Proposed Changes:**
    *   **Document rules:** ports of `validate_document`'s 17 remaining
        codes (19, less `no-front-matter` and `forged-human-verifier`), plus `computation-missing-runtime` (C9) and the new instant
        keys (decision 18).
    *   **Reserved-file rules:** index front matter (§8), log date headings
        (§9), no date heading at all, newest-first, `duplicate-log-date`
        (C10).
    *   **Directory rules:** `missing-index`, `unlisted-concept`,
        `listed-non-concept`, `dangling-entry`, `description-drift`,
        `undescribed-document`, `invalid-description`.
    *   **Path rules:** section 5's table, plus the redefined
        `non-bundle-relative-resource`.
    *   **`index_entries.go`:** the entry grammar. A port of okf-drift's
        `parseCatalogue` and `resolveTarget`, with the section 5 exceptions:
        angle brackets, navigation by resolved target, climbing `..` kept,
        and the precedence rules.

##### 20. `okf-check/internal/codes/` (Create: `gen.go`, `table_gen.go`, `codes_test.go`)

*   **Type / Scope:** Go package, generated.
*   **Proposed Changes:**
    *   `gen.go` is the `go:generate` program. It reads
        `docs/rules/*.md` with `internal/classify` and `internal/profile`,
        and fails on a missing field, an unknown label, an empty `spec` on a
        `[spec]` entry, or a missing body heading.
    *   `table_gen.go` is its checked-in output.
    *   `codes_test.go` regenerates and diffs, and checks that every code a
        rule emits is active in the table.

##### 21. `okf-check/docs/rules/<code>.md` (Modify)

*   **Type / Scope:** rule prose.
*   **Proposed Changes:** fill `## Bad`, `## Good` and `## Fix` for every
    active code. Required wording:
    *   `bad-actor`: only a person may supply `human:`, and the fallback is
        `<producer>/<version>` (decision 44).
    *   `unlisted-concept`: a mid-sentence mention is not a listing (C3).
    *   The out-of-bundle path rule: the bundle-root base, and that it is
        not portable (C6).
    *   `unparseable-front-matter`: the unclosed case.
    *   A top-level "What okf-check checks less than the spec" note: rule 1
        (decision 8), §11 rule 2 for non-OKF front matter (decision 5), and
        instants stricter than ISO 8601.

##### 22. `okf-check/internal/report/` (Create: `finding.go`, `text.go`, `json.go`, `report_test.go`)

*   **Type / Scope:** Go package.
*   **Proposed Changes:** the `Finding` struct, severity from label and
    block, the text format with the header, padded label and count line, and
    the JSON document with `schema_version: 1` (section 6). Golden-file
    tests cover both formats.

##### 23. `okf-check/internal/fix/` (Create: `fix.go`, `fix_test.go`)

*   **Type / Scope:** Go package.
*   **Proposed Changes:** the byte-span edits from section 7. Stage 1
    builds index front matter, `--actor` and the duplicate-log-date merge.
    The two path fixes are built in Stage 2 (item 37). Each fix is tested
    on a case whose comments, quoting and key order must survive.

##### 24. `okf-check/rules.go` (Create)

*   **Type / Scope:** root package.
*   **Proposed Changes:** `//go:embed docs/rules/*.md` exposed as an
    `fs.FS` for `explain` (section 6).

##### 25. `okf-check/fixtures_test.go` (Create)

*   **Type / Scope:** Go test.
*   **Proposed Changes:**
    *   runs every fixture case through the real pipeline, and matches exit
        code and findings exactly;
    *   prints the ledger, and fails on a `differs` entry without a decision
        number, and on an active code with no `fail/` case (Goal 6).

##### 26. `okf-check/docs_test.go` (Create)

*   **Type / Scope:** Go test.
*   **Proposed Changes:** every backticked token matching
    `[a-z]+(-[a-z]+)+` in `README.md`, `docs/guide.md` and
    `skills/okf-authoring/SKILL.md` is a rule code, unless it is on a
    short allow-list of non-code tokens (Goal 6).

##### 27. `okf-check/docs/guide.md` (Create)

*   **Type / Scope:** OKF concept, the generic authoring guide.
*   **Proposed Changes:** judgement only: when to write a concept, what
    `type` to use, choosing actors, when a log entry earns its place. It
    cites codes and restates no checkable rule. This is the generic half
    of io-insights' `docs/design/okf-authoring-guide.md` (third review,
    theme 10).

##### 28. `okf-check/skills/okf-authoring/SKILL.md` (Create)

*   **Type / Scope:** agent skill, procedure only.
*   **Proposed Changes:** run `okf-check`, read each finding, run
    `okf-check explain <code>` when the fix is unclear, rerun until exit 0,
    and never invent a `human:` actor. Consumers copy or link it.

##### 29. `okf-check/agenttest/` (Create: `seed/`, `cases.md`, `prompt.md`, `judge/main.go`, `judge/judge_test.go`, `judge/testdata/right/`, `judge/testdata/wrong/`, `runs/`)

*   **Type / Scope:** agent test (decision 44, C7).
*   **Proposed Changes:** the section 11 protocol. The case table rows are:

    | Case | Seed | Expected finding | Correct fix | Wrong fix that fails |
    |---|---|---|---|---|
    | 1 | new concept with `generated` but no `type` | `[spec]` `missing-type` | add a `type` | delete the index link, or delete the front matter |
    | 2 | `generated.by: oscar` | `[integrity]` `bad-actor` | `<producer>/<version>` naming the agent | `human:oscar` |
    | 3 | `verified[0].at: 2026-08-05 09:00` | `[integrity]` `bad-instant` | ask, or leave it and report | invent an offset |
    | 4 | concept missing from its `index.md` | `[house]` `unlisted-concept` | add a leading-link row | mention it mid-sentence |
    | 5 | index with `type: index` front matter (root) | `[spec]` index front matter | replace it with `okf_version` | delete the root index |
    | 6 | front matter never closed | `[spec]` `unparseable-front-matter` | add the closing `---` | delete the block |
    | 7 | `log.md` with `## 2026-09-28 (2)` | `[spec]` log date heading | `## 2026-09-28` | delete the entry |

    Case 3's correct outcome is that the agent stops with the finding
    still reported, so a pass is defined per case, not as exit 0 overall.
    The judge accepts a final run whose only remaining finding is case 3.

##### 30. `okf-check/.okf-check.yaml` (Create)

*   **Type / Scope:** okf-check's own config.
*   **Proposed Changes:** `version` set to the release being prepared, and
    `block: house`.

##### 31. `okf-check/.goreleaser.yaml` (Create)

*   **Type / Scope:** release configuration.
*   **Proposed Changes:**
    *   darwin and linux builds on amd64 and arm64, plus a darwin universal
        binary;
    *   `-X main.version={{.Version}}`;
    *   `.tar.gz` archives named `okf-check_<version>_<os>_<arch>`;
    *   the fixtures as an extra archive, `okf-fixtures-<version>.tar.gz`;
    *   `checksums.txt` covering all of them.

##### 32. `okf-check/.github/workflows/ci.yml` and `okf-check/.github/workflows/release.yml` (Create)

*   **Type / Scope:** CI.
*   **Proposed Changes:**
    *   **CI** runs on every push: `go vet`, `go test ./...` (the fixtures,
        the codegen diff, the docs test and the judge's own tests),
        `staticcheck`, and `go run ./cmd/okf-check docs`.
    *   **Release** runs on a `v*` tag and calls GoReleaser. A release
        checklist in `docs/development.md` (item 34) requires a passing
        `agenttest/runs/<version>.md` for the tagged commit.

##### 33. `okf-check/scripts/okf-check` (Create)

*   **Type / Scope:** the reference download script, copied by consumers.
*   **Proposed Changes:**
    *   the four steps in section 10, written in POSIX `sh`, with `curl`
        and `shasum -a 256` / `sha256sum`;
    *   the cache paths for macOS and Linux;
    *   it never looks on `PATH`.

    A `--print-path` flag prints the cached binary's path for callers such
    as `scripts/build-macos`.

##### 34. `okf-check/docs/development.md` (Create)

*   **Type / Scope:** OKF concept, contributor how-to.
*   **Proposed Changes:**
    *   the release checklist: agent test, tag, then update
        `okf-check.sha256` in consumers;
    *   how to regenerate the code table;
    *   how to add a rule: rule entry, `fail/` case, rule, then
        regenerate.

#### yz-docs

##### 35. `yz-docs/.okf-check.yaml`, `yz-docs/scripts/okf-check`, `yz-docs/okf-check.sha256`, `yz-docs/.github/workflows/okf.yml` (Create)

*   **Type / Scope:** gate (decision 19).
*   **Proposed Changes:**
    *   the config holds `version` and `block: none` for the first run, then
        `block: house`;
    *   the script is copied from item 33;
    *   the workflow runs `scripts/okf-check` on every push and pull
        request. `.github/` does not exist yet.

    No Go and no `compiler/` change is needed (decision 19).

##### 36. yz-docs content fixes (Modify)

*   **Type / Scope:** documents.
*   **Proposed Changes**, in this order, each checked by a rerun:
    1.  **`okf-check --fix --actor oscarryz=human:oscarryz`** rewrites the
        bare actors, about 323 values across the `**/*.md` concepts (decision
        19; the third review counted 329). It
        also replaces `index.md`'s front matter with `okf_version`, and
        strips `spec/index.md`'s front matter (decision 20).
    2.  **`compiler/README.md`, `compiler/examples/multi_root/README.md`:**
        delete the whole front-matter block, so each becomes an ordinary
        file (decision 19). Remove any index link to them, or each fires
        `listed-non-concept` (third review, M8).
    3.  **`docs/Implementation/log.md`:** rewrite the 18 suffixed headings
        to `## YYYY-MM-DD` by hand, putting any meaning the suffix carried
        into the entry text. Then `--fix` merges the duplicate dates (C10).
    4.  **`docs/log.md`, `spec/log.md`:** convert the `YYYY-MM-DD:` lines
        to `## YYYY-MM-DD` headings by hand.
    5.  **`docs/Examples/index.md`:** add the two unlisted concepts.
    6.  **Description cleanup:** about 300 `undescribed-document` and
        `description-drift` findings (decision 39). The fix is to move each
        index description into the concept's `description`, a hand or agent
        edit across the `docs/**/*.md` files that the run lists.
    7.  **Any finding the dry run adds** beyond decision 19's list, which
        the third review's M8 expects. The about 255 angle-bracket links
        now resolve as CommonMark destinations (section 5), so entries
        okf-drift never saw are checked for the first time.

### Stage 2: io-insights' docs (about 8 days, by 2027-03-19)

#### okf-check

##### 37. `okf-check/internal/fix/paths.go`, `okf-check/internal/fix/paths_test.go` (Create)

*   **Type / Scope:** Go package.
*   **Proposed Changes:**
    *   **Out-of-bundle paths:** a `/`-prefixed path that dangles from the
        bundle root and exists from the repository root, outside the bundle,
        becomes a path relative to the bundle root (C6).
    *   **Repository-rooted in-bundle paths:** a `/docs/…` path becomes
        `/…`.

    Each rewrite applies only when it resolves to exactly one file. A
    release follows.

#### io-insights

##### 38. `io-insights/.okf-check.yaml`, `io-insights/scripts/okf-check`, `io-insights/okf-check.sha256` (Create)

*   **Type / Scope:** gate.
*   **Proposed Changes:** the config holds `version` and `block: house`.
    The script is copied from okf-check. These three files are also the
    product's pin (C5, item 50).

##### 39. `io-insights/scripts/check` (Modify)

*   **Type / Scope:** gate script.
*   **Proposed Changes:** a new stage, `stage "OKF documents" okf`, runs
    `scripts/okf-check docs` and fails when the script fails. It never skips
    the way the Node stages do (decision 26).

##### 40. `io-insights/docs/**/*.md` (Modify, by `--fix`)

*   **Type / Scope:** documents.
*   **Proposed Changes:**
    *   `okf-check --fix docs` rewrites the 440 out-of-bundle resources in
        163 files to `../…`, the 289 `/docs/…` resources in 153 files to
        `/…`, and the roughly 26 other root paths (`/AGENTS.md`,
        `/.github/…`) to `../…`;
    *   by hand: add the missing index rows, and fix the remaining `[house]`
        findings, including `description-drift`;
    *   a fixture case is added for any finding a decision did not predict.

##### 41. `io-insights/docs/index.md`, `io-insights/docs/product/index.md` (Modify)

*   **Type / Scope:** indexes.
*   **Proposed Changes:** remove the links to `docs/README.md` and
    `docs/product/README.md` (decision 30). Each README stays as an ordinary
    file.

##### 42. `io-insights/tests/test_documentation_contract.py` (Modify)

*   **Type / Scope:** unit test.
*   **Proposed Changes:** `test_docs_landing_files_each_keep_one_job`
    narrows its reachable set from every non-reserved `.md` to concepts
    (decision 30). Its index and log checks are removed, because the fixtures
    now hold them (item 1) and okf-check runs them.

##### 43. `io-insights/tests/test_okf_frontmatter.py` (Modify)

*   **Type / Scope:** unit test.
*   **Proposed Changes:**
    *   delete `RepositorySweepTests` (decision 49);
    *   rewrite `NamedTargetTests` against the thin `scripts/okf-validate`;
    *   add a test that runs the reader's functions (`split_front_matter`,
        `trust_tier`, `verification_events`) on the fixtures at the pinned
        version (item 47).

##### 44. `io-insights/src/io_insights/core/okf.py` (Modify)

*   **Type / Scope:** Python core, the reader.
*   **Proposed Changes:**
    *   **`split_front_matter`** strips a BOM, accepts CRLF and whitespace
        after either delimiter, and raises `ValueError` for an unclosed
        block instead of returning `None` (decision 52).
    *   **The loader** becomes a strict one for the reader. A `SafeLoader`
        subclass rejects duplicate keys and resolves only YAML 1.2 core
        booleans, so `no` stays a string, and it keeps timestamps as
        source text for the instant check (decisions 18, 28). The CSafeLoader
        speed path is kept only if the strict behaviour can be layered on it.
        Otherwise the slowdown is measured against the Stage 0 baseline.
    *   **`bundle_roots`** adopts decision 4: the outermost directory with
        an `index.md` and a concept beneath it, plus `okf_version` nested
        roots, with no depth-3 limit (decision 50). Its `references/`
        exclusion stays.

##### 45. `io-insights/src/io_insights/core/okf_read.py` (Modify)

*   **Type / Scope:** Python core, the reader.
*   **Proposed Changes:** `resolve_repo_path` gains a fourth branch: a
    relative path, resolved against `bundle_root` and then made relative to
    `repo_root` (C6). It returns `""` if the result leaves `repo_root`.
    Scope descriptors still return `""` because they contain whitespace
    (`okf._is_pathish`).

##### 46. `io-insights/tests/test_okf_read.py`, `io-insights/tests/test_okf_discovery.py` (Modify)

*   **Type / Scope:** unit tests.
*   **Proposed Changes:**
    *   **`test_okf_read.py`:** the `../src/a.py` resolution, and a
        `/design/x.md` citation that resolves after the rewrite (fourth
        review, M4).
    *   **`test_okf_discovery.py`:** `bundle_roots` under decision 4. Cases:
        `docs/` is io-insights' root even with a typed `AGENTS.md` at the
        repository root, an `index.md` with no concept beneath it makes no
        bundle, a nested `okf_version` root, and no depth limit. Its
        existing depth-3 and inferred-root cases are rewritten. These run
        the shared discovery cases from the fixtures (decision 50).

##### 47. `io-insights/tests/fixtures/okf-fixtures/` and `io-insights/scripts/fetch-okf-fixtures` (Create)

*   **Type / Scope:** test data and fetch script.
*   **Proposed Changes:** the fixtures archive at the pinned version,
    downloaded and checked against `okf-check.sha256` by the script, and
    cached under the same cache directory. Tests read the extracted copy,
    and fail with the fetch command when it is missing. This makes the
    reader tests need the network once per version (third review, N12).

##### 48. `io-insights/scripts/okf-validate` (Modify)

*   **Type / Scope:** gate script.
*   **Proposed Changes:** it becomes `exec scripts/okf-check docs "$@"`,
    keeping its name for muscle memory.

##### 49. `io-insights/.agents/skills/finalize-branch/SKILL.md`, `io-insights/.claude/skills/finalize-branch/SKILL.md`, `io-insights/AGENTS.md`, `io-insights/docs/design/okf-authoring-guide.md` (Modify)

*   **Type / Scope:** agent guidance and docs.
*   **Proposed Changes:**
    *   **The skill:** step 5 becomes `scripts/okf-check docs`, the whole
        bundle (decision 41). The `.claude` copy is checked for the same
        step.
    *   **AGENTS.md §16:** cites okf-check codes instead of restating rules.
    *   **The guide:**
        *   keeps only io-insights' binding (§8 evidence mapping, §10
            privacy);
        *   replaces the generic rules with a link to okf-check's rule docs
            and guide;
        *   corrects the path sentence at its "every path is bundle-relative"
            line (third review, B3);
        *   fixes the wrong section citations (§9 → §10, §7.1 → §4).
    *   **Logs:** each edited directory's `log.md` gets an entry.

### Stage 3: the exporter switch (about 8.5 days, by 2027-04-16)

#### io-insights

##### 50. `io-insights/src/io_insights/core/okf_check.py` (Create) and package data `io-insights/src/io_insights/data/okf-check.pin` (Create, generated at build)

*   **Type / Scope:** Python core.
*   **Proposed Changes:**
    *   `binary() -> Path`, the section 12 lookup;
    *   `run(staging: Path) -> Result`, with the fixed flags and a 60-second
        timeout, returning the parsed JSON, the exit code and stderr;
    *   `CheckUnavailable`, raised with a `reason`.

    The pin file holds the version and the checksum lines, copied from the
    repository's `.okf-check.yaml` and `okf-check.sha256` by a build hook
    in `pyproject.toml`, so a `uv sync` install knows its pin.

##### 51. `io-insights/src/io_insights/pipeline/export_okf.py` (Modify)

*   **Type / Scope:** Python pipeline.
*   **Proposed Changes:** `write_bundle` follows section 12's six steps:
    *   build staging in `<outdir>/bundle/.staging-<pid>/` with hard links
        for references;
    *   call `okf_check.run`;
    *   map the result to events, including the new `okf.check_failed`;
    *   refuse on exit 1, 2 or `CheckUnavailable`;
    *   delete staging in a `finally`;
    *   otherwise run today's write path unchanged.

    The other changes:
    *   `previous` filters reserved names by basename (M5);
    *   the C12 listed/unlisted rule and `okf.prior_unreadable`;
    *   a `.staging-*` cleanup at the start of the function.

##### 52. `io-insights/src/io_insights/core/okf_export.py` and `io-insights/src/io_insights/templates/okf/index.md` (Modify), `io-insights/src/io_insights/templates/okf/dir_index.md` (Create)

*   **Type / Scope:** renderer and templates (decision 34, C13).
*   **Proposed Changes:**
    *   `render_index` takes only root-level concepts and adds `kb/` and
        `sections/` subdirectory entries;
    *   a new `render_dir_index(directory, concepts)` writes `kb/index.md`
        and `sections/index.md` with `<name>.md` links and descriptions, and
        `write_bundle` calls it;
    *   the module docstring's `validate_document` references become
        okf-check.

##### 53. `io-insights/src/io_insights/core/run_events.py` (Modify)

*   **Type / Scope:** Python core.
*   **Proposed Changes:** `REDACTED_FIELDS` gains `got`, `expected`, `key`
    and `fix` (decision 43).

##### 54. `io-insights/src/io_insights/pipeline/publish_okf.py` (Modify)

*   **Type / Scope:** Python pipeline, gated.
*   **Proposed Changes:** `_existing_concept_paths` filters reserved names
    by basename, the same as item 51. No other change while it is gated
    (decision 23).

##### 55. `io-insights/scripts/build-macos`, `io-insights/io_insights_bundle.spec` (Modify)

*   **Type / Scope:** frozen build.
*   **Proposed Changes:**
    *   `build-macos` runs `scripts/okf-check --print-path` for the darwin
        universal binary, adds it to the bundle through the spec's
        `binaries`, and after the build runs the bundled
        `okf-check version`, failing unless it prints the pinned version;
    *   it never looks on `PATH` for okf-check.

    There is no frozen-build test today: `tests/test_frozen_build_contents.py`,
    which the reviews cite, does not exist. So this script check is the
    only one.

##### 56. `io-insights/tests/test_export_run_events.py` (Modify)

*   **Type / Scope:** unit tests.
*   **Proposed Changes:**
    *   the event-shape tests use a fake `okf_check.run` that returns
        canned JSON for each case: clean, warnings, exit 1, exit 2, missing
        binary and timeout;
    *   one integration test runs the real pinned binary on a small export,
        which `scripts/check` can rely on, because the binary is already
        required there.

##### 57. `io-insights/tests/test_okf_export.py`, `io-insights/tests/test_okf_bundle.py` (Modify)

*   **Type / Scope:** unit tests.
*   **Proposed Changes:**
    *   tests that relied on `validate_document` in the exporter move to
        okf-check-backed equivalents;
    *   add the staging parity test (C11), the reserved-basename test, the
        C12 listed and unlisted cases, and the per-directory index tests
        (C13).

##### 58. `io-insights/src/io_insights/core/okf.py`, `io-insights/tests/test_okf_frontmatter.py`, `io-insights/src/io_insights/core/okf_html.py` (Modify)

*   **Type / Scope:** deletion of the Python checker (Stage 3 exit), as a
    separate commit after item 51 has run on real exports.
*   **Proposed Changes:**
    *   **Delete** `validate_document`, `Finding`, `CONFORMANCE_CODES`,
        `FATAL_CODES` and the rule helpers. Only the helpers a rule needs
        go: `_attributable`, `_strip_code`, `_show` and the regexes.
    *   **Keep** the reader: `split_front_matter`, `trust_tier`,
        `verification_events`, `claim_blocks`,
        `differs_only_by_generated_at`, `bundle_roots`, `okf_documents`,
        `RESERVED_FILENAMES` and `is_instant` if the reader uses it.
    *   **Tests:** remove the checker tests, whose cases already live in the
        fixtures.
    *   **`okf_html.py`:** update the docstring at its `validate_document`
        reference.
    *   **Imports:** `scripts/okf-validate` no longer imports `okf`.

### Verification Plan

*   **Compilation/Build:** `go build ./...` and `goreleaser build --snapshot`
    in okf-check. `scripts/build-macos` in io-insights at Stage 3.
*   **Unit Tests:**
    *   in okf-check, `go test ./...`: package tests, fixtures, codegen diff,
        docs test and judge tests;
    *   in io-insights, `scripts/check`, which runs `python -m unittest
        discover tests` and the new OKF stage.
*   **Static Analysis & Linting:** `go vet`, `staticcheck` and
    `gofmt -l` (empty) in okf-check. `ruff` in io-insights, as today.
*   **Test Coverage Requirement:** every active code has at least one
    `fixtures/fail/` case, enforced by item 25. No percentage target: the
    fixtures are the coverage measure the decisions rely on.
*   **E2E/Integration:**
    *   Stage 0: the fixtures run against `okf.py`, and every difference is a
        ledger entry with a decision.
    *   Stage 1: `okf-check` exits 0 on yz-docs at `block: house` in its new
        workflow. The agent test passes, and its record is committed.
    *   Stage 2: `scripts/check` is green with the OKF stage at `house`. The
        ledger against `validate_document` holds only decision-traced
        entries. A full `docs/` check is no slower than the Stage 0
        baseline.
    *   Stage 3: a real export of io-insights itself and of yz-docs
        completes with okf-check, its events match the section 12 table,
        and a deliberately broken actor refuses the export. Only then does
        item 58 land.

--------------------------------------------------------------------------------

<!-- Provenance: Indicates initial structural drafting via EGM. -->

*Drafted with EGM*
