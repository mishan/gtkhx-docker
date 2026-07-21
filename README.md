# gtkhx-docker

GtkHx CI base image

## Images

| Image (`images/<dir>`) | Published tag    | Default ports        | Role |
|------------------------|------------------|----------------------|------|
| `socks-proxy`          | `gtkhx-socks`    | 1080                 | SOCKS5 proxy (microsocks) |
| `ci-base`              | `gtkhx-ci-base` | —                    | Fedora + the GtkHx build/test toolchain |

## Where images are published

GitHub Container Registry (GHCR), under the repo owner's namespace:

```
ghcr.io/mishan/gtkhx-socks:latest
ghcr.io/mishan/gtkhx-ci-base:latest      # also :fedora43
```

Each is also tagged with the commit SHA; server images get the git tag
name on `v*` releases. Two workflows publish (both push with the built-in
`GITHUB_TOKEN` — enable `packages: write`, already declared):

- `.github/workflows/publish-ci-base.yml` — the CI base. Triggers on
  `images/ci-base/**`, weekly (Fedora updates), and manually.

After the first run, make the packages **public** (or grant readers) on
GHCR so they can be pulled without auth — new packages are private by
default.

## Using in GtkHx's CI

GtkHx's CI pulls the `gtkhx-ci-base` toolchain instead of
building/installing on every run. The exact
GtkHx-side changes are in
[docs/consuming-in-gtkhx-ci.md](docs/consuming-in-gtkhx-ci.md).

## Keeping in sync with GtkHx

`images/ci-base/Dockerfile`'s package list mirrors GtkHx's CI build step —
if GtkHx grows a build dep, add it there and re-publish. The server
images are independent of GtkHx's port matrices (the overlay pins those).
