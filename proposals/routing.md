# Improved Routing API Proposal

Diego's story around routing, while sufficient for the CF use-case, has much room for improvement.

To set the stage, let's clarify some roles and responsibilities.

### Roles and Responsibilities

#### Diego

In the context of routing Diego (i.e. Cell+BBS+Brain) is solely responsible for:

1. Opening ports on containers.
2. Making routing-related information available to consumers.

Here "routing information" refers specifically to:

- The `DesiredLRP` and its routing related fields:
    + `Routes`: currently, an array of strings
    + `Ports`: the set of container-side ports to open up on the container.
- The `ActualLRP` and its routing related fields:
    + `Address`: the IP address of the host machine running the container.
    + `Ports`: an array of port mappings of the form `{ "container_port": XXX, "host_port": YYY }`.  There will an entry in the `ActualLRP` `Ports` array for every corresponding entry in the `DesiredLRP` `Ports` array.  To connect to a particular container-side port, you must first lookup the corresponding host-side port and then construct an address of the form `actualLRP.Address:actualLRP.Ports[i].HostPort`

Consumers of Diego are free to specify arbitrary `Ports` on the `DesiredLRP`.  Diego will dutifully open up the corresponding ports on the container and populate the `ActualLRP` `Ports` array appropriately.  Diego does not allow modifying the `DesiredLRP`'s `Ports` array after-the-fact.

Consumers of Diego are *also* free to specify an arbitrary array of `Routes` strings on the `DesiredLRP`.  Diego doesn't actually care about these at all and does nothing with them.  It simply holds onto the array on the consumer's behalf.  Unlike `Ports`, the `Routes` array can be modified dynamically.

It is up to Diego's consumer to fetch the set of `DesiredLRP`s and `ActualLRP`s and construct a routing table.  This is done by:

1. Periodically fetching the entire set of `ActualLRP`s and `DesiredLRP`s via the Receptor API.
2. Attaching to an event stream emanating from the Receptor API.  This event stream emits changes to `ActualLRP`s and `DesiredLRP`s soon after they occur.

Both are necessary to ensure efficiency (real-time events) and robustness (periodic polling to catch for missed events).

#### Router & Route-Emitter

The Route-Emitter communicates with Diego via the Receptor API to construct a routing table.  It then emits this routing table to the router via NATS (there are plans to eventually have the routers communicate directly with the Receptor and cut out the route-emitter).

For a given `ProcessGuid` the route-emitter connects all the FQDNs provided in the `DesiredLRP`s `Routes` array to the **first** port in the `Ports` array on the `ActualLRP`s.

So, concretely, if the consumer requests a `DesiredLRP` with:

```
{
    ...
    "routes": ["foo.com", "bar.com"],
    "ports": [4000, 5000],
    ...
}

```

and Diego starts an `ActualLRP` with:
```
{
    ...
    "address": "10.10.1.2",
    "ports": [
        {"container_port":4000, "host_port":59001},
        {"container_port":5000, "host_port":59002},
    ],
    ...
}
```

Given this, requests to the router for `foo.com` and `bar.com` will both proxy through to `10.10.1.2:59001`.  There is currently no way to connect to port 5000.

> Note: the description in this section applies *after* the stories for adding an [event stream](https://www.pivotaltracker.com/story/show/84607000) and updating the [route-emitter](https://www.pivotaltracker.com/story/show/84607028) are complete.


### Routing 2.0

#### Diego Changes

The fact that `DesiredLRP`'s `Routes` is an array of strings is an accident of history.  What Diego supports today is the minimum necessary to get the CF usecase to work.  In truth, Diego is routing agnostic and we can leave it up to the consumer to encode routing information as they see fit.  This opens up several possibilities, including implementing custom service discovery solutions on top of Diego.

The first part of this proposal, then, is to make Diego less restrictive around what can go into `DesiredLRP.Routes`.  I propose turning `DesiredLRP.Routes` into (in Go parlance) a `map[string]string`.  The key in the map would correspond to a routes provider and the string would correspond to arbitrary metadata associated with said provider.  This allows multiple service-discovery/routing providers to live alongside one-another in Diego.  It also frees up the consumer to define an arbitrary schema to suit their needs

Here's an example that supports the CF Router and a DNS service (e.g. skydns):

```
{
    ...
    "routes": {
        "cf-router": "[{\"port\": 4000, \"routes\": [\"foo.com\", \"bar.com\"]}, {\"port\": 5000, \"routes\":n[\"admin.foo.com\"]}]",
        "skydns" "[{\"port\":5000, \"host\":\"admin-api.service.skydns.local\", \"priority\":20}]"
    },
    "ports": [4000, 5000],
    ...
}
```

The important thing here is that Diego does not care about what goes into  `DesiredLRP.Routes` at all.  This frees the user to cook up whatever schema they deem fit.

#### Route-Emitter Changes

##### Supporting multiple ports

The most basic modification to the route-emitter that I would propose would be to support routing to multiple ports on the same container.

For this the schema for `cf-router`'s entry in `DesiredLRP.Routes` would look like (from above):

```
[
    {
        "port": 4000,
        "routes": ["foo.com", "bar.com"]
    },
    {
        "port": 5000,
        "routes": ["admin.foo.com"]
    }
]
```

Given the sample `ActualLRP` given above, requests to `admin.foo.com` will now be routed to `10.10.1.2:59002`

##### Supporting Index-Specific-Routing

Another (straightforward) addition to route-emitter would be support for routing to a *particular* container index.  This might be useful (for example) to target admin/metric panels for a particular instance or (alternatively) to deterministically refer to specific instances of a database (e.g. giving individual member addresses to an etcd cluster).  The schema for `cf-router` might look like:

```
[
    {
        "port": 4000,
        "routes": ["foo.com", "bar.com"]
    },
    {
        "port": 5000,
        "routes": ["admin.foo.com"],
        "route_to_instances": true
    }
]
```

Now requests to `admin.foo.com` will be balanced across all containers, but requests to `0.admin.foo.com` will *only* go to the container at index 0.

##### Future Extensions

The aforementioned additions can be trivially implementd with the existing gorouter.  Potential future features can also be expressed easily with this flexible schema.  Consider two features:

1. The ability to require `ssl` for a given route.
2. The ability to route `tcp`/`udp` traffic.

These could be expressed via (for example)

```
[
    {
        "port": 4000,
        "protocol": "tcp",
        "incoming_port": 62312
    },
    {
        "port": 4000,
        "protocol": "udp",
        "incoming_port": 43218
    },
    {
        "port": 5000,
        "routes": ["admin.foo.com"],
        "ssl": true
    }
]
```

> As an aside: `incoming_port` here reflects a potential implementation of `tcp/udp` routing whereby a user first checks out a load-balancer port.  This checked-out port can then be associated with a particular application - the gorouter could use the incoming port, and the information expresed in the schema above, to figure out which application to route to.
