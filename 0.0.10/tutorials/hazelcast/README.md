# Hazelcast Cluster Example

## Preparation
- Assume you have the following software installed:
    - kubectl (and oc if running on OpenShift)
    - operator-sdk
    - [kubectl-mvmcli](../../scripts/kubectl-mvmcli)

- Follow instructions [here](../../README.md) to
    - label the worker node for Memory Machine license server
    - label each worker node that you want to use with Memory Machine

## Mirror Images If Using Your Own Private Container Registry
- If you would like to host all images in your own private container registry, script [`mirror_images.sh`](../../scripts/mirror_images.sh) can help.
```
$ mirror_images.sh --image-repo <your-private-container-registry>/<repo>
```

## Deploy Memory Machine
```
$ install-operator.sh --bundle-image ghcr.io/memverge/memory-machine-operator-bundle:0.0.10 \
    --mm-license <path-to-your-license-file>
```
If you use your own private container registry, specify both bundle images and index images from your registry:
```
$ install-operator.sh --bundle-image <your-private-container-registry>/<repo>/memory-machine-operator-bundle:0.0.10 \
    --index-image <your-private-container-registry>/<repo>/opm:latest
    --mm-license <path-to-your-license-file>
```

NOTE: A SecurityContextConstraints with SELinuxContext strategy being RunAsAny is needed to allow injecting SELinux level to the application pods on OpenShift. An example is [`scc.yml`](../../scripts/scc.yml). Since SecurityContextConstraints is global, `scc.yml` needs to be applied only once.
(For Kubernetes, there is no need to apply `scc.yml` since there is no SecurityContextConstraints on Kubernetes.)

## Config Pmem
Follow instructions [here](../../README.md#configuring-pmem) to config Pmem.
If you use your own private container registry, specify image repo and pull secret:
```
$ config-pmem.sh --pmem-emulation --mm-capacity <pmem-size> \
    --pmem-csi-image <your-private-container-registry>/<repo>/pmem-csi-driver:v1.0.2 \
    --pause-image <your-private-container-registry>/<repo>/pause \
    --namespace memverge \
    --pull-secret memverge-dockerconfig
```

## Deploy Hazelcast Cluster (StatefulSet)
```
$ deploy_hazelcast.sh
```
If you use your own private container registry, change spec `imageRepository` and `etcdConfig:image` in [`memorymachine.yml`](memorymachine.yml).

NOTE: It is better to NOT use `:latest` tag for the application container image, because it is usually not stable. 
If it changed before restoring a snapshot, the restored container starts with a different image, which may cause restore failure.
Use a stable tag such as versioned tag instead.

NOTE: To make the Hazelcast cluster working properly after snapshot/restore, the following configurations are required:
- Hazelast's REST API access should be enabled. The REST endpoint groups `HEALTH_CHECK` and `CLUSTER_WRITE` should be enabled.
We need them to query and change the state of the Hazelcast cluster before/after snapshot/restore.
- A `ClusterIP` Service is needed to provide in-cluster DNS names for Hazelcast pods.
Since the pod IP addresses may change after restore, the Hazelcast members might be unable to join together after restore.
We use the service-provided DNS to solve the ip change issue.
- If the Readiness probe is used in the application container, the `ClusterIP` Service should have `publishNotReadyAddresses` enabled,
to allow the pod DNS accessible before the Readiness probe reporting the container ready.
- Hazelcast's `tcp-ip` discovery mechanism should be used, with the service-provided DNS name of each member listed.
In this way, the Hazelcast members can still recognize each other after restore.
- Each Hazelcast member should have its local address set as its service-provided DNS name. 
It can be configured via `hazelcast.local.localAddress` in the environment variable `JAVA_OPTS`.
In this way, the name of each Hazelcast member can keep unchanged after restore. (By default, the IP address is used, which may change after restore.)

Please see [`statefulset.yml`](statefulset.yml) for example.

## Show Memory Machine Usage
```
$ kubectl mvmcli usage
```

## List Cluster Members Running on Memory Machine
```
$ kubectl mvmcli list
```

## Add Data into Hazelcast Cluster
```
$ kubectl -n demo exec --it hz-client -- bash
$ python /home/client.py --help
```

## Snapshot Hazelcast Cluster (StatefulSet)
```
$ kubectl mvmcli clustersnap create hz-snap-test --namespace demo --statefulset hz --profile hazelcast --profile-options '--nopause'
```

## List Snapshots Have Taken
```
$ kubectl mvmcli clustersnap list
```

## Restore Hazelcast Cluster (StatefulSet) From a Snapshot
```
$ kubectl mvmcli clustersnap restore hz-snap-test
```

## Delete Cluster Snapshots
```
$ kubectl mvmcli clustersnap delete hz-snap-test
```