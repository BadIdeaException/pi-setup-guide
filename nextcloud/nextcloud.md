# Installing a dockerized Nextcloud and a database

In this part, we will set up a Nextcloud container and a database container, then make it ready to be routed through Traefik.

First, make a docker-compose file, for instance in `/srv/nextcloud/`:

```
nano /srv/nextcloud/docker-compose.yml
```

## Prep work

The preamble of any docker-compose file is an indication of it's version. We're using version three here, the most current at the time of this writing (April 2020).

Then, tell Docker Compose to use a pre-existing network (the one you created when you installed Docker) as the default network. (Otherwise, Docker Compose will create one when it spins up the containers).

Then start the `services` definitions section.
```
version: "3"

networks:
    default:
        external:
           name: internal-network

services:
```

## The database

Nextcloud is compatible with a variety of databases: PostgreSQL, MySQL, MariaDB, and even SQLite, although this is not recommended for larger installations. Here, we will use MariaDB, which is an Open Source fork of MySQL.

Unfortunately, the [official Docker MariaDB image](https://hub.docker.com/_/mariadb) is unavailable for the armhf architecture. There's only arm64, and even though the more recent Raspberry Pi's have a 64 bit CPU, Raspbian is still a 32 bit OS. (Again, in April 2020). So instead, we will use [webhippie's image](https://hub.docker.com/r/webhippie/mariadb), which seems to be a fairly popular choice for armhf and can therefore be considered both bug-free and safe. ([Linuxserver](https://hub.docker.com/r/linuxserver/mariadb) also have an image for armhf, but they have a bit of a weird configuration, exposing only a single volume `/config` that contains everything from config file to backups to the actual databases. The webhippie one is a little more standard this way, allowing to mount directly to `/var/lib/mysql`.)

```
    db:
        image: webhippie/mariadb
        container_name: mariadb
        networks:            
            default:
                aliases:
                    - mysql

        env_file: /srv/nextcloud/.mariadb.env

        restart: always
        volumes:
            - /var/vol/mariadb:/var/lib/mysql
```
We're also giving it a container name, telling it to connect to the default network (which it would have done anyways) and to be available there under the alias `mysql`. (So it will now be available under the hostname of both `mariadb` and `mysql`.) This would not have been strictly necessary, but if you have moved over to MariaDB from an actual MySQL in the past, it just seemed like it might save trouble down the road. 

We're also telling it to always restart and mounting a volume for the database files to live in. Note that we're **not** exposing ports (3306 by default) - if we did, this container would become available on the Raspberry Pi's 3306 port. We don't need (or want) that; containers on the same network can communicate freely anyway. 

There's one more line: `env_file: /srv/nextcloud/.mariadb.env`. This tells Docker Compose to read environment variables from a file called `.mariadb.env`. This file will contain your mysql root password:
```
MYSQL_ROOT_PASSWORD=<your long and complex password that you hopefully wrote down somewhere>
```
That's also why this file is not uploaded here. ;)

## Nextcloud itself

Unlike MariaDB, the official Nextcloud image does come in an armhf flavor, so we can just use that. Note that we're using a specific version here (18 was the most current at the time of this writing). This has something to do with the upgrade process and is explained [further below](#how-to-upgrade).

We're also giving the container a name again and setting the restart policy to always. Also, by saying the `nextcloud` container `depends_on` the `db` container, we're making sure the database will be spun up first.

We're mounting volumes for the configuration and container data.

By making ports 80 and 443 available on the host, Nextcloud can now be reached from your LAN via HTTP and HTTPS (although the latter is pointless because there are no certificates yet. [Traefik](../traefik/traefik.md) will handle those.) _(Side note: Once you enable traefik, you will have to remove or comment out this part.)_

```
    nextcloud:
        image: nextcloud:18
        container_name: nextcloud

        restart: always
        depends_on:
            - db

        ports:
            - 80:80
            - 443:443

        volumes:
            - /var/vol/owncloud/config:/var/www/html/config
            - /var/vol/owncloud/data:/var/www/html/data
            - /var/vol/owncloud/custom_apps:/var/www/html/custom_apps
```

At this point, you should have a working - albeit insecure! - Nextcloud. Power it up with `docker-compose up -d` to check if it's working. On the first run, this will make Docker Compose download the images, which may take a few minutes. Be aware that even after this is complete, and the containers have been reported as started, it can take a long time until it can be reached by your browser - sometimes as much as four or five minutes! 

## Get it ready for Traefik

We will eventually want Traefik to act as a reverse proxy on our server, both to allow for several websites to be served from the Pi, but also because it makes dealing with Let's Encrypt certificates spectacularly easy. Much of this was taken from https://chriswiegman.com/2020/01/running-nextcloud-with-docker-and-traefik-2/. 

Add the following lines to the `nextcloud` service definition to 

- tell Traefik to expose this service,
- re-route dav requests,
- redirect HTTP requests to HTTPS,
- use TLS,
- and, most importantly, answer to requests to the domain `<your.domain>` (obviously, replace with your actual domain):

```
        labels:
            - "traefik.enable=true"
            - "traefik.http.middlewares.nextcloud-caldav.redirectregex.permanent=true"
            - "traefik.http.middlewares.nextcloud-caldav.redirectregex.regex=^https://(.*)/.well-known/(card|cal)dav"
            - "traefik.http.middlewares.nextcloud-caldav.redirectregex.replacement=https://$${1}/remote.php/dav/"
            - "traefik.http.middlewares.nextcloud-https.redirectscheme.scheme=https"
            - "traefik.http.routers.nextcloud-http.entrypoints=web"
            - "traefik.http.routers.nextcloud-http.rule=Host(`<your.domain>`)"
            - "traefik.http.routers.nextcloud-http.middlewares=nextcloud-https@docker"
            - "traefik.http.routers.nextcloud.entrypoints=web-secure"
            - "traefik.http.routers.nextcloud.rule=Host(`<your.domain>`)"
            - "traefik.http.routers.nextcloud.middlewares=nextcloud-caldav@docker"
            - "traefik.http.routers.nextcloud.tls=true"
            - "traefik.http.routers.nextcloud.tls.certresolver=default"
```
Because this configuration is attached to the definition as labels, they won't hurt anything even if you don't have Traefik running (or even installed). Once you actually want to enable Traefik, be sure to remove the `ports` section, though.

Just to be on the safe side, also add this to the `db` part of the file:
```
        labels:
            - "traefik.enable=false"
```
Strictly speaking, this should not be necessary, as we will configure Traefik to not expose containers by default. But it never hurts to explicitly set it here, as well.

## How to upgrade

For MariaDB, all you should need to do is pull a newer image. For Nextcloud, things are a little more complicated, because Nextcloud only allows going from one major version to the next (i.e. from 18 to 19, from 19 to 20, etc., but not from 18 diretly to 20). That's also why we pinned an image version earlier - if we had just used the default `latest`, it might lead to exactly this in the future.

So to upgrade, stop and remove the current `nextcloud` container: `docker stop nextcloud & docker rm nextcloud`. Then edit `docker-compose.yml` to increment the image version by one. Run `docker-compose up -d`, and the newer image version should be downloaded and the container spun up. 

Unforunately, that's not enough: Nextcloud needs to also update your data structures and database tables to the new version (presumably, that's also why it doesn't want to upgrade over multiple major versions). This can be done easily using the `occ` script, though. Get a bash in the `nextcloud` container as the user `www-data`:
```
docker exec -it -u www-data nextcloud /bin/bash
```
Then run 
```
/var/www/html/occ upgrade
```
That should be all you have to do. For more details, see [Command line upgrade on the occ documentation](https://docs.nextcloud.com/server/15/admin_manual/configuration_server/occ_command.html#command-line-upgrade).

Sometimes this seems to disable non-default apps such as the calendar or contacts app. In this case, just log in as the admin user and re-enable them.
