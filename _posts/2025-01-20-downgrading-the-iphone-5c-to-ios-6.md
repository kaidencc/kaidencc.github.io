---
layout: post
title: Downgrading the iPhone 5C to iOS 6
date: 2025-02-04 07:30 +1300
description: A guide for tether downgrading the iPhone 5C to iOS 6
---
## Disclaimer
I am **not** responsible for any damage to your devices caused by following this guide. Please proceed with caution and at your own risk.<br>

## Credits
- [NyanSatan](https://x.com/nyan_satan) for the 32-Bit iOS Dualboot guide and fixkeybag
- [throwaway](https://x.com/throwaway167074) for telling me how they booted iOS 6 on the iPhone 5C
- [libimobiledevice](https://github.com/libimobiledevice) for irecovery
- [LukeZGD](https://github.com/LukeZGD) for Legacy iOS Kit
- [dora2ios](https://x.com/dora2ios) for ipwnder_lite, xpwn (*Note: This is a fork of multiple forks, go to the repository to see who made the original and other forks*) and iBoot32Patcher (*Note: Original by [iH8sn0w](https://x.com/ih8sn0w)*)
- [Darwin on ARM Project](https://github.com/darwin-on-arm) for image3maker

## Note
When I use angle brackets (`< >`), they indicate placeholders. Do not include the brackets themselves in your input. For instance, `<enter>` means press the Enter key, and `<default value - 4>` means you should input the default value minus 4.<br>

## Requirements
- **A macOS system**, you can also do this on Linux, but this guide will be focused on macOS<br>
- [IDA Pro](https://hex-rays.com/ida-pro) for patching the kernelcache<br>
- **An iPhone 5 6.x iPSW, and an iPhone 5C 7.0 iPSW**, you can get these from [The Apple Wiki](https://theapplewiki.com/wiki/Firmware)<br>
- [gnu-tar](https://formulae.brew.sh/formula/gnu-tar) to compress the RootFS<br>
- [fixkeybag](/assets/iOS6-5C/fixkeybag) for generating the system keybag
- [irecovery](https://formulae.brew.sh/formula/libirecovery) to send bootchain components<br>
- [Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit) for the SSH ramdisk to install the iOS 6 RootFS on to the device<br>
- [ipwnder_lite](https://github.com/dora2ios/ipwnder_lite) to put the device in pwndfu mode
- [image3maker](https://github.com/darwin-on-arm/image3maker) to repack images into an img3 container
- [iBoot32Patcher](https://github.com/iH8sn0w/iBoot32Patcher) to patch iBoot components<br>
- [xpwn](https://github.com/Kaiden-AC/xpwn) for **xpwntool** and **dmg**, we will use **xpwntool** to decrypt and repack some firmware components, and **dmg** to decrypt the RootFS<br>

## Preparations
First decrypt the RootFS DMG from your iPhone 5 6.x iPSW, you can get firmware keys and file names from [The Apple Wiki](https://theapplewiki.com/wiki/Firmware)<br>
`dmg extract encrypted.dmg extract.dmg -k <key>`<br>

Then convert it to UDZO format<br>
`dmg build extract.dmg udzo.dmg`<br>

Mount the DMG, take note of the mount point<br>
`hdiutil attach udzo.dmg`<br>

Enable ownership on the volume<br>
`sudo diskutil enableOwnership <mountpoint>`<br>

Create a tar from the volume<br>
`sudo gtar -cvf fw.tar -C <mountpoint> .`<br>

## Partitioning
First, we need to boot the SSH ramdisk, enter DFU mode on your device and run Legacy iOS Kit<br>
`./restore.sh`<br>

Then navigate to **Other Utilities > SSH Ramdisk** and enter **11A470a** for the build number, follow the steps to boot the ramdisk, then select `Connect to SSH`<br>

Now once we are in the ramdisk, we need to partition the disk<br>
`gptfdisk /dev/rdisk0s1`<br>

Delete the existing partitions<br>
`d <enter> 1 <enter> d <enter>`<br>

Now create the new partitions<br>
`n <enter> 1 <enter> <enter> 524294 <enter> <enter>`<br>
`n <enter> <enter> <default value - 4> <enter> <enter>`<br>

Rename the new partitions<br>
`c <enter> 1 <enter> System <enter>`<br>
`c <enter> 2 <enter> Data <enter>`<br>

Write the new partition table<br>
`w <enter> Y <enter>`<br>

Now we need to create filesystems<br>
`/sbin/newfs_hfs -s -v System -J -b 4096 -n a=4096,c=4096,e=4096 /dev/disk0s1s1`<br>
`/sbin/newfs_hfs -s -v Data -J -P -b 4096 -n a=4096,c=4096,e=4096 /dev/disk0s1s2`<br>

## Extracting RootFS
Mount the new partitions<br>
`mount_hfs /dev/disk0s1s1 /mnt1`<br>
`mount_hfs /dev/disk0s1s2 /mnt2`<br>

On macOS open another Terminal window and extract the RootFS tar over SSH<br>
`cat fw.tar | ssh -p 6414 -oHostKeyAlgorithms=+ssh-dss root@localhost "cd /mnt1; tar xvf -"`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

After that completes, we need to move files to the Data partition, back on your device run<br>
`mv -v /mnt1/private/var/* /mnt2`<br>

We need to edit fstab to use the new partitions, back on macOS, create a new fstab file<br>
`nano fstab`<br>

It's contents should be the following:<br>
```
/dev/disk0s1s1 / hfs ro 0 1
/dev/disk0s1s2 /private/var hfs rw,nosuid,nodev 0 2
```

Send it to the device<br>
`scp -P 6414 -oHostKeyAlgorithms=+ssh-dss fstab root@localhost:/mnt1/private/etc`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

Now we need to install fixkeybag<br>
`scp -P 6414 -oHostKeyAlgorithms=+ssh-dss fixkeybag root@localhost:/mnt1`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

Now create launchd.conf and set executable permissions<br>
`nano launchd.conf`<br>

And enter the following contents<br>
`bsexec .. /fixkeybag`<br>

Send it to your device<br>
`scp -P 6414 -oHostKeyAlgorithms=+ssh-dss launchd.conf root@localhost:/mnt1/private/etc`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

Now back on the device, set UNIX permissions to 755<br>
`chmod 755 /mnt1/fixkeybag`<br>

Unmount both partitions and reboot the device<br>
`umount /mnt1 /mnt2`<br>
`reboot_bak`<br>

## Patching boot components

**Note: I will not patch boot files for you, please do not contact me for this**<br>

First decrypt iBSS and iBEC from your iPhone 5C 7.0 iPSW<br>
`xpwntool iBSS.boardconfig.RELEASE.dfu iBSS.raw -iv <iv> -k <key>`<br>
`xpwntool iBEC.boardconfig.RELEASE.dfu iBEC.raw -iv <iv> -k <key>`<br>

Patch the iBSS and iBEC<br>
`iBoot32Patcher iBSS.raw iBSS.patched --rsa`<br>
`iBoot32Patcher iBEC.raw iBEC.patched --rsa -b "-v amfi=0xff cs_enforcement_disable=1"`<br>

Pack the iBSS and iBEC into an img3 container<br>
`image3maker -f iBSS.patched -t ibss -o iBSS.img3`<br>
`image3maker -f iBEC.patched -t ibec -o iBEC.img3`<br>

Decrypt the DeviceTree from your iPhone 5 6.x iPSW<br>
`xpwntool DeviceTree.boardconfig.img3 devicetree.img3 -iv <iv> -k <key> –decrypt`<br>

Decrypt the kernelcache from your iPhone 5 6.x iPSW<br>
`xpwntool kernelcache.release.boardconfig kernelcache.dec -iv <iv> -k <key> –decrypt`<br>

Decompress the kernelcache from your iPhone 5 6.x iPSW<br>
`xpwntool kernelcache.release.boardconfig kernelcache.raw -iv <iv> -k <key>`<br>

Open your decompressed kernelcache in IDA Pro, make sure your settings are the same as below when opening it<br>

![IDA Pro settings for kernelcache](/assets/iOS6-5C/kernelcache-settings-ida.png)<br>

*Note: If you get any extra windows just click **OK***

Once the file is open, navigate to **Edit > Select all** in the toolbar, then press **C**, then click **Analyze**, this may take up to an hour<br>
*Note: If it asks "Undefine already existing code/data?" click Yes*<br>

Once the kernelcache is fully analyzed, navigate to **Search > Text...**<br>
Now search for "could not find system ID"<br>

Once the search is finished, you should see something like this<br>

![IDA Pro could not find system ID function](/assets/iOS6-5C/could-not-find-system-id-function-ida.png)<br>

Place your cursor just before `BL` and switch to hex view<br>

![IDA Pro BL hex 1](/assets/iOS6-5C/bl-hex-1-ida.png)<br>

Press **F2** and type `00BF00BF` and press **F2** again, this should replace the highlighted 4 bytes with `00BF00BF`<br>

![IDA Pro NOP hex 1](/assets/iOS6-5C/nop-hex-1-ida.png)<br>

Now switch back to IDA view and navigate to **Search > Text...** again, this time searching for "XIP is still set"<br>

Once the search has finished, you should see something like this<br>

![IDA Pro XIP is still set function](/assets/iOS6-5C/xip-is-still-set-function-ida.png)<br>

Place your cursor just before `BL` and switch to hex view<br>

![IDA Pro BL hex 2](/assets/iOS6-5C/bl-hex-2-ida.png)<br>

Press **F2** and type `00BF00BF` and press **F2** again, this should replace the highlighted 4 bytes with `00BF00BF`<br>

![IDA Pro NOP hex 2](/assets/iOS6-5C/nop-hex-2-ida.png)<br>

Now switch back to IDA view and navigate to **Edit > Patch program > Apply patches to input file...**<br>
Leave default settings and press **OK**<br>

Now repack the kernelcache<br>
`xpwntool kernelcache.raw kernelcache.img3 -t kernelcache.dec`<br>

## Booting the device

Put the device in pwndfu mode<br>
`ipwnder_macosx`<br>

Send iBSS<br>
`irecovery -f iBSS.img3`<br>

Send iBEC<br>
`irecovery -f iBEC.img3`<br>

Send DeviceTree<br>
`irecovery -f devicetree.img3`<br>

Execute DeviceTree<br>
`irecovery -c devicetree`<br>

Send kernelcache<br>
`irecovery -f kernelcache.img3`<br>

Boot the device<br>
`irecovery -c bootx`<br>

**Done!**
