# OpenBSD minimal workstation

Some of my notes for setting up OpenBSD as a minimal workstation

Tested with:

- install66.fs on MSI U135
- install66.fs on Dell Studio 1535

I haven't done exhaustive tests but both seem to work fine, with the exception 
of the Broadcom BCM4315 in the Studio (which currently is without a driver)

A USB wireless adapter or replacing the internal adapter with something 
friendlier should resolve that, but it's stuck with wired ethernet for now


## Guided Install

*Reading: https://www.openbsd.org/faq/faq4.html*

Preparing installation media and running the guided install is fairly
straightforward

Some of my specific practices:

- Configure network interfaces as DHCP during the guided install and manually
  configure them later
- Ensure a non-root user is created
- Enter `no` when asked to start X with xenodm and do initial setup through
  tty (including customizing xenodm itself) before enabling xenodm manually
- Depending on the workstation, using `whole` during disk configuration may be
  preferable if no other operating systems are going to be used on the machine

## After guided install

Remove the installation media

Reboot

Sign in as root after rebooting

## Services

Inspect rc.conf for services configured by default in this release

```console
# cat /etc/rc.conf | less
```

Services enabled by default in 6.6

- cron
- ntpd
- pflogd
- slaacd
- smtpd
- sndio
- spamlogd
- sshd
- syslogd
- pf
- check_quotas
- library_aslr
- savecore

Enable power management (for battery powered devices)

```console
# rcctl enable apmd
# rcctl set apmd flags -A
# rcctl start apmd
```

For a number of reasons I'm not running my mailserver on workstation, so smtp
and spamlogd can be disabled here

```console
# rcctl stop smtpd
# rcctl disable smtpd
# rcctl stop spamlogd
# rcctl disable spamlogd
```

## Configure sshd

I like to change standard services to use non-standard ports when possible

**Replace NON_STANDARD_PORT with preferred ssh port** (e.g. `2222`)

```console
# echo 'Port NON_STANDARD_PORT' >> /etc/ssh/sshd_config
```

I like using public keys for authentication instead of account names and
passphrases when possible

*Reading: `PAGER='less -p AUTHORIZED_KEYS' man sshd` , `man ssh-keygen`*

Disable password authentication to force public key authentication

```console
# echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config
```

Check sftp subsystem is already in config or add it

```console
# grep "Subsystem[[:blank:]]sftp" /etc/ssh/sshd_config || echo -e \
  'Subsystem\tsftp\t/usr/libexec/sftp-server/' >> /etc/ssh/sshd_config
```

For workstations, since they're not frequently left powered on when unattended,
disable sshd and only start it manually before expected use

```console
# rcctl stop sshd
# rcctl disable sshd
```

<!-- Not currently customizing pf macros or rules for workstations
## Configure pf

*Reading: `man pf`, `man pf.conf`, https://www.openbsd.org/faq/pf*
-->

## Configure network

*Reading: `man hostname`, `man myname`, `man mygate`, `man hostname.if`,
 `man dhclient`, `man dhclient.conf`*

## Hostname

**Replace HOSTNAME with desired hostname for the machine** (e.g. foo)

```console
# hostname 'HOSTNAME'
```

## Gateway

**Replace GATEWAY with target gateway addresses (one per line)** (e.g.
  `192.168.1.1` or `10.0.2.1`)

```console
# echo -e 'GATEWAYS' > /etc/mygate
```

## Static address

Inspect interfaces

```console
# ifconfig
```

IPv4

**Replace ADDR4 with static address** (e.g. `192.168.1.2` or `10.0.2.2`)

**Replace NETMASK with netmask** (e.g. `255.255.255.0`)

**Replace BROADCAST with broadcast address** (e.g. `192.168.1.255` or
  `10.0.2.255`)

**Replace IF with target interface** (e.g. `eth0` or `wlan0`)

```console
# echo 'inet ADDR4 NETMASK BROADCAST' > /etc/hostname.IF
```

<!--
I still haven't learned a lot about configuring or using static IPv6, but
this is probably correct
-->

IPv6

*Add `alias` when specifying additional address on already existing interface*

**Replace ADDR6 with static address** (e.g. `fddc:afc0:ffee:bea1::`)

**Replace PREFIXLEN with block prefix length** (e.g. `64`)

**Replace IF with target interface** (e.g. `eth0` or `wlan0`)

```console
# echo 'inet6 alias ADDR6 PREFIXLEN' >> /etc/hostname.IF
```

For interfaces using DHCP, a DHCP server can return unwanted fields that are
used by that interface without question (e.g. nameservers, domain-name,
domain-name-search). So it's often useful to configure some sane `supersede`
rules in dhclient.conf

**Replace DNS_ADDRESSES with comma-separated nameservers** (e.g. a
  known friendly nameserver and/or local Pi-Hole)

```console
# echo -e \
  'supersede domain-name-servers DNS_ADDRESSES;\
  \nsupersede domain-name ".";\
  \nsupersede domain-name-search ".";' > /etc/dhclient.conf
```

Restart network

*`/etc/netstart` doesn't have execute by default? Flip execute bit before and
 after restarting network*

**Replace IF with target interface** (e.g. `eth0` or `wlan0`)

```console
# chmod +x /etc/netstart
# /etc/netstart IF
# chmod -x /etc/netstart
```

