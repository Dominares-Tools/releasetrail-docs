# releasetrail — Approve one exact hash, then deploy and verify

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/releasetrail**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

releasetrail turns a change into a release you can defend: your own gates and scans run first, you approve one exact SHA-256 hash, it deploys, checks the live result, writes a receipt and keeps a rollback one command away. It has a command line and a local web desk.

## Get it

Buy releasetrail at https://dominares.org/tools/releasetrail: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the releasetrail command, the local web desk, an example contract, and the full test suite.

Requirements: Python 3.10 or newer and Git, no other dependencies. Built and tested on macOS; Windows is supported only by a fallback that is not tested.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

You put a `.releasetrail.json` contract in a Git project. It names your test gates, your deploy command, your rollback command and how to check the live result. Then:

1. **Plan** (`releasetrail plan`). It runs your gates, scans the source, fingerprints every file, and stores an immutable plan. The plan's SHA-256 is the thing you approve.
2. **Approve** (`releasetrail approve --plan-hash ...`). You must pass the exact 64-character hash the plan printed. A different hash, a blocked plan, or source that changed since planning is refused.
3. **Apply** (`releasetrail apply`). It re-checks the source and the Git commit, writes a safety archive of the source, runs your deploy command, then runs your verification checks (commands and/or URLs).
4. **Receipt** (`releasetrail receipt`). A JSON or HTML record of the plan, every check, the approved commands and the timeline, with a content hash you can re-check later (`releasetrail verify-receipt`).
5. **Rollback** (`releasetrail rollback`). It runs your rollback command and records the result.
6. **Reconcile** (`releasetrail reconcile`). If a deploy or rollback times out, the release stops in a state that needs a person. It never retries something that may already have happened. You settle it by writing down what you checked.

It is a release desk for one machine, not a hosting platform, a CI system or a coding agent. It runs your commands; it does not know how to deploy anything itself.

### The contract

`releasetrail init PATH` writes a starter. A complete one:

```json
{
  "version": 1,
  "project": "Customer portal",
  "gates": [
    {"name": "tests", "command": ["{python}", "-m", "pytest", "-q"], "timeout_seconds": 180}
  ],
  "forbidden_patterns": ["OLD_CUSTOMER_NAME"],
  "deploy":   {"command": ["./deploy.sh"],   "timeout_seconds": 600},
  "rollback": {"command": ["./rollback.sh"], "timeout_seconds": 300},
  "verify": [
    {"name": "homepage", "url": "https://example.com/", "status": 200,
     "contains": ["Customer portal"], "attempts": 5, "interval_seconds": 3},
    {"name": "smoke", "command": ["./smoke.sh"], "timeout_seconds": 30}
  ],
  "policy": {"clean_tree": "block", "environment": "warn"},
  "scan": {"exclude": ["generated"], "include": ["dist"], "max_backup_bytes": 250000000}
}
```

| Field | Meaning |
|---|---|
| `version` | Must be `1`. |
| `project` | Name shown in the ledger and receipt. |
| `gates` | Commands that must exit 0 at plan time. Optional, but a plan with none carries a warning. |
| `deploy`, `rollback` | Required. `command` is a JSON array, never a shell string. `timeout_seconds` is 1 to 3600. |
| `verify` | Required, at least one. A `command` check passes on exit 0. A `url` check passes on the expected `status` (default 200) and when the body contains every `contains` string. `attempts` (1 to 20) and `interval_seconds` (0 to 60) retry a check that needs a moment to come up. |
| `forbidden_patterns` | Case-insensitive regexes that must not appear in any scanned text file. |
| `policy` | Per check, `block`, `warn` or `off`. Keys and defaults: `credential_scan` block, `brand_boundary` block, `silent_errors` block, `environment` warn, `clean_tree` warn. |
| `scan.exclude` | Extra folder names to leave out of scans and the fingerprint. A name matches a folder at any depth. |
| `scan.include` | Folder names that are normally left out (`dist`, `build`, `coverage`, `node_modules`, `.venv`, ...) to cover after all, at every depth. Version-control folders can never be included. |
| `scan.max_backup_bytes` | Largest project the safety archive may hold (default 250 MB); above it the plan is blocked. |

