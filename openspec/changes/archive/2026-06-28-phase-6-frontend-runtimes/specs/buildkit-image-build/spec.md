## ADDED Requirements

### Requirement: NPM_REGISTRY build arg and npm_token secret are passed to every build
The `steps/docker-build.yml` template SHALL include `--build-arg "NPM_REGISTRY=$NPM_REGISTRY_URL"` and `--secret id=npm_token,env=SYSTEM_ACCESSTOKEN` on every `docker build` invocation. `NPM_REGISTRY_URL` SHALL be passed as a parameter from the base template (`npmRegistryUrl`). `SYSTEM_ACCESSTOKEN` SHALL be mapped from `$(System.AccessToken)` in the step `env:` block.

Dockerfiles that do not declare `ARG NPM_REGISTRY` or use `--mount=type=secret,id=npm_token` SHALL silently ignore these flags — Docker ignores undeclared build args; BuildKit ignores unmounted secrets.

#### Scenario: Angular or React Dockerfile consumes npm credentials
- **WHEN** the Dockerfile contains `ARG NPM_REGISTRY` and `RUN --mount=type=secret,id=npm_token npm ci`
- **THEN** the build step provides the registry URL as a build arg and the ADO pipeline token as a BuildKit secret; npm authenticates to Azure Artifacts successfully

#### Scenario: Go or Python Dockerfile ignores npm credentials
- **WHEN** the Dockerfile contains no `ARG NPM_REGISTRY` and no `--mount=type=secret,id=npm_token`
- **THEN** Docker ignores `--build-arg NPM_REGISTRY`; BuildKit ignores the unmounted `npm_token` secret; the build proceeds normally without error

#### Scenario: npm_token secret is not exposed in image layers
- **WHEN** the build completes successfully
- **THEN** `docker history <image>` does not contain the value of `SYSTEM_ACCESSTOKEN`; the token is accessible only within `RUN --mount=type=secret,id=npm_token` steps and is never written to any layer
