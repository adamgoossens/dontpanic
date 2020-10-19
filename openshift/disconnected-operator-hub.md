# Disconnected Operator Hub

The OperatorHub comprises a few components:

* The OpenShift Marketplace operator, under the `openshift-marketplace` namespace.
* The OperatorHub Catalog Image. This is actually a gRPC server and SQLite database that is queried by the marketplace operator; this is what populates the OperatorHub section of the OpenShift Console.
* The collection of images used by the operators in the catalog

## The environment

This article assumes an air-gapped environment, where OperatorHub images must be somehow transferred to removable media, brought across an air-gap, and then uploaded to another host for subsequent use.

![Air-gapped architectural diagram](/_images/disconnected-operator-hub-concept.png)

## Pre-requisites

### Documentation

This document must be read in parallel with the official Red Hat OpenShift documentation on using the OperatorHub in restricted networks, [available here](https://docs.openshift.com/container-platform/4.5/operators/admin/olm-restricted-networks.html).

### Physical hosts

This guide expects two VMs or physical machines, one on either side of your airgap. The hostnames can be whatever you like, but for this article I use `connected.home.lab` and `disconnected.home.lab` to illustrate which side of the air-gap they will be on.

* `connected.home.lab` - this is Internet-connected and will be where you run your `oc adm release mirror` and `oc adm catalog mirror` commands from.
* `disconnected.home.lab` - this will be the destination host for your data transfer, and will be where you then push the mirrored content to a destination registry.

### Air-gapped registry

In this article I assume that you already have an official destination for your air-gapped containers, e.g. Red Hat Quay.

If you do not, then you can also use the Docker v2 registry we will be standing up on `disconnected.home.lab` as a temporary measure too. If this is you, then you can **skip step 6**, where we mirror everything to the destination registry.

## The process, high level

At a high level, the following steps are performed:

1. Stand up an Internet-connected Docker v2 registry as a target for OperatorHub image mirroring.
2. Create the OperatorHub bundle, mirroring it into the Docker v2 registry.
3. Mirror all of the OperatorHub images into the Docker v2 registry.
4. Save a copy of `docker.io/library/registry:2` to disk`.
5. Transfer the data directory of the registry across the air-gap. Also include the ImageContentSourcePolicy, the mappings.txt file, and your `registry:2` image.
6. On the air-gapped host, stand up a new Docker v2 registry and mount the transferred data directory.
7. Using mappings.txt, create a new source-to-destination mapping file with the source as the air-gapped host, and the destination as the final corporate registry.
8. Use `oc image mirror` or `skopeo copy --all` to perform the copy.

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

```
[root@connected bin]# cat catalog-mirror
#!/bin/bash

LOCAL_SECRET_JSON=/root/pull.json

oc adm -a ${LOCAL_SECRET_JSON} catalog mirror \
  connected.home.lab:5000/olm/redhat-operators:v1 \
  connected.home.lab:5000 \
  --filter-by-os=".*"

# get a coffee, maybe lunch and dinner too, because this will take a long time.
[root@connected bin]# catalog-mirror
```

### --manifests-only, or not?

When using the `oc adm catalog mirror` command, you have the option of providing `--manifests-only=true` to the command. This will result in only the `imageContentSourcePolicies.yaml` and `mapping.txt` files being written to disk; no actual mirroring will take place.

Use this when you want to customise the `mapping.txt` file to reduce the mirroring overhead.

**Note**: if you do not do manifests only mirroring, you **must** set `--filter-by-os=".*"`, as discussed next.

### --filter-by-os - watch out

Whichever command performs the actual image mirroring (e.g. `oc adm catalog mirror` or `oc image mirror`), this command must have `--filter-by-os` specified and it **must** be set to `--filter-by-os=".*"`. This is to ensure that the mirroring code mirrors the manifest lists in their entirety - the Operators in the catalog generally refer to the manifest list digests, *not* to the digest of a particular image architecture.

There is a design gotcha in the mirroring library, whereby if you specify `--filter-by-os="linux/amd64` it will build a **new** manifest list that contains only the `linux/amd64` architecture image. This new manifest list will have a completely different sha256 digest from the upstream manifest list, and so the operator that references the upstream manifest list will fail to run.

In short, `--filter-by-os=".*"` is the way to go for now.

## Step 4: bring a copy of the Docker v2 registry container

If you do not have it already, you will need the `docker.io/library/registry:2` container in your air-gapped environment.

Your simplest course of action is to `podman save` this to your removable media, transfer it over the airgap, then `podman load` followed by `podman run` on the other side of the air-gap.

```
[root@registry ~]# podman save docker.io/library/registry:2 -o registry_2_image.tar
```

## Step 5: data transfer

Using whatever data transfer process applies in your environment, bring the entire contents of the directory you mounted to `/var/lib/registry` in the container, plus `imageContentSourcePolicy.yaml`, `mappings.txt` and `registry:2` tarball, across the air-gap.

From here on I will assume you have them both available somewhere on your air-gapped host.

This air-gapped host will be `disconnected.home.lab`.

## Step 6: Stand up your docker v2 registry

If you don't have it in the environment, `podman load` your `registry:2` tarball and then `podman run` it, just like you did on the connected side. 

Use TLS certificates and/or authentication as required by your security policy.

Make sure you mount your mirrored registry data directory to `/var/lib/registry` in the container.

```
[root@disconnected ~]# podman load < registry_2_image.tar
Getting image source signatures
Copying blob b3f465d7c4d1 skipped: already exists
Copying blob 3e207b409db3 skipped: already exists
Copying blob 239a096513b5 skipped: already exists
Copying blob a5f27630cdd9 skipped: already exists
Copying blob f5b9430e0e42 skipped: already exists
Copying config 2d4f4b5309 done
Writing manifest to image destination
Storing signatures
Loaded image(s): docker.io/library/registry:2
```

You will now have all of your mirrored OperatorHub images, and associated catalog, available in your air-gapped environment.

The next step is to push them to a destination registry, assuming you aren't using your Docker v2 registry on your air-gapped destination host.

## Step 7: Customise mappings.txt

The mappings.txt file has the following basic structure. Each line is one container image, and the line is of the form:

```
source.registry/repository/image@sha256:<digest>=destination.registry/repository/image:<tag>
```

The source image may also be a tag, but for air-gapped operators **only images referenced by digest are supported**.

We will build a new mapping.txt file where we change the source registry to `disconnected.home.lab` and the destination registry to our official container registry in the air-gapped environment - here, that'll be `quay.air.gapped`.

Some basic bash magic is all that's required:

```
[root@disconneccted tmp]# UPSTREAM_MIRROR_HOST="connected.home.lab:5000"
[root@disconneccted tmp]# SOURCE_REGISTRY="disconnected.home.lab:5000"
[root@disconneccted tmp]# DESTINATION_REGISTRY="quay.air.gapped"
[root@registry tmp]# sed -e "s/$UPSTREAM_MIRROR_HOST/$DESTINATION_REGISTRY/" mappings.txt | sed -r -e "s#^[^/]+/(.*)#${SOURCE_REGISTRY}/\1#g" > mappings-new.txt
```

Also, add your operator bundle image into the mapping file - we need this to be pushed to the registry too:

```
[root@registry tmp]# echo "$UPSTREAM_MIRROR_HOST/olm/redhat-operators:v1=$DESTINATION_REGISTRY/olm/redhat-operators:v1" >> mappings-new.txt
```

From here, we can take `mappings-new.txt` and feed it into `oc image mirror`:

```
[root@disconnected tmp]# oc image mirror -f ./mappings-new.txt --filter-by-os=".*"
```

Everything should be mirrored into your final destination registry. **Note**: don't forget `--filter-by-os=".*"`.

## Step 8: Update imageContentSourcePolicy.yaml

We will also need to change the mirrors listed in `imageContentSourcePolicy.yaml`, as these will be set to your upstream mirroring host. Again, `sed` to the rescue:

```
[root@disconnected tmp]# sed -e "s/$UPSTREAM_MIRROR_HOST/$DESTINATION_REGISTRY/g" imageContentSourcePolicy.yaml  > imageContentSourcePolicy-new.yaml
```

## Step 9: Continue with OperatorHub deployment.

Follow the remaining steps [here](https://docs.openshift.com/container-platform/4.5/operators/admin/olm-restricted-networks.html#olm-restricted-networks-operatorhub_olm-restricted-networks) to:

* Disable existing OperatorHub sources.
* Create your `ImageContentSourcePolicy` on your cluster - note, be prepared for all of your nodes to reboot while the MachineConfigOperator rolls out the changes
* Create a new `CatalogSource` to point at the bundle image that you mirrored to your disconnected registry.
* Start using the OperatorHub!