# Securing Diego's Internal APIs: Auctioneer and Cell Rep

## Motivation

While we think of Diego's BBS server as presenting the main API for interaction with Diego, two other APIs, those presented by the auctioneer and the cell reps, are just as important. Although the BBS API was secured with mutual TLS before Diego declared API stability in v0.1434.0, the same care has not been given to the auctioneer and cell rep APIs. We aim to rectify that situation, and to allow operators to configure these APIs to communicate using mutual TLS.

### Deployment and Release Constraints

In order to make introducing security an acceptable option for existing Diego deployments, operators must be able to deploy it without incurring downtime. In practice, this will mean that both the auctioneer and the cell will need to serve requests on two different ports for some period of time, and their clients will need to be configured to transition from one port to the other. The differences in service discovery between the auctioneer and the cells will afford slightly different solutions to this transition problem.

Additionally, the release must accommodate a range of operator behaviors. Some operators will want to take advantage of the configuration options to secure their deployment immediately through several deploys. Others may wish to or may have to rely on the versioning contract of the Diego release, which stipulates that operators need deploy only once on each major release to avoid downtime in an HA deployment. This contract limits how quickly we may remove deprecated functionality and configuration from the release.

Finally, the Diego release should continue to accommodate the developer case, in which the operator is not operating a production-level or publicly exposed environment and so does not want to bother generating certificates and keys for the TLS configuration.

## Auctioneer API

### Clients and Servers

The auctioneer API interactions are relatively constrained in the Diego deployment, as the only client of the API is the BBS. The BBS also already uses Consul DNS to discover the active auctioneer. We may also assume that operators adhere to the requirement that the BBS instances all update before the auctioneer servers, as the auctioneers are themselves clients of the BBS API.

### Transition to Secure Communication

Consequently, once suitable configuration is available, an operator can switch the deployment to communicate over mutual TLS in two deploys:

- existing deployment:
	- auctioneer listens insecurely on port 9016
	- BBS makes requests insecurely to auctioneer on port 9016
- deploy #1:
	- BBS continues to make requests insecurely to auctioneer on port 9016
	- auctioneer changes also to listen with mutual TLS on port 9017
- deploy #2:
	- BBS changes to make requests to auctioneer using mutual TLS on port 9017
	- auctioneer changes to stop listening on port 9016 and to serve only securely on port 9017

The certificate for the auctioneer server will include `auctioneer.service.cf.internal` as a domain SAN so that the BBS client will consider the server valid when addressed through Consul DNS.

Even if this configuration is introduced in version 0.X of the Diego release, support for the insecure auctioneer server will be removed only in release version 2.0 or later, to accommodate those operators deploying only once per major Diego version.


## Cell Rep API

### Clients and Servers

The cell rep API interactions are more complicated than the auctioneer, as both the BBS and the auctioneer are clients. Additionally, the Diego BOSH release uses the cell rep API locally to health-check the rep on startup and to drain the cell of work.

Fortunately, though, the cell rep also exposes richer data through its cell presence records for its API clients to use in communication. For example, the cell presence contains a `rep_address` field with an IP-addressed base URL for its insecure communication, and the BBS and auctioneer use this value to communicate with each cell. As part of the transition towards a securable API, we propose for the cell to add a similar `rep_url` field to be intended for communication with the secure server, and to make the clients prefer it to the existing `rep_address` field if present. Because of this richer flow of information in the cell presence records, the transition to a secure deployment can still be achieved in 2 depoloys:


- existing deployment:
	- cell rep listens insecurely on port 1800
	- BBS and auctioneer make request insecurely to cells on port 1800
- deploy #1:
	- cell rep changes to:
		- register self as `cell` Consul service with tag `CELL_ID`
		- listen with mutual TLS on port 1801, handling only auction, task, and LRP requests
		- add `rep_url` field to cell presence record with value `https://CELL_ID.cell.service.cf.internal:1801`
	- BBS and auctioneer change to:
		- make secure requests to `rep_url` field if present
		- otherwise, fall back on existing insecure `rep_address` field
- deploy #2:
	- cell rep changes to:
		- listen for auction/task/LRP requests only securely on port 1801
		- listen for administrative requests only insecurely on port 1800 and only on localhost
		- remove `rep_address` field from the cell presence record
	- BBS and auctioneer change to:
		- make requests using only `rep_url` field in the cell presence record

As with the auctioneer, the TLS certificate that the cells present on the secure port will have `*.cell.service.cf.internal` as a domain SAN, so that the BBS and auctioneer clients will consider it valid when connecting.

