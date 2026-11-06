---
slug: pipeline-identity
type: challenge
title: A least-privilege deployment pipeline identity
teaser: ci-deployer, app-runtime and a narrowly scoped actAs.
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
# 3.1 A least-privilege deployment pipeline identity

<details>
<summary>📖 Background: workload identities and `actAs` (optional)</summary>

A CI/CD pipeline is a **workload**, so it gets a **service account** — never a human's credentials and ideally never a key (from GitHub Actions, GitLab, etc. you'd use *Workload Identity Federation*; here you'll simulate the pipeline with impersonation).

Deploying to Cloud Run involves **two** identities:

| Identity | Job |
|----------|-----|
| `ci-deployer` | *Deploys* the service (pipeline) |
| `app-runtime` | The identity the *running app* uses to call other APIs |

To deploy a service that runs **as** `app-runtime`, the deployer needs `iam.serviceAccounts.actAs` on `app-runtime` (role *Service Account User*).

> [!WARNING]
> Granting *Service Account User* at **project** level lets the deployer act as **every** service account in the project — including any powerful ones. That's a classic privilege-escalation path. Grant it on the one runtime service account only.

</details>

## Your goal (in the **staging** project)

1. Create service accounts **`ci-deployer`** and **`app-runtime`**.
2. Give `ci-deployer` **only** `roles/run.developer` and `roles/artifactregistry.writer` at project level.
3. Give `ci-deployer` `roles/iam.serviceAccountUser` **on `app-runtime` only**.
4. Let the staging `platform-admin` impersonate `ci-deployer` (simulating the CI system) — on the service account, not the project.
5. Prove it works: as `ci-deployer`, list Cloud Run services; and prove a denial.
6. **No** user-managed keys on `ci-deployer`.

### Step 1 — Create the identities

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud config configurations activate staging

gcloud iam service-accounts create ci-deployer \
  --display-name="CI/CD deployer" --description="Pipeline identity that deploys to staging"

gcloud iam service-accounts create app-runtime \
  --display-name="App runtime" --description="Identity the Cloud Run service runs as"

gcloud iam service-accounts list
```

</details>

### Step 2 — Grant the pipeline its narrow project roles

Before granting, compare `roles/run.developer` with `roles/run.admin` — what can *admin* do that *developer* can't?

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
diff <(gcloud iam roles describe roles/run.developer --format="value(includedPermissions)" | tr ';' '\n') \
     <(gcloud iam roles describe roles/run.admin     --format="value(includedPermissions)" | tr ';' '\n')
#  -> run.admin adds run.services.setIamPolicy (e.g. make a service PUBLIC). The pipeline doesn't need that.

for r in roles/run.developer roles/artifactregistry.writer; do
  gcloud projects add-iam-policy-binding "$STAGING_PROJECT" \
    --member="serviceAccount:$CI_DEPLOYER" --role="$r" --condition=None
done
```

`--condition=None` tells gcloud explicitly that this binding has no IAM Condition.

</details>

### Step 3 — Grant `actAs` on the runtime service account only

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam service-accounts add-iam-policy-binding "$APP_RUNTIME" \
  --member="serviceAccount:$CI_DEPLOYER" \
  --role="roles/iam.serviceAccountUser"
```

</details>

### Step 4 — Let the "CI system" impersonate the deployer

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
gcloud iam service-accounts add-iam-policy-binding "$CI_DEPLOYER" \
  --member="serviceAccount:$STAGING_ADMIN_SA" \
  --role="roles/iam.serviceAccountTokenCreator"
```

In production the member would be a Workload Identity Federation principal such as
`principalSet://iam.googleapis.com/projects/NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/northwind/payments`.

</details>

### Step 5 — Test as `ci-deployer` (wait ~60 s first)

<details>
<summary>💡 Show step-by-step instructions</summary>

```bash
# allowed
gcloud run services list --region="$LAB_REGION" --impersonate-service-account="$CI_DEPLOYER"
gcloud artifacts repositories list --location="$LAB_REGION" --impersonate-service-account="$CI_DEPLOYER"

# denied - the pipeline has no storage or IAM admin rights
gcloud storage buckets list --impersonate-service-account="$CI_DEPLOYER"
gcloud iam service-accounts keys create /tmp/k.json --iam-account="$CI_DEPLOYER" \
  --impersonate-service-account="$CI_DEPLOYER"

# no user-managed keys exist (and none should)
gcloud iam service-accounts keys list --iam-account="$CI_DEPLOYER" --managed-by=user
```

</details>

<details>
<summary>⭐ Optional stretch — deploy a real (private) Cloud Run service as the pipeline</summary>

```bash
gcloud run deploy hello \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --region="$LAB_REGION" \
  --service-account="$APP_RUNTIME" \
  --no-allow-unauthenticated \
  --impersonate-service-account="$CI_DEPLOYER"
```

Now try adding `--allow-unauthenticated` — it fails, because `run.developer` cannot change the service's IAM policy. That is least privilege doing its job. Clean up afterwards with `gcloud run services delete hello --region=$LAB_REGION --impersonate-service-account=$CI_DEPLOYER`.

</details>

## Checklist

Click **Check** when all of these are done (the message tells you which item is still missing):

- [ ] Service accounts `ci-deployer` and `app-runtime` exist in staging
- [ ] `ci-deployer` has roles/run.developer and roles/artifactregistry.writer on the staging project
- [ ] `ci-deployer` has NO broad project roles (owner, editor, run.admin, serviceAccountUser, tokenCreator...)
- [ ] `ci-deployer` has Service Account User ON `app-runtime` only
- [ ] staging platform-admin can impersonate `ci-deployer` (granted on the SA, not the project)
- [ ] `ci-deployer` has NO user-managed keys
- [ ] A gcloud command succeeded while acting as ci-deployer
