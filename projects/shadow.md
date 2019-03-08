# shadow

## Status

There are:
/etc/default/useradd
/etc/login.defs
/etc/pam.d/\*

## Todo

Necessary changes:
* move /etc/pam.d/\* to /usr/share/defaults/pam.d
* Use libeconf for /etc/login.defs
* Use systemd-tmpfiles for /etc/default/useradd
