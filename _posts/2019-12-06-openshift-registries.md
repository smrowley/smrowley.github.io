---
layout: post
title:  Openshift Registries, ImageStreams, Tagging and Running Out Of Space
date:   2019-12-06 17:00:00 +0530
categories: Openshift Registries Linux
---

### Image Registries

When deploying container-based applications, developers will need an image registry to push their application image releases after being built. For Java developers, an image registry can be seen as equivalent to a Maven repository for Java artifacts. The difference is that a container image contains all of the dependencies an application requires to run, not just the Java jars.

### Openshift

Openshift 3.x provides an internal Docker registry, whereas 4.x provides a Quay registry. Managing images is accomplished through `ImageStream` and `ImageStreamTag` resources, which are custom Openshift resources. They are an Openshift feature that I enjoy using, as it's easy to create, delete, and tag images.

#### ImageStreams

An `ImageStream` can be created with a definition as simple as the following:

```yaml
apiVersion: v1
kind: ImageStream
metadata:
  name: myimagestream
  namespace: mynamespace
  labels:
    application: myapp
```

Any tags on the `ImageStream` create new `ImageStreamTag` resources that can be managed. The `ImageStreamTag` references a specific image SHA in the container registry. The namespace of the `ImageStream` resource also corresponds with the namespace in the registry.

Example:

```uri
docker-registry.default.svc:5000/mynamespace/myimagestream:latest
```

#### BuildConfigs

An `ImageStream` becomes very powerful when coupled with Openshift's custom `BuildConfig` resource, which defines how to build a container image with Openshift. The `BuildConfig` allows you to easily specify an `ImageStreamTag` to target the output of new image builds. All that is required is for the `ImageStream` to exist.

Example:

```yaml
apiVersion: v1
kind: BuildConfig
metadata:
  name: mybuildconfig
spec:
  output:
    to:
      kind: ImageStreamTag
      name: myimagestream:latest
  ...
```

### Versioning

