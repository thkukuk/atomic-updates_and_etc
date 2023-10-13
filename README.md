# Atomic Updates and /etc

`Version 3.1, 2023-10-13`

## Rationale

More and more Linux Distributors have a Distribution using atomic updates to
update the system (for SUSE/openSUSE it's transactional-update). They all have
the problem of updating the files in `/etc`, but everybody come up with
another solution which solved their usecase, but is not generic useable.


Additional there are the ["Factory
Reset"](http://0pointer.net/blog/projects/stateless.html) and
["systemd.volatile="](https://www.freedesktop.org/software/systemd/man/systemd-volatile-root.service.html)
features from systemd, which no distribution has really fully implemented
today. A unique handling of /etc for atomic updates could also help to
convience upstream developers to add support to their applications for this
cases.

## Upstream

Meanwhile the problem and requirements are also seen in other projects or needed for additional use cases:

* Hermetic `/usr/`, as explained in the blog [Fitting Everything Together](https://0pointer.net/blog/fitting-everything-together.html)
* [UAPI Group tracking upstream projects not supporting hermetic-usr for configuration](https://github.com/uapi-group/specifications/issues/76)
* Formal [Configuration Files Specification](https://github.com/uapi-group/specifications/blob/main/specs/configuration_files_specification.md)

## Goal

The goal is to provide a concept working for most Linux Distributors (like the
FHS, preferred is to get this into the FHS). Short to midterm, it should solve
the problems with atomic updates. Midterm to longterm, the result should be,
that no package installs anything in `/etc`, it should only contain host
specific configuration files, changes made by the system administrator or
configuration files managed by the system administrator.

## Definition "Atomic Updates"

An atomic update is a kind of update that:

* Is atomic
  * Either fully applied or not at all
  * The update does not influence your running system
* Can be rolled back
  * If the update fails or if the udpate is not compatible, you can quickly restore the situation as it was before the update


## RPM handling of config files

There are different ways to handle config files in RPM. They can be normal
files in the filelist of the spec file, they can be marked with %config or
%config(noreplace).
During an update, if the configuration file is modified and the RPM contains a new configuration file:
1. files not marked: an update would replace the file and all changes are lost
2. files marked as %config: modified files are moved away as \*.rpmsave
3. files marked as %config(noreplace): modified files stay, new files are written as \*.rpmnew

In all cases, the services can be broken, insecure, etc. after an update until
the admin has to look for \*.rpmsave and \*.rpmnew files and merge his changes
with the new configuration files. This is already troublesame.

With atomic updates, another layer of complexity is added: after an update,
before the reboot, the changes done by the update are not visible. If an admin
changes the configuration files in this time, they will be, depending of how
the atomic update is implemented, most likely lost. Or RPM does not see, that
there are modified versions of the config file (e.g. they are stored in an
overlay filesystem) and thus does not even create \*.rpmsave or \*.rpmnew
files. In this case, it's really complicated for the admin to find out that
there is a new configuration file and what he has to adjust, since the new
file is shadowed by the copy in the overlay filesystem.

## Requirements for a solution

So we need a new way to store and manage configuration files, which:
* It's visible to the admin that something got updated
* It's visible, which changes the admin made
* Best if the changes could be merged automatically
* There should be only one directory to search below for default configuration files

Another problem is, that we have different kinds of "configuration" files:
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

For SUSE/openSUSE the decission was made to use `/usr/etc` as the directory
below `/usr` for the distribution provided configuration files.
In theory, this can be replaced with any other directory.

### Application configuration files

Problem: this problem is not alone for atomic updates, but also for normal
distributions: the administrator makes changes to a configuration file and the
next update contains a complete new, incompatible configuration file for that
package. Somehow the admin needs to be informed about this and the changes
needs to be ported to the new config file.
Additional, it should not be necessary for a "Factory Reset" or something
simmilar to copy many configuration files from somewhere to `/etc` (which
means write systemd-tmpfiles for everything), but the application should look
in the right directory for the system default configuration file and apply
changes from `/etc`.

#### Formal Proposal

Something similar to what systemd is already doing today:

Applications install their default configuration in
`/usr/etc/<application>/`. The admin can copy this configuration file to
`/etc/<application>/` and adjust that. The application prefers the
`/etc/<application>` version over the
`/usr/etc/<application>/` version. This possibility is important if the
configuration file is in a format, where parts cannot be overwritten easily,
like xml or json.

If the format of the configuration file allows overrides of entries, there
should be a `/etc/<application>/app.conf.d` directory in which local changes can be
written. The application has to merge this entries when reading the
configuration.

As workflow:
* Applications looks for `/etc/<application>/app.conf`.
* If this file does not exist, load `/usr/etc/<application>/app.conf`
* Look for overrides in `/etc/<application>/app.conf.d` and merge them

See https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Examples, "Overriding vendor settings" for more details and examples.

#### Analysis

While this sounds at first for a lot of work, an analysis of configuration
files in `/etc` on an openSUSE Tumbleweed Base installation shows, that
already many applications supports `.d` directories with configuration
snippets (aliases.d, ant.d, bash-completion.d, chrony.d, cron.d, depmod.d,
dnsmasq.d dracut.conf.d, grub.d, issue.d, logrotate.d,  modprobe.d,
netconfig.d, sudoers.d, sysctl.d,  ...) and there are already applications,
which installs their default configuration file below `/usr/`. So let's at
first make use of it, combine it and re-configure it for our needs.

The disadvantage of the currently used `/usr/lib` directory is, based on
feedback from several talks: the files are spread over different directories
in `/usr/lib`, combined with binary files. Admins don't like that, as it is
hard to find something for them with grep. For this reasons, all packages,
which have a .d directory in `/usr/lib`, should get an additional directory
in `/usr/etc` and the search path should be, for compatibility reasons, the
following:
1. /etc/<app>.d/
2. /usr/lib/<app>.d/
3. /usr/etc/<app>.d/

`/usr/lib/<app>.d` should only be there for compatibilty reasons and no longer
be used by distribution provided packages.

A very flexible ini file style config parser is currently under development,
which makes it really easy for an application to merge all the artefacts
together: [libeconf](https://github.com/parlt91/libeconf)

There is also stuff, which does not belong to `/etc` at all. `/etc/uefi/certs`
does contain binary files installed by several RPMs. This are clearly no config
files (not editable by the user) and thus belongs to `/usr/lib`, `/usr/share`
or something similar.

By simple packaging changes, many problems with atomic updates and "Factory
Reset" would be already solved. There are many more packages for which the
problem could be solved relative simple by changes in how it gets packaged.

#### Note

This proposal does not solve all problems, there are still many open
issues. But this proposal, especially consolidating all distribution
provided configuration files in one place, is already a big step in the right
direction and also helps package managers to easier track configuration files
and their changes.

### System databases (rpc,services,protocols)

There are files in `/etc` which are strictly spoken no configuration
files. Like `/etc/rpc`, `/etc/services` and `/etc/protocols`. They are changed
very seldom, but sometimes new system applications or third party software
needs to make additions.
Move this files to `/usr/etc` and let the system search at first in `/etc` and
afterwards in `/usr/etc`. `/etc` would contain only the changes done by the
admin and third party software.
A glibc NSS plugin to read files in `/usr/etc` exist already
(https://github.com/kubic-project/libnss_usrfiles), so it is easy configurable
and the admin sees, which changes he made.

### /etc/passwd, /etc/group and /etc/shadow

Problem: if a package installation or update creates a new user or group, this
new user and/or group are not visible until the next reboot. If a user changes
his password or other account information meanwhile, the account from the
snapshot cannot be merged anymore with the changes in `/etc` and either the
account is not visible and the software requiring it fails, or the password of
the user will be resetted to the old version. If a new user is created by the
admin, it could get the same UID and afterwards there is a security problem.
systemd-sysusers will not solve this problem, as the new user is often already
requried for installation of the package and created in the %pre section.

While several options were evaluated, we didn't found a real good solution.

#### /etc/{passwd.d,group.d,shadow.d}

A natural solution would be to also use `.d` directories, so split
`/etc/passwd` into single files for every entry and store them in
`/etc/passwd.d`. Every entry would need an unique filename, so that it is easy
to merge it later. Which should solve the problem of modified accounts hiding
new accounts, but the problem of duplicate UIDs will exist.
There are several implementations (Openwall tcb,
https://github.com/thkukuk/libnss_filesd), but the big disadvantage is: every
tool able to modify accounts needs to support this, and there are only patches
for preliminary support of a few tools, nothing more.
In this end, this is a huge development efford for something, which does not
solve the most important problems.

#### /usr/etc/{passwd,group,shadow}

Idea: Packages create and manage the needed system accounts in
`/usr/etc/{passwd,group,shadow}`, the admin will create accounts in
`/etc/{passwd,group,shadow}`. Changes of system accounts will be stored in
`/etc`. No need to merge data, but also does not solve the problem of
duplicate UIDs.

Pro:
* glibc NSS plugin to read files in `/usr/etc` exist already (https://github.com/kubic-project/libnss_usrfiles)
* Easy to configure in `/etc/nsswitch.conf`: `passwd: files usrfiles`. If `/etc` does not contain the entry, look in `/usr/etc`.

Contra:
* Does not solve all problems
* nss_compat does not work with this
* could be confusing to users, if there are entries for an account in `/etc/shadow`, but not `/etc/passwd`.


## Where to store original or system configuration files

As there is not yet a standard directory below `/usr`, a new one needs to be created. There are some requirements:
* Easy to find and remember for the system administrator (everything in one place)
* No conflict with FHS
* The name should not confuse administrators
* It should be clear, that this are default configuration files

### Currently used by Linux Distributions:
1. `/usr/share/defaults/{etc,skel,ssh,ssl}`: ClearLinux and some packages
2. `/usr/share/{baselayout,skel,pam.d,coreos,...},/usr/lib64/pam.d,...`: CoreOS/Container Linux
3. `/writeable,/etc/writeable`: Ubuntu Core
4. `/usr/etc`: openSUSE MicroOS, RedHat/Fedora/CentOS Atomic

### The decission

The feedback was very clear: `/usr/etc` is the directory which should be used:

* `/usr/etc` - contains everything, which else would be belong to `/etc`
  * aliases, ethers, protocols, rpc, services: read by glibc NSS plugins after versions in `/etc`
  * shells, ethertypes, network: copied with systemd-tmpfiles
* `/usr/etc/skel` - systemd-tmpfiles will copy this files to `/etc/skel` if that directory does not exist
* `/usr/etc/pam.d` - default distribution specific PAM configuration files. `/etc/pam.d` will overwrite this.
* `/usr/etc/<application>` - application specific files, read directly or copied to `/etc` via systemd-tmpfiles
* `/usr/*/<application>` - application specific files, can include configuration files, like today


## Roadmap

The roadmap for openSUSE will be:
1. Create `/usr/etc` in filesystem.rpm
2. Adjust all check scripts to allow `/usr/etc` again
3. Go through `/etc` and move the easy ones
4. Continue with the harder ones
5. Start adjusting the tests to forbid files in `/etc`
6. At some point in time this is hopefully done :)
