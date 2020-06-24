# Quotas and Limits Demo

### Setup

This demo was prepared for an audience using OpenShift 3.11.  In this demo, we will show quotas and limits applied to projects and we will spin up some sample applications to provide workloads.  To set up our application, we will import the rhel7 image into the openshift namespace.

If necessary, set up your [registry authentication] first, to allow pulling images from registry.redhat.io.

```
oc project openshift
oc create secret generic my-pull-secret --from-file=.dockerconfigjson=/root/.docker/config.json --type=kubernetes.io/dockerconfigjson
oc import-image rhel7 --from=registry.redhat.io/rhel7/rhel --confirm
```

### Quotas and Limits with Httpd Application

Create a new project, set up upstream pull secret:
```
oc new-project httpd
oc create secret generic my-pull-secret --from-file=.dockerconfigjson=/root/.docker/config.json --type=kubernetes.io/dockerconfigjson
oc secrets link builder my-pull-secret
```

Apply our limit ranges to the project.  For some more reading, refer to the docs on [limit ranges].
```
oc create -f resources.limitrange.yaml
```

Apply our quotas to the project.  For some more reading, refer to the docs on [quotas].
```
oc create -f resources.resourcequota.yaml
```

Let's inspect our build config to show it does not have resource limits defined.
```
less sample-httpd.bc.yaml
```

Create an image stream and build config.  Note that the build pod inherits the default limit ranges applied to the project.  Once the build pod completes, it no longer consumes quota.
```
oc create -f sample-httpd.is.yaml ### or 'oc create is sample-httpd'
oc create -f sample-httpd.bc.yaml
```

Let's inspect our deployment config to show the app pods have resource limits defined.  Create a deployment config which spins up two pod replicas.
```
less sample-httpd.dc.yaml
oc create -f sample-httpd.dc.yaml
```

Hey, we ran out of quota, and can't get the second replica started!  What went wrong?
```
# oc get pods
NAME                    READY     STATUS      RESTARTS   AGE
sample-httpd-1-build    0/1       Completed   0          6m
sample-httpd-1-deploy   1/1       Running     0          4m
sample-httpd-1-gs96v    1/1       Running     0          4m
```

The deploy pod `sample-httpd-1-deploy` also inherits the default limit ranges applied to the project.  Once it completes, it no longer consumes quota.

### Multi-project Quotas

For some more reading, refer to the docs on [multi-project quotas].

Let's clean up our previous quota first
```
oc project httpd
oc delete resourcequota resources
```

Create a service account to own the multi-project quota
```
oc create sa clusterquota
```

Create a ClusterResourceQuota with an annotation of `openshift.io/requester: system:serviceaccount:httpd:clusterquota`
```
oc create -f resources.clusterresourcequota.yaml
```

Annotate the projects to be governed by multi-project quota
```
oc patch namespace httpd -p '{"metadata":{"annotations":{"openshift.io/requester":"system:serviceaccount:httpd:clusterquota"}}}'
```

Let's create another project under this same quota
```
oc new-project cronjob
oc create secret generic my-pull-secret --from-file=.dockerconfigjson=/root/.docker/config.json --type=kubernetes.io/dockerconfigjson
oc create -f resources.limitrange.yaml
oc patch namespace cronjob -p '{"metadata":{"annotations":{"openshift.io/requester":"system:serviceaccount:httpd:clusterquota"}}}'
oc create -f pi.cronjob.yaml
```

Note that the cluster quotas are shared between the two projects, allowing our developers the visibility to right-size their pods and services between the two projects.

## License

GPLv3

## Author

Kevin Chung


[registry authentication]: https://access.redhat.com/RegistryAuthentication
[limit ranges]: https://docs.openshift.com/container-platform/3.11/dev_guide/compute_resources.html#dev-limit-ranges
[quotas]: https://docs.openshift.com/container-platform/3.11/dev_guide/compute_resources.html#dev-quotas
[multi-project quotas]: https://docs.openshift.com/container-platform/3.11/admin_guide/multiproject_quota.html
