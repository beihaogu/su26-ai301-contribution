# Contribution [1]: Domain-grouped summary of checked links

**Contribution Number:** 1

**Student:** Beihao Gu

**Issue:** https://github.com/lycheeverse/lychee/issues/1998

**Status:** Phase III Completed
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
