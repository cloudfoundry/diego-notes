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
1. Assosiates the image to the LRP guid (we will add the guid as part of the tag) 


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

Private Registry pulls the image locally and stores it in the Private Registry (currently) running on the same machine. 

The Builder drives the caching steps (pull, tag, push). 

**Pros:**

- saves bandwidth

  *The image is not transferred between the Builder task and the Registry*

**Cons:**

- disk space for pulling images
- no isolation between staging tasks

  *Single big image might block other staging tasks*

- need to scale registry with staging tasks


## Implementation details

We can take two approaches to implement the caching process (pull, tag and push steps): 

### Programmatic

We can extending the code in garden-linux that is used to fetch the rootfs by adding pull image, tag image and push image functionality

**Pros:**

- efficiency: reduced memory consumption, number of processes
- no need to provision Docker daemon

**Cons:**

- dependency on Docker internals

 
### Docker daemon

We can run Docker daemon and request its public REST API. 

**Pros:**

- stable public API (does not depend on internal implementation)

**Cons:**

- additional process and memory overhead
- more complex provisioning (configuration, scripts, manifests)