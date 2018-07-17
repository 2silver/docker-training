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