# Changelog

## 1.0.1

Fixes from the post-release review.

- A `dist` or `build` folder below the project root was left out of the fingerprint and the scans with no warning, so a file in `apps/web/dist` could change after approval and the release still verified. The coverage warning now lists every such folder at any depth, by path. The README says plainly that excludes match folder names at every depth.
- The web desk no longer puts its session token in the page it serves. `releasetrail serve` prints and opens a link ending in `#session=...`; the page reads the token from the URL fragment and clears it from the address bar. Fetching `/` now reveals nothing.
- README corrections: the exact credential-assignment names (a bare `token` is not one), that files under `fixtures/` are not scanned for credentials, the full set of environment-name fragments scrubbed from output, the exit codes of `apply` and `rollback`, that the working-tree check is plan-time only, and the single-operator and process-id-reuse limits.
- Tests: 156 (4 added).

## 1.0.0

First release. A plan, an approval of its exact SHA-256, a deploy that is verified against the live result, a receipt, and a rollback, all driven by a checked-in `.releasetrail.json` contract. Command line, local web desk and a library, standard library only, Python 3.10+.

Built from an earlier prototype and put through a hostile review before release. What the review found, and what 1.0.0 does about it:

- Two callers could both pass the state check and both run the deploy. State changes are now `BEGIN IMMEDIATE` transactions with compare-and-set updates.
- A process that died mid-apply left the release stuck in `APPLYING` with no way out. The next start moves it to a reconcilable state (or to `FAILED` when the deploy never launched).
- A release whose deploy never ran could be "rolled back", rolling production back to the previous version. Rollback now requires that the deploy command was launched.
- A timed-out command left its children running and could hang on their open pipes. The whole process group is killed; stdin is closed.
- The local server checked a token and the `Origin` header but not `Host`, so DNS rebinding could read the session token. Every request must now address a loopback host. A non-ASCII token header no longer causes a server error. `--root` limits which folders may be planned.
- The stored plan could be edited in the database without its hash noticing. The plan is re-hashed before approve, apply and rollback.
- The default state folder was inside the working directory; it changed on every run and broke the fingerprint. The default is now `~/.releasetrail` and a state folder inside the project is refused.
- Edits to `dist` and `build` after approval went unnoticed. The plan now warns, and `scan.include` covers them. Symlink targets and the Git HEAD are now part of the drift check.
- Reconciliation accepted a canned sentence as "evidence" and stored it unredacted. Evidence is now required, redacted and recorded; the web desk asks for it.
- Output redaction missed Stripe, Slack, Google and `sk-` keys, bearer tokens, URL credentials, short passwords and any secret held in an environment variable. All are now scrubbed.
- The credential scan flagged `password = os.environ[...]` style references and skipped `.env.production`. References and placeholders are skipped, and `.env*` files are scanned unless Git ignores them.
- The silent-error scan only knew `catch (e) {}`. It now also finds `catch {}`, comment-only catches, `.catch(() => {})` and Python `except: pass`.
- A malformed `status`, `contains` or `scan.max_backup_bytes` in the contract raised after the deploy had run. All contract fields are validated before anything executes.
- A freshly deployed page was checked once, immediately. Verification checks can retry (`attempts`, `interval_seconds`).
- The body hash on URL checks hashed a JSON encoding of the text instead of the bytes. It is now the SHA-256 of the response bytes.
- Release order was unstable within one second; the ledger now lists newest first by insertion order.
- New: per-check `policy` (block, warn, off), a working-tree check, `releasetrail init`, `verify-receipt`, and a receipt content hash.
