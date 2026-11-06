---
slug: iam-conditions
id: 6eugdlxvamky
type: challenge
title: Time-bound cross-project access with IAM Conditions
teaser: Access that expires by itself.
tabs:
- id: hmgowm5c6xio
  title: Terminal
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: uxelduy8bqav
  title: Terminal 2
  type: terminal
  hostname: cloud-client
  workdir: /root/week1
- id: dyezcbgnxu0i
  title: Notes
  type: code
  hostname: cloud-client
  path: /root/week1
- id: 5y2235ojkq5k
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
# 3.2 Time-bound cross-project access with IAM Conditions

<details>
<summary>📖 Background: IAM Conditions (optional)</summary>

A binding can carry a **condition** written in CEL. The role only applies while the condition is `true`.

| Attribute | Example | Use case |
|-----------|---------|----------|
| `request.time` | `request.time < timestamp("2026-12-01T00:00:00Z")` | Temporary / break-glass access that **expires by itself** |
| `resource.name` | `resource.name.startsWith("projects/_/buckets/reports")` | Limit a project-level grant to some resources |
| `resource.type` | `resource.type == "storage.googleapis.com/Bucket"` | Limit to one kind of resource |

Rules worth remembering:

- Policies with conditions use **policy version 3**.
- Conditions do **not** work with basic roles (owner/editor/viewer).
- A condition can only *restrict* the binding it's on — it never removes access granted elsewhere.

**Project boundaries vs identities:** a service account *lives* in one project but can be granted access in **any** project. `dev-alice` (from dev) has no access to staging until someone with IAM admin rights in **staging** grants it.

</details>

## Your goal

QA asked that Alice can **read objects in staging for one week** while she verifies a release — and that the access must disappear automatically.

In the **staging** project, grant `dev-alice` (from **dev**) `roles/storage.objectViewer` **with a condition** that expires **at most 30 days** from now (7 days is the request). Give the condition a title. Alice must have **no unconditional** access in staging.

### Step 1 — Compute an expiry timestamp

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
EXPIRY=$(date -u -d "@$(( $(date +%s) + 7*24*3600 ))" +%Y-%m-%dT%H:%M:%SZ)
echo "$EXPIRY"
```

(`date -d @EPOCH` works in both BusyBox and GNU date.)

</details>

### Step 2 — Add the conditional binding (as the **staging** platform admin)

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate staging

gcloud projects add-iam-policy-binding "$STAGING_PROJECT" \
  --member="serviceAccount:$DEV_ALICE" \
  --role="roles/storage.objectViewer" \
  --condition="expression=request.time < timestamp('$EXPIRY'),title=week1-temporary-access,description=QA verification - expires automatically"
```

Note: the **staging** admin performs this grant. The dev admin couldn't — try it:
`gcloud --configuration=dev projects add-iam-policy-binding "$STAGING_PROJECT" --member="serviceAccount:$DEV_ALICE" --role=roles/storage.objectViewer --condition=None` → `PERMISSION_DENIED`. Project boundary!

</details>

### Step 3 — Inspect the version-3 policy

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud projects get-iam-policy "$STAGING_PROJECT" --format=json | grep -B3 -A8 dev-alice
gcloud projects get-iam-policy "$STAGING_PROJECT" --format="value(version)"   # -> 3
```


</details>

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] `dev-alice` has roles/storage.objectViewer in staging WITH a request.time condition
- [ ] The access expires in the future and within 30 days
- [ ] `dev-alice` has NO unconditional access in staging
