## 1. Caddy configuration and persistent state

- [x] 1.1 Remove the versioned Nginx configuration and create `caddy/` with a Caddy configuration plus placeholder directories for mounted `/data` and `/config`; ignore generated ACME/runtime files while retaining the directory structure, and verify no Nginx configuration remains tracked.
- [x] 1.2 Configure Caddy's explicit HTTP and HTTPS handling for the API, public, and logs hosts: preserve the existing upstreams, Host forwarding, connection upgrades, HSTS, and secret gate before API/public redirects or proxying; verify the rendered configuration passes `caddy validate` in the selected image.
- [x] 1.3 Configure Caddy file logging for public-site access and errors in the persistent proxy-log mount; verify sample requests and failures create structured records in both configured files.

## 2. Compose and observability migration

- [x] 2.1 Replace the `nginx` Compose service with a pinned Caddy service that is the only publisher of ports 80 and 443, consumes the renamed proxy environment file, and bind-mounts all Caddy persistence below `caddy/`; verify `docker compose --env-file env/compose.env config` renders without `nginx` or `LETSENCRYPT_DIR`.
- [x] 2.2 Replace the Nginx log volume and update Promtail's mounts, file paths, job names, and source labels for Caddy while retaining separate production info and error streams; verify the rendered Compose configuration connects both services to the same log storage.
- [x] 2.3 Rename `env/nginx.env.example` for Caddy while preserving the three host and two origin-secret settings, remove `LETSENCRYPT_DIR` from `env/compose.env.example`, and verify no versioned example retains obsolete Nginx or certificate-directory variables.

## 3. Operational documentation and end-to-end validation

- [x] 3.1 Update the README's component inventory, initial VPS setup, migration, rollback, validation, and daily-operation guidance for Caddy-managed ACME certificates and `caddy/` persistence; verify it no longer instructs operators to manage or test Nginx certificates.
- [ ] 3.2 In an environment where the three DNS names resolve to the VPS and ports 80/443 are reachable, start the stack and verify Caddy obtains certificates and reuses its state after recreation.
- [x] 3.3 Verify edge behavior with requests: valid API/public secrets redirect on HTTP and proxy on HTTPS, invalid or absent secrets receive 403 before redirect/proxying, and the logs host redirects/proxies without a secret; verify HSTS is present on HTTPS responses.
- [ ] 3.4 Verify Promtail delivers Caddy public access and error streams to Loki with the expected production/info-or-error labels, then retain the legacy rollback path until API, public, Grafana, certificate, and log checks pass.
