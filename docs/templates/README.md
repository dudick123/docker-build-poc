# Template Reference

Individual reference documents for every template file in `platform-templates/`. Each document covers description, parameters, steps, output variables, and usage.

---

## Base Template

| File | Document | Description |
|---|---|---|
| `container-build-v2.yml` | [container-build-v2.md](container-build-v2.md) | Tenant entry point. Defines all five stages, cross-stage wiring, `dryRun` conditions, and runtime dispatch. |

---

## Step Templates (`steps/`)

| File | Document | Description |
|---|---|---|
| `steps/setup.yml` | [steps/setup.md](steps/setup.md) | Tool-version resolution from `platform-tool-versions` variable group + six-parameter validation. Runs in Stage 1. |
| `steps/dockerfile-lint.yml` | [steps/dockerfile-lint.md](steps/dockerfile-lint.md) | Downloads Hadolint and lints the tenant Dockerfile. ERROR findings fail the build. Runs first in Stage 2. |
| `steps/docker-build.yml` | [steps/docker-build.md](steps/docker-build.md) | BuildKit image build with OCI labels, ACR layer cache, and npm credential injection. Image held locally. Runs second in Stage 2. |
| `steps/sbom-sign-publish.yml` | [steps/sbom-sign-publish.md](steps/sbom-sign-publish.md) | Multi-phase template covering SBOM generation + Cosign signing (Stage 3), ACR tag push + provenance (Stage 4), and PR comment + Teams notification (Stage 5). |

---

## Runtime Step Templates (`steps/runtime/`)

These run after `docker-build.yml` in Stage 2. Only the template matching the tenant's `runtimeType` is included in the compiled pipeline.

| File | Document | Runtime | Pattern | Hard failures |
|---|---|---|---|---|
| `steps/runtime/go.yml` | [runtime/go.md](runtime/go.md) | Go | Multi-stage Dockerfile | `go.mod` missing |
| `steps/runtime/python.yml` | [runtime/python.md](runtime/python.md) | Python | Multi-stage Dockerfile | None (advisory only) |
| `steps/runtime/springboot.yml` | [runtime/springboot.md](runtime/springboot.md) | Spring Boot | Multi-stage Dockerfile + two-invocation BuildKit | Build tool missing; `AS test-export` missing; test failures |
| `steps/runtime/angular.yml` | [runtime/angular.md](runtime/angular.md) | Angular | Multi-stage Dockerfile + BuildKit npm secret | None (advisory only) |
| `steps/runtime/react.yml` | [runtime/react.md](runtime/react.md) | React / Next.js | Multi-stage Dockerfile + BuildKit npm secret | None (advisory only) |

---

## Output Variables Reference

All output variables are emitted with `isOutput=true` and are referenced across stages using the pattern:
```
$(stageDependencies.<StageName>.<JobName>.outputs['<stepName>.<VARNAME>'])
```

| Variable | Emitted by | Cross-stage reference |
|---|---|---|
| `ACR_HOST` | `setup.yml` / `resolveTools` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.ACR_HOST'])` |
| `HADOLINT_VERSION` | `setup.yml` / `resolveTools` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.HADOLINT_VERSION'])` |
| `SYFT_VERSION` | `setup.yml` / `resolveTools` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.SYFT_VERSION'])` |
| `COSIGN_VERSION` | `setup.yml` / `resolveTools` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.COSIGN_VERSION'])` |
| `DOCKER_BUILDKIT_VERSION` | `setup.yml` / `resolveTools` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.DOCKER_BUILDKIT_VERSION'])` |
| `NPM_REGISTRY_URL` | `setup.yml` / `resolveTools` | `$(stageDependencies.Setup.Setup.outputs['resolveTools.NPM_REGISTRY_URL'])` |
| `IMAGE_DIGEST` | `docker-build.yml` / `buildImage` | `$(stageDependencies.Build.Build.outputs['buildImage.IMAGE_DIGEST'])` |
| `GO_VERSION` | `runtime/go.yml` / `goRuntime` | `$(stageDependencies.Build.Build.outputs['goRuntime.GO_VERSION'])` |
| `PYTHON_VERSION` | `runtime/python.yml` / `pythonRuntime` | `$(stageDependencies.Build.Build.outputs['pythonRuntime.PYTHON_VERSION'])` |
| `SPRINGBOOT_VERSION` | `runtime/springboot.yml` / `springbootRuntime` | `$(stageDependencies.Build.Build.outputs['springbootRuntime.SPRINGBOOT_VERSION'])` |
| `ANGULAR_VERSION` | `runtime/angular.yml` / `angularRuntime` | `$(stageDependencies.Build.Build.outputs['angularRuntime.ANGULAR_VERSION'])` |
| `REACT_VERSION` | `runtime/react.yml` / `reactRuntime` | `$(stageDependencies.Build.Build.outputs['reactRuntime.REACT_VERSION'])` |
| `MANIFEST_DIGEST` | `sbom-sign-publish.yml` / `signAttest` | `$(stageDependencies.SignAndAttest.SignAndAttest.outputs['signAttest.MANIFEST_DIGEST'])` |
| `IMAGE_REF` | `sbom-sign-publish.yml` / `publish` | `$(stageDependencies.Publish.Publish.outputs['publish.IMAGE_REF'])` |

---

## Call Sequence

```
container-build-v2.yml
│
├── Stage 1: Setup
│     └── steps/setup.yml
│           ├── resolveTools       → ACR_HOST, HADOLINT_VERSION, SYFT_VERSION,
│           │                        COSIGN_VERSION, NPM_REGISTRY_URL, DOCKER_BUILDKIT_VERSION
│           └── validateParams
│
├── Stage 2: Build
│     └── steps/dockerfile-lint.yml
│           └── Run Hadolint
│
│     └── steps/docker-build.yml
│           └── buildImage         → IMAGE_DIGEST
│
│     └── steps/runtime/<runtimeType>.yml   (exactly one, selected at compile time)
│           ├── go.yml      → goRuntime         → GO_VERSION
│           ├── python.yml  → pythonRuntime      → PYTHON_VERSION
│           ├── springboot.yml → springbootRuntime → SPRINGBOOT_VERSION
│           │                 → extractTests (docker build --target test-export)
│           │                 → PublishTestResults@2
│           │                 → buildFinalImage (docker build --target final)
│           ├── angular.yml → angularRuntime     → ANGULAR_VERSION
│           └── react.yml   → reactRuntime       → REACT_VERSION
│
├── Stage 3: Sign & Attest  [skipped if dryRun: true]
│     └── steps/sbom-sign-publish.yml (phase: signAndAttest)
│           ├── Download Syft
│           ├── Download Cosign
│           ├── AzureKeyVault@2
│           ├── generateSbom      (Syft → sbom.cdx.json)
│           ├── PublishPipelineArtifact@1  (sbom-<tenant>-<app>)
│           └── signAttest        → MANIFEST_DIGEST
│
├── Stage 4: Publish  [skipped if dryRun: true]
│     └── steps/sbom-sign-publish.yml (phase: publish)
│           ├── assertTags
│           ├── publish           → IMAGE_REF
│           ├── writeProvenance   (jq → provenance.json)
│           └── PublishPipelineArtifact@1  (provenance-<tenant>-<app>)
│
└── Stage 5: Notify  [always runs, dry-run aware]
      └── steps/sbom-sign-publish.yml (phase: notify)
            ├── postPrComment     (ADO REST API)
            ├── setBuildTag       (##vso[build.addbuildtag])
            └── notifyTeams       (Teams Adaptive Card webhook)
```
