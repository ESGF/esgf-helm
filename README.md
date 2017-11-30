# esgf-helm

This project provides a Helm chart for deploying `esgf-docker` components on
Kubernetes/OpenShift.

## Deploying on OpenShift

### Running a local cluster for testing

Getting a local OpenShift cluster for testing is really simple. First, install
[Docker](https://docs.docker.com/engine/installation/) and the
[OpenShift CLI](https://www.openshift.org/download.html). Then run:

```bash
$ oc cluster up
```

By default, OpenShift is extremely secure, and overrides the `USER` set in the
`Dockerfile` with a UID from a range allocated to the project. The ESGF containers
expect to be able to run as the user specified in the `Dockerfile`, and some
must be allowed to run as root.

To relax this to something more akin to Kubernetes default behaviour, run the
following command as a cluster admin:

```bash
$ oc adm policy add-scc-to-group anyuid system:authenticated
```

### Installing Helm

Tiller (the in-cluster component of Helm) can be installed on a per-project
basis in OpenShift using the following commands:

```bash
# Create a service account for tiller to use
$ oc create sa tiller
# Allow the tiller service account to do anything within the project
$ oc policy add-role-to-user admin -z tiller
# Set the tiller namespace to the current project for all subsequent helm commands
$ export TILLER_NAMESPACE="$(oc config current-context | cut -d "/" -f 1)"
# Install tiller using the given service account, and force it to run as a non-root user
$ helm init --service-account tiller
```

### Creating the ESGF service account

The `esgf-helm` chart uses Helm `pre-install` and `post-delete` jobs to manage
`ConfigMap` resources containing the ESGF trusted certificates. These certificates
are downloaded at deployment time from the distribution site named in `values.yaml`.

Due to a [limitation with Helm pre-install hooks](https://github.com/kubernetes/helm/issues/3165),
the service account used to create these `ConfigMap`s must be created *before*
installing the chart.

```bash
# Create the service account
$ oc create sa esgf
# Bind the edit role to the service account
# Technically, the esgf user only needs to be able to manipulate ConfigMaps,
# but this is easier
# Making a more restrictive role is left as an exercise for the user :-)
$ oc policy add-role-to-user edit -z esgf
```
