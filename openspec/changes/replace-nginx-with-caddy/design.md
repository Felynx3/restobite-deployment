## Context

The current Compose stack has one Nginx service exposing ports 80 and 443, templates its configuration from `env/nginx.env`, mounts host-managed Let's Encrypt files read-only, and publishes public-site log files through the `nginx_logs` volume. Promtail tails those two files and labels them as Nginx. The three virtual hosts and their upstreams are defined in `nginx/templates/restobite.conf.template`. See proposal.md for the motivation and `specs/edge-reverse-proxy/spec.md` for the resulting behavior contract.

## Goals / Non-Goals

**Goals:**

- Replace every Nginx-specific deployment artifact with a Caddy equivalent while preserving the current edge behavior exactly.
- Make certificate acquisition, renewal, and recovery owned by the running proxy and durable across container recreation.
- Keep public-site proxy logs consumable by the existing Loki/Promtail flow.

**Non-Goals:**

- Change CloudFront, DNS ownership, upstream application ports, Grafana settings, or origin-secret values.
- Add a DNS-01 provider, alter certificate authority policy, or automate DNS changes.
- Preserve manual certificates or support a parallel Nginx proxy during operation.

## Decisions

### Replace the Compose service and all Nginx-named operational artifacts

Use the official Caddy container as the sole service binding host ports 80 and 443. Replace `nginx/` with `caddy/`, rename the proxy environment example to Caddy terminology, remove `LETSENCRYPT_DIR`, and update the README's component, bootstrap, validation, migration, and observability instructions. Rename the proxy log volume and Promtail jobs/labels so none imply Nginx remains.

Keeping Nginx filenames or variables would obscure the completed replacement and retain an unsupported certificate dependency. Running both proxies is rejected because they cannot bind the same ports and violates the requested complete replacement.

### Make Caddy configuration explicit for all three hosts

Use a Caddy configuration that reads the existing host and secret values from the proxy environment file. Define explicit HTTP handling so API and public requests are checked for the correct secret before being redirected; define their HTTPS handling with the same check before proxying. Define the logs host without a secret check. Preserve the existing upstream targets, Host forwarding, connection-upgrade support, redirect behavior, and HSTS value.

Relying solely on Caddy's automatic HTTP-to-HTTPS redirect is rejected because its processing order could redirect an unauthorized API or public request before the secret gate. Encoding hosts or secrets directly in the versioned configuration is rejected because current deployment values are environment-owned and secrets must not enter version control.

### Persist all Caddy runtime state below the repository's caddy directory

Version the Caddy configuration in `caddy/` and create mount-backed subdirectories for Caddy's `/data` and `/config` paths. The Compose service will bind mount those subdirectories so ACME accounts, certificates, renewal data, and Caddy's persisted configuration remain inside `caddy/` and survive recreation. The repository will include placeholders as needed to retain otherwise-empty persistence directories without committing generated certificate material.

Docker named volumes are rejected for this state because the requested host-mounted `caddy/` folder must contain everything Caddy needs to persist. A read-only mount of host-issued certificates is rejected because it leaves issuance and renewal external.

### Preserve Promtail file-based public proxy observability

Configure Caddy to emit structured public-site access and error records to files in a persistent proxy log mount, and point Promtail at their Caddy paths. Keep the two streams and their production/info-versus-error labels while changing their source identifier to Caddy. The exact Caddy logging directives will be validated against the selected Caddy image before deployment.

Sending only container stdout to Docker's default logging is rejected because the current Promtail configuration explicitly scrapes durable files and distinguishes access from error streams.

## Risks / Trade-offs

- [A host is not publicly reachable or DNS does not resolve to this VPS during ACME validation] → Verify DNS and inbound ports 80/443 before the cutover; retain the existing deployment rollback path until certificates are issued.
- [An upstream CDN does not forward `X-Origin-Secret`] → Validate API and public traffic through CloudFront before declaring the cutover complete.
- [Caddy logging differs from Nginx's file separation] → Validate the concrete configuration with a started container and confirm both Promtail streams arrive in Loki.
- [Host-mounted ACME files gain overly broad permissions] → Restrict access to the deployment account and do not commit generated contents of `caddy/data` or `caddy/config`.

## Migration Plan

1. Ensure each configured hostname resolves to the VPS and ports 80 and 443 are reachable for HTTP-01 validation; preserve the existing origin-secret values.
2. Create the Caddy persistence subdirectories with deployment-account permissions, populate the proxy environment file from its new example, and remove the obsolete certificate-directory variable.
3. Render and validate the Compose configuration, then start Caddy in the maintenance window after the legacy proxy relinquishes ports 80 and 443.
4. Confirm Caddy has obtained certificates, test authorized and unauthorized API/public requests plus Grafana, and confirm both Caddy public log streams reach Loki through Promtail.
5. If validation fails, stop the new Compose stack and restart the legacy containers using their existing certificate setup; preserve Caddy state for diagnosis and do not delete valid ACME material.
