## BOSH release migration path to use Go modules

### Summary After [proposing
different](https://docs.google.com/document/d/1MeiXIqzsj_j1ziYAfhVXfCCFJLiKhi7HiOyF3g-7Wpk/edit#heading=h.9jamy725425y)
routes for converting bosh releases to use Go modules, we are going to move
forward with the following implementation:

- Components will be managed by git submodules and each will have a unique
  module name e.g. `code.cloudfoundry.org/rep`
- Stand alone libraries will have their own vendor dir and manage their own
  dependency via go modules

As far as estimation for this work, I estimate 3-4 weeks of work for
diego-release as an example of a release with over 130 submodules.

### Migration Path

#### Circular Dependencies When there is a circular import When there is a
circular import ([code.cloudfoundry.org/auction imports
code.cloudfoundry.org/auctioneer](https://github.com/cloudfoundry/auction/blob/1378dd51e4050f3755f82d3a11461a71cb80aefb/auctionrunner/batch.go#L7)
and [code.cloudfoundry.org/auctioneer imports
code.cloudfoundry.org/auction](https://github.com/cloudfoundry/auctioneer/blob/master/cmd/auctioneer/main.go#L37)
there should be a refactor/new repository/new package that is spun out with the
common code to break the circular dependency graph.

- **Refactor** For example code.cloudfoundry.org/bbs is used in capi-release,
  routing-release, and more. `bbs` is currently importing `rep` and `auctioneer`
  component in order to use the Clients. The code should be refactored so that
  it only depends on libraries and no other internal components
- **New Internal Package** `auction` and `auctioneer` relationship is an example
  of the kind of relationship where the model could be it's own internal package
  under `diego-release` and doesn't need to another repo under cloudfoundry.
- **New Repository** If the common code is used outside of the release, then
  it's suggested to create a new repository with the common code so that consumers
  outside of the release can consume.

#### Add Depedabot
When a new module is created, let's also add dependabot so that the
dependencies can remain updated.
[archiver](https://github.com/cloudfoundry/archiver/blob/2762da2677ce6ba931a6e9fff947c7541f470615/.dependabot/config.yml)
is an example of the dependabot config.


#### Module dependency versions

BOSH releases have specific SHAs checkout out for each dependency. [Some
repositories](https://github.com/cloudfoundry/diego-release/blob/68b60677acffd6ab241e2698f581c52f5da3ed83/.dependabot/config.yml)
are setup to receive security update (if dependabot security bump is working
as expected), but we are not bumping dependencies to the latest all the time.
When these are converted to Go module's dependency:

- Go mod tooling with automatically get the latest one from the remote repository
- Ensure tests pass with the latest version
- If Broken, let's lock it back to the submoduled SHA and exclude that from dependabot