The existing rep API server will stop serving the auction, Task, and LRP endpoints only in Diego release version 2.0 or later, again to accommodate operators deploying only once on each major version of the release.


## Stories

### Auctioneer

- As a Diego operator, I expect the auctioneer also to serve its API on a TLS-configurable port
	- include docs for new set of config values, recommended way to transition
	- script to generate new set of certs and keys
- As a Diego operator, I expect to be able to configure the BBS to communicate with the auctioneer on its TLS-configurable port
	- don't forget the TLS session cache
	- update docs for new set of config values, recommended way to transition
- As a Diego operator, I expect to be able to configure the auctioneer to accept only secure communication
	- update docs for new set of config values, recommended way to transition


### Cell Rep

- As a Diego operator, I expect each cell reps to register itself in Consul DNS as the `cell` service with a tag given by its cell ID, suitably normalized
	- domains can't have underscores, so convert them to hyphens
- As a Diego operator, I expect the cell reps also to serve their API on a TLS-configurable port
	- include docs for new set of config values, recommended way to transition
	- script to generate new set of certs and keys
- As a Diego operator, I expect the BBS and auctioneer cell clients to prefer communicating to the cell on their TLS-configurable port if possible
	- don't forget the TLS session cache
	- update docs for new set of config values, recommended way to transition
- As a Diego operator, I expect to be able to configure the cell reps to accept only secure communication
	- update docs for new set of config values, recommended way to transition


## BOSH Properties

We explain in detail the properties to add to the Diego BOSH release.

### Auctioneer Server and Clients

#### Auctioneer

Existing:

```yaml
  diego.auctioneer.listen_addr:
    description: "address where auctioneer listens for LRP and task start auction requests"
    default: "0.0.0.0:9016"
```

Add in Diego release version 0.X:

```yaml
  diego.auctioneer.listen_addr_securable:
    description: "address where auctioneer listens for LRP and task start auction requests"
    default: "0.0.0.0:9017"

  diego.auctioneer.enable_insecurable_api_server:
    description: "Whether to enable the legacy, insecurable API server"
    default: true

  diego.auctioneer.require_tls:
    description: "Whether to require TLS for all communication to the securable auctioneer port"
    default: true

  diego.auctioneer.ca_cert:
    description: "PEM-encoded CA certificate"

  diego.auctioneer.server_cert:
    description: "PEM-encoded client certificate"

  diego.auctioneer.server_key:
    description: "PEM-encoded client key"
```

Change in Diego release version 1.0:

- `diego.auctioneer.listen_addr` becomes `diego.auctioneer.listen_addr_insecurable`.
- `diego.auctioneer.listen_addr_securable` becomes `diego.auctioneer.listen_addr`.

Change in Diego release version 2.0:

- Remove `diego.auctioneer.listen_addr_insecurable`.
- Remove `diego.auctioneer.enable_insecurable_api_server`.
- Auctioneer API server listens only on one port, defaulting to 9017.

#### BBS as Auctioneer Client

Existing:

```yaml
  diego.bbs.auctioneer.api_url:
    description: "Address of the auctioneer API"
    default: "http://auctioneer.service.cf.internal:9016"
```

Add in Diego release version 0.X:

```yaml
  diego.bbs.auctioneer.use_insecurable_client:
    description: "Whether to use the legacy, insecurable auctioneer client instead of the TLS-configurable client."
    default: true

  diego.bbs.auctioneer.api_location:
    description: "Hostname and port of the auctioneer API."
    default: "auctioneer.service.cf.internal:9017"

  diego.bbs.auctioneer.require_tls:
    description: "Whether to use TLS for communication with the auctioneer API."
    default: true

  diego.bbs.auctioneer.ca_cert:
    description: "CA cert for communication to the auctioneer."

  diego.bbs.auctioneer.client_cert:
    description: "Client cert for communication to the auctioneer."

  diego.bbs.auctioneer.client_key:
    description: "Client key for communication to the auctioneer."
```

Change in Diego release version 2.0:

- Remove `diego.bbs.auctioneer.api_url`.
- Remove `diego.bbs.auctioneer.use_insecurable_client`.


#### Deployment configurations

`(*)` signifies that a value other than the spec default is used. We abbreviate `diego.auctioneer` as `a` and `diego.bbs.auctioneer` as `b.a`.

##### Migrate existing deployment to TLS on Diego v0

- existing deployment:
	- BBS communicates to 9016, insecure
		- `b.a.api_url: "http://auctioneer.service.cf.internal:9016"`
	- auctioneer listens on 9016, insecure
		- `a.listen_addr: "0.0.0.0:9016"`
