# PREREQUISITES

This script requires jq and acl packages. Install them:

    apt install jq acl

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

This script automates backup of Proxmox VMs (both qemu and lxc) to a borg repository.

Storing backups in a borg repo is an excellent solution, because of its deduplication and
compression capabilities. Try it! :-)

VERY IMPORTANT:
THIS SCRIPT WORKS WITH *ZFS* ONLY!

Proxmox server must be configured to use ZFS. Proxborg will not work with another storage.
Proxmox server must be configured to use ZFS. Proxborg will not work with another storage.


(Yes, that was twice.)


# CONFIGURATION

Edit the script and fill in available variables, or provide them in environment.
At least you must set:

    export BORG_REPO='ssh://where/is/repo'

Possibly you will need also other borg configuration variables, like
BORG_PASSPHRASE, BORG_KEY_FILE, BORG_BASE_DIR or BORG_FILES_CACHE_TTL.
See there: https://borgbackup.readthedocs.io/en/stable/usage/general.html


# BASICS

There are 2 basic modes of operation implemented:
1. You may simply store normal VM images into a borg repo. They are created by
   the standard backup tool (vzdump) and thus are 100% compatible with Proxmox.
   In order to restore them you simply extract the image from the repo and
   provide it to Proxmox via gui or cli.
   This is the only way to store backups of qemu VMs, but works also with lxc.
   In this method the deduplication works, but is not perfect, because the images
   consist of unstructured data (at least they look so from borg point of view).
2. The better way: You may dump the whole lxc directory tree into borg repository.
   Deduplication with this method is superior, because borg can track every
   single file from the container filesystem.
   Restoring the container is almost as easy as restoring a normal Proxmox backup
   (see below).
3. Extra: You may also archive any mounted zfs dataset to a borg repo.
   For example you may want to backup the hypervisor root filesystem
   or particular mount points of containers.


# USAGE:

    proxborg  [@]vmid|vmhostname[:[!]] | /local/path   [...]



#
# HOWTO?
#

#  How to backup standard vzdump image of a VM? (whatever - qemu or lxc)

    proxborg 100 tank @pool

Meaning:
  - backup vm #100
  - backup vm named "tank".
    Note: The hostname must be unique! If more than one VM with the given name exist, none will be archived.
  - backup all VMs belonging to pool (note: pool names are case sensitive).



# How to restore a qemu image?

    borg extract --stdout repo::archivename | qmrestore - <new_id>

Alternatively you may extract the image from the borg repository and restore it
via Proxmox GUI.



# How to restore an lxc image?

Method 1:

    borg extract --stdout <repo>::<archive> | pct restore <new_id> - --rootfs rpool:<size> [...]

Note: when a container is restored via a pipe you must set the rootfs size manually.
Also all mountpoints must be set via commandline (--mp) manually. (If not - everything
will be extracted to rootfs.)

Example:

    borg extract --stdout ::test1-100-2020_01_02-03_04_05 - | pct restore 120 - --rootfs rpool:3 --mp0 rpool:8,mp=/var/www --mp1 rpool:15,mp=/srv/vmail --unprivileged 1


Method 2:

You may mount borg archive and read the image from there. This way mounts will be
restored automatically.

    borg mount <repo>::<archive> /mnt/tmp
    pct restore <new_id> /mnt/tmp/imagename.tar --storage rpool --unprivileged 1
    umount /mnt/tmp


Alternatively, you may extract the image from the borg repository and restore it
via Proxmox GUI.



#
# HERE IS THE MAGIC:
#

# How to backup lxc container THE BETTER WAY?

Borg's deduplication is magic. But there is a problem: whenever you create a new
vzdump-compatible image - it is different than the last one. This makes borg's
job harder and reduces its deduplication success rate. That's why it's a good
idea to provide borg with the real filesystem to be archived. This way borg can
use metadata (like file size, ctime) to find more deduplicatable data.
(And this is real magic!)

This script can automatically dump a container root filesystem and all mount
points directly into borg repository. The end result is 1:1 image of whole
container directory tree. The container config is also archived.

By the way: Did you know that separate archives in the same borg repository
are all deduplicated together? (I told you - magic!)
Imagine you have 20 containers with identical Debian base systems. Each base
system takes 1GB. You make 20 standard vzdump backups. Each backup takes about
0.5GB after compression. So all backups take 20x0.5=10GB total.
Now you switch to Borg: instead of keeping separate dumps - you put them
together into one repository. Hocus-pocus-deduplication: The repository with
backups of all 20 containers takes 0.5GB total!


Usage:

    proxborg vmid:[!]
                 ^ note colon here :-)

Where vmid is the numerical id of the container or its hostname. (Note: Hostname must be
unique! If more than one VM with the given name exist, none will be archived.)

"!" means: dump all mount points, *including* the ones without "backup=1" flag in config.


Example:

    proxborg tank:! 100: @pool:

Meaning:
Archive the following data:
- container named "tank", including all mountpoints
- contaner 100, excluding mountpoints without "backup=1" flag
  (This is how vzdump usually does its job.)
