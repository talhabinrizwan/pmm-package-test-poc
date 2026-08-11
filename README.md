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

> **The two `upgrade` playbooks cannot pass with `install_repo=release`.** They install
> the old client from the `release` repo and then upgrade to whatever `install_repo`
> points at, so with `release` both ends are the same version, nothing upgrades, and
> the run fails with `PMM Agent was not restarted. Old PID is: N New PID is: N`. Run
> them with `experimental` or `testing` — this is why Jenkins defaults to
> `experimental`. The other six playbooks are happy either way.

The playbook is checked out on the runner and copied into the container. When this
workflow runs **inside `percona/pmm-qa`** it uses the commit that triggered the run,
so a pull request touching a playbook is tested against its own change rather than
against `main`; the `git_branch` input is only used elsewhere.

Results land as a pass/fail grid in the run summary. Failed cells attach their full
`ansible -vvv` output as a `logs-<os>-<test>` artifact.

## Operating systems

| OS | family | base image | needs `--privileged` |
|---|---|---|---|
| `ol8` | rpm | `oraclelinux:8` | no |
| `ol9` | rpm | `oraclelinux:9` | no |
| `ol10` | rpm | `oraclelinux:10` | yes |
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

**The cgroup root must stay empty, or PMM's Nomad agent will not start.** Nomad
enables cgroup controllers on the container's cgroup root, and cgroup v2 forbids that
on any cgroup with processes sitting directly in it — the agent exits with `cgroups
are not writable` and the playbook fails a Nomad assertion ~25 minutes later. Two
things put processes there: some systemd builds leave a couple behind instead of
migrating them into `init.scope`, and `podman exec` places its session in the root —
which matters because the playbook itself runs via `podman exec`. Both are handled:
the root is drained after boot, and the playbook shell moves itself into a child
cgroup before running ansible. Measured across all nine OSes, "0 processes at the
cgroup root" predicted success exactly.

**`--privileged` is about hardened systemd units, not Nomad.** `valkey-server`,
`mysqlrouter` and `valkey@default` set `PrivateTmp=`/`ProtectSystem=` and need
`CAP_SYS_ADMIN` to build a private mount namespace; without it they fail with
`status=217/USER` or `226/NAMESPACE`. Granting only `CAP_SYS_ADMIN` instead of full
`--privileged` was tried and is worse.

**`ol8` and `ol9` must stay unprivileged.** They do not need it, and it actively
breaks them — twice over. It used to break `pmm-admin add mysql`; that turned out to be
the host AppArmor profile (see above) and is fixed. Running them privileged was then
retried and still fails, now in a different place: `sudo` stops working inside the
container (`PAM account management error: Authentication service cannot retrieve
authentication info`) and the play dies at *Change Postgresql Password*. More privilege
is not automatically safer — re-verify every OS after any change here.

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
