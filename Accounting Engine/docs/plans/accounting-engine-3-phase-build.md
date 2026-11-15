# Accounting Engine — 3-Phase Build Plan

Repo: `/Users/nageshbhosle/sushama-project` → `https://github.com/nagesh-bhosle/accounting-engine` (branch `main`).

Project lives under **`Accounting Engine/`** (repo root stays `sushama-project/`, as you chose). Current contents: the two plans in `docs/plans/` and the interview guide `.docx` (tracked in git).

**This plan owns build order, script numbering and locked decisions.** `accounting-engine-azure-poc.md` is the detailed design spec and was re-checked against the interview guide on 2026-11-15 (its §0a lists what was added). Where the two disagree, this plan wins.

**Phasing you defined:**
1. **Infrastructure** — get everything up, validate it, pause/start scripts, sanity tests.
2. **Real project** — pipeline code, infra vars embedded, pytest, docs, using the data-engineering skills.
3. **On-demand data + analysis + UI** — test-data generator, analysis scripts and UI, local start/stop, README, and a UI that invokes every script.

---

## Locked decisions

| Decision | Value |
|---|---|
| Cloud | Real Azure subscription, **East US 2**, you are **Owner** |
| IaC | Terraform, own folder, fully automatic |
| Auth | `az login` interactively + service principal minted by script 01 for CI |
| SFTP | Native Azure Blob SFTP (2 local users, one per container) |
| Local-first | docker-compose for daily iteration at $0; Azure on demand |
| Lifecycle | create / pause / start / destroy scripts; `destroy` = your default "done" |
| Phase 1 depth | **Full stack, validate, then pause** |
| Phase 3 UI | **Streamlit** |
| Tool docs | A prerequisites doc so you can rebuild on any machine |
| Installation | **Ask before installing anything** on your machine |

---

## Verified prerequisites (the "new device" checklist)

Checked on this machine just now: `python3 3.14.6` ✅, `docker 29.6.2` ✅, `java 25.0.1` ✅, `git 2.39.5` ✅, **`terraform` ❌, `az` ❌, `databricks` ❌**, Homebrew `7.0.11` ✅.

Exact commands, all verified against vendor docs:

```bash
# Terraform (tap is required; direct formula is deprecated)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Azure CLI — this is a CASK, not a formula
brew install --cask azure-cli

# Databricks CLI — Homebrew 6+ requires an explicit trust step first
brew tap databricks/tap
brew trust databricks/tap          # required on Homebrew >= 6.0.0 (you are on 7.0.11)
brew install databricks            # must report >= 0.205.0 via `databricks version`

# Helpers
brew install jq coreutils
```

Then `az login` once. `docs/PREREQUISITES.md` will carry this plus per-OS notes, versions, and a verification script.

**I will not install any of these without asking you first.**

---

# PHASE 1 — Infrastructure

Deliverables, in build order.

## 1.1 Tooling + prerequisites

- `docs/PREREQUISITES.md` — every tool, version floor, install command per OS, and the check command.
- `scripts/00_check_prereqs.sh` — verifies `az`, `terraform`, `databricks`, `python3`, `java`, `docker`, `jq`; prints a green/red table with the exact fix for each failure. Read-only, exits non-zero if anything is missing.
- This is the first thing that runs in every other script, so a fresh machine produces one clear list instead of a mid-apply failure.

## 1.2 Terraform (`Accounting Engine/infra/terraform/`)

```
infra/terraform/
├── bootstrap/                  # state RG + container, local state, applied once
│   ├── main.tf  providers.tf  outputs.tf  README.md
├── modules/
│   ├── storage/                # GPv2 + HNS + SFTP + 2 local users
│   ├── keyvault/               # secrets + ADF KV linked service backing
│   ├── databricks/             # Premium workspace, access connector, cluster policy
│   ├── unity_catalog/          # metastore (account-level), catalog, schemas
│   ├── adf/                    # factory, linked services, datasets, pipelines, trigger
│   ├── monitoring/             # Log Analytics, diagnostic settings, action group
│   └── budget/                 # budget + 50/80/100% alerts
└── envs/dev/
    ├── backend.tf  main.tf  variables.tf  outputs.tf  terraform.tfvars
```

