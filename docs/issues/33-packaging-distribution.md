# Title

npm packaging, bin smoke test, and release workflow draft

## Summary

Make musicmap installable per DESIGN.md §2.1 G12/§13.4: correct `files`/`bin` packaging with
the built SPA included, a CI `pack-smoke` job proving tarball → global-style install →
`musicmap serve` works cold, version/CHANGELOG conventions, and a manually-triggered release
workflow draft that builds and attaches artifacts WITHOUT publishing.

## Context

Repo policy: merge ≠ release; publishing to npm is a human act. This issue makes the package
provably installable and drafts the release mechanics so the human release step is a button
press later.

## Scope

- `package.json` packaging fields; `scripts/pack-smoke.sh` (or `.ts`); CI job; 
  `.github/workflows/release-draft.yml`; `CHANGELOG.md` scaffold; README install section.

## Detailed Requirements

1. Packaging correctness:
   - `files: ["dist", "README.md", "LICENSE", "CHANGELOG.md"]` — verify `dist/web` (SPA) and
     `dist/db/migrations/*.sql` ship. Migrations are `.sql` files NOT emitted by tsc: add a
     build copy step (`scripts/copy-assets.ts` invoked from `npm run build`) that copies
     `src/db/migrations/**` → `dist/db/migrations/` and the server resolves migrations dir
     relative to its own module URL (works from any install location) — adjust issue 04's
     path resolution if it assumed `src/` (single `migrationsDir()` helper).
   - `bin` executable bit via shebang (already in 01); `prepack` runs `npm run build`.
   - `postinstall` scripts: NONE of our own; document that `better-sqlite3` builds natively
     when prebuilds are unavailable (README toolchain note — K4).
2. `pack-smoke` (script + CI job): `npm pack` → temp dir → `npm install -g` the tarball with
   a scoped prefix (`npm_config_prefix=$TMP/prefix`) → run `musicmap --version`,
   `musicmap status --data-dir $TMP/data`, boot `musicmap serve --no-open --port 14950
   --data-dir $TMP/data` in background → poll `/api/health` (with valid Host header) →
   fetch `/` and assert it serves the SPA HTML AND contains no token substring (T11: the
   token exists only in the stdout URL fragment; parse it from stdout and use it solely as
   `Authorization: Bearer` for one authenticated API call, e.g. `/api/overview`) → import
   the committed issue-10 mini Spotify ZIP fixture via `musicmap import` → `musicmap
   status` shows events > 0 → clean shutdown (SIGINT, exit 0). Runs on ubuntu AND macos in
   CI.
3. Versioning/CHANGELOG: `CHANGELOG.md` (Keep a Changelog format, `## [Unreleased]` section);
   version stays `0.x` until v1 completion; add `npm run release:check` = lint + typecheck +
   test + build + pack-smoke locally.
4. `release-draft.yml`: `workflow_dispatch` with `version` input; SHA-pinned actions,
   top-level `permissions: contents: read` (no job elevates it — the workflow creates NO
   tags and NO GitHub Releases, avoiding implicit tag creation): run full checks +
   pack-smoke → upload the tarball, `perf-report.md` (if present), and a generated
   `release-notes-draft.md` as workflow ARTIFACTS. The human release steps (documented in
   the artifact): verify artifacts → create tag `v<version>` → create the GitHub Release
   from the tag attaching the artifacts → `npm publish --provenance`. The workflow must
   NOT reference any npm token or secret and must not use `pull_request_target`.
5. README install section (brief here; full docs in 34): supported Node (>=24), install
   from source (`git clone && npm ci && npm run build && npm start`) and from tarball; where
   data lives (§17 table); uninstall note (delete data dir).

## Acceptance Criteria

- [ ] `npm pack` tarball contains dist/web assets + migrations; besides npm-mandatory
      metadata (`package.json`), it contains NO `src/`, fixtures, tests, or docs beyond
      README/LICENSE/CHANGELOG (tarball listing asserted in pack-smoke).
- [ ] pack-smoke green on ubuntu + macos CI: cold install → serve → `/` token-free →
      authenticated API call → import → status counts → clean exit, all asserted.
- [ ] Fresh-clone flow verbatim from README works (CI already proves most; run locally and
      paste).
- [ ] `release-draft.yml` dry run (workflow_dispatch on a branch) uploads the three
      artifacts; creates no tag, no release; references no secrets; permissions stay
      `contents: read` (workflow file asserted by issue 31's T12 grep too).
- [ ] Migrations resolve correctly from the installed location (pack-smoke exercises a
      migration on first serve).
- [ ] `engines` enforced: `.npmrc` with `engine-strict=true` committed; `npm ci` under
      Node 22 fails (verified locally once, log in PR); README states the Node >= 24
      requirement.

## Validation

CI links (pack-smoke both OSes, release-draft dry run) + local fresh-clone log in PR.

## Dependencies

01, 02, 04 (`migrationsDir()` helper adjusted here), 07 (serve/status), 10 (committed mini
fixture used by pack-smoke), 14 (`musicmap import`), 17 (built SPA is part of the package).

## Non-goals

Actual npm publish (human act), signed binaries/installers (v2/Tauri), Docker images,
auto-update.

## Design References

DESIGN.md §2.1 G12, §3.3 (build), §13.4 (release posture), §17 (paths); ISSUE_PLAN §6.6;
repo policy: merge ≠ release.
