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
This says that *no* tags are required or disallowed.  Cells 1-8 satisfy this `constraint`.

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

#### Where does this leave `stack`?

I say we burn it with fire as a concept within Diego -- it's not a meaningful first-class concept once we have Placement Pools.

It is, however, a concept at the CF layer.  So, I propose having `stack` be just another tag and teaching CC(-Bridge) to request the `stack` appropriately. Concretely I propose that we: 

- tell BOSH to add the `stack:lucid64` tag to our Cells
- have CC-Bridge construct the `{Require:["stack:lucid64"]}` constraint.

#### Are Placement Pools dynamic?

No.  Not for MVP.

You cannot change the `constraint` on a DesiredLRP.  You must request a new DesiredLRP.

Also, you cannot change the tags on a running Cell.  You will need to perform a rolling deploy to change the tags.

Because of these constraints we do not need to make the converger aware about Placement Pools: they can't change so there's nothing to keep consistent once ActualLRPs are scheduled on Cells.

## Changes to Diego

#### Task/DesiredLRP

We replace `stack` on Tasks and DesiredLRPs with `constraint`.  `constraint` will be **immutable** and is optional (leaving it off means "run this anywhere").

#### Rep

The `rep` should take accept a new command line flag `-tags` -- a comma separated list of tags.  We should make these tags bosh configurable and should include `stack:lucid64` as one of the bosh-configured tags.  We drop `-stack`.

The `rep` will include the list of `tags` in `CellPresence` and responses to `State` requests from the Auctioneer.

#### Auctioneer

The `auctioneer` will be responsible for enforcing `constraint`s.  It does this today with `stack` -- extending it to apply the rules outlined above should be fairly straightforward.

The only subtelty here is around the `PlacementError` that the `auctioneer` applies to ActualLRPs and Tasks that fail to be placed.  There are two and these should be strictly interpreted as follows:

- `diego_errors.CELL_MISMATCH_MESSAGE`: should be returned if there are *no* Cells satisfying the required `constraint`.
- `diego_errors.INSUFFICIENT_RESOURCES_MESSAGE`: should be returned only if there *are* Cells satisfying the required `constraint` but those Cells do not have sufficient capacity to to run the requested work.

## Changes to CF/CC

There are two phases to the CF/CC work.  The first entails getting the existing behavior (stacks) to work with Placement Pools.  The second entails adding support for Placement Pools to the CC itself.

#### Phase 1: Stack

I propose not modifying the CC for this.  Instead both stager and NSYNC will be updated to translate the `stack` parameter on incoming requests to `stack:X` constraints.

#### Phase 2: CC Support

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

For MVP I don't think that CC should enforce any rules on PP (one could imagine a world in which the CC is taught by an operator about which `tags` are deployed -- I don't think we need to go there).

As with ASGs, modifications to a PP will only go through once applications are restarted.

Finally: I imagine stack will remain a first-class concept in the CC.  Either the CC-Bridge or the CC itself will need to fold the stack into the PP by appending it to the `Require` field.