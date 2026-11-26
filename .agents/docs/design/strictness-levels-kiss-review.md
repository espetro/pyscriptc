# Strictness levels: the KISS/YAGNI review and the unsafe-bypass question (#1332)

Status: research note, 2026-11-26, on the landed surface of #1315 (PR #1396).
It reviews, it does not redesign: every recommendation below is keep, cut-or-defer, or reject-with-a-reopening-evidence-bar. The decisions.md row proposed at the end is what turns it into a decision.

Refs: #1332, #1315 (the levels feature, closed by #1396), `.agents/docs/design/typing-quality-lane.md` (the note that specified the levels), `src/pyscriptc/typinglevels.py`, `src/pyscriptc/cli/_report.py` (`_typing_policy`), `tests/test_typing_levels.py`, `docs/reference/config.md` and `docs/guide/python-subset.md` (the shipped docs).

## Summary

The strictness-lane surface landed at the KISS floor: four report-only levels, one config table, one raise-only pragma, one static scan. Nothing in it admits, lowers or gates. This note reviews the surface piece by piece against the KISS/YAGNI bar and answers the one open surface question the typing-quality-lane note left behind: does the lane need an unsafe bypass (a suppression comment), the `# type: ignore` of this lane?

Two verdicts:

1. **Keep the landed surface as is.** Each of its ten pieces survives the review; three named pieces carry standing triggers (below) rather than redesigns. The one citation-level finding: the pragma's raise-only rule is accurately attributed to Pyright's mode comments (issue 4321, closed as designed), and pyscriptc is deliberately *narrower* than Pyright because it has no per-rule surface at all - which is exactly the piece of Pyright's file-comment surface that does allow downgrades.
2. **Reject the unsafe bypass until evidence.** There is no `# pyscriptc: ignore`. Three reasons, one per paragraph below: the compiler's real errors (fences) are not comment-bypassable by design, so a suppression comment could only hide report rows; the coarse valve already exists (`off` globs are file-level suppression); and every peer that shipped suppression comments had to ship a second mechanism to police their rot, which is two features bought before the lane has a single measured false positive. The shape to adopt if evidence arrives is recorded, not scheduled.

## What landed (the inventory under review)

`src/pyscriptc/typinglevels.py`, about 900 lines with tests beside it in `tests/test_typing_levels.py`:

- the four levels `off/basic/standard/strict` (`LEVELS`), project default `standard`;
- the severity algebra `severity_for`: `off` suppresses everything; `basic` and `standard` report the missing-annotation family (PS1001, PS1202) as warnings; `strict` adds the construct-form family (PS1103, PS1104, PS1105, PS1106) as errors;
- `reports_coverage`, the one boolean that separates `basic` from `standard`;
- `TypingPolicy`: the `[tool.pyscriptc.typing]` table (`mode` plus one glob list per level), longest-pattern-wins resolution with strictest-on-tie, and the strictest-of-config-and-pragma combination;
- the hand-written glob matcher (`_glob_here`), written out instead of delegating to `fnmatch`;
- the pragma parser (`PRAGMA_RE`, `parse_pragma`): a whole-comment match on `# pyscriptc: typed=<level>`, unknown level is an error, several pragmas in one file resolve to the strictest;
- the static scan (`scan_tree` and its helpers): the missing-annotation findings the lowerer would fence, and the construct forms that fence in the annotation itself, read from the lowerer's own tables (`_lowerer_tables`, `_HANDLE_ALIASES`, `alias_name`) so the report cannot name a construct two ways.

Wiring: `src/pyscriptc/cli/_report.py` resolves the policy from the project above the source (`_typing_policy`, malformed table falls back to the default), `src/pyscriptc/config.py` validates the subtable into PS3102, and the docs sections in `docs/reference/config.md` and `docs/guide/python-subset.md` document the table, the globs and the pragma.

## KISS review, piece by piece

### 1. Four named levels, default `standard` - keep

