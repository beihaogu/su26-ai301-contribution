# Contribution [1]: Domain-grouped summary of checked links

**Contribution Number:** 1

**Student:** Beihao Gu

**Issue:** https://github.com/lycheeverse/lychee/issues/1998

**Status:** Phase II In Progress

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
