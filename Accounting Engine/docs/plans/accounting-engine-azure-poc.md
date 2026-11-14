# Accounting Engine — Azure POC Implementation Plan

Greenfield build in `/Users/nageshbhosle/sushama-project`. The folder currently holds only `Accounting_Engine_Project_Interview_Guide.docx` and `llm.txt` — no code, no git repo.

**Goal:** a production-shaped replica of the banking Accounting Engine described in the guide (SFTP in → ADF → ADLS Gen2 Bronze/Silver/Gold Delta → Databricks/PySpark accounting + amortization logic → daily CSV.GZ out), where **you never touch Azure tooling yourself**: I own the infrastructure, you get four lifecycle scripts plus one upload script.

| You asked for | Where it lives |
|---|---|
| 1. Create Infrastructure | `scripts/01_create_infra.sh` |
| 2. Attach infra variables to the project running locally | `scripts/02_attach_config.sh` |
| 3. Pause infra | `scripts/03_pause_infra.sh` |
| 4. Delete infra | `scripts/04_destroy_infra.sh` |
| Terraform in its own folder, fully automatic | `infra/terraform/` |
| Script to drop a file on the SFTP server | `scripts/sftp_upload.sh` |
| README / how-to-run / HLD / LLD / components + alternatives | `README.md`, `docs/HOW_TO_RUN.md`, `docs/HLD.md`, `docs/LLD.md`, `docs/COMPONENTS_AND_ALTERNATIVES.md` |

**Confirmed decisions:** subscription exists, you are **Owner** at subscription scope; region **eastus2**; auth model **`az login` interactively + service principal created by script 01 for Jenkins**; local-first exploration; `destroy` is the default "done for now" action.

---

## 0. Corrections applied after design review

Verified against Microsoft Learn + Terraform/Databricks registries. These were errors in the earlier draft:

| # | Was wrong | Correct now |
|---|---|---|
| 1 | Terraform generates the SFTP password (`random_password`) | **Azure generates it**; you cannot set or re-retrieve it. Read the computed `password` attribute off `azurerm_storage_account_local_user`. If lost → rotate the user. |
| 2 | `home_directory = "/"` | Home directory must be a **container path** (`sftp-inbound`). Without a valid one, connections fail with `Home Directory not accessible`. |
| 3 | One local user for inbound + outbound | **Cross-container traversal in one connection is unsupported.** Two local users, one per container. |
| 4 | `chmod` in the SFTP batch; `put` then rename over existing file | Drop `chmod`; rename onto an existing name is a **protocol violation** → delete-then-rename or unique timestamped name. |
| 5 | Trust Azure-side MD5 | **Content-MD5 is null for SFTP uploads** → compute sha256 client-side and verify against the landing copy. |
| 6 | Unity Catalog catalogs created from the workspace provider | `databricks_metastore` / `databricks_metastore_assignment` are **account-level**. Need an aliased account provider + `DATABRICKS_ACCOUNT_ID`; the metastore must exist before catalogs. |
| 7 | `az provider register -n A/B/C --wait` | One namespace per invocation. |
| 8 | Assumed RBAC covers SFTP users; assumed port 22 closed when disabled | Local users **do not interoperate with RBAC/ABAC**; **port 22 stays open even when SFTP is disabled** (requests just disconnect). Firewall + key auth required. |
| 9 | Pause toggles SFTP via `az`, which `terraform apply` silently reverts | SFTP on/off driven by **`TF_VAR_sftp_enabled`**; pause/resume go through Terraform so state and reality never diverge. |
| 10 | `pyspark 4.0.*` + `delta-spark 4.0.*` | Delta's compatibility table: 4.2.x/4.3.x cover Spark 4.0–4.1, while `delta-spark 4.0.x` pairs only with Spark 4.0.x. Pinned to **pyspark 4.1.x + delta-spark 4.2.x**, matching **Databricks Runtime 18.x LTS (Spark 4.1)**. |
| 11 | `az group delete` used alongside Terraform destroy | **Terraform destroy only** — one owner of the resource group. |
| 12 | Paused cost "~$3–5/mo" derived loosely | ADF has no idle charge; paused is storage + empty workspace only, **< $2/mo**. SFTP is the only hourly cost: **$0.30/hr ≈ $7.20/day while enabled**. |