Key details, each of which was wrong in an earlier draft and is now verified:

**storage module**
- GPv2 + `is_hns_enabled = true` (both are hard prerequisites for SFTP).
- `sftp_enabled` and `local_user_enabled` bound to `var.sftp_enabled` — so pause/start go through Terraform and never drift.
- Two `azurerm_storage_account_local_user` resources:
  - `svc-inbound`: `home_directory = "sftp-inbound"`, `permission_scope` = that container only, `rwlcd`.
  - `svc-outbound`: `home_directory = "sftp-outbound"`, same but read+list+delete (write only needed if we later push back).
  - **Passwords cannot be set or re-read** — Azure generates them; export the computed `password` attribute to outputs. Losing one means rotating the user, documented.
- Containers: `bronze`, `silver`, `gold`, `exports`, `_control`, `sftp-inbound`, `sftp-outbound`.
- Lifecycle rule: delete `exports/` blobs older than 30 days in dev.
- Firewall: `default_action = "Deny"`, allow `var.admin_ip_cidr`, plus the ADF managed-IR caveat documented (no fixed egress IP → `--allow-all` switch exists for e2e runs).
- RBAC: Storage Blob Data Contributor for the ADF MI and the Databricks access connector MI.

**unity_catalog module**
- Account-level aliased provider for `databricks_metastore` + `databricks_metastore_assignment`.
- Workspace-level provider for `databricks_storage_credential`, one `databricks_external_location` per lake container (`bronze`, `silver`, `gold`, `exports`, `_control`), catalogs/schemas.
- Grants per the guide's governance model: engineers read/write the engineering schemas, analysts `SELECT` on `gold` only, the job service principal least-privilege (detail in the POC plan §8 Phase 2).
- Explicit `depends_on` chain: metastore → assignment → credential → external location → catalog → schemas.

**adf module**
- `ls_adls` (MI), `ls_sftp` (host `<account>.blob.core.windows.net:22`, user `<account>.svc-inbound`, password from KV, **managed Azure IR**), `ls_databricks`, `ls_keyvault`.
- Pipelines rendered from `adf/pipelines/*.json` so they stay reviewable in git.
- Daily trigger with retries and a failure-path activity. **In Phase 1 only a sanity pipeline (SFTP → Bronze copy) is deployed and the daily trigger is created stopped** — the Databricks job it calls does not exist until Phase 2.
- Terraform is the only deployer of ADF objects; `deployment/deploy_adf.sh` (Phase 2) wraps a targeted `terraform apply`.

**Every module ships `outputs.tf`** — the validation scripts and Phase 2 config depend on them.

## 1.3 Lifecycle scripts (`Accounting Engine/scripts/`)

```
scripts/
├── lib/{common.sh,azure_auth.sh,log.sh}     # one auth entry point, one logging format
├── 00_check_prereqs.sh
├── 01_create_infra.sh                       # creates everything, idempotent
├── 02_attach_config.sh                      # terraform outputs → project config
├── 03_pause_infra.sh                        # terraform apply -var sftp_enabled=false
├── 04_start_infra.sh                        # reverse of pause (NEW, you asked for start)
├── 05_validate_infra.sh                     # validates every service
├── 06_destroy_infra.sh                      # typed confirmation, Terraform only
└── sftp_upload.sh                           # drop a file for the pipeline to pick up
```

Behavioural contract for all of them:
- `set -Eeuo pipefail`, `--help`, `--dry-run` where mutating, consistent `[ OK ] / [FAIL]` output.
- Idempotent: `01` on an existing environment converges rather than erroring; `03`/`04` are reversible in either order.
- State in `config/dev/azure.state` (`ABSENT | RUNNING | PAUSED`), so every script can state where you are.
- `06_destroy` requires typing `DESTROY-DEV`; never calls `az group delete` (Terraform is the only owner).
- Nothing writes secrets to stdout; SFTP credentials land in `config/dev/.sftp.env` at mode 0600.

## 1.4 Validation scripts

