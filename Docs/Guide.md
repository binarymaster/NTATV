# How to install Windows XP on your Original Apple TV
Installing Windows XP on the Apple TV is quite a bit more difficult than installing it on most other hardware for various reasons. There are 2 ways to do this: the easy way and the hard way.

## The easy way
If you just want to see Windows run on your TV, you can just [download a pre-installed image from archive.org](https://archive.org/details/apple-tv-windows-xp-full.img). However, there are a few disadvantages to doing this:
* This image teaches you absolutely nothing.
* It's already installed and slightly used by me, meaning that some of my configuration is on there.
* I might not update this image as often as I update the other components since regenerating it is a pain.

To use this image, simply download it, extract it, connect your Apple TV's drive to your computer using a USB to IDE adapter (a bootable USB will not, and will never, work, due to the complexity of writing my own USB stack from scratch), and use a tool like `dd`, `gnome-disks` or Etcher to restore the image to the TV's hard drive. Windows should then boot on your TV. Huzzah!

## The hard way (AKA creating your own image)
Can you tell I want people to do it this way yet?

This method of installing Windows is quite a bit more involved than the previous method. You will need the following items:

* Mac or PC with at least 20GB of free space (Linux-based computer recommended)
* A second computer with Windows XP already installed (don't worry, we won't be erasing this one)
* USB to IDE adapter and IDE hard disk (a bootable USB will not, and will never, work, due to the complexity of writing my own USB stack from scratch)
* QEMU for your computer's OS 
    * For macOS, use [Homebrew](https://brew.sh)
    * For Linux, use your system's package manager
    * For Windows, use [Stefan Weil's binaries](https://qemu.weilnetz.de/w64/) (apparently, I've never used QEMU on Windows)
* A Windows XP ISO
* A copy of the NTATV Drivers and Utilities ISO (see [Releases](https://github.com/DistroHopper39B/NTATV/releases/))

First, we need a virtual hard disk image. We will restore this to the Apple TV's hard drive once the install completes. To create one, use the `qemu-img` command-line utility like this:
```
qemu-img create appletv_windows.img 30G
```
This will be the amount of space available to Windows. You can resize the OS to fit the full size of your Apple TV's hard drive later. Ensure the size is lower than your hard drive's stated capacity by a few gigabytes as shrinking the disk image is difficult.

Now that your hard drive image is created, we need to start the install of Windows XP. Run the following command:
```
qemu-system-i386 -enable-kvm -m 2048 -hda appletv_windows.img -cdrom /path/to/windowsxp.iso -boot d -device usb-ehci,id=ehci -device usb-kbd
```
For macOS, replace `-enable-kvm` with `-accel hvf`. For Windows, there are a few ways to set hardware accelerated virtualization up. I've never used QEMU on Windows, so you can look it up. Alternatively, you could use VirtualBox for the install, but if you do, **you must enable PAE/NX in VirtualBox settings. IF YOU DON'T YOU WILL REGRET IT. DON'T ASK ME HOW I KNOW THIS, AND DON'T ASK ME HOW MANY MONTHS I WASTED ON THIS PROJECT UNTIL I FIGURED THIS OUT.**

Once you get to the partition list, instead of pressing Enter like usual, press C to create a new partition. Make your new partition at least 20MB smaller than the full size of the disk; I usually leave a bit more room than that.

![Windows XP partition setup](Assets/Guide/QEMU_001.png)

Then, create a partition on the remaining space. By the end, your partition map should look something like this:

![Windows XP partition setup](Assets/Guide/QEMU_002.png)

Now, you can install to the larger partition and go through the installation as normal. This should only take a few minutes unless you type in your product key incorrectly 37 times.

Once XP is installed, navigate to File Explorer and format the extra partition we created as FAT32 (use Quick Format). This will act as the EFI system partition and is where the bootloader will be stored. Once you do that, shut down the system.

Back on the host system's command line, we now need to swap out the CDROM from the Windows XP ISO to the NTATV Drivers and Utilities ISO. We also need to tell QEMU to boot from the hard drive instead of the CD-ROM. The QEMU parameters should look like this:
```
qemu-system-i386 -enable-kvm -m 2048 -hda appletv_windows.img -cdrom /path/to/ntatv_drivers_and_utils.iso -boot c -device usb-ehci,id=ehci -device usb-kbd
```

Open the CD in Explorer and double click on the `HDDFix` registration entries. When prompted click "Yes" and you should see "Information from HDDFix.reg has been entered into the registry".

![HDDFix Registry Entries](Assets/Guide/QEMU_004.png)

Drag the contents of the `Boot` folder to the FAT32 partition we just created.

![Boot files in E: drive](Assets/Guide/QEMU_005.png)

Drag `freeldr.ini` to the root of the `C:` drive.

![Freeldr.ini](Assets/Guide/QEMU_011.png)

Now it's time to set up the video driver. Open the Device Manager, right click on "Video Controller (VGA Compatible)", and select "Update Driver...". Select "No, not this time", then "Install from a list or specific location (Advanced)", then include the `D:\Video` location in the search.

![Hardware Update Wizard](Assets/Guide/QEMU_006.png)

Select "Continue Anyway", then when prompted for the Windows XP CD-ROM click "OK". Go to "Browse..." and go to `D:\Video` for the additional file (`BOOTVID.DLL`). This 

![Windows XP CD prompt](Assets/Guide/QEMU_007.png)

You will get to a screen stating "This device cannot start. (Code 10)." This is normal. Click Finish, then close out of Device Manager.

Now, shut down your computer and restore the appletv_windows.img to an IDE hard drive using a hard drive flashing tool of your choice (DD, Etcher, GNOME Disks). Once the restore is complete, mount the Windows XP ISO on your host machine and navigate to the `I386` directory, then extract `PCIIDE.SY_` to `pciide.sys` using either Windows' `expand` command line utility or 7-Zip/p7zip. `pciide.sys` should be placed in the Apple TV hard drive `\WINDOWS\system32\drivers`; ensure the file name is in all lowercase.

Now, connect the Apple TV's hard drive to the secondary computer already running XP. It is possible to do this using another VM and passing through the hard drive as either an IDE disk or USB device, but using another computer is easier. On your other computer, open Registry Editor, click on the `HKEY_LOCAL_MACHINE` folder, then go to `File -> Load Hive`.

![Load Hive](Assets/Guide/Screenshot_1.png)

It will ask you to select a hive; navigate to your Apple TV XP install `\WINDOWS\system32\config` and select the `system` file.

![Navigated to config folder](Assets/Guide/Screenshot_2.PNG)

When asked, name the key `SYSTEM_TV`. Once you do that open `SYSTEM_TV\ControlSet001`, then right click on `Enum` and select `Permissions...`. Enable full control for Everyone.

![Permissions for Enum](Assets/Guide/Screenshot_4.PNG)

Now, right click on `Enum` again and select `Delete`. You'll get an `Error while deleting key` error, this can be safely ignored. Go back to `SYSTEM_TV` and go to `File -> Unload Hive`, then safely eject the hard drive from the system.

Place the hard drive into the Apple TV, but don't reassemble it; we'll need the drive back in a few minutes. Plug in your TV, connect a keyboard and mouse to it, and within about a minute, you should see the glorious NTATV logo, followed by FreeLoader and the Windows XP logo! The system will hang for several minutes; it needs to reenumerate all system devices, which takes a while. You'll hear the hard drive spinning for several minutes, then the system might reboot, or it might hang infinitely (there is no consistency as to what it does). Either way, after the hard drive stops spinning, the Apple TV reboots, or about 5 minutes have elapsed, remove the hard drive and connect it back to the Windows XP computer.

Open `regedit` again and mount the hive again. Go to `ControlSet001\Control\Enum\PCI`, then select what should be the first entry in that list (the entry starts with `VEN_10DE`). Navigate to the key (folder) inside of that key (should be a string of mostly numbers with some letters and ampersands).

![VEN_10DE broken registry entry](Assets/Guide/Screenshot_6.PNG)

Now, we need to manually enumerate these entries. This part of the installation process cannot easily be scripted because the "string of mostly numbers..." is different on every Windows install. The contents of your key **must match the below screenshot exactly**; most of the entries are present, but `ConfigFlags`, `Driver`, `Mfg` and `Service` must be created manually. By the end, the contents should look exactly like this:

![VEN_10DE working registry entry](Assets/Guide/Screenshot_7.PNG)

*Tip: `ClassGUID` and `Driver` have the same UUID, so there's no reason to type it in manually; just copy/paste the value then add `\0000` to the end of it.*

Congratulations, if you did everything right, you successfully set up Windows XP on your Apple TV! Now it's time to test it out. Unload the hive and disconnect the drive as before, then put the Apple TV back together. You should get to the desktop!

## Installing Drivers
"The Easy Way" already has drivers installed. For "The Hard Way", however, very few drivers for the hardware are present. The RTL8139 Ethernet controller works out of the box, but the sound and WiFi require the installation of drivers from [Boot Camp 3.1.1](https://archive.org/details/boot-camp-3.1.1). Running the main `setup.exe` will not work, but WiFi will work if you run `Boot Camp\Drivers\Broadcom\BroadcomNetworkAdapterXP.exe`, and audio will work if you run `Boot Camp\Drivers\RealTek\RealtekSetupXP.exe`. I haven't tested the chipset driver as of yet, but it probably works fine.

Installing the NVIDIA drivers through Snappy Driver Installer Origin or by hacking the GeForce 179.48 INF file leads to system breakages as of now. **Unless you want a broken install, do NOT try to install the NVIDIA driver!**

# Troubleshooting
## USB
As of now, I'm not sure if my issue is hardware or software, but I've had just infinite issues with USB. Some hubs just don't work, and when the keyboard is connected without a hub, both Linux and Windows constantly crash. Windows also refuses to detect my USB hub immediately after installing it, although I have been able to get it to work in the past somehow. If you have issues with crashing or (more likely) issues with your USB hub not being detected, I have come up with a hack solution that might be helpful. After performing the manual enumeration as stated above, here's what you do:
* Connect only the keyboard to the Apple TV with no hub
* When the Found New Hardware Wizard appears, very quickly tell Windows not to connect to Windows Update, then tell it to install the software automatically and don't let it connect to the Internet. It will fail, but as long as you don't uncheck "Don't prompt me again to install this software" at the end, it will work.
* Do this repeatedly (I think about 7 times) until the wizard stops appearing.
* Properly shut down the Apple TV. It won't actually turn off, but it will freeze on the shutdown screen and Caps Lock will stop working.
* Boot back up with the keyboard and mouse connected to a USB hub.

Again, I'm not sure if anyone else's Apple TV has the same issue or if it's just an issue with mine.

## FreeLoader errors (Error when detecting hardware, Error opening freeldr.ini or file not found)
This is due to the extremely buggy FreeLoader IDE driver that was designed for the original Xbox. It seems to have issues on drives larger than 128GB, drives made before about 2004, or SSDs. I'd recommend replacing the drive with a different one for the time being.

## System hangs at a black screen for 31 seconds after displaying the boot logo, then continues
This is also an IDE driver issue. The same advice applies.
