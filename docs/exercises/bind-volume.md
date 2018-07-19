# Use Bind Mounts

!!! abstract

    How to use bind mounts to share files.

Bind mounts have been around since the early days of Docker.
Bind mounts have limited functionality compared to volumes.
When you use a bind mount, a file or directory on the host machine is mounted into a container.
The file or directory is referenced by its full or relative path on the host machine.

 By contrast, when you use a volume, a new directory is created within Docker’s storage directory on the host machine, and Docker manages that directory’s contents.

The file or directory does not need to exist on the Docker host already.
It is created on demand if it does not yet exist. Bind mounts are very performant, but they rely on the host machine’s filesystem having a specific directory structure available.

 If you are developing new Docker applications, consider using [named volumes](https://docs.docker.com/storage/volumes/) instead.

You can not use Docker `CLI` commands to directly manage bind mounts.

## Building

Change into the various directories and ``build`` the image.

We will use *share-data* for this example.

First we ``clone`` the repository.

To do so, please open a terminal or use [GitHub Desktop](https://desktop.github.com/).

If you use the terminal do

``` console
git clone https://github.com/wundertax/docker-training
```

If you use GitHub Desktop, please read the [user guide](https://help.github.com/desktop/guides/).

Change into the directory *share-data*

``` console
cd share-data
```

Build the image and tag it with a name

``` console
docker build -t logger .
```

Or with a name of your choice.

## Running The Image
```console
docker run -v $(pwd)/logs:/srv/logs -p 2015:2015 dtl
```

Now open your browser and browse to: http://0.0.0.0:2015/

Hit the URL a couple of times !

After that check the content of the ``logs`` directory !

