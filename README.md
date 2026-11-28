# Marketly deployment charts

This directory contains Helm charts for deploying Marketly and its observability stack to Kubernetes. Cluster provisioning remains in `ecom-infra`.

## Layout

- `charts/ecom-service`: reusable Deployment, Service, probe, and optional ServiceMonitor templates.
- `apps/marketly`: umbrella chart configuring PostgreSQL, the web app, API gateway, and six backend services.
- `charts/postgres`: PostgreSQL StatefulSet, ClusterIP Service, static local PV/PVC, and database bootstrap config.
- `apps/observability`: upstream Grafana/Prometheus, Loki, and Alloy charts.
- `postgres/init-databases.sql`: Compose bootstrap script; keep it synchronized with the copy packaged in `charts/postgres/files/`.

## Configure and install with Helm

1. Set the container registry and image tag in `apps/marketly/values/dev.yaml` or `prod.yaml`.
2. Create `marketly-postgres-auth` in the `marketly` namespace with `username` and `password` keys. Create `marketly-runtime-secrets` there with `DATABASE_URL_USER`, `DATABASE_URL_CATALOG`, `DATABASE_URL_CART`, `DATABASE_URL_ORDER`, `DATABASE_URL_PAYMENT`, and `JWT_SECRET`. Each database URL should use `postgresql://<username>:<password>@postgres:5432/<service_db>` and match the password in `marketly-postgres-auth`.
3. Build both charts' dependencies from the monorepo root: `helm dependency build ecom-deploy/apps/observability` and `helm dependency build ecom-deploy/apps/marketly`.
4. Install observability first with `helm upgrade --install marketly-observability ecom-deploy/apps/observability -n monitoring --create-namespace -f ecom-deploy/apps/observability/values/prod.yaml`. After Prometheus Operator has installed its `ServiceMonitor` CRD, install Marketly with `helm upgrade --install marketly ecom-deploy/apps/marketly -n marketly --create-namespace -f ecom-deploy/apps/marketly/values/prod.yaml`. Use the corresponding `dev.yaml` files for development.

PostgreSQL is a single-replica StatefulSet. Its local PV points to `/mnt/postgres-data` on the labeled `db-worker`, backed by the EBS volume Terraform attaches in the isolated subnet. Set Terraform `db_volume_size` and Helm `postgres.persistence.size` to the same capacity. Ansible formats a new empty volume, mounts it, and assigns ownership to the PostgreSQL Alpine user before joining the node. The isolated worker AMI must already contain containerd, kubelet, kubeadm, kubectl, and the PostgreSQL image because the subnet has no internet route.

The Marketly services are ClusterIP-only behind ingress routes for the web app (`/`) and API gateway (`/api`). Install an ingress controller, then replace `marketly.example.com` and `https://marketly.example.com` in the values with the real frontend hostname. Create `grafana-admin` in the `monitoring` namespace with `admin-user` and `admin-password` keys. Credentials are provided through Kubernetes Secrets, not chart values.

Prometheus scrapes services exposing `/metrics`: API gateway, cart, order, and payment. User, catalog, search, and web do not currently expose Prometheus metrics. Alloy runs on each node, collects Kubernetes pod logs, and writes them to Loki.

The observability chart uses upstream Helm repositories and deploys Prometheus Operator CRDs via kube-prometheus-stack. Install it before Marketly so the `ServiceMonitor` CRD exists. For production, configure persistent storage, resource limits, authentication, and retention to match the cluster capacity and operational requirements.