`05_validate_infra.sh` — one command, per-service checks, non-zero exit on any failure:

| Service | Check |
|---|---|
| Resource groups, all resources | `az resource list` count + Terraform `plan -detailed-exitcode` expecting **0** (proves no drift) |
| Storage | `az storage container list` returns all 7; HNS on; SFTP enabled state matches `azure.state` |
| SFTP | real `sftp` connection as **each** local user, `pwd` == its home container, `ls`; `svc-inbound` gets a write/delete probe; `svc-outbound` (no write permission) reads and deletes a probe file seeded through the ADLS API, and a write attempt must be refused |
| Key Vault | all expected secrets present; `az keyvault secret show` succeeds |
| ADF | factory reachable, 4 linked services present, pipelines present, trigger state as expected (stopped throughout Phase 1; from Phase 2 it matches `azure.state`) |
| Databricks | workspace provisioning state `Succeeded`, cluster policy exists, catalog + 4 schemas visible via CLI |
| Unity Catalog | metastore assigned, all 5 external locations valid (`databricks external-location validate` or equivalent), grants present |
| Monitoring | action group + diagnostic settings attached; budget present with alerts |
| Cost guardrail | `az consumption budget show` confirms threshold set |

Plus `scripts/lib/` unit-testable helpers so validation logic is testable without Azure.

## 1.5 Sanity tests

`Accounting Engine/tests/sanity/` — pytest, using `pytest.mark.azure` so they skip cleanly without credentials:

- `test_prereqs.py` — every required CLI present and above the version floor.
- `test_terraform.py` — `terraform fmt -check`, `terraform validate` on every module; `terraform plan` exits 0 (no-op) on a created environment.
- `test_azure_resources.py` — each resource exists with the expected SKU/config (storage HNS, SFTP state, workspace premium, ADF linked services).
- `test_sftp_roundtrip.py` — as `svc-inbound`: upload via the real script, list it, read it back, verify sha256, delete it. As `svc-outbound`: fetch a file seeded through the ADLS API, verify sha256, delete it, and assert an upload is refused.
- `test_rbac.py` — ADF MI and access connector hold Blob Data Contributor; SFTP local users have **no** access to `gold`/`silver`/`_control` (asserts the permission-scope boundary); an analyst-scoped identity can read `gold` and cannot read `silver`.
- `test_lifecycle.py` — assert pause→`plan` is clean, start→`plan` is clean, and state file transitions are correct. Uses `--dry-run` so it cannot flip a real environment by accident.

Run: `task phase1:test` (or `pytest tests/sanity -m azure`).

## 1.6 Phase 1 exit criteria

1. `00_check_prereqs.sh` green on a clean machine.
2. `01_create_infra.sh` completes with **zero manual steps**.
3. `05_validate_infra.sh` all green.
4. Sanity suite passes.
5. `03_pause_infra.sh` → `terraform plan` shows **no drift**; `04_start_infra.sh` restores it.
6. `06_destroy_infra.sh` leaves only the tfstate RG, and `01` recreates successfully.
7. `docs/PHASE1_INFRA.md` documents every resource, script, and check.

**Cost:** creation day ≈ $8–14 (SFTP $0.30/hr is the driver), then < $2/mo paused.

---

# PHASE 2 — The real project