The names are the field's converged answer (Pyright's `typeCheckingMode`), so a user's prior from Pyright config transfers for free. A fifth name or a rename buys nothing and breaks the transfer. The default is the Pyright default for a project that configures nothing, which means the do-nothing configuration is the one every Pyright user already understands.

### 2. `basic` versus `standard`, differing only in the coverage flag - keep, with a review trigger

The honest description of the difference is one boolean: `standard` additionally surfaces the coverage numbers that say how much of the unannotated surface the pins would lift. That is close to too fine to be two levels. It stays, for three reasons: the names match the peer shape users know; the marginal cost is a boolean, not a code path; and collapsing two published levels later is a smaller event than splitting them. **Standing trigger:** if, after one release of the lane being on by default, the reports show no project resolving to `basic` (nothing in the wild pins the level), collapse `basic` into `standard` and keep the name as an alias that maps to `standard`. Record the collapse in this note's follow-up rather than in a new design round.

### 3. Mode plus per-level glob lists, and no per-rule table - keep; the per-rule table is the YAGNI trap

The landed config is four level names, each optionally carrying a list of globs. What it deliberately does not have is a table of per-rule severities (`reportPrivateUsage = "warning"`-style), the shape mypy per-module overrides, Pyright's full severity table and ty's per-rule config all carry. That table is the 80/20 line: four levels times two finding families is the whole spread any user has asked for, and a per-rule surface multiplies the config grammar, the validation surface (`PS3102` cases) and the documentation, for a knob nobody can set without first understanding the families better than the report explains them. The peer evidence cuts both ways - Pyright's table is large precisely because Pyright is a checker with hundreds of rules; this lane reports two families from one scan. **Reject with a reopening bar:** a per-rule table enters only if the scan grows a third family and users ask to mute one family inside one glob - and then it enters as per-glob-per-family, not per-rule-code.

### 4. The raise-only pragma - keep; the attribution is right, and narrower-than-Pyright is deliberate

`# pyscriptc: typed=` may only raise a file's level over the config, never lower it. The attribution in the code docstring is accurate: Pyright's maintainer closed issue 4321 as designed, stating that `# pyright: basic` and `# pyright: strict` "override the configured ruleset only in cases where those rules *increase the severity*" and "you cannot do the opposite". One precision the review adds: Pyright does allow downgrades through a different part of its file-comment surface, the per-rule forms (`# pyright: reportPrivateUsage=false`), which is exactly the surface pyscriptc does not have (piece 3). So the landed rule is not just Pyright's mode rule - it is the mode rule with the per-rule escape removed, which is the stronger ratchet and the smaller surface. A strict glob list therefore holds: the way out of strict for a file is taking it out of the globs, visible in the config diff, not a comment added inside the file.

### 5. Strictest-of-config-and-pragma, strictest-among-pragmas - keep

Both resolutions are total orders over the same four names, so there is no case the algebra has not answered: several globs match, the strictest wins on ties; several pragmas appear in one file, the strictest wins. The alternative - last one wins, or first one wins - makes a level depend on comment order, which is the kind of nondeterminism that turns into a heisenbug in a report nobody can reproduce.

### 6. Longest-pattern-wins, strictest-on-tie, order-stable - keep

The resolution key is `(len(pattern), rank(level), index)`: the most specific pattern wins, an overlap between two globs of different levels never lowers a level by accident, and between equal candidates the config's own order decides deterministically. mypy's per-module section list and Pyright's glob lists both answer the same question with the same convention (most specific wins), so this is also the prior users carry. Nothing to change.

### 7. The hand-written glob matcher - keep, and it is the review's one justified reimplementation

