# Supporting multiple RootFSes.  AKA what is Stack?

Stack has been a confusing concept in the CC.  I'd like to propose that stack simply correlates with the RootFS.  I also propose that a single Cell should be able to support multiple RootFSes (this has many benefits including, for example, simplifying the process of upgrading from one RootFS to another).

Where we need to head in the short-term is support for the following:

- `lucid64` - a preloaded tarball based on `lucid`
- `cflinxfs2` - a preloaded tarball based on `trusty`
- `docker` - a dynamically download RootFS.

I propose making this clearer in the Diego API by dropping `Stack` from the `DesiredLRP/Task` and, instead, relying on the existing `RootFS` field which would take on the values (e.g.):

```
lucid64: {
    RootFS: "preloaded://lucid64",
}

cflinxfs2: {
    RootFS: "preloaded://cflinuxfs2",
}

docker: {
    RootFS: "docker:///foo/bar#baz",
}
```

## Supporting multiple RootFSes

Once the `DesiredLRP/Task` has this information we would modify the components as follows.

### Rep

The Rep would take a list of preloaded RootFSes and a list of supported RootFSProviders.  For example:

```
rep --preloaded-rootfses='{"lucid64":"/path/to/lucid64", "cflinuxfs2":"/path/to/cflinuxfs2"} --supported-root-fs-providers="docker"
```

This information could make its way onto the Cell Presence.  Not strictly necessary at this point, though it is convenient to have access to this information from the API.

### Auction

During the auction the Auctioneer will be given the RootFS and the Rep will furnish its available preloaded RootFSes and supported RootFS providers in its response to `State` requests:

```
State: {
    RootFSProviders = ["docker"],
    PreloadedRootFSes = ["lucid64", "cflinuxfs2"],
}
```

The Auctioneer would then know whether or not a Cell could support the requested RootFS.  In pseudocode:

```
func (c Cell) CanRunTask(task Task) bool {
    if task.RootFS.Scheme == "preloaded" {
        return c.PreloadedRootFSes.Contains(task.RootFSResource)
    } else {
        return c.RootFSProviders.Contains(task.RootFS.Scheme)
    }
}
```

### Stager/NSYNC

Stager/NSYNC would translate CC's "Stack" requests to appropriate values for the two RootFS fields.

---

# Placement Pools

Placement Pools are going to be one of the first new features that Diego brings to the platform.  This proposal is intended to get that ball rolling.

## What are Placement Pools?

At the highest level Placement Pools will allow operators to group Cells arbitrarily by painting them with tags.  Diego workloads can then be constrainted to run on a certain set of tags.

#### Tags

A `tag` is a an arbitrary case-insensitive string (e.g. `"staging"`, `"production"`, `"skynet"`).  Cells can be painted with arbitrarily many `tags`.  For example

Cell 1 | Cell 2 | Cell 3 | Cell 4 | Cell 5 | Cell 6 | Cell 7 | Cell 8 | Cell 9
-------|--------|--------|--------|--------|--------|--------|--------|-------
staging|staging|staging|staging|production|production|production|production|
skynet|skynet|||skynet|skynet|||skynet

A `tag` must be less than 64 characters long (arbitrary!)

#### Constraints

`constraint` is a set of rules associated with a Task or DesiredLRP.  The `constraint` determines which Cells are eligible for running the given workload.

Here is an empty `constraint`:
```
Constraint: {
    Require: [],
    Disallow: []
}
```
This says that *no* tags are required or disallowed.  Cells 1-9 satisfy this `constraint`.

Here is a `constrant` that requires a workload run on `staging` Cells:
```
Constraint: {
    Require: ["staging"],
    Disallow: []
}
```
Cells 1-4 satisfy this `constraint`.

Here is a subtly different `constraint` that disallows a workload from running on `production` Cells:
```
Constraint: {
    Require: [],
    Disallow: ["production"]
}
```
Cells 1-4,9 satisfy this `constraint`.

