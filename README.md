# esgf-helm

This project provides a Helm chart for deploying `esgf-docker` components on
Kubernetes/OpenShift.


## Quickstart

First, install [Minishift](https://www.openshift.org/minishift/) and the
[Helm](https://helm.sh/) CLI.

Then start a MiniShift cluster with plenty of RAM:

```sh
minishift start --openshift-version v3.7.1 --memory 8GB --disk-size 100GB
eval $(minishift oc-env)
```

By default, OpenShift is extremely secure, and overrides the `USER` set in the
`Dockerfile` with a UID from a range allocated to the project. The ESGF containers
expect to be able to run as the user specified in the `Dockerfile`, and some
must be allowed to run as root.

To relax this to something more akin to Kubernetes default behaviour, run the
following commands.

```sh
oc login -u system:admin
oc adm policy add-scc-to-group anyuid system:authenticated
oc login -u developer
```

Install a Tiller to manage the current project/namespace:

```sh
oc create sa tiller
oc policy add-role-to-user admin -z tiller
export TILLER_NAMESPACE="$(oc config current-context | cut -d "/" -f 1)"
helm init --service-account tiller
# Wait for the tiller pod to start
oc get pods
```

Configure and deploy ESGF components:

```sh
export ESGF_HOSTNAME="esgf.$(minishift ip).xip.io"
# This should be an empty directory
export ESGF_CONFIG=/path/to/esgf/config
docker run -v "$ESGF_CONFIG":/esg -e ESGF_HOSTNAME cedadev/esgf-setup generate-secrets
docker run -v "$ESGF_CONFIG":/esg -e ESGF_HOSTNAME cedadev/esgf-setup generate-test-certificates
docker run -v "$ESGF_CONFIG":/esg -e ESGF_HOSTNAME cedadev/esgf-setup create-trust-bundles
docker run -v "$ESGF_CONFIG":/esg -e ESGF_HOSTNAME cedadev/esgf-setup create-helm-config
helm install -f "$ESGF_CONFIG/helm/values.yaml" -n esgf .
```