`_glob_here` is about 60 lines reimplementing shell-style matching. The delegation targets were checked and both fail the documented semantics: `fnmatch` is not path-aware (`*` matches across `/`, so `src/*` would swallow `src/a/b.py` and the per-level lists would silently mean something other than what `docs/reference/config.md` says), and `pathlib` only documents recursive `**` matching from Python 3.13 (`PurePath.full_match`) while the tool host runs on CPython 3.11 through 3.15. The cost of keeping it is real but bounded: it is pure string algebra with no I/O, and the level algebra tests must keep covering the three documented behaviors (`*` stays in a segment, `**` crosses, `?` does not cross). **Standing trigger:** if the matcher ever grows a feature beyond those three (`[...]` classes, brace expansion), stop and re-ask whether the config should adopt `pathlib` semantics wholesale instead.

### 8. The scan's structural re-read of the lowerer's peek - keep, with the floor rule made explicit

The largest piece of the module is the static scan, and its one structural risk is that it re-derives, structurally, facts the lowerer proves with its full peek: which return annotations a body proves (`_body_proves_return`), which empty displays the body pins (`_name_is_pinned`). Two implementations of "what proves a type" can drift. The code already buys the important half of the safety - the scan reads the lowerer's own name tables (`_lowerer_tables`, `_HANDLE_ALIASES`, `alias_name`) rather than re-spelling which names lower - and the docstring already names the contract: the report is a floor on what a project sees, never an exact replay, and "a finding the compiler later proves away costs the reader nothing". The review makes the rule explicit so the drift has a direction:

- over-reporting (the scan flags, the compiler proves away) is acceptable and costs a warning at most;
- under-reporting a family the compiler actually fences is a bug and is fixed in the scan, not explained away;
- the scan stays structural. No call graph, no cross-module facts, no value-level constant folding. When a finding family cannot be decided structurally, it moves out of the scan and into the diagnostics the compiler itself emits, and the scan drops the family.

### 9. The pragma parses strictly and unknown levels error - keep

A typo (`# pyscriptc: typed=stirct`) is a `TypingLevelError` that the config layer surfaces, never a silent no-op. This is the fence-never-guess house rule applied to the quality lane: a lane whose whole job is legibility cannot itself be illegible about its own configuration. Bare-word pragmas, short spellings and prefix matches are all ways to reintroduce the silent no-op, and all stay out.

### 10. `typing_report` reads each file a second time for the pragma - keep, with a fix trigger

`scan_files` parses each file once for findings; `typing_report` then reads each file again to run `parse_pragma`. On a large tree that is two passes of I/O per file for the typing section. It is not worth a cache today: the reads are sequential, the section is report-time, and no benchmark has flagged it. **Standing trigger:** the inference bench suite (the `#1328` lane, `benchmarks/baseline/inference.py`) already records per-tree wall times; if the typing section shows up in its numbers after the default-on rollout, memoize the pragma pass into the scan pass (the AST is already in hand; only the comment scan is missing) rather than adding a cache layer.

### 11. Deferred, and staying out

Each of these was priced during the design round and stays out:

