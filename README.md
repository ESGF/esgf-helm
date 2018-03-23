# esgf-helm

This project provides a Helm chart for deploying `esgf-docker` components on
Kubernetes/OpenShift.


## Quickstart

First, install [Minishift](https://www.openshift.org/minishift/) and the
[Helm](https://helm.sh/) CLI.

Then start a MiniShift cluster with plenty of RAM:

```sh
minishift start --openshift-version v3.7.1 --memory 8GB --disk-size 100GB
eval $(minishift oc-env)
```

Pods must be able to loop back to themselves via their own service - this is known
as "hairpin mode", and requires the docker bridge network to be in promiscuous
mode:

```sh
minishift ssh -- sudo ip link set docker0 promisc on
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
# Generate configuration
docker-compose run esgf-setup generate-secrets
docker-compose run esgf-setup generate-test-certificates
docker-compose run esgf-setup create-trust-bundle
# Deploy the ESGF components
docker-compose run -T esgf-setup helm-values | helm upgrade esgf . --install -f -
```

Publish the test dataset (similar to [esgf-docker](https://cedadev.github.io/esgf-docker/usage/publishing/)):

```sh
# Scale up the publisher deployment
$ oc scale --replicas=1 deployment esgf-publisher
# Wait for the publisher pod to start and get the pod name
# The first time you scale up is when the image gets pulled, so this might take
# a long time
$ oc get pods -w -l "component=publisher"
# Open a shell inside the pod
$ oc exec -it <publisher pod name> bash
# Download the data
[publisher] $ mkdir -p /esg/data/test
[publisher] $ wget -O /esg/data/test/sftlf.nc http://distrib-coffee.ipsl.jussieu.fr/pub/esgf/dist/externals/sftlf.nc
# Make sure the correct SSL_CERT_FILE is being used
[publisher] $ export SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
# Fetch a certificate
[publisher] $ fetch-certificate
# Publish the data
[publisher] $ esgprep mapfile --project test /esg/data/test
[publisher] $ esgpublish --project test --map mapfiles/test.test.map --service fileservice
[publisher] $ esgpublish --project test --map mapfiles/test.test.map --service fileservice --noscan --thredds
[publisher] $ esgpublish --project test --map mapfiles/test.test.map --service fileservice --noscan --publish
# Exit the publisher pod
[publisher] $ exit
# Scale the deployment back down to remove the pod
$ oc scale --replicas=0 deployment esgf-publisher
```
