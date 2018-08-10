---
title: "Dockerizing vim"
categories: article
tags: dev vim docker
---

I have dockerized my vim so that I can pull the image and get my familiar vim setup within minutes from any machine (with docker).

I was heavily inspired by [this](https://github.com/JAremko/alpine-vim) dockerized vim project. Check it out if you think that's enough for you. Or you can use the [images](https://hub.docker.com/r/asyazwan/vim/tags/) I have prepared:

- `asyazwan/vim:tiny`   - 24MB compressed / 61MB uncompressed
- `asyazwan/vim:small`  - 24MB compressed / 61MB uncompressed
- `asyazwan/vim:normal` - 25MB compressed / 62MB uncompressed
- `asyazwan/vim:big`    - 25MB compressed / 62MB uncompressed
- `asyazwan/vim:huge`   - 25MB compressed / 62MB uncompressed

Check out [this](http://www.drchip.org/astronaut/vim/vimfeat.html) table of comparison for the differences between features
{: .notice--info}

But if you want to make your own image, here's how I did it.

I have pushed the Dockerfiles used into [this](https://github.com/asyazwan/docker-vim) repository. Check out the main Dockerfile:

```Dockerfile
FROM alpine:3.8 as builder

WORKDIR /tmp

RUN apk add --no-cache \
  build-base \
  ctags \
  git \
  libx11-dev \
  libxpm-dev \
  libxt-dev \
  make \
  ncurses-dev \
  python3 \
  python3-dev \
  perl-dev \
  ruby-dev

RUN git clone https://github.com/vim/vim && cd vim \
  && ./configure \
  --disable-gui \
  --disable-netbeans \
  --enable-multibyte \
  --enable-perlinterp \
  --enable-rubyinterp \
  --enable-python3interp \
  --with-features=huge \
  --with-python3-config-dir=/usr/lib/python3.6/config-3.6m-x86_64-linux-gnu/ \
  --with-compiledby=asyazwan@gmail.com \
  && make install


FROM alpine:3.8

COPY --from=builder /usr/local/bin /usr/local/bin
COPY --from=builder /usr/local/share/vim  /usr/local/share/vim
RUN apk add --update --no-cache \
  libice \
  libsm \
  libx11 \
  libxt \
  ncurses \
  python3 \
  ruby \
  perl \
  php7

RUN apk add --update --no-cache \
  git \
  bash \
  fish \
  ctags \
  fzf \
  the_silver_searcher

RUN git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install


ENTRYPOINT ["vim"]
```

I am utilizing [multi-stage](https://docs.docker.com/develop/develop-images/multistage-build/) build for this. It's to reduce the size of final image since it won't have all the packages required for building vim like `gcc` and various libraries and their headers. Also, if you take out the last 2 `RUN` statements and also remove `ruby`, `perl` and `php7` packages, your vim image (uncompressed) will only be around 30MB!

Remove `--enable-<language>interp` from the build stage and the language package from the last stage if you don't want them. Chances are you won't need them if you don't already know what they are for.

The second-last `RUN` will install additional stuff that I need. `the_silver_searcher` is for using `:Ag` in vim and `fzf` is for lots of useful commands that I use many times a day.
The last `RUN` will clone `fzf` vim plugin and run the install script.

Use `docker build -t vim-base .` to build and you will have a pretty capable vim. Run `docker run --rm vim-base "--version"` to see what features have been compiled with vim.

Now that I have base vim, it's time to bake in some plugins. I've been using the same plugins for some time now and I don't think they're going to change anytime soon, so I decided to put the plugins into my final vim image. Check out `Dockerfile.plugins`:

```Dockerfile
FROM asyazwan/vim

ARG USERNAME=syazwan
ARG GROUPNAME=syazwan
ARG WORKSPACE=/home/syazwan
ARG UID=1000
ARG GID=1000
ARG SHELL=/bin/sh

RUN apk add --no-cache sudo \
  && mkdir -p "${WORKSPACE}" \
  && echo "${USERNAME}:x:${UID}:${GID}:${USERNAME},,,:${WORKSPACE}:${SHELL}" \
  >> /etc/passwd \
  && echo "${USERNAME}::17032:0:99999:7:::" \
  >> /etc/shadow \
  && echo "${USERNAME} ALL=(ALL) NOPASSWD: ALL" \
  > "/etc/sudoers.d/${USERNAME}" \
  && chmod 0440 "/etc/sudoers.d/${USERNAME}" \
  && echo "${GROUPNAME}:x:${GID}:${USERNAME}" >> /etc/group \
  && chown "${UID}":"${GID}" "${WORKSPACE}"

WORKDIR $WORKSPACE

USER $USERNAME

COPY plug.vim ${WORKSPACE}/.vim/autoload/plug.vim
COPY plugged ${WORKSPACE}/.vim/plugged/
COPY vimrc ${WORKSPACE}/.vimrc
COPY fzf-default.sh ${WORKSPACE}/.vim/fzf-default
RUN sudo mv /root/.fzf ${WORKSPACE}/.vim/

RUN sudo chown -R "${UID}":"${GID}" .vimrc .vim/
RUN ~/.vim/.fzf/install
RUN mkdir -p .vim/backups .vim/undo
ENV FZF_DEFAULT_COMMAND ~/.vim/fzf-default

ENTRYPOINT ["/usr/bin/fish"]
```

Let's break down what is going on here. From the base vim image, I create user account the same as I have on the current machine, which can be overriden via `--build-arg`. This is to avoid possible file permission/ownership issues. Then I copy my plugin manager [Plug](https://github.com/junegunn/vim-plug), the plugins, vimrc file, and  [fzf](https://github.com/junegunn/fzf) files. Entrypoint is set to my current favorite terminal.

To build:
```bash
docker build -f Dockerfile.plugins -t vim \
  --build-arg USERNAME=$(id -un) \
  --build-arg GROUPNAME=$(id -gn) \
  --build-arg UID=$(id -u) \
  --build-arg GID=$(id -g) \
  --build-arg WORKSPACE=$$HOME \
  --build-arg SHELL=/usr/bin/fish .
```

Then to run:
```bash
docker run --rm -it --hostname vim -v "$PWD":$HOME/dev/ vim
```

Which will bring you into fish terminal and `$HOME/dev/` workspace. Invoke vim with `vim .`.

That's the idea, basically. From any machine with docker I can just clone the repo and build `Dockerfile.plugins` to get my familiar development environment.

Please share if you have suggestions on how to improve it.
