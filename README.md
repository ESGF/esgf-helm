# esgf-helm

This project provides a Helm chart for deploying `esgf-docker` components on
Kubernetes/OpenShift.


## Quickstart using Minikube

First, install [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)
and the [Helm](https://helm.sh/) CLI.

Then start a Minikube cluster with plenty of RAM. For now, we need to downgrade
the Kubernetes version due to [this bug](https://github.com/kubernetes/kubernetes/issues/61076).
See `minikube get-k8s-versions` for the available versions:

```sh
minikube start --kubernetes-version v1.10.0 --memory 8096 --disk-size 100GB
```

Pods must be able to loop back to themselves via their own service - this is known
as "hairpin mode", and requires the docker bridge network to be in promiscuous
mode:

```sh
minikube ssh -- sudo ip link set docker0 promisc on
```

Install Tiller:

```sh
helm init
# Wait for the tiller pod to start
kubectl -n kube-system get pods -w -l "name=tiller"
```

Install the Nginx ingress controller using Helm. Unfortunately, we can't use
`minikube addons enable ingress` because we need an additional flag to enable
SSL passthrough, which is required to use SSL client certificates. By default,
the Nginx ingress controller also uses a `308` response to redirect from `http`
to `https`. While, strictly speaking, `308` is better than `301`, it is a
(relatively) recent standard that breaks `urllib2`, and hence CoG:

```sh
# Make sure to update the stable channel before installing
helm repo update stable
helm install stable/nginx-ingress \
  --namespace kube-system \
  --name ingress \
  --set controller.hostNetwork=true \
  --set "controller.extraArgs.report-node-internal-ip-address=" \
  --set "controller.extraArgs.enable-ssl-passthrough="
# Patch the configmap to make Nginx use 301 instead of 308 for redirects
# We currently can't do this with --set to the helm install command because it is an integer
kubectl -n kube-system patch configmap ingress-nginx-ingress-controller --patch "{\"data\": {\"http-redirect-code\": \"301\"}}"
# Wait for the containers to become available
kubectl -n kube-system get pods -w -l "release=ingress"
```

Configure and deploy ESGF components:

```sh
export ESGF_HOSTNAME="esgf.$(minikube ip).xip.io"
# This should be an empty directory
export ESGF_CONFIG=/path/to/esgf/config
# Generate configuration
docker-compose run esgf-setup generate-secrets
docker-compose run esgf-setup generate-test-certificates
docker-compose run esgf-setup create-trust-bundle
# Deploy the ESGF components
docker-compose run -T esgf-setup helm-values | helm upgrade esgf . --install --namespace esgf -f -
```

You can inspect the state of the pods using `kubectl -n esgf get pods`, or by launching the
dashboard using `minikube dashboard`.

Once all the containers are running, visit `https://$ESGF_HOSTNAME` to see the CoG interface.
Try the following as a basic test of functionality:

  *  Log in with the `rootAdmin` account from CoG (OpenID `https://$ESGF_HOSTNAME/esgf-idp/openid/rootAdmin`)
     using the password from the `esgf-cog-secrets` secret in Kubernetes, or the file
     `$ESGF_CONFIG/secrets/rootadmin-password`
  * Log in with the `rootAdmin` account from the ORP (`https://$ESGF_HOSTNAME/esg-orp`)
  * Check THREDDS is running at `https://$ESGF_HOSTNAME/thredds`
  * Check Solr is running at `https://$ESGF_HOSTNAME/solr`


Publish the test dataset (similar to [esgf-docker](https://esgf.github.io/esgf-docker/usage/publishing/)):

```sh
# Scale up the publisher deployment
$ kubectl -n esgf scale --replicas=1 deployment esgf-publisher
# Wait for the publisher pod to start and get the pod name
# The first time you scale up is when the image gets pulled, so this might take
# a long time
$ kubectl -n esgf get pods -w -l "component=publisher"
# Start a shell inside the publisher pod
$ PUBLISHER_POD="$(kubectl -n esgf get pods -l "component=publisher" -o name | cut -d "/" -f 2)"
$ kubectl -n esgf exec -it "$PUBLISHER_POD" /usr/local/bin/docker-entrypoint.sh bash
# Download the data
[publisher] $ mkdir -p /esg/data/test
[publisher] $ wget -O /esg/data/test/sftlf.nc http://distrib-coffee.ipsl.jussieu.fr/pub/esgf/dist/externals/sftlf.nc
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
$ kubectl -n esgf scale --replicas=0 deployment esgf-publisher
```

After allowing time for the data to replicate from the master to the slave, check
that search is working at `https://$ESGF_HOSTNAME/search/testproject/` and that
the data is accessible from THREDDS.


## Quickstart using Minishift

First, install [Minishift](https://www.openshift.org/minishift/) and the
[Helm](https://helm.sh/) CLI.

Then start a Minishift cluster with plenty of RAM:

```sh
minishift start --openshift-version v3.9.0 --memory 8GB --disk-size 100GB
eval $(minishift oc-env)
```

Pods must be able to loop back to themselves via their own service - this is known
as "hairpin mode", and requires the docker bridge network to be in promiscuous
mode:

```sh
minishift ssh -- sudo ip link set docker0 promisc on
```

Because of the way OpenShift manages permissions, we need a Tiller per project.
So install a Tiller to manage the current project/namespace:

```sh
oc create sa tiller
oc policy add-role-to-user admin -z tiller
export TILLER_NAMESPACE="$(oc project -q)"
helm init --service-account tiller
# Wait for the tiller pod to start
oc get pods -w -l "name=tiller"
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
docker-compose run -T esgf-setup helm-values | \
  helm upgrade esgf . --install --set "proxy.ingressMode=openshift" -f -
```

Publish the test dataset (similar to above for `minikube`):

```sh
# Scale up the publisher deployment
$ oc scale --replicas=1 deployment esgf-publisher
# Wait for the publisher pod to start and get the pod name
# The first time you scale up is when the image gets pulled, so this might take
# a long time
$ oc get pods -w -l "component=publisher"
# Start a shell inside the publisher pod
$ PUBLISHER_POD="$(oc get pods -l "component=publisher" -o name | cut -d "/" -f 2)"
$ oc exec -it "$PUBLISHER_POD" /usr/local/bin/docker-entrypoint.sh bash
[publisher] $ # ... follow commands from minikube example ...
# Exit the publisher pod
[publisher] $ exit
# Scale the deployment back down to remove the pod
$ oc scale --replicas=0 deployment esgf-publisher
```
