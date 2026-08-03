# PMM package test POC

Proof-of-concept for [PMM-15197](https://perconadev.atlassian.net/browse/PMM-15197): can PMM's real package-testing suite run on standard GitHub-hosted Actions runners instead of dedicated Jenkins EC2 agents, using Podman for systemd support?

`.github/workflows/package-test-matrix.yml` runs a 3-way matrix (Oracle Linux 8, Debian 12, Ubuntu 22.04) in parallel on stock `ubuntu-latest` runners. Each leg:
1. Starts a real `percona/pmm-server:3.9.0` container.
2. Starts its OS as a container via Podman with real `systemd` as PID 1, on host networking so it can reach the PMM Server.
3. Clones `percona/pmm-qa` and runs the actual, unmodified package-testing playbook (`package_tests/pmm3-client_integration.yml`) — the same one run in Jenkins.

**Result: all three OSes pass the real playbook end-to-end, in parallel** — real `pmm-client` install, version check, PMM Server registration, monitoring agents running, full DB-provisioning and metrics verification.
