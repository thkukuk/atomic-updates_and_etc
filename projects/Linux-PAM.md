# Linux-PAM

## Status

Linux-PAM has already everything to move the distribution specific config
files from `/etc/pam.d` to `/usr/lib/pam.d`. Add as last fallback
`/usr/etc/pam.d`.

## Todo

Necessary changes:
* Add `/usr/etc/pam.d` in search list
* move config files
* adjust pam-config
* libeconf for /etc/login.defs
