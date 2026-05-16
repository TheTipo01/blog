+++
title = 'How to use ntfs2btrfs, convert to space_cache v2 and enable BTRFS compression'
date = 2026-05-16T14:54:43+02:00
draft = false
+++

As I still get questions from my friend on how to use ntfs2btrfs, I decided to condense everything in a blog post (because I hate repeating myself too many times).

# Installing ntfs2btrfs

This is probably the easy part: there's a package of it on just every major repo.

Just install it like so:
- Debian and Ubuntu: `sudo apt install ntfs2btrfs`
- Arch: `sudo pacman -Syu ntfs2btrfs`
- Fedora: `sudo dnf install ntfs2btrfs`

Or, if not available, download and compile it from the [repo](https://github.com/maharmstone/ntfs2btrfs).

## Using ntfs2btrfs

> A word of warning: while the script has worked fine for two of my disks without data losses: please, for the love of god, backup your truly important data. You've been warned.

While your disk is **unmounted** (you can do that from your file manager by right clicking the drive), just call the program like so `sudo ntfs2btrfs /dev/sda1`.

Obviously, you need to replace `/dev/sda1` with your correct drive: with a simple `lsblk`, you can print all your drives and their device name.

For example, this is the output from my PC:
```
❯ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   1,8T  0 disk 
└─sda1        8:1    0   1,8T  0 part /media/hdd
sdb           8:16   0 465,8G  0 disk 
└─sdb1        8:17   0 465,8G  0 part /media/ssd
zram0       253:0    0  15,5G  0 disk [SWAP]
nvme0n1     259:0    0 465,8G  0 disk 
├─nvme0n1p1 259:1    0     2G  0 part /boot
└─nvme0n1p2 259:2    0 463,8G  0 part /var/cache
                                      /var/log
                                      /var/tmp
                                      /srv
                                      /root
                                      /home
                                      /
```

Let's say i want to convert my 465G drive (my very slow SATA ssd), i would call ntfs2btrfs like so `sudo ntfs2btrfs /dev/sdb1`.

### Waiting
This conversion does take a *while*, and you will probably notice some warnings about files with extended permission not being supported: they are completely safe to ignore, as the data will get carried around, just without those extra permissions.


### Profit
Congrats, you now have converted your drive to BTRFS! But are we truly done?

## Space cache v2 & block group tree
This script is really nifty, but it doesn't enable space cache v2 and block group tree: in short, if you have a slow and big drive (for example, a 2TB HDD), without this it will take a lot of time to mount your drive. And the boot process could even fail, because timeouts (ask me how I know lol).

### How to convert to it
Firstly, I will assume you know how to mount your disk normally (see the [CachyOS wiki](https://wiki.cachyos.org/configuration/automount_with_fstab/#tldr-1) if you don't know how).

You will need to simply mount your drive with `space_cache=v2` as an additional parameter.

For example, for my HDD I would have to go from this
```
UUID=1554bd55-3b4f-f451-bcd3-b00f3f847e27 /media/hdd     btrfs   defaults,noatime,commit=120 0 0
```
to this
```
UUID=1554bd55-3b4f-f451-bcd3-b00f3f847e27 /media/hdd     btrfs   defaults,noatime,commit=120,space_cache=v2 0 0
```

Then just do `sudo systemctl daemon-reload` and `sudo mount -a`.

Now, unmount your drive again and run
```
sudo btrfstune --convert-to-block-group-tree /dev/sda1
```

> If you don't have btrfstune, you're probably missing the package btrfs-progs

Then, you're done and you can just `sudo mount -a` your drives again!

## Compression
BTRFS supports file compression: this is huge for things like your game drives, as some older games don't employ asset compression and compressing them will save you some space.

To enable it, just add `compress=zstd` to your mount parameters like so:

```
UUID=1554bd55-3b4f-f451-bcd3-b00f3f847e27 /media/hdd     btrfs   defaults,noatime,commit=120,space_cache=v2,compress=zstd 0 0
```
And then, again, remount your drive with `sudo systemctl daemon-reload` and `sudo mount -a`.

Congrats, you now have enabled compression for your written files from now on!

### Compressing already existing files
For your already existing files, you've got to use the defragment utility of btrfs while the drive is mounted.

Be warned that this process will take a lot of time, as it has to read and write each file that needs to be compressed.
```
sudo btrfs -v filesystem defragment -r -czstd /mnt/hdd
```

### Checking savings
There's `compsize` that accept as a parameter a folder or file on a BTRFS drive, and it will tell you it's disk usage and compression ratio.
Taking as an example my Payday 2 folder
```
❯ sudo compsize "/media/hdd/Giochi/Steam/steamapps/common/PAYDAY 2/"
Processed 13683 files, 663952 regular extents (663952 refs), 7252 inline.
Type       Perc     Disk Usage   Uncompressed Referenced  
TOTAL       61%       52G          85G          85G       
none       100%      6.1G         6.1G         6.1G       
zstd        59%       46G          79G          79G 
```
you can see that's quite the saving: 33G!

### Some notes about compression
The algorithm I used in this article, zstd, is really really fast: you shouldn't need to worry about perfomance losses too much.
But if you're really worried, you can switch `zstd` for `lzo`, as it's even more faster (but it achieve worst compression ratios).