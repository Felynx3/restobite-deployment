## 1. Compose platform

- [x] 1.1 Create the six-service production Docker Compose stack with private networking, immutable GHCR application image references, persisted volumes, and Nginx-only public ports; verify `docker compose config` renders all services without a `build`, database, or Jenkins section.
- [x] 1.2 Add versioned Nginx, Loki, and Promtail configuration that preserves API/public origin checks, API/public/logs routing, and both log ingestion paths; verify the rendered Nginx configuration passes `nginx -t` with valid VPS certificates.

## 2. Runtime configuration and operations

- [x] 2.1 Add ignored runtime-environment and credential paths plus non-secret example files for Compose, backend, public frontend, Nginx, and Grafana; verify tracked files contain no credentials and Compose requires explicit application image tags.
- [x] 2.2 Write the deployment guide with the component inventory, GHCR login/pull procedure, data backup/migration, cutover, validation, rollback, and Jenkins DNS decommission note; verify all referenced service names and paths match Compose.

## 3. Application image publishing guidance

- [x] 3.1 Create `WORKFLOW_PROMPTS.md` with decision-complete OpenSpec prompts for backend and public frontend GitHub Actions image publication; verify both specify private GHCR images, `sha-<commit>` and `vX.Y.Z` tags, package permissions, Buildx caching, and the runtime-secret boundary.

## 4. Verification

- [x] 4.1 Validate the OpenSpec change and Docker Compose configuration using generated non-secret fixtures; verify the change is strict-valid and the rendered stack has the expected image references, network, mounts, and environment contracts.