Additional verified facts the design now depends on: SFTP requires a **general-purpose v2** account **with hierarchical namespace enabled**; **max 100 containers per local user** (we use 1 each); **locate a local user's own ACLs carefully** — default/extra ACLs anywhere in the access path cause `Permission denied`; **idle connections time out after 2 minutes**; **max upload 500 GB**; SFTP blocks `symlink`, `df`, and random-write operations; **you must disable SFTP before changing redundancy/replication**.

---

## 1. How the scripts connect to Azure

I cannot log in for you — no browser or credential in my sandbox. Every script authenticates through machinery on your machine. **One 30-second manual step, total.**

**Step 1 — one-time interactive login**
```bash
az login                                  # --use-device-code on a headless box
az account show --output table            # confirm subscription id + tenant
```
Writes the MSAL token cache to `~/.azure/`; all scripts read from it.

**Step 2 — Terraform inherits that token.** The `azurerm` provider uses the Azure CLI credential chain automatically, so `apply` runs as *your* identity. No secrets on disk beyond the existing token cache. `.gitignore` covers `*.tfstate`, `azure.env`, `.sftp.env`.

**Step 3 — script 01 mints a CI-only service principal**
```bash
az ad sp create-for-rbac --name sp-accteng-dev-cicd --role Contributor \
  --scopes /subscriptions/$SUB_ID --years 1
```
`--role` and `--scopes` are required — without them the SP exists with no access. Secret goes to Key Vault + Jenkins credentials, never to the repo or stdout logs. Interactive scripts keep using `az login`; Jenkins uses `ARM_CLIENT_ID/ARM_CLIENT_SECRET/ARM_TENANT_ID/ARM_SUBSCRIPTION_ID`.

**Step 4 — Databricks CLI reuses the same AAD token.** `databricks auth login --host <workspace-url> --profile dev` (OAuth via `az`). Fallback for tenants that block it: mint a PAT via `az rest` and store it in Key Vault.

**Step 5 — provider registration**, one call per namespace:
```bash
for ns in Microsoft.Databricks Microsoft.DataFactory Microsoft.KeyVault; do
  az provider register -n "$ns" --wait
done
```

**Step 6 — account-level Databricks auth** (needed only once, for the Unity Catalog metastore): Terraform's `databricks` provider runs with two aliases — `workspace` (per-workspace host) and `account` (`accounts.azuredatabricks.net` + `DATABRICKS_ACCOUNT_ID`), the latter resolved from `az databricks workspace list`/ARM.

All auth lives in `scripts/lib/azure_auth.sh`; a missing token yields exactly one message: `run az login first`.

---

## 2. Design decision: native Azure Blob SFTP, no VM, no Integration Runtime

A self-hosted Integration Runtime requires a manual key exchange and a VM — incompatible with "fully automatic infra, four scripts". **Azure Blob Storage native SFTP** on the existing HNS-enabled account gives a real SFTP endpoint (`<account>.blob.core.windows.net:22`) that ADF reaches with the **managed Azure IR**.

Constraints this imposes, all now handled explicitly:
- Requires **GPv2 + HNS** (our lakehouse account already satisfies both).
- **$0.30/hr while enabled** → enabled only in `RUNNING`, disabled in `PAUSED`, gone in `DESTROYED`.
- **One container per connection** → two local users, `svc-inbound` (home `sftp-inbound`) and `svc-outbound` (home `sftp-outbound`).
- **No RBAC for local users** → access is container-scoped permission sets only; neither user gets `gold`, `silver`, or `_control`.
- **Port 22 is open regardless** → storage firewall restricted to your admin IP; SSH key auth enabled alongside password; storage access keys never leave Key Vault.
- **Public network access required** → ADF managed IR has no fixed egress IP, so "allow all networks + firewall" is a documented POC risk, with private endpoint/managed VNet listed as prod hardening.

