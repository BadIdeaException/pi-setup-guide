# Reverse proxying through a Traefik container

In this section, we will set up [Traefik](https://docs.traefik.io/) as a reverse proxy (also called a gateway) that acts as a single point of entry for all requests incoming on the Pi and distributes them to their proper services. This will, as well, be containerized. 

This has a few advantages to it:

1. It should be less easy to inadvertently expose containers to the outside

2. It offers flexibility: you can serve multiple websites on port 80 (e.g. Nextcloud and a Wordpress blog) using domain-based routing

3. Traefik makes it super easy to get and renew certificates from Let's Encrypt

So let's get started. Most of this was shamelessly taken from [Chris Wiegman's article "Running Nextcloud With Docker and Traefik 2"](https://chriswiegman.com/2020/01/running-nextcloud-with-docker-and-traefik-2/); I just adapted it to my needs.

## Prep work

Create a `docker-compose.yml` in `/srv/traefik` of version 3 and have it set the network called `internal-network` as the default.

```
version: "3"

networks:
    default:
        external:
            name: internal-network
```

Make a volume directory for traefik (`/var/vol/traefik`) and place two files in it: `acme.json` and `traefik.toml`. These are for certificate storage and Traefik's configuration, respectively. For security reasons, `acme.json` must have permissions 600 or Traefik will refuse working with it.

```
mkdir /var/vol/traefik
touch /var/vol/traefik/acme.json
chmod 600 /var/vol/traefik/acme.json
touch /var/col/traefik/traefik.toml
```

What to put in `traefik.toml` will be explained in a [later section](#configuring-traefik). First, we will complete the `docker-compose.yml` file.

## Finishing the container definition

Append to the `docker-compose.yml` file created above the following lines:

```
services:
    traefik:
        image: traefik:2.2
        container_name: traefik
        restart: always
        
        networks:
            - default

        ports:
            - 80:80
            - 443:443

        volumes:
            - /var/vol/traefik/traefik.toml:/etc/traefik/traefik.toml
            - /var/vol/traefik/acme.json:/acme.json
            - /var/run/docker.sock:/var/run/docker.sock:ro
```
This will use Traefik v2.2. (Be sure to have this > v2, as major changes were introduced with that version. Anything older will not work.) It exposes ports 80 and 443 by default - if you want to have additional services routed through traefik, you need to expose their ports as well, obviously. 

Finally, it mounts the previously created `traefik.toml` and `acme.json` as volumes, as well as Docker's socket file `/var/run/docker.sock`. Traefik needs this for its auto-discovery features to work (e.g. what ports to connect to). Be aware that security-wise, this is touchy. There is a way to increase security by exposing the file through a dedicated container and an internal network, see Wiegman's guide. For now, we'll go with the basic version as I feel the risk is acceptable and it's less moving parts.

## Configuring Traefik

Tell Traefik to work with Docker, to not expose containers by default and point it to the Docker socket file by putting the following in `traefik.toml`:
```
[providers.docker]
  exposedByDefault = false
  endpoint = "unix:///var/run/docker.sock"
```

Next, create entrypoints for HTTP, HTTPS and Traefik's own dashboard feature:
```
[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.web-secure]
    address = ":443"
```

Enabling Let's Encrypt certificates is this easy with Traefik: 
```
[certificatesResolvers]
  [certificatesResolvers.default.acme]
    email = "me@email.com"
    storage = "acme.json"
    [certificatesResolvers.default.acme.tlsChallenge]
```
Obviously, replace the email with your actual one, or you won't get notifications from Let's Encrypt about your certificates. Also, their server is rate-limited. If you're experimenting, use their staging server instead by adding this line:
```
caServer = "https://acme-staging-v02.api.letsencrypt.org/directory"
```
If you're done experimenting and want to move on to a legit certificate, remove this line and empty out `acme.json` to force Traefik to get a new certificate on the next start.
