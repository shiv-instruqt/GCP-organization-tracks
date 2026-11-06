---
slug: org-policies-boundaries-quotas
type: challenge
title: Org policies, project boundaries & quotas
teaser: The guard-rails above IAM.
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
# 4. Org policies, project boundaries & quotas

<details>
<summary>📖 Background: two different control planes (optional)</summary>

| | IAM allow policy | Organization policy |
|--|------------------|---------------------|
| Answers | **Who** can do **what** | **What is allowed at all**, no matter who asks |
| Example | "ci-deployer may deploy Cloud Run" | "Nobody may create service-account keys", "Resources only in `us-central1`" |
| Set by | Anyone with `setIamPolicy` on the resource | *Organization Policy Administrator* (org level) |
| Inherited | Yes (additive) | Yes (can merge, override or be enforced from above) |

Even an Owner of a project **cannot** override an org policy enforced on the folder or organization. That is the guard-rail layer of a secure foundation.

**Quotas** are per project (and per region/API). They protect you from runaway cost and Google from abuse. Raising one is a request, not a command.

</details>

## Your goal

1. Read the **effective** org policies on your projects and record at least **two** constraints, their effective value and their impact.
2. Prove the **API guard-rail**: try to enable an API that is not allowed (it must be refused).
3. Prove a **project boundary**: as `ci-deployer` (staging), try to read the **dev** bucket → denied.
4. Try to **change** an org policy as `platform-admin` → denied (and explain why).
5. Look up at least one **quota** (service, metric, limit) for an API you enabled.
6. Record everything in `~/week1/04-org-policy-quotas.md`.

### Step 1 — Read effective org policies

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev

# policies SET on the project (uses the Resource Manager API - already enabled)
gcloud resource-manager org-policies list --project="$DEV_PROJECT"

# the EFFECTIVE value after inheritance - pick constraints that matter for identity:
for c in iam.disableServiceAccountKeyCreation \
         iam.allowedPolicyMemberDomains \
         gcp.resourceLocations \
         gcp.restrictServiceUsage \
         iam.automaticIamGrantsForDefaultServiceAccounts \
         storage.uniformBucketLevelAccess; do
  echo "=== $c"; gcloud resource-manager org-policies describe "$c" --project="$DEV_PROJECT" --effective
done
```

If a command is denied, that is a finding too — write down which permission was missing.

Questions to answer in your notes:
- Which constraint explains why you could only use **certain APIs** and **regions** in this sandbox?
- If `iam.disableServiceAccountKeyCreation` were enforced, which step of this lab would break? (Hint: the bootstrap!)

</details>

### Step 2 — Hit the API guard-rail

Try to enable an API that is **not** on this sandbox's allow-list, e.g. `compute.googleapis.com`. It must be refused — note which constraint refused it.

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate dev
gcloud services enable compute.googleapis.com
#   -> FAILED_PRECONDITION / PERMISSION_DENIED ... constraints/gcp.restrictServiceUsage
```

Even though your identity has *Service Usage Admin*, an org policy above the project decides which services may exist at all. IAM says **who**, org policy says **what is allowed**.

</details>

### Step 3 — Test the location guard-rail (optional but fun)

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud storage buckets create "gs://$DEV_PROJECT-tokyo" --location=asia-northeast1
```

If an org policy restricts locations you'll get a `constraints/gcp.resourceLocations` violation. If it *does* get created, delete it right away: `gcloud storage rm -r "gs://$DEV_PROJECT-tokyo"` — and write down that there is no location guard-rail.

</details>

### Step 4 — Prove the project boundary

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate staging

# ci-deployer is a staging identity - it has no access to dev data
gcloud storage ls "gs://$DEV_BUCKET" --impersonate-service-account="$CI_DEPLOYER"
#   -> 403 ... does not have storage.objects.list access
```

A service account is *owned* by one project, but access is decided by the **IAM policy of the resource** being accessed. Nothing in dev grants ci-deployer anything.

</details>

### Step 5 — Try to change an org policy

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud resource-manager org-policies disable-enforce iam.disableServiceAccountKeyCreation --project="$STAGING_PROJECT"
#   -> PERMISSION_DENIED: orgpolicy.policy.set
```

*Organization Policy Administrator* can only be granted at the **organization** level — by design, project admins can't loosen guard-rails.

</details>

### Step 6 — Look up a quota

<details>
<summary>💡 Show step-by-step instructions</summary>

Via the Service Usage API (the identity's access token is passed in the header):

```bash
TOKEN=$(gcloud auth print-access-token)
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://serviceusage.googleapis.com/v1beta1/projects/$STAGING_PROJECT/services/run.googleapis.com/consumerQuotaMetrics" \
  | jq -r '.metrics[] | .displayName as $d | .consumerQuotaLimits[] | [$d, .unit, (.quotaBuckets[0].effectiveLimit // "n/a")] | @tsv' \
  | head -15
```

(No `jq`? Replace the `jq …` line with `python3 -m json.tool | less`.)

Or in the console (*Cloud Consoles* tab): **IAM & Admin → Quotas & System Limits**, filter by service.

</details>

Now complete `~/week1/04-org-policy-quotas.md`.

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] You inspected org policies with `gcloud resource-manager org-policies`
- [ ] Enabling a non-allowed API was refused (API guard-rail)
- [ ] As ci-deployer you tried to read the dev bucket and were denied
- [ ] `~/week1/04-org-policy-quotas.md` names constraints and a quota, no `_TODO_` left