VM + self-hosted IR remains documented in `COMPONENTS_AND_ALTERNATIVES.md` as the private-network alternative, including the manual step it reintroduces.

---

## 3. Azure inventory (Terraform, `eastus2`)

| # | Service | Role |
|---|---|---|
| 1 | `rg-accteng-dev` | all app resources — destroyed by script 04 |
| 2 | `rg-accteng-tfstate` | Terraform state only — **never destroyed** |
| 3 | Storage Account (GPv2, HNS, SFTP) | `bronze/ silver/ gold/ exports/ _control/` + `sftp-inbound/ sftp-outbound/` + 2 local users |
| 4 | Data Factory | ingestion, orchestration, outbound, failure paths, managed Azure IR |
| 5 | Databricks Workspace (Premium) | Spark, Unity Catalog |
| 6 | Databricks Access Connector (MI) | Unity Catalog → ADLS |
| 7 | Unity Catalog metastore + assignment | **account-level**; created once, reused by DEV/UAT/PROD |
| 8 | Key Vault | SFTP creds (2 users), Databricks token, Jenkins SP secret |
| 9 | Managed Identities + RBAC | ADF→ADLS, connector→ADLS, CI SP→RG |
| 10 | Log Analytics + Monitor action group | ADF/Databricks failures → email, 5 GB/mo cap |
| 11 | Budget + alerts | 50/80/100% |

**Omitted deliberately** (each with its alternative in the components doc): metadata DB, private endpoints/VNet injection, Bastion, Purview, dedicated Jenkins agents, HDInsight, self-hosted IR.

### Cost

| State | Billed | Est. |
|---|---|---|
| Local only (daily work) | nothing | **$0** |
| `PAUSED` (SFTP off, triggers off) | storage + empty workspace | **< $2/mo** |
| `DESTROYED` | tfstate account only | **~$0.02/mo** |
| `RUNNING` | SFTP $0.30/hr ($7.20/day) + ADF runs + Databricks jobs | **~$8–14 per active day** |

Levers: jobs clusters only, single node, `autotermination_minutes = 10`, 30-day export retention in DEV, SFTP enabled only while RUNNING. **Being honest: SFTP is the cost driver, and it is unavoidable if you want a real SFTP endpoint.** If ~$8/day is too much, the local-only path exercises the identical pipeline for $0 and Azure is used only for occasional e2e.

---

## 4. The four lifecycle scripts

Idempotent, re-runnable. State in `config/dev/azure.env` + `config/dev/azure.state`.

### `scripts/01_create_infra.sh` (~25–35 min, fully automatic)
0. `source scripts/lib/azure_auth.sh` — token, subscription, region, Owner check; install hints for `terraform`/`databricks`/`az` (none installed here yet).
1. Register providers (one namespace per call).
2. `terraform -chdir=infra/terraform/bootstrap apply` → state RG + container.
3. `terraform -chdir=infra/terraform/envs/dev apply` with `TF_VAR_sftp_enabled=true` → storage (HNS, SFTP, 2 local users), Key Vault, workspace, access connector, UC metastore + assignment + catalog + schemas, ADF + linked services + pipelines + trigger, monitoring, budget.
4. Post-apply: read the **Azure-generated passwords** from Terraform outputs into `config/dev/.sftp.env` (mode 0600) and Key Vault; create the Databricks secret scope; UC grants; ADF triggers on; create the CI SP.
5. `databricks bundle validate && databricks bundle deploy -t dev`.
6. Smoke run on generated sample data (upload → wait → verify outbound checksum).
7. Print outputs + next commands.

`--resume` is the pause→running path (re-enables SFTP via Terraform, starts triggers, redeploys bundle if changed).

### `scripts/02_attach_config.sh` (~5 s)
The only bridge between infra and code — reads `terraform output -json`.
- `config/dev/azure.env`: `ADLS_ACCOUNT`, `STORAGE_BACKEND=azure`, `DATABRICKS_HOST`, `DATABRICKS_TOKEN`, `ADF_FACTORY`, `SFTP_HOST`, `SFTP_USER_INBOUND`, `SFTP_USER_OUTBOUND`, `SFTP_PORT=22`, `KEYVAULT_URI`, `EXPORT_PATH`.
- Regenerates `config/dev/pipeline.yml` from a template (`abfss://` vs `file://`).
- `--local` re-points the same file at the compose backend.
- Prints a green/red connectivity table (ADLS list, ADF ping, cluster list, SFTP login check for each user).

