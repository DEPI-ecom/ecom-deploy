# ecom-deploy

What runs in the cluster. Argo CD watches this repo.

```text
marketly/               one Helm chart
  values.yaml           public address and image tags (the only file you normally edit)
  templates/            one file per service: its Deployment and Service
argocd/marketly.yaml    the Argo CD Application
apps/observability/     monitoring (not deployed yet, see below)
postgres/               database init script used by Docker Compose
```

How traffic flows:

    browser -> public node :80 -> ecom-web -> /api -> ecom-api-gateway -> backend services -> postgres

## Before the first deploy

1. The cluster and Argo CD come from `ecom-infra`. Nodes need the labels `marketly.io/role=public`,
   `backend` or `database`, and the database node the taint `marketly.io/role=database:NoSchedule`.
2. All 8 images must exist in GHCR and be **public**.
3. Set `frontendOrigin` in `marketly/values.yaml`.
4. Create the secret once, on the control plane. It is never committed.

       kubectl create namespace marketly
       kubectl -n marketly create secret generic marketly-secrets \
         --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 16)" \
         --from-literal=JWT_SECRET="$(openssl rand -hex 32)"

   Postgres stores the password on first start. Creating a new secret later does not change it.
5. Load the sample products once:

       kubectl -n marketly exec deploy/ecom-catalog -- node prisma/seed.js

## Deploying a new version

Change a tag in `marketly/values.yaml` to the commit SHA that CI pushed, and commit. Argo CD does the rest.
Roll back with `git revert`. Pushing a new `latest` image alone does not trigger Argo CD.

## Monitoring

`apps/observability` does not work on this cluster yet: the Alloy repository URL is wrong,
Loki's default caches ask for about 11 GiB of memory, and there is no StorageClass for its volumes.
