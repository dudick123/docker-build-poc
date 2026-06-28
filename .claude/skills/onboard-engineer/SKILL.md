---
name: tenant-onboarding
description: Guide an operator through configuring and provisioning a new tenant, or diagnose and fix a failing onboarding run.
---

You are assisting a platform operator in this repository. Your role is to help them successfully onboard a new tenant — or diagnose why an existing onboarding run failed.

Start by asking the operator which mode they need:

1. **New tenant** — write a request file and walk through the full provisioning flow
2. **Add apps to an existing tenant** — extend a request file with new apps and re-provision
3. **Diagnose a failure** — investigate why a previous run failed and determine the fix

Then follow the appropriate path below.

---

## Mode 1: New tenant

### Step 1 — Gather requirements

Ask the operator for the following. Do not proceed until you have all required fields:

**Required:**
- `tenant_slug` — lowercase, letters/digits/hyphens only, 2–29 chars, must start with a letter (e.g. `acme`, `payments-api`)
- `tenant_display_name` — human-readable name (e.g. "ACME Industries")
- `ado_org_name` — the subdomain in `dev.azure.com/{org}`
- `ado_project` — the ADO project (default: `platform-tenants`)
- `resource_tier` — `small`, `medium`, or `large`
- `environments` — ordered list (e.g. `[dev, staging, prod]`); first env gets no approval gate, subsequent envs require ADO environment approval
- `apps` — list of apps; for each: `name` (slug), `type` (one of: `angular`, `react`, `springboot`, `go`, `python`, `dotnet`), `container_image` (full registry path without tag)
- `key_vault_name` — Azure Key Vault backing this tenant's secrets
- `argocd_target_revision` — git branch ArgoCD should track (default: `main`)
- `include_app_gateway` — `true` or `false`

**Required only if `include_app_gateway: true`:**
- `app_gateway_backend_fqdn` — backend FQDN the listener routes to
- `app_gateway_hostname` — public hostname (e.g. `tenant.platform.example.com`)
- `app_gateway_priority` — unique integer 1–20000 on the shared gateway

### Step 2 — Validate inputs

Before writing anything, check:

- `tenant_slug` matches `^[a-z][a-z0-9-]{1,28}$` — reject uppercase, spaces, leading digits, or hyphens at start
- Each app `name` follows the same pattern and is unique within the tenant
- Each app `type` is one of the six supported values
- Each app `container_image` looks like a fully-qualified registry path (contains at least one `/` and no tag suffix)
- `app_gateway_priority` is an integer and has not been used by another tenant (ask the operator to confirm)
- No two apps share the same `container_image`

Flag any issues and ask the operator to resolve them before continuing.

### Step 3 — Write the request file

Create `onboarding-requests/{tenant_slug}-input.yaml` with all collected values. Follow this structure exactly:

```yaml
# Tenant onboarding request — committed to git for review before generation
tenant_slug: {tenant_slug}
tenant_display_name: "{tenant_display_name}"
ado_project: {ado_project}
ado_org_name: {ado_org_name}
resource_tier: {resource_tier}
environments:
  - dev
  - staging     # include only the environments the operator specified
  - prod

apps:
  - name: {app_name}
    type: {app_type}
    container_image: {registry}/{tenant}/{app_name}

key_vault_name: {key_vault_name}
argocd_target_revision: main
include_app_gateway: {true|false}
# app_gateway_* fields only if include_app_gateway: true
app_gateway_backend_fqdn: {fqdn}
app_gateway_hostname: {hostname}
app_gateway_priority: {priority}
```

Do not include `provisioned_components` in a new request file — the field is absent until the first `provision_tenant.py` run.

### Step 4 — Confirm before generating

Show the operator the complete request file content and ask them to confirm it is correct before proceeding. This file will be committed to git and reviewed via PR — it is the change-control record for what gets provisioned.

### Step 5 — Generate the tenant

Once confirmed, provide the exact commands to run:

