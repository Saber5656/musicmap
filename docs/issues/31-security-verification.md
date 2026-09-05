# Title

Security verification suite and SECURITY.md

## Summary

Turn DESIGN.md §13.2's threat table into executable, permanently-green tests: the full
auth/Host matrices, DNS-rebinding simulation, the adversarial-archive corpus, CSP/header
assertions, SSRF-allowlist attempts, PII-at-rest audit, and log-redaction checks — plus
SECURITY.md and a threat-table→test traceability map.

## Context

Individual issues shipped their own controls; this issue proves them **as a system** and
pins them against regression. Every T-row in §13.2 must map to at least one test that fails
if the control is removed.

## Scope

- `src/security-tests/` (vitest, runs in normal `npm test`), e2e additions where a real
  browser matters; `SECURITY.md`; `docs/security-traceability.md`; `README.md` (one
  Security-policy link line only).

## Detailed Requirements

1. Traceability doc: table T1-T14 → test file/name(s) → issue that implemented the control.
   CI-checked completeness is manual review; the doc is normative for release readiness.
2. Test contents (beyond regression-duplicating 06/08/25 unit tests — these are
   integration-level against the REAL built app factory):
   - **T1/T11**: full matrix — no header / empty Bearer / wrong token / token as query param
     (must fail — only the fragment→header flow is valid) / correct token, across GET/POST/
     DELETE representatives; token absent from all response bodies and log capture.
   - **T2**: Host values `evil.com`, `localhost.evil.com`, `127.0.0.1.evil.com`,
     `127.0.0.1:<wrong-port>`, `[::1]:<port>`, IP-literal tricks (`0x7f000001`), missing
     Host — all 403 on `/`, `/api/health`, `/assets/*`. A DNS-rebinding simulation: correct
     token + hostile Host → still 403 (token alone is insufficient).
   - **T3**: assert the listening socket address is 127.0.0.1 (real listen test);
     repo-grep/AST test asserting: the only bind-host value anywhere in `src/` is the
     literal `127.0.0.1` (issue 06's `listenLoopback`), no CLI/env/config option named
     `host` exists, and the strings `0.0.0.0`/`'::'` never appear as listen values.
   - **T4/T5**: run the issue-08 adversarial corpus through the FULL import API (not just
     the iterator): bomb/encrypted/lying-header/nested/corrupt → correct 4xx/failed-import
     codes, server stays alive and responsive after each (health check between cases);
     plus a 100-entry zip of tiny valid JSONs (cap sanity).
   - **T6** (expected outcomes per source spec): Spotify >64 KiB string → that row
     `skip:invalid`; Spotify depth >20 → import fails `ERR_SCHEMA_MISMATCH`; Apple >64 KiB
     cell → row `skip:invalid`; Apple >10 MiB record → import fails `ERR_SCHEMA_MISMATCH`
     (`csv_record_too_large`); YTM mirrors Spotify. Memory stays bounded (no crash) in all
     cases.
   - **T7**: seed DB with hostile strings (`<img onerror>`, `<script>`, `javascript:` URLs,
     SVG payloads) as artist/track/chapter titles → e2e: rendered pages contain them as
     text, no dialog/script execution (CSP violation listener + DOM assertions); CSP header
     exact-match test on every route class.
   - **T8**: repo-grep test: no template-literal SQL (`db.prepare(\``) with interpolation
     pattern) — enforce via eslint rule test + a runtime probe inserting `'; DROP TABLE
     artists;--` as artist name through the full import path, then verifying schema intact.
   - **T9/T13**: with enrichment enabled and a hostile mock (redirect to
     `https://internal.local`, oversized image, wrong content-type, malformed JSON) →
     allowlist errors / item failures, zero requests dispatched off-allowlist (MockAgent
     accounting), runner survives.
   - **T10**: after importing fixtures containing planted PII markers (issue 29's
     `plantedPiiMarkers` literals), scan EVERY text column of EVERY table for them → zero
     hits. Permissions: on POSIX assert data dir `0700` and DB `0600`; on Windows the
     check is skipped with a logged note (DESIGN's "best-effort" caveat) — the suite never
     fails solely on unsupported chmod semantics.
   - **T12**: verified via issue 02's artifacts + a traceability review row: committed
     lockfile present, CI installs with `npm ci`, audit+OSV jobs exist and are hard gates,
     every workflow `uses:` is SHA-pinned, workflow-level `permissions: contents: read`,
     Dependabot config present. (Automated where cheap — a repo test greps workflow files
     for mutable `@v` tags and missing `permissions` keys.)
   - **T14**: log files after a full journey contain no `Bearer`, no token substring, no
     planted-PII markers.
3. `SECURITY.md`: supported-versions table (v1), private reporting via GitHub Security
   Advisories, response targets (ack 7 days), scope (local app — no bounty), disclosure
   policy (90 days), and the §13 model summary linking DESIGN.md.
4. Fix policy: any failing MANDATORY §13.2 control is fixed in this issue before merge —
   no deferral. Only non-mandatory hardening ideas or test-scope expansions may become
   linked follow-up issues. The suite is green at merge either way.

## Acceptance Criteria

- [ ] Every T1-T14 row has ≥1 named test; traceability doc complete and cross-linked.
- [ ] Control-removal spot check: temporarily disabling the Host hook and the token hook
      (locally) each makes ≥1 test fail (documented in PR, changes reverted).
- [ ] Hostile-strings e2e shows zero CSP violations and zero script execution.
- [ ] PII column scan and log scans green on the full e2e journey's data dir.
- [ ] Suite runs in `npm test` + e2e without network egress; CI green.
- [ ] SECURITY.md present and linked from README.

## Validation

`npm test`, `npm run e2e`, CI link; traceability doc reviewed against DESIGN §13.2 row by
row.

## Dependencies

02 (T12 CI artifacts), 06, 08, 14, 30 (harness + full app), 29 (PII-planted fixtures).

## Non-goals

External pentest, fuzzing campaigns (v2 candidate), CodeQL setup (repo-settings task noted
in issue 02), fixing K-class unknowns.

## Design References

DESIGN.md §13 (normative threat table), §6.3, §12.3; ADR-003, ADR-004; ISSUE_PLAN §6.4.
