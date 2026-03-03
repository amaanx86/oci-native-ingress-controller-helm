# Changelog

All notable changes to the OCI Native Ingress Controller Helm Chart are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-03-03

### ✨ Initial Release

Initial v1.0.0 release with health probe enhancements for Cloud Guard compliance.

### Fixed

- **Fix oracle#138: Add health checker implementation for readiness and liveness probes**
  - Implement HealthChecker with endpoints for tracking cache synchronization and controller readiness status
  - Provides `/healthz/ready` and `/healthz/live` handlers for Kubernetes probe support

- **Fix oracle#138: Add health check endpoints and controller readiness tracking**
  - Register `/healthz/ready` and `/healthz/live` HTTP endpoints on the metrics server
  - Mark controllers as ready after initialization for proper health probe support

- **Fix oracle#138: Mark informer caches as synced for health checks**
  - Signal to the health checker that all informer caches have been synced after setup
  - Enable readiness checks to report ready status

- **Fix oracle#138: Update health probes to use HTTP endpoints on metrics server**
  - Replace TCP socket probes on webhook-server with HTTP GET endpoints
  - Readiness probe: `/healthz/ready` on metrics server port
  - Liveness probe: `/healthz/live` on metrics server port

- **Fix oracle#138: Update probe comments in values.yaml**
  - Document HTTP readiness and liveness endpoints on the metrics server

- **Fix oracle#138: Move health probe comments to metrics section**
  - Relocate health probe documentation to the metrics section where the probes actually connect

---

## How to Submit Changes

When making changes to the chart:

1. Update the `version` field in `Chart.yaml` using semantic versioning
2. Add an entry to this CHANGELOG.md under the new version
3. Create a pull request with your changes
4. Once merged to main, a GitHub Actions workflow will automatically:
   - Create a release with the new version tag
   - Package the Helm chart
   - Make it available on ArtifactHub

### Example Commit Message

```
feat: add pod disruption budget support

- Adds optional podDisruptionBudget configuration
- Improves cluster stability during maintenance windows
```

Then update CHANGELOG.md and Chart.yaml version accordingly.
