### RHDH profile of Backstage operator

Deploys the Backstage operator with the RHDH profile configuration on Openshift (OLM) and vanilla Kubernetes. 
Contains Orchestrator plugin dependencies OOTB.

**TODOs:** 
- Add some automation (shell script or so)
- Add e2e tests (at least smoke)

See: https://github.com/redhat-developer/rhdh-operator/blob/main/docs/profiles.md#external-profiles

#### Tools:

**Common:**
- kubectl/oc
- kustomize (part of kubectl command)
- 
**For OLM (openshift):**
- operator-sdk
- opm 
- docker or podman 

#### Prerequisites

- Make sure rhdh-operator of needed version is built and pushed to the registry
- Check profile/kustomization.yaml resources to contain https://github.com/redhat-developer/rhdh-operator/config/profile/external for main branch or with ?ref=release-1.7 to use release branch

For OLM (openshift):
- Make sure manifests/backstage-operator.clusterserviceversion.yaml is updated with the correct metadata.name (it shoule be rhdh-operator.vX.Y.Z) and spec.version. The metadata.name is a key for the operator-sdk bundle command (must fit --package parameter).
- Make sure you have a valid image repository account (in this example it is quay.io/gazarenk) to push bundle and catalog images

#### Commands

**deploy**
`kubectl apply -k profile`

**deploy plugin-infra (if any)**
`kubectl apply -k profile/plugin-infra`

**OLM (Openshift):**

Here:
- the Package name is `rhdh-operator`

**create bundle**

Make sure controller image is built and pushed to the registry.

**Important**: parameter `--package rhdh-operator` should fit manifests/backstage-operator.clusterserviceversion.yaml-> metadata.name minus v{version}
`
kustomize build manifests | bin/operator-sdk generate bundle --kustomize-dir manifests --overwrite --package rhdh-operator --version=1.7.0 --channels=fast,fast-\${CI_X_VERSION}.\${CI_Y_VERSION} --default-channel=fast
`
**validate bundle**
`bin/operator-sdk bundle validate bundle`

**build bundle image**
**Here and later use 'podman' instead of 'docker' if needed, it is safe to use with the same commands.**
`
Replace `-t quay.io/gazarenk/rhdh-operator-bundle` with your repo
`
docker build --platform linux/amd64 -f bundle.Dockerfile -t quay.io/gazarenk/rhdh-operator-bundle --label quay.expires-after=14d .
`
**push bundle image**
Replace `-t quay.io/gazarenk/rhdh-operator-bundle` with your repo
`
docker push quay.io/gazarenk/rhdh-operator-bundle
`

**create catalog**

- Option1: Generate from scratch
Replace `-bundles quay.io/gazarenk/rhdh-operator-bundle` and `--tag quay.io/gazarenk/rhdh-operator-catalog` with your image
```
mkdir catalog
bin/opm generate dockerfile catalog
# rhdh-operator here is the name of package, ('rhdh' in prod)
bin/opm init rhdh-operator --default-channel=fast --output yaml > catalog/operator.yaml
# this generate package==rhdh-operator anyway (as generated from bundle)
bin/opm render quay.io/gazarenk/rhdh-operator-bundle:latest --output=yaml >> catalog/operator.yaml
cat << EOF >> catalog/operator.yaml
---
schema: olm.channel
package: rhdh-operator 
name: fast
entries:
- name: rhdh-operator.v1.7.0
  EOF
```
Note: render quay.io/gazarenk/rhdh-operator-bundle:latest => change WITH tag

- Option2: Use existing catalog
Check catalog/operator.yaml and modify if needed (make sure you understand the [FBC](https://olm.operatorframework.io/docs/tasks/creating-a-catalog/))

- If catalog Dockerfile is not generated use `bin/opm generate dockerfile catalog` where catalog is the catalog folder

Note: this sample catalog uses rhdh-operator package name
 
**build catalog image** 

Replace `-t quay.io/gazarenk/rhdh-operator-catalog` with your image name

`docker build --platform linux/amd64 -f catalog.Dockerfile -t quay.io/gazarenk/rhdh-operator-catalog --label quay.expires-after=14d .
`

**push catalog image**
Replace `-t quay.io/gazarenk/rhdh-operator-catalog` with your image name

`docker push quay.io/gazarenk/rhdh-operator-catalog
`

**update catalog** 
Note: spec.image: quay.io/gazarenk/rhdh-operator-catalog

`kubectl apply -n openshift-marketplace -f samples/catalog-source.yaml
`
**deploy with olm**
`
kubectl create namespace rhdh-operator  
kubectl apply -n rhdh-operator -f samples/operator-group.yaml
kubectl apply -n rhdh-operator -f samples/subscription.yaml
`

