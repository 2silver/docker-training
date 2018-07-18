# Dockerfile Best Practices

!!! abstract

    Best practices for writing Dockerfiles.

!!! quote

    Docker builds images automatically by reading the instructions from a Dockerfile -- a text file that contains all commands, in order, needed to build a given image.
    A Dockerfile adheres to a specific format and set of instructions which you can find at [Dockerfile reference](https://docs.docker.com/engine/reference/builder/).

## Cache

While building an image, Docker will step through the instructions mentioned in the Dockerfile, executing them in chronological order.
As each instruction is examined Docker will look for an existing image layer in its cache that it can reuse, rather than creating a new image layer.

### Check Build Order

Let's take a example with a Python application:

``` docker
FROM python:3.7.0-alpine3.8

COPY . /app
WORKDIR /app

RUN pip install -r requirements.txt

ENTRYPOINT ["python"]
CMD ["ap.py"]
```

It's actually an example I have seen several times online.

The problem is that the `COPY . /app` command will invalidate the cache as soon as any file in the current directory is updated.
Let's say you just change the *README* file and run `docker build` again.
Docker will have to re-install all the requirements because the RUN `pip` command is run after the `COPY` that invalidated the cache.

The requirements should only be re-installed if the *requirements.txt* file changes:

``` docker
FROM python:3.7.0-alpine3.8

WORKDIR /app

COPY requirements.txt /app/requirements.txt
RUN pip install -r requirements.txt

COPY . /app

ENTRYPOINT ["python"]
CMD ["ap.py"]
```

## Minimize layers 

Each instruction in the Dockerfile adds an extra layer to the docker image.
The number of instructions and layers should be kept to a minimum as this ultimately affects build performance and time.

In your Dockerfile, try to connect as many statements as possible with `&& \`

**Not that good:**

``` docker
From alpine:3.8

RUN apk --no-cache add python3
RUN apk --no-cache add bash
RUN apk --no-cache add tini
RUN apk --no-cache add su-exec
```

**Better:**

``` docker
RUN apk --no-cache add \
    python3 \
    bash \
    tini \
    su-exec
```

## Avoid Unnecessary Packages

To reduce complexity, dependencies, file sizes, and build times, avoid installing unnecessary packages.

Remove everything you don’t need when running your container, which includes:

- Package manager caches, `apt` (Debian) or `apk` (Alpine).
- All packages needed for build-time only ( compilers, kernel-headers etc. ).
- All packages of the default install which are not needed by your container at runtime.

Following this practice, your Dockerfile could look like this:

Example with Debian:

``` docker
FROM debian:9

RUN apt-get update && apt-get -y upgrade && \
    apt-get -y install $BUILD_PACKAGES && \
    do-your-stuff && \
    apt-get remove --purge -y $BUILD_PACKAGES && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

Example with Alpine:

``` docker
FROM alpine:3.8

RUN apk add --no-cache --virtual .build-deps \
    gcc \
    libc-dev \
    zlib-dev \
    libjpeg-turbo-dev \
    libpng-dev \
    libxml2-dev \
    libxslt-dev \
    pcre-dev \
    do-your-stuff && \
    apk del .build-deps
```