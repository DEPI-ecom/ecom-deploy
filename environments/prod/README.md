# Production environment

Use the production values files with the Helm install commands in the deployment README. Set image repositories and immutable tags for the production registry, set Terraform `db_volume_size` equal to Helm `postgres.persistence.size`, and provision persistent volume capacity for Prometheus, Grafana, and Loki. Create `marketly-postgres-auth` and `marketly-runtime-secrets` in `marketly`, and `grafana-admin` in `monitoring`, before installation.
