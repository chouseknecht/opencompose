# Volumes

A top-level `volumes` directive is available for creating named volumes. A named volume can have a cloud configuration specified using the `persistent_volume_claim` directive, and a local configuration using `local_overrides`.

Consider the following example:

```yml
services:
  postgres:
    image: postgresql:9.5
    volumes:
      - pgdata:/var/lib/pgsql
      
volumes:
  pgdata: 
    persistent_volume_claim:
      storage: 5Gi
      accessModes:
      - ReadWriteOnce
    local_override: {}
```

which defines a *postgres* service, running a single container, with a volume named *pgdata* mounted to path */var/lib/pgsql*.

When deployed to K8S, the  named volume, *pgdata*, is created with a persistent volume claim (PVC). And, when deployed locally, the persistent volume claim settings are ignored, and the local container engine creates the volume according to its default settings.

  
## Volume attributes

Each volume in the top-level *volumes* directive is a mapping with the following attributes:

### local_overrides

| type  | required |
|-------|----------|
|  dict |    no    |


Use `local_overrides` to create the named volume when the container run under a local container engine, and use it to set any local container engine specific properties for the volume. In most cases you will set this to an empty mapping: `{}` to simply cause the volume to be created during a local container run.


### persistent_volume_claim:

| type  | required |
|-------|----------|
|  dict |    no    |

Define persistent_volume_claim when the volume should be created on K8S. Within persistent_volume_claim, the following attributes can be set:

 
#### storage

| type   | required | default |
|--------|----------|---------|
| string |    no    |   1Gi   | 

Specify the amount of storage needed. See [resource quantities](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resources.md#resource-quantities) for help with quantity abbreviations.

#### accessModes

| type           | required |     default     |
|----------------|----------|-----------------|
| list of string |    no    | [ReadWriteOnce] | 

Request storage with specific access modes. Valid values are:

- ReadWriteOnce – the volume can be mounted as read-write by a single node
- ReadOnlyMany – the volume can be mounted read-only by many nodes
- ReadWriteMany – the volume can be mounted as read-write by many nodes

#### selector:

| type  | required |
|-------|----------|
| dict  |    no    |

`selector` allows setting `matchLabels` and `matchExpressions`, where both provide ways to select which persistent volume is used to satisfy the claim.

`matchLabels` is a mapping of key:value pairs. The matching volume will have a `labels` attribute containing matching key:value paris.

`matchExpressions` is a list of requirements made by specifying a `key`, list of `values`, and an `operator` to relate the key and values. Valid operators include: In, NotIn, Exists, and DoesNotExist

Here's an example, taken from the K8S doc site:

```yml
selector:
  matchLabels:
    release: "stable"
  matchExpressions:
    - {key: environment, operator: In, values: [dev]}
```

For more information, view [Persistent Volume Claims](https://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims).

  