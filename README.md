# Week 1 — GCP Organization, Projects and Identity (Instruqt Track)

Hands-on track on Instruqt (team `instruqt-support`, track slug `gcp-week1-organization-projects-identity`, id `cmbnpxpyim6c`). Learners build a least-privilege identity setup across two fresh GCP projects — every challenge is hands-on, with step-by-step commands hidden in collapsible blocks and a **Check** button that validates the work.

Repo: `shiv-instruqt/GCP-organization-tracks` — connected to the track with Instruqt version control (push to `main` → Instruqt syncs).

## Sandbox (must match the Instruqt sandbox page exactly)

| Sandbox item | Name | Settings |
|---|---|---|
| Host (container) | **`cloud-client`** | `gcr.io/instruqt/cloud-client`, 1024 MB, port 80, `/bin/bash`, env `LANG=C.UTF-8` |
| Google project | **`northwinddev`** | 7 services, region us-central1, 7 roles (incl. `roles/storage.admin`), no admin roles |
| Google project | **`northwindstaging`** | same 7 services and region, 6 roles (no `roles/storage.admin`), no admin roles |

Allowed services (both projects): `cloudresourcemanager`, `serviceusage`, `iamcredentials`, `iam`, `storage`, `artifactregistry`, `run` (`.googleapis.com`). Anything else is refused by the sandbox's `constraints/gcp.restrictServiceUsage` org policy — the track uses that as a hands-on guard-rail lesson (challenge 7).

`policytroubleshooter` and `orgpolicy` are **not** available to this team, so the track diagnoses access denials with `testIamPermissions` + bindings + role inspection, and reads org policies through `gcloud resource-manager org-policies` (Resource Manager API).

Renaming `cloud-client`, `northwinddev` or `northwindstaging` breaks the track: the setup script reads `INSTRUQT_GCP_PROJECT_northwinddev_*` / `INSTRUQT_GCP_PROJECT_northwindstaging_*`, and every challenge's tabs and scripts run on host `cloud-client`.

## Files

| File | What it does |
|---|---|
| `track.yml` | Track metadata written by Instruqt (id, slug, owner, 3 h limit, 30 min idle timeout, skipping enabled, layout). Keep `id` and `revision` as Instruqt sets them. |
| `config.yml` | The sandbox above (host + two Google projects), as published by Instruqt from the sandbox page. |
| `track_scripts/setup-cloud-client` | Runs once on `cloud-client` before challenge 1. Writes the two platform-admin key files to `/root/.keys/`, creates a private gcloud config for the checks (`/opt/lab/gcloud`), disables `iamcredentials`, `artifactregistry` and `run` so learners enable them, creates the note templates in `~/week1`, the help files in `~/week1/help/`, an identity-aware prompt and the automatic command log `~/week1/commands.log`. It never exits non-zero (a failing setup would stop the sandbox); problems go to `/opt/lab/SETUP_ERROR`. |
| `NN-*/assignment.md` | The challenge: YAML header (slug, title, tabs, time limit) + instructions. Goal first, commands hidden in **💡 Show step-by-step instructions**, background hidden in **📖 Background**, checklist at the bottom. |
| `NN-*/check-cloud-client` | Runs when the learner clicks **Check**. Validates every checklist item in order; the first missing one is shown as `FAIL: …`. Uses the private checker config, so learners can't break it by changing their own gcloud setup. |
| `NN-*/solve-cloud-client` | Runs when the learner skips the challenge (and for `instruqt track test`): performs the reference solution so later challenges still work. |

Every challenge has the same four tabs: **Terminal**, **Terminal 2** (both on `cloud-client`, working dir `/root/week1`), **Notes** (code editor on `/root/week1`) and **Cloud Consoles** (service on `cloud-client` port 80 — console links + logins for both projects).

## Challenges

| # | Folder | Learners do | Checklist (check script) |
|---|---|---|---|
| 1 | `01-hierarchy-and-cli-identities` | Explore org → folder → project, project numbers, billing; try `gcloud projects create` (denied); create gcloud configurations `dev` and `staging`, each with that project's platform-admin identity | both configs correct; both project numbers in `01-hierarchy.md`, no `_TODO_` |
| 2 | `02-enable-apis` | Enable `iamcredentials` (dev) and `iamcredentials`, `artifactregistry`, `run` (staging) with gcloud; document which identity did it | APIs enabled; `02-api-enablement.md` lists them with the identity email |
| 3 | `03-developer-identity` | Compare predefined roles; create SA `dev-alice`, bucket `<dev>-app-data`, custom role `appDeveloper` (4 permissions) granted **on the bucket only** | SA, bucket, exact role permissions, bucket binding, no project roles |
| 4 | `04-impersonation` | Grant Token Creator on `dev-alice` (not the project); upload/list as dev-alice; get denied on delete; prove **inheritance** with a second bucket and a temporary project-level grant | Token Creator scoped to SA; `hello.txt`; allowed + denied actions; inheritance denied → allowed → cleaned up |
| 5 | `05-pipeline-identity` | Create `ci-deployer` + `app-runtime` in staging; only `run.developer` + `artifactregistry.writer`; `serviceAccountUser` on app-runtime only; impersonate the pipeline; no keys | SAs; exact project roles; scoped actAs; scoped Token Creator; no user keys; a successful call as ci-deployer |
| 6 | `06-iam-conditions` | Give dev-alice (dev) read access in staging that **expires automatically** (IAM Condition on `request.time`) | conditional binding with title; expiry in the future and ≤ 30 days; no unconditional access |
| 7 | `07-org-policies-boundaries-quotas` | Read org policies (`gcloud resource-manager org-policies`); try enabling a non-allowed API (refused); prove ci-deployer can't read dev data; try to change an org policy (denied); look up a quota | org-policy command run; refused `services enable`; denied cross-project read; notes |
| 8 | `08-failure-drill` | Remove `storage.objects.list` from appDeveloper; see dev-alice denied; diagnose with `testIamPermissions` + bindings + role; broad fix (`storage.admin`) then undo; narrow fix | break logged; denial after break; `05-permission-test.txt` without the permission; role restored to 4 permissions; no broad roles; notes |
| 9 | `09-deliverable` | Identity map, least-privilege table, bundle everything with the command log into `week1-deliverable.md` | all 5 SA emails; all granted roles named; ≥ 15 gcloud commands logged; bundle exists |

## Learner files created by the setup (`~/week1`)

`00-environment.md` (project IDs, identities), `01-hierarchy.md`, `02-api-enablement.md`, `03-identity-map.md`, `04-org-policy-quotas.md`, `05-failure-drill.md`, `06-least-privilege.md` (templates with `_TODO_`), `commands.log` (automatic), `help/cheatsheet.md`, `help/troubleshooting.md`.

## Working on the track

```bash
instruqt track validate          # offline validation
git add -A && git commit -m "…" && git push origin HEAD:main   # Instruqt syncs from main
```

Make the scripts executable if git doesn't keep the mode: `chmod +x track_scripts/* */check-cloud-client */solve-cloud-client`.
