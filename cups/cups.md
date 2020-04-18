Notes:

Fresh install only:
- Because installing cups is baked into the image, no volumes were mounted yet. So if you mount `/var/vol/cups/config` to '/etc/cups`, it will hide the actual configuration contents, and also the default configuration at install time will not be written to the volume. You need to copy the contents of `/etc/cups` from the container to the volume by hand using `docker cp`

- In cupsd.conf, you need to replace the Listen directives with Port 631 to listen to port 631 on all interfaces

- In cupsd.conf, you need to add `ServerAlias *` to allow access using hostname instead of IP. Otherwise you get an HTTP Bad Request response

- After you first start the daemon, `exec` into the container and run `cupsctl --remote-admin --remote-any --share-printers` to enable admin over the web interface and share the printers