In a command, `{python}`, `python`, `python3` and `py` all mean the interpreter running releasetrail. Deploy, rollback and verify commands receive `RELEASETRAIL_RELEASE_ID` and `RELEASETRAIL_PLAN_HASH` in their environment.

### What the plan checks

| Check | Blocks when | Default |
|---|---|---|
| Git repository | not a Git repository, or no commits yet | blocker |
| Working tree | uncommitted changes (the release is not a commit) | warning |
| Your gates | a gate exits non-zero or times out | blocker |
| Credential scan | an AWS, GitHub, Stripe, Slack, Google or `sk-` key, a private-key header, or `api_key`, `secret`, `password`, `passwd`, `auth_token` or `access_token` assigned a literal value (a bare `token = ...` is not a trigger; a token with a known shape is caught by its shape). Placeholders (`changeme`, `${VAR}`, `os.environ[...]`, `xxxxxxxx`) and `.example`/`.sample` files are skipped, and so is every file under a folder named `fixtures`. Git-ignored `.env*` files are not scanned. Values are never written to the plan. | blocker |
| Brand boundary | any `forbidden_patterns` regex matches | blocker |
| Silent error scan | an empty or comment-only JavaScript `catch`, `.catch(() => {})`, or Python `except:` / `except Exception:` whose body is only `pass` | blocker |
| Environment contract | variable names in `.env` and `.env.example` differ | warning |
| Fingerprint coverage | a folder named `dist` or `build` exists anywhere in the project and the fingerprint and scans leave it out (the warning lists each path) | warning |
| Rollback artifact size | the source is larger than `scan.max_backup_bytes` | blocker |
| Preflight | a `url` verify check already fails before deploy | warning |

Gates run first, so the scans and the fingerprint also cover whatever the gates produced.

### Commands

```text
releasetrail demo                          harmless end-to-end run in a temporary folder
releasetrail init PATH                     write a starter .releasetrail.json
releasetrail plan PATH [--json]            exit 0 ready, 1 blocked
releasetrail approve REL --plan-hash H
releasetrail apply REL --plan-hash H [--idempotency-key K]
releasetrail rollback REL --plan-hash H [--idempotency-key K]
releasetrail reconcile REL --plan-hash H --outcome deployed|rolled_back|failed --evidence TEXT
releasetrail list | show REL [--json]
releasetrail receipt REL [--format json|html] [-o FILE]
releasetrail verify-receipt FILE
releasetrail serve [--port 8790] [--root DIR]... [--no-browser]
```

`--state DIR` (or `RELEASETRAIL_STATE`) sets the folder for the ledger, the safety archives and the receipts. The default is `~/.releasetrail`. It must be outside the project, or `plan` refuses. `apply` and `rollback` are idempotent per key: repeat the same key and you get the earlier result, not a second run. The CLI's default `apply` key is derived from the release id, so asking twice never deploys twice. `apply` and `rollback` exit 0 when the release ends `VERIFIED` or `ROLLED_BACK`, 1 for any other end state, and 2 when the command is refused (wrong hash, drifted source, unknown release).

#### The web desk

`releasetrail serve` starts a local page with the same flow: paste a project folder, review the checks, approve, deploy, download the receipt. It binds only to a loopback address, rejects any request whose `Host` is not a loopback name (so a web page that points its own hostname at your machine cannot talk to it), rejects cross-origin requests, and needs a random per-run session token for everything except the page and `/api/v1/health`. The token is never in the served page: `serve` prints a link ending in `#session=...` and opens it, the page reads the token from the URL fragment (which a browser never sends to the server) and removes it from the address bar. Anyone who has that link can drive the desk, so do not share it. `--root DIR` (repeatable) limits which folders it may plan; set it, because planning runs the project's gate commands as you. The HTTP API is `GET /api/v1/releases[/ID[/receipt.json|receipt.html]]` and `POST /api/v1/releases/plan`, `.../ID/approve`, `.../ID/apply`, `.../ID/rollback`, `.../ID/reconcile`; apply and rollback need an `Idempotency-Key` header.

