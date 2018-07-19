# Docker Compose

!!! abstract

    Use Docker Compose to run WordPress in an isolated environment built with Docker containers.

This quick-start guide demonstrates how to use Compose to set up and run WordPress.

Before starting, make sure you have [Compose installed](https://docs.docker.com/compose/install/).

## Setup

Change into your project directory:

``` console
cd compose-wordpress
```

Now, run ``docker-compose up -d`` from your project directory.

This runs docker-compose up in detached mode, pulls the needed Docker images, and starts the wordpress and database containers.

``` console
docker-compose up -d
Creating network "my_wordpress_default" with the default driver
Pulling db (mysql:5.7)...
5.7: Pulling from library/mysql
efd26ecc9548: Pull complete
a3ed95caeb02: Pull complete
...
Digest: sha256:34a0aca88e85f2efa5edff1cea77cf5d3147ad93545dbec99cfe705b03c520de
Status: Downloaded newer image for mysql:5.7
Pulling wordpress (wordpress:latest)...
latest: Pulling from library/wordpress
efd26ecc9548: Already exists
a3ed95caeb02: Pull complete
589a9d9a7c64: Pull complete
...
Digest: sha256:ed28506ae44d5def89075fd5c01456610cd6c64006addfe5210b8c675881aff6
Status: Downloaded newer image for wordpress:latest
Creating my_wordpress_db_1
Creating my_wordpress_wordpress_1
```

## Bring Up WordPress In A Web Browser

At this point, WordPress should be running on port 8000 of your Docker Host, and you can complete the “famous five-minute installation” as a WordPress administrator.

**Note:** The WordPress site is not immediately available on port 8000 because the containers are still being initialized and may take a couple of minutes before the first load.

If you are using Docker for Mac or Docker for Windows, you can use http://localhost as the IP address, and open ``http://localhost:8000`` in a web browser.

## Shutdown And Cleanup

The command ``docker-compose down`` removes the containers and default network, but preserves your WordPress database.

The ``command docker-compose down --volumes`` removes the containers, default network, and the WordPress database.
