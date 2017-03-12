# Pods

A Pod is a collection of containers. 

Typically Pods are defined implicitly within the context of a service. When a service is deployed to K8S, any associated containers are automatically encapsulated in a Pod

However, there are times when it is convenient to define Pods directly, and OpenCompose provides that ability. 

Consider the following example:

```yml
version: 1.0-dev
pods:
  web:
    labels:
      app: web
      level: development
    containers:
    - image: acmeco/web:2.0
      environment:
      - STATIC=/var/www/public/static
      command: ["/usr/bin/httpd-foreground"]
      expose:
      - 8080
```

which creates a Pod named *web*, encapsulating a single container listening on port 8080.


## Pod attributes 

The following attributes can be used to describe Pods:


### annotations

| type | required |
|------|----------|
| dict |    no    | 

Arbitrary key:value pairs added to the metadata of the Pod.

Unlike labels, the values can be structured or unstructured, and may contain special characters. Annotations are not used by K8S to identify or select objects.


#### containers

| type                | required |
|---------------------|----------| 
| list of containers  |    yes   |

Provide a list of containers to include in the pod. See [Containers](#containers) for available container attributes.


### labels

| type | required |
|------|----------|
| dict |    no    | 

Arbitrary key:value pairs added to the metadata of the Pod.

Labels are for grouping and organizing. Both the key and value must be of type string, and cannot contain special characters. Labels can be used by K8S to identify and select objects. 


