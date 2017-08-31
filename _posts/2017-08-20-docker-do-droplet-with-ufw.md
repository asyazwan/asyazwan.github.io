---
title: "Docker DO droplet with UFW"
categories: article
tags: dev docker
---

Recently I setup a docker host on DigitalOcean using the [docker droplet](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-docker-application) and encountered a problem where outbound network connection is not possible with UFW turned on.

UFW is [Uncomplicated Firewall](https://help.ubuntu.com/community/UFW) which you can read more about in the linked wiki. Basically it makes it easier to manage `iptables` rules. **You should not have an instance in the cloud without any kind of protection** so it makes sense to enable UFW which is disabled by default.

As of this writing I'm using **Docker 17.06.0-ce** and **Ubuntu 16.04.2 LTS**.
{: .notice--info}

So this is how to get it working from scratch:

Create droplet -> One-click apps -> Docker xxx on 16.04

Get into your new droplet:

```
$ ssh -i <mykey> root@<droplet-ip>
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-83-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.


-------------------------------------------------------------------------------
Thank you for using DigitalOcean's Docker Application.

Docker has been preinstalled and configured per Docker's Recommendations.

"ufw" has not been enabled, however it has been configured. To enable it,
run "ufw enable".

Let's Encrypt has been pre-installed for you. If you have a domain name, and
you will be using it with this 1-Click app, please see: http://do.co/le-apache

'docker-compose' is installed in Docker container and is executed via
 /usr/local/bin/docker-compose. On your first run, the container
will be downloaded. To upgrade docker-compose version, edit
/usr/local/bin/docker-compose and change the version string.

-------------------------------------------------------------------------------

You can learn more about using this image here: http://do.co/docker

-------------------------------------------------------------------------------
To delete this message of the day: rm -rf /etc/update-motd.d/99-one-click

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

```

Note what it says: **ufw has not been enabled**. But it's actually enabled by default:

```bash
root@docker-test:~# ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         LIMIT       Anywhere
2375/tcp                   ALLOW       Anywhere
2376/tcp                   ALLOW       Anywhere
22 (v6)                    LIMIT       Anywhere (v6)
2375/tcp (v6)              ALLOW       Anywhere (v6)
2376/tcp (v6)              ALLOW       Anywhere (v6)
```

The two unexpected ports are for orchestration with `docker-machine` / swarm so that the docker clients can communicate with each other. Feel free to remove them if you won't be using them. Test container connectivity:

```bash
root@docker-test:~# docker run --rm -it busybox ping 8.8.8.8 -c 4
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
9e87eff13613: Pull complete
Digest: sha256:2605a2c4875ce5eb27a9f7403263190cd1af31e48a2044d400320548356251c4
Status: Downloaded newer image for busybox:latest
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=49 time=2.441 ms
64 bytes from 8.8.8.8: seq=1 ttl=49 time=2.412 ms
64 bytes from 8.8.8.8: seq=2 ttl=49 time=2.606 ms
64 bytes from 8.8.8.8: seq=3 ttl=49 time=2.378 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 2.378/2.459/2.606 ms
```

And inbound connectivity:

```bash
root@docker-test:~# docker run --rm -d -p 30000:80 nginx:alpine
Unable to find image 'nginx:alpine' locally
alpine: Pulling from library/nginx
019300c8a437: Pull complete
a3fe4a77433d: Pull complete
a5443900e7f5: Pull complete
0ae275323c0f: Pull complete
Digest: sha256:24a27241f0450b465f9e9deb30628c524aa81a1aa6936daa41ef7c4345515272
Status: Downloaded newer image for nginx:alpine
71f90fc2d8e013283d71e1dbff9ea65ea46d9d09a66e171a595b9b63e5c4103d
```

And from your browser or local terminal:
```bash
syaz@mbp:~$ curl -I http://<droplet-IP>:30000/
HTTP/1.1 200 OK
Server: nginx/1.13.3
Date: Thu, 31 Aug 2017 13:11:30 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 11 Jul 2017 18:57:58 GMT
Connection: keep-alive
ETag: "59651fb6-264"
Accept-Ranges: bytes
```

It works! **But wait a minute!** As [this guy](http://blog.viktorpetersson.com/post/101707677489/the-dangers-of-ufw-docker) discovered, docker changes your iptables to accommodate your container's exposed ports. This is actually the default behavior according to the [documentation](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communicating-to-the-outside-world):

> Docker will never make changes to your system iptables rules if you set `--iptables=false` when the daemon starts. **Otherwise the Docker server will append forwarding rules to the DOCKER filter chain.**
>
> Docker will flush any pre-existing rules from the `DOCKER` and `DOCKER-ISOLATION` filter chains, if they exist. For this reason, any rules needed to further restrict access to containers need to be added after Docker has started.

If it's not clear enough, it means _your iptables you set via_ `ufw` _can be misleading because docker will add entries to it to allow connections from anywhere_. We did not allow port 30000 -- let's quickly test this:

```bash
root@docker-test:~# ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         LIMIT       Anywhere
22 (v6)                    LIMIT       Anywhere (v6)

root@docker-test:~# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
71f90fc2d8e0        nginx:alpine        "nginx -g 'daemon ..."   28 seconds ago      Up 26 seconds       0.0.0.0:30000->80/tcp   awesome_chandrasekhar

root@docker-test:~# ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         LIMIT       Anywhere
22 (v6)                    LIMIT       Anywhere (v6)

root@docker-test:~# iptables -L -t nat -nv
Chain PREROUTING (policy ACCEPT 3 packets, 156 bytes)
 pkts bytes target     prot opt in     out     source               destination
  115  7260 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 3 packets, 156 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    84 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
    0     0 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:30000 to:172.17.0.2:80
```

As you can see in `DOCKER` chain, `ufw` is _bypassed_. If you are aware that running a container with exposed port will expose the port to the whole world, then it's fine. But it's highly unlikely that is the case. Even if you bind only loopback address to docker port argument, chances are you will forget to do it eventually. This is definitely not something I want so the next logical thing to do is to tell docker not to mess with iptables.

For Ubuntu prior to 16.04 you can achieve this by appending `--iptables=false` to `DOCKER_OPTS` in `/etc/default/docker`.
{: .notice--info}

The default `dockerd` daemon [configuration file](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file) is located in `/etc/docker/daemon.json`. It doesn't exist for us so we'll have to create it:

```javascript
{
  "iptables": false
}
```

```bash
root@docker-test:~# service docker restart
```

Now you can re-run our nginx test container and verify that you can no longer access the port publicly. You will timeout because `iptables` is no longer altered by `dockerd` so port 30000 is inaccessible from outside.

We are not done yet. After you reboot, your `iptables` will be in correct state and **your containers will have no outbound connectivity**.

```bash
root@docker-test:~# reboot now
Connection to X.X.X.X closed by remote host.
Connection to X.X.X.X closed.
```

Reconnect and redo our test ping we did at the very start:

```bash
root@docker-test:~# docker run --rm -it busybox ping 8.8.8.8 -c 4
PING 8.8.8.8 (8.8.8.8): 56 data bytes
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss

root@docker-test:~# iptables -L -t nat -nv
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

No outbound connectivity, and a very clean `iptables`. We could manually add `iptables` rule for docker chain every reboot, or we can automate it by appending to `/etc/ufw/after.rules`:

```
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -o docker0 -s 172.17.0.0/16 -j MASQUERADE
COMMIT
```

Reboot and you will now be able to do outbound requests from within your container.
