[[Vicharak]]


# Creating a new user in the SD Card RootFS

 - Copying ARM chroot utility(?) Into the SD Card Rootfs - 
	 sudo cp /usr/bin/qemu-arm-static /mnt/root/usr/bin/
 - Chang the root directory for the current session into the mount partition 
	 sudo chroot /mnt/root /bin/bash
 -  Use the `adduser` command -
	 `adduser ronin`
 - Add user to the supermod group - 
	 usermod -aG sudo ronin
 - Exit -
	 `exit`
 - clean up the binaries -
	sudo rm /mnt/root/usr/bin/qemu-arm-static


# Desktop Env Install (From the Laptop, BEFORE BOOTING)

 - Install the QEMU Binaries - `sudo cp /usr/bin/qemu-arm-static /mnt/rootfs/usr/bin/~
 - Bind necessary system dirs - 
```
sudo mount --bind /dev /mnt/rootfs/dev
sudo mount --bind /sys /mnt/rootfs/sys
sudo mount --bind /proc /mnt/rootfs/proc
sudo cp /etc/resolv.conf /mnt/rootfs/etc/resolv.conf
```
 - Chroot into the rfs - 
`	 sudo chroot /mnt/rootfs`
 - Update & Install Packages 
```
apt update
apt install -y xserver-xorg lightdm lxde-core
```

 - Exit and Clean Up
	`exit`
	`sudo umount /mnt/rootfs/dev`
	`sudo umount /mnt/rootfs/sys`
	`sudo umount /mnt/rootfs/proc`

# Clean-Slate Reinstall of the X-packages

1. Cleaning all the x-related packages.
sudo apt purge --autoremove xorg xserver-xorg x11-common xinit lxde lxde-core lxde-common lightdm openbox

2. Clean all utilities
sudo apt purge --autoremove x11-utils x11-xserver-utils xterm lightdm lxde*

3. Remove everything related to lxde as well
sudo apt purge --autoremove x11-utils x11-xserver-utils xterm lightdm lxde*

4. Clean all dependancies
sudo apt autoremove --purge
sudo apt clean

5. Verify the cleaning 
dpkg -l | grep -i xorg
dpkg -l | grep -i x11

6. Re-install it all
sudo apt install xserver-xorg xinit xterm lxde-core lxde-common lightdm lxterminal xinit



** Message: 06:22:28.623: main.vala:101: Session is LXDE  
** Message: 06:22:28.624: main.vala:102: DE is LXDE  
** Message: 06:22:28.910: main.vala:133: log directory: /home/vicharak/.cache/lxsession/LXDE  
** Message: 06:22:28.910: main.vala:134: log path: /home/vicharak/.cache/lxsession/LXDE/run.log