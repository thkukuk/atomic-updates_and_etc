# Atomic Updates and /etc

## Rationale

More and more Linux Distributors have a Distribution using atomic updates to update the system (for SUSE it's transactional-update). They all have the problem of updating the files in /etc, but everybody come up with another solution which solved their usecase, but is not generic useable.
Additional there is the "Factory Reset" of systemd, which no distribution has really fully implemented today. A unique handling of /etc for atomic updates could also help to convience upstream developers to add support to their applications.

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

In all cases, the services can be broken, insecure, etc. after an update until the admin looks for \*.rpmsave and \*.rpmnew files and merges his changes with the new configuration file. This is already troublesame.

With atomic updates, another layer of complexity is added: after an update, before the reboot, the changes done by the update are not visible. If an admin changes the configuration files in this time, they will be, depending of how the atomic update is implemented, most likely lost. Or RPM does not see, that there are modified versions of the config file (e.g. they are stored in an overlay filesystem) and thus does not even create \*.rpmsave or \*.rpmnew files. In this case, it's really complicated for the admin to find out that there is a new configuration file and what he has to adjust, since the new file is shadowed by the copy in the overlay filesystem.

## Requirements for a solution

So we need a new way to store and manage configuration files, which:
* It's visible to the admin that something got updated
* It's visible, which changes the admin made
* Best if the changes could be merged automatically

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

If this proposals mentions `/usr/etc`, this is only used as example for a location below `/usr`. This can be replaced with any other path in the `/usr` namespace.
  
### /etc/passwd, /etc/group and /etc/shadow

Problem: if a package installation or update creates a new user or group, this new user and/or group are not visible until the next reboot. If a user changes his password or other account information meanwhile, the account from the snapshot cannot be merged anymore with the changes in /etc and either the account is not visible and the software requiring it fails, or the password of the user will be resetted to the old version. If a new user is created by the admin, it could get the same UID and afterwards there is a security problem.

#### /etc/{passwd.d,group.d,shadow.d}

Split `/etc/passwd` into single files for every entry and store them in `/etc/passwd.d`. Every entry would need an unique filename, so that it is easy to merge it later. So the problem of modified accounts hiding new accounts would be solved, but the problem of duplicate UIDs will exist.

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

#### /usr/etc/{passwd,group,shadow}

Idea: Packages create and manage the needed system accounts in `/usr/etc/{passwd,group,shadow}`, the admin will create accounts in `/etc/{passwd,group,shadow}`. Changes of system accounts will be stored in `/etc`. No need to merge data, but also does not solve the problem of duplicate UIDs.

Pro:
* glibc NSS plugin to read files in `/usr/etc` exist already (https://github.com/kubic-project/libnss_usrfiles)
* Easy to configure in `/etc/nsswitch.conf`: `passwd: files usrfiles`. If `/etc` does not contain the entry, look in `/usr/etc`.

Contra:
* Does not solve all problems
* nss_compat does not work with this
* could be confusing to users, if there are entries for an account in `/etc/shadow`, but not `/etc/passwd`.

#### /usr/etc/{passwd,group,shadow} with system user policy

This idea is based on the abvoe `/usr/etc` proposal, but is enhanced with a policy: System accounts are only to be created in `/usr/etc`, normal accounts have to be created in `/etc`. As the UID space for system accounts and normal accounts is different, and packages should only create system accounts, the problem of duplicate UIDs should be solved. If an administrator has to create a system account, he has to do that in `/usr/etc`. As `/usr` is normally read-only on such systems, the distributor has to provide a solution for this. On openSUSE/SUSE, our packaging guidline (https://en.opensuse.org/openSUSE:Packaging_guidelines#Users_and_Groups) provides such a solution.

Pro:
* glibc NSS plugin to read files in `/usr/etc` exist already (https://github.com/kubic-project/libnss_usrfiles)
* Easy to configure in `/etc/nsswitch.conf`: `passwd: files usrfiles`. If `/etc` does not contain the entry, look in `/usr/etc`.
* Solves duplicate UID problem

Contra:
* nss_compat does not work with this
* Needs changes in useradd and systemd-sysusers

  
## Where to store original or system configuration files
### Currently used by Linux Distributions
1. /usr/share/defaults/{etc,skel,ssh,ssl): ClearLinux
2. /usr/share/{baselayout,skel,pam.d,coreos,...},/usr/lib64/pam.d,...: CoreOS/Container Linux
3. /writeable,/etc/writeable: Ubuntu Core
4. /usr/etc: openSUSE MicroOS, RedHat/Fedora/CentOS Atomic

### My current favorite
* /usr/share/defaults - contains everything, which else would be belong to /etc
* /usr/share/defaults/etc - passwd, group, shadow containing system users
* /usr/share/defaults/etc - aliases, ethers, protocols, rpc, services: read by glibc NSS plugins after versions in /etc
* /usr/share/defaults/etc - shells, ethertypes, network: copyied with systemd-tmpfiles
* /usr/share/defaults/skel - systemd-tmpfiles will symlink this files into /etc/skel
* /usr/share/defaults/pam.d - default distribution specific PAM configuration files. /etc/pam.d will overwrite this.
* /usr/share/defaults/\<application\> - application specific files, read directly or copied to /etc via systemd-tmpfiles
* /usr/\*/\<application\> - application specific files, can include configuration files, like today
