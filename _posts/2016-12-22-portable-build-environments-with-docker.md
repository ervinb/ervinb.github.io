---
layout: post
title: Portable build environments with Docker
tags: phantomjs docker linux compiling
---

Chances are, when you want to compile something from source, you will need to
install an exotic dependency or two on your box beforehand. Do this enough times,
and your screams won't be able to penetrate the software bloat you built around yourself.

Docker's great isolation and flexibilty proved to be a great asset for building
custom build environments: sandboxes with vastly different pre-installed libraries,
specialized for the current job at hand.

In this post, we will go through the process of creating one such sandbox, specialized
for compiling a popular headless browser.

PhantomJS is a great open source effort, but their hard work rarely gets a release
in a timely manner. Even if the fix you're looking for is merged, there's a very
good chance that no binary is available for download. Compiling from source to the rescue!

## Run!

Here's the "*I'm being chased by a wolfpack!*" version:

```sh
# optional build-arg for specifying which code revision to compile (2.1.1 by default)
$ git clone https://github.com/ervinb/phantomjs-sandbox.git && cd phantomjs-sandbox
$ docker build -t phantomjs-sandbox-211 --build-arg TAG=2.1.1 .
# reload your shotgun
$ out_dir="$(pwd)/trunk-$(date +%s)"; docker run --rm --volume $out_dir:/phantomjs-src/bin phantomjs-sandbox
# reload again (and visit the factory for more shells)
$ ./trunk-1482688158/phantomjs --version
> 2.1.1
```

This ran for 43 minutes, in a Vagrant box with 3 maxed out vCores and 4GB's of memory.

If you're interested in the implementation details, do continue reading.

## Setting up the stage

The preassumption is that you have Docker installed on your host,
and some basic Docker knowledge to go with it. If that's not the case,
follow [this guide](https://docs.docker.com/engine/installation/) for the installtion,
and explore their docs a bit for the rest.

Now, we can write the "recipe" for the build environment: a
[Dockerfile](https://docs.docker.com/engine/reference/builder/).
This will define the steps needed for baking an image, capable of producing a PhantomJS
binary. It looks something like this:

```sh
FROM ubuntu:14.04
ARG DEPENDENCY_BUSTER
ARG REPO_BUSTER
ARG REPO_URL=git://github.com/ariya/phantomjs.git
ARG SRC_DIR=phantomjs-src
ARG TAG=2.1.1
RUN echo $DEPENDENCY_BUSTER > /dev/null
RUN apt-get update -qq &&              \
    apt-get install -y build-essential \
    g++ flex bison gperf ruby perl     \
    libsqlite3-dev libfontconfig1-dev  \
    libicu-dev libfreetype6 libssl-dev \
    libpng-dev libjpeg-dev python      \
    libx11-dev libxext-dev git
RUN echo $REPO_BUSTER > /dev/null
RUN git clone $REPO_URL $SRC_DIR
WORKDIR $SRC_DIR
RUN git fetch --all &&       \
    git reset --hard $TAG && \
    git submodule init &&    \
    git submodule update
CMD python build.py
```
We base our image on Ubuntu 14.04 with the [FROM](https://docs.docker.com/engine/reference/builder/#/from)
instruction. Then, we set some optional build arguments, which can be overriden (more on this later),
install the dependecies, and compile the checked out code with `python build.py`.

### Using the same stage curtain

The same way buying new curtains between acts isn't that profitable, rebuilding
Docker images from zero can be cumbersome as well.

The structure and the composition of the commands are fairly important. They're
constructed in such a way that the cache can be utilized for parts which don't change often.
Each line in `Dockerfile` creates its own [layer](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/),
which can be then reused in subsequent runs, when no changes are detected. In the case of [RUN](https://docs.docker.com/engine/reference/builder/#/run),
this happens only when the command string changes. In other words, executing
`docker build .` twice, with a `Dockerfile` which has `apt-get update`, will run the
update the first time, but it will reuse the cached layer on the second run.

Armed with this knowledge, we can understand the `echo $DEPENDENCY_BUSTER` and
`echo $REPO_BUSTER` instructions. They serve to "bust" the cache, which makes sure that
`apt-get install` isn't reused or that the repository is freshly picked.
Dependencies are a subject to change, so it's nice to have an option to reinstall them on-demand.
It goes the same for large, long running repository fetches. When we provide a random
value to these variables as a Docker build argument, the command strings will
change, thus busting the cache.
If you're using `bash`, you can use the built-in `$RANDOM` variable to pass a
random number like so:

```
$ docker build -t phantomjs-sandbox-211 --build-arg DEPENDENCY_BUSTER=$RANDOM .
```

Anything below `RUN echo $DEPENDENCY_BUSTER` will not use the layer cache, and will
run as if it was its first time. This was more of an example, than a real world  scenario,
but you get the point.

In a nutshell, to have effective Docker layer caching, moving parts go to the bottom,
static files and dependencies go on top. Plus, you can spice up the process with
cache checkpoints, for smart layer re-use.

## A phantom from a whale

Once we have baked the image, the last step to is to do something useful with it. Also,
you should check if you left the oven on, like, right now.

In our Dockerfile, we have a [CMD](https://docs.docker.com/engine/reference/builder/#/cmd)
instruction call, meant to execute the compilation process when the image is started.
We use it in the `shell` form. Two additional forms are available, `exec`and `param`,
about which you can have an interesting read by clicking the link above.

```sh
# --rm will remove the container after the run completes but the phantomjs executable
# will persist
$ out_dir="$(pwd)/trunk-$(date +%s)"; docker run --rm --volume $out_dir:/phantomjs-src/bin phantomjs-sandbox
```

To get our hands on the PhantomJS binary, we use a Docker [volume](https://docs.docker.com/engine/tutorials/dockervolumes/).
The command above will create a timestamped `trunk` in the current directory,
and map it to `/phantomjs-src/bin` inside the container. Once `python build.py` completes,
it will produce an artifact inside the `/phantomjs-src/bin` directory,
and you'll have a nice `phantomjs` executable waiting for you in the `./trunk` on the host.

```sh
$ ./trunk-1482688158/phantomjs --version
> 2.1.1
```

The image can be explored interactively with:

```sh
$ docker run --rm -ti --volume $(pwd)/trunk-$(date +%s):/phantomjs-src/bin phantomjs-sandbox /bin/bash
```

## Closing the curtains

We explored a way to make isolated, disposable build environments, and our toe touched the
waters of Docker layer caching as well.

The post was aimed at PhantomJS, but as you can imagine, this approach can be used
for other purposes as well. Instead of a single `CMD` command, you can use complex
shell scripts which give you great powers.

Have a blast until next time!
