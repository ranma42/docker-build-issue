# docker-build-issue

## Cleaning the environment
Each step in this reproducer assumes you run in a clean environment, i.e. right
before running the step you should do something like:

```
docker system prune -a
```

WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all images without at least one container associated to them
  - all build cache

so **DO NOT RUN IT** unless you are in an environment which you are willing do
completely destroy (example: a test environment created for reproducing this
issue) or are well aware fo what you are doing.

## No network
Building in a clean environment without network connectivity fails as expected:

```
$ DOCKER_BUILDKIT=0 docker build .
Sending build context to Docker daemon   59.9kB
Step 1/1 : FROM alpine
Get "https://registry-1.docker.io/v2/": dial tcp: lookup registry-1.docker.io on 192.168.65.5:53: no such host
$ 
$ DOCKER_BUILDKIT=1 docker build .
[+] Building 0.2s (3/3) FINISHED
 => [internal] load build definition from Dockerfile                      0.0s
 => => transferring dockerfile: 37B                                       0.0s
 => [internal] load .dockerignore                                         0.0s
 => => transferring context: 2B                                           0.0s
 => ERROR [internal] load metadata for docker.io/library/alpine:latest    0.0s
------
 > [internal] load metadata for docker.io/library/alpine:latest:
------
failed to solve with frontend dockerfile.v0: failed to create LLB definition: failed to do request: Head "https://registry-1.docker.io/v2/library/alpine/manifests/latest": dial tcp: lookup registry-1.docker.io on 192.168.65.5:53: no such host
```

Enabling the network allow either one of them to succeed, but they seem to have
a different effect on the caches.

## Build without BuildKit, then disable network
If you run the non-BuildKit command in a clean environment *with connectivity*,
you should get something like:

```
$ DOCKER_BUILDKIT=0 docker build .
Sending build context to Docker daemon     64kB
Step 1/1 : FROM alpine
latest: Pulling from library/alpine
29291e31a76a: Pull complete
Digest: sha256:eb3e4e175ba6d212ba1d6e04fc0782916c08e1c9d7b45892e9796141b1d379ae
Status: Downloaded newer image for alpine:latest
 ---> 021b3423115f
Successfully built 021b3423115f
```

Then disable/disconnect the network; you will still be able to run the commands successfully (and almost instantaneously):

```
$ DOCKER_BUILDKIT=0 docker build .
Sending build context to Docker daemon  64.51kB
Step 1/1 : FROM alpine
 ---> 021b3423115f
Successfully built 021b3423115f
$ 
$ DOCKER_BUILDKIT=1 docker build .
[+] Building 0.2s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                      0.1s
 => => transferring dockerfile: 55B                                       0.0s
 => [internal] load .dockerignore                                         0.1s
 => => transferring context: 2B                                           0.0s
 => [internal] load metadata for docker.io/library/alpine:latest          0.0s <==
 => [1/1] FROM docker.io/library/alpine                                   0.0s
 => exporting to image                                                    0.0s
 => => exporting layers                                                   0.0s
 => => writing image sha256:16a98022286ca55293845f2f506ae822f4588be116c9  0.0s
```

Notice that no time is spent to `load metadata for
docker.io/library/alpine:latest`; the locally cached copy is being used.

## Build without BuildKit, then disable network
If you run the BuildKit command in a clean environment *with connectivity*,
you should get something like:

```
$ DOCKER_BUILDKIT=1 docker build .
[+] Building 4.5s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                      0.1s
 => => transferring dockerfile: 55B                                       0.0s
 => [internal] load .dockerignore                                         0.0s
 => => transferring context: 2B                                           0.0s
 => [internal] load metadata for docker.io/library/alpine:latest          3.6s
 => [1/1] FROM docker.io/library/alpine@sha256:eb3e4e175ba6d212ba1d6e04f  0.7s
 => => resolve docker.io/library/alpine@sha256:eb3e4e175ba6d212ba1d6e04f  0.0s
 => => sha256:eb3e4e175ba6d212ba1d6e04fc0782916c08e1c9d7 1.64kB / 1.64kB  0.0s
 => => sha256:be9bdc0ef8e96dbc428dc189b31e2e3b05523d96d12ed6 528B / 528B  0.0s
 => => sha256:021b3423115ff662225e83d7e2606475217de7b55f 1.47kB / 1.47kB  0.0s
 => => sha256:29291e31a76a7e560b9b7ad3cada56e8c18d50a96c 2.81MB / 2.81MB  0.5s
 => => extracting sha256:29291e31a76a7e560b9b7ad3cada56e8c18d50a96cca8a2  0.1s
 => exporting to image                                                    0.0s
 => => exporting layers                                                   0.0s
 => => writing image sha256:16a98022286ca55293845f2f506ae822f4588be116c9  0.0s
```

