# Contribution [1]: Domain-grouped summary of checked links

**Contribution Number:** 1

**Student:** Beihao Gu

**Issue:** https://github.com/lycheeverse/lychee/issues/1998

**Status:** Phase IV Completed
---

## Why I Chose This Issue

I chose this issue from lychee, a fast Rust-based link checker, because it has clear scope: the underlying --host-stats feature already exists, and the maintainer has explicitly identified what needs to be added (a summary line showing total domains and links checked). My background in Rust through CSE 231 (compilers) gives me the language familiarity needed, and the issue is small enough to complete in 2-3 weeks while still touching real user-facing functionality. The maintainer is active (last interaction in March 2025) and has marked this as good first issue with detailed guidance in the comments, which reduces ambiguity for a first contribution.

---

## Understanding the Issue

### Problem Description

lychee currently prints a run-level summary (totals across all checked links: OK,
errors, redirects, etc.), but it does not break those totals down per domain. When a
run covers hundreds of links across many external hosts, users cannot quickly see
which domains dominate the link set or whether failures cluster on a single host.

### Expected Behavior

A new opt-in CLI flag (`--domain-summary`, with a short form like `-D`) should print a
"Domain Summary" section after the existing stats block. It should list each unique
domain with the count of links checked against it and a per-status breakdown
(e.g. `[✓ 65, ✗ 2]`). For runs with many domains it should default to a top-N view
and support `all` plus a `--domain-summary-min` threshold flag.

### Current Behavior

Running `cargo run -- <inputs>` prints the standard summary (Total / OK / Errors Excluded / Timeouts / etc.) with no per-domain grouping. There is no `--domain-summary`
flag in `lychee --help`.

### Affected Components

- `lychee-bin/src/options.rs` — CLI flag definition (clap)
- `lychee-bin/src/stats.rs` — `ResponseStats` aggregation
- `lychee-bin/src/formatters/stats/` — rendering of the summary block (plain, JSON,
  markdown variants)
- `lychee-bin/tests/cli.rs` — integration tests for CLI flag behavior

---

## Reproduction Process

### Environment Setup

Cloned my fork and built locally with the standard Rust toolchain (no devcontainer in
this repo, plain Cargo workspace):
```bash
git clone https://github.com/beihaogu/lychee.git
cd lychee
cargo build
cargo test
```
The build succeeded on the first attempt on macOS with a current stable Rust
toolchain — no system dependency issues to document. `cargo test` passes against
`main` before any of my changes, which gives me a clean baseline to compare against.

### Steps to Reproduce

Because #1998 is a feature request rather than a bug, "reproducing" here means
confirming the feature is absent and capturing the current summary output as a
baseline.

1. From the repo root, run lychee against a small fixture with several distinct
   domains:
```bash
   cargo run -- --no-progress fixtures/TEST_HTML5.html
```
2. Observe the printed summary block. It contains run-level totals (Total, OK,
   Errors, Excluded, Timeouts, Unknown, Redirected) but no per-domain breakdown.
3. Run `cargo run -- --help | grep -i domain`. There is no `--domain-summary` flag,
   confirming the feature does not exist.
4. **Expected (after my change):** A `Domain Summary` section appears below the
   existing summary when `--domain-summary` is passed.
5. **Actual (today):** No such section, no such flag.