- introduce new release 0.X with new clients and servers
- operator generates new certificates and keys:
	- `DIEGO_CA_CERT`: CA certificate used for Diego inter-component communication (can be reused from BBS configuration)
	- `AUCTIONEER_SERVER_CERT`: certificate with domain SAN auctioneer.service.cf.internal, signed by Diego CA
	- `AUCTIONEER_SERVER_KEY`: corresponding key
	- `AUCTIONEER_CLIENT_CERT`: certificate for BBS client, signed by Diego CA
	- `AUCTIONEER_CLIENT_KEY`: corresponding key
- deploy 1:
	- BBS communicates to 9016, insecure
		- `b.a.api_url: "http://auctioneer.service.cf.internal:9016"`
		- `b.a.use_insecurable_client: true`
		- `b.a.api_location: "auctioneer.service.cf.internal:9017"`
		- `b.a.require_tls: false (*)`
		- `b.a.ca_cert: null`
		- `b.a.client_cert: null`
		- `b.a.client_key: null`
	- auctioneer listens on 9016, insecure, and 9017, secure
		- `a.listen_addr: "0.0.0.0:9016"`
		- `a.listen_addr_securable: "0.0.0.0:9017"`
		- `a.enable_insecurable_api_server: true`
		- `a.require_tls: true`
		- `a.ca_cert: DIEGO_CA (*)`
		- `a.server_cert: AUCTIONEER_SERVER_CERT (*)`
		- `a.server_key: AUCTIONEER_SERVER_KEY (*)`
- deploy 2:
	- BBS communicates to 9017, secure
		- `b.a.api_url: "http://auctioneer.service.cf.internal:9016"`
		- `b.a.use_insecurable_client: false (*)`
		- `b.a.api_location: "auctioneer.service.cf.internal:9017"`
		- `b.a.require_tls: true`
		- `b.a.ca_cert: DIEGO_CA_CERT (*)`
		- `b.a.client_cert: AUCTIONEER_CLIENT_CERT (*)`
		- `b.a.client_key: AUCTIONEER_CLIENT_KEY (*)`
	- auctioneer listens only on 9017, secure
		- `a.listen_addr: "0.0.0.0:9016"`
		- `a.listen_addr_securable: "0.0.0.0:9017"`
		- `a.enable_insecurable_api_server: false (*)`
		- `a.require_tls: true`
		- `a.ca_cert: DIEGO_CA_CERT (*)`
		- `a.server_cert: AUCTIONEER_SERVER_CERT (*)`
		- `a.server_key: AUCTIONEER_SERVER_KEY (*)`
- auctioneer serving only secure endpoint

##### Migrate existing deployment to new port without TLS

- existing deployment:
	- BBS communicates to 9016, insecure
		- `b.a.api_url: "http://auctioneer.service.cf.internal:9016"`
	- auctioneer listens on 9016, insecure
		- `a.listen_addr: "0.0.0.0:9016"`
- introduce new release 0.X with new clients and servers
- deploy 1:
	- BBS communicates to 9016, insecure
		- `b.a.api_url: "http://auctioneer.service.cf.internal:9016"`
		- `b.a.use_insecurable_client: true`
		- `b.a.api_location: "auctioneer.service.cf.internal:9017"`
		- `b.a.require_tls: false (*)`
		- `b.a.ca_cert: null # TODO: ok value without spec default?`
		- `b.a.client_cert: null`
		- `b.a.client_key: null`
	- auctioneer listens on 9016, insecure, and 9017, insecure
		- `a.listen_addr: "0.0.0.0:9016"`
		- `a.listen_addr_securable: "0.0.0.0:9017"`
		- `a.enable_insecurable_api_server: true`
		- `a.require_tls: false (*)`
		- `a.ca_cert: null`
		- `a.server_cert: null`
		- `a.server_key: null`
- deploy 2:
	- BBS communicates to 9017, insecure
		- `b.a.api_url: "http://auctioneer.service.cf.internal:9016"`
		- `b.a.use_insecurable_client: false (*)`
		- `b.a.api_location: "auctioneer.service.cf.internal:9017"`
		- `b.a.require_tls: false (*)`
		- `b.a.ca_cert: null`
		- `b.a.client_cert: null`
		- `b.a.client_key: null`
	- auctioneer listens only on 9017, insecure
		- `a.listen_addr: "0.0.0.0:9016"`
		- `a.listen_addr_securable: "0.0.0.0:9017"`
		- `a.enable_insecurable_api_server: false (*)`
		- `a.require_tls: false (*)`
		- `a.ca_cert: null`
		- `a.server_cert: null`
		- `a.server_key: null`


