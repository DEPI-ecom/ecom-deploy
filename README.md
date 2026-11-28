# ecom-deploy

What runs in the cluster. Argo CD watches this repo.

```text
argocd/
  root.yaml             the root Application (applied once by Ansible)
  apps/<name>.yaml      one Application per service
charts/<name>/          one Helm chart per service
  values.yaml           image, tag and replicas
  templates/
    deployment.yaml
    service.yaml
    configmap.yaml      only for services that need settings
apps/observability/     monitoring (not deployed yet, see below)
postgres/               database init script used by Docker Compose
```

App of apps: `root` watches `argocd/apps`, and each file there deploys one folder in `charts/`.
To add a service, add its chart and one file in `argocd/apps`.

Secrets are not in git. `configmap.yaml` holds normal settings; passwords come from the `marketly-secrets` Secret.

The database names are listed twice: `charts/postgres/templates/configmap-init.yaml` and `postgres/init-databases.sql`.

How traffic flows:

    browser -> public node :80 -> ecom-web -> /api -> ecom-api-gateway -> backend services -> postgres

## Before the first deploy

1. The cluster and Argo CD come from `ecom-infra`. Nodes need the labels `marketly.io/role=public`,
   `backend` or `database`, and the database node the taint `marketly.io/role=database:NoSchedule`.
2. All 8 images must exist in GHCR and be **public**.
3. Set `frontendOrigin` in `charts/ecom-api-gateway/values.yaml`.
4. Create the secret once, on the control plane. It is never committed.

       kubectl create namespace marketly
       kubectl -n marketly create secret generic marketly-secrets \
         --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 16)" \
         --from-literal=JWT_SECRET="$(openssl rand -hex 32)"

   Postgres stores the password on first start. Creating a new secret later does not change it.
5. Load the sample products once:

       kubectl -n marketly exec deploy/ecom-catalog -- node prisma/seed.js

## Deploying a new version

Change `tag` in `charts/<service>/values.yaml` to the commit SHA that CI pushed, and commit. Argo CD does the rest.
Roll back with `git revert`. Pushing a new `latest` image alone does not trigger Argo CD.

## Monitoring

`apps/observability` does not work on this cluster yet: the Alloy repository URL is wrong,
Loki's default caches ask for about 11 GiB of memory, and there is no StorageClass for its volumes.
