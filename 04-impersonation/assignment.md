---
slug: impersonation
type: challenge
title: Act as the developer with impersonation
teaser: Short-lived tokens instead of keys.
tabs:
- title: Terminal
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- title: Terminal 2
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- title: Notes
  type: code
  hostname: cloud-client
  path: /root/week1
- title: Cloud Consoles
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
# 2.2 Act as the developer with impersonation

<details>
<summary>📖 Background: service account impersonation (optional)</summary>

Instead of downloading a key for `dev-alice`, a trusted principal can **impersonate** it: it asks the *IAM Service Account Credentials API* for a **short-lived token** (1 hour by default).

```text
platform-admin ──(roles/iam.serviceAccountTokenCreator ON dev-alice)──► token for dev-alice ──► Cloud Storage
```

- The permission is `iam.serviceAccounts.getAccessToken`, granted by **Service Account Token Creator**.
- Grant it **on the service account resource**, not on the project — otherwise the grantee can impersonate *every* service account in the project.
- Every impersonated call is audit-logged with **both** identities (the caller and the service account).
- No keys to leak, rotate or forget. This is how humans and CI systems should "become" a workload identity.

</details>

## Your goal

1. Allow **only** the dev `platform-admin` to impersonate `dev-alice` — grant on the `dev-alice` service account, not the project.
2. As `dev-alice`, upload `hello.txt` to `$DEV_BUCKET` and list the bucket.
3. As `dev-alice`, try at least one thing the role does **not** allow and confirm it is denied.
4. Prove **inheritance**: a role granted on the *project* reaches a bucket that had no grant — then remove it again.

### Step 1 — Try impersonating before you are allowed to

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev
gcloud storage ls "gs://$DEV_BUCKET" --impersonate-service-account="$DEV_ALICE"
```

Expected error: `Permission 'iam.serviceAccounts.getAccessToken' denied`. Being *Service Account Admin* lets you manage service accounts — it does **not** let you act as them. That separation is deliberate.

</details>

### Step 2 — Grant Token Creator on the service account

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam service-accounts add-iam-policy-binding "$DEV_ALICE" \
  --member="serviceAccount:$DEV_ADMIN_SA" \
  --role="roles/iam.serviceAccountTokenCreator"

gcloud iam service-accounts get-iam-policy "$DEV_ALICE"
```

Wait ~60 seconds for the grant to propagate.

</details>

### Step 3 — Work as `dev-alice`

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
echo "hello from alice" > ~/week1/hello.txt

# allowed: upload (objects.create) and list (objects.list)
gcloud storage cp ~/week1/hello.txt "gs://$DEV_BUCKET/hello.txt" --impersonate-service-account="$DEV_ALICE"
gcloud storage ls "gs://$DEV_BUCKET"                              --impersonate-service-account="$DEV_ALICE"
gcloud storage cat "gs://$DEV_BUCKET/hello.txt"                   --impersonate-service-account="$DEV_ALICE"
```

The yellow `WARNING: This command is using service account impersonation` line is expected.

</details>

### Step 4 — Prove what `dev-alice` can NOT do

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
# delete -> denied (no storage.objects.delete)
gcloud storage rm "gs://$DEV_BUCKET/hello.txt" --impersonate-service-account="$DEV_ALICE"

# overwrite the same object -> denied (overwriting needs storage.objects.delete too!)
gcloud storage cp ~/week1/hello.txt "gs://$DEV_BUCKET/hello.txt" --impersonate-service-account="$DEV_ALICE"

# list buckets in the project -> denied (no project-level role at all)
gcloud storage buckets list --project="$DEV_PROJECT" --impersonate-service-account="$DEV_ALICE"
```

Check `~/week1/commands.log`: each line shows `acting-as=dev-alice@…` and the exit code. That's your audit trail for the identity map.

</details>

### Step 5 — See IAM inheritance in action

Create a second bucket `gs://<dev-project-id>-reports` (`$DEV_PROJECT-reports`). As `dev-alice`, try to list it (denied — her role is only on the app-data bucket). Then grant her `roles/storage.objectViewer` on the **project**, list it again (allowed — the project grant is **inherited** by every bucket), and finally **remove** the project-level grant again.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev
gcloud storage buckets create "gs://$DEV_PROJECT-reports" --location="$LAB_REGION" --uniform-bucket-level-access

# 1. denied: dev-alice's role is granted on the OTHER bucket only
gcloud storage ls "gs://$DEV_PROJECT-reports" --impersonate-service-account="$DEV_ALICE"

# 2. grant on the PROJECT -> inherited by every bucket in it
gcloud projects add-iam-policy-binding "$DEV_PROJECT" \
  --member="serviceAccount:$DEV_ALICE" --role="roles/storage.objectViewer" --condition=None

# 3. after ~60 s: allowed
gcloud storage ls "gs://$DEV_PROJECT-reports" --impersonate-service-account="$DEV_ALICE"

# 4. clean up - dev-alice must have NO project-level roles
gcloud projects remove-iam-policy-binding "$DEV_PROJECT" \
  --member="serviceAccount:$DEV_ALICE" --role="roles/storage.objectViewer" --all
```

One grant high up = access to **everything below it**, including buckets created later. That's why least privilege means granting at the **lowest** level that works.

</details>

> [!NOTE]
> To impersonate for a whole session: `gcloud config set auth/impersonate_service_account $DEV_ALICE` — and **always** `gcloud config unset auth/impersonate_service_account` afterwards.

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] dev platform-admin has Token Creator ON the dev-alice service account (not on the project)
- [ ] `hello.txt` exists in the dev bucket
- [ ] A gcloud command succeeded while acting as dev-alice
- [ ] You observed at least one DENIED action as dev-alice
- [ ] As dev-alice, listing the `-reports` bucket was DENIED (no grant on it yet)
- [ ] After a PROJECT-level grant, dev-alice listed the `-reports` bucket successfully (inheritance)
- [ ] The project-level grant was removed again - dev-alice has no project roles
