---
slug: failure-drill
type: challenge
title: 'Failure drill: break it, diagnose it, fix it narrowly'
teaser: Diagnose an access denial and compare broad vs narrow fixes.
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
timelimit: 2400
lab_config:
  default_layout_sidebar_size: 0
enhanced_loading: null
---
# 5. Failure drill — break it, diagnose it, fix it narrowly

<details>
<summary>📖 Background: why this matters (optional)</summary>

"Access denied" is the most common ticket in any cloud team. The wrong reflex is to grant a **broad** role until the error goes away. The right one is to **find the exact missing permission** and grant the **narrowest** thing that contains it.

You diagnose like a **policy troubleshooter** does, by hand:

```text
1. which permission is missing?      -> read the error, then ask the resource: testIamPermissions
2. which bindings apply?             -> bucket policy + project policy (+ folder/org, inherited)
3. why doesn't the binding grant it? -> describe the role: is the permission in it?
```

*(Google's Policy Troubleshooter automates exactly this, but its API is not on this sandbox's allow-list — so you'll do it with the APIs underneath it.)*

</details>

## Your goal

1. **Break:** remove `storage.objects.list` from the `appDeveloper` custom role.
2. **Observe:** as `dev-alice`, listing the bucket fails. Capture the error.
3. **Diagnose:** ask the bucket which permissions `dev-alice` really has (`testIamPermissions`), save the answer to `~/week1/05-permission-test.txt`, and find the binding + role that should have granted it.
4. **Compare:** fix it *broadly* with `roles/storage.admin` on the bucket — then **remove** that grant.
5. **Fix narrowly:** add only `storage.objects.list` back to `appDeveloper`.
6. Document the comparison in `~/week1/05-failure-drill.md`.

### Step 1 — Break it

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev

# baseline: works
gcloud storage ls "gs://$DEV_BUCKET" --impersonate-service-account="$DEV_ALICE"

# remove ONE permission from the custom role
gcloud iam roles update appDeveloper --project="$DEV_PROJECT" --remove-permissions=storage.objects.list
```

Custom-role changes can take **1–7 minutes** to propagate. Retry the next step until it fails.

</details>

### Step 2 — Observe the denial

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud storage ls "gs://$DEV_BUCKET" --impersonate-service-account="$DEV_ALICE"
# -> HTTPError 403: dev-alice@... does not have storage.objects.list access to the Google Cloud Storage bucket.
```

Paste the error into section 2 of `05-failure-drill.md`. Notice the error **names the missing permission** — always read it carefully.

</details>

### Step 3 — Diagnose: which permission is missing, and why?

Ask the bucket — **as dev-alice** — which of the permissions she needs she actually has. Save the answer to `~/week1/05-permission-test.txt`. Then find the binding that applies and check whether its role contains the permission.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev

# 1. testIamPermissions: the resource answers "which of these does the caller have?"
TOKEN=$(gcloud auth print-access-token --impersonate-service-account="$DEV_ALICE")
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://storage.googleapis.com/storage/v1/b/$DEV_BUCKET/iam/testPermissions?permissions=storage.objects.list&permissions=storage.objects.get&permissions=storage.objects.create" \
  | tee ~/week1/05-permission-test.txt
#   -> {"kind": "storage#testIamPermissionsResponse", "permissions": ["storage.objects.get", "storage.objects.create"]}
#      storage.objects.list is MISSING from the answer

# 2. which bindings apply to dev-alice? (bucket, then project)
gcloud storage buckets get-iam-policy "gs://$DEV_BUCKET" --format=json | grep -B2 -A3 dev-alice
gcloud projects get-iam-policy "$DEV_PROJECT" --flatten="bindings[].members" \
  --filter="bindings.members:dev-alice" --format="value(bindings.role)"     # -> nothing

# 3. does the bound role contain the permission?
gcloud iam roles describe appDeveloper --project="$DEV_PROJECT" --format="value(includedPermissions)"
#   -> storage.objects.list is no longer in appDeveloper -> root cause found
```

The verdict, the binding that matched (`appDeveloper` on the bucket) and the reason (role doesn't include `storage.objects.list`) is exactly what a policy troubleshooter reports.

</details>

### Step 4 — The broad fix (and why you'll undo it)

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam roles describe roles/storage.admin --format="value(includedPermissions)" | tr ';' '\n' | wc -l

gcloud storage buckets add-iam-policy-binding "gs://$DEV_BUCKET" \
  --member="serviceAccount:$DEV_ALICE" --role="roles/storage.admin"

# after ~60s: it "works"...
gcloud storage ls "gs://$DEV_BUCKET" --impersonate-service-account="$DEV_ALICE"
# ...but Alice can now also DELETE the bucket's objects, change its IAM policy (= grant anyone access) etc.
gcloud storage buckets get-iam-policy "gs://$DEV_BUCKET" --impersonate-service-account="$DEV_ALICE"

# undo it
gcloud storage buckets remove-iam-policy-binding "gs://$DEV_BUCKET" \
  --member="serviceAccount:$DEV_ALICE" --role="roles/storage.admin"
```

</details>

### Step 5 — The narrow fix

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam roles update appDeveloper --project="$DEV_PROJECT" --add-permissions=storage.objects.list
gcloud iam roles describe appDeveloper --project="$DEV_PROJECT" --format="value(includedPermissions)"

# after propagation:
gcloud storage ls "gs://$DEV_BUCKET" --impersonate-service-account="$DEV_ALICE"

# confirm with testIamPermissions (save separately so the "before" evidence stays)
TOKEN=$(gcloud auth print-access-token --impersonate-service-account="$DEV_ALICE")
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://storage.googleapis.com/storage/v1/b/$DEV_BUCKET/iam/testPermissions?permissions=storage.objects.list" \
  | tee ~/week1/05-permission-test-after.txt     # -> now includes storage.objects.list
```

Fill in `05-failure-drill.md` — including the permission counts for *Broad* (`storage.admin`) vs *Narrow* (`appDeveloper`).

</details>

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] You removed storage.objects.list from appDeveloper
- [ ] After the change, dev-alice was denied
- [ ] `testIamPermissions` answer saved to `~/week1/05-permission-test.txt` WITHOUT storage.objects.list
- [ ] appDeveloper contains storage.objects.list again (exactly 4 permissions)
- [ ] `dev-alice` does NOT hold storage.admin / objectAdmin / editor / owner anywhere in dev
- [ ] `~/week1/05-failure-drill.md` compares broad vs narrow, no `_TODO_` left
