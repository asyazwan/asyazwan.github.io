---
title: "Running composer install with docker"
categories: article
tags: dev php docker
---

If you run everything in a container for development like I do, chances are your host PHP is not configured to support multiple PHP projects due to potential version differences. To provision the projects you can setup composer in each container or you can run a dedicated composer container to provision your other containers. This post demonstrates the latter.

# Choose an image

Composer has several images available over at [docker hub](https://hub.docker.com/r/composer/composer/) that you can pull.
Personally I'd choose Alpine base for anything if it's available as the size saving can be in the hundreds of MBs per image.
In my case, the difference is over 300MB:

```
│└─5afb0951f2a4 Virtual Size: 635.7 MB Tags: composer/composer:latest
│└─6d48ee9405f5 Virtual Size: 317.6 MB Tags: composer/composer:alpine
```

Then you can do:

`docker run --rm -v $PWD:/app composer/composer:alpine install`

Composer is already set as the entrypoint so all you need to pass is the command. In the above it's `install`. The key here is mounting your current directory inside the container so that your `vendor/` folder gets created and saved. You can also make an alias so `composer` will expand to the full docker command.

# Problems and solutions

## Your requirements could not be resolved to an installable set of packages.

This will most likely happen everytime because of missing platform requirements in your composer image. The solution is to append `--ignore-platform-reqs`:

`docker run --rm -v $PWD:/app composer/composer:alpine install --ignore-platform-reqs`

Another option is to define `platform` in your config to mirror your target platform settings. It can be tedious the more specific you get, though. See [example](https://getcomposer.org/doc/06-config.md#platform) syntax from the docs.
