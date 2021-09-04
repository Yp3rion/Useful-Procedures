1. Boot on a live USB
2. In GParted, create a fat32 partition of about 512 MiB with boot/esp flags in the destination disk
3. Turn off swap with ```sudo swapoff /dev/sdXXXX```, where /dev/sdXXXX is the source swap partition
4. Copy the root and swap partitions from the source drive to the destination drive, then change their UUID from within GParted
5. Mount the root partition with ```sudo mount /dev/sdYYY /mnt```, where /dev/sdYYY is the destination root partition
6. Edit the /mnt/etc/fstab file by changing the UUIDs with the ones of the destination EFI, root and swap partitions
7. Mount the EFI partition with ```sudo mount /dev/sdYY /mnt/boot/efi```, where /dev/sdYY is the destination EFI partition
8. If necessary, create the directory /mnt/boot/efi/EFI
9. Mount /dev, /dev/pts, /sys, /proc, /run at /mnt$i
10. Chroot with ```sudo chroot /mnt```
11. Install grub in the /dev/sdYY partition, with ```grub-install --target=x86_64-efi /dev/sdY --efi-directory=/boot/efi```, where /dev/sdY is the destination drive and assuming that the architecture is x86_64