- `src/accounting_engine/` — `common` (config, io, spark, logging, control tables), `bronze`, `silver`, `gold`, `export`, `dq`, exactly as specified in `docs/plans/accounting-engine-azure-poc.md`.
- Guide-driven items added in the 2026-11-15 review (detail in the POC plan §8): additive-vs-breaking schema handling at Bronze, range checks and per-column null policy in Silver, Gold partitioned by processing date (never by schedule date), `gold/accounting_output` as the retained export source, output schema gate before export, late-file alert in ADF, and the performance work (broadcast reference joins, skew handling, `OPTIMIZE`/`VACUUM`).
- Jenkins: feature branches and PRs run the quality gate only; deployment stages run on `main` after approval.
- Local/azure path abstraction in `common/io.py`; `dbutils` confined there.
- Infra variables embedded: `02_attach_config.sh` generates `config/dev/azure.env` + `pipeline.yml`; the app reads only those.
- Amortization engine (straight-line + declining balance), adjustments with future-period re-amortization, SCD2 account state, reconciliation with control totals, deterministic CSV.GZ export.
- `databricks.yml` bundle with the 4-task workflow; ADF invokes it via the Jobs API.
- **Pytest**: `unit` (property-based amortization math via hypothesis), `contract` (layer schemas), `integration` (local Spark + Delta on `file://`), `e2e` (compose SFTP → full pipeline; `-m azure` for real Azure).
- Skills applied: `spark-and-distributed-processing`, `lakehouse-table-format-engineering`, `data-quality-and-contract-testing`, `warehouse-and-schema-design`, `airflow-and-workflow-orchestration`.
- Docs: `README.md`, `docs/HOW_TO_RUN.md`, `docs/HLD.md`, `docs/LLD.md`, `docs/COMPONENTS_AND_ALTERNATIVES.md`, `docs/GUIDE_TRACEABILITY.md` (each guide section and Q&A → the file or test that demonstrates it).

---

# PHASE 3 — Data, analysis, and UI

- `tools/generate_sample_data.py` — on-demand volumes (`--rows/--days/--seed/--region`), `--dirty` for edge cases, `--schema-break rename|drop|retype` for the upstream-schema-change scenario, `--skew` for hot-account data; wired to `sftp_upload.sh`.
- Analysis: `analysis/` notebooks/scripts for reconciliation trends, DQ pass rates, schedule projections; parquet/Delta reads so it works locally and on Azure.
- Local lifecycle: `scripts/local/{up.sh,down.sh,pipeline.sh,e2e_azure.sh}` + `docker-compose.dev.yml` (SFTP, local Spark, Jenkins, Streamlit).
- **Streamlit UI** (`ui/app.py`) — this replaces the FastAPI monitoring API and `dashboard.html` from the earlier POC draft: one page per phase — Infrastructure (create/pause/start/validate/destroy with live status), Data (generate + upload), Pipeline (run/rerun/force, live run log), Analysis (DQ + reconciliation tables and charts), Docs. Every button shells out to the *same* script the CLI uses, so there is no second implementation to drift. Guardrails: destroy requires typed confirmation, destructive actions show what will happen first, and a long-running action streams logs.
- `README.md` covering all three phases, plus `docs/` index.

---

## Cross-cutting conventions

- **`Accounting Engine/`** holds all project code, the plans and the guide; repo root keeps only `.gitignore`.
- `.gitignore` already blocks `*.tfstate`, `azure.env`, `.sftp.env`, `data/`, `*.csv.gz`; will add `config/*/azure.state` and `ui/.streamlit/`.
- Commits per phase, descriptive messages, pushed to `main`.
- Nothing is installed, applied, or destroyed without asking you first.

## Risks / open items

- **SFTP is the cost driver** (~$7.20/day while enabled) — pause disables it.
- Native SFTP requires **public network access** (ADF managed IR has no fixed egress IP); private networking is Phase-2+ hardening.
- Unity Catalog metastore is **account-level and region-scoped** — creating it affects the whole subscription in `eastus2`; if one already exists I reuse it rather than create a second.
- The `python3` on this machine is **3.14**, which may lack wheels for some Spark-adjacent packages. Plan: use a pinned venv (`python3 -m venv`, 3.11/3.12 via Homebrew if 3.14 fails), documented in prerequisites.
- **Storage firewall vs Databricks compute — to verify in Phase 1.** With `default_action = "Deny"`, job clusters in the workspace's managed VNet may be unable to reach the lake even with the access connector's RBAC in place. Candidate fix is a storage resource-instance rule for the access connector; this is not yet confirmed against Microsoft's docs, so `05_validate_infra.sh` includes a cluster-side read/write probe and the fix is chosen from what that shows.
- **Commit flow.** "Pushed to `main`" above is your stated convention. The guide's CI story is feature branch → PR → Jenkins gate; if you want the repo history to show that, switch to one PR per phase — your call, not changed here.
