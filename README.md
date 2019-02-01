# Atomic Updates and /etc


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
2. files marked as %config: modified files are moved away as *.rpmsave
3. files marked as %config(noreplace): modified files stay, new files are written as *.rpmnew

In all cases, the services can be broken, insecure, etc. after an update until the admin looks for *.rpmsave and *.rpmnew files and merges his changes with the new configuration file. This is already troublesame.

With atomic updates, another layer of complexity is added: after an update, before the reboot, the changes done by the update are not visible. If an admin changes the configuration files in this time, they will be, depending of how the atomic update is implemented, most likely lost. Or RPM does not see, that there are modified versions of the config file (e.g. they are stored in an overlay filesystem) and thus does not even create \*.rpmsave or \*.rpmnew files. In this case, it's really complicated for the admin to find out that there is a new configuration file and what he has to adjust, since the new file is shadowed by the copy in the overlay filesystem.

## Requirements for a solution

So we need a new way to store and manage configuration files, which:
* It's visible to the admin that something got updated
* It's visible, which changes the admin made
* Best if the changes could be merged automatically

Another problem is, that we have different kind of "configuration" files:
1. configuration files for applications
2. configuration files for the system (network, hardware, ...)
3. "databases" like /etc/rpc, /etc/services, /etc/protocols
4. system and user accounts (/etc/passwd, /etc/group, /etc/shadow, ...)

There are already different solutions. But they all cover more or less only one part, but not everything.

## Existing Solutions

* Three-Way-Diff
  * Conflicts still need to be solved manually
* /etc contains symlinks
  * Files are always current and in the right version as long as an admin does not modify them
  * Admin has to replace them with a copy of the file
  * Admin has to check after every update, if adjustements in the copy are needed
* Systemd like
  * /usr/lib/.../ -> Main configuration
  * /etc/... -> Admin changes
  
## Proposals
  
### passwd, group and shadow
  
  
## Where to store original or system configuration files
### Currently used by Linux Distributions
1. /usr/share/defaults/{etc,skel,ssh,ssl): ClearLinux
2. /usr/share/{baselayout,skel,pam.d,coreos,...},/usr/lib64/pam.d,...: CoreOS/Container Linux
3. /writeable,/etc/writeable: Ubuntu Core
4. /usr/etc: openSUSE MicroOS, RedHat/Fedora/CentOS Atomic

### My current favorite
* /usr/share/defaults - contains everything, which else would be belong to /etc
* /usr/share/defaults/etc - passwd, group, shadow containing system users
* /usr/share/defaults/etc - rpc, services, protocols: read by glibc NSS plugins after versions in /etc
* /usr/share/defaults/etc - shells, [ethers, ethertypes, network]: copyied with systemd-tmpfiles
* /usr/share/defaults/skel - systemd-tmpfiles will symlink this files into /etc/skel
* /usr/share/defaults/pam.d - default distribution specific PAM configuration files. /etc/pam.d will overwrite this.
* /usr/share/defaults/\<application\> - application specific files, read directly or copied to /etc via systemd-tmpfiles
* /usr/\*/\<application\> - application specific files, can include configuration files, like today