### `scripts/03_pause_infra.sh` (~2–3 min, → < $2/mo)
Stop ADF triggers → await in-flight runs → cancel active Databricks runs → **`terraform apply -var sftp_enabled=false`** (drift-free, not `az`) → state `PAUSED`. Data untouched.

### `scripts/04_destroy_infra.sh` (~10 min, → ~$0)
Typed `DESTROY-DEV` (or `--yes`): stop triggers → `databricks bundle destroy -t dev` → `terraform destroy` of `envs/dev` → optional `--purge` Key Vault → **leaves the metastore assignment cleared and `rg-accteng-tfstate` intact** → state `DESTROYED` + recreate command. Never `az group delete`.

`Taskfile.yml`: `task infra:create|attach|pause|destroy|resume`.

---

## 5. SFTP upload script

`scripts/sftp_upload.sh` — real `sftp` against the Azure endpoint.

```
./scripts/sftp_upload.sh                  # generate today's sample + upload
./scripts/sftp_upload.sh --file x.csv.gz  # upload an existing file
./scripts/sftp_upload.sh --dirty          # intentionally bad file
./scripts/sftp_upload.sh --list           # ls sftp-inbound/
./scripts/sftp_upload.sh --fetch          # pull latest outbound CSV.GZ
./scripts/sftp_upload.sh --local          # target the compose SFTP instead
```

Implementation notes that matter given the platform limits:
- Connects as `svc-inbound` whose home is `sftp-inbound` — **no cross-container navigation**, so `--fetch` opens a second connection as `svc-outbound`.
- Batch: `-o StrictHostKeyChecking=accept-new`, `put`, `ls -l`, `bye`. **No `chmod`** (unsupported semantics).
- **Atomic publish:** upload to `<name>.<epoch>.part`, then rename — rename target must not exist, so the final name is deleted first if present (delete permission granted to the user).
- Computes **sha256 client-side** (Content-MD5 is null for SFTP uploads) and writes `<file>.sha256` next to the artifact; the pipeline compares it against the Bronze landing copy.
- Idle timeout is 2 minutes → short-lived connections, no `--persist`.
- Non-zero exit on any failure so Jenkins can gate on it.

---

## 6. Architecture

```
scripts/sftp_upload.sh  →  sftp-inbound/     (native SFTP, 2 local users, no RBAC)
                                 │  ADF pl_ingest_daily: presence + size + checksum → Copy (Azure IR)
                                 ▼
                    bronze/source=accounting/region=NAM/dt=YYYY-MM-DD/  (raw CSV.GZ + audit cols)
                                 │  Databricks: bronze_ingest (CSV → Delta, checksum dedupe)
                                 ▼
                    bronze/accounting_delta/dt=…
                                 │  Databricks: silver_transform (contract, DQ, dedupe, standardize)
                                 ▼
                    silver/accounting/dt=…  +  silver/accounting_exceptions
                                 │  Databricks: gold_accounting (amortization, adjustments,
                                 │  SCD2 account state, reconciliation/control totals)
                                 ▼
                    gold/amortization_schedule, gold/account_state, gold/accounting_output
                                 │  Databricks: export_generate (deterministic CSV.GZ + sha256 + trailer)
                                 ▼
                    exports/accounting_output/dt=…  ──ADF Copy──►  sftp-outbound/
```

`_control/` Delta tables: run registry, DQ results, control totals, file checksums — a metadata DB at zero extra cost.

**Storage abstraction enables local-first:** every path resolves via `src/accounting_engine/common/io.py` + `config/<env>/pipeline.yml`; identical Spark code runs `file://` in Docker and `abfss://` in Azure; `dbutils` confined to that module.

---

## 7. Repository layout

