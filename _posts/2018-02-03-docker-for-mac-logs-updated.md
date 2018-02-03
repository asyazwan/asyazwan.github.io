---
title: "Docker for Mac logs (updated)"
categories: article
tags: dev docker xhyve linuxkit
---

Some time last month new docker for mac [shipped with LinuxKit](https://docs.docker.com/docker-for-mac/release-notes/#docker-community-edition-17120-ce-mac46-2018-01-09-stable):


```bash
$ screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
linuxkit-025000000001:~# uname -a
Linux linuxkit-025000000001 4.9.75-linuxkit-aufs #1 SMP Tue Jan 9 10:58:17 UTC 2018 x86_64 Linux
```

But eventually you'll discover `/dev/log` is missing. Apparently `syslog` is not running too.

So [one way](https://github.com/docker/for-mac/issues/2465#issuecomment-359111018) to get the log working again is to run syslog:

```
linuxkit-025000000001:~# syslogd
```

Which will create `/dev/log` socket and write to `/var/log/messages` by default and you can continue tailing that log file like before.

Happy logging!
