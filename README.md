# PMM package test on GitHub Actions

Replaces PMM's Jenkins package-testing pipeline
([PMM-15197](https://perconadev.atlassian.net/browse/PMM-15197)) with GitHub Actions:
**8 test playbooks × 9 operating systems = 72 cells**, on stock GitHub-hosted runners,
no dedicated Jenkins EC2 agents.

The Ansible playbooks are cloned from [`percona/pmm-qa`](https://github.com/percona/pmm-qa)
and run **byte-for-byte as Jenkins runs them**. Everything OS-specific lives in the
prebuilt container images, never in the playbook.

## The workflows

`package-test-matrix.yml` is the **original proof of concept** — 3 OSes, 1 playbook,
images built inline. It is kept untouched as a known-good reference while the full
matrix below is brought up, and can be retired once all 72 cells are green.

The two workflows below are the full replacement.

### 1. `build-test-images.yml` — build the OS images

Builds nine systemd-capable container images and publishes them to GHCR. One job, one
matrix, nine cells. Runs **weekly (Mon 02:17 UTC)**, on demand, and on any change to
`docker/`.

amd64 only, deliberately: nothing can consume arm64 images until pmm-server ships an
arm64 build, and supporting it costs a second build axis plus a manifest-merge job.
The header comment in the workflow says exactly how to add it back.

Each image is **booted and verified before it is pushed** — systemd must reach
`running` or `degraded` and the test tooling must be present — so a broken image never
becomes `:latest`. Each OS publishes independently: one broken OS leaves the other
eight pointing at last week's good image.

Tags: `:latest` and `:YYYY-MM-DD`. To roll back, run the test matrix with an older
date tag.

### 2. `pmm3-package-tests-matrix.yml` — run the tests

On-demand only (**Actions → PMM package tests matrix → Run workflow**). Dropdowns let
you run everything or narrow to a single OS and/or a single test.

Inputs mirror the Jenkins job parameters one-for-one: `pmm_server_image`
(`DOCKER_VERSION`), `install_repo`, `pmm_version`, `metrics_mode`, `tarball`,
`admin_password`, `git_branch`.

> **`pmm_version` must match what `install_repo` actually ships** — the playbook
> asserts it. This is the same coupling Jenkins has, and Jenkins' own defaults do not
> satisfy it, so operators override there too. `release` ships the latest released
> version; `experimental`/`testing` ship the in-development one.

Results land as a pass/fail grid in the run summary. Failed cells attach their full
`ansible -vvv` output as a `logs-<os>-<test>` artifact.

## Operating systems

| OS | family | base image | needs `--privileged` |
|---|---|---|---|
| `ol8` | rpm | `oraclelinux:8` | no |
| `ol9` | rpm | `oraclelinux:9` | no |
| `alma10` | rpm | `almalinux:10` | no |
| `debian11` | deb | `debian:11` | yes |
| `debian12` | deb | `debian:12` | yes |
| `debian13` | deb | `debian:13` | yes |
| `ubuntu2204` | deb | `ubuntu:22.04` | yes |
| `ubuntu2404` | deb | `ubuntu:24.04` | yes |
| `ubuntu2604` | deb | `ubuntu:26.04` | yes |

## How it works, and why

**Podman with `--systemd=always`** boots real systemd as PID 1 inside the container, so
`systemctl start` brings up genuinely running daemons. This is the load-bearing trick —
it is what lets a container stand in for a Jenkins VM.

**RHEL vs Debian images differ fundamentally.** RHEL-family images ship systemd
already; Debian-family images deliberately ship none at all and actively block services
from starting on install. `docker/Dockerfile.deb` therefore installs systemd and
removes `/usr/sbin/policy-rc.d`, which otherwise leaves services "enabled" but never
running, silently and with no error.

**`--privileged` is scoped to the deb family only.** Its hardened units
(`valkey-server`, `mysqlrouter`) need a private mount namespace. Granting it to the
RHEL legs as well reproducibly broke `pmm-admin add mysql` there — so more privilege is
not automatically safer, and any change here needs re-verifying on every OS.

**PMM Server is mirrored into GHCR once per run.** A 72-cell run would otherwise cost
72 Docker Hub pulls against a 100-per-6-hours limit shared with every other GitHub
runner on that IP. It also pins all 72 cells to one digest, which a moving tag like
`3-dev-latest` would not otherwise guarantee.

**The Dockerfiles are single-stage on purpose.** The deliverable is the OS userspace
itself; there is no build artifact to discard, and copying a systemd install between
stages breaks package-manager state and unit symlinks.

## Maintenance

- **Add or remove an OS**: edit the `include:` list in `build-test-images.yml` and the
  `os:` list in `pmm3-package-tests-matrix.yml`. Each file has a comment pointing at
  the other.
- **The client container has its own network on purpose.** PMM asserts a Nomad node
  exists whose address is not `127.0.0.1`, so the client must be distinct from the
  server's own local node. It reaches pmm-server via the podman bridge gateway. Do not
  "simplify" this back to `--network host`.
- **Move the images to another org** (e.g. `perconalab`): change the `IMAGE_REPO` /
  `CLIENT_IMAGE_REPO` env value at the top of each workflow.
- **Bump the default expected version**: edit the `pmm_version` input default, the same
  way Jenkins maintains its version list by hand.

Supply chain: all actions are pinned to full commit SHAs, only first-party `actions/*`
actions are used, `GITHUB_TOKEN` defaults to `contents: read` with elevated scopes
granted per-job, and every published image carries a
[build provenance attestation](https://docs.github.com/actions/security-guides/using-artifact-attestations)
(`gh attestation verify`).
