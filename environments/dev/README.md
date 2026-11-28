# Development environment

Use the `apps/marketly/values/dev.yaml` and `apps/observability/values/dev.yaml` files with the Helm install commands in the deployment README. Set image repositories and tags for the development registry before installing. Create `marketly-postgres-auth` and `marketly-runtime-secrets` in `marketly`, and `grafana-admin` in `monitoring`, first. PostgreSQL uses the isolated worker and EBS-backed volume in dev as well.
