# Master build prompt

Paste into Claude Code from the repo root, or just say: "Read BUILD_PROMPT.md and execute it."

You are building the HelloVibe DevOps Engineer (Cloud) take-home for the candidate (Sai Kaushik). The **target** is a Windows laptop with PowerShell 7 and Docker Desktop, but you will usually be running in a **Claude Code cloud session (a Linux container, probably without Docker)**. Write everything for the Windows target, make it cross-platform-safe, and verify as much as the environment allows (see "Environment modes"). Read these first: `CLAUDE.md`, `CHECKLIST.md`, `docs/DECISIONS.md`. The decisions in `docs/DECISIONS.md` are final. Do not reopen them. The original brief is confidential and is not in the repo; everything you need is below. Never commit it.

## Environment modes

At the start of Phase 0, detect the environment and record it in `docs/WALKTHROUGH.md`:

- Run `docker info`. If it works, you are in **full mode**: do every acceptance check in every phase.
- If it fails, you are in **no-Docker mode** (expected in a cloud session). Still write all the files. For every check that needs a Docker daemon (image build and run, Compose up, Trivy image scan, the rollback drill, the alert drill, anything calling `docker`), do NOT fake it and do NOT tick the matching `CHECKLIST.md` item. Instead add the exact commands to `docs/VERIFY_LOCALLY.md` (a numbered list the candidate runs on the laptop, each with the expected result). Never claim in a commit message, the walkthrough or the final summary that something was verified when it was not.
- Try to install what is missing: `pwsh` 7, Pester 5, PSScriptAnalyzer, Terraform 1.7+, `promtool`, and the `trivy` binary. Network access may be restricted to package registries and GitHub; GitHub release downloads usually work. If a download is blocked, say so in `docs/VERIFY_LOCALLY.md` and move on. Do not keep retrying.
- Make the code testable without Docker: keep logic in plain PowerShell functions (for the service, for example `app/lib/Metrics.ps1`, `app/lib/Config.ps1`) so Pester can test it without a running web server or container. All calls to `docker` go through one private wrapper function per script or module, which tests mock.
- Substitute checks where possible in no-Docker mode: parse every `.ps1`/`.psm1` with `[System.Management.Automation.Language.Parser]::ParseFile`; validate YAML files with a YAML parser; run `terraform fmt/validate/test`; run `promtool check rules/config` and `promtool test rules` (see Phase 6); run `trivy config` as a local binary against Terraform, the Dockerfile and Compose if the binary can be installed.

## How to work

