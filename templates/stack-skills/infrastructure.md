# Stack skill: Infrastructure

Injected for `infrastructure` areas: deployment, CI/CD, containers, environments, observability.

- Follow the project's existing pipeline and IaC conventions — check repo CI config and any infra/ or deploy/ directories before adding anything.
- Configuration via environment variables with validated schemas at boot; never hardcode env-specific values; document every new variable in the README/env example.
- Secrets never enter the repo, images, or logs — use the project's secret manager pattern.
- Containers: small images (multi-stage builds), pinned base versions, non-root user, health checks.
- Every deployable change ships with a rollback path; migrations must be backward-compatible for one release when the project runs rolling deploys.
- Observability for new services/paths: structured logs with correlation ids, error tracking, and at least one metric/alert for the failure mode that matters.
- CI: changes must keep the pipeline green and fast — cache dependencies, run affected tests first.
- Record infra decisions (new services, changed topology) as F67 decision records.
