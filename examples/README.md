# Pipeline Examples

Self-contained, production-ready examples of the platform container build pipeline for every supported runtime. Each example includes an annotated `azure-pipelines.yml`, a reference `Dockerfile`, and in-depth documentation.

Copy the contents of the relevant example directory into your application repository root and update `tenantName` and `appName`.

---

## Examples

| Directory | Runtime | Build tool | Base image | Notes |
|---|---|---|---|---|
| [go/](go/) | Go | Multi-stage Dockerfile | `gcr.io/distroless/static-debian12:nonroot` | Requires `go.mod`; optional `VERSION` file |
| [python/](python/) | Python | pip + requirements.txt | `python:3.12-slim` | Advisory dep file check; Poetry variant documented |
| [springboot-gradle/](springboot-gradle/) | Spring Boot | Gradle wrapper | `eclipse-temurin:21-jre-alpine` | Requires `gradlew` + `AS test-export` Dockerfile stage |
| [springboot-maven/](springboot-maven/) | Spring Boot | Maven wrapper (mvnw) | `eclipse-temurin:21-jre-alpine` | Requires `pom.xml` + `AS test-export` Dockerfile stage |
| [angular/](angular/) | Angular | Node.js (inside Docker) | `nginx:1.27-alpine` | npm credentials auto-injected; nginx.conf included |
| [react/](react/) | React / Next.js static | Node.js (inside Docker) | `nginx:1.27-alpine` | Vite, CRA, and Next.js `output: 'export'` |
| [react-nextjs-ssr/](react-nextjs-ssr/) | Next.js SSR | Node.js (inside Docker) | `node:22-alpine` | Requires `output: 'standalone'` in next.config.js |

---

## Files in Each Example

| File | Present in |
|---|---|
| `azure-pipelines.yml` | All examples |
| `Dockerfile` | All examples |
| `nginx.conf` | angular/, react/ |
| `README.md` | All examples |

---

## Choosing an Example

```
What is your language?
  ├── Go            → go/
  ├── Python        → python/
  ├── Java          → What is your build tool?
  │     ├── Gradle  → springboot-gradle/
  │     └── Maven   → springboot-maven/
  └── JavaScript / TypeScript → What is your framework?
        ├── Angular   → angular/
        ├── React (Vite, CRA)         → react/
        ├── Next.js with output:export → react/
        └── Next.js SSR / Server Components → react-nextjs-ssr/
```

---

## Quick Start

1. Copy the example directory contents to your repository root
2. Update `tenantName` and `appName` in `azure-pipelines.yml`
3. Customize the `Dockerfile` for your application's entry point
4. Commit and push to trigger the pipeline

For detailed prerequisites and customization guidance, see each example's `README.md` and the top-level [docs/QUICK-START.md](../docs/QUICK-START.md).

---

## What Every Example Produces

All examples produce the same set of pipeline outputs on a successful non-dry-run run:

### ACR image tags

| Tag | Format | Purpose |
|---|---|---|
| Primary | `<40-char-git-sha>` | Use in Kustomize manifests (immutable) |
| Alias | `<branch>-<12-char-sha>` | Human navigation, CI dashboards |
| Version | `<version>` (main) or `<version>-<12-char-sha>` (other branches) | Omitted if no version in project files |

### Cosign objects (in ACR)

| Object | Suffix | Description |
|---|---|---|
| Signature | `<digest>.sig` | Detached signature of manifest digest |
| Attestation | `<digest>.att` | CycloneDX SBOM attached as OCI attestation |

### ADO pipeline artifacts

| Artifact | Contents |
|---|---|
| `sbom-<tenantName>-<appName>` | CycloneDX JSON SBOM of all image packages |
| `provenance-<tenantName>-<appName>` | JSON: digest, tags, SBOM location, git commit, cosign status |

### PR comment

A Markdown summary table is posted to the triggering PR showing: status, tenant/app, runtime, image ref, manifest digest, Cosign status, SBOM artifact name.

### ADO build tag

The manifest digest (`sha256:...`) is set as the ADO pipeline run build tag, correlating the run to the exact image produced.

---

## Common Customization Points

### Non-root build context

If your application is in a subdirectory:

```yaml
parameters:
  tenantName: my-team
  appName: my-service
  runtimeType: go
  buildContext: services/api    # Dockerfile and source root
  dockerfilePath: Dockerfile    # relative to buildContext
```

### Dry run

Validate lint and build without pushing to ACR:

```yaml
parameters:
  tenantName: my-team
  appName: my-service
  runtimeType: go
  dryRun: true
```

### Teams notifications

Add a variable group named `tenant-<tenantName>-notifications` to your ADO project with:

```
TEAMS_WEBHOOK_URL = https://your-org.webhook.office.com/webhookb2/...
```

The Notify stage picks this up automatically. If the variable is absent, Teams notification is silently skipped.
