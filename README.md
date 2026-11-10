# ci-workflows

Reusable GitHub Actions workflows for every ZOVA repository.
Services reference them with `@v1`, so a fix here reaches every repo without copying YAML.

| Workflow | For | What it runs |
| --- | --- | --- |
| `go-ci.yml` | Go services and libraries | golangci-lint, `go test -race`, govulncheck |
| `docker-publish.yml` | Anything with a Dockerfile | buildx build, Trivy scan, push to ghcr.io |
| `web-ci.yml` | Vite + React apps | pnpm install, lint, typecheck, build |

## Go service

`.github/workflows/ci.yml` in a service repo:

```yaml
name: ci

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:

permissions:
  contents: read

jobs:
  go:
    uses: zova-platform/ci-workflows/.github/workflows/go-ci.yml@v1

  image:
    needs: go
    if: github.event_name == 'push'
    uses: zova-platform/ci-workflows/.github/workflows/docker-publish.yml@v1
    permissions:
      contents: read
      packages: write
```

Pull requests run lint and tests only. Pushes to `main` and `v*` tags also build, scan and push the image.

## Go library (no image)

```yaml
jobs:
  go:
    uses: zova-platform/ci-workflows/.github/workflows/go-ci.yml@v1
```

## Web app

```yaml
jobs:
  web:
    uses: zova-platform/ci-workflows/.github/workflows/web-ci.yml@v1

  image:
    needs: web
    if: github.event_name == 'push'
    uses: zova-platform/ci-workflows/.github/workflows/docker-publish.yml@v1
    permissions:
      contents: read
      packages: write
```

## Inputs

### go-ci.yml

| Input | Default | Description |
| --- | --- | --- |
| `go-version-file` | `go.mod` | File that sets the Go version |
| `golangci-lint-version` | `v2.13` | golangci-lint version (v2 only) |
| `working-directory` | `.` | Directory of the Go module |
| `race` | `true` | Run tests with `-race` |

### docker-publish.yml

| Input | Default | Description |
| --- | --- | --- |
| `context` | `.` | Build context |
| `dockerfile` | `Dockerfile` | Dockerfile path |
| `platforms` | `linux/amd64` | Platforms for the pushed image |
| `push` | `true` | Push after a clean scan |
| `fail-on-severity` | `CRITICAL` | Trivy severities that fail the build |

Image name is always `ghcr.io/zova-platform/<repo>`. Tags: short SHA on every push, SemVer on `v*` tags, `latest` on `main`.
Outputs: `image`, `version` (short SHA) for later deploy steps.

### web-ci.yml

| Input | Default | Description |
| --- | --- | --- |
| `node-version` | `24` | Node.js version |
| `working-directory` | `.` | Directory with `package.json` |

`package.json` must have `lint`, `typecheck` and `build` scripts and a `packageManager` field.

## Status checks for rulesets

Job names show up as `<caller job> / <job>`, for example `go / lint`, `go / test`, `go / govulncheck`.
Select those in the repository ruleset after the first run.

## Versioning

- `v1.x.y` tags are immutable releases.
- `v1` moves to the latest `v1.x.y`. Breaking changes go to `v2`.

```bash
git tag v1.0.1 && git tag -f v1
git push origin v1.0.1 && git push -f origin v1
```

## Supply-chain notes

- `aquasecurity/trivy-action` tags were hijacked in March 2026. Always reference it by full commit SHA (`uses: aquasecurity/trivy-action@<sha> # v0.36.0`), never by tag, and update it only to a verified release.
- Dependabot opens weekly PRs for action updates.
