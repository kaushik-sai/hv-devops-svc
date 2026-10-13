# CLAUDE.md

Project: HelloVibe DevOps Engineer (Cloud) take-home assignment. Candidate: Sai Kaushik.
Source brief: the assignment PDF (October 2026). Full task list lives in `CHECKLIST.md`; keep it updated as items complete.

## Goal
Ship a tiny containerised service through an Azure Pipelines workflow with PowerShell release governance, automatic rollback, Terraform (tested, never applied), monitoring, and ops docs. 6-8 hours of effort total.

## Where things are decided
- `docs/DECISIONS.md` holds the final design decisions (language, ports, names, gates, secrets, alerting). Treat them as fixed.
- `BUILD_PROMPT.md` is the phase-by-phase build plan. `docs/MANUAL_STEPS.md` lists what only the user can do (accounts, agent, UI settings, evidence runs).
- The original brief is confidential: never commit it.

## Hard rules (from the brief)
- PowerShell 7 only. Every script must pass PSScriptAnalyzer and have Pester 5 tests.
- **Never commit secrets, tokens, PATs, `.env` files, or Terraform state.** Check before every commit.
- One public GitHub repo for all code. Commit as we go, in small meaningful commits (history is scored).
- Pipelines run on a **self-hosted agent** in a free Azure DevOps org. If ADO blocks us, GitHub Actions is acceptable, but record why in the docs.
- Everything runs on free tools. No Azure subscription: Terraform is never applied and is tested with `mock_provider "azurerm"`.
- The candidate must be able to explain every line on the follow-up call. Keep code simple and readable; prefer clarity over cleverness, and add short comments explaining the "why".
- If something is ambiguous, make a call, write it in `docs/DECISIONS.md`, and move on.

## Layout
```
app/                      tiny HTTP service (/healthz, /version, /metrics)
pipelines/                azure-pipelines.yml (+ templates/)
modules/ReleaseGuard/     PowerShell module (Public/, Private/)
tests/                    Pester tests
terraform/modules/service Container Apps + identity + Key Vault (+ tests/)
terraform/envs/           dev.tfvars, prod.tfvars
monitoring/               prometheus.yml, alert rules, alertmanager.yml
migrations/               optional numbered SQL files
scripts/                  agent registration, bring-up, tear-down
docs/                     DECISIONS.md, TIME.md, EVIDENCE.md, evidence/ (screenshots)
docker-compose.yml        dev + prod + prometheus + alertmanager
README.md DESIGN.md RUNBOOK.md   required deliverables
```

## Build order (dependencies)
1. Service + Dockerfile + Compose (Part 3)
2. ReleaseGuard module + tests (Part 2), because the pipeline calls it
3. Pipeline + templates + branch policy (Part 1)
4. Terraform + `terraform test` (Part 3)
5. Prometheus/Alertmanager, RUNBOOK, DESIGN (Part 4)
6. Evidence runs: blocked PR, green prod deploy, bad release rolled back

## Scoring weights
Pipeline 30 | PowerShell 25 | Container + Terraform 20 | Operations 15 | Writing 10

## Commands (fill in as they are built)
- Lint: `pwsh -c "Invoke-ScriptAnalyzer -Path . -Recurse -Settings ./PSScriptAnalyzerSettings.psd1"`
- Test: `pwsh -c "Invoke-Pester -Path ./tests -CI"`
- Terraform: `cd terraform/modules/service && terraform fmt -check && terraform init -backend=false && terraform validate && terraform test`
- Up: `docker compose up -d --build`
- Down: `docker compose down -v`

## Working style
- Work one checklist section at a time; tick items in `CHECKLIST.md` when done and verified.
- Log time per part in `docs/TIME.md`.
- Run lint + tests before each commit.
- Do not add anything beyond the brief unless asked; note skipped ideas in `docs/TIME.md` under "What I would do next".
