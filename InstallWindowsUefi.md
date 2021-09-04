1. Create the Windows Installation Media
2. During Installation, delete all partition on the disk which will be dedicated to Windows (except from EFI?) until only unallocated space remains on that disk
3. Install Windows in the unallocated space (if EFI is only on the other disk it will use it with Ubuntu)
4. After the installation, log again on Ubuntu (with UEFI it should be possible to select it from the BIOS), re-install GRUB to /boot/efi and do ```sudo update-grub```
5. From now on default bootloader is GRUB and Windows can be accessed from there
