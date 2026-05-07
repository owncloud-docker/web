# owncloud-docker/web — Design Spec

**Date:** 2026-05-07
**Purpose:** New standalone repository that builds and publishes `owncloud/web` Docker images using the `owncloud-docker/ubuntu` reusable workflow infrastructure, replacing the inline build in `owncloud/web`'s `release-docker.yml`.

---

## Goals

- Build and publish `owncloud/web` Docker images from pre-built release assets (no source checkout, no Node/pnpm build).
- Use the `owncloud-docker/ubuntu` reusable `docker-build.yml` workflow consistently with the rest of the org.
- Multi-arch images (`linux/amd64`, `linux/arm64`) via BuildKit.
- Version updates are manual: a human (or Renovate) updates the matrix version and commits.

---

## Repository Structure

```
owncloud-docker/web/
├── .editorconfig
├── .gitignore
├── .github/
│   ├── dependabot.yml              # keeps GitHub Actions up to date
│   └── workflows/
│       └── main.yml                # calls reusable docker-build.yml
├── overlay/
│   └── etc/
│       └── nginx/
│           ├── nginx.conf          # copied from owncloud-ops/nginx
│           └── vhost.conf          # copied from owncloud-ops/nginx
├── Dockerfile.multiarch
└── README.md
```

---

## Dockerfile

Two-stage build: `downloader` (Alpine) fetches and verifies the release asset; the final stage (nginx:alpine) serves the static files.

### Base images (SHA-pinned, kept current by Dependabot)

| Stage    | Image                  | Current digest                                                                    |
|----------|------------------------|-----------------------------------------------------------------------------------|
| builder  | `docker.io/alpine:3.22` | `sha256:310c62b5e7ca5b08167e4384c68db0fd2905dd9c7493756d356e893909057601` |
| final    | `docker.io/nginx:alpine` | `sha256:5616878291a2eed594aee8db4dade5878cf7edcb475e59193904b198d9b830de` |

### Build arg

| Arg       | Required | Description                                  |
|-----------|----------|----------------------------------------------|
| `VERSION` | yes      | owncloud/web release version, e.g. `12.3.3`  |
| `REVISION` | no      | Git SHA of this repo at build time            |

### Stage 1 — downloader

- Base: `alpine:3.22` (SHA-pinned)
- Installs `curl`
- Downloads `web.tar.gz` and `sha256sum.txt` from `https://github.com/owncloud/web/releases/download/v${VERSION}/`
- Verifies integrity: `grep "web.tar.gz" sha256sum.txt | sha256sum -c -`
- Extracts into `/var/lib/nginx/html` (strip top-level directory component)

### Stage 2 — final

- Base: `nginx:alpine` (SHA-pinned)
- OCI labels: title, vendor, authors, description, licenses, documentation, url, source, version, revision
- `COPY overlay/ /` — installs nginx.conf + vhost.conf
- `COPY --from=downloader /var/lib/nginx/html /var/lib/nginx/html`
- Exposes port `8080`
- Runs as user `nginx`
- CMD: `["nginx", "-g", "daemon off;"]`

### nginx configuration

Copied verbatim from `owncloud-ops/nginx` (Apache-2.0):

- **nginx.conf**: single worker, port 8080, SPA-friendly `try_files`, `sub_filter` injects `<base href="/">` into `index.html`
- **vhost.conf**: serves `/var/lib/nginx/html`, error pages for 5xx

---

## Workflow (`main.yml`)

### Triggers

```yaml
on:
  push:
    branches: [master]
    tags: ["*"]
  pull_request:
  schedule:
    - cron: "0 0 * * 0"   # weekly rescan on Sunday midnight
```

### Jobs

1. **lint** — calls `owncloud-docker/ubuntu/.github/workflows/lint-editorconfig.yml@master`
2. **build** — calls `owncloud-docker/ubuntu/.github/workflows/docker-build.yml@master`

### Build job inputs

| Input               | Value                                                      |
|---------------------|------------------------------------------------------------|
| `docker-repo-name`  | `owncloud/web`                                             |
| `docker-tag`        | `${{ matrix.release.version }}`                            |
| `docker-context`    | `.`                                                        |
| `docker-file`       | `./Dockerfile.multiarch`                                   |
| `docker-hub-username` | `${{ vars.DOCKERHUB_USERNAME }}`                         |
| `docker-build-args` | `VERSION=${{ matrix.release.version }}`                    |
| `docker-extra-tags` | semver aliases + `latest` (see matrix)                     |
| `push`              | `${{ startsWith(github.ref, 'refs/tags/') }}`              |
| `smoke-test-cmd`    | `wget -q --spider http://localhost:8080 \|\| exit 1`       |

Secret: `docker-hub-password` → `${{ secrets.DOCKERHUB_TOKEN }}`

### Matrix structure

```yaml
strategy:
  matrix:
    release:
      - version: "12.3.3"
        extra-tags: |
          12.3
          12
          latest
```

To release a new version: update `version` and `extra-tags` in the matrix, commit, and push a matching tag.

### Tag publishing scheme

For a release tag `v12.3.3`, the following Docker tags are published:
- `owncloud/web:12.3.3`
- `owncloud/web:12.3`
- `owncloud/web:12`
- `owncloud/web:latest`

No `master` branch publishing — master/PR runs validate only (build + Trivy scan + smoke test, no push).

---

## Dependabot

`.github/dependabot.yml` configured for the `github-actions` ecosystem, checking weekly. This keeps `actions/checkout`, `docker/build-push-action`, etc. up to date. Dockerfile base image digest pins are also updated via Dependabot's `docker` ecosystem.

---

## Out of Scope

- Migrating or modifying the `owncloud/web` release workflow — that is a separate concern.
- Automated version bump on new `owncloud/web` releases (Renovate could be added later).
- Custom nginx modules or config changes beyond what `owncloud-ops/nginx` ships.