Here is a `constraint` that only runs on `staging` Cells allocated to the `skynet` corporation:
```
Constraint: {
    Require: ["staging", "skynet"],
    Disallow: []
}
```
Cells 1-2 satisfy this `constraint`.

And here is a `constraint` that only runs on `staging` Cells that are *not* associated with the `skynet` corporation:
```
Constraint: {
    Require: ["staging"],
    Disallow: ["skynet"]
}
```
Cells 3-4 satisfy this `constraint`.

Finally, here are some `constraint`s that cannot be satisfied with the given Cell setup:
```
Constraint: {
    Require: ["staging"],
    Disallow: ["staging"]
}

Constraint: {
    Require: ["alfalfa"],
    Disallow: []
}
```
No cells could possiby satisfy these particular `constraint`s.  Diego is not in the business of identifying these sorts of inconsistencies -- it is up to the consumer to coordinate their Placement Pool `tags` and `constraint`s.  Diego will, however, inform the user (asynchronously after a failed attempt to auction) when it fails to satisfy a constraint.

#### How does this interact with `Stack`?

It does not need to.  CC's `Stack` is related to the RootFS (discussed above).

#### Are Placement Pools dynamic?

No.  Not for MVP.

You cannot change the `constraint` on a DesiredLRP.  You must request a new DesiredLRP.

Also, you cannot change the tags on a running Cell.  You will need to perform a rolling deploy to change the tags.

Because of these constraints we do not need to make the converger aware about Placement Pools: they can't change so there's nothing to keep consistent once ActualLRPs are scheduled on Cells.

#### Querying Diego for `tags`

For MVP we propose using the `/v1/cells` Receptor API to fetch all Cells and derive the set of tags.

If/when we switch to a relational database we will be able to support `/v1/tags` to fetch all tags cheaply.

## Changes to Diego

#### Task/DesiredLRP

We add `constraint` to Tasks and DesiredLRPs.  `constraint` will be **immutable** and is optional (leaving it off means "run this anywhere").

#### Rep

The `rep` should take accept a new command line flag `-tags` -- a comma separated list of tags.  We should make these tags bosh configurable.

The `rep` will include the list of `tags` in `CellPresence` and responses to `State` requests from the Auctioneer.

#### Auctioneer

The `auctioneer` will be responsible for enforcing `constraint`s in addition to `rootfs` (see above).  Extending it to apply the rules outlined above should be fairly straightforward.

The only subtelty here is around the `PlacementError` that the `auctioneer` applies to ActualLRPs and Tasks that fail to be placed.  There are two and these should be strictly interpreted as follows:

- `diego_errors.CELL_MISMATCH_MESSAGE`: should be returned if there are *no* Cells satisfying the required `constraint`.
- `diego_errors.INSUFFICIENT_RESOURCES_MESSAGE`: should be returned only if there *are* Cells satisfying the required `constraint` but those Cells do not have sufficient capacity to to run the requested work.

## Changes to CF/CC

Like Application Security Groups (ASG), Placement Pools (PP) will be a assigned on a per-space basis.  I imagine we would mirror the organization of ASGs as closely as possible with the difference that the PP associated with an application will apply to staging and running applications.  Looking at the [CC API docs](http://apidocs.cloudfoundry.org/197/) for ASG this would entail APIs that support:

- CRUDding PPs
- Associating/disassociating spaces with a PP
- Specifying a default PP

For CC, a PP would look like identical to a Diego `constraint`:

```
PP = {
    Require: [],
    Disallow: []
}
```

As with ASGs, modifications to a PP will only go through once applications are restarted.

#### Validations

For MVP I don't think that CC should enforce any rules on PlacementPools.  One could imagine a world in which the CC is taught by an operator about which `tags` are deployed (or reaches out to Diego to learn about available tags) -- I don't think we need to go there quite yet.

#### Placement Pools & Restarts vs Restages

The CC is somewhat unclear (and confused) about what actions must trigger restages and what actions must trigger restarts.

We propose that modifications to Placement Pools need only trigger a restart.
