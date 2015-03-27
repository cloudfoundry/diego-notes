# Background

Private Docker Registry aims to:
- guarantee that we fetch the same layers when the user scales an application up.
- guarantee uptime (if the docker hub goes down we shall be able to start a new instance)
- support private docker images

# Private Registry Configuration

The configuration of the registry is proposed [here](https://github.com/pivotal-cf-experimental/diego-dev-notes/blob/master/proposals/docker_registry_configuration.md)

# Image Caching

To guarantee predictable scaling and uptime we have to copy the Docker image from the public Docker Hub to the private registry.

The basic steps we need to perform are:

* pull image from public Docker Hub
* tag the image
* push the tagged image to the Private Registry
* clean up the pulled image 

The tagging has two purposes:

1. We target the private registry host (since the tag includes the host)
1. Assosiates the image to the desired application
   - generate guid
   - send the guid back to CC as a URL pointing back to the private docker registry

## Pull location 

There are two ways to do this, based on the location where we store the (intermediate) pulled image: 

### Builder

Builder pulls the image and stores it locally in the container. Then it tags and pushes it to the private registry.

**Pros:**

- staging remains isolated with respect to disk space and CPU usage
- easily scale the Docker image staging 

**Cons:**

- limited disk space in staging container (currently 4096 MB)

  *We may need to increase the default quota for staging of docker images.*


### Private Registry

We can use the docker daemon running in the [Private Docker Registry job](https://github.com/pivotal-cf-experimental/docker-registry-release), instructing it to pull, tag and push the image to the registry. To do this we need to either export the daemon's own API or provide a new endpoint that knows how to call it locally. Since the registry runs on the same machine as the daemon, this should be fairly cheap operation.

The Builder can drive the caching steps (pull, tag, push) by using API remotely exposed by the Registry Release. 

**Pros:**

- saves bandwidth

 *The image is not transferred between the Builder task and the Registry*
  
- caches layers

 *Since docker graph is common for all cache requests, the layers are cached in the same place*

**Cons:**

- disk space for pulling images 

 *Docker graph will quickly fill up with layer data for different images. While this serves as a cache and can significantly speed up the pull step, this also creates a problem if we want to get rid of the unused layers. We might use the CC URL pointing to tag guid for cleanup*

- no isolation between staging tasks

  *Single big image might block other staging tasks. To isolate the staging tasks we can limit the size of pulled images. This can be done by creating a file-based file system for every cache request.*

- need to scale registry with staging tasks

## Implementation details

### Caching process

We can take two approaches to implement the caching process (pull, tag and push steps): 
- programmatically drive [docker code](https://github.com/docker/)
- use docker client & daemon
 
Both approaches require privileged process to be able to mount the docker graph file system.

#### Programmatic

We can extending the code in garden-linux that is used to fetch the rootfs by adding pull image, tag image and push image functionality

**Pros:**

- efficiency: reduced memory consumption, number of processes
- no need to provision Docker daemon

**Cons:**

- dependency on Docker internals

 
#### Docker daemon

We can run Docker daemon and request its public REST API. To do this we shall run the daemon as root or in priviledged container.

**Pros:**

- stable public API (does not depend on internal implementation)

**Cons:**

- additional process and memory overhead
- more complex provisioning (configuration, scripts, manifests)
- root access **and** priviliged container

##### Processes

We need two processes:  
- Docker daemon  
- Builder  

We have to launch the two processes in parallel since there is no point in waiting for the daemon to finish.

The Builder is responsible for:  
- wait the daemon to start by reading the response from daemon's `_ping` access point  
- fetch metadata  
- cache image by invoking `docker` CLI:  
   - pull image
   - tag image
   - push image

The Docker daemon accepts requests made directly from Builder or triggered by the `docker` client.

The processes are launched as [ifrit](https://github.com/tedsuo/ifrit) group, which guarantees that if one of them exits or crashes the other one will be terminated. We also use ifrit to be able to terminate both of them if the parent process is signalled.

## Cached images

### URL

The images that are pushed to the private registry are stored as `<ip>:<port>/<GUID>:latest`, where GUID is a generated V4 uuid.

### Tag

The cached images are stored in `library/` with tag `latest`

## Cloud Controller

Cloud Controller table `droplets` is extended with `cached_docker_image` that stores the image URL returned by the staging process.

When desire app request is generated, the `cached_docker_image` is sent, instead of the user provided `docker_image` in `app` table.

If the user opts-out of caching the image, on restage we have to nil the `cached_docker_image` since this will disable the caching on desire app request.
