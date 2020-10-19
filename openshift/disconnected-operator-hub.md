# Disconnected Operator Hub

The OperatorHub comprises a few components:

* The OpenShift Marketplace operator, under the `openshift-marketplace` namespace.
* The OperatorHub Catalog Image, that is actually a gRPC server that is queried by the marketplace operator.
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