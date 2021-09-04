# Setting up QEMU
## Create image and install operating system

1. First of all it is necessary to create the image where the guest OS will be installed; this can be done with a command such as:
   ```bash
   qemu-img create -f qcow2 <hda_img> <size>
   ```
   which will create the empty `.qcow2` image `<hda_img>` with the specified size. The size can be expressed in human-readable terms, e.g. "40G" to request a size of 40 gigabytes.

2. Assuming that the installation image `<install_image>` is already available, run `qemu` and request it to boot from the installation image with the following command:
   ```bash
   qemu-system-<arch> -hda <hda_img> -enable-kvm -boot d -cdrom <install_image> -m <memory_size> -k <layout>
   ```
   where:  
   + `<arch>` is the system architecture of the VM
   + `-enable-kvm` requests QEMU to leverage the KVM virtualization technology in order to improve the performance of the VM by allowing the Linux Kernel to act as a hypervisor.
   + `-boot d` means that the system will boot from the image specified by the `-cdrom` option
   + `<memory>` is the amount of physical memory allocated to the VM  
   + `-k <layout>` should in theory set the keyboard layout to the specified one (e.g. the italian one if layout="it"); in practice **it does not seem to be working**, so I add this just for reference

   After boot, install the operating system on <hda_img>, then turn off the VM.

3. Now the VM could already be started (remember to remove the `-boot` and `-cdrom` options from the command above), but in order to have better user experience there is still a bit that can be done.

## Automatic resolution switching and clipboard sharing (with SPICE)
With the basic setup described above I had quite a few issues, especially when running Windows guests; these included having the VM stay at a fixed screen resolution not corresponding to the natural resolution of my physical screen and not being able to share the clipboard between host and guest.

Solving both issues was quite easy with Linux guests; for the first, I just had to add the option `-vga virtio` to the `qemu-system-<arch>` command line, whereas for the second I had to request QEMU to use SPICE by adding the following options:  
```bash
-spice port=<spice_server_port>,disable-ticketing -device virtio-serial-pci \ 
-device virtserialport,chardev=spicechannel0,name=com.redhat.spice.0 \
-chardev spicevmc,id=spicechannel0,name=vdagent 
```
Everything worked directly out of the box, since the Linux guest I used (running Ubuntu 20.04) is suitable to this application by default.  
Things were only slightly more complicated with Windows guests; the main differences were two:
+ Rather than the `virtio` driver for the VGA card I had to select the `qxl` driver
+ I had to explicitly install the SPICE guest agent on the guest system with the Windows guest tools (downloadable at https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe). Notice that this also installs the QXL driver (which is also missing by default), thus solving both issues at once!  

I also needed to replace the default viewer used by QEMU with something supporting SPICE, such as `remote-viewer`. For this reason, I removed the default display by adding the option `-display none` to the `qemu-system-<arch>` command line, running instead the following command after launching the VM:
```bash
remote-viewer spice://localhost:<spice_server_port>
```

## Setting up shared folders with Virtio-FS
The last step was setting up a way to share folders between the host and the guest; I wanted to do this without the need of a network connection between the two (because why not!), which excluded common solutions like using `samba`. I did this by using the Virtio-FS shared filesystem; this required a few steps: 
1. Install `libseccomp-dev`
2. Clone the repository at https://gitlab.com/virtio-fs/qemu
3. Inside the root folder of the repository, run `./configure -disable-werror` (else it will refuse to proceed due to warnings), then run `make virtiofsd`. 
4. Before running the VM, run the following command from the root folder of the repository:
   ```bash
   sudo ./virtiofsd -o vhost_user_socket=<socket_file_path> -o source=<shared_folder_path> \
   -o cache=always
   ```
   Where <shared_folder_path> is the path (on the host) of the folder to be shared. Notice that `sudo` might not be necessary, I still need to try doing without.
5. Now it is possible to run the VM by adding the following options to the `qemu-system-<arch>` command line:
   ```bash
   -chardev socket,id=char0,path=<socket_file_path> \
   -device vhost-user-fs-pci,queue-size=1024,chardev=char0,tag=<mount_tag> \
   -object memory-backend-memfd,id=mem,size=<memory_size>,share=on -numa node,memdev=mem
   ```
   Where <mount_tag> is the tag that must be used by the guest when mounting the shared folder. 
 
At this point, Linux guests are almost ready; they only need to mount the shared folder with the following command:
```bash
mount -t virtiofs <mount_tag> <mount_location>
```

For Windows guests, it is still necessary to perform a few steps:  
1. Download the `<driver_image>` containing the virtio-win drivers from https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md  
2. Run the VM by adding the option `-cdrom <driver_image>` to the `qemu-system-<arch>` command line so that the drivers are accessible from within the VM.
3. From within the VM, download and install the Windows File System Proxy (WinFsp); the core installation should be enough.
4. From within the VM, access the device manager and install the Virtio FS driver from the image for the "mass storage" device.
5. Copy the `viofs` folder from the image to a different location on the guest's filesystem (such as the C:\ directory)
6. Run the `virtiofs.exe` file found in the `viofs/<guest_os>/<arch>` folder. This will enable the folder sharing service; the shared folder will be accessible as a new drive, typically identified by the letter Z  

Notice that it is also possible to create a service so that folder sharing is automatically enabled everytime the guest is started. This can be done by running the following command in a shell with elevated privileges:
```
sc create VirtioFsSvc binpath="(your binary location)\virtiofs.exe" start=auto depend="WinFsp.Launcher/VirtioFsDrv" DisplayName="Virtio FS Service"
```
Then the new service can be started manually with the following command:
```
sc start VirtioFsSvc
```
And this is it! Now we have a QEMU guest with automatic resolution switching, clipboard sharing **and** folder sharing too! Sweet, huh?
