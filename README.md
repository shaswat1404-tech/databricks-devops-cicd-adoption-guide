# Databricks DevOps CI/CD — Asset Bundle Pipeline
### Azure DevOps + Databricks Asset Bundles | Fashion Retail Industry

> A practical guide to adopting Azure DevOps CI/CD for a production Databricks workspace using Databricks Asset Bundles (DAB) — covering pipeline design, environment harmonisation via Azure Key Vault, and day-to-day workflow for managing notebooks, jobs, workflows, and DLT pipelines as code.

---

## Overview

This repository documents the CI/CD setup and day-to-day DevOps workflow for a production Databricks platform in the fashion retail industry. The pipeline was designed and set up by a dedicated DevOps engineer; this guide covers the support work, onboarding of existing assets into the bundle, ongoing maintenance, and the practical patterns used to manage code and asset changes across **Dev and Prod** environments.

**Key technologies:**
- **Databricks Asset Bundles (DAB)** — defines and deploys all Databricks assets (jobs, workflows, DLT pipelines) as code via `databricks.yml`
- **Azure DevOps Pipelines** — automates validation and deployment on branch push/merge
- **Service Principal**  — used for both pipeline authentication and as the job run-as identity in Databricks
- **Azure Key Vault** — environment-specific secrets (e.g. catalog names) stored in AKV; same variable names and secret scope used across environments for harmonisation

---

## Repository Structure

```
project-repo/
├── LIVE/                          # Notebooks — synced to /Workspace/Shared/LIVE
│                                  # structured in a folder/sub-folder tree containing
│                                  # all extraction and transformation logic
├── databricks.yml                 # Databricks Asset Bundle definition
│                                  # defines all jobs, workflows, DLT pipelines
├── azure-pipelines.yml            # Azure DevOps pipeline definition
└── README.md
```

**Notebooks** are managed directly in Git — changes made via PR like any code file.

**Jobs, Workflows, DLT Pipelines** are defined in `databricks.yml` — the UI is read-only for DAB-managed assets. Any config change must go through `databricks.yml` and be deployed via the pipeline.

---

## Pipeline Architecture

### Branch Strategy

```
feature/fix branch
       │
       │  Pull Request
       ▼
  develop branch ──────────────────→ Auto-deploy to DEV
       │
       │  Pull Request + Review
       ▼
   main branch ───────────────────→ Manual approval gate → Deploy to PROD
```

### Pipeline Stages (`azure-pipelines.yml`)

**Stage 1 — Build & Validate** *(runs on every PR to `develop` and every push to `develop`/`main`)*
- Installs Databricks CLI
- Authenticates via AAD token (Service Principal through Azure Service Connection)
- Runs `databricks bundle validate` — static validation of bundle configuration
- Acts as a quality gate; downstream stages depend on this passing

**Stage 2 — Deploy to Dev** *(runs on push to `develop` only)*
- Authenticates via Service Principal
- Syncs notebooks to `/Workspace/Shared/LIVE` in the Dev workspace (for UI visibility)
- Runs `databricks bundle deploy --target dev`
- Deploys to Dev workspace automatically — no manual approval

**Stage 3 — Deploy to Prod** *(runs on push to `main` only)*
- Same steps as Dev deployment but targets the Prod workspace
- Linked to the **`Prod` Azure DevOps environment** — requires **manual approval** before execution
- Prod workspace URL and credentials kept separate from Dev at pipeline variable level

```yaml
# Branch-to-environment mapping
trigger:
  branches:
    include: [develop, main]

# develop → Deploy to Dev (automatic)
# main    → Deploy to Prod (manual approval gate)
```

---

## Environment Harmonisation via Azure Key Vault

Dev and Prod environments differ in catalog name and other environment-specific values. Rather than branching code or maintaining separate config files, harmonisation is achieved through **Azure Key Vault**:

- The **same variable names** and **same secret scope name** are used in all notebooks and `databricks.yml`
- Each environment's AKV holds **different values** for those same keys
- Code is identical across environments — AKV handles what differs

```
Dev AKV                          Prod AKV
├── catalog-name = "dev_catalog" ├── catalog-name = "prod_catalog"
├── storage-account = "stg-dev"  ├── storage-account = "stg-prod"
└── ...                          └── ...

Notebooks always call:
  dbutils.secrets.get(scope="kv-scope", key="catalog-name")
  → returns the right value per environment automatically
```

No environment-specific branches. No hardcoded values. The same code deploys to both environments unchanged.

---

## Databricks Asset Bundles — How Assets Are Managed

DAB manages all non-notebook assets: **jobs, workflows, and DLT pipelines**. Once an asset is managed by DAB, it is **read-only in the Databricks UI** — changes must go through `databricks.yml`.

### Adding a New Asset

Define it in `databricks.yml`, raise a PR, merge, and the pipeline deploys it.