### Cell Rep Server and Clients

#### Cell Rep

Existing:

```yaml
  diego.rep.listen_addr:
    description: "address to serve auction and LRP stop requests on"
    default: "0.0.0.0:1800"
```

Add in Diego release version 0.X:

```yaml
  diego.rep.listen_addr_securable:
    description: "address where rep listens for LRP and task start auction requests"
    default: "0.0.0.0:1801"

  diego.rep.enable_insecurable_api_server:
    description: "Whether to enable the auction, LRP, and Task endpoints on the legacy, insecurable API server"
    default: true

  diego.rep.advertise_domain:
    description: "base domain at which the rep should advertise its secure API"
    default: "cell.service.cf.internal"

  diego.rep.require_tls:
    description: "Whether to require TLS for communication to the securable rep API server"
    default: true

  diego.rep.ca_cert:
    description: "PEM-encoded CA certificate"

  diego.rep.server_cert:
    description: "PEM-encoded client certificate"

  diego.rep.server_key:
    description: "PEM-encoded client key"
```

Change in Diego 2.0:

- Change `diego.rep.listen_addr` to `diego.rep.listen_addr_internal`, with default `127.0.0.1:1800`.
- Change `diego.rep.listen_addr_securable` to `diego.rep.listen_addr`.
- Remove `diego.rep.enable_insecurable_api_server`.
- Rep component exposes only `/ping` and `/evacuate` endpoints on the internal server.

#### BBS as Cell Client

Existing: none

Add in Diego release version 0.X:

```yaml
  diego.bbs.rep.require_tls:
    description: "Whether to require TLS for communication to the securable rep API server"
    default: true

  diego.bbs.rep.ca_cert:
    description: "CA cert for communication to the rep."

  diego.bbs.rep.client_cert:
    description: "Client cert for communication to the rep."

  diego.bbs.rep.client_key:
    description: "Client key for communication to the rep."
```


#### Auctioneer as Cell Client

Existing: none

Add in Diego release version 0.X:

```yaml
  diego.auctioneer.rep.require_tls:
    description: "Whether to require TLS for communication to the securable rep API server"
    default: true

  diego.auctioneer.rep.ca_cert:
    description: "CA cert for communication to the rep."

  diego.auctioneer.rep.client_cert:
    description: "Client cert for communication to the rep."

  diego.auctioneer.rep.client_key:
    description: "Client key for communication to the rep."
```

#### Deployment Configurations

`(*)` signifies that a value other than the spec default is used. We abbreviate `diego.rep` as `r`, `diego.auctioneer.rep` as `a.r` and `diego.bbs.rep` as `b.r`.

##### Migrate existing deployment to TLS on Diego v0

- existing deployment:
	- cell reps listen on 1800, insecure
		- `r.listen_addr: "0.0.0.0:1800"`
	- auctioneer communicates to CELL_IP:1800, insecure
	- BBS communicates to CELL_IP:1800, insecure
- introduce new release 0.X with new clients and servers
- operator generates new certificates and keys:
	- `DIEGO_CA_CERT`: CA certificate used for Diego inter-component communication (can be reused from BBS configuration)
	- `CELL_SERVER_CERT`: certificate with domain SAN *.cell.service.cf.internal, signed by Diego CA
	- `CELL_SERVER_KEY`: corresponding key
	- `CELL_CLIENT_CERT`: certificate for BBS and auctioneer clients, signed by Diego CA
	- `CELL_CLIENT_KEY`: corresponding key
- deploy 1:
	- BBS prefers `rep_url` in cell presence, using TLS configuration
		- `b.r.require_tls: true`
		- `b.r.ca_cert: DIEGO_CA_CERT (*)`
		- `b.r.client_cert: CELL_CLIENT_CERT (*)`
		- `b.r.client_key: CELL_CLIENT_KEY (*)`
	- auctioneer prefers `rep_url` in cell presence, using TLS configuration
		- `a.r.require_tls: true`
		- `a.r.ca_cert: DIEGO_CA_CERT (*)`
		- `a.r.client_cert: CELL_CLIENT_CERT (*)`
		- `a.r.client_key: CELL_CLIENT_KEY (*)`
	- cells register as `cell` service in Consul with tag `CELL_ID`, also listen on 1801 securely, publish `rep_url` field in presence as `https://CELL_ID.cell.service.cf.internal:1801`
		- `r.listen_addr: "0.0.0.0:1800"`
		- `r.listen_addr_securable: 0.0.0.0:1801`
		- `r.enable_insecurable_api_server: true`
		- `r.advertise_domain: cell.service.cf.internal`
		- `r.require_tls: true`
		- `r.ca_cert: DIEGO_CA_CERT (*)`
		- `r.server_cert: CELL_SERVER_CERT (*)`
		- `r.server_key: CELL_SERVER_KEY (*)`
