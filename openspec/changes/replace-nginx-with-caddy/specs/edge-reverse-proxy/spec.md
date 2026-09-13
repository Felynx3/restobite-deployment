## Purpose

Define the secure, observable public edge for RestoBite services while the proxy owns TLS certificate lifecycle automatically.

## ADDED Requirements

### Requirement: Published hosts retain their routing and TLS policy
The deployment SHALL publish only ports 80 and 443 at the edge. It SHALL redirect requests for `API_ORIGIN_HOST`, `PUBLIC_ORIGIN_HOST`, and `LOGS_HOST` received on HTTP to the equivalent HTTPS URL. HTTPS responses for all three hosts SHALL include `Strict-Transport-Security` with `max-age=31536000; includeSubDomains`.

#### Scenario: HTTP request reaches a published host
- **WHEN** a permitted request is received on port 80 for one of the configured hosts
- **THEN** the response redirects to the same host, path, query string, and HTTPS scheme

#### Scenario: HTTPS response is served for a published host
- **WHEN** a request is served through HTTPS for one of the configured hosts
- **THEN** its response includes the configured one-year HSTS policy with subdomains

### Requirement: API and public origins retain secret-gated access
The edge SHALL require `X-Origin-Secret` to exactly match `API_ORIGIN_SECRET` for every request to `API_ORIGIN_HOST` and `PUBLIC_ORIGIN_SECRET` for every request to `PUBLIC_ORIGIN_HOST`, before redirecting HTTP or proxying HTTPS. It SHALL return HTTP 403 when the required header is absent or does not match.

#### Scenario: API request has a valid origin secret
- **WHEN** a request for `API_ORIGIN_HOST` includes an `X-Origin-Secret` equal to `API_ORIGIN_SECRET`
- **THEN** the edge redirects it to HTTPS or proxies its HTTPS request to the backend service

#### Scenario: Public request has an invalid origin secret
- **WHEN** a request for `PUBLIC_ORIGIN_HOST` omits `X-Origin-Secret` or supplies a value other than `PUBLIC_ORIGIN_SECRET`
- **THEN** the edge returns HTTP 403 without redirecting or contacting the public service

### Requirement: HTTPS requests proxy to their existing private services
The edge SHALL proxy authorized HTTPS requests for `API_ORIGIN_HOST` to `backend:4000`, authorized HTTPS requests for `PUBLIC_ORIGIN_HOST` to `public:3000`, and HTTPS requests for `LOGS_HOST` to `grafana:3000`. It SHALL preserve the original Host header and support HTTP connection upgrades for each proxied service. `LOGS_HOST` SHALL not require an origin-secret header.

#### Scenario: Authorized API request is proxied
- **WHEN** an HTTPS request for `API_ORIGIN_HOST` presents the configured API origin secret
- **THEN** it is forwarded to `backend:4000` with its original Host header and upgrade capability

#### Scenario: Grafana request is proxied without an origin secret
- **WHEN** an HTTPS request for `LOGS_HOST` is received without an origin-secret header
- **THEN** it is forwarded to `grafana:3000`

### Requirement: The edge manages certificate lifecycle automatically
The edge SHALL automatically obtain and renew publicly trusted TLS certificates for `API_ORIGIN_HOST`, `PUBLIC_ORIGIN_HOST`, and `LOGS_HOST` through ACME without depending on a pre-provisioned certificate directory or `LETSENCRYPT_DIR`. All certificate-management state that must survive container recreation, including accounts, issued certificates, renewal metadata, and persisted runtime configuration, SHALL reside under the mounted `caddy/` directory.

#### Scenario: Edge starts without existing certificates
- **WHEN** the edge starts with no certificate material in its persisted directory and each configured host is reachable for ACME validation
- **THEN** it obtains a trusted certificate and begins serving that host through HTTPS

#### Scenario: Edge is recreated after certificate issuance
- **WHEN** the edge container is recreated with the same mounted `caddy/` directory
- **THEN** it reuses its persisted certificate-management state and continues to manage renewals automatically

### Requirement: Public-site proxy logs remain observable
The edge SHALL write structured access logs and error logs for `PUBLIC_ORIGIN_HOST` to persistent files that Promtail can scrape and send to Loki. The deployment SHALL retain separate production, info-level access and error-level log streams, and identify their source as the edge proxy rather than Nginx.

#### Scenario: Public request produces an access log
- **WHEN** a request is handled for `PUBLIC_ORIGIN_HOST`
- **THEN** a structured access record is written to the persistent access-log stream for Promtail

#### Scenario: Public proxy reports an error
- **WHEN** the public-site proxy emits an error
- **THEN** it is written to the persistent error-log stream for Promtail with the production and error-level labels
