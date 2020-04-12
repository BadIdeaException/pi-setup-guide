# Installing Docker and Docker Compose

In this step, we will install the Docker engine and it's convenience side kick, Docker Compose. 

## Installing Docker

Theoretically, installing Docker could be as simple as adding their repository and apt-getting it. Unfortunately, using the docker repository is not supported for Raspbian (as of April 2020). Instead, you have to [use the installation script](https://docs.docker.com/engine/install/debian/#install-using-the-convenience-script):

```
cd /tmp
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Run `docker version` to check if it worked. (Or, if you want to be really sure, `docker run hello-world`.)

To enable Docker on boot, run `systemctl enable docker`. 

## Installing Docker Compose

Theoretically, the Docker engine would be enough. However, you would have to start containers using a barrage of command line parameters. This is where Docker Compose can make life a lot easier, by allowing container startup parameters and their relationships to each other in service definition files. 

Unfortunately, much like with the Docker engine itself, the [default method using a convenience script](https://docs.docker.com/compose/install/#alternative-install-options) doesn't seem to work on Raspbian (as of April 2020). Instead, [install using pip](https://docs.docker.com/compose/install/#install-using-pip):

```
apt-get install -y libffi-dev libssl-dev python3 python3-pip 
pip3 install docker-compose
```

Again, you can check if it worked by doing `docker-compose version`.

## Preparing some infrastructure

Now that the tools are in place, this is a good time to lay down some basic infrastructure for your containers. Make

- a shared network for your containers and communicate with each other: `docker networt create internal-net`

- a place for their volumes to live: `mkdir /var/vol`

That's it. You can now begin to deploy services. Maybe start with [Nextcloud](nextcloud/nextcloud.md).
