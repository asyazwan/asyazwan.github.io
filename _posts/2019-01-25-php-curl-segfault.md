---
title: “PHP cURL segfault”
categories: article
tags: dev curl php
---

Around August 2018, my colleague started to have breaking tests involving curl functions. Few days later, the problem infected our CI. Upon further debugging, we confirmed that it was related to segfaults with curl/libcurl 7.60.0. We didn’t pin our curl version so it got upgraded from 7.59 -> 7.60 during docker image build. Apparently, around June 2018 there were [quite](https://github.com/curl/curl/pull/2669) [a](https://github.com/curl/curl/issues/2688) [few](https://github.com/curl/curl/issues/2674) issues reported related to segfaulting and 7.60.0.

This is the minimal test that could reproduce the problem:

```php
<?php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, "https://www.google.com");
curl_exec($ch);
curl_close($ch);
```

Which would get you some variation of this coredump:
```
Thread 1 "php" received signal SIGSEGV, Segmentation fault.
0x00007ffff540dfc3 in ?? () from /lib/libssl.so.43
(gdb) bt
#0  0x00007ffff540dfc3 in ?? () from /lib/libssl.so.43
#1  0x00007ffff6545eac in ssl_create_cipher_list () from /lib/libssl.so.1.0.0
#2  0x00007ffff653faf4 in SSL_CTX_new () from /lib/libssl.so.1.0.0
#3  0x00007ffff58a495a in ?? () from /usr/lib/libcurl.so.4
#4  0x00007ffff58a59b9 in ?? () from /usr/lib/libcurl.so.4
#5  0x00007ffff58a6553 in ?? () from /usr/lib/libcurl.so.4
#6  0x00007ffff5867b3c in ?? () from /usr/lib/libcurl.so.4
#7  0x00007ffff5868e7f in ?? () from /usr/lib/libcurl.so.4
#8  0x00007ffff5872030 in ?? () from /usr/lib/libcurl.so.4
#9  0x00007ffff5882810 in ?? () from /usr/lib/libcurl.so.4
#10 0x00007ffff5883470 in curl_multi_perform () from /usr/lib/libcurl.so.4
#11 0x00007ffff5acd1ba in ?? () from /usr/lib/php7/modules/curl.so
```

Ok what the heck, let’s just downgrade, right? That’s what we did via:

```
RUN echo "http://dl-cdn.alpinelinux.org/alpine/v3.4/main" >> /etc/apk/repositories \
    && apk update \
    && apk add --no-cache \
                ...
                curl=7.59.0-r0 \
                libcurl=7.59.0-r0 \
```

That solved it… until a month later when build failed again because the pinned version has gone missing from alpine 3.4 repository. I’ve experienced this a few times, and I’m not [alone](https://medium.com/@stschindler/the-problem-with-docker-and-alpines-package-pinning-18346593e891).

Our solution then is to just upgrade to alpine 3.8 with fixed version of curl and unpinned them. As of this moment of writing, we have migrated away from Alpine, which I will cover in future post.
