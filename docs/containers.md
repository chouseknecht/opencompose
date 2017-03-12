# Containers

Containers are defined inside of a service, and generally a service will define a single container.

Consider the following example:

```yml
services:
  web:
    image: httpd:2.2
    ports:
    - 8000:80
    command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"]
```

which defines a single service, *web*, backed by a single container.

A service can also define multiple containers. Take a look at the following:

```yml
services:
  web:
    containers:
    - image: httpd:2.2
      ports:
      - 8000:80
      command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"]      
    - image: httpd:2.2
      ports:
      - 8080:80
      command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"]
```

In this case the `containers` directive is used, allowing a list of containers to be defined. The *web* service is now backed by two containers, one listening on port 8000, and one listening on port 8080.


## Container attributes

Use the following attributes to define a container:


### args

| type            | required |
|-----------------|----------|
| list of string  |    no    |

Specify a list of arguments to pass to the command. You can include arguments in the `command` directive, or enumerate them with `args`. 


### command

| type           | required |
|----------------|----------|
| list of string |    no    |

The command to run at container startup, overriding the `Cmd` specified on the image. 

For example:

```
command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"]
```

Each item within the `command` list must be a string.


### entrypoint

| type           | required |
|----------------|----------|
| list of string |    no    |

When Kubernetes starts a container, it runs the image’s default `entrypoint`, and passes the image’s default `Cmd` as arguments. Use the `entrypoint` directive to override the image entrypoint setting.

For example:

```
entrypoint: ["/usr/local/bin/entrypoint.sh"] 
```

Each item in within the `entrypoint` list must be a string.


### environment 

| type                             | required |
|---------------------------------|----------|
| list or dict of key=value pairs |    no    |

Provide a list or mapping of environment variables to be set inside the container.

An example of a list of environment variables:

```yml
environment:
  - REDIS_VERSION=2.1
  - REDIS_PATH=/options/share/redis/bin
  - QUEUE_NAME="worker queue"
```

An example of a mapping:

```yml
environment:
  REDIS_VERSION: "2.1"
  REDIS_PATH: /options/share/redis/bin
  QUEUE_NAME: worker queue
```


### expose 

| type            | required |
|-----------------|----------|
| list of strings |    no    |

Provide a list of ports on which the container listens for requests. Like the `ports` directive, the list of `expose` ports is added to the service's `ports` list. However, the ports are not  exposed to requests originating outside of the cluster. They are intended to be private, servicing internal requests only.

Specify each port in the format `port/protocol`, where protocol is optional, and can be one of *TCP* or *UDP*, defaulting to *TCP*.

For example: 

```yml
expose:
  - 8080
  - 8000/UDP
```

When deployed to K8S, exposed ports will be added to the `ports` list of the service.

Consider the following:

```yml
services:
  web: 
    image: httpd:2.2
    expose:
    - 8080
    - 8000/UDP
    command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"] 
```

which results in a service that includes the exposed ports:

```yml
apiVersion: v1
kind: Service
metadata:
  name: web
selector:
  web: web
spec:
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
    name: web-8080-TCP 
  - port: 8000
    targetPort: 8000
    protocol: UDP
    name: web-8000-UDP
```

### image

| type | required |
|------|----------|
|string|    yes   |

Provide a path to the image used to start the container. If the path does not include a registry, the default registry for the cluster will be used. 


### imagePull


### lifecycle


### liveness


### local_overrides

| type   | required |
|--------|----------|
| dict   |    no    |

Used to specify container attributes applicable only when the container is beging manipulated by a local container engine. This directive will be ignored during a deployment to K8S.

For example: 

```yml
version: 1.0-dev
services:
  web:
    ports:
    - 5000:80
    local_overrides:
      build: .
      volumes
        - .:/code
      links:
        - redis:redis
  redis:
    image: redis:latest
```

In the above, the directives found under `local_overrides` will be ignored by K8S. However, a local tool set or container engine will follow the `build`, `volume` and `links` directives.


### name

| type   | required |
|--------|----------|
| string |    no    |

Name of the container specified as a DNS_LABEL. Each container in a pod must have a unique name (DNS_LABEL). Cannot be updated.


### ports

| type            | required |
|-----------------|----------|
| list of strings |    no    |

The `ports` directive supports mapping an external port to a container port. The implication is that the port is intended to be exposed to users and services external to the cluster. Like `expose`, the list of `ports` will be added to the `ports` list of the service. However, the service will be exposed externally.

For each port, provide a mappings in the format `external_port:container_port/protocol`. Protocol is optional, and can be one of *UDP* or *TCP*, and defaults to *TCP*. If the `container_port` is missing, it will default to the external port.

Here are some examples: 

```yml
ports:
  - 8000:8000
  - 8080/UDP
  - 9000
```

Like `expose`, the list of `ports` will be added to the `ports` list of the service. 

Consider the following example:

```yml
services:
  web: 
    image: httpd:2.2
    ports:
    - 8080:80
    - 8000/UDP
    command: ["/usr/bin/dumb-init", "/usr/bin/httpd-foreground"] 
```

which results in a service that includes the exposed ports:

```yml
apiVersion: v1
kind: Service
metadata:
  name: web
selector:
  web: web
spec:
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: web-8080-TCP 
  - port: 8000
    targetPort: 8000
    protocol: UDP
    name: web-8000-UDP
```

Unlike `expose`, `ports` results in a service where the port is exposed externally. How the service is exposed will depend on the OpenCompose implementation, and the cloud provider. 


### readiness

### resources

### volumeMounts

### workdir

### securityContext

### stdin

### stdinOnce

### volumes

| type            | required |
|-----------------|----------|
| list of strings |    no    |

Mount volumes to a container by providing a list of containers, where each is in the format of `



### tty