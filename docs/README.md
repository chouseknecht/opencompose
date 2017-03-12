# OpenCompose Reference Specification

**NOTE: None of what is described here has been implemented in the `opencompose` CLI.**

This is a specification only with the purpose of enabling the community to collaborate on defining the OpenCompose language. It's a work in progress, so expect frequent changes, and possible breakage. 

## Mission statement

The goal of OpenCompose is to create a simple container orchestration language that can potentially be consumed by any container orchestration platform or tool set, while putting Kuberneters (K8S) first. With OpenCompose developers will be able to describe complex, multi-container applications that can be deployed to K8S, or nearly any other container platform, without giving up access to critical K8S features.

Additionally, OpenCompose aims to make K8S configuration less verbose, and more approachable to new developers, without taking away access to all the knobs and dials experienced users expect. By limiting the verbosity, reducing unfamiliar syntax, and providing sane defaults, OpenCompose aims to reduce the learning curve, and lessen the option-overload often experienced by new users, without removing access to key features.

## OpenCompoe by example

To illustrate the vision of OpenCompose, and the supported syntax, we've come up with several examples. They range from simple to complex, demonstrating how `docker-compose` syntax is supported, and how few limitations there are when you need to manipulate K8S specific objects and settings.


### Example 1 - Create a service from an existing image

```yml
version: 1.0-dev
services:
  redis:
    image: redis:latest
    expose: 6379
    environment:
      REDIS_VERSION: 3.0.7
    workdir: /data
    command: ["redis-server"]
```

If you're familiar with `docker-compose`, this example should look very familiar.

Here we're using an off-the-shelf Redis image to deploy a redis service. A local container engine, such as Docker, can clearly consume this with one minor tweak. Since this is OpenCompose, the `version` is set to a valid OpenCompose release. 

And, if we were to deploy this to K8S, it would result in a Service load balancing local requests on port 6379 to a single Pod, encapusulating a single Container running the `redis-service` process, and accepting requests on port 6379. And, of course, the container would have it's `workdir` set to `/data`, and the `REDIS_VERSION` environment variable set.



### Example 2 - local_overrides

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

In this example, we're again using `docker-compose` syntax, but this time with a minor tweak. We moved directives that only work locally into a special directive called `local_overrides`. Think of it as an escape hatch, where we can put directives that don't make sense on a K8S cluster, but make complete sense for a local container engine like [runc](https://runc.io/), for example.

For tool developers, this provides a convenient way to hook in custom directives, and build support for engine specific functionality.


### Example 3 - Using named volumes

```yml
version: 1.0-dev
services:
  web:
    image: postgresql:9.5
    expose:
    - 5432
    volumes:
    - pgdata:/var/lib/pgsql
    
volumes:
  pgdata:
    persistent_volume_claim:
      storage: 5Gi
      accessMode: ReadWriteOnce  
    local_overrides: {}
```
  
This example creates a container running PostgreSQL, and mounts a named volume, called *pgdata*, to the container at path */var/lib/pgsql*.

Under the top-level `volumes` directive we've defined *pgdata* with a *persistent volume claim* (PVC) for use on K8S. Run with a local container engine, a PVC likely holds no meaning, so again, we use a *local_overrides* directive to provide any configuration options that the local engine may need.

For information on using PVCs see [Volumes](./volumes.md)

### Example 4 - Local host volumes

```yml
version: 1.0-dev
services:
  web:
    image: acmeco/django:latest
    ports:
    - 8000:8000
    command: ["/bin/false"]
    local_overrides:
      volumes
        - ./project:/project
      workdir: /project
      command: ["/venv/bin/python", "manage.py", "runserver"] 
```

We used a local host volume in [Example 2](#example-2---local_overrides), but didn't specifically point it out. It's repeated here, just to make sure it's clear how a developer can mount host paths to a container while using a local container engine. 

To mount local volumes to a container, add a `volumes` directive to `local_overrides`, as pictured above. When running locally, mounting host paths from the local environment to the container makes sense. However, when the container is deployed to the cloud, the host path is no longer available.

Notice also in the above example, we use `local_overrides` to modify the `command` that executed locally versus when deployed to K8S. Anything about the container that needs to operate differently when running under a local container engine, should appear within `local_overrides`. 

### Example 5 - Services with multiple containers

```yml
version: 1.0-dev
services:
  web:
    restartPolicy: Never
    replicas: 2
    containers:
    - name: apache
      image: apache:latest
      ports:
      - 80:80
      local_overrides:
        links:
        - django:django       
    - name: django
      image: acmeco/django:latest
      expose:
      - 8000
      command: ["/bin/false"]
      local_overrides:
        volumes
          - ./project:/project
        workdir: /project
        command: ["/venv/bin/python", "manage.py", "runserver"] 
```

This example should still look familiar, withe one exception. We've added the `containers` directive, which allows us to define a list of multiple containers under a single service. This is an example of where OpenCompose puts K8S first.

In K8S, this configuration will result in creating a Service named *web* that load balances requests received on port 80 and port 8000, and a single Pod encapsulating both containers. The *web* service would route the traffic to the correct container within the Pod, based on port number. The `replicas` directive tells K8S to scale the Pod to 2 total instances, resulting in 2 sets of containers, or a total of 4 containers. And finally, it sets the `restartPolicy` for the pods to *Never*. 

Running locally, the local tool or container engine would simply create two containers, assuming it doesn't have the ability to create pods, or scale containers. For each container it would also apply the associated `local_overrides` directive.


### Example 6 - Working with k8s primitives

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

In this example, we've decided we don't need services and controllers. Instead, we just want to create a single Pod encapsulating a single container.

How this gets interpreted by a local container engine will depend on whether or not it has the ability to create Pods. It may simply choose to create a single container, with an exposed port.


### Examples 7 - Services that select rather than define a set of containers

```yml
services:
  # Create a service that load balances internal requests to 3, replicated pods.
  internal_web: 
    replicas: 3
    labels:
      app: web
      level: production
    exposed: false
    image: acmeco/web:2.0
    environment:
    - STATIC=/var/www/public/static
    - HTTP_PORT=8080
    command: ["/usr/bin/httpd-foreground"]
    expose
    - 8080

  # Create a service that load balances external requests to the existing pods.
  external_web:
    selector:
      app: web
      level: production
    servicePorts:
    - name: http-port
      port: 80
      targetPort: 8080
    exposed: true
```

When deployed to K8S, this example creates 2 Services. The first listens for internal requests on port 8080, and load balances them to the associated containers on port 8080. It creates a single Pod encapsulating a single container, and replicates the Pod 3 times, creating a total of 3 containers. Notice that it applies *key:value* labels to the Pods as well. 

The second service does not contain any container definitions. Instead, it contains a `selector` directive with *key:value* pairs matching the *key:value* pairs set in the `labels` directive of the first service. The fact that the `label` *key:value* pairs on the Pods match the `selector` *key:value* pairs on the service, causes traffic to be routed to the containers. 

Both `labels` and `selectors` are K8S constructs for identifying and organizing objects. This is again an example of where OpenCompose puts K8S configuration first.
   
How this example is interpreted by a local container engine will depend on it's capabilities, and the tool set being used.


## Reference 

The above examples provide a good overview of what is possible with the OpenCompose language. For a full reference to all of the available directives, visit the following links organized by topic:

- [Services](./services.md)
- [Containers](./containers.md)
- [Volumes](./volumes.md)
- [Pods](./pods.md)
- [Secrets](./secrets.md)
- [Init Containers](./init-containers.md)
- [Config Map](./config-map.md)
- [StatefulSet](./statefulset.md)
- [Jobs](./jobs.md)