- Work through phases 0 to 8 in order, without stopping between phases. Stop and ask only if you are truly blocked.
- **Small commits** with clear messages (history is scored). Before every commit: run PSScriptAnalyzer and Pester; fix failures.
- **Push** to `origin main` after each phase (the repository is configured for you; bootstrap commits may go straight to `main`). If a push fails, say so once in the final summary and continue; never search for tokens, never change remotes, never create the `governance-baseline` tag (the candidate does that later).
- Tick items in `CHECKLIST.md` as they are verified, not before.
- **Never commit secrets**, PATs, tokens, `.env`, `.secrets/`, Terraform state or the brief PDF. Check `git status` before each commit.
- Everything is free tooling. Nothing is applied to Azure. No Azure subscription exists.
- The candidate must explain every line on a follow-up call. Prefer simple, readable code with short "why" comments. No clever one-liners, no unnecessary abstraction, no extra features beyond this prompt.
- When something is ambiguous, make a call, add a row to `docs/DECISIONS.md`, and move on.
- **Cross-platform rules:** PowerShell 7 only (`pwsh`); code must run on Windows and on Linux pwsh 7; use `Join-Path` and `$HOME`, never hardcode `\` or `/` in paths; no bash-isms; shell scripts are not allowed; LF line endings (see `.gitattributes`). The Compose file and Dockerfile target Linux containers.
- Every `.ps1`, `.psm1` in the repo (scripts, module, tests) passes PSScriptAnalyzer (Error and Warning fail the build; Information only warns) and has Pester 5 tests where it contains logic. Do not suppress rules without a one-line justification comment.
- After each phase, append to `docs/WALKTHROUGH.md` a plain-English section for that phase: what each file does, how the pieces connect, and three likely follow-up questions with short answers. The candidate will read this to prepare.

## Fixed facts (use exactly)

| Thing | Value |
|---|---|
| Workload name | `sk-svc` |
| Image | `sk-svc:<full 40-char commit SHA>`; never only `latest`; images stay local on the agent (no registry) |
| Container port | 8080 |
| Host ports (bind to 127.0.0.1) | dev 7311, prod 7312, Prometheus 7390, Grafana 7391, Alertmanager 7393 |
| Compose project names | `sksvc-dev`, `sksvc-prod`, `sksvc-data` (database), `sksvc-mon` (monitoring); shared Docker network `sksvc-net`, created by the bring-up script |
| Compose profiles | `dev`, `prod`, `data`, `monitoring` in one `docker-compose.yml` |
| Environments | `dev`, `prod` |
| Deploy history | JSON per environment in `$HOME/.releaseguard/history/<env>.json` (override with env var `RELEASEGUARD_HOME`); outside the repo and workspace |
| Branch policy | allowed branch names: `main`, `feature/*`, `release/*` (lowercase letters, digits, `.`, `_`, `-` after the slash). Squash merge only. PR commits on main end with `(#<number>)` |
| Azure names | pattern `<type>-sk-svc-<env>`: `rg`, `cae`, `ca`, `id`, `kv`; region `southindia` |
| Azure DevOps | pool name default `sk-local`; pipelines `sk-svc-pr` and `sk-svc-main`; environments `sk-svc-dev` and `sk-svc-prod` (prod has an approval check, created by the user) |

## Phase 0: Preflight and foundations

- Detect the environment mode (above). Check and report versions: `pwsh` 7+, `git`, `docker` and `docker compose`, `terraform` 1.7+ (needed for `mock_provider`), Pester 5, PSScriptAnalyzer, `promtool`, `trivy`. Install what you can for the current user. Trivy in the pipeline runs as a pinned Docker image (`aquasec/trivy`); locally in no-Docker mode use the binary.
- Create `scripts/Test-Prereqs.ps1` (with tests) that checks the above and prints the exact install command for anything missing. Docker and Trivy are reported as "optional here, required on the agent".
- Create `PSScriptAnalyzerSettings.psd1` (Error and Warning fail). Avoid `Write-Host` in scripts; use `Write-Output` / `Write-Information`.
- Add `.secrets/` to `.gitignore`. Keep `.gitattributes`.
- Commit and push.

## Phase 1: Service, container, Compose (Part 3)

**Service** in `app/` using PowerShell 7 with the Pode module (pin the module version). Put the logic in `app/lib/*.ps1` functions and keep the Pode route file thin.
- `GET /healthz`: JSON `{ "status": "ok", "secret_loaded": true|false }`. Never print the secret.
- `GET /version`: JSON `{ "version": "<GIT_SHA env var>" }`.
- `GET /work`: simulates a request: sleeps a random 10 to 200 ms, then returns 200, **or returns 500 with probability `FAIL_RATE`** (env var, 0 to 1, default 0). Only this endpoint obeys `FAIL_RATE`, so `/healthz` and `/version` stay reliable and Docker's health check never flaps.
- `GET /metrics`: Prometheus text format: `http_requests_total{status="200"}` (counter by status code), `app_info{version="<sha>"} 1`, and `http_request_duration_seconds` histogram (buckets, `_sum`, `_count`). Counters must be thread-safe.
- Reads the secret from the file path in env var `DB_PASSWORD_FILE` at startup (sets `secret_loaded`).

**Dockerfile**: multi-stage, non-root user, small final image. Build arg `GIT_SHA` becomes env var `GIT_SHA` and an image label. Use a current patched PowerShell base image tag (verify it exists; if you cannot reach a registry, pick the documented tag and list it in `docs/VERIFY_LOCALLY.md`).

**Compose** (`docker-compose.yml`): services `app-dev` and `app-prod` (profiles `dev`/`prod`) from the same image, image tag from env var `IMAGE_TAG`, env `FAIL_RATE` (default 0), a health check each, a Docker secret `db_password` from `./.secrets/db_password`, and a Postgres service `db` (profile `data`, image `postgres:16-alpine`, password from the same secret via `POSTGRES_PASSWORD_FILE`, named volume, no published port, health check). One Postgres container with two databases: `svc_dev` and `svc_prod`.

**Scripts** (each with tests): `scripts/Start-Stack.ps1` creates `sksvc-net`, generates `.secrets/db_password` (random, 24+ chars) if missing, builds the image tagged with the current commit SHA, and brings up data, dev and prod. `scripts/Stop-Stack.ps1` tears everything down (all four projects, network; `-Purge` also deletes volumes and the history folder). README must say: one command up, one command down.

**Acceptance** (full mode): `curl http://127.0.0.1:7311/version` returns the SHA; `/healthz` healthy; `/metrics` shows the three metrics; with `FAIL_RATE=1`, `/work` returns 500. Image runs as non-root (verify with `docker run ... id`). **No-Docker mode:** Pester tests for the library functions (metrics text format, fail-rate behavior with a seeded random source, config/secret reading) pass; the rest goes to `docs/VERIFY_LOCALLY.md`.

Write `docs/WALKTHROUGH.md` section 1. Commit in small steps, push.

## Phase 2: ReleaseGuard module (Part 2)

`modules/ReleaseGuard/` with `ReleaseGuard.psd1`, `ReleaseGuard.psm1`, `Public/`, `Private/`. **Exactly three exported commands**, each with comment-based help (synopsis, description, parameters, examples), error handling, and Pester tests in `tests/` (tests build a throwaway Git repo fixture in a temp folder; mock Docker through one private wrapper function). This phase is fully testable without Docker.

**`Test-BranchCompliance -RepoPath [-BranchName] [-PolicyPath] [-Since]`**
- Policy in a JSON file (`modules/ReleaseGuard/policy.json`): allowed branch name patterns, PR-merge commit pattern `\(#\d+\)$`, and `baseline` (a tag name, default `governance-baseline`).
- Checks the branch names (all branches, or just `-BranchName`) against the policy.
- Walks `main`'s **first-parent** history from the baseline tag (commits at or before the baseline are exempt, because early bootstrap commits were pushed directly). Flags any commit whose subject does not match the PR pattern. If the baseline tag does not exist, report a clear warning and check nothing (do not fail).
- Prints a readable report and returns non-zero exit status for violations (script-friendly; the pipeline step fails on it).

**`New-ReleaseAudit -From -To [-ImageTag] [-ChecksPassed] [-OutputDir]`**
- Lists commits between the two refs: SHA, author, subject, PR number (from `(#n)`), and writes `release-audit-<to>.md` and `.json`, including image tag and checks passed. If `-From` is not given or history is empty, use the repo's first commit.

**`Invoke-Rollback -Environment {dev|prod} [-ComposeFile] [-TimeoutSeconds]`**
- Reads `<history>/<env>.json`. Entries: `{ tag, deployedAtUtc, status: deployed|rolled_back|failed }`. Finds the previous successfully deployed tag, redeploys it with `docker compose -p sksvc-<env> up -d` and `IMAGE_TAG=<tag>`, polls `/version` until it returns that tag or times out, marks the bad release `rolled_back`, and records the rollback.
- **Idempotent**: if the environment is already on the target tag (for example, a second run), do nothing and report "already rolled back" (success).
- Clear terminating errors with distinct error IDs for: no previous release, Docker not running, health check timed out.
- History read/write helpers live in `modules/ReleaseGuard/Private/History.ps1` as plain functions, so `scripts/Invoke-Deploy.ps1` can dot-source that one file. Keep the module's public surface at three commands.

**`scripts/Invoke-Deploy.ps1 -Environment -ImageTag [-FailRate]`**: runs migrations first (Phase 3), then `docker compose -p sksvc-<env> up -d`, waits for healthy, and appends a `deployed` entry to the history (only if the tag differs from the latest entry). **`scripts/Test-Smoke.ps1 -BaseUrl -ExpectedSha [-Requests 20]`**: `/healthz` ok, `/version` equals the SHA, and all N `/work` requests return 200; exits non-zero otherwise. Test the smoke script against a tiny in-process fake HTTP listener, or with mocked `Invoke-RestMethod`.

Acceptance: all tests green, including rollback behavior against mocked Docker (previous tag chosen, bad release marked, second run is a no-op, each error ID raised). Full mode only: run a real local rollback drill by hand (deploy tag A, deploy tag B, `Invoke-Rollback -Environment prod`, verify `/version` shows A, run it again, confirm no-op); otherwise add it to `docs/VERIFY_LOCALLY.md`. Write walkthrough section 2. Commit, push.

## Phase 3: Migrations (welcome extra)

- `migrations/001_*.sql`, `002_*.sql` (a small example table, then an added column). Each file is safe to apply once.
- `scripts/Invoke-Migrations.ps1 -Environment [-MigrationsPath]` (with tests; mock Docker): reads the password from `.secrets/db_password`, creates the database if missing, creates table `schema_migrations(filename, applied_at)` if missing, applies the numbered files in order, each in a transaction (`psql -v ON_ERROR_STOP=1` via `docker exec`), skips ones already recorded. Running twice applies nothing the second time (prove it in the mocked test).
- Document in DESIGN.md: rollback does not undo migrations, so migrations must be backward compatible.
- Real-Postgres run goes to `docs/VERIFY_LOCALLY.md` in no-Docker mode.
- Walkthrough section 3. Commit, push.

## Phase 4: Azure Pipelines (Part 1)

Files in `pipelines/`: `pr.yml`, `main.yml`, and templates in `pipelines/templates/`. Pool: self-hosted (`name: sk-local`, demand Windows). Configuration via **template parameters** (no variable groups). Use `pwsh:` steps. Checkout with `fetchDepth: 0` and `fetchTags: true`. The code is on GitHub: triggers are YAML `pr:` and `trigger:` for `main`.

**`pr.yml`** (on every PR to main), all of: PSScriptAnalyzer; Pester (publish results); `Test-BranchCompliance` (pass the PR source branch name `$(System.PullRequest.SourceBranch)`); `terraform fmt -check`, `init -backend=false`, `validate`, `test`; Trivy config scan (Terraform, Dockerfile, Compose); image build; Trivy image vulnerability scan. **Fail** on: lint errors/warnings, test failures, Terraform failures, branch-policy violations, HIGH or CRITICAL vulnerabilities **that have a fix available** (`--ignore-unfixed`), and HIGH/CRITICAL config findings. **Warn only** (print, do not fail): lower severities and unfixed vulnerabilities. Run Trivy as a pinned Docker image; scan the image via `docker save` to a tar and `--input` (reliable on Windows). If findings come from our own code, fix them. If a base image finding cannot be fixed, prefer a newer base tag; use `.trivyignore` only with a justification comment per entry. Document the fail/warn table in `DESIGN.md`.

**`main.yml`** (on merge to main): stages `Build` (build once, tag with full SHA, run the PR checks again cheaply or skip as justified) then `Dev` then `Prod`, using shared templates (`templates/build.yml`, `templates/deploy.yml`, `templates/smoke.yml`) called with different parameters (environment, port, compose project, fail rate). The **same image** is promoted (no rebuild). Dev: deploy to `sksvc-dev`, smoke test `/version` equals the SHA. Prod: deployment job targeting Environment `sk-svc-prod` (the user adds the approval check in the UI); run `New-ReleaseAudit` (From = last deployed prod SHA from the history, To = this SHA) and publish the Markdown and JSON as a pipeline artifact; deploy; smoke test; on smoke-test failure run `Invoke-Rollback -Environment prod` (condition `failed()`) and let the run end red (it failed, and it self-healed). Add a queue-time parameter `prodFailRate` (default 0) so the user can create a deliberately bad **prod-only** release for the evidence run (dev passes, prod's smoke test fails, rollback runs).

Acceptance: parse every YAML file with a YAML parser, check that every template path and parameter name referenced exists, and read it through once for syntax. Pipelines can only truly run on the candidate's agent, so list "first real run" in `docs/VERIFY_LOCALLY.md`. Walkthrough section 4. Commit, push.

## Phase 5: Terraform (Part 3)

`terraform/` (root config calling `terraform/modules/service/`), `terraform/envs/dev.tfvars` and `prod.tfvars`. Pin Terraform >= 1.7 and azurerm 4.x. Never applied.

**Module** creates: resource group (or takes one), Container Apps environment, Container App (image from a variable with a placeholder default, ingress target port 8080, health probes on `/healthz`, env var `FAIL_RATE`, replica min/max from variables), user-assigned managed identity attached to the app, Key Vault with RBAC authorization and `purge_protection_enabled` from a variable, a role assignment giving the identity "Key Vault Secrets User" on the vault, a Key Vault secret `db-password` whose value comes from a **sensitive** variable `db_password` (no default; never in tfvars), and the Container App secret referencing that Key Vault secret through the identity, mounted as a secret volume so the app reads `/mnt/secrets/db-password` (matches `DB_PASSWORD_FILE`). Names follow `<type>-sk-svc-<env>`; region `southindia`.
- `dev.tfvars`: 1 replica, purge protection off. `prod.tfvars`: 2 replicas, purge protection on.

**Tests** (`.tftest.hcl`, `mock_provider "azurerm"`, `command = plan`): at least five assertions: prod replicas = 2, prod purge protection on, dev replicas = 1, dev purge protection off, app uses the user-assigned identity, secret `db-password` exists and is referenced from Key Vault via the identity, Key Vault uses RBAC. A dummy `db_password` is supplied by the test only. Make the tests actually exercise the real tfvars values (for example run per environment with `-var-file` and `-filter`) and document the exact commands.

Add a short "Before a real apply" note (in `terraform/README.md`): remote encrypted state (state holds the secret value), passwordless pipeline login, a real registry, networking and private endpoints, logging with retention, tags, budgets, check Container Apps availability in `southindia`, globally unique Key Vault name suffix.

Acceptance (does not need Docker): `terraform fmt -check`, `init -backend=false`, `validate`, and `test` all pass. The azurerm provider download needs registry access; if it is blocked in this environment, say so in `docs/VERIFY_LOCALLY.md` and do not tick the item. Walkthrough section 5. Commit, push.

## Phase 6: Monitoring (Part 4)

`monitoring/prometheus.yml`, `monitoring/alerts.yml`, `monitoring/alertmanager.yml`, `monitoring/grafana/` (datasource and dashboard provisioning). Added to Compose under profile `monitoring` (project `sksvc-mon`, attached to `sksvc-net`; ports bound to 127.0.0.1). Pin image versions. Prometheus scrapes both apps (jobs `sk-svc-dev`, `sk-svc-prod`) every 15 s.

**Alert** `ProdHighErrorRate`, severity `page`, `for: 2m`:
```
(
  sum(rate(http_requests_total{job="sk-svc-prod",status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total{job="sk-svc-prod"}[5m]))
) > 0.05
and
sum(rate(http_requests_total{job="sk-svc-prod"}[5m])) > 0.2
```
Annotations: summary, description, and a link to `RUNBOOK.md`. Alertmanager: simple route and a default receiver with no external integration (the alert shows in its UI); include a commented Slack/email example and note that real credentials would come from a secret.

**Rule unit test (works without Docker):** write `monitoring/alerts.test.yml` for `promtool test rules` with synthetic series: a prod error ratio of about one third with healthy traffic makes `ProdHighErrorRate` fire after the 2-minute wait; a high ratio at near-zero traffic does not fire; a ratio under 5% does not fire. Run `promtool check config`, `promtool check rules` and `promtool test rules`. Get `promtool` from the Prometheus GitHub release tarball if it is not installed. Add the commands to the README and a `scripts/Test-Monitoring.ps1` wrapper (with tests), and add a non-blocking PR pipeline step for it if `promtool` is available on the agent.

**Grafana**: anonymous read-only (Viewer) access, no committed password, bound to localhost. Dashboard panels: requests per second, 5xx ratio, p95 latency (from the histogram), running version (from `app_info`). Validate the dashboard JSON parses.

**`scripts/Send-Traffic.ps1 -BaseUrl -DurationSeconds -RatePerSecond`** (with tests) to drive load. Document the demo in the README: restart prod with `FAIL_RATE=0.33`, run `Send-Traffic.ps1`, watch Prometheus `/alerts` go pending then firing (allow a few minutes), then see it in Alertmanager. The user takes the screenshot.

Acceptance: `promtool` checks and the rule unit test pass. Full mode only: targets are UP, the alert fires in a live local drill, the dashboard loads with data; otherwise add these to `docs/VERIFY_LOCALLY.md`. Walkthrough section 6. Commit, push.

## Phase 7: Documentation

- `README.md`: prerequisites, one-command bring-up and one-command teardown, how to register the agent (see `docs/MANUAL_STEPS.md`), how to run lint/tests/terraform tests/promtool tests, the alert demo, which registry is used (local on the agent) and why.
- `DESIGN.md` (one page, about 450 words): Compose-to-Terraform/Container Apps mapping (service to app, port to ingress, health check to probes, Docker secret to secret volume, env vars to app settings, project to environment); secrets and identity flow (Key Vault, managed identity, pipeline, app) and the Terraform-state caveat; PR gate fail/warn table with reasons; what changes with fifteen pipelines sharing agents (parallel job limits, one shared Docker daemon so project names/ports/cleanup matter, pinned tool versions, ephemeral agents, queue priorities, isolating environments); keeping Azure spend visible (tags, budgets and alerts, scale-down of dev, log retention caps, cost reviews). Also: migrations must be backward compatible; what changes with a real registry.
- `RUNBOOK.md`: scenario: prod deploy at 01:50, alert at 02:00, a third of requests return 5xx. First 15 minutes minute by minute (acknowledge, confirm impact on the dashboard, check what changed, decide, act, verify); rollback vs fix-forward criteria (rollback by default when the problem started with the deploy and no irreversible migration; fix-forward only if the cause is known and the fix is tested and faster, or rollback is unsafe); the exact commands (`Invoke-Rollback -Environment prod`, verify `/version`, watch the 5xx ratio fall); who to tell and when (engineering lead, service owner, support lead; updates at 5 and 15 minutes); post-mortem outline (blameless: summary, impact, timeline, root cause, detection, response, what went well/badly, action items with owners).
- `docs/TIME.md` and `docs/EVIDENCE.md`: keep the templates; leave blanks for the user.
- Commit, push.

## Phase 8: Final verification and handoff

Run everything the environment allows from a clean state: all Pester tests; PSScriptAnalyzer over the repo; the Terraform commands; the `promtool` checks; YAML and JSON parse checks. In full mode also run `Stop-Stack.ps1 -Purge`, then `Start-Stack.ps1`, the rollback drill and the alert drill. Scan for secrets: `git ls-files` must not include `.secrets/`, state files, tokens or the brief PDF; search history for token-like strings. Tick the final section of `CHECKLIST.md` only for what you verified. Make sure `docs/VERIFY_LOCALLY.md` is complete (every unverified item, with exact commands and expected results) and linked from the README. Commit and push. Then print a short summary: what was built, what you verified and how, what is NOT verified and where it is listed, whether pushes succeeded, and the exact next manual steps from `docs/MANUAL_STEPS.md`.

## Out of scope

Creating the Azure DevOps organization, PAT, agent, pipelines, Environment approval, GitHub branch protection, the governance-baseline tag, running the evidence pipeline runs, the alert screenshot, and the email. These are in `docs/MANUAL_STEPS.md` for the user.
