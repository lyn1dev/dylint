---
name: auto-correct
description: Use when Dylint lints or lint libraries have been upgraded with `cargo dylint upgrade` and may have broken Rust or Clippy API uses.
metadata:
  version: "0.1.0"
  license: "MIT OR Apache-2.0"
  compatibility:
    requires:
      - shell access
      - cargo and git
---

# Correct lints after upgrading

Repair API uses broken by the upgraded Rust toolchain or `clippy_utils`. Use diagnostics to locate failures and historical changes to establish the right migration. Compilation and tests determine whether a repair succeeded.

## Scope

- If the user names a lint, work on that lint; if they name a library, cover every lint it defines. Otherwise, cover every lint in the current directory and its subdirectories, including nested workspaces. Inventory manifests, lint declarations, and Dylint workspace metadata; one `cargo metadata` call may miss nested workspaces.
- Preserve existing edits and the upgraded toolchain. Keep the selected `clippy_utils` revision unless an upstream fixture or dependency failure necessitates a later compatible revision. Repair upgrade-related failures in the lints in scope; identify unrelated failures separately.

## Establish the upgrade and failures

1. Inspect each package or workspace's manifest, toolchain file, and upgrade diff. Find the old nightly channel and the old and upgraded `clippy_utils` revisions, including revisions in `[workspace.dependencies]`. If the old channel is unavailable, use the errors to guide a broader history search.
2. Check every lint in scope with `cargo check --message-format=json`, grouping packages by workspace when useful. Compile test targets too, such as with `cargo test --no-run -p <package>`. Check exit statuses and read Cargo manifest warnings as well as compiler diagnostics, including primary spans, related notes, and secondary spans. A failed build with no usable spans is still a failure.
3. If compilation succeeds, run the relevant lint and UI tests before concluding that no repair is needed. Continue checking other lints when one fails.
4. Run the repository's relevant CI lint checks for affected workspaces, including `cargo clippy` where configured. Compilation can succeed while upgrade-related lint warnings remain.

## Find and apply migrations

1. Search Clippy's Rust source history between the old and upgraded `clippy_utils` revisions when both are known. Otherwise, search back from the upgraded revision through the old nightly's date, using an explicit midnight cutoff with a timezone (for example, `--since='2026-08-20 00:00:00 +0000'`); a bare `YYYY-MM-DD` can exclude same-day commits. Narrow by changed symbols with `git log -S` or `-G`; inspect candidate commits, their parents, and uses at the upgraded revision. Read an existing Clippy clone without changing its checkout, or use a temporary clone.
2. If Clippy history is insufficient, inspect Rust source and commit history for the target toolchain. Search relevant open and closed issues and PRs in `rust-lang/rust` and `rust-lang/rust-clippy`. Check that any proposed migration applies to the upgraded toolchain.
3. If a failing test comes from Clippy's own fixture or dependency at the selected revision, inspect later Clippy commits for a fix. Verify that any replacement revision uses the intended nightly and passes the affected test, then document why it was chosen.
4. Make a small source edit supported by that evidence. Preserve the lint's intended diagnostics, checking surrounding types, imports, signatures, and related call sites. For structural HIR changes, inspect lowering and visitor behavior; keep existing cases and diagnostic spans unless the new API requires a change. Resolve conflicting candidate migrations through more investigation rather than a token match alone.
5. Rerun the affected check and test-target compilation after each coherent edit. Use new errors to guide the next pass. Mark a lint unresolved only after the applicable Clippy, Rust, and discussion leads have been exhausted; continue with the remaining lints.

## Verify and report

Run the affected lint and UI tests, and confirm that the intended tests actually ran; zero selected tests do not count as verification. For changed UI snapshots, compare warning counts, messages, and spans with previous expectations; identify whether differences come from upstream diagnostics or lint behavior before updating expected output. Check formatting and inspect the final diff. Report every lint checked, source changes and supporting commits or discussions, verification results, and remaining errors. A successful upgrade command does not establish that the lints build.
