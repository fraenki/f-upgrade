# f-upgrade

#### Table of Contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Usage](#usage)
4. [Caveats](#caveats)
5. [Examples](#examples)
6. [Reference](#reference)
    - [Hooks](#hooks)
    - [Tasks](#tasks)
7. [Compatibility](#compatibility)
8. [Development](#development)
    - [Contributing](#contributing)

## Overview

f-upgrade automates the process of installing FreeBSD operating system updates. It targets server fleets, not desktop environments.

f-upgrade is basically a wrapper around `freebsd-update`. However, it will also support PkgBase when it is officially supported and no longer experimental.

Although f-upgrade is new software, the concept behind it was already used in server fleets in the early 2010s with great success.

Note that there is also a [Puppet module available](https://forge.puppet.com/modules/fraenki/f_upgrade/readme).

## Requirements

Performing unattended operating system upgrades is not impossible. However, the following requirements should be considered before attempting an upgrade:

* The upgrade should be tested for every release
* The upgrade path must be officially supported by FreeBSD (e.g. 13.2-RELEASE to 14.0-RELEASE)
* Knowledge of breaking changes is important (e.g. study FreeBSD Release Notes, Upgrade Instructions)
* Add required pre/post hooks for breaking changes that would affect your servers
* Some kind of automation (Puppet, Ansible, Chef) to fix/update configuration files
* OPTIONAL: Build a custom package repository for all servers using poudriere. This way the OS can be upgraded to a new version, while preserving most applications on their previous (major) version. This is purely optional, but highly recommended.

Note that f-upgrade automatically disables the `MergeChanges` option in freebsd-update, because this would require user-interaction. It is important that this option remains disabled until the upgrade process is finished.

## Usage

If all requirements are met, running unattended FreeBSD upgrades is petty simple:

```
# install
sudo pkg install f-upgrade

# enable (required!)
sudo sysrc f_upgrade_enable=YES

# configure target release
sudo vi /usr/local/etc/f-upgrade.conf
upgrade=14.0

# start upgrade
sudo service f-upgrade start
```

Note that enabling the `f-upgrade` service is mandatory. It's not a real service, but it ensures that the upgrade process resumes after performing a reboot. The system will reboot several times during the upgrade process, so this step is crucial.

Also note that the script and logs remain completely silent, if no upgrade is configured or when the target version matches the currently running version.

It's also useful to add a cron job to start f-upgrade periodically. This way an upgrade is automatically started, whenever an "upgrade" option is added to the configuration:

```
sudo crontab -e

15 * * * * /usr/local/sbin/f-upgrade
```

## Caveats

Unattended upgrades come with some risks. A number of things may go wrong, especially when upgrading to the next major release:

* Changes to /etc/passwd and /etc/group are not automatically applied (because `MergeChanges` is disabled)
* Changes to system configuration files are not automatically applied (because `MergeChanges` is disabled)
* Changes to drivers / hardware support may cause issues
* Packages that were renamed or removed
* Failure in pkg's conflict/dependency resolver

When carefully testing every upgrade path, it should be possible to workaround these issues by adding hook scripts.

## Examples

### Enable debug logging

For extensive debug output, the following options need to be changed in `/usr/local/etc/f-upgrade.conf`:

```
log_debug=2
log_syslog=1
```

This will not only log all script output to the system logs, but also dump the output of every task/hook to `/var/log/f-upgrade`.

### Hook to send mail when upgrade is finished

To inform someone when the upgrade is finished, add hook `/usr/local/etc/f-upgrade.hook.d/8000.post` with the following content:

```
echo "Success" | mail -s "Uprade to ${HOOK_REL} successfully installed on $(hostname)" f-upgrade@example.com
```

Task ID 8000 is the final upgrade task, so this mail will be send immediately when f-upgrade reports success.

### Hook to trigger configuration management

Task ID 5000 performs a reinstallation of all packages. At this point it may be desireable to ensure that the configuration management software is still installed. Besides that it may be a good idea to run it in order to fix configuration files.

Below is an example for Puppet, the hook should be saved as `/usr/local/etc/f-upgrade.hook.d/5000.post`:

```
# ensure that Puppet is installed
pkg install -yf puppet

# run Puppet Agent
puppet agent -t

exit 0
```

## Reference

### Hooks

Hooks are arbitrary commands that are run either before or after a certain upgrade task is executed. This makes it possible to customize the upgrade process to a certain degree.

Hooks are shell scripts that need to be placed in `/usr/local/etc/f-upgrade.hook.d`. Two types of hooks are supported: pre and post. Pre hooks are executed before running an upgrade task, while post hooks are executed afterwards.

The filename of a hook script always contains the type of the hook. Besides that it may optionally contain the version that it applies to. This also affects the order in which hook scripts are executed:

1. `TASK_ID.type.MAJOR.MINOR`, e.g. `1000.pre.14.0` - Hooks that match the exact target release are executed first.
2. `TASK_ID.type.MAJOR`, e.g. `1000.pre.14` - Hooks that match the major target release are executed afterwards.
3. `TASK_ID.type`, e.g. `1000.pre` - Hooks that don't provide any version information are executed last.

Although hooks may contain arbitrary code, some rules should apply:

* Write POSIX-compatible shell code.
* Stick to OS tools (if possible). Try not to rely on ports/packages (they might break during upgrades).
* Add error handling. Hook scripts are expected to return exit code 0 on success.

Hook scripts may rely on and use the following env variables (which are exported by f-upgrade):

```
$HOOK_OSVER (full OS release)
$HOOK_OSVER_MAJOR (major OS release)
$HOOK_OSVER_MINOR (minor OS release)
$HOOK_OSVER_KERNEL (4-digit kernel version)
$HOOK_OSVER_USER (4-digit userland version)
$HOOK_REL (full target release)
$HOOK_REL_MAJOR (major target release)
$HOOK_REL_MINOR (minor target release)
$HOOK_REL_KERNEL (4-digit target kernel version)
$HOOK_REL_USER (4-digit target userland version)
```

### Tasks

f-upgrade splits the whole upgrade process into several tasks:

| task ID | minor upgrade | major upgrade | summary    |
| :---    | :---:         | :---:         | :------    |
| 0       | yes           | yes           | validation passed, ready for upgrade |
| 500     | yes           | yes           | alter freebsd-update.conf |
| 1000    | yes           | yes           | send a warning to all logged-in users |
| 1100    | yes           | yes           | fetch files to install latest patchlevel |
| 1200    | yes           | yes           | install latest patchlevel |
| 2000    | yes           | yes           | fetch files for release upgrade |
| 3000    | yes           | yes           | install new kernel|
| 3100    | yes           | yes           | reboot (activate new kernel) |
| 3200    | yes           | yes           | verify reboot |
| 4000    | yes           | yes           | install new userland |
| 5000    | no            | yes           | reinstall/upgrade all packages |
| 5100    | no            | yes           | clean local package cache |
| 6000    | no            | yes           | cleanup: remove old libraries |
| 7000    | yes           | yes           | reboot into fully upgraded system |
| 7100    | yes           | yes           | verify reboot |
| 8000    | yes           | yes           | verify OS upgrade, report success/failure |

The gaps between task IDs are intentional, they allow to add tasks if future upgrades make this necessary.

## Compatibility

f-upgrade 1.0 was successfully tested for the following upgrade paths:

* FreeBSD 13.0 -> FreeBSD 13.1
* FreeBSD 13.1 -> FreeBSD 13.2
* FreeBSD 13.2 -> FreeBSD 14.0

Some numbers from a low-end VPS (2 CPU cores, 2 GB RAM) with 80 packages installed: a minor upgrade was completed in about 15 minutes, a major upgrade was completed in about 21 minutes.

## Development

Please use the GitHub issues functionality to report any bugs or requests for new features. Feel free to fork and submit pull requests for potential contributions.
