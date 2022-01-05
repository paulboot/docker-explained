# Docker Image Tag and why latest is confusing and breaks stuff Explained

[toc]

# TL;DR

* Images running for months have the tag latest

* If you pull the latest image, your old image has no tag at-all only an ID

* >  **NOTE:** If you `down` and `up` (not `start\stop`) an old container and forgot that you pulled the latest image you potentially upgrade from release 1.* to 2.* and break configs or database schema's!

* This will show how an image was billed and perhaps show a version number:

  * `docker image ls`
  * `docker history <Image-ID>`

* Use this to find a matching version if maintainers put the version number in the labels does not work for all images: https://ryandaniels.ca/blog/find-version-tag-latest-docker-image/

  ```
  $ docker images
  REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
  traefik             latest              96c63a7d3e50        2 months ago        85.7MB
  
  $ IMAGE_ID=96c63a7d3e50
  
  $ docker image inspect --format '{{json .}}' "$IMAGE_ID" | jq -r '. | {Id: .Id, Digest: .Digest, RepoDigests: .RepoDigests, Labels: .Config.Labels}'
  ```

* 

* Images that do not have a tag are images that used to have the `: latest` tag and are called dangling images

  * The `docker image prune` command allows you to clean up unused images. By default, `docker image prune` only cleans up *dangling* images. A dangling image is one that is not tagged and is not referenced by any container. To remove dangling images:

    ```
    $ docker image prune
    
    WARNING! This will remove all dangling images.
    Are you sure you want to continue? [y/N] y
    ```

    To remove all images which are not used by existing containers, use the `-a` flag:

    ```
    $ docker image prune -a
    
    WARNING! This will remove all images without at least one container associated to them.
    Are you sure you want to continue? [y/N] y
    ```

    By default, you are prompted to continue. To bypass the prompt, use the `-f` or `--force` flag.

    You can limit which images are pruned using filtering expressions with the `--filter` flag. For example, to only consider images created more than 24 hours ago:

    ```
    $ docker image prune -a --filter "until=24h"
    ```

    Other filtering expressions are available. See the [`docker image prune` reference](https://docs.docker.com/engine/reference/commandline/image_prune/) for more examples.





# Finding : latest version

If you’ve ever used Docker, you’ve probably used the latest Docker image tag. This is bad. Do not do this! You will be in a situation where you need to find what version you were actually using. This is how you can find the version of that “latest” image you have running.

**Every time** you build a new image, use a **new version** for your image tags.
**Every time** you pull an image, use a specific image tag **version**.

This [article](https://vsupalov.com/docker-latest-tag/) describes why latest is bad.

For me, in this theoretical example.. An image I was using introduced breaking changes. This is fine, since they tagged the new releases as version 2. But, only if I was actually using a specific image version and not latest would this be okay. Latest is latest. So the latest that was version 1, now turned into version 2. And everything broke.

Now that you will never use latest again, you could still have a problem. All those docker containers you have running are using images with the :latest tag. How do you find what version latest was?

## I hope you’re Lucky.. Trick #1

Check the labels on your image. If you’re lucky, the developer added a label to the image with the version. I’m using the utility `jq` below. It’s very helpful, be sure to install it.

```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
traefik             latest              96c63a7d3e50        2 months ago        85.7MB

$ IMAGE_ID=96c63a7d3e50

$ docker image inspect --format '{{json .}}' "$IMAGE_ID" | jq -r '. | {Id: .Id, Digest: .Digest, RepoDigests: .RepoDigests, Labels: .Config.Labels}'
```

Output:

```
{
  "Id": "sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834",
  "Digest": null,
  "RepoDigests": [
    "traefik@sha256:5ec34caf19d114f8f0ed76f9bc3dad6ba8cf6d13a1575c4294b59b77709def39"
  ],
  "Labels": {
    "org.opencontainers.image.description": "A modern reverse-proxy",
    "org.opencontainers.image.documentation": "https://docs.traefik.io",
    "org.opencontainers.image.title": "Traefik",
    "org.opencontainers.image.url": "https://traefik.io",
    "org.opencontainers.image.vendor": "Containous",
    "org.opencontainers.image.version": "v1.7.20"
  }
}
```

Bingo! That’s version 1.7.20 of Traefik!

## Lucky Trick #2

Pull a bunch of images and hope the Image ID matches. Not much to this trick..

## Lucky Trick #3

If your image is from Docker Hub, they have a new experimental tool you can use called “[Docker Hub Tool](https://github.com/docker/hub-tool)“.

But what if you are unlucky? What if there’s a way to check all version tags of an image?

## Find Version Tag for Latest Docker image

There’s a way to check all version tags on Docker Hub (for example), against the local docker image’s “Image ID”.

You can get every tag from a Docker Registry (like Docker Hub), then use every tag you found, to get the image ID information from the manifest of every image.

Docker Hub has some quirks compared to a proper Docker Registry, and the API isn’t well documented. But let’s focus on Docker Hub, since that’s a huge public image repository.

If you don’t want to do all of this manually, you can use my script (on GitHub). Here’s the script in action:

```
# ./docker_image_find_tag.sh -n traefik -i 96c63a7d3e50 -f 1.7. -l 10 -v

Using IMAGE_NAME: traefik
Using REGISTRY: https://index.docker.io/v2
Found Image ID Source: sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
Found Total Tags: 610
Found Tags (after filtering): 178
Limiting Tags to: 10
Limit reached, consider increasing limit (-l [number]) or use more specific filter (-f [text])

Found Tags:
v1.7.21-alpine
v1.7.21
v1.7.20-alpine
v1.7.20
v1.7.19-alpine
v1.7.19
v1.7.18-alpine
v1.7.18
v1.7.17-alpine
v1.7.17

Checking for image match..
Found match. tag: v1.7.20
Image ID Target: sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
Image ID Source: sha256:96c63a7d3e502fcbbd8937a2523368c22d0edd1788b8389af095f64038318834
```

This example is searching for an image named traefik, with an image ID of 96c63a7d3e50 (which we got from running `docker images`.

The total number of tags for traefik images is 610! You could use this script to check every one, but that will take a minute or two. Instead, you can filter the results a bit. I *think* I’m running something in version 1.7.x. I’m also limiting the search to 10 tags in this example to keep the output small.

And the find version tag script found it! The local image ID matches an image ID on the Docker Registry with a tag of 1.7.20!

You can find the script on GitHub: https://github.com/ryandaniels/docker-script-find-latest-image-tag

Some issues to note:
Issue #1: If the image no longer exists on the Docker Registry. You obviously won’t find it.
Issue #2: The script doesn’t work for Windows images. There are no manifests available for those.

Side note: In this example I’m using the traefik image. What is [traefik](https://containo.us/traefik/)? It’s “The Cloud Native Edge Router” which means it’s a reverse proxy and load balancer for HTTP and TCP-based applications. It works really well from my experience! Just be careful when upgrading from version 1 to version 2.

# Why : latest is bad

> For me, “latest” is an anti-pattern
>
> Do not run any container with the latest tag. Been there, done that. Did not end very well.
>
> Latest is a proliferating ball of confusion that should be avoided like a ravine full of venomous snakes

Docker images tagged with :latest have caused many people a lot of trouble.

But what exactly is wrong with the :latest tag? Should you avoid it completely when working with Docker images?

Let’s go over the most frequent misconceptions and ways in which the :latest tag can cause suffering - and how to avoid the :latest pain.

## Latest is Just a Tag

> The ‘latest’ tag does not actually mean latest, it doesn’t mean anything

Some people expect that :latest always points to the most-recently-pushed version of an image. That’s not true.

It’s just the tag which is applied to an image by default which **does not have a tag**. Those two commands will both result in a new image being created and tagged as :latest:

```
# those two are the same:
$ docker build -t company/image_name .
$ docker build -t company/image_name:latest .
```

Nothing magical. Just a default value.

## Latest is Not Dynamic

> So many people are not realizing that :latest is not dynamic

If you push a new image with a tag which is neither empty nor ‘latest’, :latest will not be affected or created.

```
$ docker build -t company/image_name:0.1 .
# :latest doesn't care
$ docker build -t company/image_name
# :latest was created
$ docker build -t company/image_name:0.2 .
# :latest doesn't care
$ docker build -t company/image_name:latest .
# :latest was updated
```

If you are not pushing it explicitly, the :latest tag will stay the same.

## Latest is Easily Overwritten By Default

Latest is however the default value, which makes it both vulnerable to human mistakes and concurrent usage patterns.

> I’m weary of using latest on a team as I can see somebody forgetting to tag a build

Imagine your image building scripts are a bit off, or used incorrectly. Any developer who does not provide an image tag can overwrite the :latest image without meaning to.

Also, imagine the confusion which can be caused if you rely on :latest pointing to your most recent image, but another team member pushes their version in the meanwhile.

## Latest is Not Descriptive and Hard to Work With

The Kubernetes docs are [pretty clear](https://kubernetes.io/docs/concepts/configuration/overview/#using-labels) on using Docker images with the :latest tag in production environments:

> You should avoid using the :latest tag when deploying containers in production, because this makes it hard to track which version of the image is running and hard to roll back.

If you’re working with an image which is tagged with “latest”, that’s all the information you have apart from the image ID. Operations like deploying a new version of your app, or rolling back are simply not possible, if you don’t have two distinctly tagged images which are stable.

There’s no tool which will make it pleasant to deploy a “new :latest image” instead of the “old :latest image”.

## Are You Really Using the :latest You Intended to?

> The CI used the :latest tag […] suddenly and magically, people started having issues with their images

Depending on the tooling, you might be working with a stale image. Kubernetes is configured to **always** try to pull an image if it’s tagged with :latest. Even if a copy is already available.

Other tools might not be designed to be used with images which are updated while keeping the same label. This can cause confusion, complicated workflows and tricky bugs.

## Is :latest Evil?

No. But it’s misunderstood and can be used poorly without much effort.

“Latest” is just another tag name, but vulnerable to sloppy mistakes by design and people are expecting too much of it.

If you’re a developer, are building images locally for your own consumption and are careful about it - :latest can be a reliable way to save effort and frustration. However, you should not use it for deployments as that practice will backfire badly, repeatedly and in many interesting ways.

If you’re looking for a way to tag your Docker images, a safe bet is to use the Git commit hash as the image tag. This way, you will be able to:

* Tell immediately what code version is running based on the image tag
* Roll back and deploy new versions of your app
* Avoid naming collisions among developers
* Make accidental overwrites harder to do

When using “moving tags”, use custom ones instead of :latest - :stable, :master or a shorter version of semantic versioning are nice examples. This way you can avoid default overwrites by accident. Make sure that your tools handle different images with the same tag well.

If you want to implement [semantic versioning](https://semver.org/) or moving tags, it’s best to do so in addition to the SHA-based tags - just push the same image with multiple tags. Make sure to be careful about pushing different images with the same tag - do your workflows account for it?

And finally, don’t use moving tags for permanent or production deployments. Try to be as precise and specific as possible about what is running to make operations predictable and reliable to save yourself from yet another painful :latest-esque experience.

> You should always be explicit about versioning, otherwise things will break

# What's A Better Docker Tag Than :latest?

**Source:** https://vsupalov.com/docker-better-image-tags/

What’s a better naming scheme for Docker image tags than [the tricky :latest tag](https://vsupalov.com/docker-latest-tag/)? What naming schemes can help you have an easier time handling your Docker images, and make it harder to run into unexpected errors?

Let’s look at a few options!

## The Thing With “Reusing” Tags

If you use another tag than :latest, you might still run into issues if you end up “reusing” the same tag for different images.

Imagine tagging your staging images with `:staging`. If you’re using an orchestration tools, it might just look at the tag and refuse to download a new version of the image, because “there’s a :staging image right here, I don’t need another one”.

Of course, you can configure your orchestration tool to work around that - Kubernetes for example can be configured to “always try pulling” an image. (That’s the default behaviour if you tell it to use a :latest image by the way). However! Using such a `imagePullPolicy: always` policy can make deploys unnecessary slow. You’d have to try pulling a new image version each time after all.

## Tags As Artifact Names

In some way, the names of Docker images are very similar to build artifacts in the 12-factor kind of way. You can read about them [here](https://12factor.net/build-release-run). Build artifacts are only based on a certain version of the codebase, and can be “named” according to a commit or [semantic version](https://semver.org/) of the application being packaged.

## Tags As Release IDs

If you’re using Docker images to produce releases (bundling up configs as well as code), then it could even make sense to name them in a similar fashion as releases. Here’s the relevant part: “Every release should always have a unique release ID, such as a timestamp of the release (such as 2011-04-06-20:32:17) or an incrementing number (such as v100)“.

## So What’s A Good Tag?

Whether you prefer a commit hash as name, a semantic version or a steadily increasing number depends as much on personal taste as it does on your other deployment workflows.

You could tag your images with a “semantic version” of your app, you could tag them with a commit hash, you could tag them with an incrementing number or the date & time…

You can also decide to reuse tags (if you’re careful), but it can make some other things tricky: if you want to “go back” (aka roll back) to a previous image version overwriting previous image names isn’t handy.

## In Conclusion

There is no “one correct way” to do it, but multiple options which are all fine if you are fine with them. You can choose your own adventure and do what suits your taste and needs best.

If you need some more inspiration, you can check out how naming is handled in other ecosystem - [on Heroku](https://blog.heroku.com/releases-and-rollbacks) for example. Docker is just one of many tools for packaging up and shipping your code after all :)