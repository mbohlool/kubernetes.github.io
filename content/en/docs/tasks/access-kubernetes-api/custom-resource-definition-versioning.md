---
title: Extend the Kubernetes API with CustomResourceDefinitions
reviewers:
- mbohlool
- sttts
- liggitt
content_template: templates/task
---

{{% capture overview %}}
This page explains how
[CustomResourceDefinition](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#customresourcedefinition-v1beta1-apiextensions) versioning works and how to upgrade from one version to another. 
{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* Make sure your Kubernetes cluster has a master version of 1.11.0 or higher.

* Read about [custom resources](/docs/concepts/api-extension/custom-resources/).

{{% /capture %}}

{{% capture body %}}
Kubernetes standard API types has versioning to let developers extend them and roll out new features. To extend kubernetes with
Custom Resources, you should be able to do the same by supporting multiple versions of Custom Resources. The new API to support that has a new `versions` field to list all supported versions. Note the the previous `versions` field is deprecated and optional but if it is not empty, it must be the first item in the `versions` field.

For example, a CustomResourceDefinition with two versions would look like this:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
  - Name: v1
    # Each version can be enabled/disabled by Served flag.
    Served: true
    # One and only one version must be marked as storage version.
    storage: true
  - Name: v2
    Served: true
    storage: false
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

If you save the above definition to `my-versioned-crontab.yaml` and create it:

```shell
kubectl create -f my-versioned-crontab.yaml
```

By creating this CustomResourceDefinition, API server will add endpoints for each enabled version (e.g. /apis/stable.example.com/v1 and /apis/stable.example.com/v2).

## Version order
The order of the version define a priority for each version. Most importantly the version that comes first in term of priority
will be used by kubectl as the default version to access objects.
The version name in the CustomResourceDefiniton version list will be used to compute the order. If the version string is "kube-like", it will sort above non "kube-like" version strings, which are ordered
lexicographically. "Kube-like" versions start with a "v", then are followed by a number (the major version),
then optionally the string "alpha" or "beta" and another number (the minor version). These are sorted first
by GA > beta > alpha (where GA is a version with no suffix such as beta or alpha), and then by comparing
major version, then minor version. An example sorted list of versions:

```
- v10
- v2
- v1
- v11beta2
- v10beta3
- v3beta1
- v12alpha1
- v11alpha2
- foo1
- foo10.
```

For the above example, the version order would be v2, v1. That means any kubectl command will use v2 as default version unless
the version can be infered from the provided object.

## Conversion
Currently API Server does not do any conversions on CustomResourceDefinition other than fixing apiVersion of the object. The stored object will always have the storage version's apiVersion and return objects will have the same apiVersion as the request version.

If you save the following YAML to `my-crontab-v1.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

and create it:

```shell
kubectl create -f my-crontab.yaml
```

it will store the object with `stable.example.com/v1` API version. However when you access the object:

```shell
kubectl get my-new-cron-object -o yaml
```

you should see the object with `stable.example.com/v2` API version as `v2` is the prefered version.

```console
apiVersion: stable.example.com/v2
kind: CronTab
metadata:
  clusterName: ""
  creationTimestamp: 2017-05-31T12:56:35Z
  deletionGracePeriodSeconds: null
  deletionTimestamp: null
  name: my-new-cron-object
  namespace: default
  resourceVersion: "285"
  selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
  uid: 9423255b-4600-11e7-af6a-28d2447dc82b
spec:
  cronSpec: '* * * * */5'
  image: my-awesome-cron-image
```

## Stored Versions
While the object will always being stored with the version that has storage flag on, the CustomResourceDefinition can change
hence the storage version can change. To make sure we capture all storage versions, the CustomResourceDefinition Status object
has a `storedVersions` field that will have all versions ever marked as storage version. This field can be modified after you
made sure you upgraded all stored version. Assume you have the above CustomResourceDefinition with version v1 and v2 with v1 as storage version for a while. That means there are some objects stored as v1 in the storage backend and `storedVersions` will be a list of one item: `v1`. Now that we have `v2` this is the recommended process to upgrade storage to `v2`:

1. Set the `v2` as storage in the CustomResourceDefinition file and apply it using kubectl. After that the `storedVersions` would be `v1, v2`.
2. Set the `served` flag of `v1` to `false`. This will make sure no client will write `v1` objects.
3. Write a controller loop to list all existing objects and touch them (write them with the same content) in `v2` version. This will force the backend to write objects in `v2` version.
4. Update the CustomResourceDefinition by completely removing `v1` item.
5. Update CustomResourceDefinition `Status` by removing `v1` from `storedVersions` field.


{{% /capture %}}

