# BOSH release migration path to use Go modules

## Summary

After [proposing
different](https://docs.google.com/document/d/1MeiXIqzsj_j1ziYAfhVXfCCFJLiKhi7HiOyF3g-7Wpk/edit#heading=h.9jamy725425y)
routes for converting bosh releases to use Go modules, we are going to move
forward with the following implementation:

- Components will be managed by git submodules and each will have a unique
  module name e.g. `code.cloudfoundry.org/rep`
- Stand alone libraries will have their own vendor dir and manage their own
  dependency via go modules

As far as estimation for this work, I estimate 3-4 weeks of work for
diego-release as an example of a release with over 130 submodules.

## Migration Path

### Circular Dependencies 

When there is a circular import (e.g. [auction imports
auctioneer](https://github.com/cloudfoundry/auction/blob/1378dd51e4050f3755f82d3a11461a71cb80aefb/auctionrunner/batch.go#L7)
and [auctioneer imports
auction](https://github.com/cloudfoundry/auctioneer/blob/master/cmd/auctioneer/main.go#L37)
there should be a refactor/new repository/new package that is spun out with the
common code to break the circular dependency graph.

- **Refactor** For example code.cloudfoundry.org/bbs is used in capi-release,
  routing-release, and more. `bbs` is currently importing `rep` and `auctioneer`
  component in order to use the Clients. The code should be refactored so that
  it only depends on libraries and no other internal components
- **New Internal Package** `auction` and `auctioneer` relationship is an example
  of the kind of relationship where the common-code could be it's own internal package
  under `diego-release` and doesn't need to another repo under cloudfoundry.
- **New Repository** If the common code is used outside of the release, then
  it's suggested to create a new repository with the common code so that
  consumers outside of the release can consume.

### Add Depedabot

When a new module is created, let's also add dependabot so that the dependencies
can remain updated.
[archiver](https://github.com/cloudfoundry/archiver/blob/2762da2677ce6ba931a6e9fff947c7541f470615/.dependabot/config.yml)
is an example of the dependabot config.


### Module dependency versions

BOSH releases have specific SHAs checkout out for each dependency. [Some
repositories](https://github.com/cloudfoundry/diego-release/blob/68b60677acffd6ab241e2698f581c52f5da3ed83/.dependabot/config.yml)
are setup to receive security update (if dependabot security bump is working as
expected), but we are not bumping dependencies to the latest all the time.  When
these are converted to Go module's dependency:

- Go mod tooling will automatically get the latest one from the remote
  repository
- Ensure tests pass with the latest version
- If Broken, let's lock it back to the submoduled SHA and exclude that from
  dependabot


### Vendoring external modules that use replace directive

[`replace`
directive](https://thewebivore.com/using-replace-in-go-mod-to-point-to-your-local-module/)
is very handy when a developer is making changes to modules for BOSH releases
and wants to see the changes applied after deploy without having to commit the
change to upstream. Using `replace` directive works when it's internal to the
release, but when the component is used outside of the release (e.g.
`diego-release` uses `guardian` from `garden-runc-release`) we would have to
manually add the replace commands before being able to vendor `guardian`. In
order to use `guardian`, you must clone the needed repositories using submodules. Here
is an example workflow when adding those components

```
cd inigo
go mod init
go mod edit \
 -replace code.cloudfoundry.org/guardian=../guardian \
 -replace code.cloudfoundry.org/garden=../garden \
 -replace code.cloudfoundry.org/grootfs=../grootfs \
 -replace code.cloudfoundry.org/idmapper=../idmapper
go mod vendor
```

### Add a script to revendor-submodules

We need a way to revendor submodules for components and libraries when changed.
[Here](https://github.com/cloudfoundry/garden-runc-release/blob/develop/scripts/revendor-submodules)
is a script used by Garden team to revendor-submodules for inspiration.

### Add a script to update golang version for each module

Each module would have a script to update go version. [More
info](https://golang.org/ref/mod#go-mod-file-go) for how to do that.

## End Goal

After migration is completed:

- A release should only submodule internal components used under `packages`
  folder.
- All components will have dependabot configured to pull in the latest
  dependencies
- `GOPATH` in `.envrc` is removed
- Release is using Go 1.16+
- `GO111MODULE` is unset
