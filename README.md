# Build Containers the Hard Way \(DRAFT\)

_Like_ [_Kubernetes the Hard Way_](https://github.com/kelseyhightower/kubernetes-the-hard-way)_, but for building containers._

This guide is geared towards anyone interested in learning the low-level details regarding how to build containers. This guide is _not_ intended for those who wish to learn to build container images with high-level tools like Docker.

At the end of this guide, you should understand the internals of a container image, how to construct a container image, and how to push a container image to a Docker container registry piece-by-piece.

## What is a container image?

A _container_ is a way of executing processes with isolation provided by 3 Linux technologies - `chroot`, `namespaces`, and `cgroups`.

`chroot` changes the file system root \(`/`\) that a process can see. This allows a process to use any directory as if it were a file system root instead of the actual file system root.

`namespaces` group resources together \(like network and process IDs\) so that only processes within a namespace can see the resources of that namespace.

`cgroups` set CPU and memory limits for processes.

The combination of these allows processes to run in isolation from other processes both on the file system level and on the resource utilization level. Processes inside a container do not see other processes in other containers. Rather, they only see their own view of the system that is not cluttered other processes.

A _container image_ a way to package up an application so that it can run as a _container_. This package includes the application and any run-time dependencies and is simply a directory of files along with metadata about how to run the container.

{% hint style="success" %}
A _container image_ is simply a directory of files along with metadata about how to run the container.
{% endhint %}

[Containers from Scratch](https://ericchiang.github.io/post/containers-from-scratch/) is a good article explaining how to run your own containers using simple Linux commands.

### Build a container image with Docker

Docker is the most popular tool for working with container images. It has built-in support for building and running containers. Docker defines its own scripting language for defining how to build a container image.

For example, this Dockerfile builds a simple Docker image that serves any static files in the current working directory as webpages:

{% code-tabs %}
{% code-tabs-item title="Dockerfile" %}
```text
FROM python
COPY . /public
WORKDIR /public
ENTRYPOINT ["python3", "-m", "http.server"]
```
{% endcode-tabs-item %}
{% endcode-tabs %}

To build the Docker image, save the `Dockerfile` in the current working directory and tell Docker build it:

```bash
$ docker build .
```

```text
Sending build context to Docker daemon  3.072kB
Step 1/4 : FROM python
 ---> 338b34a7555c
Step 2/4 : COPY . /public
 ---> edcd805ec657
Step 3/4 : WORKDIR /public
 ---> Running in cd5f924a79fe
Removing intermediate container cd5f924a79fe
 ---> bba7b6ca42fc
Step 4/4 : ENTRYPOINT ["python", "-m", "http.server"]
 ---> Running in 4198534c1c9c
Removing intermediate container 4198534c1c9c
 ---> 5550043a7340
Successfully built 5550043a7340
```

The logs of the `docker build` command clearly show how Docker executes the build based on our Dockerfile.

The `docker` CLI is the client that you use to send commands and data to the _Docker daemon_. The Docker daemon stores and runs containers. First, the `docker build .` command tells Docker to use the local current working directory \(`.`\) as the _Docker context_. The Docker context is sent to the Docker daemon.

The commands in the Dockerfile are then run in the given order. The Docker daemon runs a container to execute each command and generates a new container image at the end of each step.

The `FROM python` step tells Docker to build the new container starting with the [python container image from Docker Hub](https://hub.docker.com/_/python) as the base.

`COPY . /public` copies all the files in the Docker context into the `/public` directory on our container image. This includes all the files in our local current working directory that was sent over to the daemon as the Docker context.

`WORKDIR /public` sets the working directory for the container image we are building, so that when the container image is run, the current working directory for the process is `/public`.

{% hint style="info" %}
Note that the container that was created by the `WORKDIR` command is removed. Steps that do not change the container file system only modify the _container configuration_ and are removed. The container configuration is metadata that describes how to run the container entrypoint process.
{% endhint %}

`ENTRYPOINT ...` sets the command to run when the container is run, which, in this case, runs an HTTP server that serves all the files in `/public`. So, for example, if we had HTML files in our local current working directory, those would be served by this container.

The final image built has ID `5550043a7340`, which can be used in subsequent `docker` commands.

This container can then be run with:

```bash
$ docker run -p 8000:8000 5550043a7340
```

This runs the container and forwards the port to `localhost:8000`. If you had, say, an `index.html` in your local current working directory, going to `localhost:8000` would serve the contents of `index.html`.

{% hint style="info" %}
Another common Dockerfile instruction is `RUN <command>`, which executes the `<command>` using whichever shell is present in the container, creating a new container following the result of executing that command.
{% endhint %}

### Advanced Docker Builds

Docker has its own caching mechanism that helps Dockerfile-based builds run faster. Essentially, Docker caches the container image generated after each Dockerfile step. If it determines that a step has not changed, it will use the container image cached for that step rather than run that step again. However, if a step changes, it will invalidate the cache for that step and all steps afterwards. The way the caching mechanism works in Docker merits some tips for writing a more efficient Dockerfile.

#### Frequently changing steps go last

Dockerfile builds can take quite a while, especially with heavy steps that download dependencies or compile your application. Since any step that changes invalidates steps afterwards, better Dockerfiles tend to place more frequently-changing steps last. Heavy steps that do not change much tend to be front-loaded so that subsequent builds can just use the cached result.

#### **Keep the number of steps minimal**

Since each step generates a separate cached container image, steps should be kept minimal to reduce the number of layers \(each cached step has its own overhead\). For example, a group of `RUN`s can usually be combined:

_Bad:_

```
RUN mkdir mydirectory
RUN touch mydirectory/myexecutable
RUN chmod +x mydirectory/myexecutable
```

_Better:_

```text
RUN mkdir mydirectory && \
    touch mydirectory/myexecutable && \
    chmod +x mydirectory/myexecutable
```

However, frequently-changing steps should not be combined with infrequently-changing heavy steps - rather, an efficient Dockerfile should try to separate these as much as possible to avoid running infrequently-changing steps.

{% hint style="warning" %}
One important caveat to note is how Docker caches RUN instructions. Even though RUN commands may produce different results each time, Docker only invalidates the cache for a RUN instruction if the command itself changes. This may produce unexpected results if you intended a RUN instruction to not use a cached version.
{% endhint %}

#### **Use multistage builds for more flexibility than just a linear Dockerfile build**

> TODO

## **Container image format**

Container images can be built in a specific format. Different runtimes may support different formats. The most common formats are the Docker image format and Open Container Initiative \(OCI\) format.

> TODO: Support for formats, like in Kubernetes \(CRI\)

### Layers

As mentioned, a container image is simply a directory of files \(along with metadata\). In the image format, this directory of files is split up into layers that are compatible with AUFS.

Layers are independent, but are composed together with a defined order in the image format. The order of the layers in the image matters for files that may exist in multiple layers. The contents of a file that appears in a later layer replaces the contents of the same file in a prior layer. A file can also be deleted in a later layer by prepending its filename with `.wh.`.

Each layer is archived into a single tarball. In the Docker image format, this tarball must be compressed with GZIP compression.

Caveat: when run, Docker shares common layers between containers and creates another overlay layer for file system writes - any writes to existing files would be a copy-on-write. This overlay layer is deleted when the container is removed.

> TODO: Show some bash commands that can generate a layer

### Digests

BLOBs \(like layer archives\) in the image format are identified by their digests. These digests are generated by hashing the BLOB contents. The digest for a layer would be the hash of the compressed tarball archive. The hashing algorithm to use for Docker image format is SHA-256, but [OCI format supports SHA-512 as well](https://github.com/opencontainers/image-spec/blob/master/descriptor.md#registered-algorithms). The benefit of identifying BLOBs is that they become content-addressable. Uniqueness is provided by the collision-resistance of SHA-256.

> TODO: Add example descriptor digest

### Container configuration

The metadata for a container image is represented by its container configuration. This container configuration contains mainly information about how the container should be run. This includes fields such as for architecture \(GOARCH\) and OS \(GOOS\), exposed ports, volume mounts, and environment variables.

The most important of the runtime configuration found in the container configuration is the entrypoint, which is the command that is run when the container is run. This essentially defines the entrypoint process the container is isolating.

Note that all runtime configuration in the container configuration exist as defaults. When actually running the container either with Docker or with Kubernetes, any/all of these runtime configuration can be overridden.

> TODO: Might want to talk about reasonings for burning runtime config into an image vs. defining in a runtime spec

> TODO: Show example container configuration and running with Docker/Kubernetes and effective runtime configuration

There are two other parts of the container configuration: the `rootfs` object and the `history` array. Both contain metadata about the layers that make up the container file system.

The `rootfs` contains a list of _diff IDs_. A diff ID for a layer is the digest \(usually SHA256 hash\) of the **uncompressed** tarball archive containing the files for that layer. Note that this is different from the descriptor digest, which is a hash of the compressed archive. The `rootfs` defines a list of these diff IDs in the order in which the layers belong in the container overlay file system, from first to last. Note that these layers must match those defined in the _manifest_.

### Manifest

A container image _manifest_ describes the components that make up a container image. Manifests come in multiple forms. These forms include the registry manifest formats such as _Docker Image Manifest V 2, Schema 2_ and _OCI Image Manifest_, as well as manifest formats that the Docker toolchain uses to save and load images stored locally.

The main information in a manifest are the components that make up a container image. These components include the layers and the container configuration.

#### `docker load` format

`docker load` can load a container image stored as a tarball archive into the Docker local repository store. This tarball archive includes the compressed tarball archives for all the layers, the container configuration JSON file, and the manifest JSON file \(must be named `manifest.json`\). Here’s example of the manifest JSON file:

```javascript
[
  {
    "Config":"config.json",
    "RepoTags":["myimage"]
    "Layers": [
      "layer1.tar.gz",
      "layer2.tar.gz",
      "layer3.tar.gz"
    ]
  }
]
```

> TODO: Add citation for [https://github.com/moby/moby/blob/master/image/tarexport/load.go](https://github.com/moby/moby/blob/master/image/tarexport/load.go#L55)

When Docker loads this image tarball, Docker finds this `manifest.json` and reads it. The manifest tells Docker to load the container configuration from the `config.json` file and load the layers from `layer1.tar.gz`, `layer2.tar.gz`, and `layer3.tar.gz`, in that order. Note that this order must match the order of the layers defined in the container configuration under `rootfs`. The `RepoTags` here tells Docker to "tag" the image with the name `myimage`. Note that "tag" is a confusing term here since it refers to the full reference for the image, whereas "tag" in an actual image reference refers to a label for an image stored under a repository.

#### **`docker save` format**

`docker save` can also save images in a legacy Docker tarball archive that `docker load` also supports. In this legacy format, each layer would be stored in its own directory named with its SHA256 hash \(digest of compressed layer tarball archive\). Each of these directories contains a `json` file with a legacy container configuration \(we won’t go into the details of this\), a `VERSION` file, and a `layer.tar` that is the uncompressed tarball archive of the layer contents.

#### **Registry format - Docker Image Manifest V 2, Schema 2**

Registry image manifests define the components that make up an image on a container registry \(see section on container registries\). The more common manifest format we’ll be working with is the _Docker Image Manifest V2, Schema 2_ \(more simply, _V2.2_\). There is also a _V2, Schema 1_ format that is commonly used but more complicated than V2.2 due to backwards-compatibility reasons against V1.

The V2.2 manifest format is a JSON blob with the following top-level fields:

`schemaVersion`  - `2` in this case

`mediaType`  - `application/vnd.docker.distribution.manifest.v2+json`

`config`  - _descriptor_ of container configuration blob

`layers`  - list of descriptors of layer blobs, in the same order as the `rootfs` of the container configuration

Blob _descriptors_ are JSON objects containing 3 fields:

`mediaType`  - `application/vnd.docker.container.image.v1+json` for a container configuration or `application/vnd.docker.image.rootfs.diff.tar.gzip` for a layer

`size`  - the size of the blob, in bytes

`digest`  - the digest of the content

Here is an example of a V2.2 manifest format \(for the Docker Hub [`busybox`](https://hub.docker.com/_/busybox) image\):

```javascript
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 1497,
    "digest": "sha256:3a093384ac306cbac30b67f1585e12b30ab1a899374dabc3170b9bca246f1444"
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 755724,
      "digest": "sha256:57c14dd66db0390dbf6da578421c077f6de8e88edd0815b4caa94607ba5f4c09"
    }
  ]
}
```

This manifest states that the `busybox` image is composed of a container configuration stored as a blob with digest `sha256:3a093384ac306cbac30b67f1585e12b30ab1a899374dabc3170b9bca246f1444` and a single layer blob with digest `sha256:57c14dd66db0390dbf6da578421c077f6de8e88edd0815b4caa94607ba5f4c09`. Note that manifests simply contain references. The actual blob content is stored elsewhere and would need to be fetched and provided to a container runtime when running the image as a container.

#### **Registry format - OCI Image Manifest**

The OCI image manifest is also a registry image manifest that defines components that make up an image. The format is essentially the same as the Docker V2.2 format, with a few differences.

`mediaType`  **-** must be set to `application/vnd.oci.image.manifest.v1+json`

`config.mediaType`  - must be set to `application/vnd.oci.image.config.v1+json`

Each object in `layers` must have `mediaType` be either `application/vnd.oci.image.layer.v1.tar+gzip` or `application/vnd.oci.image.layer.v1.tar`. Note that OCI allows for layer blobs to be not compressed with GZIP with the `application/vnd.oci.image.layer.v1.tar` `mediaType`.

#### **Manifest List/Image Index**

A _manifest list_ \(or _image index_ in OCI terms\) is a way to specify multiple manifests for a single image. Each manifest is associated with a specific platform \(operating system and architecture\) so that a container runtime can choose the appropriate manifest to use for the platform it is running on. Manifest lists are fairly uncommon and we will not go into details about them. See official documentation for more information about a [manifest list ](https://docs.docker.com/registry/spec/manifest-v2-2/#manifest-list) or [image index](https://github.com/opencontainers/image-spec/blob/master/image-index.md).

### Let’s explore a Docker image with docker

> TODO

## **Container image registry**

In order for a container orchestration system like Kubernetes to run container imgaes, it needs to pull the container imges from some container image repository. This is where container image registries come in. Container image registries store container images to be served via specific addresses called **image references**. These registries are usually blob stores that store the container image layers in separate blobs. They also store manifests that describe which blobs make up a specific image. The most common container image registry is the Docker Registry V2, which is a API specification for a registry server. Some examples of image registries include Docker Hub \(the default image registry for Docker\), Google Container Registry \(GCR\), AWS Elastic Container Registry \(ECR\), Microsoft Azure Container Registry \(ACR\), [Harbor](https://github.com/goharbor/harbor), [JFrog Artifactory](https://www.jfrog.com/confluence/display/RTF/Getting+Started+with+Artifactory+as+a+Docker+Registry), [Quay](https://quay.io/), and [Sonatype Nexus](https://help.sonatype.com/repomanager3/private-registry-for-docker).

We will go over the details of _Docker Registry API V2_ \(we’ll just refer to it as the _Registry API_\), since most container registries implement that specification. For the full specification details, read the [Docker Registry HTTP API V2 spec](https://docs.docker.com/registry/spec/api/).

### **How a registry stores an image**

Although the specific implementation details are undefined and differ between each implementation of the Registry API, conceptually, the registry can be thought of as a blob store that can have blobs pushed to and pulled from the registry. The Registry API works with the Docker/OCI Image Format, so the blobs would be the layers and the container configurations. The manifests are not referred to as blobs, but rather are the artifacts that actually represent an image \(by listing its component blobs\).

![Topology of a registry](.gitbook/assets/build-containers-the-hard-way-section-drafts.png)

The diagram shows the topology of a registry. A registry contains multiple repositories containing images. Each repository contains a set of blobs. An “image” is actually a manifest that refers to a sequence of blobs that make up the image. Tags are labels used to name manifests. A tag can only point to a single manifest but multiple tags can point to the same manifest.

### Image reference

An image is identified by an image reference. This image reference provides a human-readable way to address images and is composed of a few components:

* _Registry_ - The URL to reach the registry server. By default, this registry would be Docker Hub \(`registry.hub.docker.com`\).
* _Repository_ - Also known as the _namespace_, repositories help organize images into their own “directories” on the registry. Authentication applies on a per-repository basis. When the registry is Docker Hub, the default repository prefix for a single-level repository is `library`.
* _Tag_ - Each image within a repository is addressable by the digest of its manifest. However, digests can be difficult to manage and is not human-friendly. Tags can be set to point to specific digests to more conveniently identify specific images. These tags can also be reassigned to different digests as well. A specific image can also have multiple tags pointing to it. Each new manifest must go under a specific tag and the default tag is `latest`.

For example, consider the following image reference: `openjdk:8-jre-alpine`

* Since there is no registry URL specified, the registry is by default Docker Hub. The repository component is specified as `openjdk`. Since the repository is only a single level and the registry is Docker Hub, the repository is actually resolved as [library/openjdk](https://hub.docker.com/_/openjdk). The tag component is specified after a colon. In this example, the tag is `8-jre-alpine`. Although this tag is arbitrarily-defined, the maintainers of this repository chose to convey some information about the image through this tag. Here, this tag says that the specific image contains OpenJDK 8 with just the JRE \(Java Runtime Environment\) and Alpine \(a tiny Linux distribution\). Some other tags in the repository refer to the same image but contain more specific information, like `8u191-jre-alpine3.9`. Other tags may be less specific, like `jre`, which, at the time of writing, refers to an OpenJDK 11 image with the JRE and Debian Stretch. This tag will most likely move to refer to newer OpenJDK versions as they become GA. The latest tag also refers to the latest stable, default image the maintainers believes users will find useful in most cases and will move the tag as new versions become stable. There are some general tips and best practices for managing tags for a repository. 

> TODO: Maybe explain some tips?

Consider another image reference: `gcr.io/my-gcp-project/my-awesome-app/message-service:v1.2.3`

* The registry URL here is `gcr.io`, which is the server URL for Google Container Registry. The repository is `my-gcp-project/my-awesome-app/message-service`, which in GCR, means that the repository is on the `my-gcp-project` Google Cloud Platform project and under a `my-awesome-app/message-service` Google Cloud Storage subdirectory. The tag is `v1.2.3`, which is used to identify the version of the `message-service` this container image contains.

An image can also be referred to by its specific image digest: `gcr.io/distroless/java@sha256:0430beea631408faa266e5d98d9c5dcc3c5f02c3ebc27a65d72bd281e3576097`.

There are also specific acceptable patterns for each component. The full regex can be found in the [Docker distribution code](https://github.com/docker/distribution/blob/master/reference/reference.go). For a brief summary:

* Registry - can be a standard DNS hostname \(no protocol\) with an optional port number
* Repository - slash-separated components that contain lowercase letters and digits separated by period, one or two underscores, or one or more dashes
* Tag - any of letters, digits, underscores, periods, and dashes for a maximum of 128 characters

Note that "tag" is a confusing name in Docker terminology since when using the local `docker` CLI tool, "tag" also refers to a label given to an image on the Docker daemon. For example, you might build an image called `myimage` with:

```bash
$ docker build -t myimage .
...
Successfully built 4e905a76595b
Successfully tagged myimage:latest
```

Docker uses the current directory as the context and builds an image with an image ID `4e905a76595b`. This image ID is a shortened SHA256 digest of the built container configuration. Docker then "tags" the `4e905a76595b` with a label `myimage`, which can be used to refer to the same image with future `docker` commands rather than using `4e905a76595b` all the time. However, when you want to build and push an image to a registry, you would "tag" the image with the full image reference:

```bash
$ docker build -t gcr.io/my-gcp-project/myimage:v1 .
...
Successfully built 4e905a76595b
Successfully tagged gcr.io/my-gcp-project/myimage:v1
$ docker push gcr.io/my-gcp-project/myimage:v1
```

Here, Docker "tags" the image with `gcr.io/my-gcp-project/myimage:v1` to be used in later docker commands, but `gcr.io/my-gcp-project/myimage:v1` is actually an image reference with its tag as `v1`.

### Pulling an image

Pulling an image involves two steps:

1. Pull the manifest.
2. Pull the blobs in the manifest.

#### Pull the manifest

To pull the manifest, send a request to:

```text
GET /v2/<repository>/manifests/<tag or digest>
```

For example, to get the manifest for `openjdk`, the endpoint would be `https://registry.hub.docker.com/v2/library/openjdk/manifests/latest`.

Headers to send include `Authorization` \(explained in the Token Authentication section\) and `Accept`. For example, if you want to only accept and parse manifests for Docker Image Format V2 Schema 2, you would want to set `Accept: application/vnd.docker.distribution.manifest.v2+json`.

{% hint style="info" %}
Manifests can also come in list format for some images. These would have a media type like `application/vnd.docker.distribution.manifest.list.v2+json`. These manifests lists contain specific manifests for different architectures and operating systems. For example, if you are looking specifically for a manifest for the amd64/linux platform, you would parse the manifest list for the corresponding manifest for that platform and then pull that manifest using its specific digest. See [an example of a manifest list](https://docs.docker.com/registry/spec/manifest-v2-2/#example-manifest-list).
{% endhint %}

#### Pull the blobs

After the manifest is pulled, the specific blob digests can be extracted for the container configuration and the layers. Each blob can be pulled in parallel via:

```text
GET /v2/<repository>/blobs/<digest>
```

You might want to verify that the size of the blob pulled is what you expected based on the size specified in the manifest.

### Pushing an image

Pushing an image is like pulling an image except in reverse:

1. Push the blobs.
2. Push the manifest.

#### Push the blobs

Blobs can be pushed in parallel and pushing each blob is a 3-step process:

1. Initialize upload.
2. Send content.
3. Commit with digest.

Initialize the upload by sending a request to:

```http
POST /v2/<repository>/blobs/uploads/
```

A successful `202 Accepted` response should contain a `Location` header with the URL to send the blob content to. There are a few ways to send the blob content, but the recommended way is to send the content in a single request. This request should be automatically chunked as an octet stream. Send the blob content to the `Location` via:

```http
PATCH <Location>

<blob binary data>
```

The success response should be `202 Accepted`. Once a blob is uploaded, the blob can be committed with its digest:

```http
PUT <Location>?digest=<digest>
```

The success response should be `201 Created`.

#### Avoid unnecessary pushes

**Check if blob already exists**

To avoid pushing a blob that already exists on a registry, if the blob’s digest is known beforehand, you can check the existence of the blob with:

```http
HEAD /v2/<repository>/blobs/<digest>
```

If the blob exists, the response will be `200 OK`.

**Cross-repository blob mount**

If the blob exists in another repository in the same registry, you can also try directly mounting the blob into the target repository:

```http
POST /v2/<repository>/blobs/uploads/?mount=<digest>&from=<source repository>
```

If the blob mount succeeds, the response will be `201 Created`. If the blob mount fails \(blob doesn’t exist or cross-repository blob mount not supported\), the response will be a normal blob upload initialization response with a `Location` to send the blob content to.

An important aspect to note is that the `Authorization` to send with this request must have permissions to pull from the source repository in addition to permissions to push to the target repository. See the Token Authentication section below for details.

#### Push the manifest

Once all the blobs are committed, the manifest can be pushed via:

```http
PUT /v2/<repository>/manifests/<tag>
```

Make sure to set the `Content-Type` to the correct manifest media type. For Docker V2.2, the media type would be `application/vnd.docker.distribution.manifest.v2+json` and for OCI, the media type would be `application/vnd.oci.image.manifest.v1+json`.

### Authenticating requests

Most registry requests would require a sufficient `Authorization` header to be authenticated. Otherwise, a `401 Unauthorized` or `403 Forbidden` response would be received.

The standard flow for any request should be:

1. Try the request.
2. If unauthorized, obtain a token.
3. Retry the request with the token.

For example, an authorized response may include a [WWW-Authenticate header](https://tools.ietf.org/html/rfc6750#section-3) like:

```http
Www-Authenticate: Bearer realm="https://auth.docker.io/token",service="registry.docker.io",scope="repository:coollog/myimage:pull,push"
```

To obtain a token, send the following request along with an `Authorization` header containing the credentials for access to the repository:

```http
GET https://auth.docker.io/token?service=registry.docker.io&scope=repository:coollog/myimage:pull,push
```

For example, if the Docker Hub password for the `coollog` account was `donthackme123`, the `Authorization` header would be `Authorization: Basic Y29vbGxvZzpkb250aGFja21lMTIz`, where `Y29vbGxvZzpkb250aGFja21lMTIz` is the Base64 encoding of `coollog:donthackme123`.

The response should be a JSON containing either a `token` or `access_token` \(OAuth 2.0\) that is the authenticated Bearer token. The JSON also may also contain an `expires_in` and `issued_at` field for knowing when the token expires. These tokens should be cached and applied to any subsequent requests \(until expired\) to skip steps 1 and 2.

Note that multiple `scope`s can be included. For example, for a cross-repository blob mount with `otheruser/baseimage` as the source repository, the request may be:

```http
GET https://auth.docker.io/token?service=registry.docker.io&scope=repository:coollog/myimage:pull,push&scope=repository:otheruser/baseimage:pull
```

This request would also include two `Authorization` headers, with the additional `Authorization` containing credentials for pulling from `otheruser/baseimage`.

#### Where to find registry credentials

If a user has Docker installed, they most likely have it configured with credentials to use for various repositories they have access to. This configuration can be found at the `.docker/config.json` file located in the user’s home directory. This configuration can store both raw credentials and references to credential helpers, where credentials can be fetched from.

The raw credentials are stored in an `auths` field that is a map with registry URLs as keys. For example:

```javascript
{
  "auths": {
    "https://gcr.io": {
      "email": "not@val.id",
      "auth": "<Base64-encoded username:password>"
    }
  }
}
```

The credential helper references are stored in a `credHelpers` field that maps from registry URLs to credential helper names. For example:

```javascript
{
  "credHelpers": {
    "gcr.io": "gcr",
    "us.gcr.io": "gcr",
    "aws_account_id.dkr.ecr.region.amazonaws.com": "ecr-login"
  }
}
```

A default credential helper can also be specified in a `credsStore` field. For example:

```text
{
  "credsStore": "osx-keychain"
}
```

Note that registry URLs can come with or without a protocol and with or without arbitrary suffixes. For example, the following all refer to `gcr.io`:

```text
gcr.io
https://gcr.io
gcr.io/v2/
https://gcr.io/v2/
```

Credential helpers are command-line tools that can be used to fetch credentials for a registry. To fetch credentials from a credential helper, call the credential helper `get` command, passing in the server URL as the input. For example, to fetch credentials from the credential helper named `gcr` for the `gcr.io` registry, call:

```bash
$ echo gcr.io | docker-credential-gcr get
```

If the credentials are present, the output should be a JSON that includes a `Username` and `Secret` field which can be used as the username and password, respectively, for authenticating a token.

### Local caching

When building an image by constructing layers on top of a base image, it may be expensive to pull base image layers every time. To resolve this, there should be some local cache to store images.

For example, Jib uses a simple cache mechanism that stores blobs in a specific directory structure. The cache directory contains two subdirectories: `blobs/` and `selectors/`. The `blobs/` directory contains files for each blob named by their digest. The `selectors/` directory contains **selector** files whose names are the selectors themselves and contents are the digests of the blobs they point to. Both blobs and selectors should be immutable and unique.

This design supports fetching a blob by digest and by selector. When writes are made atomic, the cache can be used fully concurrently across multiple blobs. Selectors are useful for “finding” a blob by some representation of its contents. For example, in Jib, some blobs are layers that were built from the Java application source code. The selectors for these blobs are a list of files that went into building that layer. The name of a selector is the SHA256 digest of that file list. Uniqueness is provided by associating each file with its last modified time and immutability is provided by the determinism of SHA256. What this allows Jib to do is that when files don’t change, they will always generate the same selector and give back an existing blob so that that blob does not need to be re-generated; but, when files do change, the generated selector is different and a new selector-blob pair will be added to the cache.

An example cache directory layout could look like:

```text
blobs/
  36a2b7401dcddc50a35aeaa81085718b9d5fbce9d607c55a1d79beec2469f9ac
  ...
selectors/
  65de3b72aaf98e4f300ccdf7d64bf9a3b1e23c8c44a1242265f717db1a0877e9
  ...
```

Here, the `65de3b72…` selector file contains the 36a2b740… digest. See the proposal for the original [Jib cache design](https://github.com/GoogleContainerTools/jib/blob/master/proposals/archives/cache_v2.md).

#### Large caches

As caches grow in size, they may need to be pruned. A simple mechanism is to use a least-recently-used method. For caches that grow large, however, resolving a blob or selector may become expensive since directory traversal can is at best logarithmic to the number of directory entries. A simple technique to alleviate this would be to shard the directory into a number of bins, and potentially do so recursively within each bin. For example, the bins could be named `00` through `ff`, with each file being placed into the bin that matches the first two characters in its file name.

### Walk through pulling an image with bash

> TODO

## Making an efficient container builder

> TODO
>
> * **Separate into fine layers**
> * **Minimize change**
> * **stream everything through sinks/sources**

## Useful links

* [https://www.danlorenc.com/posts/containers-part-1/](https://www.danlorenc.com/posts/containers-part-1/)
* [https://www.danlorenc.com/posts/containers-part-2/](https://www.danlorenc.com/posts/containers-part-2/)
* [https://github.com/opencontainers/image-spec/blob/master/config.md](https://github.com/opencontainers/image-spec/blob/master/config.md)
* [https://github.com/opencontainers/image-spec/blob/master/manifest.md](https://github.com/opencontainers/image-spec/blob/master/manifest.md)
* [https://github.com/opencontainers/image-spec/blob/master/image-index.md](https://github.com/opencontainers/image-spec/blob/master/image-index.md)
* [go/microlayers](http://go/microlayers)
* [go/millilayers](http://go/millilayers)

