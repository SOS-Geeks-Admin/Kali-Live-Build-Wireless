# Kali Linux Bootable USB with Persistence and Wireless on OSX

**Download the appropriate Kali Linux `.iso`**

* Download site: [https://www.kali.org/downloads/](https://www.kali.org/downloads/)

I used a 64 bit `.iso` image, downloaded via HTTP. I downloaded the amd64 weekly version, as the `pool` linux headers (needed below for installation of wireless drivers) were ahead of the stable release kernel.

Download the `SHA256SUMS` and `SHA256SUMS.gpg` files from the same location.

**Check the hash**

Check that the hashes were not tampered with. First, get the Kali GPG public key, and verify the fingerprint:

```bash
$ wget -q -O - https://www.kali.org/archive-key.asc | gpg --import
$ gpg --fingerprint 7D8D0BF6
pub   rsa4096 2012-03-05 [SC] [expires: 2018-02-02]
      44C6 513A 8E4F B3D3 0875  F758 ED44 4FF0 7D8D 0BF6
uid           [ unknown] Kali Linux Repository <devel@kali.org>
sub   rsa4096 2012-03-05 [E] [expires: 2018-02-02]
$ gpg --verify SHA256SUMS.gpg SHA256SUMS
gpg: Signature made Sun 12 Nov 03:47:29 2017 GMT
gpg:                using RSA key 44C6513A8E4FB3D30875F758ED444FF07D8D0BF6
gpg: Good signature from "Kali Linux Repository <devel@kali.org>" [unknown]
```
Compare the SHA256 hash with that reported in `SHASUMS`:

```bash
$ cat SHA256SUMS
16123b76a6d4fc3ed72aef508bee9542462f2d1d5376acd1fcc3369ad337a505  kali-linux-2017-W46-amd64.iso
$ shasum -a 256 kali-linux-2017-W46-amd64.iso
16123b76a6d4fc3ed72aef508bee9542462f2d1d5376acd1fcc3369ad337a505  kali-linux-2017-W46-amd64.iso
```

**Create the USB disk**

Identify your external USB with `diskutil` - the disk ID (`disk2`, `disk3` etc is represented as `<DISK>` below):

* `diskutil list`

If necessary, prep the external USB with `diskutil` to get a single partition:

* `diskutil eraseDisk FAT32 KALI /dev/<DISK>`

Unmount the volume in DIsk Utility, or at the command-line:

* `diskutil unmountDisk /dev/<DISK>`

Then use `dd` to make a bootable image on the USB:

* `sudo dd if=<path to downloaded .iso> of=/dev/<DISK> bs=1m`

**Boot into Kali Linux**

* Restart the Mac
* Hold down the Option key when you hear the chime
* Select `EFI` as the startup disk
* Select `Kali Linux (persistence)`

**Create a new persistent partition**

* Start `gparted` from the terminal
* Select the USB disk
* Select the `Unallocated` partition
* Create a new partition (by default this will fill the free space on the USB)
    * `Partition -> New`
    * Create as: `Primary Partition`
    * File system: `ext3`
    * Label: `persistence`
* Apply the operations
    * `Edit -> Apply All Operations`
    * Confirm this action
* Exit `gparted`

**Combine the new partition with Kali Linux, persistently**

Create a mount point for the persistence particion, and mount it

* `mkdir -p /mnt/my_usb`
* `mount <DISK> /mnt/my_usb`

Create a `partition.conf` file. This will control persistence on USB startup

* `echo "/ union" > /mnt/my_usb/persistence.conf`
* `umount <DISK>`

**Check the persistent partition**

* Restart the Mac
* Hold down the Option key when you hear the chime
* Select `EFI` as the startup disk
* Select `Kali Linux (persistence)`
* At the terminal:
    * `df -h` will bring up a list of mounted drives. There should be a mountpoint `/lib/live/mount/persistence/<DISK>` pointing to your new persistent partition
    * `ls -ltrh /lib/live/mount/persistence/<DISK>` should show four entries: `lost+found`, `persistence.conf`, `rw`, and `work`. The `rw` directory is a persistent link to `/`.

**Update the OS**

Update the installer and acquire the appropriate linux headers

```bash
$ apt-get update
$ apt-get install linux-image-$(uname -r|sed 's,[^-]*-[^-]*-,,')
$ apt-get install linux-headers-$(uname -r|sed 's,[^-]*-[^-]*-,,')
```

**Install kernel headers**

```bash
$ wget http://http.kali.org/kali/pool/main/l/linux/linux-kbuild-<VERSION>_amd64.deb
$ wget http://http.kali.org/kali/pool/main/l/linux/linux-headers-<VERSION>-common_<VERSION>_amd64.deb
$ wget http://http.kali.org/kali/pool/main/l/linux/linux-headers-<VERSION>-amd64_<VERSION>_amd64.deb
dpkg -i linux-kbuild-<VERSION>_amd64.deb
dpkg -i linux-headers-<VERSION>-common_<VERSION>_amd64.deb
dpkg -i linux-headers-<VERSION>-amd64_<VERSION>_amd64.deb
```

**Install the Broadcom drivers**

```bash
apt-get install broadcom-sta-dkms
```

**Enable and disable modules**

```bash
$ modprobe -r b44 b43 b43legacy ssb brcmsmac bcma
modprobe wl
```

**Enable network-manager**

```bash
nano /etc/NetworkManager/NetworkManager.conf
```

Set the value of `managed` to `true`,