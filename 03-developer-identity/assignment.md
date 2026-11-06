---
slug: developer-identity
id: m9hdwl8puvw0
type: challenge
title: Predefined vs custom roles - the developer identity
teaser: Create dev-alice with a custom role granted on one bucket only.
tabs:
- id: pllham4c6xwv
  title: Terminal
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: vkzgxvjeojzj
  title: Terminal 2
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: uai0a7stubxw
  title: Notes
  type: code
  hostname: cloud-client
  path: /root/week1
- id: fl0kd1hz447r
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
# 2.1 Predefined vs custom roles — the developer identity

<details>
<summary>📖 Background: who, what, where (optional)</summary>

An **IAM allow policy** is attached to a resource (org, folder, project, bucket, service account…) and contains **bindings**:

```text
binding = principal (WHO)  +  role (WHAT: a bundle of permissions)  [+ condition (WHEN/WHICH)]
attached to a resource      (WHERE - and everything below it inherits it)
```

| Role type | Example | Use it when… |
|-----------|---------|--------------|
| **Basic** | `roles/owner`, `roles/editor`, `roles/viewer` | Almost never in production — thousands of permissions across every service |
| **Predefined** | `roles/storage.objectViewer` | A Google-curated role matches the job well |
| **Custom** | `projects/P/roles/appDeveloper` | No predefined role is narrow enough — you list the exact permissions |

**Least privilege = smallest role × smallest scope.** A narrow role granted on a single bucket beats the same role granted on the whole project.

</details>

## Your goal (in the **dev** project)

Build the identity for *Alice, a developer*:

1. A service account **`dev-alice`** (stands in for Alice's user account — in real life you'd bind her Google user or group).
2. A bucket **`gs://<dev-project-id>-app-data`** (`$DEV_BUCKET`) in `us-central1` with uniform bucket-level access.
3. A custom role **`appDeveloper`** containing **exactly** these permissions:
   `storage.buckets.get`, `storage.objects.get`, `storage.objects.list`, `storage.objects.create`
   *(read + upload; no delete, no overwrite of IAM, no bucket admin)*
4. Grant `appDeveloper` to `dev-alice` **on the bucket only** — *no* project-level roles for `dev-alice`.

### Step 1 — Read the current IAM policy and compare roles

Look at the dev project's IAM policy. Then compare how many permissions are in `roles/storage.objectViewer`, `roles/storage.objectAdmin`, `roles/storage.admin` and `roles/editor`.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev

# who has what on the project?
gcloud projects get-iam-policy "$DEV_PROJECT" \
  --flatten="bindings[].members" --format="table(bindings.role, bindings.members)"

# what is inside a predefined role?
gcloud iam roles describe roles/storage.objectViewer

# count permissions per role
for r in roles/storage.objectViewer roles/storage.objectAdmin roles/storage.admin roles/editor; do
  n=$(gcloud iam roles describe "$r" --format="value(includedPermissions)" | tr ';' '\n' | wc -l)
  echo "$r -> $n permissions"
done
```

Keep these numbers — you'll need them in the failure drill and the least-privilege table.

</details>

### Step 2 — Create the developer service account and the bucket

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam service-accounts create dev-alice \
  --display-name="Developer: Alice" \
  --description="Stand-in for a human developer in dev"

gcloud storage buckets create "gs://$DEV_BUCKET" \
  --location="$LAB_REGION" \
  --uniform-bucket-level-access

echo "$DEV_ALICE"     # dev-alice@<dev-project>.iam.gserviceaccount.com
```

Uniform bucket-level access turns off per-object ACLs so **only IAM** controls access — required for clean least-privilege and for IAM Conditions on buckets.

</details>

### Step 3 — Create the custom role

No predefined role gives "read + upload, but never delete". Create one.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam roles create appDeveloper \
  --project="$DEV_PROJECT" \
  --title="App Developer (dev data)" \
  --description="List/read/upload app objects. No delete, no IAM, no bucket admin." \
  --permissions=storage.buckets.get,storage.objects.get,storage.objects.list,storage.objects.create \
  --stage=GA

gcloud iam roles describe appDeveloper --project="$DEV_PROJECT"
```

Custom roles created in a project can be granted on that project **or any resource inside it** (like your bucket). Its full name is `projects/$DEV_PROJECT/roles/appDeveloper`.

</details>

### Step 4 — Grant the role on the bucket (not the project)

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud storage buckets add-iam-policy-binding "gs://$DEV_BUCKET" \
  --member="serviceAccount:$DEV_ALICE" \
  --role="projects/$DEV_PROJECT/roles/appDeveloper"

# bucket-level policy
gcloud storage buckets get-iam-policy "gs://$DEV_BUCKET"

# confirm dev-alice has NOTHING at project level (should print nothing)
gcloud projects get-iam-policy "$DEV_PROJECT" --flatten="bindings[].members" \
  --filter="bindings.members:dev-alice" --format="value(bindings.role)"
```

**Inheritance in action:** a role on the *project* would flow down to *every* bucket in it. Granting on the bucket limits Alice to this one bucket.

</details>

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] Service account `dev-alice` exists in dev
- [ ] Bucket `<dev-project>-app-data` exists
- [ ] Custom role `appDeveloper` has exactly: storage.buckets.get, storage.objects.get, storage.objects.list, storage.objects.create
- [ ] `dev-alice` holds `appDeveloper` on the bucket
- [ ] `dev-alice` has NO project-level roles in dev
