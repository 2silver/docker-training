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

## Sort Multi-line Arguments

Whenever possible, ease later changes by sorting multi-line arguments alphanumerically.
This helps to avoid duplication of packages.

Adding a space before a backslash (`\`) helps as well.

Example:

``` docker
FROM alpine:3.8

RUN apk add --no-cache --virtual .build-deps \
    gcc \
    libc-dev \
    python3 \
    zlib-dev
```

## Use Pipes

Some `RUN` commands depend on the ability to pipe the output of one command into another, using the pipe character (|), as in the following example:

``` docker
RUN wget -O - https://some.site | wc -l > /number
```

Docker runs these commands using the `/bin/sh -c` interpreter, which only evaluates the exit code of the last operation in the pipe to determine success.

In the example above this build step succeeds and produces a new image so long as the `wc -l` command succeeds, even if the `wget` command fails.

If you want the command to fail due to an error at any stage in the pipe, prepend `set -o pipefail &&` to ensure that an unexpected error prevents the build from inadvertently succeeding.

 For example:

``` docker
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```


!!! note

    Not all shells support the `-o pipefail` option.
    In such cases (such as the `dash` shell, which is the default shell on Debian-based images),
    consider using the *exec* form of `RUN` to explicitly choose a shell that does support the `pipefail` option.
    
    For example:

    ``` docker
    RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
    ```

## Instructions

### USER

Do not run your stuff as root, be humble, use the `USER` instruction to specify the user.

This user will be used to run any subsequent `RUN`, `CMD` AND `ENDPOINT` instructions in your Dockerfile.

For running `ENTRYPOINT` scripts as non-root you can use for example:

- [su-exec](https://github.com/ncopa/su-exec)
- [gosu](https://github.com/tianon/gosu)


### WORKDIR
A convenient way to define the working directory, it will be used with subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD` instructions.

You can specify `WORKDIR` multiple times in a Dockerfile.

If the directory does not exists, Docker will create it for you.

### ADD Or COPY

[Dockerfile reference for the ADD instruction](https://docs.docker.com/v17.09/engine/reference/builder/#add)
[Dockerfile reference for the COPY instruction](https://docs.docker.com/v17.09/engine/reference/builder/#copy)

Although `ADD` and `COPY` are functionally similar, generally speaking, `COPY` is preferred.

That is because it’s more transparent than `ADD`.

`COPY` only supports the basic copying of local files into the container, while `ADD` has some features (like local-only tar extraction and remote URL support) that are not immediately obvious.

Consequently, the best use for `ADD` is local tar file auto-extraction into the image, as in `ADD rootfs.tar.xz /`.

If you have multiple `Dockerfile steps` that use different files from your context, `COPY` them individually, rather than all at once.

This will ensure that each step’s build cache is only invalidated (forcing the step to be re-run) if the specifically required files change.

For example:

``` docker
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

Results in fewer cache invalidations for the `RUN` step, than if you put the `COPY . /tmp/` before it.

Because image size matters, using `ADD` to fetch packages from remote URLs is **strongly discouraged**;
you should use `curl` or `wget` instead.

That way you can delete the files you no longer need after they’ve been extracted and you won’t have to add another layer in your image.

For example, you should avoid doing things like:

``` docker
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

Instead, do something like:

``` docker
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```

For other items (files, directories) that do not require `ADD`’s tar auto-extraction capability, you should always use `COPY`.

### CMD And ENTRYPOINT

`CMD` is the instruction to specify what component is to be run by your image with arguments in the following form:

``` docker
CMD [“executable”, “param1”, “param2”…].
```

You can override `CMD` when you’re starting up your container by specifying your command after the image name like this:

``` docker
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...].
```

You can only specify one `CMD` in a `Dockerfile` (OK, physically you can specify more than one, but only the last one will be used).

It is good practice to specify a `CMD` even if you are developing a generic container,
in this case an interactive shell is a good `CMD` entry.

Do `CMD ["python"] or CMD [“php”, “-a”]` to give your users something to work with.

What’s the deal with `ENTRYPOINT`?

When you specify an entry point, your image will work a bit differently.

You use `ENTRYPOINT` as the main executable of your image.
In this case whatever you specify in `CMD` will be added to `ENTRYPOINT` as parameters.

``` docker
ENTRYPOINT ["git"]
CMD ["--help"]
```

This way you can build Docker images that mimic the behavior of the main executable you specify in `ENTRYPOINT`.

### ONBUILD

[Dockerfile reference for the `ONBUILD` instruction](https://docs.docker.com/v17.09/engine/reference/builder/#onbuild)

You can specify instructions with `ONBUILD` that will be executed when your image is used as the base image of another `Dockerfile`.

An `ONBUILD` command executes after the current Dockerfile build completes.

`ONBUILD` works in any child image derived `FROM` the current image.
Think of the `ONBUILD` command as an instruction the parent `Dockerfile` gives to the child `Dockerfile`.

A Docker build runs `ONBUILD` commands before any command in a child `Dockerfile`.

`ONBUILD` is for images that are going to be built `FROM` a given image.

For example, you would use `ONBUILD` for a language stack image that builds arbitrary user software written in that language within the `Dockerfile`, as you can see in [Ruby’s `ONBUILD` variants](https://github.com/docker-library/ruby/blob/master/2.4/jessie/onbuild/Dockerfile).

Images built from `ONBUILD` should get a separate tag, for example: `ruby:1.9-onbuild` or `ruby:2.0-onbuild`.

**Be careful** when putting `ADD` or `COPY` in `ONBUILD`.

The “onbuild” image will fail if the new build’s context is missing the resource being added.
Adding a separate tag, as recommended above, will help mitigate this by allowing the `Dockerfile` author to make a choice.

### Label

[Understanding object labels](https://docs.docker.com/config/labels-custom-metadata/)

You can add labels to your image to help organize images by project, record licensing information, to aid in automation, or for other reasons.
For each label, add a line beginning with `LABEL` and with one or more key-value pairs.
 
The following examples show the different acceptable formats. Explanatory comments are included inline.

!!! note
    Strings with spaces must be quoted **o**r the spaces must be escaped.
    Inner quote characters (`"`), must also be escaped.

``` docker
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""
```

An image can have more than one label. Prior to Docker 1.10, it was recommended to combine all labels into a single `LABEL` instruction, to prevent extra layers from being created.

This is no longer necessary, but combining labels is still supported.

``` docker
# Set multiple labels on one line
LABEL com.example.version="0.0.1-beta" com.example.release-date="2015-02-12"
```

The above can also be written as:

``` docker
# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```

See [Understanding object labels](https://docs.docker.com/config/labels-custom-metadata/) for guidelines about acceptable label keys and values.

For information about querying labels, refer to the items related to filtering in [Managing labels on objects](https://docs.docker.com/config/labels-custom-metadata/#managing-labels-on-objects).
See also [`LABEL`](https://docs.docker.com/engine/reference/builder/#label) in the `Dockerfile` reference.