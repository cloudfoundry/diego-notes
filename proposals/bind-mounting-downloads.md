# Bind-Mounting Downloads to Containers

Although the executor's current cacheddownloader prevents us from downloading buildpacks and other large files repeatedly on a given cell, we still incur a large system penalty by copying them into the container each time. We now wish to avoid that penalty by providing them to containers without copying them into the container's own filesystem space. 


## Changes to `DownloadAction`

We propose to extend the current `DownloadAction` to support making the downloaded assets available to the container via a bind-mounted directory. We will add an optional string-valued field, `BindMount`, to the current `DownloadAction`, to provide the following behavior:

- If `BindMount` is unspecified, the `DownloadAction` behaves as it does today, downloading an archive, optionally caching it, and streaming it into the container.
- If `BindMount` is set to `readonly`, the `DownloadAction` instead provides the downloaded asset to the container as follows:
	- The contents of the archive provided in the `From` field are extracted into a directory on the host system.
	- This host-side directory is bind-mounted in read-only mode to the container on the directory specified in the action's `To` field at the time of container creation.
	- Files inside the bind-mounted directory do not count towards the disk-usage quota of the container.

The bind-mounting of the host-side directory and the copying of the downloaded asset into it are not guaranteed to occur in a particular order. The only guarantee made is that if the `DownloadAction` finishes successfully, the directory is present as a bind-mount and contains the downloaded assets.

Opting into bind-mounting for a cached download will not prevent another `DownloadAction` from using the same download with the current stream-in behavior. It will have consequences or altered behavior for some of the other `DownloadAction` fields, though:

- The `CacheKey` field must be set. Bind-mounted downloads are always cached resources.
- The `User` field is disregarded, and from the perspective of the container processes files inside the bind-mounted folder may be owned by an arbitrary user or users (possibly one not even named in their container's `/etc/passwd` file or other platform-specific mechanism for user definitions).

The presence of this new `DownloadAction` variant also has global ramifications for the actions on a container. If a directory is specified in the `To` field of a `readonly`-style bind-mounting `DownloadAction`, it should not be used elsewhere:

- If multiple bind-mounting `DownloadAction`s specify the same `To` directory, only one of them will actually bind-mount its directory there. The directory selected is deliberately left undefined.
- If an ordinary `DownloadAction` also specifies that directory as its `To` directory, it will fail, as the `readonly` nature of the bind-mount will prevent it from writing files there.
- If a file already exists at the specified bind-mount location, garden-linux will fail to create the container.

We do not intend to validate that these issues do not arise when a Task or DesiredLRP is submitted to the BBS. Instead, it is the responsibility of the client to understand the structure of the root filesystem and how the desired `DownloadAction`s may interact with it and with each other.



## Changes to CF workloads

### Buildpack Staging Tasks

Buildpack staging tasks currently download the following groups of assets and copy them into the following conventional locations:

- the app package bits, to the build dir (`/tmp/app`)
- the lifecycle binaries for the stack, to the executable path (`/tmp/lifecycle`)
- the app's buildpack artifact cache, to the cache path (`/tmp/cache`)
- the admin buildpacks, if provided:
	- each admin buildpack is downloaded to `/tmp/buildpacks/<buildpack-key>`

With the new bind-mounting option, we intend to bind-mount some of these assets:

- the lifecycle binaries to `/tmp/lifecycle`
- each admin buildpack to `/tmp/buildpacks/<buildpack-key>`

We observe that there are no directory collisions between these mount locations, and the staging task will be able to operate with these files on a readonly filesystem (as currently happens on the DEAs).


#### Custom buildpacks

If the staging task uses a custom buildpack, the buildpack builder will download it to a different subdirectory of `/tmp/buildpacks`, which will not interfere with any other bind-mounted buildpack downloads (although we do not expect there to be any in this case anyway).


### Buildpack App-Instance DesiredLRPs

Buildpack app-instance DesiredLRPs download a smaller set of assets:

- the droplet bits, to the vcap user's HOME dir (`/home/vcap`)
- the lifecycle binaries for the stack, to the executable path (`/tmp/lifecycle`)

As with the staging task, we intend to bind-mount the lifecycle binaries. This will have the added benefit of causing the lifecycle binaries not to count towards the disk usage quota of the instance.


### Docker Staging Tasks and App-Instances

A Docker staging task is even simpler, as it downloads only the docker lifecycle binaries to `/tmp/lifecycle`. These assets can be bind-mounted to this location instead.

Likewise, Docker app-instance DesiredLRPs also download only the lifecycle binaries to this location, and can bind-mount them instead.


## Implementation Notes

The `DownloadAction` change is a backwards-compatible addition to that BBS model, and can likely be done transparently, especially given our conventions about upgrading the BBS servers before the other Diego components.

The logic for managing the downloads and the bindmounts can likely be contained in the executor and the cacheddownloader.

- The cacheddownloader will have to manage both its current downloaded tarballs and the bind-mountable directories containing the untarred payloads.
- We still intend the cacheddownloader to remove the least frequently used items from its download cache, and including the bind-mounted directories. The cacheddownloader must therefore know which directories are still in use by containers, so it can avoid removing them during its cache cleanup. Some of this locking around downloaded files already exists for the tarballs, but is limited in scope to downloading and streaming in.
- The executor will have to process the container actions before creating the container to add the bind-mount directories to the container spec it sends to Garden. It will also have to retain and release the bind-mounted directories so the cacheddownloader know which ones containers are still using.
- We'll have to decide when the executor or cacheddownloader actually downloads assets that are not in the cache: this could happen at the time of the folder creation (possibly simplifying the locking around access to that folder), or just in time to satisfy the `DownloadAction` contract that the downloaded files are present once the action completes successfully (possibly preventing us from downloading assets too early).
- Any interactions with the filesystem will have to work on Linux, on Windows, and on OS X.


### Current Behavior of the Cacheddownloader

The cacheddownloader currently caches an asset if it meets the following criteria:

- the client provides a cache-key for the asset,
- the download response has either an 'ETag' or a 'Last-Modified' header,
- if the ETag header is present, and the header content can interpreted as an MD5 checksum, the MD5 checksum of the downloaded asset must match that ETag content,
- room can be made for the *transformed* asset in the cache.

The actual entry in the cache is the downloaded asset, transformed into an uncompressed tar archive.

On subsequent requests to fetch an asset with a cache-key,

- if the cacheddownloader has an ETag in its cache info, it sends it in an If-None-Match header,
- if the cacheddownloader has a Last-Modified timestamp in its cache info, it sends it in an If-Modified-Since header.

It then reuses the asset if the response has a 304 Not Modified status code. If not, it has invalidates the entry in the cache:

- if the new request is not cacheable as described above, the key is removed from the cache.
- if the new request is cacheable, it and its caching info replace the current cache entry for that key.

Files are removed from the system only when they are removed from the in-memory cache and when all the readers for their tar-streams that have been issued have been closed.


## Open Questions and Areas for Further Investigation

- How will garden-windows handle the `BindMounts` in the `ContainerSpec`?
- How can we best support the stream-in and bind-mount cases for a single cached asset, while keeping at most one copy in the cache? Converting a cached tarball into an expanded archive in a directory is easy; perhaps for the stream-in of an expanded archive, we can construct the tar stream for the Garden StreamIn operation on the fly from the directory contents.
- Are there malicious downloads we need to insulate ourselves from? Tarballs that would consume all the inodes on disk? If so, how can we best do this in the absence of a volume-manager service that would be able to impose these constraints automatically?

