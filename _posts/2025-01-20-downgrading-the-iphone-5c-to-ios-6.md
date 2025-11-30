---
layout: post
title: Downgrading the iPhone 5C to iOS 6
date: 2025-02-04 07:30 +1300
description: A guide for tether downgrading the iPhone 5C to iOS 6
---

> **Note:**  
> When you see angle brackets (`< >`), they indicate placeholders. **Do not** include the brackets themselves in your input. For instance, `<enter>` means press the Enter key, and `<default value - 4>` means you should input the default value minus 4.

---

## Disclaimer

I am **not** responsible for any damage to your devices caused by following this guide. Please proceed with caution and at your own risk.

## Credits

- [NyanSatan](https://x.com/nyan_satan) for the 32-Bit iOS Dualboot guide and fixkeybag  
- [throwaway](https://x.com/throwaway167074) for telling me how they booted iOS 6 on the iPhone 5C  
- [libimobiledevice](https://github.com/libimobiledevice) for irecovery  
- [LukeZGD](https://github.com/LukeZGD) for Legacy iOS Kit  
- [dora2ios](https://x.com/dora2ios) for ipwnder_lite, xpwn (*Note: This is a fork of multiple forks; go to the repository to see who made the original and other forks*) and iBoot32Patcher (*Note: Original by [iH8sn0w](https://x.com/ih8sn0w)*)  
- [Darwin on ARM Project](https://github.com/darwin-on-arm) for image3maker  

---

## Prerequisites

- **A macOS system**  
  *(You can also do this on Linux, but this guide will be focused on macOS.)*
- **[IDA Pro](https://hex-rays.com/ida-pro)** for patching the kernelcache  
- **An iPhone 5 6.x iPSW and an iPhone 5C 7.0 iPSW**  
  *(You can get these from [The Apple Wiki](https://theapplewiki.com/wiki/Firmware).)*
- **[gnu-tar](https://formulae.brew.sh/formula/gnu-tar)** to compress the RootFS  
- **[fixkeybag](/assets/iOS6-5C/fixkeybag)** for generating the system keybag  
- **[irecovery](https://formulae.brew.sh/formula/libirecovery)** to send bootchain components  
- **[Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit)** for the SSH ramdisk to install the iOS 6 RootFS on the device  
- **[ipwnder_lite](https://github.com/dora2ios/ipwnder_lite)** to put the device in pwndfu mode  
- **[image3maker](https://github.com/darwin-on-arm/image3maker)** to repack images into an img3 container  
- **[iBoot32Patcher](https://github.com/iH8sn0w/iBoot32Patcher)** to patch iBoot components  
- **[xpwn](https://github.com/Kaiden-AC/xpwn)** for **xpwntool** and **dmg**  
  *(We will use **xpwntool** to decrypt and repack some firmware components, and **dmg** to decrypt the RootFS.)*

---

## Preparations

1.  **Decrypt the RootFS DMG:**  
    From your iPhone 5 6.x iPSW, use `dmg` to extract the encrypted filesystem. (Get firmware keys and file names from [The Apple Wiki](https://theapplewiki.com/wiki/Firmware)).
    ```bash
    dmg extract encrypted.dmg extract.dmg -k <key>
    ```

2.  **Convert to UDZO format:**  
    ```bash
    dmg build extract.dmg udzo.dmg
    ```

3.  **Mount the DMG:**  
    Take note of the mount point.
    ```bash
    hdiutil attach udzo.dmg
    ```

4.  **Enable ownership:**  
    ```bash
    sudo diskutil enableOwnership <mountpoint>
    ```

5.  **Create a tar from the volume:**  
    ```bash
    sudo gtar -cvf fw.tar -C <mountpoint> .
    ```

---

## Partitioning

1.  **Boot the SSH Ramdisk:**  
    Enter DFU mode on your device and run Legacy iOS Kit:
    ```bash
    ./restore.sh
    ```
    Navigate to **Other Utilities > SSH Ramdisk** and enter **11A470a** for the build number. Follow the steps to boot the ramdisk, then select `Connect to SSH`.

2.  **Partition the disk:**  
    Once in the ramdisk, run the following:
    ```bash
    gptfdisk /dev/rdisk0s1
    ```

3.  **Delete existing partitions:**  
    ```bash
    d <enter> 1 <enter> d <enter>
    ```

4.  **Create new partitions:**  
    ```bash
    n <enter> 1 <enter> <enter> 524294 <enter> <enter>
    n <enter> <enter> <default value - 4> <enter> <enter>
    ```

5.  **Rename the partitions:**  
    ```bash
    c <enter> 1 <enter> System <enter>
    c <enter> 2 <enter> Data <enter>
    ```

6.  **Write the new partition table:**  
    ```bash
    w <enter> Y <enter>
    ```

7.  **Create filesystems:**  
    ```bash
    /sbin/newfs_hfs -s -v System -J -b 4096 -n a=4096,c=4096,e=4096 /dev/disk0s1s1
    /sbin/newfs_hfs -s -v Data -J -P -b 4096 -n a=4096,c=4096,e=4096 /dev/disk0s1s2
    ```

---

## Extracting RootFS

1.  **Mount the new partitions:**  
    ```bash
    mount_hfs /dev/disk0s1s1 /mnt1
    mount_hfs /dev/disk0s1s2 /mnt2
    ```

2.  **Extract the RootFS tar over SSH:**  
    On macOS, open another Terminal window and run:
    ```bash
    cat fw.tar | ssh -p 6414 -oHostKeyAlgorithms=+ssh-dss root@localhost "cd /mnt1; tar xvf -"
    ```
    *Note: When asked for a password, enter "alpine".*

3.  **Move files to the Data partition:**  
    Back on your device, run:
    ```bash
    mv -v /mnt1/private/var/* /mnt2
    ```

4.  **Edit fstab:**  
    Back on macOS, create a new fstab file to use the new partitions:
    ```bash
    nano fstab
    ```
    Paste the following content:
    ```
    /dev/disk0s1s1 / hfs ro 0 1
    /dev/disk0s1s2 /private/var hfs rw,nosuid,nodev 0 2
    ```

5.  **Send fstab to the device:**  
    ```bash
    scp -P 6414 -oHostKeyAlgorithms=+ssh-dss fstab root@localhost:/mnt1/private/etc
    ```
    *Note: When asked for a password, enter "alpine".*

6.  **Install fixkeybag:**  
    ```bash
    scp -P 6414 -oHostKeyAlgorithms=+ssh-dss fixkeybag root@localhost:/mnt1
    ```
    *Note: When asked for a password, enter "alpine".*

7.  **Configure launchd:**  
    Create `launchd.conf` on macOS:
    ```bash
    nano launchd.conf
    ```
    Enter the following contents:
    ```bash
    bsexec .. /fixkeybag
    ```
    Send it to your device:
    ```bash
    scp -P 6414 -oHostKeyAlgorithms=+ssh-dss launchd.conf root@localhost:/mnt1/private/etc
    ```
    *Note: When asked for a password, enter "alpine".*

8.  **Finalize permissions and reboot:**  
    Back on the device, run:
    ```bash
    chmod 755 /mnt1/fixkeybag
    umount /mnt1 /mnt2
    reboot_bak
    ```

---

## Patching Boot Components

> **Note:**  
> I will not patch boot files for you, please do not contact me for this.

### iBSS and iBEC

1.  **Decrypt iBSS and iBEC:**  
    Use your iPhone 5C 7.0 iPSW files:
    ```bash
    xpwntool iBSS.boardconfig.RELEASE.dfu iBSS.raw -iv <iv> -k <key>
    xpwntool iBEC.boardconfig.RELEASE.dfu iBEC.raw -iv <iv> -k <key>
    ```

2.  **Patch the files:**  
    ```bash
    iBoot32Patcher iBSS.raw iBSS.patched --rsa
    iBoot32Patcher iBEC.raw iBEC.patched --rsa -b "-v amfi=0xff cs_enforcement_disable=1"
    ```

3.  **Pack into img3 containers:**  
    ```bash
    image3maker -f iBSS.patched -t ibss -o iBSS.img3
    image3maker -f iBEC.patched -t ibec -o iBEC.img3
    ```

### DeviceTree

1.  **Decrypt DeviceTree:**  
    Use your iPhone 5 6.x iPSW file:
    ```bash
    xpwntool DeviceTree.boardconfig.img3 devicetree.img3 -iv <iv> -k <key> –decrypt
    ```

### Kernelcache

1.  **Decrypt and decompress the kernelcache:**  
    Use your iPhone 5 6.x iPSW file:
    ```bash
    xpwntool kernelcache.release.boardconfig kernelcache.dec -iv <iv> -k <key> –decrypt
    xpwntool kernelcache.release.boardconfig kernelcache.raw -iv <iv> -k <key>
    ```

2.  **Open in IDA Pro:**  
    Open your decompressed kernelcache in IDA Pro. Ensure your settings match the image below:

    ![IDA Pro settings for kernelcache](/assets/iOS6-5C/kernelcache-settings-ida.png)  
    *Note: If you get any extra windows just click **OK**.*

3.  **Analyze the file:**  
    Navigate to **Edit > Select all**, press **C**, then click **Analyze**.  
    *Note: This may take up to an hour. If it asks "Undefine already existing code/data?" click Yes.*

4.  **Patch "could not find system ID":**  
    - Navigate to **Search > Text...** and search for `could not find system ID`.
    - You should see the following function:
      
      ![IDA Pro could not find system ID function](/assets/iOS6-5C/could-not-find-system-id-function-ida.png)
      
    - Place your cursor just before `BL` and switch to hex view.
      
      ![IDA Pro BL hex 1](/assets/iOS6-5C/bl-hex-1-ida.png)
      
    - Press **F2**, type `00BF00BF`, and press **F2** again. This replaces the highlighted bytes with NOPs.
      
      ![IDA Pro NOP hex 1](/assets/iOS6-5C/nop-hex-1-ida.png)

5.  **Patch "XIP is still set":**  
    - Switch back to IDA view.
    - Navigate to **Search > Text...** and search for `XIP is still set`.
    - You should see the following function:
      
      ![IDA Pro XIP is still set function](/assets/iOS6-5C/xip-is-still-set-function-ida.png)
      
    - Place your cursor just before `BL` and switch to hex view.
      
      ![IDA Pro BL hex 2](/assets/iOS6-5C/bl-hex-2-ida.png)
      
    - Press **F2**, type `00BF00BF`, and press **F2** again.
      
      ![IDA Pro NOP hex 2](/assets/iOS6-5C/nop-hex-2-ida.png)

6.  **Apply patches and repack:**  
    - Switch back to IDA view and navigate to **Edit > Patch program > Apply patches to input file...**.
    - Leave default settings and press **OK**.
    - Repack the kernelcache:
      ```bash
      xpwntool kernelcache.raw kernelcache.img3 -t kernelcache.dec
      ```

---

## Booting the Device

1.  **Put the device in pwndfu mode:**  
    ```bash
    ipwnder_macosx
    ```

2.  **Send bootchain components:**
    ```bash
    irecovery -f iBSS.img3
    irecovery -f iBEC.img3
    irecovery -f devicetree.img3
    ```

3.  **Execute DeviceTree:**  
    ```bash
    irecovery -c devicetree
    ```

4.  **Send and boot Kernelcache:**  
    ```bash
    irecovery -f kernelcache.img3
    irecovery -c bootx
    ```

**Done!**