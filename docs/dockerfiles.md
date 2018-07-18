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

## Use Multi-stage Builds

Multi-stage builds are a feature requiring Docker 17.05 or higher on the daemon and client. 

Multistage builds are useful to anyone who has struggled to optimize Dockerfiles while keeping them effortless to read and maintain.

With multi-stage builds, you use multiple `FROM` statements in your Dockerfile.
Each FROM instruction can use a different base, and each of them begins a new stage of the build.

You can selectively copy artifacts from one stage to another, leaving behind everything you don’t want in the final image.
To show how this works, Let’s adapt the Dockerfile from the previous section to use multi-stage builds.

Example:

``` docker
FROM golang:1.7.3
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]  
```

You need one single Dockerfile.
You don’t need a separate build script, either. `docker build .` is all what you need.

``` docker
docker build -t alexellis2/href-counter:latest .
```

The end result is the same tiny production image as before, with a significant reduction in complexity. You don’t need to create any intermediate images and you don’t need to extract any artifacts to your local system at all.

How does it work?
The second `FROM`instruction starts a new build stage with the `alpine:latest` image as its base.
The `COPY --from=0` line copies just the built artifact from the previous stage into this new stage. 

The Go SDK and any intermediate artifacts are left behind, and not saved in the final image.

### Name Your Build Stages

By default, the stages are not named, and you refer to them by their integer number, starting with 0 for the first `FROM` instruction.

However, you can name your stages, by adding an `as <NAME>` to the `FROM` instruction.

This example improves the previous one by naming the stages and using the name in the `COPY` instruction.

This means that even if the instructions in your Dockerfile are re-ordered later, the `COPY` does not break.

Example:

``` docker
FROM golang:1.7.3 as builder
WORKDIR /go/src/github.com/alexellis/href-counter/
RUN go get -d -v golang.org/x/net/html  
COPY app.go    .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/alexellis/href-counter/app .
CMD ["./app"]
```

For more info, please consult the [official docs](https://docs.docker.com/develop/develop-images/multistage-build/#name-your-build-stages).