Then disable/disconnect the network; the commands will fail as if the build had
not just been done.

```
$ DOCKER_BUILDKIT=0 docker build .
Sending build context to Docker daemon  68.61kB
Step 1/1 : FROM alpine
Get "https://registry-1.docker.io/v2/": dial tcp: lookup registry-1.docker.io on 192.168.65.5:53: no such host
$ DOCKER_BUILDKIT=1 docker build .
[+] Building 0.9s (3/3) FINISHED
 => [internal] load build definition from Dockerfile                      0.1s
 => => transferring dockerfile: 37B                                       0.0s
 => [internal] load .dockerignore                                         0.0s
 => => transferring context: 2B                                           0.0s
 => ERROR [internal] load metadata for docker.io/library/alpine:latest    0.8s
------
 > [internal] load metadata for docker.io/library/alpine:latest:
------
failed to solve with frontend dockerfile.v0: failed to create LLB definition: failed to do request: Head "https://registry-1.docker.io/v2/library/alpine/manifests/latest": dial tcp: lookup registry-1.docker.io on 192.168.65.5:53: no such host
```

If you re-enable the network, you will obviously be able to build again, but if
you keep doing the BuildKit build, it will be slow:
```
$ DOCKER_BUILDKIT=1 docker build .
[+] Building 1.6s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                      0.0s
 => => transferring dockerfile: 37B                                       0.0s
 => [internal] load .dockerignore                                         0.0s
 => => transferring context: 2B                                           0.0s
 => [internal] load metadata for docker.io/library/alpine:latest          1.4s <==
 => CACHED [1/1] FROM docker.io/library/alpine@sha256:eb3e4e175ba6d212ba  0.0s
 => exporting to image                                                    0.0s
 => => exporting layers                                                   0.0s
 => => writing image sha256:16a98022286ca55293845f2f506ae822f4588be116c9  0.0s
```

As expected, the `load metadata for docker.io/library/alpine:latest` step is now
requiring some time: without networking it fails, so it is actually fetching the
manifest instead of relying on a local copy; this makes the step slower (1.4s
BuildKit vs 0.0s non-) and impossible to use while offline.

Building from this state (i.e. network on + a BuildKit build completed) with
BuildKit disabled succeeds:
```
$ DOCKER_BUILDKIT=0 docker build .
Sending build context to Docker daemon  70.14kB
Step 1/1 : FROM alpine
latest: Pulling from library/alpine
29291e31a76a: Already exists
Digest: sha256:eb3e4e175ba6d212ba1d6e04fc0782916c08e1c9d7b45892e9796141b1d379ae
Status: Downloaded newer image for alpine:latest
 ---> 021b3423115f
Successfully built 021b3423115f
```

After this, it is possible to perform either build without connection:
```
$ DOCKER_BUILDKIT=0 docker build .
Sending build context to Docker daemon  70.66kB
Step 1/1 : FROM alpine
 ---> 021b3423115f
Successfully built 021b3423115f
$ DOCKER_BUILDKIT=1 docker build .
[+] Building 0.2s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                      0.0s
 => => transferring dockerfile: 37B                                       0.0s
 => [internal] load .dockerignore                                         0.0s
 => => transferring context: 2B                                           0.0s
 => [internal] load metadata for docker.io/library/alpine:latest          0.0s
 => CACHED [1/1] FROM docker.io/library/alpine                            0.0s
 => exporting to image                                                    0.0s
 => => exporting layers                                                   0.0s
 => => writing image sha256:16a98022286ca55293845f2f506ae822f4588be116c9  0.0s
```
