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

In all cases, the services can be broken, insecure, etc. after an update until the admin looks for *.rpmsave and *.rpmnew files and merges his changes with the new configuration file.

## Existing Solutions

* Three-Way-Diff
  * Conflicts still need to be solved manually
* /etc contains symlinks
  * Files are always current and in the right version as long as an admin does not modify them
  * Admin has to replace them with a copy of the file
  * Admin has to check after every update, if adjustements in the copy are needed
* Systemd like
  * /usr/lib/.../ → Main configuration
  * /etc/... → Admin changes
