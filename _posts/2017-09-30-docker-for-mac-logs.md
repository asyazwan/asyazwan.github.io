---
title: "Docker for Mac logs"
categories: article
tags: dev docker xhyve
---

When docker [ditched](https://blog.docker.com/2016/05/docker-unikernels-open-source/) Alpine for [xhyve](https://github.com/mist64/xhyve), I didn't know how to get into docker itself to eg. see logs.

> The xhyve hypervisor is a port of bhyve to OS X. It is built on top of Hypervisor.framework in OS X 10.10 Yosemite and higher, runs entirely in userspace, and has no other dependencies.

So it's a lightweight hypervisor that runs Linux with Docker Engine inside. If you can't find your logs you need via `docker logs` (what's sent to `STDOUT`), chances are it's in the docker daemon logs (sent for example via syslog facilities).

This is the magic:
```
$ screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

I will spare the explanation of what `screen` is but it comes bundled with OSX. The trick is just knowing where the tty to attach to is at.

Sometimes you might have been disconnected or you exit the terminal using `^d`.
This actually *detach* from the terminal.
What this means is that the next time you run the command again, you will see weird artifacts like crazy text indentation and gibberish characters.
Restarting docker will of course fix this as the Linux effectively reboots.

Or you can be civilized and just reattach (assuming you only have one session running):
```
$ screen -R
```

If you have multiple sessions, find your docker session:
```
$ screen -ls
There is a screen on:
	48617.ttys019.syazwan	(Detached)
1 Socket in /var/folders/_4/gl_zvn7x4y7g8m66vwwvz4nr0000gn/T/.screen.
```
Then reattach with the correct `<pid>.<tty>.<host>`:
```
$ screen -r 48617.ttys019.syazwan
```

Happy logging!
