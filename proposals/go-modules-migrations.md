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

### Refactoring test

Tests (especially integration tests) are sometimes written in ways where they
assume that certs are written under GOATH. [locket
test-runner](https://github.com/cloudfoundry/locket/blob/74d8e4fe8d79ad084fd0343cbe96674d11be7cdb/cmd/locket/testrunner/runner.go#L22-L26)
is an example of such assumption. We often time use the test-runner in a
[different
module](https://github.com/cloudfoundry/bbs/blob/4070ad0e44b12992db094ef803a33dc94e38e73e/cmd/bbs/sql_lock_test.go#L14)
and this causes an issue after conversion to Modules. In order to solve this
issue, consider revising the tests so that it only relies on certs local to the
modules. If a package refers to a cert path but doesn't exercise using them,
consider removing them and make it the job of the caller to provide the certs.
Here is an example of such workflow

- Remove cert from provider

```
diff --git a/cmd/bbs/main_suite_test.go b/cmd/bbs/main_suite_test.go
index e1f83e0..6ef6df6 100644
--- a/cmd/bbs/main_suite_test.go
+++ b/cmd/bbs/main_suite_test.go
@@ -19,11 +19,12 @@ import (
 	bbsconfig "code.cloudfoundry.org/bbs/cmd/bbs/config"
 	"code.cloudfoundry.org/bbs/encryption"
 	"code.cloudfoundry.org/bbs/test_helpers"
-	"code.cloudfoundry.org/bbs/test_helpers/sqlrunner"
 	"code.cloudfoundry.org/consuladapter"
 	"code.cloudfoundry.org/consuladapter/consulrunner"
-	"code.cloudfoundry.org/diego-logging-client"
+	diego_logging_client "code.cloudfoundry.org/diego-logging-client"
 	"code.cloudfoundry.org/diego-logging-client/testhelpers"
+	sql_test_helpers "code.cloudfoundry.org/diegosqldb/test_helpers"
+	"code.cloudfoundry.org/diegosqldb/test_helpers/sqlrunner"
 	"code.cloudfoundry.org/durationjson"
 	"code.cloudfoundry.org/go-loggregator/rpc/loggregator_v2"
 	"code.cloudfoundry.org/inigo/helpers/portauthority"
@@ -79,7 +80,7 @@ var _ = SynchronizedBeforeSuite(
 		bbsPath, err := gexec.Build("code.cloudfoundry.org/bbs/cmd/bbs", "-race")
 		Expect(err).NotTo(HaveOccurred())
 
-		locketPath, err := gexec.Build("code.cloudfoundry.org/locket/cmd/locket", "-race")
+		locketPath, err := gexec.Build("code.cloudfoundry.org/locket/cmd/locket", "-race", "-mod=mod")
 		Expect(err).NotTo(HaveOccurred())
 
 		return []byte(strings.Join([]string{bbsPath, locketPath}, ","))
@@ -99,7 +100,7 @@ var _ = SynchronizedBeforeSuite(
 		SetDefaultEventuallyTimeout(15 * time.Second)
 
 		dbName := fmt.Sprintf("diego_%d", GinkgoParallelNode())
-		sqlRunner = test_helpers.NewSQLRunner(dbName)
+		sqlRunner = sql_test_helpers.NewSQLRunner(dbName)
 		sqlProcess = ginkgomon.Invoke(sqlRunner)
 
 		consulStartingPort, err := portAllocator.ClaimPorts(consulrunner.PortOffsetLength)
@@ -132,7 +133,10 @@ var _ = BeforeEach(func() {
 	var err error
 	logger = lagertest.NewTestLogger("test")
 	ctx = context.Background()
-	fixturesPath := path.Join(os.Getenv("GOPATH"), "src/code.cloudfoundry.org/bbs/cmd/bbs/fixtures")
+
+	wd, err := os.Getwd()
+	Expect(err).To(Succeed())
+	fixturesPath := path.Join(wd, "fixtures")
 
 	consulRunner.Reset()
 	consulClient = consulRunner.NewClient()
```

- Caller should provide the certs

```
diff --git a/cmd/locket/main_test.go b/cmd/locket/main_test.go
index 40da7ec..586a3a6 100644
--- a/cmd/locket/main_test.go
+++ b/cmd/locket/main_test.go
@@ -59,6 +59,9 @@ var _ = Describe("Locket", func() {
 				cfg.DatabaseDriver = sqlRunner.DriverName()
 				cfg.DatabaseConnectionString = sqlRunner.ConnectionString()
 				cfg.ReportInterval = durationjson.Duration(time.Second)
+				cfg.CaFile = "fixtures/ca.crt"
+				cfg.CertFile = "fixtures/cert.crt"
+				cfg.KeyFile = "fixtures/cert.key"
 			},
 		}
 	})
@@ -115,7 +118,11 @@ var _ = Describe("Locket", func() {
 		JustBeforeEach(func() {
 			locketProcess = ginkgomon.Invoke(locketRunner)
 
-			config := testrunner.ClientLocketConfig()
+			config := testrunner.ClientLocketConfig(
+				"fixtures/ca.crt",
+				"fixtures/cert.crt",
+				"fixtures/cert.key",
+			)
 			config.LocketAddress = locketAddress
 
 			var err error
```

### `gexec.Build` Shortcomings with Modules

Often times, we build binaries by invoking `gexec.Build` in order to test our
integration path. After converting to modules, we could run into [an
issue](https://github.com/onsi/gomega/issues/340) when building other vendored
executables. [An example of
this](https://github.com/cloudfoundry/bbs/blob/4070ad0e44b12992db094ef803a33dc94e38e73e/cmd/bbs/main_suite_test.go#L82)
is evident when `bbs` tries to build `locket`. Since the compilation path is not
requiring the `main` package of `locket`, the executable source code for `locket` is not vendored. One way to solve this problem
would be the following change:

```
-		locketPath, err := gexec.Build("code.cloudfoundry.org/locket/cmd/locket", "-race")
+		locketPath, err := gexec.Build("code.cloudfoundry.org/locket/cmd/locket", "-race", "-mod=mod")
```

This will download `locket` from the remote repository in-order to build it.
This will have the downside of the getting the latest on main branch, and it
would be a broken workflow until the main branch is a module.

####Action Item
- Change `-mod=mod` to `-mod=vendor` after `locket` is fully a standalone module
  and see if go is able to figure out how to import the `main.go` file

## End Goal

After migration is completed:

- A release should only submodule internal components used under `packages`
  folder.
- All components will have dependabot configured to pull in the latest
  dependencies
- `GOPATH` in `.envrc` is removed
- Release is using Go 1.16+
- `GO111MODULE` is unset
- When we `ag GOPATH`, all results should be converted/removed including doc
  including docs