### Modifying an Existing DAB-Managed Asset

Because DAB-managed assets are locked in the UI, the workflow for making config changes is:

```
1. In Databricks UI — click "Disconnect from source"
   (asset is now editable in the UI)

2. Make and test config changes freely in the UI

3. Copy the updated asset configuration (YAML) from the UI

4. Paste updated config back into databricks.yml in Git

5. Raise PR → merge to develop → pipeline deploys to Dev
   (asset automatically reconnects to DAB on deploy)

6. Validate in Dev → raise PR to main → manual approval → deploy to Prod
```

> "Disconnect from source" is a Databricks UI feature that temporarily detaches an asset from DAB control, allowing free editing. On the next `databricks bundle deploy`, the asset is reattached and updated as defined in `databricks.yml`.

### Notebook Changes

Notebooks are not DAB-managed assets — they live in Git as `.py` or `.ipynb` files and are synced to the workspace via `databricks sync`. Changes follow standard Git flow: edit → PR → merge → pipeline syncs to workspace.

---

## Service Principal — Authentication & Run-As Identity

A single Service Principal is used consistently:

| Role | How |
|---|---|
| Azure DevOps → Databricks authentication | AAD token obtained via Azure Service Connection, passed as `DATABRICKS_TOKEN` |
| Bundle deployment identity | Same SP executes `databricks bundle deploy` |
| Job / Workflow run-as identity | SP is set as the **Run as** identity for all deployed jobs |

Using the same SP as both deployer and runner ensures consistent permissions — no gap between what the pipeline can deploy and what the job can execute.

---

## Day-to-Day Workflow

Once the DevOps setup was in place, all enhancements, fixes, and new development followed this process:

**For notebook changes:**
1. Edit notebook in feature branch
2. Raise PR to `develop` → review → merge
3. Pipeline automatically syncs to Dev workspace
4. Validate in Dev
5. PR to `main` → approval → deploy to Prod

**For job / workflow / DLT pipeline changes:**
1. Disconnect asset from source in Databricks UI
2. Make config changes and test in UI (Dev)
3. Copy updated YAML config from UI into `databricks.yml`
4. Raise PR to `develop` → merge → pipeline deploys (asset reconnects to DAB)
5. Validate in Dev
6. PR to `main` → manual approval → deploy to Prod

**For pushing a new code file type** (e.g. `.sql` files alongside `.py` notebooks):
- `databricks.yml` must be updated — DAB needs to know which file extensions to include when syncing and deploying the bundle
- Raise PR to `develop` → merge → pipeline deploys with new file type included

**For processing a new data file type** (e.g. parquet from ADLS):
- Change is made in notebook code only — `databricks.yml` is unchanged
- Follows notebook change workflow above
- `databricks.yml` only changes if a new job, workflow, or DLT pipeline is being added to process it

---

## Support Work — Onboarding an Existing Project to DevOps

The DevOps pipeline and DAB setup were designed and built by a dedicated DevOps engineer. The support work involved in enabling and maintaining this setup included:

- **KT to DevOps engineer** on existing project architecture, asset inventory, and environment structure
- **Onboarding existing assets** into `databricks.yml` — capturing job/workflow/DLT configs from the UI into the bundle definition
- **Rectifying YAML issues** in `databricks.yml` and `azure-pipelines.yml` — fixing config errors that caused validation or deployment failures
- **Testing deployments** in Dev and validating asset behaviour before promoting to Prod
- **Raising PRs** for all code and config changes — no direct commits to `develop` or `main`
- **Using the pipeline for all enhancements** going forward — no manual deployments outside the DevOps process

---

## Key Design Decisions

**Why Databricks Asset Bundles over manual job management?**
Without DAB, job and pipeline configs live only in the Databricks UI — no version history, no review process, no rollback. DAB brings jobs and pipelines into Git, making every change auditable and reversible.

**Why a manual approval gate for Prod?**
Prod deployments require a deliberate human decision. Auto-deploy to Dev is fine for fast iteration; Prod needs a checkpoint.

**Why sync notebooks to `/Workspace/Shared/LIVE`?**
DAB deploys bundle assets to a bundle-managed path. Syncing notebooks to a shared workspace path keeps them visible and accessible to all workspace users regardless of bundle structure — useful for ad-hoc queries and debugging.

**Why same SP for deploy and run-as?**
Simplifies permission management — one identity to maintain, one set of grants to verify. Reduces the risk of permission gaps between deployment and runtime.

---

## Tech Stack

`Azure DevOps` `Databricks Asset Bundles (DAB)` `Azure Databricks` `Delta Live Tables` `Databricks Workflows` `Azure Key Vault` `Service Principal (AAD)` `Databricks CLI` `Python` `YAML` `Delta Lake` `Unity Catalog`
