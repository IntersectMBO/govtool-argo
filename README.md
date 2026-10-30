# GovTool Kubernetes Deployment

This repository contains the Helm chart and Argo CD configuration used to deploy the GovTool stack on Kubernetes.

## Overview

GovTool is deployed to a multi-node Kubernetes cluster using:

- Helm for Kubernetes deployment configuration
- Argo CD for GitOps-based deployment and synchronization
- Kubernetes Ingress for routing traffic
- Kubernetes Secrets for runtime configuration
- A PostgreSQL instance for the metadata service and the proposal discussion forum

## Architecture

```txt
GitHub Repository
  -> Argo CD Application
  -> Helm Chart
  -> Kubernetes Resources
  -> GovTool Services
```

## Services

The following GovTool services are deployed in the `govtool` namespace:

```txt
govtool-frontend   ghcr.io/intersectmbo/govtool-frontend
govtool-backend    ghcr.io/intersectmbo/govtool-backend (/api, /swagger-ui)
govtool-metadata   ghcr.io/intersectmbo/govtool-metadata-service (internal only)
govtool-pdf        ghcr.io/intersectmbo/govtool-pdf-backend (/pdf)
govtool-postgres   postgres:16-alpine, metadata and pdf databases
```

These are GovTool v3's services, the same as docker/swarm-stack/docker-stack.yml
in IntersectMBO/govtool; the four app images share one tag. The old
metadata-validation, outcomes and Strapi (proposal pillar) services are gone.
The Secrets the chart reads (govtool-v3-app, govtool-v3-db) are listed at the
top of values.yaml and are created in the namespace out of band.

## Ingress

GovTool is exposed through an NGINX Ingress.

```txt
Host: k8s.dev.gov.tools
Ingress class: nginx
```

## Argo CD

Argo CD is installed in the `argocd` namespace and manages the GovTool deployment from GitHub.

Everything Argo CD runs is defined under `argocd/` (app of apps):

```txt
argocd/
  bootstrap/root.yaml               applied once by hand; syncs the rest of argocd/
  apps/govtool.yaml                 the govtool Application (this chart, namespace govtool)
  appsets/govtool-pr-previews.yaml  one Application per previews/pr-*.yaml
  config/argocd-ingress.yaml        Argo CD's own ingress (argo.dev.gov.tools)
```

To change an Application, the preview ApplicationSet or Argo CD's config, edit
the file and merge to main; root applies it and reverts edits made by hand.
Root does not prune: a file removed from `argocd/` leaves its object in the
cluster until it is deleted on purpose.

Bootstrap a new cluster (once, after installing Argo CD):

```bash
kubectl apply -f argocd/bootstrap/root.yaml
```

## Helm Chart

The Helm chart defines:

- Deployments
- Services
- Ingress
- Runtime environment variables
- Secret/config mounts
- The postgres volume mount and its init script (files/postgres-init.sh)

### Render Locally

```bash
helm template govtool .
```

### Lint Chart

```bash
helm lint .
```

### Manual Helm Install

```bash
helm upgrade --install govtool . -n govtool --create-namespace
```

## Required Secrets

Two Secrets in the namespace, created before the first sync:

```txt
govtool-v3-app  GOVTOOL_DBSYNC_PASSWORD, METADATA_DATABASE_URL,
                PDF_DATABASE_URL, PDF_JWT_SECRET, PDF_REFRESH_SECRET
govtool-v3-db   postgres_password, metadata_db_password, pdf_db_password
```

The database URLs use the role passwords from govtool-v3-db:
postgresql://metadata:<metadata_db_password>@govtool-postgres:5432/metadata and
postgresql://pdf:<pdf_db_password>@govtool-postgres:5432/pdf?schema=public.
The JWT and refresh secrets are at least 32 characters each and differ.

```bash
kubectl create secret generic govtool-v3-db -n govtool \
  --from-literal=postgres_password="$POSTGRES_PASSWORD" \
  --from-literal=metadata_db_password="$METADATA_DB_PASSWORD" \
  --from-literal=pdf_db_password="$PDF_DB_PASSWORD"

kubectl create secret generic govtool-v3-app -n govtool \
  --from-literal=GOVTOOL_DBSYNC_PASSWORD="$DBSYNC_PASSWORD" \
  --from-literal=METADATA_DATABASE_URL="postgresql://metadata:$METADATA_DB_PASSWORD@govtool-postgres:5432/metadata" \
  --from-literal=PDF_DATABASE_URL="postgresql://pdf:$PDF_DB_PASSWORD@govtool-postgres:5432/pdf?schema=public" \
  --from-literal=PDF_JWT_SECRET="$PDF_JWT_SECRET" \
  --from-literal=PDF_REFRESH_SECRET="$PDF_REFRESH_SECRET"
```

## Database

`govtool-postgres` holds the metadata and pdf databases. On an empty data
directory files/postgres-init.sh creates one role and database per app; the
roles keep the password they were created with, so changing one later needs
ALTER ROLE as well as the Secret. The metadata service and the forum backend
migrate their databases on start.

The data is a `hostPath` on the node labelled `govtool-pdf-db=true`
(/var/lib/govtool/v3-postgres; PR previews use their own path), and the
Deployment uses the Recreate strategy so two Postgres processes never share it.

```bash
kubectl label node k8s-govtool-worker-1 govtool-pdf-db=true
```

### Check Application

```bash
kubectl get application govtool -n argocd
```

### Describe Application

```bash
kubectl describe application govtool -n argocd
```

### Check Resources

```bash
kubectl get pods -n govtool
kubectl get svc -n govtool
kubectl get ingress -n govtool
```


