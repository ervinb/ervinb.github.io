---
layout: post
title: Portable build environments with Docker
tags: phantomjs docker linux
---

Docker is a great asset for creating ephemeral envrionments. This is especially useful in our case, where
a lot of build dependecies have to be pre-installed, before hitting the switch. We can create various
build environments, having vastly different packages and having a tidy host too.

PhantomJS is a great open source effort, but their hard work rarely gets a release in a timely manner. Even if the fix
you're looking for is merged, there's a very good chance that there is no binary available for download.

Compiling from source to the rescue!

Here's the "I'm-being-chased-by-a-wolfpack" version:

```sh
# optional build-arg for specifying which code revision
# will be checked out (2.1.1 by default)
$ git clone ervinb/pd
$ docker build -t phantomjs --build-arg TAG=2.1.1 .
# reload your shotgun, twice
# (and get the shells from the factory)
$ docker run --rm --volume $(pwd)/trunk:/phantomjs-src/bin phantomjs-stage
$ ./trunk/phantomjs --version
> 2.1.1
```

This ran for 43 minutes, in a Vagrant box with 3 maxed out vCores and 4GB's of memory,
and the resulting image is almost 3GB in size.

## Setting up the stage

The preassumption is that you have Docker installed on your host,
and a basic knowledge of it along side -- to go with it. If that's not the case,
follow [this guide](https://docs.docker.com/engine/installation/) for the installtion,
and explore their docs a bit for the rest.

Now, we can write the "recipe" for the build environment: a [Dockerfile](https://docs.docker.com/engine/reference/builder/).
This will define the steps needed for baking an image, capable of producing a PhantomJS
binary. It looks something like this:

```sh
FROM ubuntu:14.04
ARG DEPENDENCY_BUSTER=no
ARG REPO_BUSTER=no
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
RUN git fetch --all && \
    git reset --hard origin/master
RUN git checkout $TAG && \
    git submodule init && \
    git submodule update
CMD python build.py
```
We base our image on Ubuntu 14.04 with the [FROM](https://docs.docker.com/engine/reference/builder/#/from)
instruction. Then, we set some environment variables, which can be overriden (more on this later),
install the dependecies and compile the checked out code with `python build.py`.

The structure and the composition of the commands are fairly important. They're
constructed so that the cache can be utilized for parts which don't change often.
Each line creates it's own layer, which can be then reused in subsequent runs,
when no changes are detected. In the case of [RUN](https://docs.docker.com/engine/reference/builder/#/run),
this happens only when the command string changes. In other words, executing
`docker build .` twice, with a Dockerfile which has `apt-get update`, will run the
update the first time, but it will reuse the cached layer on the second run.

Armed with this knowledge, we can understand the `echo $DEPENDENCY_BUSTER` and
`echo $REPO_BUSTER` instructions. They serve to "bust" the cache, which makes sure that
`apt-get install` isn't reused or the repository is fresh.
Dependencies are a subject to change, so it's nice to have an option to reinstall the on-demand.
It goes the same for large, long running repository fetches. When we provide a random
value these variables as a Docker `build-arg`, the command strings will change, thus
busting the cache.
If you're using `bash`, you can use the built-in `$RANDOM` variable to pass a
random number like so:

```
$ docker build -t phantomjs-stage --build-arg DEPENDENCY_BUSTER=$RANDOM .
```

Anything below `RUN echo $DEPENDENCY_BUSTER` will not use the layer cache. When
passing a value for `$DEPENDENCY_BUSTER` we get a fresh set of build dependencies,
otherwise we use the ones stored in the cached layer.

This was an example to show you how you can thoughtfully re-use layers. In a nutshell,
to have effective Docker layer caching, moving parts go to the bottom, static files and
dependencies go on top. Plus, you have an option to set up markers at specific stages,
for smart layer re-use.

## A phantom and a whale

Once we have baked the image, the last step to is to do something useful with it. Also,
you should check if you left the oven on, like, right now.

In our Dockerfile, we have a [CMD](https://docs.docker.com/engine/reference/builder/#/cmd)
instruction call, meant to execute the compilation process we the image is started.
We use it in the `shell` form. Two additional forms are available, `exec`and `param`,
about which you can have an interesting read by clicking the link above.

```sh
# the 'run' command will mount a new dir (not pre-created) and pooplate it after build
docker run --rm --volume $(pwd)/trunk:/phantomjs-src/bin phantomjs-stage
```

To get our hands on the PhantomJS binary, we use a Docker volume. Make sure that `trunk`
isn't created in the current directory, from where you're running it. # more on this

Once the lengthy compilaton process is done, you'll have a nice `phantomjs` executable waiting
for you in the `./trunk`.

```sh
$ ./trunk/phantomjs --version
> 2.1.1
```

## Curtains in

We explored a way to make isolated, disposable build environments and our toe touched the
waters of Docker layer caching.

It was aimed at PhantomJS, but as you can image, this can be used for other purposes as well.
Complex shell scripts can be used instead of a single `CMD` command, which give you and unlimited
number of doors to open.

Have a blast until next time!
