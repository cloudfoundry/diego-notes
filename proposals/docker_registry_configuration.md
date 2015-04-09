# Configuration settings for using private Docker Registry

Private Docker Registry aims to:
- guarantee that we fetch the same layers when the user scales an application up.
- guarantee uptime (if the docker hub goes down we shall be able to start a new instance)
- support private docker images

## Security

The Docker Registry can be run in 3 flavours with regards to security:
- HTTP
- HTTPS with self-signed certificate
- HTTPS with CA signed certificate

The first two are considered insecure by the docker registry client. The client needs additional configuration to allow access to insecure registries.

## Configuration

### Components

The following components are involved in the staging and running of Docker image:
- Builder
The Docker app life-cycle builder needs to access the registry to fetch image meta data. To do this it uses the Docker Registry Client. Therefore the client has to be configured to allow insecure connections if this is needed.
- Stager
The builder runs inside a container. This container should allow access to internally hosted (in the protected isolated CF network) registry. The container is configured by the stager that requests launch of the docker builder task.

### Goals

The configuration goals are:
- to allow access to the private Docker Registry
- to enable Docker Registry Client to access the host

### Proposal

#### Stager

We shall use `-dockerRegistryURL` to specify the private Docker Registry location.

The URL shall include the scheme (HTTP or HTTPS) so that the Stager can guess if we host insecure registry in case we have provided HTTP scheme such as: `http://docker_registry.service.dc1.consul:8080`

We may also host a registry that is protected with self-signed certificate. In this case we shall provide `--skip-ssl-validation` flag to instruct the builder to add the HTTPS scheme private registry to the list with insecure hosts.

#### Builder

Builder shall read the optional command line argument `-insecureDockerRegistries` to configure Docker Registry Client and allow access to any insecure hosts. The argument shall accepts comma separated list of \<ip\>:\<port\> pairs. 

Docker client needs a list of all insecure registry service instances. The list is available in Consul but Builder is running inside a container with no access to Consul Agent. That's why the `-insecureDockerRegistries` list is built by Stager. 

The `-dockerRegistryURL` scheme is used to determine if the registry is insecure (if using HTTP). Additionally Stager uses `--skip-ssl-validation` argument to add HTTPS registry to the insecure list.

As a side effect the docker app life-cycle builder may provide access to public registries that are insecure (either HTTP or self-signed cert HTTPS) if they are listed in `-insecureDockerRegistries`. This however will require also forking/modifying Stager code.

**Pros:**

- Simpler configuration
- Consistent with the CF CLI

**Cons:**

- No support for external insecure registries

## Consul service

We shall register the private docker registry with Consul (as we do with the file server). Then we shall privide the Docker Registry configuration in the form: `-dockerRegistryURL=http(s)://docker_registry.service.dc1.consul:8080` to the Stager.

This will help us easily discover the service instances. We do not need to specify concrete IPs of the service nodes in the BOSH manifests as well.

## Egress rules

To be able to access the private Docker Registry we have to open up the container. We have these options:

- Stager fetches a list of all registered `docker_registry` services. This would return all registered IPs and ports and we shall poke holes allowing access to all those IPs and ports.
- We pick a unique port for the private docker registry that doesn't conflict with anything else in the VPC (hard/tricky!) and then open up that port on the entire VPC.

If we use the first option, there's a small race in that a new Docker registry may appear/disappear while we are staging. This would result in a staging failure but this should be very infrequent.

### Staging

To enable discovery of all service instances we shall use [Consul service nodes endpoint](http://www.consul.io/docs/agent/http.html#_v1_catalog_service_lt_service_gt_) (/v1/catalog/service/<service>). This will return the following JSON:
```
[
  {
    "Node": "foobar",
    "Address": "10.1.10.12",
    "ServiceID": "redis",
    "ServiceName": "redis",
    "ServiceTags": null,
    "ServicePort": 8000
  }
]
```

To execute the above request we need additional consul agent URL command line argument in Stager: `-consulAgentURL`.

Currently `ServicePort` is always 0 since we do not register a concrete port and rely on hardcoded one. Using different ports requires changes in:

- consul agent scripts (to register service with different ports)
- stager task create request (to add several egress rules)

For MVP0 we can hardcode the port and make the needed changes for all services later.

Once we find all service instances of the registry we add them explicitly to the staging task container. This presents some problems since we are opening access to the registry even if the desired egress rules (the default CF configuration) does not allow this.

### Running

We don't need to add egress rules to allow access to the Docker Registry instances since Garden fetches the image outside of the container.

## Opt-in

Image caching in private docker registry will not be enabled by default. User can opt-in by:

```
cf set-env <app> DIEGO_DOCKER_CACHE true
```

The app environment is propagated all the way to the stager, which configures the docker lifecycle builder accordingly with the help of the `-cacheDockerImage` command line flag.

If the `-cacheDockerImage` is present the builder:  
- generates cached docker image GUID  
- caches the image in the private registry (pull, tag & push)  
- includes the cached image metadata (`<ip>:<port>/<guid>:latest`) in the staging response

The stager returns the generated staging response back to Cloud Controller (CC). CC stores the cached image metadata in the CCDB.

If the user opt-out then the cached image metadata is cleared to enable restage with the user-provided docker image.

## Private images

Users need to provide credentials to access the Docker Hub private images. The default flow is as follows:
* user provides credentials to `docker login -u <user> -p <password>`
* the login generates ~/.dockercfg file with the following content:
```
{
	"https://index.docker.io/v1/": {
		"auth": "aHNpbGlldjpSZW0wdGVwYXNz",
		"email": "<user email>"
	}
}
```
* `docker pull` can now use the authentication token in the configuration file
 
Therefore we can implement two possible flows in Diego:
* user provides user/password
* user provides authentication token and email
 
The second option should be safer since the authentication token is supposed to be temporary, while user/password credentials can be used to generate new token and gain access to Docker Hub account.
