---
title: Docker
description: Running Kimai inside docker
canonical: /documentation/docker.html
---

We offer Kimai Docker images, both for development and a docker-compose setup suitable for running in a production environment.
Any issues with the container rather than the application itself can be raised [here]({{ site.kimai_v2_docker }}/issues/new?template=docker.yml).

The **Docker Hub repo**, where you find the auto-building prod and dev containers for Kimai is: [https://hub.docker.com/r/kimai/kimai2](https://hub.docker.com/r/kimai/kimai2)

## Available tags

- `kimai/kimai2:apache`: the latest Kimai version bundled in an Apache container, needs a reverse proxy only
- `kimai/kimai2:fpm`: the latest Kimai version bundled in PHP-FPM Alpine container (smaller image, but [needs extra setup steps]({% link _documentation/docker/docker-compose.md %})) 
- `kimai/kimai2:latest`: same as `kimai/kimai2:fpm`
- `kimai/kimai2:dev`: development image based on Apache, only to be used locally with debug mode and more 

Each new release creates tag names containing the Kimai release number:

- `kimai/kimai2:apache-x.xx.x`: version specific release (Apache container)  
- `kimai/kimai2:fpm-x.xx.x`: version specific release (PHP-FPM container)

The following tags are deprecated and will not receive updates anymore, replace them:

- `kimai/kimai2:apache-prod`: replaced by `kimai/kimai2:apache` 
- `kimai/kimai2:apache-latest`: replaced by `kimai/kimai2:apache` 
- `kimai/kimai2:fpm-prod`: replaced by `kimai/kimai2:fpm`
- `kimai/kimai2:fpm-latest`: replaced by `kimai/kimai2:fpm` 
- `kimai/kimai2:prod`: replaced by `kimai/kimai2:fpm`
- `kimai/kimai2:apache-dev`: replaced by `kimai/kimai2:dev`  
- `kimai/kimai2:fpm-dev`: no replacement
- `kimai/kimai2:apache-x.xx.x-dev`: no replacement
- `kimai/kimai2:fpm-x.xx.x-dev`: no replacement 
- `kimai/kimai2:apache-x.xx.x-prod`: replaced by `kimai/kimai2:apache-x.xx.x` 
- `kimai/kimai2:fpm-x.xx.x-prod`: replaced by `kimai/kimai2:fpm-x.xx.x` 

## Environment variables / Runtime args

The Kimai image supports the following environment variables:

- `DATABASE_URL` - Default "mysql://kimai:kimai@127.0.0.1:3306/kimai?charset=utf8mb4&serverVersion=5.7.40"
- `APP_SECRET` - Default: "change_this_to_something_unique"
- `TRUSTED_PROXIES` - Default: "nginx,localhost,127.0.0.1" 
- `TRUSTED_HOSTS` - Default: "nginx,localhost,127.0.0.1"
- `MAILER_FROM` - Default: "kimai@example.com"
- `MAILER_URL` - Default: "null://localhost"
- `ADMINMAIL` - Email for the new `admin` user (will be created if not existing)
- `ADMINPASS` - Password for the new `admin` user (will be created if not existing)
- `memory_limit` - Default: "256M", the [maximum amount of memory](https://php.net/memory-limit) a script may consume

It is possible to set the user that FPM or Apache run as. If the user does not exist a new user called www-kimai is created and the server is then run under that user. Note these must be numbers, not names.

- `USER_ID=1000`
- `GROUP_ID=1000`

## Quick start

This will run the latest production build and make it accessible at <http://localhost:8001>.

{% alert danger %}This setup is NOT intended for production use as it is temporary and the data will disappear when the containers are removed (see point 4).{% endalert %}

1. Start a DB

    ```bash
    docker run --rm --name kimai-mysql-testing \
        -e MYSQL_DATABASE=kimai \
        -e MYSQL_USER=kimai \
        -e MYSQL_PASSWORD=kimai \
        -e MYSQL_ROOT_PASSWORD=kimai \
        -p 3399:3306 -d mysql
    ```

2. Start Kimai

    ```bash
    docker run --rm --name kimai-test \
        -ti \
        -p 8001:8001 \
        -e DATABASE_URL=mysql://kimai:kimai@${HOSTNAME}:3399/kimai?charset=utf8mb4&serverVersion=8.3.0 \
        kimai/kimai2:apache
    ```

3. Add a user using the terminal

    ```bash
    docker exec -ti kimai-test \
        /opt/kimai/bin/console kimai:user:create admin admin@example.com ROLE_SUPER_ADMIN
    ```

4. Stop the containers

    ```bash
    docker stop kimai-mysql-testing kimai-test
    ```
 
5. When you are finished testing Kimai, you can remove the containers (warning: **you will lose your data**!).

    ```bash
    docker rm kimai-mysql-testing kimai-test
    ```

If you are happy with Kimai, you can now setup your Docker installation using [Docker Compose]({% link _documentation/docker/docker-compose.md %}).

## Updating Kimai

The usual update step is simple: stop, pull latest version, restart. 

This example is based on the `Apache` image used with the `Docker compose` plugin:

```bash
# Pull latest version
docker compose pull
# Stop and remove older version
docker compose down
# Start the container
docker compose up -d
```

### FPM image

The FPM image will need to be upgraded with a manual step. 
Because the FPM image will have a HTTP proxy (e.g. caddy or nginx) serving the static assets the `public` directory is mounted into that container. This is done via volumes:

```yaml
version: '3.5'
services:
    kimai:
        image: kimai/kimai2:latest
        ...
        volumes:
            - public:/opt/kimai/public
        ...
    nginx:
        ...
        volumes:
            - public:/opt/kimai/public:ro
    ...
```

When the kimai image is updated, and the container is restarted any new assets in the public directory are never included. These will be things like CSS files, images and especially version specific javascript code! To fix this you need to copy the newer files from a fresh image over the top.

```bash
# You might need to use `docker volume ls | grep kimai` to find the name of your Kimai "public" volume
docker run --rm -ti -v kimai_public:/public --entrypoint /bin/bash kimai/kimai2
#                      ^^^^^^^^^^^^ -> Kimai public volume
cp -r /opt/kimai/public /
exit
```

Now you'll need to tell the running kimai to update its assets:

```bash
docker-compose exec kimai /opt/kimai/bin/console assets:install
```

That should do it.

{% comment %} 

### Build the docker

```bash
docker build -t my-local-kimai .
```

### Run the docker

```bash
docker run -ti -p 8001:8001 --name kimai2 --rm my-local-kimai
```

You can then access the site on http://127.0.0.1:8001. If that doesn't work check the IP of your docker:

```bash
{% raw %}docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kimai2{% endraw %}
```

#### Mac using docker-machine

When using docker-machine on your Mac, you need to use the IP of your machine.
Considering you started the machine named `default`, you find the IP with:

```bash
docker-machine ip default
```

### Using a custom local.yaml

You can mount a [custom configuration]({% link _documentation/local-yaml.md %}) into the container while starting it:
```bash
docker run --rm -ti -p 8001:8001 --name kimai2 -v $(pwd)/config/packages/local.yaml:/opt/kimai/config/packages/local.yaml kimai/kimai2:dev
```

{% endcomment %}

The [official docker documentation](https://docs.docker.com/) has more options on running the container.

## Tips & Tricks

If you're using Docker for Windows or Docker for Mac, and you're getting "Connection refused" or other errors, you might need to change `${HOSTNAME}` to `host.docker.internal`.
This is because the Kimai Docker container can only communicate within its network boundaries. Alternatively, you can start the container with the flag `--network="host"`.
See [here](https://stackoverflow.com/questions/24319662/from-inside-of-a-docker-container-how-do-i-connect-to-the-localhost-of-the-mach) for more information.
