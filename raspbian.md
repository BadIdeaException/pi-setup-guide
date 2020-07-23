# Install Raspbian

How to install a fresh Raspbian onto an SD card is very well documented on the [Raspberry Pi website](https://www.raspberrypi.org). It basically consists of these two (or three) steps:

1. Grab the [current image](https://www.raspberrypi.org/downloads/raspbian/).

2. Follow the [installation instructions](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md).

3. If you want a static IP configuration, follow the steps in <https://www.raspberrypi.org/documentation/configuration/tcpip/README.md>. I usually prefer configuring my DHCP server to issue a fixed address to the Pi instead.

## Enabling SSH by default

By default, SSH is disabled. Of course you could power up the server, attach a keyboard and a display, and enable SSH. Fortunately, there's a [simpler way](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md).

For this, mount the boot partition and place an empty file named `ssh` into it. This will enable SSH on the next boot.

```
mount /dev/sdX1 <mountpoint>
cd <mountpoint>
touch ssh
```
where `X` is the drive letter corresponding to the SD card and `mountpoint` is the target folder to mount to, e.g. `/media`. 

You can now power up the Pi and ssh into it. The default user is called `pi` and the default password is `raspberry`.

## Changing the default user

Obviously, leaving the standard login active (with sudo privileges, at that) on a machine reachable from the internet is a huge security risk. Instead, create a new user and give it sudo privileges:

```
adduser <username>
usermod -a -G adm,dialout,cdrom,sudo,audio,video,plugdev,games,users,input,netdev,gpio,i2c,spi <username>
```
Check that the new user actually does have sudo privileges by running `sudo su - <username>`. Or, if you want to be absolutely sure, close the ssh connection and reopen it with the new user: `ssh <username>@<ip>`, then try to run `sudo su`.

Next, delete the default `pi` user. If you did not close the ssh connection in the step above, you have to first terminate the `pi` user's process with `pkill -u pi`. Now delete the user: 

```
deluser -remove-home pi
```
_Side note: According the corresponding section of the [Raspbian configuration documentation](https://www.raspberrypi.org/documentation/configuration/security.md), "with the current Raspbian distribution, there are some aspects that require the pi user to be present. If you are unsure whether you will be affected by this, then leave the pi user in place. Work is being done to reduce the dependency on the pi user." 
I have never had any problems without having a `pi` user, but if you run into problems, just change `pi`'s password and leave the user in place.)_

## Auto-mount a data drive

If you want the server's data to be saved on a separate drive, like an external hard drive, this is a good time to set this up for auto-mounting. Edit `/etc/fstab` and add the following line:

```
UUID=<uuid> /var/vol defaults,relatime 0 0
```
You can find out the UUID of the drive using the command `blkid`. Note that if the drive is partitioned, you need to use `PARTUUID` instead. The above line automounts the drive with the corresponding UUID to `/var/vol` regardless of device number in read-write mode and disables `fsck` on startup on it (that's the second 0).

Be careful - a bad `fstab` can prevent the system from booting.

## Some more basic admin

There are some minor bits and pieces left to be done. Fortunately, they can all be done conveniently by using the Raspbian admin tool `raspi-config`:

1. Expand the filesystem to use the entire SD card (found in Advanced Options)

2. Generate the default locales 

3. Change the hostname (Network Options). 

And finally, of course, doing `apt-get update & apt-get upgrade` is not a bad idea!

Reboot. You're ready for the [next step](docker.md).

