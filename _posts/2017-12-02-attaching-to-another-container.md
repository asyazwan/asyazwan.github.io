---
title: "Attaching to another container"
categories: article
tags: dev docker nsenter
---


Sometimes you have to debug a running container and you cannot modify the container (without much hassle). For example, you're running a container that you don't want to modify, probably because everybody in your office is using the image and you'd want to run the exact same image. Or it could be that the container is highly dependent on by other containers (service containers) and bringing it down to add debug tools will mean restarting the whole stack, which could be very trivial or very involved. Or maybe you started the container with `--rm` or `-t` options which will make `docker attach` tricky or impossible. Or you're running a redis container and would like to `strace` why it seems stuck. Or you're running a python container and would like to use `ipython` inside the container which is not available so you'd have to install, commit and rebuild the image (or ephemerally install on the running container). Or you have a Go standalone binary container and just want to poke around inside.

There could be a lot of reasons. Use docker enough and eventually you'll bump into situation where `docker exec` isn't sufficient. This is just a cool trick to add to your docker swiss army knife collection.

The tool we will be using is called [nsenter](https://github.com/jpetazzo/nsenter) (**N**ame**S**pace**ENTER**). The original docker image uses `debian:jessie` which is over 300 MB in size.
If you don't mind using someone's binary, [there](https://github.com/walkerlee/docker-nsenter) is a standalone binary with statically compiled `musl` as the dependency and the whole thing is only 500 KB.

## Get into container

Suppose you need to use a container's network (eg. to access non-exposed services) or debug the process itself (PID: 1):

```bash
# enter by container name
$ docker run --rm -it --privileged --net=container:mycontainer --pid=container:mycontainer  jpetazzo/nsenter -t 1 -m -u -i -n sh
# enter by container id
$ docker run --rm -it --privileged --net=container:21ee7ab8ddd7 --pid=container:21ee7ab8ddd7  jpetazzo/nsenter -t 1 -m -u -i -n sh
```

The equivalent* with official ways `docker exec` / `docker run`:

```bash
$ docker exec -it --privileged --net=container:mycontainer --pid=container:mycontainer bash
# use minimal privelege instead of --privileged because strace only need these 2
$ docker run --rm -it --cap-add sys_admin --cap-add sys_ptrace --net=container:mycontainer --pid=container:mycontainer myimage/strace strace
```

<div class="notice--info">
*It's not technically the same. As stated <a href="https://github.com/jpetazzo/nsenter#looking-to-start-a-shell-inside-a-docker-container">here</a>:

<blockquote>
There are differences between nsenter and docker exec; namely, nsenter doesn't enter the cgroups, and therefore evades resource limitations. The potential benefit of this would be debugging and external audit, but for remote access, docker exec is the current recommended approach.
</blockquote>
</div>

## Get into docker host / Docker for Mac

Suppose you want to see docker's syslog or debug docker host itself:

```bash
$ docker run --rm -it --privileged --pid=host --net=host jpetazzo/nsenter -t 1 -m -u -i -n sh
```

