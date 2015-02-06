# Private Docker Registry

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

We shall use `-dockerRegistryURL` to specify the private Docker Registry location. The URL shall include the scheme (HTTP or HTTPS) so that the Stager and Builder can guess if we host insecure registry in case we have provided HTTP schema. 

We may however host a registry that is protected with self-signed certificate. We will then need additional command line argument `-insecureRegistryHosts` to instruct the builder to configure Docker Registry Client to allow access to the host.

The `-insecureRegistryHosts` argument shall accepts comma separated list of hosts. In this way we will allow the docker app life-cycle builder to provide access to public registries that are insecure (either HTTP or self-signed cert HTTPS).

**Pros:**
- Access to insecure private and external registries

**Cons:**
- Complex configuration 

### Alternatives
The following alternative solutions might be used instead of providing a list of insecure hosts:
#### SSL Validation Flag
We may use only `-dockerRegistryURL` that is used to separate HTTP and HTTPS registries. Additionally we can provide `--skip-ssl-validation` flag to instruct the builder to add HTTPS scheme private registry to the list with insecure hosts.

**Pros:**
- Simpler configuration
- Consistent with the CF CLI

**Cons:**
- No support for external insecure registries
 
#### Private Registry is always considered insecure
The host passed with `-dockerRegistryURL` is always added to the list of insecure hosts.

**Pros:**
- Simple configuration

**Cons:**
- No support secure private registry
- No support for external insecure registries
