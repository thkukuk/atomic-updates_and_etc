# Atomic Updates and /etc

`Version 2.4, 2019-07-19`

## Rationale

More and more Linux Distributors have a Distribution using atomic updates to
update the system (for SUSE it's transactional-update). They all have the
problem of updating the files in `/etc`, but everybody come up with another
solution which solved their usecase, but is not generic useable.

Additional there is the ["Factory Reset"](http://0pointer.net/blog/projects/stateless.html) of systemd, which no distribution has really fully implemented today. A unique handling of /etc for atomic updates could also help to convience upstream developers to add support to their applications.

## Goal

The goal is to provide a concept working for all Linux Distributors (like the FHS, preferred is to get this into the FHS). Short to midterm, it should solve the problems with atomic updates. Midterm to longterm, the result should be, that no package installs anything in `/etc`, it should only contain changes made by the system administrator or configuration files managed by the system administrator.

## Definition "Atomic Updates"

An atomic update is a kind of update that:

* Is atomic
  * Either fully applied or not at all
  * The update does not influence your running system
* Can be rolled back
  * If the update fails or if the udpate is not compatible, you can quickly restore the situation as it was before the update


## RPM handling of config files

There are different ways to handle config files in RPM. They can be normal files in the filelist of the spec file, they can be marked with %config or %config(noreplace).
During an update, if the configuration file is modified and the RPM contains a new configuration file:
1. files not marked: an update would replace the file and all changes are lost
2. files marked as %config: modified files are moved away as \*.rpmsave
3. files marked as %config(noreplace): modified files stay, new files are written as \*.rpmnew

In all cases, the services can be broken, insecure, etc. after an update until
the admin has to look for \*.rpmsave and \*.rpmnew files and merge his changes with the new configuration files. This is already troublesame.

With atomic updates, another layer of complexity is added: after an update, before the reboot, the changes done by the update are not visible. If an admin changes the configuration files in this time, they will be, depending of how the atomic update is implemented, most likely lost. Or RPM does not see, that there are modified versions of the config file (e.g. they are stored in an overlay filesystem) and thus does not even create \*.rpmsave or \*.rpmnew files. In this case, it's really complicated for the admin to find out that there is a new configuration file and what he has to adjust, since the new file is shadowed by the copy in the overlay filesystem.

## Requirements for a solution

So we need a new way to store and manage configuration files, which:
* It's visible to the admin that something got updated
* It's visible, which changes the admin made
* Best if the changes could be merged automatically
* There should be only one directory to search below for default configuration files

Another problem is, that we have different kind of "configuration" files:
1. configuration files for applications
2. configuration files for the system (network, hardware, ...)
3. "databases" like `/etc/rpc`, `/etc/services`, `/etc/protocols`
4. system and user accounts (`/etc/passwd`, `/etc/group`, `/etc/shadow`, ...)

There are already different solutions. But they all cover more or less only one part, but not everything.

## Existing Solutions

* Three-Way-Diff
  * Conflicts still need to be solved manually
* `/etc` contains symlinks
  * Files are always current and in the right version as long as an admin does not modify them
  * Admin has to replace them with a copy of the file
  * Admin has to check after every update, if adjustements in the copy are needed
* Systemd like
  * `/usr/lib/.../` -> Main configuration
  * `/etc/...` -> Admin changes

## Proposals

If this proposals mentions `/usr/config`, this is only used as example for a location below `/usr`. This can be replaced with any other path in the `/usr` namespace.

### Application configuration files

Problem: this problem is not alone for atomic updates, but also for normal
distributions: the administrator makes changes to a configuration file and the
next update contains a complete new, incompatible configuration file for that
package. Somehow the admin needs to be informed about this and the changes
needs to be ported to the new config file.
Additional, it should not be necessary for a "Factory Reset" to copy many
configuration files from somewhere to `/etc` (which means write systemd-tmpfiles
for everything), but the application should look in the right directory for
the system default configuration file and apply changes from `/etc`.

#### Analysis

An analysis of configuration files in `/etc` on an openSUSE Tumbleweed
installation shows, that already many applications supports `.d` directories
with configuration snippets (aliases.d, ant.d, bash-completion.d, chrony.d,
cron.d, depmod.d, dnsmasq.d dracut.conf.d, grub.d, issue.d, logrotate.d,
modprobe.d, netconfig.d, sudoers.d, sysctl.d,  ...) and there are already
applications, which installs their default configuration file in
`/usr/config/<application>` (at-spi2, telemetrics,...). So let's
combine that.

Look at first at Linux-PAM
We have:
* `/usr/lib/pam.d`
* `/etc/pam.d`

Why do Linux Distributions install everything in `/etc/pam.d`? Why is nearly
nobody using `/usr/lib/pam.d`? Maybe `/usr/lib/pam.d` does not really sound
like configuration files?

Compare this with `sysctl.conf`:
* `/usr/lib/sysctl.d/*.conf`
* `/etc/sysctl.d/*.conf`
* `/etc/sysctl.conf`

The distributor put's it's default configuration in `/usr/lib/sysctl.d`, the
admin can overwrite them by putting the changes in `/etc/sysctl.d`. The
`/etc/sysctl.conf` file isn't needed at all, it shouldn't be installed.
Same for dracut, it works the same way, including that `/etc/dracut.conf` only contains the comment where to put the configuration.

For glibc/ldconfig a solution could look like:
* mv `/etc/ld.so.conf` to `/usr/config/ldconfig/ld.so.conf`
* `/usr/config/ldconfig/ld.so.conf.d` for distribution specific paths, e.g. libgraphivz6
* `/etc/ld.so.conf.d` for local changes, like nvidia-gfxG04.conf

The disadvantage of this is, based on feedback from several talks: the files
are spread over different directories in `/usr/lib`, admins don't like that, as
it is hard to find something for them.

There is also stuff, which does not belong to `/etc` at all. `/etc/uefi/certs`
contain binary files installed by several RPMs. This are clearly no config
files (not editable by the user) and thus belongs to `/usr/share` or something
similar.

By simple packaging changes, many problems with atomic updates and "Factory
Reset" would be already solved. There are many more packages for which the problem
could be solved relative simple by changes in how it gets packaged.

#### Formal Proposal

Something similar to what systemd is already doing today:

Applications install their default configuration in
`/usr/config/<application>/`. The admin can copy this configuration file to
`/etc` and adjust that. The application prefers the `/etc/` version over the
`/usr/config/<application>/` version. This possibility is important if the
configuration file is in a format, where parts cannot be overwritten easily,
like xml or json.

If the format of the configuration file allows overrides of entries, there
should be a `/etc/application.conf.d` directory in which local changes can be
written. The application has to merge this entries when reading the
configuration.

As workflow:
* Applications looks for `/etc/application.conf`.
* If this file does not exist, load `/usr/config/application/application.conf`
* Look for overrides in `/etc/application.conf.d` and merge them

See https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Examples, "Overriding vendor settings" for more details and examples.

### System databases

There are files in `/etc` which are strictly spoken no configuration files. Like `/etc/rpc`, `/etc/services` and `/etc/protocols`. They are changed very seldom, but sometimes new system applications or third party software needs to make additions.
Move this files to `/usr/config` and let the system search at first in `/etc` and afterwards in `/usr/config`. `/etc` would contain only the changes done by the admin and third party software.
A glibc NSS plugin to read files in `/usr/config` exist already (https://github.com/kubic-project/libnss_usrfiles), so it is easy configurable and the admin sees, which changes he made.


### /etc/passwd, /etc/group and /etc/shadow

Problem: if a package installation or update creates a new user or group, this
new user and/or group are not visible until the next reboot. If a user changes
his password or other account information meanwhile, the account from the
snapshot cannot be merged anymore with the changes in /etc and either the
account is not visible and the software requiring it fails, or the password of
the user will be resetted to the old version. If a new user is created by the
admin, it could get the same UID and afterwards there is a security problem.
systemd-sysusers will not solve this problem, as the new user is often already
requried for installation of the package and created in the %pre section.

While several options were evaluated, there is no real good
solution.

#### /etc/{passwd.d,group.d,shadow.d}

A natural solution would be to also use `.d` directories, so split
`/etc/passwd` into single files for every entry and store them in
`/etc/passwd.d`. Every entry would need an unique filename, so that it is easy
to merge it later. Which should solve the problem of modified accounts hiding new accounts, but the problem of duplicate UIDs will exist.

Pro:
* glibc NSS plugin to read files in such directories exist already (https://github.com/thkukuk/libnss_filesd)
* Creating new users is more or less easy to implement
* Concept is known to users

Contra:
* Does not really solve all problems
* Support for every tool creating accounts needs to be implemented
* All tools modifying accounts needs support
* All code changing passwords needs support, especially all PAM modules

In this end, this is a huge development efford for something, which does not solve all problems.

#### /usr/config/{passwd,group,shadow}

Idea: Packages create and manage the needed system accounts in `/usr/config/{passwd,group,shadow}`, the admin will create accounts in `/etc/{passwd,group,shadow}`. Changes of system accounts will be stored in `/etc`. No need to merge data, but also does not solve the problem of duplicate UIDs.

Pro:
* glibc NSS plugin to read files in `/usr/config` exist already (https://github.com/kubic-project/libnss_usrfiles)
* Easy to configure in `/etc/nsswitch.conf`: `passwd: files usrfiles`. If `/etc` does not contain the entry, look in `/usr/config`.

Contra:
* Does not solve all problems
* nss_compat does not work with this
* could be confusing to users, if there are entries for an account in
`/etc/shadow`, but not `/etc/passwd`.
* `/usr/share` and passwd, group and shadow don't fit really together, as they
are not shareable across systems and architectures. Alternate path could be
`/usr/lib/sysimage/etc`, but this is not obvious for users.

#### /usr/config/{passwd,group,shadow} with system user policy

This idea is based on the abvoe `/usr/config` proposal, but is enhanced with a policy: System accounts are only to be created in `/usr/config`, normal accounts have to be created in `/etc`. As the UID space for system accounts and normal accounts is different, and packages should only create system accounts, the problem of duplicate UIDs should be solved. If an administrator has to create a system account, he has to do that in `/usr/config`. As `/usr` is normally read-only on such systems, the distributor has to provide a solution for this. On openSUSE/SUSE, our packaging guidline (https://en.opensuse.org/openSUSE:Packaging_guidelines#Users_and_Groups) provides such a solution.

Pro:
* glibc NSS plugin to read files in `/usr/config` exist already (https://github.com/kubic-project/libnss_usrfiles)
* Easy to configure in `/etc/nsswitch.conf`: `passwd: files usrfiles`. If `/etc` does not contain the entry, look in `/usr/config`.
* Solves duplicate UID problem

Contra:
* nss_compat does not work with this
* Needs changes in useradd and systemd-sysusers


## Where to store original or system configuration files

As there is not yet a standard directory below `/usr`, a new one needs to be created. There are some requirements:
* Easy to find and remember for the system administrator (everything in one place)
* No conflict with FHS
* The name should not confuse administrators
* It should be clear, that this are default configuration files and changes should not be done here

### Currently used by Linux Distributions or suggested by others:
1. `/usr/share/defaults/{etc,skel,ssh,ssl}`: ClearLinux and some packages
2. `/usr/share/{baselayout,skel,pam.d,coreos,...},/usr/lib64/pam.d,...`: CoreOS/Container Linux
3. `/writeable,/etc/writeable`: Ubuntu Core
4. `/usr/etc`: openSUSE MicroOS, RedHat/Fedora/CentOS Atomic
5. `/usr/share/sysconfig` (wouldn't people confuse this with `/etc/sysconfig`, means they want to make changes here?)
6. `/usr/share/misc`: used by several tools already, but FHS defines it a little bit different

### The current favorite
* `/usr/config` - contains everything, which else would be belong to `/etc`
  * aliases, ethers, protocols, rpc, services: read by glibc NSS plugins after versions in `/etc`
  * shells, ethertypes, network: copied with systemd-tmpfiles
* `/usr/config/skel` - systemd-tmpfiles will symlink this files into /etc/skel
* `/usr/config/pam.d` - default distribution specific PAM configuration files. `/etc/pam.d` will overwrite this.
* `usr/config/<application>` - application specific files, read directly or copied to `/etc` via systemd-tmpfiles
* `/usr/\*/<application>` - application specific files, can include configuration files, like today
* `/usr/lib/sysimage/etc` - passwd, group, shadow containing system users

`/usr/share` is defined as beeing shareable between different hosts. This is
not always the case for configuration files. Additional passwd, group and
shadow are not shareable between different systems (UID,GID could be
different) at all, that's why this should not be below `/usr/share`. Putting
everything below `/usr/lib/sysimage` is not really intiutive and has already
conflicts (e.g. rpm).
Other suggestions were `/usr/etc` as already used by some distributions or
`/usr/sysconfig`. While in discussions most people would prefer `/usr/etc`, there is a big problem with this directory: quite some Linux and Non-Linux distributions are using this directory already, but with anther purpose. So this distributions would not be able to follow us.
The general comment about `/usr/sysconfig` was, people will confuse it with `/etc/sysconfig` and thus they will not support it.
Currently, `/usr/config` is the most promising directory.
