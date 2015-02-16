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

We shall use `-dockerRegistryURL` to specify the private Docker Registry location.

The URL shall include the scheme (HTTP or HTTPS) so that the Stager and Builder can guess if we host insecure registry in case we have provided HTTP scheme.

We may host a registry that is protected with self-signed certificate. In this case we shall provide `--skip-ssl-validation` flag to instruct the builder to add the HTTPS scheme private registry to the list with insecure hosts.

**Pros:**

- Simpler configuration
- Consistent with the CF CLI

**Cons:**

- No support for external insecure registries

### Alternatives

The following alternative solutions might be used instead of providing a list of insecure hosts:

#### Insecure hosts white-list

We could use additional command line argument `-insecureRegistryHosts` to instruct the Builder to configure Docker Registry Client to allow access to the insecure hosts.

The argument shall accepts comma separated list of hosts. In this way we could also allow the docker app life-cycle builder to provide access to public registries that are insecure (either HTTP or self-signed cert HTTPS).

**Pros:**

- Access to insecure private and external registries

**Cons:**

- Complex configuration

#### Private Registry is always considered insecure

The host passed with `-dockerRegistryURL` is always added to the list of insecure hosts.

**Pros:**

- Simple configuration

**Cons:**

- No support for secure private registry
- No support for external insecure registries

### Docker Registry URL

We shall register the private docker registry with Consul (as we do with the file server). Then we shall pass `-dockerRegistryURL=http(s)://docker_registry.service.dc1.consul:PORT` to the Stager and Builder.

### Egress rules

To open up the container we have these options:

- Stager fetches a list of all registered `docker_registry` services. This would return all registered IPs and ports and we shall poke holes allowing access to all those IPs and ports.
- We pick a unique port for the private docker registry that doesn't conflict with anything else in the VPC (hard/tricky!) and then open up that port on the entire VPC.

If we use the first option, there's a small race in that a new Docker registry may appear/disappear while we are staging. This would result in a staging failure but this should be very infrequent.

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

For MVP0 we can hardcod the port and make the needed changes for all services later.
