---
slug: hierarchy-and-cli-identities
id: 4itfqwqjm0ml
type: challenge
title: Explore the hierarchy & set up your CLI identities
teaser: Org, folders, projects, billing - and one gcloud identity per project.
tabs:
- id: 9yctif4zwuaf
  title: Terminal
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: c8lzrlx3mgjv
  title: Terminal 2
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: ofxrers6blav
  title: Notes
  type: code
  hostname: cloud-client
  path: /root/week1
- id: s65gjvb00j9t
  title: Cloud Consoles
  type: service
  hostname: cloud-client
  path: /
  port: 80
difficulty: intermediate
timelimit: 2400
lab_config:
  default_layout_sidebar_size: 0
enhanced_loading: null
---
# Week 1 — GCP organization, projects and identity

You're the new platform engineer at **Northwind Payments** (fictional). Two empty projects, **dev** and **staging**, have just been handed to you. Everything from here on is hands-on: you build a least-privilege identity setup across both projects yourself.

- Each step says **what** to do. The exact commands are hidden — open **💡 Show step-by-step instructions** only if you get stuck.
- Background explanations are hidden too (**📖 Background**) — optional.
- Click **Check** to validate. If something is missing, the message tells you what.
- Your notes live in `~/week1` (*Notes* tab). Every `gcloud` command is logged to `~/week1/commands.log` with the identity that ran it.

> [!IMPORTANT]
> IAM changes take 30–90 seconds to apply. If a test or check fails right after a change, wait a moment and retry.

<details>
<summary>📖 Background: your identities and tabs (optional)</summary>

| Where | Identity |
|-------|----------|
| Terminal (`gcloud`) | the **platform-admin service account** of each project (key files in `/root/.keys/`) — admin for APIs, service accounts, custom roles and IAM, but **not** Owner/Editor |
| *Cloud Consoles* tab | each project's console user, with the same roles — use it to look around and for Policy Troubleshooter |

Help files: `~/week1/help/cheatsheet.md` and `~/week1/help/troubleshooting.md`.

</details>

---

# 1.1 Explore the hierarchy & set up your CLI identities

<details>
<summary>📖 Background: the resource hierarchy (optional)</summary>

```text
Organization (example.com)          <- org policies + IAM set here apply to EVERYTHING below
 └── Folder (e.g. "payments")       <- group projects per team / environment
      ├── Project: dev              <- the unit of isolation: APIs, billing link, quotas, IAM
      │    └── Resources (buckets, service accounts, Cloud Run services ...)
      └── Project: staging
```

- **IAM is inherited downwards.** A role granted on a folder applies to every project in it. You can *add* access lower down, but you can't take away access that was granted higher up (only *deny policies* can).
- **A project has three identifiers:** project **ID** (globally unique, you choose it, never changes), project **number** (Google-assigned, used in service-agent emails) and display **name** (can change).
- **Billing** is linked per project to a *billing account* that lives outside the hierarchy and has its own IAM.
- Projects are usually **vended** by a platform/foundation team (often with Terraform), not created by hand — exactly what happened to you.

</details>

## Your goal

1. Make the `gcloud` CLI use **one named configuration per project**: `dev` and `staging`, each with that project's *platform-admin* identity and project set as default.
2. Explore where the projects sit in the hierarchy and record what you find in `~/week1/01-hierarchy.md`.

### Step 1 — Look around

Run `gcloud auth list` and `gcloud config list`. Then open `~/week1/00-environment.md` to see your project IDs and key files.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud auth list                     # the cloud client may already have accounts - we will NOT rely on them
gcloud config configurations list    # only "default" exists
cat ~/week1/00-environment.md
echo $DEV_PROJECT $STAGING_PROJECT
```

A **configuration** is a named set of gcloud properties (account, project, region…). You'll create one per environment, each with its own identity, so you can never run a command against the wrong project by accident.

</details>

### Step 2 — Create the `dev` configuration

Create a configuration called `dev`, authenticate it with the dev platform-admin key, and set the dev project as default.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
# 1. create + activate a new, empty configuration
gcloud config configurations create dev

# 2. authenticate the dev bootstrap identity (sets core/account in the ACTIVE config)
gcloud auth activate-service-account --key-file=/root/.keys/dev-platform-admin.json

# 3. pin account + project for this configuration
gcloud config set account "$DEV_ADMIN_SA"
gcloud config set project "$DEV_PROJECT"

# 4. verify
gcloud config list
```

Your prompt should now show `[gcloud: dev | platform-admin-service-account]`.

</details>

### Step 3 — Create the `staging` configuration

Do the same for staging with the **staging** key and project.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations create staging
gcloud auth activate-service-account --key-file=/root/.keys/staging-platform-admin.json
gcloud config set account "$STAGING_ADMIN_SA"
gcloud config set project "$STAGING_PROJECT"

gcloud config configurations list           # both configs, staging is ACTIVE
gcloud config configurations activate dev   # switch back whenever you need dev
```

Tip: you can also target a config for one command only: `gcloud --configuration=staging projects describe $STAGING_PROJECT`.

</details>

### Step 4 — Explore the hierarchy, billing and project boundaries

Find each project's **number** and **parent**, list its **ancestors**, check its **billing** link, and try to **create a new project**. Write what you observe (including errors!) in `~/week1/01-hierarchy.md`.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev

# ID, number, parent, labels, lifecycle state
gcloud projects describe "$DEV_PROJECT"
gcloud projects describe "$DEV_PROJECT" --format="value(projectNumber,parent.type,parent.id)"

# Walk up the tree: project -> folder(s) -> organization
gcloud projects get-ancestors "$DEV_PROJECT"

# Billing: which billing account pays for this project?
gcloud billing projects describe "$DEV_PROJECT"
#   -> expect an error (API disabled / permission denied). Billing is managed by
#      billing-account admins OUTSIDE your project boundary. That is a finding - write it down.

# Can a platform admin of a project create new projects?
gcloud projects create "northwind-test-$RANDOM"
#   -> PERMISSION_DENIED: resourcemanager.projects.create is granted on a folder/org,
#      not on a project. Your identity only has power INSIDE its own project.

# Repeat the describe/ancestors commands for staging:
gcloud --configuration=staging projects describe "$STAGING_PROJECT" --format="value(projectNumber)"
gcloud --configuration=staging projects get-ancestors "$STAGING_PROJECT"
```

Now open the **Notes** tab → `01-hierarchy.md` and replace every `_TODO_` (both project **numbers** must be in the table).

</details>

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] gcloud configuration `dev` uses the dev platform-admin identity and the dev project
- [ ] gcloud configuration `staging` uses the staging platform-admin identity and the staging project
- [ ] `~/week1/01-hierarchy.md` has both project NUMBERS and no `_TODO_` left