### Safety properties

Each of these has a test:

- Approval needs the exact plan hash; the stored plan is re-hashed before it is approved, applied or rolled back, so an edited database is refused.
- Source files, symlink targets and the Git HEAD are re-checked at approval and again at apply. Drift fails closed before the deploy command runs.
- Two callers racing to apply the same release run the deploy exactly once (every state change is a `BEGIN IMMEDIATE` transaction).
- A rollback is refused for a release whose deploy command never launched, so a failed pre-flight cannot roll production back.
- A deploy or rollback that times out is never retried. The whole process group is killed, so a grandchild holding the output pipes cannot hang the run.
- If the process dies mid-apply, the next start moves the release to a state a person can reconcile instead of leaving it stuck.
- Commands run from an argument array with no shell and no stdin. Credential-looking values (by shape, by `name=value`, and the values of environment variables whose name contains `key`, `token`, `secret`, `passw`, `credential` or `auth`; this over-redacts on purpose, so something like `SSH_AUTH_SOCK` is hidden too) are scrubbed from every captured output and from reconciliation evidence.
- The safety archive leaves out `.env*` files, private keys and certificates.
- The receipt replaces the project path with `<project>` and carries a content hash (`evidence_sha256`).

### Limits

- It executes the commands in the contract you point it at, including your gates at plan time. Read `.releasetrail.json` before you plan a project you did not write; it does not sandbox anything.
- The receipt hash detects accidental or casual edits. It is not a signature: someone who can rewrite the receipt can recompute the hash.
- Approval does not expire.
- The fingerprint ignores `dist`, `build`, `coverage`, `node_modules`, `.venv`, `venv`, caches and VCS folders, matched by folder name at any depth, unless you list them under `scan.include`. The scans skip them too. The plan warns, with the path, when a `dist` or `build` folder exists; nothing warns about the others. An edit inside an ignored folder after approval goes unnoticed.
- The working-tree check (`policy.clean_tree`) runs at plan time only. At approval and apply, drift is judged by the fingerprint and the Git HEAD.
- The credential scan does not read files under any folder named `fixtures`.
- The web desk is for one person on one machine. Any other process or user on the machine that gets the session link can drive it, and `--root` is off unless you pass it.
- Crash recovery decides a run is dead by checking its process id. If the operating system reuses that id for another process, a dead run stays marked as running until you reconcile it. File permission bits are not fingerprinted. Git itself is only asked for HEAD and status.
- Scans read text files up to 2 MiB with common source extensions (plus `Dockerfile`, `Procfile`, `.env*`). The scans are pattern matches: they find the shapes listed above, not every possible secret or every swallowed error.
- A URL check reads at most the first 2 MiB of the page and follows redirects.
- The safety archive is a copy of the source for you to restore from. releasetrail does not restore it for you; "rollback" means your rollback command.
- A deploy command that leaves a background process holding its output open past the timeout is reported as timed out.
- Built and tested on macOS with Python 3.10 and 3.14. The process-group kill and the crash recovery use POSIX calls; on Windows the kill falls back to `taskkill` and crash recovery is switched off (a release interrupted there stays `APPLYING` until you reconcile it by hand). That Windows path is not tested.
- One machine, one operator, standard library only. There are no accounts, no remote state and no notifications.

### Install and test

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

```bash
pytest
```

Python 3.10 or newer; the tool itself has no dependencies.

---

Docs mirror of the private repository Dominares-Tools/releasetrail; updated with each release.