- all VMs belonging to given pool (note: Pool names are case sensitive!)

Note: Qemu VMs are always archived "the standard way" (as vzdump images), so
"proxborg 100" and "proxborg 100:" mean exactly the same (if VM 100 is a qemu VM).



# How to restore lxc container from borg repository?

It's really easy:

    borg export-tar <repo>::<archive> - | pct restore <new_id> - --rootfs <storagepool>:<size> --mp[n] <storagepool>:<size>,mp=<mountpoint> [--unprivileged 1 ...]

Example:

    borg export-tar ::test1-100-2020_01_02-03_04_05 - | pct restore 120 - --rootfs rpool:3 --mp0 rpool:8,mp=/mnt/abc --mp1 rpool:8,mp=/mnt/xyz --unprivileged 1

Make sure to set up mountpoints correctly! Use this command to see the
necessary mountpoints configuration:

    borg extract --stdout <repo>::<archivename> etc/vzdump/pct.conf

Read about ACLs below!


# What to do when restore fails?

Well... It shouldn't fail. :-)
Content of "borg export-tar" is very similar to normal effect of vzdump,
so most likely everything will work very well. But if not...
First of all - remember: you have a full image of all files belonging to the
container, as well as a copy of the old config. This is actually everything you
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

# What about ACLs?

THIS IS IMPORTANT if you have any mountpoint with ACLs enabled.
You must read and UNDERSTAND this:

The ACLs are correctly stored inside Borg repository (this is good).
The "borg export-tar" command does not extract ACLs (this is bad).

So in order to restore the ACLs you must manually extract the archive
(or at least the affected subfolders) using "borg extract".

I suggest you do something like this:

    borg export-tar --exclude path/with/acls/ <repo>::<archive> - | pct restore [options as described above]

Now create missing mountpoints (for example via Proxmox GUI) and extract missing subfolders from archive.
Let's assume folder /srv/sambashare is to be restored. Do something like this:

    cd /tmp
    mkdir temporary
    mount -t tmpfs -o posixacl tmpfs temporary
    cd temporary
    mkdir sambashare
    # Note: the above folder name ("sambashare") MUST match the lowest-level folder name to be restored.
    mount --bind /rpool/data/path/to/container/mountpoint/with/acls/ sambashare
    borg extract --strip-components 1 <repo>::<archivename> srv/sambashare
    #                                                                     ^ no slash here!
    #                               ^ here the depth of "sambashare" (number of slashes in "srv/sambashare")
    umount sambashare
    cd ..
    umount temporary
    cd ..
    rm -rf temporary

The above procedure is so complicated in order to restore correctly the
ACLs of mountpoint root.  If it is not important you may simplify it to:

    cd /rpool/data/path/to/container/mountpoint/with/acls/
    borg extract --strip-components 2 <repo>::<archivename> srv/sambashare/
    # Note slash:                   ^                                     ^


If the container is unprivileged - the uid/gid shift must be considered.
You may use for example shiftfs or fuse-overlayfs. It might be easier
to simply restore the container as privileged, then backup via Proxmox GUI and
restore again as unprivileged. :-)



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

  Because of deduplication. :-)

- Is this script safe?

  Well... It should be. But do your own tests.
  Remember: any untested backup solution is worthless!

- Is it as safe as standard Proxmox backup tools?

  Of course it's not... but should work. Actually, the result of "borg export-tar"
  is equivalent to vzdump. You should notice no difference as long as you use stdin
  as restore source (as described above).

- Will it work with non-ZFS storage?

  NO! It was designed to backup data stored on ZFS. It will also not work if you have mixed
  zfs and non-zfs storage in one container.
  "The standard way" (full VM images) might still work, but it has never been tested.



# Hints:

[Compression]

Do first backup with high compression level (like "auto,lzma" or "auto,zstd,9").
It will take plenty of time (especially lzma is very slow), but thanks to deduplication
never-changing data will stay in your repository forever with higher compression.
Important: If you make first archives with lower compression and later want to
switch to higher - create new repository! Otherwise the low-compressed data will stay
in the repo forever. It's a feature of the deduplication: it writes only new data
to repository, and compression is done on writes. Whatever has already been
written stays in repository as-is.

[Qemu images]

It is generally a good idea to fill virtual disks with zeros from time to time.
This will effect in smaller images, as zeros are compressible very well. :-)
In Windows guests you may use sdelete.exe -z.
In Linux consider using sfill.

[Borg cache]

In order to speed up backup process set BORG_FILES_CACHE_TTL accordingly.
Generally it is good idea to set it to at least 2-3 times more than the number
of VMs you are going store in repository.
Read this: https://borgbackup.readthedocs.io/en/stable/faq.html#it-always-chunks-all-my-files-even-unchanged-ones

[ACLs]

Consider setting "backup=0" on all mountpoints with ACLs enabled and store
these mountpoints in separate archives (might be in the same repository).
This will make your life easier on restore.
Note that separate backups of particular mountpoints will be done on different
times, so your backup will not be "atomic". Depending on circumstances this
might be bad or not-so-bad. :-)