### Reproduction Evidence
- **Working branch:** https://github.com/beihaogu/lychee/tree/feat-issue-1998-domain-summary
- **Baseline help output:** [notes/baseline-help.txt](https://github.com/beihaogu/lychee/blob/feat-issue-1998-domain-summary/notes/baseline-help.txt) — `grep -i domain` returns no matches, confirming `--domain-summary` does not exist on `main`.
- **Baseline summary output:** [notes/baseline-output.txt](https://github.com/beihaogu/lychee/blob/feat-issue-1998-domain-summary/notes/baseline-output.txt) — shows the current summary block (Total / OK / Errors / Excluded / Timeouts / Unknown / Redirected) with no per-domain breakdown.
- **My findings:** The feature is absent as described. The `Response` objects already carry full `Uri`s, so the data needed for grouping is reachable from `ResponseStats::add` without plumbing changes — only aggregation and rendering need to be added.

---

## Solution Approach

### Analysis

The current `ResponseStats` struct already aggregates `Response`s into per-status
buckets. Each `Response` carries the full `Uri`, which exposes `host()`. So the data
needed for a domain breakdown already flows through the stats pipeline; nothing new
needs to be plumbed end-to-end. The work is (a) add a `domain_stats: HashMap<String,
PerDomainStats>` field, (b) populate it in `ResponseStats::add`, (c) gate rendering
on the new CLI flag, and (d) implement the three formatter variants (text, JSON,
markdown).

### Proposed Solution

Add an opt-in `--domain-summary` flag and an aggregation field on `ResponseStats`.
When the flag is set, render a new section after the existing summary in each
formatter. Default to "top 10 domains by count"; accept `all` or `top:N` as values;
add a `--domain-summary-min N` threshold flag.

### Implementation Plan

**Understand:** lychee should optionally print a per-domain breakdown of checked
links after the run-level summary, with status counts per domain and a configurable
display limit.

**Match:** `ResponseStats` already groups responses by `Status` using `HashMap`s
keyed on category. The new per-domain map follows the same pattern, just keyed on
`uri.host_str()` instead of status. The existing `--verbose` / `--mode` style flags
in `options.rs` are the template for the new clap flags.

**Plan:**
1. Add `domain_summary: Option<DomainSummaryMode>` (None / Top(usize) / All) and
   `domain_summary_min: Option<usize>` to `lychee-bin/src/options.rs`, with clap
   derive annotations matching the issue's suggested CLI shape.
2. Add `domain_stats: HashMap<String, DomainCounts>` to `ResponseStats` in
   `lychee-bin/src/stats.rs`, where `DomainCounts` holds totals and per-status
   counts.
3. Update `ResponseStats::add` (or equivalent ingest point) to update
   `domain_stats` using `response.uri().host_str()`. Skip entries with no host
   (mailto, file paths) — they should not appear in the domain summary.
4. Add a `format_domain_summary` helper in `lychee-bin/src/formatters/stats/` and
   call it from the plain, JSON, and markdown formatters when the flag is set.
   Plain text matches the format in the issue body; JSON always includes the full
   map regardless of display limits.
5. Add CLI integration tests in `lychee-bin/tests/cli.rs` covering: flag off
   (baseline unchanged), flag on (section appears), `--domain-summary=all`,
   `--domain-summary-min`, JSON output shape.

**Implement:** branch `feat-issue-1998-domain-summary` on my fork — see link above.

**Review:** Before opening the PR I will re-read `CONTRIBUTING.md`, run
`cargo fmt`, `cargo clippy --all-targets -- -D warnings`, and `cargo test`, and
follow the project's commit-message convention (conventional commits / present
tense) based on recent merged PRs.

**Evaluate:** The new integration tests must pass; the existing test suite must
continue to pass; manual run against a fixture with ~5 distinct domains must
produce a summary matching the format in the issue body.

---

## Testing Strategy


### Snapshot Tests Updated

- ✅ `formatters::stats::compact::tests::test_formatter` — embeds the compact host-stats block; expected string updated to `📊 Per-host Statistics (1 domains, 5 links checked)` (the dummy fixture uses one host with five requests).
- ✅ `formatters::stats::detailed::tests::test_detailed_formatter` — same update for the detailed formatter.

The markdown host-stats formatter has no existing snapshot test in `host_stats/markdown.rs`; the change there is exercised end-to-end by `cargo run --bin lychee -- --host-stats --format markdown`.

### Regression Check

Ran the full workspace test suite after the change:

```bash
cargo test
```

Result: all binary, library, and integration suites pass with no unrelated failures (78 unit tests in `lychee-bin` alone). Static checks also pass:

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
```

### Manual Testing

Ran the modified binary against the lychee README, which references ~23 unique domains across ~113 link checks:

```bash
cargo run --bin lychee -- --no-progress --host-stats README.md
```

Observed output now starts with:

```
📊 Per-host Statistics (23 domains, 113 links checked)
```

Compared against the baseline captured in `notes/baseline-output.txt` (bare `📊 Per-host Statistics`), confirming the change behaves as intended on real input.

---

## Implementation Notes

### Week 1 Progress (Jun 17–23)

**What I built:**

- Added the summary line `(N domains, M links checked)` to the `📊 Per-host Statistics` header in all three host-stats formatters: compact, detailed, and markdown.
- `N` = number of unique hosts (length of `HostStatsMap::sorted()`).
- `M` = sum of `HostStats::total_requests` across all hosts.
- Updated two embedded snapshot tests in `formatters/stats/` so they include the new header substring.
- Refactored `host_stats/compact.rs` to call `HostStatsMap::sorted()` once instead of three times.

**Engineering decisions:**

- **Reused `--host-stats` instead of introducing `--domain-summary`.** The original issue proposed a new flag, but maintainer comments on Jan 25–26 redirected the work onto the existing `--host-stats` feature, with `mre` explicitly noting that the summary line was the cheap, high-value piece. Introducing a parallel flag would have duplicated semantics.
- **Used `total_requests` as the "links checked" denominator.** The run-level `Total` counter counts every link occurrence across input files (including duplicates), which would have over-represented "links checked per domain." Summing `total_requests` from `HostStatsMap` keeps the summary line consistent with the rows below it.
- **Did not add singular/plural handling for `1 domain` vs `N domains`.** Kept the output format identical to the example `mre` posted in the issue thread; consistency with the maintainer's spec outweighs a minor grammar nit. Easy to revisit in PR feedback if requested.
- **Reduced redundant `sorted()` calls in `compact.rs`.** Each call clones the entire underlying `HashMap` and re-sorts. The other two formatters already cached the result in a local; this brings `compact.rs` in line with that pattern.

### Challenges Faced

The non-obvious blocker was that the only failing tests after my change lived in `formatters/stats/` (the **stats** module), not in `formatters/host_stats/` (the module I actually edited). The stats formatters build their final output by embedding the host-stats block via `format!("{response_stats}\n{host_stats}")`, so the snapshot strings asserted in `stats/compact.rs::test_formatter` and `stats/detailed.rs::test_detailed_formatter` included the per-host header verbatim. Once I located the right files and updated the expected strings to include `(1 domains, 5 links checked)` (matching the dummy fixture of one host with five requests), both tests passed. This was a useful first-hand lesson in how snapshot tests propagate coupling across modules in a Rust codebase — the test that fails isn't always in the file you changed.

### Code Changes

- **Files modified:**
  - `lychee-bin/src/formatters/host_stats/compact.rs`
  - `lychee-bin/src/formatters/host_stats/detailed.rs`
  - `lychee-bin/src/formatters/host_stats/markdown.rs`
  - `lychee-bin/src/formatters/stats/compact.rs` (snapshot test expectation)
  - `lychee-bin/src/formatters/stats/detailed.rs` (snapshot test expectation)
- **Key commit:** `feat(host-stats): add domain/link count summary to per-host stats header` on branch [`feat-issue-1998-domain-summary`](https://github.com/beihaogu/lychee/tree/feat-issue-1998-domain-summary).
- **Branch:** https://github.com/beihaogu/lychee/tree/feat-issue-1998-domain-summary

---

## Pull Request

**PR Link:** https://github.com/lycheeverse/lychee/pull/2251

**PR Description:** Adds `(N domains, M links checked)` to the `📊 Per-host Statistics` header in all three `--host-stats` formatters (compact, detailed, markdown), addressing the summary-line piece of #1998 flagged by maintainer `@mre` as "pretty quick to add." Also includes a drive-by refactor to reduce `HostStatsMap::sorted()` from three calls to one in `host_stats/compact.rs`.

**Maintainer Feedback:**
- *(None yet)*


**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

- End-to-end open-source contribution flow: fork, clone, branch, implement, test, rebase, PR, tag reviewer.
- Practical Rust exposure inside a real workspace (`lychee-bin` calling into `lychee-lib`'s ratelimit / host-stats module), including how formatter selection (compact / detailed / markdown) is layered on top of a shared `HostStatsMap`.
- Reading and updating Rust snapshot tests that assert against a whole formatted string, including tests that live in a *sibling* module because one formatter embeds another's output via `format!("{a}\n{b}")`.
- Reading a maintainer thread carefully enough to descope a feature request: the original issue proposed a new flag with modes and thresholds, but by the time I picked it up, the maintainers had already redirected the work onto a single summary line. Recognizing that and matching the actual ask (rather than the original ask) was the single most useful skill this contribution taught me.
- `git rebase upstream/master` in a live repo where a fresh upstream PR touched the same files I did, resolved cleanly with tests still green.

### Challenges Overcome

- **Locating the failing test.** After my first change, the only failing tests lived in `formatters/stats/` (the *stats* module), not in `formatters/host_stats/` where I actually edited. Tracing the `format!("{response_stats}\n{host_stats}")` composition in `stats/compact.rs::format()` made the embedding relationship obvious. This was a useful first-hand lesson in how snapshot tests propagate coupling across module boundaries.
- **Debugging a suspicious-looking test diff.** At one point the failure output made it look like an unrelated column's whitespace had shifted, which sent me down a wrong path. Getting unstuck required stepping back and re-reading the panic string carefully, since the diff was actually smaller than it appeared. Lesson: when a test failure looks impossibly broad, re-verify the diff character-by-character before hypothesizing complicated causes.
- **Rebasing on upstream that had touched the same snapshot tests.** Upstream PR #2243 added an IGNORED-links row to the same `stats/{compact,detailed}.rs` expected strings I had updated. The three-way merge produced a clean result but I still had to run the whole test suite to be sure.

### What I'd Do Differently Next Time

- Read the *entire* issue thread, including maintainer back-and-forth on scope, before writing a Phase II plan. My initial plan proposed a full new flag with mode/threshold options because I anchored on the original reporter's wording; only after re-reading the maintainer comments did I realize the actual ask had been pared down to just the header summary line. Earlier scope clarification would have made Phase II more accurate.

---

## Resources Used

- Issue thread: https://github.com/lycheeverse/lychee/issues/1998 (especially `@mre` and `@thomas-zahner` scope-narrowing comments on Jan 25 and 26, 2026)
- lychee `CONTRIBUTING.md` (build/test workflow reference)
- `lychee-lib/src/ratelimit/host/stats.rs` (`HostStatsMap::sorted()` definition, critical to knowing what data was already accessible in the formatter layer)
- Existing formatters as style reference: patterns in `detailed.rs` and `markdown.rs` reused when refactoring `compact.rs`
- Claude Code (Rust codebase navigation, snapshot-test diff diagnosis)
# Contribution [2]: [Issue Title]

**Contribution Number:** 2
**Student:** Beihao Gu
**Issue:** https://github.com/lycheeverse/lychee/issues/2142
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it improves the user experience by making excluded links much easier to understand and debug. It also gives me an opportunity to explore how Lychee represents and reports link statuses internally. Since I have already contributed to Lychee before, I hope to build on that experience and become more familiar with the project’s architecture.

---

## Understanding the Issue

### Problem Description

When lychee excludes a link, its verbose output reports only the generic message
`This is due to your 'exclude' values`. That message does not identify which of
several `--exclude` regular expressions matched. It is also misleading for links
that were excluded for a built-in reason, such as mail checking being disabled.
As a result, users must manually compare every excluded URL against every rule to
debug their configuration.

### Expected Behavior

In verbose mode, each excluded response should explain the specific reason for the
exclusion. A link matched by a user-provided regular expression should name the
original pattern (for example, `wikipedia\.org`). A mail link
excluded because mail checking is off should instead say
`Excluded: mail checking is disabled (use --include-mail to enable)`. The final
summary should continue to report only the aggregate Excluded count so normal
non-verbose output remains compact.

### Current Behavior

All excluded links receive the same status details, regardless of whether they
matched different user patterns or were skipped by built-in filtering. With three
separate `--exclude` values, the Wikipedia URL, license image, example-domain URLs,
and both mail addresses all print the identical generic explanation. The status
therefore preserves the fact that a link was excluded but loses why it was
excluded.

### Affected Components

- `lychee-lib/src/types/status.rs` — defines the unit variant
  `Status::Excluded` and its current generic `details()` text.
- `lychee-lib/src/filter/regex_filter.rs` — wraps `RegexSet`, but exposes only a
  boolean `is_match` result and not the original matching expression.
- `lychee-lib/src/filter/mod.rs` — combines user patterns and built-in exclusion
  rules into a single boolean `Filter::is_excluded` result.
- `lychee-lib/src/client.rs` — turns that boolean into `Status::Excluded`, which is
  where the reason is currently discarded.
- `lychee-lib/src/checker/mail.rs`, cache/retry code, and
  `lychee-bin/src/formatters/` — construct or pattern-match `Status::Excluded` and
  will need mechanical updates if the status begins carrying data.
- `lychee-bin/tests/cli.rs` and formatter tests — contain the existing generic
  output expectations and are the appropriate locations for regression coverage.

---

## Reproduction Process

### Environment Setup

I updated from the canonical repository rather than branching from my previous
contribution branch, then created a clean issue-specific branch:

```bash
git fetch upstream master
git switch -c codex/issue-2142-exclude-reason upstream/master
cargo build -p lychee
```

The branch is based on upstream commit `af73b4e0`. The build succeeded with the
stable Rust toolchain on macOS. One reproduction-specific complication is that the
current root `lychee.toml` contains `exclude_path = ["benches", "fixtures"]`.
Therefore, the command copied directly from the issue now skips `fixtures/TEST.md`
and reports zero inputs. Passing `--config /dev/null` bypasses the repository config
and allows the fixture to be checked without modifying it.

### Steps to Reproduce

1. From the repository root, build the current upstream version:

   ```bash
   cargo build -p lychee
   ```

2. Run the issue's three exclusion patterns against `fixtures/TEST.md`, using an
   empty config so the repository-level fixture exclusion does not hide the input:

   ```bash
   cargo run -p lychee -- \
     --config /dev/null \
     --no-progress \
     --verbose \
     --exclude 'wikipedia\.org' \
     --exclude 'licensebuttons\.net' \
     --exclude 'example\.com' \
     fixtures/TEST.md
   ```

3. Observe that the six excluded links all receive the same explanation:

   ```text
   [EXCLUDED] https://en.wikipedia.org/wiki/Static_program_analysis (at 7:1) | This is due to your 'exclude' values
   [EXCLUDED] https://licensebuttons.net/p/zero/1.0/88x31.png (at 16:2) | This is due to your 'exclude' values
   [EXCLUDED] http://example.com/ (at 20:1) | This is due to your 'exclude' values
   [EXCLUDED] https://example.com/ (at 21:1) | This is due to your 'exclude' values
   [EXCLUDED] mailto:test@example.com (at 23:1) | This is due to your 'exclude' values
   [EXCLUDED] mailto:test2@example.com (at 24:8) | This is due to your 'exclude' values
   ```

4. The expected result is for the first four entries to name their matching
   regular expression, while the two mail entries explain that mail checking is
   disabled. The summary totals should remain unchanged.

### Reproduction Evidence

- **Working branch:**
  https://github.com/beihaogu/lychee/tree/codex/issue-2142-exclude-reason
  (prepared from the latest `upstream/master`; no implementation commit at this
  phase).
- **Baseline command/output:** Captured above from commit `af73b4e0`. The excluded
  lines match the behavior reported in issue #2142. Network errors from the four
  non-excluded external links are unrelated to the bug; all six relevant links
  are filtered before any network request.
- **My findings:** The issue is reproducible. `RegexSet` retains the configured
  expressions and can identify matching indexes, but `RegexFilter` reduces that
  information to `bool`. `Filter::is_excluded` then combines user and built-in
  rules into another `bool`, and `Client::check` constructs the data-less
  `Status::Excluded`. By the time formatters call `Status::details()`, the original
  cause is no longer available.

---

## Solution Approach

### Analysis

The root cause is information loss between filtering and status formatting. The
regex crate's `RegexSet::matches` API can return the indexes of all matching
patterns, and `RegexSet::patterns` retains their original strings. Lychee currently
uses only `RegexSet::is_match`, so it knows that some expression matched but not
which one. `Filter::is_excluded` also treats user regexes, disabled mail checking,
scheme/IP rules, example domains, known false positives, and other built-in cases
as the same boolean outcome.

`Client::check` maps every true result to the unit variant `Status::Excluded`.
Finally, `Status::details()` has no data to inspect and returns the hard-coded
generic sentence. The formatters are not the source of the problem; they already
render `status.details()`. The reason must be captured in the library before the
filter result reaches the formatter layer.

### Proposed Solution

Introduce a structured `ExcludeReason` enum and change the status to
`Status::Excluded(ExcludeReason)`. Add a `matching_pattern` method to `RegexFilter`
that returns the first matching expression in configuration order. Replace the
client's boolean-only decision path with an `exclusion_reason` query that returns
`Option<ExcludeReason>`, while retaining `is_excluded` as a compatibility wrapper
for callers that need only a boolean.

`Status::details()` can then delegate to `ExcludeReason` for human-readable text.
The most important precedence rule is that disabled mail checking should be
reported before a user regex match, matching the issue's example where
`mailto:test@example.com` also matches `example\.com`. For HTTP example-domain
links, an explicit user pattern should be reported when both the built-in rule and
the user rule apply, because the purpose of this feature is to debug the supplied
`--exclude` values. Existing summary counts, status codes, and cache semantics
should remain unchanged.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Lychee can tell that a link is excluded but discards the triggering
rule before creating the response. The task is to preserve that reason through the
filter/client/status pipeline and expose it in verbose response details without
making the default summary noisier.

**Match:** `Status` already carries structured data for errors, timeouts, unknown
mail results, unsupported requests, and cached results, and each variant derives
its human-readable details from that data. `Status::Excluded(ExcludeReason)` follows
the same design. The existing plain, color, emoji, task, JSON, and JUnit formatters
all consume `ResponseBody`/`Status::details()`, so enriching the status should flow
through those formats without adding formatter-specific filtering logic.

**Plan:**
1. Define a public, non-exhaustive `ExcludeReason` enum in
   `lychee-lib/src/types/status.rs`, attach it to `Status::Excluded`, and format
   clear messages for user patterns, disabled mail checking, and other built-in
   exclusion categories.
2. Add `RegexFilter::matching_pattern` using `RegexSet::matches` and
   `RegexSet::patterns`, returning the first configured pattern that matches.
3. Add `Filter::exclusion_reason(&Uri) -> Option<ExcludeReason>` and preserve the
   current inclusion/exclusion precedence. Keep `is_excluded` as a boolean wrapper
   to avoid unnecessary call-site churn.
4. Update `Client::check` to construct `Status::Excluded(reason)`. Give other
   producers, such as disabled compile-time mail support and request chains,
   explicit fallback reasons.
5. Update exhaustive matches in cache, retry, statistics, color, and emoji code so
   behavior and aggregate counts stay unchanged.
6. Add unit tests for selecting the first matching regex and for reason precedence,
   plus a CLI regression test covering both a pattern-excluded URL and a mail URL.
   Update formatter/JSON/JUnit expectations that currently assert the generic text.
7. Before implementation is considered complete, run `cargo fmt --check`,
   `cargo clippy --workspace --all-targets --all-features -- -D warnings`, and
   `cargo test --workspace --all-targets --all-features`, followed by the original
   reproduction command to compare the before/after output.

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