Many organizations will want to tag their images with something like [Semantic Versioning](https://semver.org/). It can be an easy way to track container images back to versions of the source code. This can be achieved with the following oc command:

```bash
oc tag myimagestream:latest myimagestream:v1.0.0
```

The challenge comes when the registry starts filling up with numerous images. Openshift has mechanisms to clean up older versions of tags, but it won't clean up tags themselves.

By describing an `ImageStream` resource, we can see details about all the tags (`is` is short for `imagestream`):

```bash
oc describe is myimagestream
```

The final details of the `ImageStream` will show us all the tags along with all the prior images associated with each tag. An asterisk next to an image SHA denotes the current image associated with the tag:

```bash
latest
  no spec tag

  * docker-registry.default.svc:5000/mynamespace/myimagestream@sha256:bb4215c2676fea1b6320d4e3be142ddfe2bf2c74a936739eb7fafb92117f93ac
      5 hours ago
    docker-registry.default.svc:5000/mynamespace/myimagestream@sha256:25a74e5651da9ab2035be2005616b055be0792bbf27c88a4886f67f4203e886f
      6 hours ago

v1.4.6
  tagged from myimagestream@sha256:25a74e5651da9ab2035be2005616b055be0792bbf27c88a4886f67f4203e886f

  * docker-registry.default.svc:5000/mynamespace/myimagestream@sha256:25a74e5651da9ab2035be2005616b055be0792bbf27c88a4886f67f4203e886f
      6 hours ago
```

You can see there are two images associated with the `latest` tag. Openshift can be configured to prune the older images associated with the tag. However, if there is a version tag for each semantic release, those images will not be pruned.

Another way of viewing `ImageStreamTags` is with the following command:

```bash
oc get imagestreamtag -l application=myapp
```

You will see something similar to the following:

```
NAME                     DOCKER REF                                                                                                                         UPDATED
myimagestream:v1.4.6   docker-registry.default.svc:5000/efm-dev/myimagestream@sha256:25a74e5651da9ab2035be2005616b055be0792bbf27c88a4886f67f4203e886f   6 hours ago
myimagestream:latest   docker-registry.default.svc:5000/efm-dev/myimagestream@sha256:bb4215c2676fea1b6320d4e3be142ddfe2bf2c74a936739eb7fafb92117f93ac   5 hours ago
```

#### Labels

Labels provide a way of filtering resources. The `-l` option allows you to specify a label like the example above: `application=myapp`. In our `ImageStream` example, we specify a label in the metadata like so:

```yaml
...
metadata:
  labels:
    application: myapp
...
```

Any Kubernetes resource can specify labels in the metadata, and are a very powerful feature once you get the hang of them. Because the `ImageStream` has a label, all of its associated `ImageStreamTags` will conveniently be created with the same labels as well. We will use them later.

### Writing a Pruning Script

If using a tagging pattern like semantic versioning, you'll probably want to prune your images. Luckily, we can do this quite simply with the `oc` tool and some shell commands. One of the tenets I enjoy about Linux/UNIX is the [Do one thing and do it well](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well) mantra. This allows for piping of Linux commands similar to [Map/Filter/Reduce patterns](https://web.mit.edu/6.005/www/fa15/classes/25-map-filter-reduce/) in functional programming.

Assuming the a tagging pattern of `v1.0.0`, the following command will delete all `ImageStreamTag` versions beyond the latest 5.

```bash
oc delete imagestreamtag $(oc get imagestreamtag -l application=myapp | awk '{print $1}' | grep -e ':v[0-9]\+.[0-9]\+.[0-9]\+' | sort -V | head --lines=-5)
```

Let's break it down:

The main command is `oc delete imagestreamtag`. This accepts a list of `ImageStreamTag` names. We can then use the BASH `$()` to pass a subcommand, which will return our list of `ImageStreamTag` names for deletion.

That brings us to our subcommand. The first portion simply returns all the `ImageStreamTags` with the label `application=myapp`.

```bash
oc get imagestreamtag -l application=myapp
```

Then we pipe the output of that command with the `|` operator. This is a very powerful Linux operator that allows us to pass the STDOUT of one command to another command, creating infinite combination possibilities. Because the `oc get` command gives us more information than we want, we need a way to get only the first column, which contains the `ImageStreamTag` names:

```
NAME                     DOCKER REF                                                                                                                         UPDATED
myimagestream:v1.4.6   docker-registry.default.svc:5000/efm-dev/myimagestream@sha256:25a74e5651da9ab2035be2005616b055be0792bbf27c88a4886f67f4203e886f   6 hours ago
myimagestream:latest   docker-registry.default.svc:5000/efm-dev/myimagestream@sha256:bb4215c2676fea1b6320d4e3be142ddfe2bf2c74a936739eb7fafb92117f93ac   5 hours ago
```

The `awk` command allows us to do this easily, only printing the first column of the piped output. You can think of this similar to a Map function that takes input and transforms it to new output.

```bash
awk '{print $1}'
```

Once we've stripped out all but the tag names, next, we can pipe it to `grep`. This command asks as a Filter function:

```bash
grep -e ':v[0-9]\+.[0-9]\+.[0-9]\+'
```

The `-e` option allows the passing of a regular expression. This will filter out all tags that do not match the semantic version pattern.

After filtering, we can sort all the versions from oldest to newest. The `sort` command has a convenient option, `-V`, for sorting output based on versioning schemes:

```bash
sort -V
```

And finally, we want to filter out the five newest tags, which are the last five lines after the sort. You can choose a number that makes sense for you. The `head` command is typically used to show the first 10 lines of a file, but with the `--lines` option, we can configure it to show all but the last N number:

```bash
head --lines=-5
```

### Running the Script

Running the full command will delete all `ImageStreamTags` except for the newest five:

```bash
$ oc delete imagestreamtag $(oc get imagestreamtag -l application=myapp | awk '{print $1}' | grep -e ':v[0-9]\+.[0-9]\+.[0-9]\+' | sort -V | head --lines=-5)
imagestreamtag.image.openshift.io "myapp:v2.0.1" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.2" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.3" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.4" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.5" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.6" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.7" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.8" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.9" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.10" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.11" deleted
imagestreamtag.image.openshift.io "myapp:v2.0.12" deleted
imagestreamtag.image.openshift.io "myapp:v2.1.0" deleted
imagestreamtag.image.openshift.io "myapp:v2.1.1" deleted
imagestreamtag.image.openshift.io "myapp:v2.1.2" deleted
imagestreamtag.image.openshift.io "myapp:v2.1.3" deleted
```

Running the `oc get` command shows that we still have our latest five version tags, plus the `latest` tag:

```bash
$ oc get imagestreamtag -l application=myapp
NAME                    DOCKER REF                                                                                                                            UPDATED
myapp:v2.1.4   docker-registry.default.svc:5000/mynamespace/myapp@sha256:795df99e65be7d8b0b473f0ca8877a1d6f836f296749e19ab1d1a374dd21a0f0   4 days ago
myapp:v2.1.5   docker-registry.default.svc:5000/mynamespace/myapp@sha256:23d4864cb193c99fdac27f11fe501f3e65aa5b44d728b9cf9aba8b2391660749   4 days ago
myapp:v2.1.6   docker-registry.default.svc:5000/mynamespace/myapp@sha256:e46b32e34f7b3f5ba4bdbf26c66205ced54d42fd5b978f12efe2d858085d3767   25 hours ago
myapp:v2.1.7   docker-registry.default.svc:5000/mynamespace/myapp@sha256:cb84f1c02ca3a8755a2ab582934603a703c72b8fdeee5773c20da12045a3385e   22 hours ago
myapp:v2.1.8   docker-registry.default.svc:5000/mynamespace/myapp@sha256:5fa5f7f79a68374a79c957be53d69e2743450f41335696bb95d214356867dcaf   6 hours ago
myapp:latest   docker-registry.default.svc:5000/mynamespace/myapp@sha256:615f3a96bfae7cd0295bab4520d2a26f206f502999b49f1313320bb538e3e629   5 hours ago
```

### Conclusion

This command could easily be added to a pipeline script or run on a `CronJob` schedule. If you are deploying at a high frequency, you may want to consider ditching the semantic versioning strategy all together, as releases act as a more continuous stream of builds. In that case, you could possibly label your container image with the commit hash from your repository or something similar. More on that topic later. However, if you are still in favor of semantic versioning tags, this could serve as a good solution for pruning.