```
sushama-project/
├── README.md  Taskfile.yml  databricks.yml  Jenkinsfile
├── .gitignore .editorconfig pyproject.toml .pre-commit-config.yaml
├── docs/{HOW_TO_RUN.md,HLD.md,LLD.md,COMPONENTS_AND_ALTERNATIVES.md}
├── scripts/
│   ├── lib/azure_auth.sh
│   ├── 01_create_infra.sh  02_attach_config.sh  03_pause_infra.sh  04_destroy_infra.sh
│   └── sftp_upload.sh
├── infra/terraform/
│   ├── bootstrap/
│   ├── modules/{storage,keyvault,databricks,unity_catalog,adf,monitoring,budget}
│   └── envs/{dev,uat,prod}/{main.tf,backend.tf,terraform.tfvars}
├── src/accounting_engine/
│   ├── common/{config.py,io.py,spark.py,logging.py,control_tables.py,delta.py}
│   ├── bronze/ingest.py
│   ├── silver/{transform.py,contracts.py,quality.py,standardize.py}
│   ├── gold/{amortization.py,adjustments.py,account_state.py,reconciliation.py,run_gold.py}
│   ├── export/{generate.py,control_totals.py,csv_gz.py}
│   └── dq/{rules.py,exceptions.py}
├── adf/{pipelines,datasets,linked-services,triggers}/
├── config/{dev,uat,prod}/pipeline.yml + templates/
├── tests/{unit,contract,integration,e2e}/
├── tools/generate_sample_data.py
├── docker/{jenkins/,sftp/,spark/,monitoring/}
├── docker-compose.dev.yml
└── deployment/{deploy_adf.sh,deploy_databricks.sh,rollback.sh}
```

---

## 8. Implementation phases

