---
layout: post
title: Building PhantomJS with Docker
tags: phantomjs docker linux
---

PhantomJS is a great open source effort, but their hard work rarely gets a release in a timely manner. Even if the fix
you're looking for is merged, there's a very good chance that there is no binary available for download.

Compiling from source to the rescue!

Docker is a great asset for creating ephemeral envrionments. This is especially useful in our case, where
a lot of build dependecies have to be pre-installed, before hitting the switch. We can create various
build environments, having vastly different packages and having a tidy host too.

Here's the "I'm-being-chased-by-a-wolfpack" version:

```sh
# optional build-arg for specifying which code revision
# will be checked out (2.1.1 by default)
$ docker build -t phantomjs --build-arg TAG=2.1.1 .
# reload your shotgun, twice
# (and get the shells from the factory)
$ docker run --volume phantom-out -td
$ ./phantom-out/phantomjs --version
```

This ran for 43 minutes, in a Vagrant box with 3 maxed out vCores and 4GB's of memory,
and the resulting image is almost 3GB in size.

## Step-by-step

The preassumption is that you have Docker on your host. If that's not the case,
follow [this guide](https://docs.docker.com/engine/installation/).

Now, we can write the "recipe" for the build environment: a [Dockerfile](https://docs.docker.com/engine/reference/builder/).
This will define the steps needed for baking an image, capable of producing a PhantomJS
binary. It looks something like this:

```sh
FROM ubuntu:14.04
ARG CACHE_BUSTER=no
ARG REPO_URL=git://github.com/ariya/phantomjs.git
ARG SRC_DIR=phantomjs-src
ARG TAG=2.1.1
RUN echo $CACHE_BUSTER > /dev/null
RUN apt-get update -qq &&              \
    apt-get install -y build-essential \
    g++ flex bison gperf ruby perl     \
    libsqlite3-dev libfontconfig1-dev  \
    libicu-dev libfreetype6 libssl-dev \
    libpng-dev libjpeg-dev python      \
    libx11-dev libxext-dev git
RUN git clone $REPO_URL $SRC_DIR
WORKDIR $SRC_DIR
# script this
RUN git fetch --all && git reset --hard origin/master
RUN git checkout $TAG && git submodule init && \
    git submodule update && python build.py
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

Armed with this knowledge, we can understand the `echo $CACHE_BUSTER` command
before setting up the repository. It "busts" the cache, which makes sure that
`apt-get install` isn't reused. Dependencies are a subject to change, so it's nice
to have an option to reinstall the on-demand. When we provide a random
value for `$CACHE_BUSTER` as a build-arg, the command string will change, thus
busting the cache.
If you're using `bash`, you can use the built-in `$RANDOM` variable to pass a
random number like so:

```
$ docker build --build-arg CACHE_BUSTER=$RANDOM .
```

This assures that the anything below `echo` will not use the layer cache. When
passing a value for `$CACHE_BUSTER` we get a fresh set of build dependencies,
otherwise we use the ones stored in the cached layer.

In a nutshell, to have effective Docker layer caching, moving parts go to the bottom,
static files and dependencies to on top.

 # extract data from container
 # the run command will mount a new dir (not pre-created) and pooplate it after build
 docker run --rm -v $(pwd)/trunk:/phantomjs-src/ -ti 3f755ca42730

# icing the cake
# script which receives params and know how to handle tags
