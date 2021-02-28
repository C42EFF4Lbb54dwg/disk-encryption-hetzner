## disk-encryption-hetzner

This is the guide found at https://github.com/TheReal1604/disk-encryption-hetzner/blob/master/ubuntu/ubuntu_swraid_lvm_luks.md with a few changes as I experienced a lot of issues (pvresize, pvchange and dropbear issues.)

### Hardware setup
"Dedicated Root Server SB36"
- 2x HDD SATA 2,0 TB Enterprise (or any nvme drives with swraid)

### First steps in rescue image

- Boot to the rescue system via hetzners server management page
- install a minimal Ubuntu 18.04 LTS minimal with hetzners "installimage" skript (https://wiki.hetzner.de/index.php/Installimage)

- as this is for SWRAID 1 and Level 0, make sure to adjust the install script under software raid.

- before making the adjustments, scroll down to where it shows (commented out) Disk and the sizes (2000 GB (=> 1863 GiB). As we using Raid 0, we multiply this by two (as a note for below) which is 3726 GiB.

- I then chose the following logical volumes on my system to keep it simple:

```
PART /boot ext4 512M
PART lvm vg0 all

LV vg0 swap swap swap 8G
LV vg0 root / btrfs 3206G
```
- you will note I deducted the /boot and swap allocation from the 3726 GiB above to make an allocation rather than use all. I find that saying all will cause problems when running pvresize and mkfs later. 

- after you adjusted all parameters in the install config file, press F10 to install the ubuntu minimal system
-
- reboot and ssh into your fresh installed ubuntu

### First steps on your fresh ubuntu installation

- connect via ssh-key you choosed before for the rescue image. You might need to run ssh-keygen -R <serverIP> to deal with hostname issues.

- let's first update the system:

- `apt update && apt upgrade`

- Dont do a dist-upgrade unless you know what you are doing.
- Let's change the root password given by Hetzner:
- `passwd` - make sure to use strong and random password. Use a keymanager.

- install required packages

- `apt update && apt install busybox dropbear lvm2 vim cryptsetup cryptsetup-bin dropbear-initramfs`

-Edit your `/etc/initramfs-tools/initramfs.conf` and set `BUSYBOX=y`

- Create a new ssh key for unlocking your encrypted volumes when it is rebooting LOCALLY
- `ssh-keygen -t rsa -b 4096 -f .ssh/dropbear`
- note, I open the dropbear and dropbear.pub files and save copies of them on my secure PC. Just incase.
- Create the needed folders for dropbear keys
- `mkdir -p /etc/initramfs-tools/root/.ssh/`
- `cat .ssh/dropbear.pub >> /etc/dropbear-initramfs/authorized_keys`
- `cat .ssh/dropbear.pub >> ~/.ssh/authorized_keys`

- reboot again to the rescue system via the hetzner webinterface (You might need to run ssh-keygen -R <serverIP> to deal with hostname issues)

### Rescue image the second
 As we are using Raid Level 0, no initial replication is done and we can immediately proceed.

If you can not find anything in `/dev/mapper/*` you will have to activate the volumes first.
`lvm vgscan -v`

Activate all volume groups:
`lvm vgchange -a y`

We now rsync our installation into the new encrypted drives 

- `mkdir /oldroot/`
- `mount /dev/mapper/vg0-root /mnt/`
- `rsync -av /mnt/ /oldroot/` (this could take a while)
- `umount /mnt/`

Backup your old vg0 configuration to keep things simple and remove the old volume group:

- `vgcfgbackup vg0 -f vg0.freespace`
- `vgremove vg0` - select Y for all.

After this, we encrypt our raid 1 now.
- `cryptsetup --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time 6000 luksFormat /dev/md1`
(!!!Choose a strong passphrase (something like `pwgen 64 1`)!!!) - use bitwarden to generate a 128 character password and save it securely. 
- `cryptsetup luksOpen /dev/md1 cryptroot`
- now create the physical volume on your mapper:
- `pvcreate /dev/mapper/cryptroot`

We have now to edit your vg0 backup:
- `blkid /dev/mapper/cryptroot`
   Results in:  `/dev/mapper/cryptroot: UUID="HEZqC9-zqfG-HTFC-PK1b-Qd2I-YxVa-QJt7xQ"`
- `cp vg0.freespace /etc/lvm/backup/vg0`

Now edit the `id` (UUID from above) and `device` (/dev/mapper/cryptroot) properties nested at `vg0 > physical_volumes > pv0` in the file according to our installation
- `nano /etc/lvm/backup/vg0`
- Restore the vgconfig: `vgcfgrestore vg0`
- Resize PV to the new size: `pvresize /dev/mapper/cryptroot`
- `vgchange -a y vg0`

Ok, the filesystem is missing, lets create it:

- `mkfs.btrfs -f /dev/vg0/root`
- `mkswap /dev/vg0/swap`

Now we mount and copy our installation back on the new lvs:

- `mount /dev/vg0/root /mnt/`
- `mkdir /mnt/home /mnt/var /mnt/var/log`
- `rsync -av /oldroot/ /mnt/`

### Some changes to your existing linux installation
Lets mount some special filesystems for chroot usage:
- `mount /dev/md0 /mnt/boot`
- `mount --bind /dev /mnt/dev`
- `mount --bind /sys /mnt/sys`
- `mount --bind /proc /mnt/proc`
- `chroot /mnt`

To let the system know there is a new crypto device we need to edit the cryptab(/etc/crypttab):
- `nano /etc/crypttab`
- copy the following line in there: `cryptroot /dev/md1 none luks`

- `nano /etc/dropbear-initramfs/authorized_keys`
- alter your public key like this: `no-port-forwarding,no-agent-forwarding,no-X11-forwarding,command="/bin/cryptroot-unlock" ssh-rsa ...`

Regenerate the initramfs:
- `update-initramfs -u`
- `update-grub`
- `grub-install /dev/sda` (or `grub-install /dev/nvme0n1` if you use nvme)
- `grub-install /dev/sdb` (or `grub-install /dev/nvme1n1` if you use nvme)

Time for our first reboot.. fingers crossed!

- `exit`
- `umount /mnt/boot /mnt/proc /mnt/sys /mnt/dev`
- `umount /mnt`
- `sync`
- `reboot`

After a few seconds, try to connect to your server with the below, the dropbear ssh server is coming up on your system, connect to it and unlock your system like this:

- `ssh -i .ssh/dropbear root@<yourserverip>`
- a busybox shell should come up
- unlock your lvm drive with:
- `echo -ne "<yourstrongpassphrase>" > /lib/cryptsetup/passfifo`

## Optional steps
You can further secure dropbear by changing its port and disabling unnecessary features:
- `nano /etc/dropbear-initramfs/config`
- add the line `DROPBEAR_OPTIONS="-p 2222 -s -j -k -I 30"`
- `update-initramfs -u`

This makes dropbear to listen to port 2222 instead of 22, `-s` disables password logins, `-j -k` disables port forwarding, `-I 30` sets the idle timeout to 30 seconds.

Reboot you server and unlock your system using
- `ssh -p 2222 -i .ssh/dropbear root@<yourserverip>`


## Sources:
Special thanks to the people who wrote already this guides:

- http://notes.sudo.is/RemoteDiskEncryption
- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system
- https://hamy.io/post/0009/how-to-install-luks-encrypted-ubuntu-18.04.x-server-and-enable-remote-unlocking/

## Contribution

- PRs are very welcome or open an issue if something not works for you as described

## Comments
- tested on 28.02.2021