- **A fifth `strong` level** (Sorbet's no-untyped audit sigil). Gated in the typing-quality-lane note on users actually asking for an audit mode after the default-on rollout. It has no lowering meaning either; it is a fourth severity table for a demand that does not exist yet.
- **A `--typed` force flag** (Sorbet's adoption-experiment tool, which runs the tree at a forced level). The globs plus the pragma already express every experiment it enables, in config the team can review; a flag that overrides config from the command line breaks the property that the report a CI job prints is the report the config produced.
- **Pragma spellings and aliases** (`typed: strict`, `typed=Strict`, prefix matches). All reintroduce the silent no-op of piece 9.
- **Typing exclude globs** (a `typing.exclude = [...]` list, mypy/Pyright style). The `off` level's glob list is already exactly that, spelled in the same table everything else is spelled in.

## The unsafe-bypass question

The typing-quality-lane note settled what levels do; the one surface question it left open is whether the lane needs a suppression comment - the per-line `# type: ignore` of this lane. The survey first, then the answer.

### What the peers actually ship

- **mypy** silences per line with `# type: ignore`, narrowed to codes as `# type: ignore[code]` ("you are less likely to silence unexpected errors", and the code documents the comment's purpose), and per file with `# mypy: disable-error-code="..."` (https://mypy.readthedocs.io/en/stable/common_issues.html, "Spurious errors and locally silencing the checker").
- **Pyright** silences per line with `# pyright: ignore [rule, ...]` and per file with mode or per-rule comments. Its maintainer declined a file-wide `# pyright: ignore` twice: the per-rule forms already express it, and the file-level `# type: ignore` is "an ill-conceived mechanism" (https://github.com/microsoft/pyright/issues/4321).
- **Ruff** ships `# noqa` and `# noqa: CODE`, and then ships a second mechanism to police the first: RUF100, the unused-noqa rule, which flags suppressions that no longer suppress anything and can remove them with `--fix` (https://docs.astral.sh/ruff/linter/#error-suppression).
- **TypeScript** grew `// @ts-expect-error` in 3.9 precisely because `// @ts-ignore` rots silently: "if there's no error, TypeScript will report that `// @ts-expect-error` wasn't necessary", while `@ts-ignore` "will do nothing if the following line is error-free" (https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html).
- **Sorbet** ships no per-line ignore at all: the sigils are the bypass ladder, and the escapes are type-level (`T.unsafe`, `T.untyped`), not comment-level, so every escape is visible in the code as an expression (https://sorbet.org/docs/static).

The pattern across all five: suppression is never one feature. Bare suppressions rot, so every ecosystem that shipped them shipped an auditor for the rot - RUF100, `@ts-expect-error`, mypy's code-specific form existing to keep the purpose documented. The bypass is two features, and the second is what keeps the first honest.

### Why pyscriptc rejects it now

1. **The thing peers silence is not the thing this lane reports.** In the peer tools the suppression comment mutes checker *errors* - the tool's output is the product. In pyscriptc the product is the binary, its errors are the PSxxxx fences, and the fences are not comment-bypassable by design: the way past a fence is routing (`@island` demotion, the `[tool.pyscriptc.imports]` lanes), which changes what the program does and is visible in the plan map. A suppression comment on this lane could do exactly one thing: hide a row from the `assess`/`infer` quality report. Cosmetic muting of a report that already has granularity controls is not a feature.
2. **The coarse valve already exists.** A file (or glob) that does not want the lane's attention sets its level to `off` in the config, where the team reviews it. What is missing without suppression comments is only the line-level valve, and no false-positive row has been measured yet: the scan is a deliberate floor that over-reports by construction, and an over-reported row costs a warning in `basic`/`standard`.
3. **The ratchet would be the casualty.** The raise-only pragma plus strict globs give the Sorbet adoption story: raise globs file by file, the report shows the split. Line-level ignores break the legibility that makes the ratchet work - a file could raise itself into strict and muzzle every strict finding with comments, and the config diff that says "this file is strict" would be lying about what the file actually shows. Sorbet's answer (no comment bypass at all) is the one consistent with a ratchet lane.

So the review's answer to the unsafe-bypass question is: **reject until evidence, record the shape now.** The reopening evidence bar: the default-on rollout's coverage report shows real, reproducible false-positive mass - findings the compiler proves away at scale on trees users actually compile, concentrated enough that `off`-globbing the file is the only remedy. Not "someone finds one wrong row"; the floor contract already prices single wrong rows as acceptable.

### The shape to adopt when evidence arrives (recorded, not scheduled)

- Form: `# pyscriptc: ignore[PS1001]` on the finding's line. **No bare form.** The code list is what mypy's docs argue for and what keeps the comment's purpose readable in review.
- An optional explanation slot (`# pyscriptc: ignore[PS1001] -- the body pins this below`), matching the convention the peer ecosystems converged on of documented suppressions.
- Unused ignores become report rows (the `@ts-expect-error`/RUF100 model, not the `@ts-ignore` model): a suppression that suppresses nothing is surfaced, never silently tolerated, so rot is visible by default.
- Report-only, forever: an ignore never changes what compiles, never names a PSxxxx fence code, never touches exit codes. The fences route; they do not muzzle.
- Strict-level files surface a suppressed count next to their findings, so the ratchet stays legible even where ignores exist.

## Recommendation

**Do now: nothing in the compiler.** The landed surface passes the review; this note is its record.

Keep the three standing triggers:

1. collapse `basic` into `standard` (alias kept) if no project in the wild resolves to `basic` after one release of the lane on default;
2. the glob matcher grows nothing beyond `*`, `**`, `?`, and the level algebra tests keep covering the three documented behaviors;
3. memoize the pragma pass into the scan pass when the `#1328` bench suite flags the typing section's wall time.

Reject, each with its reopening bar: per-rule severity tables (bar: a third family plus a per-glob mute request), suppression comments (bar: measured false-positive mass in the rollout's coverage report), a fifth `strong` level (bar: users ask for an audit mode after the rollout), a `--typed` force flag (bar: none foreseen - globs plus pragma express its uses in reviewable config), pragma spellings (never), typing exclude globs (redundant with `off` globs).

### Proposed decisions.md row (not appended)

| Date | Decision | Why (one line) | Rejected alternatives | Detail |
| --- | --- | --- | --- | --- |
| 2026-11-26 | The strictness lane ships as landed (four report-only levels, mode plus per-level globs, raise-only pragma, structural scan) and carries no suppression surface: no `# pyscriptc: ignore`, no per-rule table, no fifth level; three standing triggers (collapse `basic` if unused, matcher growth stop, pragma-pass memoization on bench evidence) and recorded shapes for what evidence would reopen | Levels only configure the report and fences are not comment-bypassable by design, so suppression would be cosmetic muting bought before a single false positive is measured, while every peer that shipped it had to ship a second auditor mechanism (RUF100, `@ts-expect-error`) to keep it honest | `# pyscriptc: ignore[...]` now (rot auditor included, ratchet legibility lost), per-rule severity tables (config grammar for no demand), Sorbet's `--typed` force flag (config-override breaks report reproducibility), a `strong` audit level (no demand), bare/loose pragma spellings (silent no-op) | `.agents/docs/design/strictness-levels-kiss-review.md` |

## Issues to file

- None new. The three standing triggers live in this note and are re-read at the v0.12.0 design round; the suppression shape is recorded here and takes its own issue only if the evidence bar is met.
- Optional, owner's call: a gate ticket for the suppression evidence, opened blocked like the docstring-suggestions gate in the typing-quality-lane note, with the evidence requirement attached (measured false-positive mass on the rollout's coverage report).

## Related

- `.agents/docs/design/typing-quality-lane.md`: the note that specified the levels (Option S2) and left the bypass question open.
- `.agents/docs/design/loose-typing-and-default-inference.md`: the default-on rollout whose coverage report is the bypass question's evidence source.
- `src/pyscriptc/typinglevels.py`, `src/pyscriptc/cli/_report.py`, `tests/test_typing_levels.py`: the surface under review.
- `docs/reference/config.md`, `docs/guide/python-subset.md`: the shipped documentation of the table, the globs and the pragma.

## Sources

- https://mypy.readthedocs.io/en/stable/common_issues.html (per-line `# type: ignore`, the code-specific form, the file-level `disable-error-code`)
- https://github.com/microsoft/pyright/issues/4321 (mode comments raise-only, closed as designed; the "ill-conceived" file-level ignore)
- https://github.com/microsoft/pyright/blob/main/docs/comments.md (the `# pyright:` file-comment forms, including the per-rule downgrades pyscriptc omits)
- https://docs.astral.sh/ruff/linter/#error-suppression (`# noqa` / `# noqa: CODE`, RUF100 unused-noqa with `--fix`)
- https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html (`// @ts-expect-error` errors when unused; `// @ts-ignore` does not)
- https://sorbet.org/docs/static (the five sigils, no per-line ignore, `T.unsafe`/`T.untyped` as the escapes)
