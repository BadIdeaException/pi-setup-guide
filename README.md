# pi-setup-guide

This is a  step-by-step guide on (re-)installing my Raspberry Pi home server. While the idea of [automated deployment](https://github.com/chris-33/setup-pi) still has a certain allure to it, I've come to realize that the server simply doesn't have to be rebuilt from scratch often enough to be worth the additional effort. So instead, here is a complete tutorial on how to do it by hand.

## Architecture

All the server's services live in docker containers. This makes it easy to swap them out, isolates them well from each other and the host OS, and provides some extra security. Volumes are kept in a centralized location, making it easy to back up server data. Containers share a common network and are only reachable from the outside through a Traefik reverse proxy, which also handles TLS/SSL and certificate handling. [Here](architecture.md) is a more detailed description.

## Steps to install

1. Start by putting a clean install of [Raspbian](raspbian.md) on the SD card. Make sure to enable SSH.

2. SSH in and [install docker and docker-compose](docker.md).

3. Install a [dockerized nextcloud](nextcloud/nextcloud.md).

4. Install a [dockerized traefik](traefik/traefik.md), which will also handle Let's Encrypt certificate handling.
