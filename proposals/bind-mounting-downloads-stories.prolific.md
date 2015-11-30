Garden-Windows can support read-only bind-mounts specified at container creation

For the Greenhouse team to implement.


L: bind-mount-downloads

---

[CHORE] Spike: verify buildpack-app staging works when buildpacks are available from bind-mounted directories

Make sure staging tasks still work correctly when buildpacks are available in bind-mounted directories instead of streamed into containers. The files should have the permissions set in the current buildpack archives, and the UIDs that result from the rep/executor expanding them as the host `vcap` user.


L: bind-mount-downloads

---

As a cached-downloader client, I would like to retrieve a cached downloaded asset as an expanded archive in a directory


The cacheddownloader should be able to provide cached assets as expanded archives in a directory, in addition to providing them as a tar stream. To be eligible to be provided in this form, a cache key must accompany the request for the asset, and the download response must provide an ETag header or a Last-Modified header (or both), which are later used as criteria for invalidating cached assets.

When a client obtains an asset directory, it is also understood to obtain a reference to the directory itself. When it is done using the directory, it must explicitly signal to the cacheddownloader that it has released its reference to the directory, so that the cacheddownloader can eventually clean up unused directories from the cache.

- If an asset is requested as a directory and is a cache hit as a directory, the existing directory path is provided.
- If an asset is requested as a directory and is a cache hit as an archive, the asset is expanded into a new directory, and that directory is provided to the client. The archive asset may be deleted only after all previous readers for that archive are closed (as happens today).
- If an existing asset is requested as a directory, but is a cache miss, the asset is downloaded into a new directory and provided to the client. Any prior asset for that cache key (directory or archive) should not be removed from disk until all other references to it are released.

The cacheddownloader must also still be able to provide these cached assets as tar streams to clients:

- If an asset is requested as an archive, but is a cache hit as a directory, the cacheddownloader should provide the contents of the directory as a tar stream, rather than redownloading the asset archive.

The cacheddownloader should also account for the size of assets in directories correctly enough, so that it can correctly account for disk usage when deciding to remove the least-recently items from the cache to make space for new assets.

However we implement this, it must work correctly on Windows as well as on Linux and Darwin.


L: bind-mount-downloads

---

As a BBS API client, I can specify that my DownloadAction on a Task or DesiredLRP is to be provided via a read-only bind-mount, and updated Diego Cells will respect that

On the BBS and its models: Add a new optional, string-valued `BindMount` field to the `DownloadAction` model. If the value is specified as `readonly`, validate that the `CacheKey` has been set, in addition to any other existing `DownloadAction` validations.

Verify that this is not a breaking change to the BBS API. We do not expect this to require any new endpoints.

The rep/executor must recognize the presence of a `readonly` value in the optional `BindMount` value on a container's DownloadAction. In this case, before container creation, the executor should use the cacheddownloader to download the asset into a directory on the host system. The executor should then bind-mount that directory to the desired container-side directory at the time of container creation, using the `BindMounts` field on the garden `ContainerSpec`.

We can impose a stricter failure policy for bind-mounted downloads: even if their failure as a streamed-in asset would not cause a failure of the action flow, their failure to download and attach as a bind-mount should cause the container creation to fail.

On deletion of the container, the executor should release references to all the cacheddownloader-provided directories that it has bind-mounted to the container.


L: bind-mount-downloads

---

Stager uses read-only bind-mounts for buildpack and lifecycle downloads

The stager should construct its tasks to use bind-mounting DownloadActions for the buildpack archives it downloads from Cloud Controller, and for the lifecycle binaries it downloads from the Diego file-server.


L: bind-mount-downloads

---

Nsync recipe generation uses read-only bind-mounts for lifecycle downloads

The nsync LRP recipe generation should construct its DesiredLRPs to use bind-mounting DownloadActions for the lifecycle binaries it downloads from the Diego file-server. It should not use them for other assets, such as the app droplet in the case of buildpack apps.


L: bind-mount-downloads