- deploy 2:
	- BBS prefers `rep_url` in cell presence, using TLS configuration
		- `b.r.require_tls: true`
		- `b.r.ca_cert: DIEGO_CA_CERT (*)`
		- `b.r.client_cert: CELL_CLIENT_CERT (*)`
		- `b.r.client_key: CELL_CLIENT_KEY (*)`
	- auctioneer prefers `rep_url` in cell presence, using TLS configuration
		- `a.r.require_tls: true`
		- `a.r.ca_cert: DIEGO_CA_CERT (*)`
		- `a.r.client_cert: CELL_CLIENT_CERT (*)`
		- `a.r.client_key: CELL_CLIENT_KEY (*)`
	- cells register as `cell` service in Consul with tag `CELL_ID`, listen only on 1801 securely for auction/task/LRP endpoints, publish `rep_url` field in presence as `https://CELL_ID.cell.service.cf.internal:1801`, do not publish `rep_address` field in presence
		- `r.listen_addr: "127.0.0.1:1800" (*)`
		- `r.listen_addr_securable: 0.0.0.0:1801`
		- `r.enable_insecurable_api_server: false (*)`
		- `r.advertise_domain: cell.service.cf.internal`
		- `r.require_tls: true`
		- `r.ca_cert: DIEGO_CA_CERT (*)`
		- `r.server_cert: CELL_SERVER_CERT (*)`
		- `r.server_key: CELL_SERVER_KEY (*)`
- All remote communication with the rep API now requires mutual TLS. BOSH control scripts still use the separate insecure endpoint, but that now listens only on 127.0.0.1 and so is inaccessible remotely.

##### Migrate existing deployment to new port without TLS

- existing deployment:
	- cell reps listen on 1800, insecure
		- `r.listen_addr: "0.0.0.0:1800"`
	- auctioneer communicates to CELL_IP:1800, insecure
	- BBS communicates to CELL_IP:1800, insecure
- introduce new release 0.X with new clients and servers
- deploy 1:
	- BBS prefers `rep_url` in cell presence, using HTTP only
		- `b.r.require_tls: false (*)`
		- `b.r.ca_cert: null`
		- `b.r.client_cert: null`
		- `b.r.client_key: null`
	- auctioneer prefers `rep_url` in cell presence, using HTTP only
		- `a.r.require_tls: false (*)`
		- `a.r.ca_cert: null`
		- `a.r.client_cert: null`
		- `a.r.client_key: null`
	- cells register as `cell` service in Consul with tag `CELL_ID`, also listen on 1801 securely, publish `rep_url` field in presence as `http://CELL_ID.cell.service.cf.internal:1801`
		- `r.listen_addr: "0.0.0.0:1800"`
		- `r.listen_addr_securable: 0.0.0.0:1801`
		- `r.enable_insecurable_api_server: true`
		- `r.advertise_domain: cell.service.cf.internal`
		- `r.require_tls: false (*)`
		- `r.ca_cert: null`
		- `r.server_cert: null`
		- `r.server_key: null`
- deploy 2:
	- BBS prefers `rep_url` in cell presence, using HTTP only
		- `b.r.require_tls: false (*)`
		- `b.r.ca_cert: null`
		- `b.r.client_cert: null`
		- `b.r.client_key: null`
	- auctioneer prefers `rep_url` in cell presence, using HTTP only
		- `a.r.require_tls: false (*)`
		- `a.r.ca_cert: null`
		- `a.r.client_cert: null`
		- `a.r.client_key: null`
	- cells register as `cell` service in Consul with tag `CELL_ID`, listen only on 1801 securely for auction/task/LRP endpoints, publish `rep_url` field in presence as `http://CELL_ID.cell.service.cf.internal:1801`, do not publish `rep_address` field in presence
		- `r.listen_addr: "127.0.0.1:1800" (*)`
		- `r.listen_addr_securable: 0.0.0.0:1801`
		- `r.enable_insecurable_api_server: false (*)`
		- `r.advertise_domain: cell.service.cf.internal`
		- `r.require_tls: false (*)`
		- `r.ca_cert: null`
		- `r.server_cert: null`
		- `r.server_key: null`
- All remote communication with the rep API now happens via the new port, but is still insecure. BOSH control scripts still use the separate insecure endpoint, but that now listens only on 127.0.0.1 and so is inaccessible remotely.


