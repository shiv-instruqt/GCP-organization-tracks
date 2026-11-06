---
slug: enable-apis
id: gxedg1f5lbo2
type: challenge
title: Enable the required APIs with gcloud
teaser: Enable APIs per project and document which identity did it.
tabs:
- id: hn3p7opehj7i
  title: Terminal
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: difefwvl8eeq
  title: Terminal 2
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: 8x6misuuqkmc
  title: Notes
  type: code
  hostname: cloud-client
  path: /root/week1
- id: r5c2flhffklk
  title: Cloud Consoles
  type: service
  hostname: cloud-client
  path: /
  port: 80
difficulty: intermediate
timelimit: 1800
lab_config:
  default_layout_sidebar_size: 0
enhanced_loading: null
---
# 1.2 Enable the required APIs with gcloud

<details>
<summary>📖 Background: APIs are per project (optional)</summary>

Every Google Cloud service is an API that must be **enabled in a project** before anything in that project can use it. Enabling an API:

- is done by an identity with `serviceusage.services.enable` (e.g. *Service Usage Admin*),
- can create **service agents** (Google-managed service accounts like `service-PROJECT_NUMBER@gcp-sa-...`),
- makes the API's **quotas** apply to that project,
- is itself an auditable action — so you must always know **which identity** enabled what.

> [!NOTE]
> Calls are charged to a **quota project**. When you impersonate a service account, the IAM Service Account Credentials API must be enabled in the project of the *calling* credentials — that's why it is needed in **both** projects.

</details>

## Your goal

Enable these APIs — and nothing you don't need (every enabled API is attack surface and quota you have to own):

| Project | API | Needed for |
|---------|-----|-----------|
| dev | `iamcredentials.googleapis.com` | Service-account impersonation (short-lived tokens) — acting as `dev-alice` |
| staging | `iamcredentials.googleapis.com` | Impersonating the pipeline identity `ci-deployer` |
| staging | `artifactregistry.googleapis.com` | Pipeline pushes images |
| staging | `run.googleapis.com` | Pipeline deploys to Cloud Run |

> [!NOTE]
> This sandbox only allows a fixed list of APIs (an org policy you'll inspect in challenge 7). Anything else — e.g. `compute.googleapis.com` — will be refused.

Then document every row in `~/week1/02-api-enablement.md`: API, project, why, **which identity** enabled it and the command.

### Step 1 — See what is (and isn't) enabled

List enabled APIs in both projects. Search the catalog for the ones you need.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud --configuration=dev     services list --enabled
gcloud --configuration=staging services list --enabled

# search the catalog of available APIs
gcloud --configuration=dev services list --available --filter="name:run.googleapis.com"
```

Notice a few APIs were already on (Resource Manager, IAM, Service Usage, Cloud Storage). The platform team needed those to vend the project — record them in the "already enabled" section of your notes.

</details>

### Step 2 — Enable the dev APIs

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev
gcloud config get-value account        # <- the identity that will perform the change

gcloud services enable iamcredentials.googleapis.com
```

`services enable` is a long-running operation; it returns when done (usually < 1 min).

</details>

### Step 3 — Enable the staging APIs

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate staging
gcloud config get-value account

gcloud services enable \
  iamcredentials.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com

gcloud services list --enabled --format="value(config.name)" | sort
```

</details>

### Step 4 — Prove who did it

Look at `~/week1/commands.log` — each line shows the configuration, the account and the exit code. Use it to fill `~/week1/02-api-enablement.md`.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
grep "services enable" ~/week1/commands.log
```

Optional (Cloud Audit Logs are the authoritative record, may take a few minutes to appear):

```bash
gcloud --configuration=dev logging read \
  'protoPayload.methodName="google.api.serviceusage.v1.ServiceUsage.EnableService"' \
  --limit=5 --format="table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.resourceName)"
```

(If `logging read` is denied, that's expected — your identity doesn't hold *Logs Viewer*. Note it as a least-privilege observation.)

Fill the table: one row per API + project. The *Enabled by* column must contain the platform-admin email.

</details>

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] dev: iamcredentials API is enabled
- [ ] staging: iamcredentials, artifactregistry and run APIs are enabled
- [ ] `~/week1/02-api-enablement.md` lists every API and the identity that enabled it
