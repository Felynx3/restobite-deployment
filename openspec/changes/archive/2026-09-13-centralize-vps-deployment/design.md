## Context

The current VPS runs Nginx, backend, frontend público, Jenkins, Loki, Promtail, and Grafana through independent scripts on a shared Docker network. Backend and frontend images are built on the host; Nginx configuration contains origin secrets; Loki and Grafana hold local persistent state. See proposal.md for motivation and the new specifications for the required behavior.

## Goals / Non-Goals

**Goals:**

- Make the six retained VPS services reproducible through one Compose invocation.
- Preserve existing CloudFront origin protection, TLS termination, host routing, and log ingestion paths.
- Move secrets out of versioned proxy configuration and give operators explicit image/tag and runtime-environment contracts.

**Non-Goals:**

- Provision or migrate Cloud SQL, Terraform/DNS, certificate renewal, or external cloud resources.
- Remotely deploy from GitHub Actions or implement workflows in the application repositories.
- Preserve Jenkins, its UI, or its `ci.restobite.com` route.

## Decisions

### Compose owns a new private bridge network

All retained services share a Compose-managed network and resolve one another by service name (`backend`, `public`, `loki`, `grafana`). This avoids retaining the legacy manually-created `restobite-network` and lets Nginx be included in the same stack. A named external network was rejected because it would retain a manual prerequisite and ownership ambiguity.

### Applications use immutable GHCR references

Backend and public services use `ghcr.io/felynx3/restobite-backend:${BACKEND_IMAGE_TAG}` and `ghcr.io/felynx3/restobite-public:${PUBLIC_IMAGE_TAG}`. Operators select an immutable SHA tag in the untracked Compose environment file; version tags remain a human-friendly release alias. Local `build` blocks were rejected because they retain VPS source checkouts and host builds.

### Runtime configuration is separated by service

Compose interpolation, backend, public frontend, proxy, and Grafana use distinct untracked environment files. This prevents backend credentials from being injected into the frontend. The GCP credential is supplied as an untracked Compose secret file. Non-secret examples document each contract.

### Nginx templates receive origin secrets at startup

The official Nginx image renders a committed server-configuration template at startup. Only opaque header-secret variables are substituted; Nginx request variables remain intact. This replaces committed origin secrets while preserving the existing `origin-api`, `origin-www`, and `logs` hostnames. The Jenkins vhost is deliberately absent.

### Observability uses managed named volumes

Loki storage, Grafana storage, Promtail positions, and Nginx logs are named volumes. This removes host-relative log/data paths while retaining data across recreation. The current Loki retention and the two ingestion paths (application direct to Loki; Nginx through Promtail) are preserved.

### Application workflows publish only

`WORKFLOW_PROMPTS.md` defines OpenSpec prompts that ask each application repo for a GitHub Actions publishing workflow. The workflows build with Buildx, cache through GitHub Actions, publish private GHCR packages, and use `sha-<commit>` plus release tags. VPS deployment remains an explicit operator action because it requires private-registry credentials and production runtime secrets.

## Risks / Trade-offs

- [Compose migration conflicts with legacy ports and container names] → Back up persistent data, stop legacy retained containers, and bring up the Compose stack during a maintenance window; retain the backup for rollback.
- [Private registry pull fails] → Validate a least-privilege `read:packages` credential with `docker login ghcr.io` and `docker compose pull` before stopping legacy services.
- [Nginx template or certificates are invalid] → Render Compose configuration and run `nginx -t` against the VPS certificate mount before switching traffic.
- [Existing Grafana/Loki state is lost] → Export/copy the legacy named volume and Grafana bind-mounted directory into the new named volumes before cutover.
- [Frontend configuration is evaluated at build time] → The frontend workflow prompt requires an explicit audit of public build-time variables and prohibits server-side secret build arguments.

## Migration Plan

1. Populate the private environment files and GCP credential file on the VPS; log in to GHCR with a package-read token.
2. Back up the legacy Loki volume and Grafana data directory; copy their contents into the new volumes.
3. Run `docker compose config`, pull the selected immutable image tags, and validate rendered Nginx configuration with the existing certificate mount.
4. During a maintenance window, stop the legacy Nginx, backend, public, Loki, Promtail, Grafana, and Jenkins containers; then run `docker compose up -d`.
5. Verify protected API/public routes, Grafana login, direct application logs, and Promtail-ingested Nginx logs. Decommission the legacy Jenkins DNS record outside this repository.
6. Roll back by stopping the Compose stack, restoring the backups if needed, and restarting the legacy containers on their original network.
