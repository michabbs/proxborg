# PREREQUISITES

This script requires jq binary. Install it:

    apt install jq

This script requires fuse-overlayfs. Version at least 1.0 is *recommended*.
Unfortunately at the time of writing the newest version provided by Proxmox
(or actually by Debian Buster) is 0.3. You may try it (apt install fuse-overlayfs),
but better simply download newer package from Debian Bullseye. Look here:
https://packages.debian.org/bullseye/fuse-overlayfs for available version.

    Example:
    wget http://ftp.us.debian.org/debian/pool/main/f/fuse-overlayfs/fuse-overlayfs_1.0.0-1_amd64.deb
    dpkg -i fuse-overlayfs_1.0.0-1_amd64.deb

This script should be run as root.



# GENERAL INFO

This script automates backup of Proxmox vm's (both qemu and lxc) to borg repository.
Storing backups in borg repo is excellent solution, because of its deduplication and
compression capabilities. Try it! :-)

VERY IMPORTANT:
THIS SCRIPT WORKS WITH *ZFS* ONLY!

Proxmox server must be configured to use ZFS. Proxborg will not work with another storage.
Proxmox server must be configured to use ZFS. Proxborg will not work with another storage.


(Yes, that was twice.)


# CONFIGURATION

Edit script and fill available variables, or provide them in environment.
At least you must set:

    export BORG_REPO='ssh://where/is/repo'

Possibly you will need also another borg configuration variables, like
BORG_PASSPHRASE, BORG_KEY_FILE, BORG_BASE_DIR or BORG_FILES_CACHE_TTL.
See more: https://borgbackup.readthedocs.io/en/stable/usage/general.html


# BASICS

There are 2 basic modes of operation implemented:
1. You may simply store normal vm images in borg repo. They are created by
   standard backup tool (vzdump) and thus are 100% compatible with Proxmox.
   In order to restore them - you simply extract the image from repo and
   provide it to Proxmox via gui or cli.
   This is the only way to store backups of qemu vm's, but works also with lxc.
   In this method the deduplication works, but is not perfect, because the images
   consist of unstructured data (at least - they look so from borg point of view).
2. The better way: You may dump whole lxc directory tree into borg repository.
   Deduplication in this method is superior, because borg can track every
   single file from the container filesystem.
   Restoring the container is almost as easy as restoring normal Proxmox backup
   (see below).
3. Extra: You may also archive any mounted zfs dataset to borg repo.
   For example you may want to backup the hypervisor root filesystem
   or particular mount points of containers.


# USAGE:

    proxborg  [@]vmid|vmhostname[:[!]] | /local/path   [...]



#
# HOWTO?
#

#  How to backup standard vzdump image of vm? (whatever - qemu or lxc)

    proxborg 100 tank @pool

Meaning:
  - backup vm #100
  - backup vm named "tank".
    Note: The hostname must be unique! If more than one vm with the given name exist - none will be archived.
  - backup all vm's belonging to pool (Note: pool names are case sensitive!)



# How to restore qemu image?

    borg extract --stdout repo::archivename | qmrestore - <new_id>

Alternatively you may download the image from borg repository and restore it
via Proxmox GUI.



# How to restore lxc image?

Method 1:

    borg extract --stdout <repo>::<archive> | pct restore <new_id> - --rootfs rpool:<size> [...]

Note: when container is restored from pipe you must set rootfs size manually.
Also all mountpoints must be set in commandline (--mp) manually. (If not - everything
will be extracted to rootfs.)

Example:

    borg extract --stdout ::test1-100-2020_01_02-03_04_05 - | pct restore 120 - --rootfs rpool:3 --mp0 rpool:8,mp=/var/www --mp1 rpool:15,mp=/srv/vmail --unprivileged 1


Method 2:

You may mount borg archive and read image from there. This way mounts will be
restored automatically.

    borg mount <repo>::<archive> /mnt/tmp
    pct restore <new_id> /mnt/tmp/imagename.tar --storage rpool --unprivileged 1
    umount /mnt/tmp


Alternatively you may download the image from borg repository and restore it
via Proxmox GUI.



#
# HERE IS THE MAGIC:
#

# How to backup lxc container THE BETTER WAY?

Borg's deduplication is magic. But there is a problem: whenever you create new
vzdump-compatible image - it is different than the last one. This makes borg's
job harder and reduces its deduplication success rate. That's why it's good
idea to provide borg with real filesystem to be archived. This way borg can
use metadata (like file size, ctime) to find more deduplicatable data.
(And this is real magic!)

This script can automatically dump container root filesystem and all mount
points directly into borg repository. The end result is 1-1 image of whole
container directory tree. Container config is also archived.

