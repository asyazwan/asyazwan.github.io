---
title: "docker-compose new naming scheme"
categories: article
tags: dev docker-compose docker
---

On 30 October 2018, docker-compose released version [1.23.0 (2018-10-30)](https://docs.docker.com/release-notes/docker-compose/#1230-2018-10-30) with a very notable change that would likely break many scripts out there, including mine:

> The default naming scheme for containers created by Compose in this version has changed from <project>_<service>_<index> to <project>_<service>_<index>_<slug>, where <slug> is a randomly-generated hexadecimal string.

The "warning" came only about 1 month prior, hidden in 1.23.0-rc1 (2018-09-26) [release note](https://github.com/docker/compose/releases/tag/1.23.0-rc1). So previously your `project_app_1` will now be `project_app_1_abc123`! Unsurprisingly this caused a lot of [breakage and rage](https://github.com/docker/compose/issues/6316). 

It was promptly reverted a month later in [1.23.2 (2018-11-28)](https://github.com/docker/compose/blob/master/CHANGELOG.md#1232-2018-11-28).

There's a lot of lesson to learn here from the perspectives of both maintainer and user of project, as evident by going through the comments in the [issue](https://github.com/docker/compose/issues/6316).
