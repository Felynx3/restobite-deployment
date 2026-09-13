# vps-compose-deployment Specification

## Purpose

Define a reproducible Docker Compose deployment for the RestoBite VPS while preserving its public routes, logging, and persistent observability data.

## Requirements

### Requirement: Compose-managed VPS services
The deployment repository SHALL define one Docker Compose stack containing Nginx, backend, frontend público, Loki, Promtail, and Grafana. It MUST NOT define a database or Jenkins service.

#### Scenario: Services are rendered from configuration
- **WHEN** an operator renders the compose configuration with required environment files present
- **THEN** it contains exactly the six managed VPS services and no database or Jenkins service

### Requirement: Private application images
The backend and frontend público services SHALL consume images from `ghcr.io/felynx3` using explicitly supplied, non-empty image tags. The deployment MUST NOT build either application image locally.

#### Scenario: Operator selects an immutable image
- **WHEN** the operator sets an application image tag to a published SHA tag
- **THEN** Compose resolves the corresponding GHCR image reference without a build context

### Requirement: Private service connectivity and public ingress
Only Nginx SHALL publish host ports. Nginx MUST proxy the API, frontend público, and Grafana hostnames to their private Compose service names, and the frontend público MUST reach the backend through the private network.

#### Scenario: CloudFront-origin request reaches the API
- **WHEN** Nginx receives a request for the API origin with the configured origin-secret header
- **THEN** it proxies the request to the backend service on its private application port

#### Scenario: Invalid origin request is rejected
- **WHEN** Nginx receives a request for an API or frontend-public origin without its configured origin-secret header
- **THEN** it responds with HTTP 403 and does not proxy the request upstream

### Requirement: Versioned configuration and external secrets
The repository SHALL version non-secret Nginx, Loki, and Promtail configuration plus non-secret environment examples. Runtime credentials, origin secrets, database connection settings, and cloud credential material MUST remain outside Git.

#### Scenario: Configuration is committed safely
- **WHEN** the deployment files are reviewed
- **THEN** they contain placeholders and variable references instead of production credentials or certificate material

### Requirement: Persistent observability
The stack SHALL persist Loki data, Grafana data, Promtail positions, and Nginx logs across service recreation. Application services SHALL be configured to send logs directly to Loki and Promtail SHALL ingest Nginx public-site logs.

#### Scenario: Logs survive a service recreation
- **WHEN** the observability services are recreated without deleting their volumes
- **THEN** previously stored Loki and Grafana data remain available and Promtail resumes from persisted positions