```bash
# Make the template a git repo if not already (one-time, local only)
cd onboarding-template && git init && git add -A && git commit -m "template" && cd ..

# Generate the tenant directory
uv run copier copy onboarding-template generated-tenants/{tenant_slug} \
  --data-file onboarding-requests/{tenant_slug}-input.yaml \
  --defaults --quiet
```

Then inspect the output:

```bash
find generated-tenants/{tenant_slug} -type f | sort
cat generated-tenants/{tenant_slug}/provisioning.yaml
```

Expected: one directory per app under `apps/` (each with `build-repo/`, `config-repo/`, `argocd-app/`), plus `app-gateway/` if enabled, plus `provisioning.yaml`.

Ask the operator to confirm the file tree looks correct before proceeding.

### Step 6 — Provision to Azure DevOps

```bash
export ADO_PAT=your-personal-access-token

uv run python provision_tenant.py \
  --tenant-dir ./generated-tenants/{tenant_slug} \
  --request-file ./onboarding-requests/{tenant_slug}-input.yaml \
  --ado-org https://dev.azure.com/{ado_org_name} \
  --pat $ADO_PAT
```

**PAT must have these scopes:** Code (Read & Write), Build (Read & Execute), Environment (Read & Manage), Graph (Read).

After a successful run, `provision_tenant.py` writes `provisioned_components` back into the request file. Ask the operator to commit this update.

### Step 7 — Post-provisioning checklist

Walk the operator through these manual steps that the script does not automate:

