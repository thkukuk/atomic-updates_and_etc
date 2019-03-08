# Linux-PAM

## Status

Linux-PAM has already everything to move the distribution specific config
files from `/etc/pam.d` to `/usr/lib/pam.d`.

## Todo

Necessary changes:
* make `/usr/lib/pam.d` configureable at build time
* move config files
* adjust pam-config
* libeconf for /etc/login.defs
