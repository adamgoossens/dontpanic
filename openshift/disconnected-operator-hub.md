# Disconnected Operator Hub

The OperatorHub comprises a few components:

* The OpenShift Marketplace operator, under the `openshift-marketplace` namespace.
* The OperatorHub Catalog Image. This is actually a gRPC server and SQLite database that is queried by the marketplace operator; this is what populates the OperatorHub section of the OpenShift Console.
* The collection of images used by the operators in the catalog

## The environment

This article assumes an air-gapped environment, where OperatorHub images must be somehow transferred to removable media, brought across an air-gap, and then uploaded to another host for subsequent use.

## The process, high level

At a high level, the following steps are performed:

1. Stand up an Internet-connected Docker v2 registry as a target for OperatorHub image mirroring.
2. Create the OperatorHub bundle, mirroring it into the Docker v2 registry.
3. Mirror all of the OperatorHub images into the Docker v2 registry.
4. Transfer the data directory of the registry across the air-gap. Also include the ImageContentSourcePolicy and mappings.txt file.
5. On the air-gapped host, stand up a new Docker v2 registry and mount the transferred data directory.
6. Using mappings.txt, create a new source-to-destination mapping file with the source as the air-gapped host, and the destination as the final corporate registry.
7. Use `oc image mirror` or `skopeo copy --all` to perform the copy.

## Step 1: stand up the Docker v2 registry.

On the internet-connected sync host, (referred to here as `connected.home.lab`), stand up the Docker v2 registry by using Podman. Use [this article](https://www.redhat.com/sysadmin/simple-container-registry) as your reference. Whether you put TLS certificates on it and/or credentials is up to you and your security considerations.

The end result needs to be a registry container running on `*:5000`, with a local filesystem directory mounted into the container under `/var/lib/registry`.

**Note: make sure the filesystem you mount to `/var/lib/registry` has at least 150GB of free space**.

Example:
```
[root@registry bin]# podman ps
CONTAINER ID  IMAGE                         COMMAND               CREATED       STATUS         PORTS  NAMES
afd23ce6d6e7  docker.io/library/registry:2  /etc/docker/regis...  2 months ago  Up 2 days ago         mirror-registry
```

```
[root@registry bin]# cat ~/run-registry.sh
#!/bin/bash
podman run \
  --rm  \
  --name mirror-registry \
  --net host \
  -p 5000:5000     \
  -v /etc/pki/tls:/etc/pki/tls:z \
  -v /opt/registry/data:/var/lib/registry:z \
  -v /opt/registry/auth:/auth:z \
  -e "REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true" \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/etc/pki/tls/registry.pem \
  -e REGISTRY_HTTP_TLS_KEY=/etc/pki/tls/private/registry.pem \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -d \
  docker.io/library/registry:2
```

## Step 2: Create the OperatorHub bundle image

The bundle image will hold all of the metadata and Kubernetes resources for all operators. This example we only mirror `redhat-operators`. We are going to mirror this bundle image directly into our Docker v2 registry we stood up on the host.

In OpenShift 4.5 and below, there is no easy way to adjust the bundle to include only a subset of the operators (e.g. to only those that work in air-gapped environments), and so the simplest solution is to bring across all of them.

In OpenShift 4.6 and above it will be possible to customise the bundle to prune out all operators except for ones you specify. When this is released, this guide will be updated.

Example:

```
[root@connected bin]# cat ~/bin/catalog-build
#!/bin/bash
LOCAL_SECRET_JSON=/root/pull.json

oc adm -a ${LOCAL_SECRET_JSON} catalog build \
  --appregistry-org redhat-operators \
  --from=registry.redhat.io/openshift4/ose-operator-registry:v4.5 \
  --filter-by-os="linux/amd64" \
  --to=connected.home.lab:5000/olm/redhat-operators:v1

[root@connected bin]# catalog-build
using registry.redhat.io/openshift4/ose-operator-registry:v4.5 as a base image for building
...
INFO[0047] directory                                     dir=/tmp/cache-254675220/manifests-688087437 file=web-terminal-_656cyb8 load=package
INFO[0047] directory                                     dir=/tmp/cache-254675220/manifests-688087437 file=1.0.1 load=package
Uploading ... 10.09MB/s
Pushed sha256:ccfb046baa4d3ad06f4e657d728c43d396d5eb7687e4272ef065be5e1aa70c37 to connected.home.lab:5000/olm/redhat-operators:v1
```

You now have a catalog bundle image available in `connected.home.lab:5000/olm/redhat-operators:v1`

## Step 3: Mirror the OperatorHub images

This is another step where while you *can* adjust it to mirror only the content you need, if you don't care about data transfer then it's quicker and simpler to just mirror the lot.

### --manifests-only, or not?

When using the `oc adm catalog mirror` command, you have the option of providing `--manifests-only=true` to the command. This will result in only the `imageContentSourcePolicies.yaml` and `mapping.txt` files being written to disk; no actual mirroring will take place.

Use this when you want to customise the `mapping.txt` file to reduce the mirroring overhead.

**Note**: if you do not do manifests only mirroring, you **must** set `--filter-by-os=".*"`, as discussed next.

### --filter-by-os - watch out

Whichever command performs the actual image mirroring (e.g. `oc adm catalog mirror` or `oc image mirror`), this command must have `--filter-by-os` specified and it **must** be set to `--filter-by-os=".*"`. This is to ensure that the mirroring code mirrors the manifest lists in their entirety - the Operators in the catalog generally refer to the manifest list digests, *not* to the digest of a particular image architecture.

There is a design gotcha in the mirroring library, whereby if you specify `--filter-by-os="linux/amd64` it will build a **new** manifest list that contains only the `linux/amd64` architecture image. This new manifest list will have a completely different sha256 digest from the upstream manifest list, and so the operator that references the upstream manifest list will fail to run.

In short, `--filter-by-os=".*"` is the way to go for now.
