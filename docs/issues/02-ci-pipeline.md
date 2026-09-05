# Title

CI pipeline and supply-chain guards (GitHub Actions + Dependabot)

## Summary

Add the GitHub Actions CI workflow (lint, typecheck, unit tests, build) with hardened defaults
(SHA-pinned actions, minimal permissions, concurrency), a dependency-audit job, and Dependabot
configuration. This implements DESIGN.md §13.2 T12 and the CI portion of §16.

## Context

The repo is public OSS from day one. Supply-chain and workflow hardening are v1 security
requirements, not a later pass. Later issues extend this workflow (E2E in issue 30, pack-smoke
in issue 33) — structure it so jobs can be appended without rewrites.

## Scope

- `.github/workflows/ci.yml`
- `.github/dependabot.yml`
- `README.md` (CI badge line only)

## Detailed Requirements

1. Workflow `ci.yml`:
   - Triggers: `pull_request` (all branches) and `push` to `main`.
   - Top level: `permissions: { contents: read }`; `concurrency: { group:
     ci-${{ github.ref }}, cancel-in-progress: true }`.
   - Every `uses:` reference — including `actions/*` first-party ones — pinned by **full
     commit SHA** with a trailing comment naming the version tag (e.g.
     `actions/checkout@<sha> # v4.x.x`). Resolve current SHAs at implementation time from
     each action's release page.
   - Job `check` (ubuntu-latest): `actions/checkout` → `actions/setup-node` (node 24, cache
     npm) → `npm ci` → `npm run lint` → `npm run typecheck` → `npm test` → `npm run build`.
     Upload `dist/` as an artifact only if a later job needs it (not required now).
   - Job `check-macos` (macos-latest): `npm ci` → `npm test` (unit only; catches
     better-sqlite3/platform issues later).
   - Job `audit` (ubuntu-latest), two hard-gate steps per DESIGN §13.2 T12:
     (a) `npm audit --omit=dev --audit-level=high` (fails on high/critical);
     (b) OSV scan of `package-lock.json` via `google/osv-scanner-action` (SHA-pinned) or the
     pinned osv-scanner binary — fails on any matching advisory. Neither step uses
     `continue-on-error`; advisory-service downtime fails the job (retry the workflow).
   - All jobs must run without any secrets; workflow must not use `pull_request_target`.
2. `dependabot.yml`: weekly `npm` updates (grouped: `minor-and-patch` group), weekly
   `github-actions` updates; open-pull-request limit 5 each.
3. Add a CI status badge to `README.md`.
4. Do not add release/publish workflows here (issue 33 drafts them).

## Acceptance Criteria

- [ ] CI runs on a PR touching any file and all jobs pass on the issue-01 skeleton.
- [ ] `permissions` at workflow level is exactly `contents: read`; no job elevates it.
- [ ] Every `uses:` is SHA-pinned with a version comment; zero mutable tags (`@v4`) remain.
- [ ] `npm audit --omit=dev --audit-level=high` gate demonstrably fails the build when a
      high-severity dep is present (verify locally by temporarily installing a known-vulnerable
      version, then remove; describe in PR).
- [ ] OSV scan runs in CI and demonstrably fails on a vulnerable lockfile entry (same
      prove-then-remove method as the npm-audit check).
- [ ] Dependabot config validates (GitHub UI shows both ecosystems enabled).
- [ ] Concurrency cancels superseded runs on force-push (observable in Actions UI).

## Validation

Open a draft PR after adding the workflow; attach the green CI run link (all jobs incl.
`check-macos` and `audit`). Include in the PR description the owner's manual checklist
(enable CodeQL, secret scanning, push protection in repo settings) — no settings changes in
this issue.

## Dependencies

01-project-scaffold.

## Non-goals

E2E job (30), perf nightly (32), pack-smoke and release workflow (33), CodeQL/secret-scanning
repo settings (these are repository-settings tasks for the owner, not workflow files — note
them in the PR description as a manual checklist for the owner).

## Design References

DESIGN.md §13.2 T12, §16 (CI matrix), §4 (Node 24).
