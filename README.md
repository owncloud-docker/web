# owncloud/web Docker Image

[![Docker CI](https://github.com/owncloud-docker/web/actions/workflows/main.yml/badge.svg)](https://github.com/owncloud-docker/web/actions/workflows/main.yml)

Docker image for [ownCloud Web](https://github.com/owncloud/web) — the modern ownCloud frontend. Published to Docker Hub as [`owncloud/web`](https://hub.docker.com/r/owncloud/web).

## Tags

| Tag | Description |
|-----|-------------|
| `latest`, `12`, `12.3`, `12.3.3` | Current stable release |

## Usage

```bash
docker run -p 8080:8080 owncloud/web:latest
```

The web UI is served on port 8080.

## Releasing a new version

1. On a feature branch, update `version` and `extra-tags` in the matrix in `.github/workflows/main.yml`
2. Open a PR, get it reviewed and merged into `master`
3. Tag the merge commit and push the tag:

```bash
git tag v<VERSION>
git push origin v<VERSION>
```

The CI workflow builds multi-arch images (`linux/amd64`, `linux/arm64`), runs a Trivy security scan, and pushes to Docker Hub only on tag events.

## Development

Build locally:

```bash
docker build --build-arg VERSION=12.3.3 -f Dockerfile.multiarch -t owncloud/web:dev .
```
