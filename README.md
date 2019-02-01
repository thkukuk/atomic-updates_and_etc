# Atomic Updates and /etc


## Definition "Atomic Updates"

An atomic update is a kind of update that:

* Is atomic
  * Either fully applied or not at all
  * The update does not influence your running system
* Can be rolled back
  * If the update fails or if the udpate is not compatible, you can quickly restore the situation as it was before the update


## RPM handling of config files


## Existing Solutions

* Three-Way-Diff
  * Conflicts still need to be solved manually
* /etc contains symlinks
  * Files are always current and in the right version as long as an admin does not modify them
  * Admin has to replace them with a copy of the file
  * Admin has to check after every update, if changes are needed
* Systemd like
  * /usr/lib/\<app\>/ → Main configuration
  * /etc/\<app\>/ → Admin changes
