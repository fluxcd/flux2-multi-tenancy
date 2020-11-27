# flux2-multi-tenancy

This repository serves as a starting point for managing multi-tenant clusters with Git and Flux v2.

![](docs/img/flux2-multi-tenancy.png)

## Roles

**Platform Admin**

- Has cluster admin access to the fleet of clusters
- Has maintainer access to the fleet Git repository
- Manages cluster wide resources (CRDs, controllers, cluster roles, etc)
- Onboards the tenant’s main `GitRepository` and `Kustomization` 
- Manages tenants by assigning namespaces, service accounts and role binding to the tenant's apps

**Tenant** 

- Has admin access to the namespaces assigned to them by the platform admin
- Has maintainer access to the tenant Git repository and apps repositories 
- Manages app deployments with `GitRepositories` and `Kustomizations`
- Manages app releases with `HelmRepositories` and `HelmReleases`

## Repository structure

The platform admin repository contains the following top directories:

- **clusters** dir contains the Flux configuration per cluster
- **infrastructure** dir contains common infra tools such as admission controllers, CRDs and cluster-wide polices
- **tenants** dir contains namespaces, service accounts, role bindings and Flux custom resources for registering tenant repositories

```
├── clusters
│   ├── production
│   └── staging
├── infrastructure
│   ├── kyverno
│   └── kyverno-policies
└── tenants
    ├── base
    ├── production
    └── staging
```

A tenant repository contains the following top directories:

- **base** dir contains `HelmRepository` and `HelmRelease` manifests
- **staging** dir contains `HelmRelease` Kustomize patches for deploying pre-releases on the staging cluster
- **production** dir contains `HelmRelease` Kustomize patches for deploying stable releases on the production cluster

```
├── base
│   ├── kustomization.yaml
│   ├── podinfo-release.yaml
│   └── podinfo-repository.yaml
├── production
│   ├── kustomization.yaml
│   └── podinfo-values.yaml
└── staging
    ├── kustomization.yaml
    └── podinfo-values.yaml
```

## Defining tenants

The Flux CLI offers commands to generate the Kubernetes manifests needed to define tenants.

Assuming a platform admin wants to create a tenant named `dev-team` with access to the `apps` namespace.

Create the tenant base directory:

```sh
mkdir -p ./tenants/base/dev-team
```

Generate the namespace, service account and role binding for the dev-team:

```sh
flux create tenant dev-team --with-namespace=apps \
    --export > ./tenants/base/dev-team/rbac.yaml
```

Create the sync manifests for the tenant Git repository:

```sh
flux create source git dev-team \
    --namespace=apps \
    --url=https://github.com/<org>/<dev-team> \
    --branch=main \
    --export > ./tenants/base/dev-team/sync.yaml

flux create kustomization dev-team \
    --namespace=apps \
    --service-account=dev-team \
    --source=GitRepository/dev-team \
    --path="./" \
    --export >> ./tenants/base/dev-team/sync.yaml
```

Create the staging overlay and set the path to the staging dir inside the tenant repository:

```sh
cat << EOF | tee ./tenants/staging/dev-team-patch.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: dev-team
  namespace: apps
spec:
  path: ./staging
EOF

cat << EOF | tee ./tenants/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base/dev-team
patchesStrategicMerge:
  - dev-team-patch.yaml
EOF
```

With the above configuration, the Flux instance running on the staging cluster will clone the
dev-team's repository, and it will reconcile the `./staging` directory from the tenant's repo
using the `dev-team` service account. Since that service account is restricted to the `apps` namespace,
the dev-team repository must contain Kubernetes objects scoped to the `apps` namespace only.