By the way: Did you know that separate archives in the same borg repository
are all deduplicated together? (I told you - magic!)
Imagine you have 20 containers with identical Debian base systems. Each base
system takes 1GB. You make 20 standard vzdump backups. Each backup takes about
0.5GB after compression. So all backups take 20x0.5=10GB total.
Now you switch to Borg: instead of keeping separate dumps - you put them
together to one repository. Hocus-pocus-deduplication: The repository with
backups of all 20 containers takes 0.5GB total!


Usage:

    proxborg vmid:[!]
                 ^ note colon here :-)

Where vmid is numerical id of container or its hostname. (Note: Hostname must be
unique! If more than one vm with the given name exist - none will be archived.)

"!" means: dump all mount points, *including* the ones without "backup=1" flag in config.


Example:

    proxborg tank:! 100: @pool:

Meaning:
Archive the following data:
- container named "tank", including all mountpoints
- contaner 100, excluding mountpoints without "backup=1" flag
  (This is how vzdump usually does its job.)
- all vm's belonging to given pool (note: Pool names are case sensitive!)

Note: Qemu vm's are always archived "the standard way" (as vzdump images), so
"proxborg 100" and "proxborg 100:" mean exactly the same (if vm 100 is qemu vm).



# How to restore lxc container from borg repository?

It's really easy:

    borg export-tar <repo>::<archive> - | pct restore <new_id> - --rootfs <storagepool>:<size> --mp[n] <storagepool>:<size>,mp=<mountpoint> [--unprivileged 1 ...]

Example:

    borg export-tar ::test1-100-2020_01_02-03_04_05 - | pct restore 120 - --rootfs rpool:3 --mp0 rpool:8,mp=/mnt/abc --mp1 rpool:8,mp=/mnt/xyz --unprivileged 1


Alternatively you may write the tarball to disk and import in Proxmox GUI.

    borg export-tar <repo>::<archive> - > <imagename>.tar

You may also compress it on the fly, which is actually useless if you are going
to restore the container immediately afterwards, but might be necessary if you
have not enough disk space:

    borg export-tar <repo>::<archive> - | gzip > <imagename>.tar.gz



# What to do when restore fails?

Well... It shouldn't fail. :-)
Result of "borg export-tar" is basically identical to normal effect of vzdump,
so most likely everything will work very well. But if not...
First of all - remember: you have full image of all files belonging to the
container, as well as copy of old config. This is actually everything you
need in order to revive it!

Read the old config:

    borg extract --stdout <repo>::<archivename> etc/vzdump/pct.conf

Using these information - create new identical container and restore files from archive.
More or less something like this:

    cd /rpool/data/path/to/container/rootfs
    rm -rf *
    borg extract <repo>::<archivename>

Consider mountpoints and permissions. Restoring as privileged container should be easier.
Good luck! :-)



# Extra: How to backup an arbitrary zfs dataset?

    proxborg /path/to/dataset/you/want/to/backup

This way you may backup for example root filesystem of the hypervisor or a particular
mount point of a container.

Important: Please note that file permissions are stored as-seen-by-hypervisor.
It does not matter for privileged containers, but is important for mountpoints
of unprivileged ones.

Even more important: Backup is NOT recursive. Only ONE dataset will be dumped.
If you want to archive children datasets too - archive them separately!



# Q&A:

- Why anyone would use Borg to store backups?

  Because deduplication. :-)

- Is this script safe?

  Well... It should be. But do your own tests.
  Remember: any untested backup solution is worthless!

- Is it as safe as standard Proxmox backup tools?

  Of course it's not... but should work. Actually result of "borg export-tar"
  is equivalent to vzdump. You should notice no difference.

- Will it work with non-ZFS storage?

  NO! It was designed to backup data stored on ZFS. It will also not work if you have mixed
  zfs and non-zfs storage in one container.
  "The standard way" (full vm images) might still work, but it has never been tested.



# Hints:

[Compression]

Do first backup with high compression level (like "auto,lzma" or even "auto,lzma,9").
It will take plenty of time, but thanks to deduplication never-changing data will
stay in your repository forever with higher compression.
Important: If you make first archives with lower compression and later want to
switch to higher - create new repository! Otherwise the low-compressed data will stay
in repo forever. It's a feature of the deduplication: it writes only new data
to repository, and compression is done on writes. Whatever has already been
written stays in repository as-is.

[Qemu images]

It is generally good idea to fill virtual disks with zeros from time to time.
This will effect in smaller images, as zeros are compressible very well. :-)
In Windows guests you may use sdelete.exe -z.
In Linux consider using sfill.

[Borg cache]

In order to speed up backup process set BORG_FILES_CACHE_TTL accordingly.
Generally it is good idea to set it to at least 2-3 times more than the number
of vm's you are going store in repository.
Read this: https://borgbackup.readthedocs.io/en/stable/faq.html#it-always-chunks-all-my-files-even-unchanged-ones