## Configure filesystem

Filesystem options `noatime` and `softdep` can improve performance on older
hardware

```console
# vi /etc/fstab
```

For all local OpenBSD partitions (except swap) add `noatime` (if you don't
generally need file access times)

For all local OpenBSD user partitions (e.g. home) add `softdep` (for faster
file creation/deletion, but with somewhat greater cpu and memory overhead)

```
DISK_IDENTIFIER MOUNT_PATH ffs rw,softdep,noatime[,OTHER_OPTIONS] \
DUMP_FLAG FSCK_PASS_FLAG
```

<!--TODO /tmp mfs to keep temp in memory instead of disk partition-->

## Enable automatic locking on suspension

```console
# mkdir /etc/apm
# echo -e '#!/bin/sh\npkill -USR1 xidle' > /etc/apm/suspend
# chmod +x /etc/apm/suspend
```

## Change ntpd constraints

*Reading: `PAGER='less -p CONSTRAINTS' man ntpd`*

NTP constraints but without hitting google every time for them

**Replace CONSTRAINT_HOST with a trusted ip address or hostname**

```console
# sed -iorig 's/www\.google\.com/CONSTRAINT_HOST/' /etc/ntpd.conf
# rcctl restart ntpd
```

## Set up superuser

**Replace USER_NAME with the name of the non-root user account created during
  install**

Permit the non-root user to perform actions as root (with some time between
having to re-authenticate and while keeping environment)

```console
# echo 'permit persist keepenv USER_NAME as root' > /etc/doas.conf
```

Add the non-root user to the staff group

```console
# usermod -G staff USER_NAME
```

## Increase resource limits for staff

Adjust based on the machine's hardware

```console
# vi /etc/login.conf
```

```
staff:\
	:datasize-cur=1024M:\
	:datasize-max=8192M:\
	:maxproc-cur=512:\
	:maxproc-max=1024:\
	:ignorenologin:\
	:requirehome@:\
	:openfiles-cur=4096:\
	:openfiles-max=8192:\
	:stacksize-cur=32M:\
	:tc=default:
```

## Lid close behavior

For clamshell devices

| 0    | 1        | 2       |
| ---- | -------- | ------- |
| none | shutdown | suspend |

```console
# echo 'machdep.lidaction=0' >> /etc/sysctl.conf
```

## MOTD

For changing the message displayed after successful login

```console
# echo -e '\nI feel safer now\n' > /etc/motd
```

## System updates

```console
# syspatch
```

## Machine specific firmware

```console
# fw_update
```

## Reboot

```console
# shutdown -r now
```

Sign in as the non-root user upon restart

## Searching for packages

**Replace PACKAGE_NAME with a partial package name**

```console
$ pkg_info -Q PACKAGE_NAME
```

## Install packages

Some packages to start with

```console
$ doas pkg_add \
  htop \
  python \
  go \
  git \
  colorls \
  vim \
  mpd \
  ncmpcpp \
  mpv \
  ranger \
  feh \
  neomutt \
  w3m \
  rxvt-unicode \
  ibus-anthy \
  awesome \
  unclutter \
```

Choose a package when the base name is ambiguous

- python -> python-3.7.4
- vim -> vim-8.1.2061-gtk3
- neomutt -> neomutt-20180716p1-gpgme
- w3m -> w3m-0.5.3p8-image

## Language

```console
$ doas pkg_add \
  ja-fonts-gnu \
  ja-sazanami-ttf \
  noto-cjk \
```

## Other fonts

```console
$ doas pkg_add \
  comic-neue \
  font-awesome \
  inconsolata-font \
```

<!--TODO
Twemoji
CC Wild Words
Update font paths
-->

## Create user for cloned repos

```console
$ doas useradd -m repos
```

## Mounting portable drives

*Reading: `man disklabel`, `man mount`*

Find drive information

```console
$ sysctl hw.disknames
```

Detailed information about a storage device

**Replace DISK_NAME with the target disk** (e.g. `sd1`)

```console
$ doas disklabel DISK_NAME
```

Create a mount point for the partition and mount it

**Replace DISK_PARTITION with target disk partition entry in /dev** (e.g. 
`/dev/sd1i`)

```console
$ doas mkdir -p /mnt/pen
$ doas chown USER_NAME:USER_NAME /mnt/pen
$ doas mount DISK_PARTITION /mnt/pen
```

## Add personal and authorized keys (ssh, gpg, etc)

From flash drive or local net backups

## Some of my stuff

```console
$ doas mkdir -p /home/repos/github/lngtth
$ cd /home/repos/github/lngtth
$ doas git clone https://github.com/lngtth/dotfiles.git
```

Add custom style for xenodm

```console
$ doas /home/repos/github/lngtth/dotfiles/plain_xenodm.sh
```

Add dotfiles (run as the user to install them for)

```console
$ /home/repos/github/lngtth/dotfiles/install.sh
```

## Enable and start xenodm

```console
$ doas rcctl enable xenodm
$ doas rcctl start xenodm
```

Sign in to X through xenodm as the non-root user

## GUI configuration

Language input methods in ibus-setup

```console
$ ibus-setup
```

Add desired input methods from the `Input Method` tab by selecting them from
the dropdown menu and clicking `Add`