**Phase 1 — skeleton + scripts.** `git init`; `.gitignore` (`*.tfstate`, `azure.env`, `.sftp.env`, `data/`, `*.csv.gz`); `pyproject.toml` (`pyspark 4.1.*`, `delta-spark 4.2.*` — Delta's table covers Spark 4.0–4.1; `pandas`, `pyyaml`, `pytest`, `hypothesis`, `ruff`, `mypy`, `fastapi`, `azure-identity`, `databricks-sdk`); `Taskfile.yml`; the six scripts + `lib/azure_auth.sh`; `bootstrap/`. Requires **Java 17+** locally for Spark 4 — verified, not assumed.

**Phase 2 — Terraform** (`envs/dev`, `location = "eastus2"`, everything from `terraform.tfvars`):
- `storage`: GPv2 HNS account; containers; `sftp_enabled` + `local_user_enabled` driven by **`var.sftp_enabled`**; two `azurerm_storage_account_local_user` blocks (`svc-inbound` → `sftp-inbound`, `svc-outbound` → `sftp-outbound`) with `ssh_password_enabled` and `permission_scope` limited to their own container; **passwords are exported attributes**; firewall = admin IP; export lifecycle > 30 d; RBAC for ADF MI + connector.
- `keyvault`: SFTP creds, Databricks token, Jenkins SP secret; ADF KV linked service; Databricks secret scope on KV.
- `unity_catalog`: account-aliased provider → `databricks_metastore` + `databricks_metastore_assignment`; workspace provider → `databricks_storage_credential`, `databricks_external_location` (`abfss://gold@…`), catalog `accteng_<env>`, schemas `bronze/silver/gold/control`. Ordering enforced with `depends_on`.
- `databricks`: Premium workspace, access connector, cluster policy (jobs-only, single node, autoterminate 10).
- `adf`: factory; `ls_adls` (MI); `ls_sftp` (host `<account>.blob.core.windows.net`, port 22, username `<account>.<localuser>`, password from KV, **managed Azure IR**); `ls_databricks`; `ls_keyvault`; datasets; pipelines rendered from `adf/pipelines/*.json`; 02:00 UTC trigger with retries; failure-path activities.
- `monitoring`: Log Analytics, ADF `PipelineFailedRuns`/`ActivityFailedRuns`, Databricks job failures, action group → email, budget 50/80/100%.
- Every module ships `outputs.tf` (script 02 depends on them).

**Phase 3 — sample data + upload.** `tools/generate_sample_data.py`: deterministic `--seed/--rows/--days/--region`; `--dirty` emits nulls in mandatory fields, duplicate `event_id`, future dates, zero/negative amounts, bad dates, an extra column. Feeds both the upload script and pytest fixtures. **File size kept well under the 500 GB SFTP cap; realistic 4–6 GB sizes are a documented scale assumption, not generated by default.**

**Phase 4 — Bronze.** Presence/size/row-count, reject `.part`/`.tmp`, sha256 verification against the client-side sidecar, add `_batch_id,_source_file,_file_checksum,_ingested_at,_row_number`; duplicate-file detection vs `_control/file_registry` → `DUPLICATE_FILE`; Delta write partitioned `dt, region, source` with `replaceWhere` on `_batch_id`.

**Phase 5 — Silver.** Explicit schema contract; unknown columns ignored/quarantined (never "mergeSchema fixes it"). Rules: mandatory nulls, datatype/date checks, duplicate `(account_id,event_id)`, `event_date <= processing_date`, known `event_type`, non-zero financial amounts, referential integrity. Failures → `silver/accounting_exceptions` with rule id + reason; results → `_control/dq_results`; hard-fail threshold blocks Gold.

**Phase 6 — Gold (core).**
- `amortization.py`: `straight_line` + `declining_balance`, day-count/rounding from config; rows `(account_id, period_seq, period_start, period_end, opening_balance, interest_amount, principal_amount, scheduled_payment, closing_balance, version, effective_date)`; invariants `closing = opening - principal`, `Σ principal = principal`.
- `adjustments.py`: draws/payments/refunds/fees; re-amortize **only affected future periods**; audit in `gold/account_event_ledger`.
- `account_state.py`: SCD2 via Delta `MERGE`, exactly one `is_current=true`, history retained.
- `reconciliation.py`: per region/dt control totals (row counts, debit/credit/interest sums, opening+activity=closing) + outbound trailer; mismatch fails the batch and blocks export.
- Guide's Account A (+500 credit) worked example as a golden-file test.

**Phase 7 — Export.** Deterministic column order, sorted rows, ISO dates, `gzip -9`, atomic write, `.sha256`, `_control/control_totals`. `pl_outbound_daily`: Copy → `sftp-outbound/`, verify remote size + sha256, idempotent via `run_registry`, never duplicates.

**Phase 8 — Databricks bundle.** Workflow `accounting_daily`: `bronze_ingest → silver_transform → gold_accounting → export_generate`, `max_retries: 2`, policy-pinned single node, wheel from `src/`; targets dev/uat/prod; params `dt, region, force`; invoked by ADF via Jobs API. Runtime pinned to **18.x LTS (Spark 4.1)** to match the local pins; confirm the exact LTS label at create time rather than hard-coding blind.

**Phase 9 — Tests.** `unit` (amortization property-based via `hypothesis`, adjustments, SCD2, DQ rules, CSV.GZ determinism, config); `contract` (Silver/Gold/export schemas, unexpected columns, null policy); `integration` (local Spark + Delta on `file://`, full run on clean and dirty data, rerun identity, duplicate skip); `e2e` (compose SFTP + Spark; `@pytest.mark.azure` skipped without creds).

**Phase 10 — Monitoring API.** FastAPI `/health`, `/runs`, `/dq`, `/control-totals`, `/reconciliations` from `_control/*` + ADF run history; `dashboard.html`; local compose only, no Azure compute.

**Phase 11 — Jenkins in Docker.** `Jenkinsfile`: checkout → cached venv → ruff/mypy → pytest → `terraform fmt -check`/`validate` → `bundle validate` → [main] plan → approval → apply dev → bundle deploy → `deploy_adf.sh`, auth via SP `ARM_*` env vars. `deployment/rollback.sh` redeploys the prior bundle version and disables triggers.

**Phase 12 — Docs.**
- `README.md`: what/why, architecture summary, service list, cost table, the scripts at a glance, platform limits (2-min idle timeout, 500 GB cap, no symlinks, port 22 always open).
- `docs/HOW_TO_RUN.md`: prerequisites (`az login`, Java 17+, CLI installs), `./scripts/01_create_infra.sh → 02_attach_config.sh → sftp_upload.sh → verify export`, local-only mode, rerun/backfill/force, troubleshooting incl. `Home Directory not accessible` and `Permission denied` ACL causes, teardown.
- `docs/HLD.md`: context diagram, layer responsibilities, orchestration + failure paths, environment model, scale assumptions (4–6 GB NAM daily, 1B+ monthly, 60–90 min end-to-end).
- `docs/LLD.md`: module/class layout, every Delta table schema + grain, function signatures, config keys, idempotency keys, DQ rule ids, sequence/state diagrams, test map.
- `docs/COMPONENTS_AND_ALTERNATIVES.md`: per component — ADLS Gen2, ADF, Databricks, Delta, PySpark, Key Vault, Unity Catalog, SFTP (native vs VM+self-hosted IR vs Blob REST), Jenkins, Terraform, pytest, FastAPI — why chosen, alternatives rejected (Iceberg/Hudi, Fabric/Airflow, Synapse, plain Parquet, Postgres metadata DB, GitHub Actions), and the trade-off.

---

## 9. Contracts

| Layer | Grain | Contract |
|---|---|---|
| Bronze | file row | raw columns + audit cols; no business transforms; sha256 verified |
| Silver | `(account_id, event_id)` | typed, deduped, explicit schema, exceptions isolated |
| Gold `amortization_schedule` | `(account_id, version, period_seq)` | balances balance, valid dates, `Σ principal = principal` |
| Gold `account_state` | `(account_id, version)` | exactly one `is_current=true` |
| Export | `(account_id, period_seq)` | byte-deterministic CSV.GZ + `.sha256` + control trailer |

Idempotency: writes `replaceWhere`/`MERGE` keyed by `_batch_id`/`version`; `_control/run_registry (pipeline, dt, region, run_id, status)` skips successful batches unless `force=true`.

---

## 10. Verification

1. `task lint && task test` — green locally, no Azure.
2. Local pipeline: `task local:up` → `task local:pipeline DT=2026-11-14` → all layers present, DQ all-pass, byte-stable CSV.GZ.
3. `./scripts/01_create_infra.sh` → all resources in `eastus2`, triggers enabled, bundle deployed, smoke run passes. **No manual step anywhere.**
4. `./scripts/02_attach_config.sh` → connectivity table all green.
5. `./scripts/sftp_upload.sh --file sample.csv.gz` → file listed in `sftp-inbound/`; ADF ingests; export lands in `sftp-outbound/` with matching sha256; Bronze rejects a tampered checksum.
6. Failure paths: missing file, partial file, duplicate file, dirty nulls, oversized/ACL-broken path → expected DQ status/alert, no duplicate export.
7. Rerun same `dt` twice → identical checksum, one export, `run_registry=SKIPPED` (unless `force`).
8. Accounting: Account A adjustment → future periods recalculated, prior versions queryable; reconciliation mismatch blocks export.
9. `./scripts/03_pause_infra.sh` → SFTP disabled **and `terraform plan` reports no drift**; `01 --resume` restores; `./scripts/04_destroy_infra.sh` → `az group list` shows only the tfstate RG and a fresh `01` succeeds.
10. Jenkins branch build runs lint→tests→plan→deploy; failing pytest stops the gate.

## 11. Open items
- `az`, `terraform`, `databricks` CLIs are **not installed** on this machine (only `docker`, `python3`, `java`) — Phase 1 prints exact install commands; nothing applies before that.
- **SFTP password cannot be chosen or re-read.** Terraform captures the Azure-generated value once into `config/dev/.sftp.env` (0600). Losing it means rotating the local user — documented, not hidden.
- **SFTP is the main running cost** (~$7.20/day whenever enabled). Local mode is the $0 alternative.
- Storage public network access is required for ADF's managed IR; private networking is prod hardening.
- `destroy` changes ADF/Databricks resource ids; script 02 re-derives all config from Terraform outputs so nothing goes stale.
