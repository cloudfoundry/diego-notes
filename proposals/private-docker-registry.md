# Private Docker Registry

Diego's docker support today is limited to fetching images from the public docker registry.  This has a number of issues:

- No way to guarantee that we fetch the same layers when the user scales an application up.
- No way to guarantee uptime (if the docker hub goes down we can't start a new instance)
- No (good) way to support private docker registries

To overcome these issues we propose running a private Docker registry backed by Riak CS.  One of the most compelling aspects of our proposed solution is that it requires *no* changes to any core Diego components (including Garden-Linux).  Instead, only CC and the Docker-Circus need to change.

**Goals**
- solve the aforementioned problems
- do so without making Diego more docker-specific

**None-Goals**
- providing CF consumers with a docker registry that they `docker push` to

## The Proposal

The basic premise here is that we would augment the existing staging step in the Docker flow to *fetch* the Docker layers and then re-upload them to a private Docker registry -- we would push to the registry using a uniquely generated guid (e.g the ProcessGuid) to guarantee that scaling the instance up/down always results in an identical set of layers.

To implement this we would modify the Docker-Tailor and Stager/CC to do the following:

1. When staging a Dockerimage the Docker-Tailor fetches the layers and uploads them to the internal registry.
    - For a V0 implementation we would always fetch all the layers and attempt to push them to the internal registry.  Docker's built-in caching support will naturally ensure we only push new layers, however will be required to download all the layers.
    - For a V1 implementation we might try to minimize the layers we *download* as well.  Perhaps we can query the registry for the set of layers that comprise the image and only download ones that are *not* in our private registry?
2. Upon staging the Docker-Tailor should return a Dockerimage url that uniquely points to the image created in the internal registry.  This will be unique to each CF push.
3. CC should store off this Dockerimage url and use it when requesting instances of the application.

Note that the Task the Stager requests will need to be able to communicate with the private docker registry (presumably in the VPC).  We'll need to use the Receptor's egress rules to poke holes to communicate with the private docker registry.  We'll need to discover these at runtime in the Stager in order to poke *just* the holes to communicate with the docker registry.  This is not an issue when it comes to *fetching* the Dockerimage as Garden-Linux does the downloading and has complete access to the VPC.

## The Road to MVP

We begin by using the [Python docker-registry](https://github.com/docker/docker-registry).  We can use the community-generated (docker-registry BOSH release)[https://github.com/cloudfoundry-community/docker-registry-boshrelease] to bosh deploy the python registry.  The first iteration would be a spike in which we back the docker-registry with local storage.

With an internal registry in place we can work on the code necessary to have the Tailor download and re-upload the Dockerimage.  It would be best if this could be optional (maybe even on an app-by-app basis for flexibility for now).  Applications that opt-into the internal registry will be copied into it, apps that aren't will always be fetched from the Dockerhub.

Once this approach is validated we can make the registry more robust and add new features.

## The Roadmap past MVP

- MVP: private docker registry backed by local disk
- MVP: opt-in support for storing Dockerimages in the private registry
- private docker registry backed by [Riak CS](https://github.com/cloudfoundry/cf-riak-cs-release)
- highly-available private docker registry
- ability to stage *private* images (i.e. images that require auth to download.  Basic idea: user supplies credentials which we use *at staging time* to fetch their dockerimage - a short-lived token is OK since we only need it for staging)
- support for [Docker's V2 API](https://github.com/docker/distribution) -- currently not ready for primetime, but when it is we'll want our private registry to speak V2 and use the new Docker go codebase
- ability to prune the private registry of unused Dockerimages (e.g. grab DesiredLRPs in fresh domains and flag any unused layers for removal)?