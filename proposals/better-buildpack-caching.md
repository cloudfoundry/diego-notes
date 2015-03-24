# Better Buildpack Caching

Things are coming to a head:
- staging on Diego is slow because we're copying buildpacks around.
- the latest buildpacks are massive -- so much so that staging fails on Diego because we exceed the disk quota in the container!

The buildpacks team is slimming down the buildpacks but this only solves a particular usecase.  Customers with their own not-so-lithe buildpacks are going to run into similar issues.

What can we do to improve Diego's staging performance?  Here's a proposal.  It isn't blocked on any new API from Garden though we can transparently transition to the Volumes API at some point in the future if it makes sense.

## Abstractions vs Optimizations

We are faced with an optimization problem that we need to adapt our abstraction to support.  To do this I propose we extend Diego's cached downloading mechanism to support the notion of a named cache that can be mounted and shared between containers at container creation time.

### The new API

We modify Tasks and DesiredLRPs to include a set of shared download caches to mount:

```
Task/DesiredLRP = {
    ...
    SharedDownloadCaches: [
        {
            SharedCacheName: "buildpacks",
            MountPoint: "/buildpacks",
        }
    ]
    ...
}
```

Then we update the `DownloadAction` to reference a shared cache.  There are many paths we can go down here but I'll put up a strawman.  I propose we introduce a new `CachedDownloadAction` so that we have:

```
DownloadAction = {
    From: "http://foo",
    To: "/absolute/path/in/container"
}
```

```
CachedDownloadAction = {
    From: "http://foo",
    To: "/path/relative/to/mount/point",
    CacheKey: "foo",
    SharedCacheName: "buildpacks",
}
```

With this split the semantics become clear.  A `DownloadAction` is never cached and always incurs the cost of a copy *into* the container.  A `CachedDownloadAction` is always cached and mounted directly into the container.  For the asset downloaded by a `CachedDownloadAction` to make it into the container the user *must* add a mount entry in the `SharedDownloadCaches` array.

### Implementation Notes

The basic idea here is that executor will use the information in `SharedDownloadCaches` to bindmount the cache directories into containers.  The cached download action will then download and untar/unzip assets directly into the cache directory.  The cache directory will be assumed to live on the local box - though with the Volumes API in Garden we will eventually have a path towards having Garden live on another box.  This is not a priority for now.

Open questions:

- today's Cached Downloader will delete files when it is running low on resources.  We will at least need to teach it to only do this if a directory is not actively bound to a container.
- what happens if an asset in the cache needs to be updated (for example: a new ruby buildpack has arrived)?  Do we overwrite the existing asset even though the (shared) directory is bound into an existing container?  What does the DEA do here?
- what about cached assets where the copy-in is not a big deal (e.g. the App-Lifecycle binaries).  Do we want to have separate `SharedCacheDownloadAction`s and `CachedDownloadAction`s?  (The latter would behave like the existing `CachedDownloadAction` and incur the copy-in penalty).