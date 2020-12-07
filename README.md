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

The [platform admin repository](https://github.com/fluxcd/flux2-multi-tenancy/tree/main) contains the following top directories:

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

A [tenant repository](https://github.com/fluxcd/flux2-multi-tenancy/tree/dev-team) contains the following top directories:

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

## Onboard tenants

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

Create the base `kustomization.yaml` file:

```sh
cd ./tenants/base/dev-team/ && kustomize create --autodetect
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

## Enforce tenant isolation

To enforce tenant isolation, cluster admins should configure Flux to reconcile 
the `Kustomization` and `HelmRelease` kinds by impersonating a service account
from the namespace where these objects are created. In order to make the
`spec.ServiceAccountName` field mandatory, you can use a validation webhook like
[Kyverno](https://github.com/kyverno/kyverno) or [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper).
On cluster bootstrap, you need to configure Flux to deploy the validation webhook and its policies before 
reconciling the tenants repositories.

Inside the `clusters` dir we define in which order the infrastructure items,
and the tenant workloads are going to be reconciled on the staging and production clusters:

```
./clusters/
├── production
│   ├── infrastructure.yaml
│   └── tenants.yaml
└── staging
    ├── infrastructure.yaml
    └── tenants.yaml
```

First we setup the reconciliation of custom resource definitions and their controllers. For this 
example we'll use [Kyverno](https://github.com/kyverno/kyverno):

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: kyverno
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/kyverno
  prune: true
  validation: client
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: kyverno
      namespace: kyverno
```

Then we setup [cluster policies](./infrastructure/kyverno-policies/flux-multi-tenancy.yaml) 
(Kyverno custom resources) to enforce tenant isolation:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: kyverno-policies
  namespace: flux-system
spec:
  dependsOn:
    - name: kyverno
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/kyverno-policies
  prune: true
  validation: client
```

With `dependsOn` we tell Flux to install Kyverno before deploying the cluster policies.

And finally we setup the reconciliation for the tenants workloads with:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: tenants
  namespace: flux-system
spec:
  dependsOn:
    - name: kyverno-policies
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./tenants/staging
  prune: true
  validation: client
```

With the above configuration, we ensure that the Kyverno validation webhook will reject `Kustomizations` and 
`HelmReleases` that don't specify a service account name when deployed in a tenant's namespace.

## Bootstrap the staging cluster

Install the Flux CLI and fork this repository on your personal GitHub account
and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Wait for the staging cluster reconciliation to finish:

```console
$ watch flux get kustomization
NAME            	READY  	MESSAGE                                                        	
flux-system     	True   	Applied revision: main/616001c38e7bc81b00ef2c65ac8cfd58140155b8	
kyverno         	Unknown	Reconciliation in progress
kyverno-policies	False  	Dependency 'flux-system/kyverno' is not ready
tenants         	False  	Dependency 'flux-system/kyverno-policies' is not ready
```

Verify that the tenant Git repository has been cloned:

```console
$ flux -n apps get sources git
NAME    	READY	MESSAGE 
dev-team	True 	Fetched revision: dev-team/ca8ec25405cc03f2f374d2f35f9299d84ced01e4
```

Verify that the tenant Helm repository index has been downloaded:

```console
$ flux -n apps get sources helm
NAME   	READY	MESSAGE
podinfo	True 	Fetched revision: 2020-10-28T10:09:58.648748663Z
```

Wait for the demo app to be installed:

```console
$ watch flux -n apps get helmreleases
NAME   	READY	MESSAGE                         	REVISION	SUSPENDED 
podinfo	True 	Release reconciliation succeeded	5.0.3   	False 
```

## Onboard tenants with private repositories

You can configure Flux to connect to a tenant repository
using SSH or token-based authentication. The tenant credentials will be stored 
in the platform admin repository as a Kubernetes secret. 

### Encrypt Kubernetes secrets in Git

In order to store credentials safely in a Git repository, you can use Mozilla's
SOPS CLI to encrypt Kubernetes secrets with OpenPGP or KMS.

Install [gnupg](https://www.gnupg.org/) and [sops](https://github.com/mozilla/sops):

```sh
brew install gnupg sops
```

Generate a GPG key for Flux without specifying a passphrase and retrieve the GPG key ID:

```console
$ gpg --full-generate-key
Email address: fluxcdbot@users.noreply.github.com

$ gpg --list-secret-keys fluxcdbot@users.noreply.github.com
sec   rsa3072 2020-09-06 [SC]
      1F3D1CED2F865F5E59CA564553241F147E7C5FA4
```

Create a Kubernetes secret in the `flux-system` namespace with the GPG private key:

```sh
gpg --export-secret-keys \
--armor 1F3D1CED2F865F5E59CA564553241F147E7C5FA4 |
kubectl create secret generic sops-gpg \
--namespace=flux-system \
--from-file=sops.asc=/dev/stdin
```

You should store the GPG private key in a safe place for disaster recovery,
in case you need to rebuild the cluster from scratch.
The GPG public key can be shared with the platform team, so anyone with 
write access to the platform repository can encrypt secrets.

### Git over SSH

Generate a Kubernetes secret with the SSH and known host keys:

```sh
flux -n apps create secret git dev-team-auth \
    --url=ssh://git@github.com/<org>/<dev-team> \
    --export > ./tenants/base/dev-team/auth.yaml
```

Print the SSH public key and add it as a read-only deploy key to the dev-team repository:

```sh
yq read git-auth.yaml 'data."identity.pub"' | base64 --decode
```

### Git over HTTP/S

Generate a Kubernetes secret with basic auth credentials:

```sh
flux -n apps create secret git dev-team-auth \
    --url=https://github.com/<org>/<dev-team> \
    --username=$GITHUB_USERNAME \
    --password=$GITHUB_TOKEN \
    --export > ./tenants/base/dev-team/auth.yaml
```

The GitHub token must have read-only access to the dev-team repository.

### Configure Git authentication

Encrypt the `dev-team-auth` secret's data field with sops:

```sh
sops --encrypt \
    --pgp=1F3D1CED2F865F5E59CA564553241F147E7C5FA4 \
    --encrypted-regex '^(data|stringData)$' \
    --in-place ./tenants/base/dev-team/auth.yaml
```

Create the sync manifests for the tenant Git repository referencing the `git-auth` secret:

```sh
flux create source git dev-team \
    --namespace=apps \
    --url=https://github.com/<org>/<dev-team> \
    --branch=main \
    --secret-ref=dev-team-auth \
    --export > ./tenants/base/dev-team/sync.yaml

flux create kustomization dev-team \
    --namespace=apps \
    --service-account=dev-team \
    --source=GitRepository/dev-team \
    --path="./" \
    --export >> ./tenants/base/dev-team/sync.yaml
```

Create the base kustomization.yaml file:

```sh
cd ./tenants/base/dev-team/ && kustomize create --autodetect
```

Configure Flux to decrypt secrets using the `sops-gpg` key:

```yaml
flux create kustomization tenants \
  --depends-on=kyverno-policies \
  --source=flux-system \
  --path="./tenants/staging" \
  --prune=true \
  --interval=5m \
  --validation=client \
  --decryption-provider=sops \
  --decryption-secret=sops-gpg \
  --export > ./clusters/staging/tenants.yaml
```

With the above configuration, the Flux instance running on the staging cluster will:

* create the tenant namespace, service account and role binding
* decrypt the tenant Git credentials using the GPG private key
* create the tenant Git credentials Kubernetes secret in the tenant namespace
* clone the tenant repository using the supplied credentials
* apply the `./staging` directory from the tenant's repo using the tenant's service account
