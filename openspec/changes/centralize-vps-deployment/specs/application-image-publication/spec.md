## Purpose

Establish a repeatable contract for publishing the RestoBite backend and public frontend images to the private GitHub Container Registry.

## ADDED Requirements

### Requirement: Application workflow prompts
The deployment repository SHALL provide one OpenSpec prompt for each application repository that requests a GitHub Actions workflow to build and publish its production image to that repository's private GHCR package.

#### Scenario: Backend prompt is used
- **WHEN** the backend prompt is applied in `restobite-backend`
- **THEN** the requested workflow publishes `ghcr.io/felynx3/restobite-backend`

#### Scenario: Frontend prompt is used
- **WHEN** the frontend prompt is applied in `restobite-public`
- **THEN** the requested workflow publishes `ghcr.io/felynx3/restobite-public`

### Requirement: Immutable and release image tags
Each requested application workflow SHALL publish an immutable `sha-<commit>` tag for every qualifying push and SHALL additionally publish the matching `vX.Y.Z` tag when triggered by a release tag. It MUST grant only the package-write permission required for publishing.

#### Scenario: Main branch build
- **WHEN** a commit is pushed to the main branch
- **THEN** the workflow publishes the image tagged with that commit SHA

#### Scenario: Version release build
- **WHEN** a tag matching `vX.Y.Z` is pushed
- **THEN** the workflow publishes both its SHA tag and its version tag

### Requirement: Runtime secret boundary
The workflow prompts SHALL require server-side credentials to be supplied to containers at VPS runtime rather than embedded in an image. They MAY use explicitly identified public build-time values when the frontend framework requires them.

#### Scenario: Workflow receives runtime secrets
- **WHEN** an application image is built by the requested workflow
- **THEN** database credentials, HMAC secrets, and cloud credentials are not passed as image build arguments or written into image layers