- [ ] **ADO service connections** — the pipelines assume ACR and Azure subscription service connections already exist in the project. Verify they are named as expected.
- [ ] **ADO environment approval checks** — `provision_tenant.py` creates ADO environments with approval requirements, but the specific approvers must be configured in the ADO portal under **Pipelines → Environments → {env-name} → Approvals and checks**.
- [ ] **`CONTAINER_IMAGE` pipeline variable** — in each build pipeline (`{app_name}`), set `CONTAINER_IMAGE` to the full registry path (e.g. `myacr.azurecr.io/tenant/app-name`) under **Pipeline → Edit → Variables**.
- [ ] **`argocd-servers` variable group** — the config and argocd-app pipelines read ArgoCD server FQDNs and tokens from a variable group named `argocd-servers`. Verify it exists in the ADO project with the correct `ARGOCD_SERVER_{ENV}` and `ARGOCD_TOKEN_{ENV}` entries for each environment.
- [ ] **ArgoCD Application CRD** — `{app_name}-argocd/{env}/application.yaml` has been pushed to ADO but not applied to the cluster. Apply it to the appropriate ArgoCD control plane: `kubectl apply -f {env}/application.yaml` (against that env's cluster), or use the `{app_name}-argocd` sync pipeline after configuring approvals.

---

## Mode 2: Add apps to an existing tenant

Read the existing request file at `onboarding-requests/{tenant_slug}-input.yaml`.

1. Identify the current `apps:` list and `provisioned_components` values.
2. Ask the operator for the new apps to add (name, type, container_image).
3. Validate the new app inputs (same checks as Mode 1 Step 2).
4. Edit the request file — add the new apps to the `apps:` list. Add a comment noting the phase/PR number.
5. Use `copier update` (not `copier copy`) to regenerate:

```bash
uv run copier update \
  --data-file onboarding-requests/{tenant_slug}-input.yaml \
  --defaults --quiet \
  generated-tenants/{tenant_slug}
```

6. Verify the new `apps/{new_app_name}/` directories were created; existing app directories should be unchanged.
7. Run `provision_tenant.py` — it will only provision the new delta components.

---

## Mode 3: Diagnose a failure

Ask the operator to share:
1. The full error output from `copier copy`/`copier update` or `provision_tenant.py`
2. The content of `onboarding-requests/{tenant_slug}-input.yaml`
3. The content of `generated-tenants/{tenant_slug}/provisioning.yaml` (if generation succeeded)

Then work through the following diagnostic tree:

### Generation failures (`copier copy` / `copier update`)

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `tenant_slug` validation error | Slug fails `^[a-z][a-z0-9-]{1,28}$` | Correct the slug value |
| `_generate.py` exits non-zero | Python error in the post-generation task | Read the full traceback; check `apps` list is valid YAML |
| `exemplar-apps/app-{type}` not found | `type` value is not one of the 6 supported types | Correct the type; supported: `angular`, `react`, `springboot`, `go`, `python`, `dotnet` |
| `copier update` fails with "no answers file" | `.copier-answers.yml` missing from tenant directory | Re-run `copier copy` from scratch; see `docs/ITERATIVE_ONBOARDING.md` recovery procedure |
| Template not a git repo | `copier` cannot resolve template version | `cd onboarding-template && git init && git add -A && git commit -m "init"` |
| Generated `provisioning.yaml` is missing or empty | `_generate.py` task did not run | Check `copier.yml` `_tasks` entry; confirm Python 3.9+ is on PATH |

### Provisioning failures (`provision_tenant.py`)

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `401 Unauthorized` | PAT is expired, wrong, or missing required scopes | Regenerate PAT with Code+Build+Environment+Graph scopes |
| `404 Project not found` | `ado_project` or `ado_org_name` is wrong | Verify values in request file; check org URL format `https://dev.azure.com/{org}` |
| Repo creation succeeds but push fails with 422 | Repo was created in a previous partial run; `push_directory` expects an empty repo | Manually delete the partially-created ADO repo and re-run |
| Pipeline creation fails with "service connection not found" | Named service connection does not exist in the ADO project | Create the missing service connection in ADO, then re-run |
| `KeyError` on `provisioning.yaml` field | `provisioning.yaml` format changed or was hand-edited | Regenerate the tenant directory with `copier copy` and inspect `provisioning.yaml` |
| Component provisioned but missing from `provisioned_components` | Script exited non-zero before writing back | Check for partial success output; re-run — succeeded components are skipped |
| `copier update` did not add new app directory | Request file `apps:` list was not updated before running update | Edit `apps:` list to include the new app, then re-run `copier update` |

### Pipeline failures (post-provisioning, in ADO)

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Build pipeline fails at Docker push | `CONTAINER_IMAGE` variable not set, or service connection missing | Set the variable in Pipeline → Variables; verify ACR service connection |
| Config pipeline not triggered after build | Pipeline resource trigger source name mismatch | In `{app}-k8s-manifests/azure-pipelines.yml`, `resources.pipelines.source` must exactly match the build pipeline name |
| Config pipeline fails at ArgoCD sync | `argocd-servers` variable group missing or has wrong keys | Create variable group `argocd-servers` with `ARGOCD_SERVER_{ENV}` and `ARGOCD_TOKEN_{ENV}` per environment |
| ArgoCD sync stage waits forever | ADO environment approval check not configured | Configure approvers in ADO portal: Pipelines → Environments → {env-name} → Approvals and checks |
| `kustomize build` fails in config pipeline | Bad manifest added to base without registering in `kustomization.yaml` | Check `config-repo/{app}/base/kustomization.yaml` resources list |

### Validation commands

Use these to confirm the state of a generated tenant before provisioning:

```bash
# Confirm all expected directories exist
find generated-tenants/{tenant_slug}/apps -maxdepth 2 -type d | sort

# Confirm provisioning.yaml matches expected app list
grep "repo_name:" generated-tenants/{tenant_slug}/provisioning.yaml

# Confirm Kustomize overlays are valid (requires kustomize CLI)
for env in dev staging prod; do
  echo "=== $env ===" && \
  kustomize build generated-tenants/{tenant_slug}/apps/{app_name}/config-repo/{app_name}/overlays/$env
done

# Check what's already provisioned vs what's pending
python3 - <<'EOF'
import yaml
with open("onboarding-requests/{tenant_slug}-input.yaml") as f:
    req = yaml.safe_load(f)
with open("generated-tenants/{tenant_slug}/provisioning.yaml") as f:
    prov = yaml.safe_load(f)
done = set(req.get("provisioned_components", []))
pending = [c for c in prov["push_order"] if c not in done]
print("Provisioned:", sorted(done))
print("Pending:    ", pending)
EOF
```
