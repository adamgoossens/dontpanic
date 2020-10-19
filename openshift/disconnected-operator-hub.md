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