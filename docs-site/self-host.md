---
title: Self-Hosting & Deployment
description: Deploying a private Cloud Harness MCP server on your own infrastructure.
---

# Self-Hosting & Deployment

Cloud Harness MCP is designed for straightforward self-hosting on any modern Linux VPS or cloud instance.

## Deployment Architectures

### 1. 1-Click Automated Deployment (Caddy / Cloudflare Tunnel)

For a modern, zero-fuss self-hosted deployment on standard Linux (Ubuntu 24.04 LTS):

```bash
# Single-command install with Let's Encrypt TLS (Caddy)
curl -fsSL https://raw.githubusercontent.com/bestagentkits/cloud-harness-mcp/main/scripts/install.sh | sudo bash
```

- **Caddy Ingress:** Automatically requests and renews Let's Encrypt TLS certificates, proxying traffic to the loopback ingress `127.0.0.1:3100`.
- **Cloudflare Tunnel:** Connects `cloudflared` directly to the `ingress` network, exposing zero host listening ports.
- **Management:** Uses the `cloudharness` CLI (`cloudharness status`, `cloudharness logs`, `cloudharness token`).

### 2. Traditional NGINX Reverse Proxy Runbook

For custom infrastructure with existing NGINX setups:

1. **Bootstrap VPS:** Run `deploy/scripts/bootstrap-vps.sh` to install Docker, systemd unit, and permissions.
2. **Deploy Release:** Run `deploy/scripts/deploy-release.sh <git-sha>` to build and start production containers. The deploy runs as a host systemd unit (`cloud-harness-deploy-<sha>`), so a dropped SSH session does not interrupt it; follow it with `journalctl -fu cloud-harness-deploy-<sha>`.
3. **Canary Verification:** Run `scripts/deploy-canary.mjs` to execute an automated end-to-end workspace test against the newly deployed instance.

When you deploy from GitHub Actions (`.github/workflows/deploy.yml`), each merge to `main` deploys once. If that merge cuts a release, the Release workflow requests a second CI run for its version commit and production deploys the tagged commit once that run passes, so the reported server version matches the newest release tag. That adds one CI cycle before the merge reaches production. If the version commit's CI fails, the merge itself deploys instead; a flaky failure can be retried with `gh workflow run ci.yml --ref main` while the version commit is still the head of `main`.

### 3. Optional: live container logs

`deploy/dozzle/compose.yaml` runs a standalone [Dozzle](https://dozzle.dev) log viewer next to the harness. Generate its login first — it refuses to start without one — then start it:

```bash
sudo install -d -m 0755 /etc/cloud-harness-dozzle
read -rsp 'Dozzle admin password: ' DOZZLE_PW; echo
printf '%s\n' "$DOZZLE_PW" | sudo docker run -i --rm amir20/dozzle:v10.6.15 generate admin \
  | sudo tee /etc/cloud-harness-dozzle/users.yml >/dev/null
unset DOZZLE_PW
sudo chmod 0600 /etc/cloud-harness-dozzle/users.yml
sudo docker compose -f deploy/dozzle/compose.yaml up -d
```

Use `-i` without `-t` (a TTY merges the password prompt into the generated YAML) and keep the file root-owned at mode `0600`.

It binds only `127.0.0.1:8080` (reach it with `ssh -L 8080:127.0.0.1:8080 <host>`), keeps container actions and shells disabled, and reads every container's logs through the Docker socket. Expose it behind an authenticating proxy only.

The image tag is pinned in `deploy/dozzle/compose.yaml`: update it there on `main`, deploy that commit, then start the stack again. Editing the deployed checkout does not persist, because a deploy force-checks-out its release commit and rejects a dirty tree.
