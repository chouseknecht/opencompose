# Services

In Kubernetes (K8S), a service is an internal load balancer, routing requests to containers. 

Requests may originate internally from other containers, or externally from end-users. The service will listen on one or more ports, and forward requests to the target ports of the appropriate containers based on their `port` and `expose` settings. Explicit port mappings can be defined for a service using the [servicePorts](#servicePorts) directive.

The `services` directive is a dictionary, defining each service in your application. A service will typically include a definition for a single container, or a set of multiple containers, that back the service and respond to incoming requests.

A simple service definition will look like the following:

```yml
services:
  web:
    image: httpd:2.2
    ports:
    - 8000:80
    command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"]
    
  database:
    image: postgresql:9.5
    entrypoint: ["/usr/bin/entrypoint.sh"]
    expose:
    - 5432
    command: ["/usr/bin/dumb-init", "postgresql"]
```

which defines two services, `web` and `database`, each backed by a single container. 


## Service attributes

Use the following attributes to define a service:


### annotations

| type | required |
|------|----------|
| dict |    no    | 

Arbitrary key:value pairs added to the metadata of the service, and any pods and containers defined within the service.

Unlike labels, the values can be structured or unstructured, and may contain special characters. Annotations are not used by K8S to identify or select objects.


### clusterIP

| type   | required |
|--------|----------|
| string |    no    | 

IP address of the service, usually assigned randomly by the master. If an address is specified manually and is not in use by others, it will be allocated to the service; otherwise, creation of the service will fail. This field can not be changed through updates.


### containers
| type  | required |
|-------|----------|
| array |   no     |

If the service will be backed by multiple containers, use the containers directive to define an array of containers. All of the containers included in the array will be deployed to a single [pod](#pods). 

If the service is backed by a single container, the `containers` directive is optional.
 
The following example defines a service backed by multiple containers:

```yml
services:
  web:
    containers:
    - image: "centos:7"
      command: ["/usr/bin/sleep", "1day"]
      ports:
      - 8000
    - image: "centos:7"
      ports:
      - 8080
      command: ["/usr/bin/sleep", "1day"]
```
      
Without the `containers` directive, only a single container can be defined for a service. The above example would then be reduced to the following:

```yml
services:
  web:
    image: "centos:7"
    command: ["/usr/bin/sleep", "1day"]
    ports:
    - 8000
```

**Note:** For more information regarding container attributes, see [Containrs](./containers).


### controller

| type |               choices              |   default   |
|------|------------------------------------|-------------|
| enum | Deployment, StatefulSet, DaemonSet |  Deployment | 

Choose the type of controller used to manage the pods backing the service. The default controller is `Deployment`. 


#### Deployment

TODO: Add descriptive and helpful words here.

#### StatefulSet

A StatefulSet is a Controller that provides a unique identity to its Pods. It provides guarantees about the ordering of deployment and scaling. 

StatefulSets provide:

- stable, unique network identifiers
- stable, persistent storage
- ordered, graceful deployment scaling
- ordered, graceful deletion and termination

For additional information about StatefulSet controllers, see [the StatefulSets ducumentation](https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/).

**NOTE:** StatefulSet is a beta resource, not available in any Kubernetes release prior to 1.5


#### DaemonSet

*Not Available*


### expose

| type | required | default |
|------|----------|---------|
| bool |    no    |  false  |

This is a convenience directive that will be implemented by tool set developers based on cloud platform and container engine capabilities. When set to `true`, the service should be exposed externally, which will cause the service properties to be set such that it acts as an external load balancer.

In the case of OpenShift, for example, this would cause a Route object to be created, which would serve to proxy external requests to the service.


### externalIPs

| type | required |
|------|----------|
| list |    no    | 

List of IP addresses for which nodes in the cluster will also accept traffic for this service. These IPs are not managed by Kubernetes.


### externalName 

| type   | required |
|--------|----------|
| string |    no    | 

The external reference that kubedns or equivalent will return as a CNAME record for this service. No proxying will be involved. Must be a valid DNS name and requires Type to be ExternalName.


### loadBalancerIP

| type   | required |
|--------|----------|
| string |    no    | 

Only applies when the `serviceType` is set to `LoadBalancer`. The load balancer will get created with the IP specified in this field. This feature depends on whether the underlying cloud-provider supports specifying the loadBalancerIP when a load balancer is created. This field will be ignored if the cloud-provider does not support the feature.


### loadBalancerSourceRanges

| type                | required |
|---------------------|----------|
| list of type string |    no    | 

If specified and supported by the platform, will restrict traffic through the cloud-provider load-balancer to the specified IP ranges.


### labels

| type | required |
|------|----------|
| dict |    no    | 

Arbitrary key:value pairs added to the metadata of the service, and any pods and containers defined within the service.

Labels are for grouping and organizing. Both the key and value must be of type string, and cannot contain special characters. Labels can be used by K8S to identify and select objects. 


### replicas

| type | required | default |
|------|----------|----------
| int  |    no    |    1    |

Total number of pods to create.

Consider the following example: 
 
```yml
services:
  web:
    replicas: 3
    image: "centos:7"
    command: ["/usr/bin/sleep", "1day"]
    ports:
    - 8000
    - 8080
```

It will result in a service named `web`, backed by 3 pods. Each pod will contain a single container, created from the *centos:7* image, and running the *sleep* command.


### selector:

| type | required |
|------|----------|
| dict |    no    |

A service definition, however, does not have to specifically define a container, nor a set of containers. Instead, it may include a `selector`, which is a map of key:value pairs used to match or select the pods to which it will route traffic. Any pods with a `label` definition matching the keys and values of the `selector` will be included.

Specify key:value pairs for selecting the pods to which incoming requests will be routed. All pods with `labels` containing matching keys and values will be selected. 

If the `selector` directive is present, then no pods or containers will be created for the service. It's expected that the pods and containers are defined elsewhere.


### serviceType ##

| type | choices required                                    | required | default   |
|------|-----------------------------------------------------|----------|-----------|
| enum | ExternalName, ClusterIP, NodePort, and LoadBalancer |    no    | ClusterIP |

Determines how the service is exposed. 


### servicePorts

| type   | required |
|--------|----------|
| list   |    no    |

When specified, each port is expected to be a *dictionary* or mapping comprised of several key:value pairs. Each port in the list may contain the following attributes: name, port, targetPort, external, and protocol. The only required attributes are port and targetPort. All other attributes have a default values. 

Example:

```yml
services:
  web:
    ports:
    - name: http 
      port: 8000
      targetPort: 8080
      protocol: TCP
```

`servicePorts` takes precedence in determining the port settings of the service. However, if `servicePorts` is not specified, the ports assigned to the service will be determined based on the containers defined within the service definition.

The following describes each of the attributes of a port mapping:

#### name

| type   | required |              default               |
|--------|----------|------------------------------------|
| string |   no     |  service_name-port_number-protocol |
 
The name of this port within the service. If not provided, a default name in the format *service_name-port_number-portocol* will be used.

If you choose to provide a nme, it must be a valid DNS_LABEL, and all port names within a service must be unique.


#### port

| type | required |
|------|----------|
| int  |   yes   |

Provide the port number on which the service listens for incoming connections.  


#### targetPort:

| type | required |
|------|----------|
| int  |   yes   |

Provide the port number on the pod to which traffic will be proxied. This will match a port exposed on a container within a pod backing the service.


#### protocol

| type   | required |  choices  | default |
|--------|----------|-----------|---------|
| enum   |    no    |  TCP, UDP |   TCP   |

Set the protocol expected by the service listening inside the container. Can be either `TCP` or `UDP`.


### sessionAffinity

|  type  | required |     choices    | default | 
|--------|----------|----------------|---------|
|  enum  |    no    | ClientIP, None |  None   |

Used to maintain session affinity. Enable client IP based session affinity. Must be ClientIP or None. Defaults to None.

