---
layout: doc
title: Installing a Windows VM
permalink: /doc/windows-vm/
---


Installing a Windows VM
=======================

Simple Windows install
----------------------

If you just want something simple and you can live without some features.

Works:
- display (1440x900 or 1280x1024 are a nice fit onto FHD hw display)
- keyboard (incl. correct mapping), pointing device
- network (emulated Realtek NIC)

Does not work:
- copy & paste (the qubes way)
- copying files into / out of the VM (the qubes way)
- assigning USB devices (the qubes way via the tray applet)
- audio output and input
- PCI device 5853:0001 (Xen platform device) - no driver
- all other features/hardware needing special tool/driver support

Installation procedure:
- Have the Windows 10 ISO image (I used the 64-bit version) downloaded in some qube.
- Create a new Qube:
  - Name: Win10, Color: red
  - Standalone Qube not based on a template
  - Networking: sys-firewall (default)
  - Launch settings after creation: check
  - Click "OK".
- Settings:
  - Basic:
    - System storage: 30000+ MB
  - Advanced:
    - Include in memory balancing: uncheck
    - Initial memory: 4096+ MB
    - Kernel: None
    - Mode: HVM
  - Click "Apply".
  - Click "Boot from CDROM":
    - "from file in qube":
      - Select the qube that has the ISO.
      - Select ISO by clicking "...".
    - Click "OK" to boot into the windows installer.
- Windows Installer:
  - Mostly as usual, but automatic reboots will halt the qube - just restart
   it again and again until the installation is finished.
  - Install on first disk.
  - Windows license may be read from flash via root in dom0:

    `strings < /sys/firmware/acpi/tables/MSDM`

    Alternatively, you can also try a Windows 7 license key (as of 2018/11
    they are still accepted for a free upgrade).
    
    I first installed Windows and all updates, then entered the license key.
- Afterwards:
  - In case you switch from `sys-network` to `sys-whonix`, you'll need a static
    IP network configuration, DHCP won't work for `sys-whonix`.
  - Use `powercfg -H off` and `disk cleanup` to save some disk space.


Qubes 4.0 - importing a Windows VM from R3.2
-------------------------------------------

Importing should work, simply make sure that you are not using Xen's newer linux stubdomain and that the VM is in HVM mode (these steps should be done automatically when importing the VM):

~~~
qvm-features VMNAME linux-stubdom ''
qvm-prefs VMNAME virt_mode hvm
~~~

Note however that you are better off creating a new Windows VM to benefit from the more recent emulated hardware: R3.2 uses a MiniOS based stubdomain with an old and mostly unmaintained 'qemu-traditional' while R4.0 uses a Linux based stubdomain with a recent version of upstream qemu (see [this post](https://groups.google.com/d/msg/qubes-devel/tBqwJmOAJ94/xmFCGJnuAwAJ)).


Windows VM installation
-----------------------

### qvm-create-windows-qube ###

An unofficial, third-party tool for automating this process is available [here](https://github.com/elliotkillick/qvm-create-windows-qube).
(Please note that this tool has not been reviewed by the Qubes OS Project.
Use it at your own risk.)
However, if you are an expert or want to do it manually you may continue below.

### Summary ###

~~~
qvm-create --class StandaloneVM --label red --property virt_mode=hvm win7new
qvm-prefs win7new memory 4096
qvm-prefs win7new maxmem 4096
qvm-prefs win7new kernel ''
qvm-volume extend win7new:root 25g
qvm-prefs win7new debug true
qvm-features win7new video-model cirrus
qvm-start --cdrom=untrusted:/home/user/windows_install.iso win7new
# restart after the first part of the windows installation process ends
qvm-start win7new
# once Windows is installed and working
qvm-prefs win7new memory 2048
qvm-prefs win7new maxmem 2048
qvm-features --unset win7new video-model
qvm-prefs win7new qrexec_timeout 300
# with Qubes Windows Tools installed:
qvm-prefs win7new debug false
~~~

To install Qubes Windows Tools, follow instructions [below](#xen-pv-drivers-and-qubes-windows-tools).

### Detailed instructions ###

MS Windows versions considerations:

- The instructions *may* work on other versions than Windows 7 x64 but haven't been tested.
- Qubes Windows Tools (QWT) only supports Windows 7 x64. Note that there are [known issues](https://github.com/QubesOS/qubes-issues/issues/3585) with QWT on Qubes 4.x
- For Windows 10 under Qubes 4.0, a way to install QWT 4.0.1.3, which has worked in several instances, is described below.

Create a VM named win7new in [HVM](/doc/hvm/) mode (Xen's current PVH limitations precludes from using PVH):

~~~
qvm-create --class StandaloneVM --label red --property virt_mode=hvm win7new
~~~

Windows' installer requires a significant amount of memory or else the VM will crash with such errors:

`/var/log/xen/console/hypervisor.log`:

> p2m_pod_demand_populate: Dom120 out of PoD memory! (tot=102411 ents=921600 dom120)
> (XEN) domain_crash called from p2m-pod.c:1218
> (XEN) Domain 120 (vcpu#0) crashed on cpu#3:

So, increase the VM's memory to 4096MB (memory = maxmem because we don't use memory balancing).

~~~
qvm-prefs win7new memory 4096
qvm-prefs win7new maxmem 4096
~~~

Disable direct boot so that the VM will go through the standard cdrom/HDD boot sequence:

~~~
qvm-prefs win7new kernel ''
~~~

A typical Windows 7 installation requires between 15GB up to 19GB of disk space depending on the version (Home/Professional/...). Windows updates also end up using significant space. So, extend the root volume from the default 10GB to 25GB (note: it is straightforward to increase the root volume size after Windows is installed: simply extend the volume again in dom0 and then extend the system partition with Windows's disk manager).

~~~
qvm-volume extend win7new:root 25g
~~~

Set the debug flag in order to have a graphical console:

~~~
qvm-prefs win7new debug true
~~~

The second part of the installation process will crash with the standard VGA video adapter and the VM will stay in "transient" mode with the following error in `guest-win7new-dm.log`:

> qemu: /home/user/qubes-src/vmm-xen-stubdom-linux/build/qemu/exec.c:1187: cpu_physical_memory_snapshot_get_dirty: Assertion `start + length <= snap->end' failed.

To avoid that error we temporarily have to switch the video adapter to 'cirrus':

~~~
qvm-features win7new video-model cirrus
~~~

The VM is now ready to be started; the best practice is to use an installation ISO [located in a VM](/doc/standalone-and-hvm/#installing-an-os-in-an-hvm):

~~~
qvm-start --cdrom=untrusted:/home/user/windows_install.iso win7new
~~~

Given the higher than usual memory requirements of Windows, you may get a `Not enough memory to start domain 'win7new'` error. In that case try to shutdown unneeded VMs to free memory before starting the Windows VM.

At this point you may open a tab in dom0 for debugging, in case something goes amiss:

~~~
tailf /var/log/qubes/vm-win7new.log \
   /var/log/xen/console/hypervisor.log \
   /var/log/xen/console/guest-win7new-dm.log
~~~

The VM will shutdown after the installer completes the extraction of Windows installation files. It's a good idea to clone the VM now (eg. `qvm-clone win7new win7newbkp1`). Then, (re)start the VM with `qvm-start win7new`.

The second part of Windows' installer should then be able to complete successfully. You may then perform the following post-install steps:

Decrease the VM's memory to a more reasonable value (memory balancing on Windows is unstable so keep `memory` equal to `maxmen`).

~~~
qvm-prefs win7new memory 2048
qvm-prefs win7new maxmem 2048
~~~

Revert to the standard VGA adapter: the 'cirrus' adapter will limit the maximum screen resolution to 1024x768 pixels, while the default VGA adapter allows for much higher resolutions (up to 2560x1600 pixels).

~~~
qvm-features --unset win7new video-model
~~~

Finally, increase the VM's `qrexec_timeout`: in case you happen to get a BSOD or a similar crash in the VM, utilities like chkdsk won't complete on restart before qrexec_timeout automatically halts the VM. That can really put the VM in a totally unrecoverable state, whereas with higher qrexec_timeout, chkdsk or the appropriate utility has plenty of time to fix the VM. Note that Qubes Windows Tools also require a larger timeout to move the user profiles to the private volume the first time the VM reboots after the tools' installation.

~~~
qvm-prefs win7new qrexec_timeout 300
~~~

At that point you should have a functional and stable Windows VM, although without updates, Xen's PV drivers nor Qubes integration (see sections [Windows Update](#windows-update) and [Xen PV drivers and Qubes Windows Tools](#xen-pv-drivers-and-qubes-windows-tools) below). It is a good time to clone the VM again.

### Installing Qubes Windows Tools on Windows 10

If the Xen bus and storage drivers version 9.0.0 are installed in a Windows 10 system without Qubes Windows Tools, and QWT 4.0.1.3 are installed after the Xen installation has finished, the Qubes interface works correctly. Files can be exchanged with other VMs, and the common clipboard works in both directions.

The installation of Qubes Windows Tools should **not** be done by using the parameter `--install-windows-tools`or by directly specifying `--cdrom=...`when starting the Windows VM, as this is bound to crash the VM on booting, showing the error `INACCESSIBLE BOOT DEVICE`- which makes no sense, but does happen.

So to get a working Windows 10 system (Standalone or Template VM) under Qubes R4.0, the following steps should be performed:

**to be replaced**
- Install Qubes Windows Tools in dom0: `sudo qubes-dom0-update qubes-windows-tools`. The iso will be the file `/usr/lib/qubes/qubes-windows-tools-4.0.1.3.iso`.
- Copy this file to some AppVM: `qvm-copy-to-vm VMname /usr/lib/qubes/qubes-windows-tools-4.0.1.3.iso`.
- In this VM, extract the file `qubes-tools-4.0.1.3.exe` from the iso, using the archive manager.
- Copy the installation kits of `xenvbd` and `xenbus` Version 9.0.0 (two Zip-files) from the Xen web site and the file `qubes-tools-4.0.1.3.exe` to the Windows system drive (normally `C:\`.)
**end of replaced text**

**new text**
- In the Windows 10 VM, download the installation kits of `xenvbd` and `xenbus` Version 9.0.0 (two files`xenvbd.tar`and `xenbus.tar`) from the Xen web site and the file `qubes-tools-4.0.1.3.exe` from https://www.qubes-os.org/doc/windows-tools/ **enter the final url** and store them on the Windows system drive (normally `C:\`.) In order to extract the contents from the tar-archives, you will need an external utility like 7zip.
**end of new text**

- Check the integrity of the file `qubes-tools-4.0.1.3.exe`by comparing its hash checksum. This can be done using the Windows command `certutil` specifying an appropriate hash algorithm like:
~~~
certutil --hashfile qubes-tools-4.0.1.3.exe SHA256
~~~
This utility supports the algorithms MD5, SHA1, SHA256 and SHA512 (to be entered in uppercase!). The correct hash values can be retrieved from the Qubes website: https://www.qubes-os.org/doc/windows-tools/ **enter the final url**

- Install `xenvbd` and `xenbus` version 9.0.0 by starting the file `dpinst.exe` from the `x64` directories of the extracted tar-files. If during installation, the Xen driver requests a reboot, select "No" and let the installation continue.
- After installation, reboot.
- Install Qubes Windows Tools 4.0.1.3 by starting `qubes-tools-4.0.1.3.exe`, not selecting the `Xen PV disk drivers` and the `Move user profiles` (which would probably lead to problems in Windows, anyhow). If during installation, the Xen driver requests a reboot, select "No" and let the installation continue - the system will be rebooted later.
- Shut down Windows.
- Set `qvm-features win10new gui 1`
- Reboot Windows. The VM starts, but does not show any window.
- Shutdown Windows from the Qube manager.
- Reboot Windows once more. Now the system is up, with QWT running correctly.

For me, this sequence worked for Windows 10 as template VM, and a corresponding AppVM worked too.

File copy operations to a Windows 10 VM are possible, if the Qubes OS `default_user` property is set to the user name used for access to that VM, which can be done via the command
~~~
qvm-prefs <VMname> default_user <username>
~~~
If this property is not set or set to a wrong value, files copied to this VM are stored in the folder
~~~
C:\Windows\System32\config\systemprofile\Documents\QubesIncoming\<source_VM>
~~~
If the target VM is an AppVM, this has the consequence that the files are stored in the corresponding TemplateVM and so are lost on AppVM shutdown.

Windows as TemplateVM
---------------------

Windows 7 and 10 can be installed as TemplateVM by selecting
~~~
qvm-create --class TemplateVM --property virt_mode=HVM --property kernel='' --label black Windows-7
qvm-create --class TemplateVM --property virt_mode=HVM --property kernel='' --label black Windows-10
~~~
when creating the VM. To have the user data stored in AppVMs depending on this template, Windows 7 and 10 have to be treated differently:

-  For Windows 7, the option to move the user directories from drive `C` to drive `D` works and causes any user data to be stored in the AppVMs based on this template, and not in the template itself.

-  After installation of Windows 10 as a TemplateVM, the Windows disk manager may be used to add the private volume as disk `D:`, and you may, using the documented Windows operations, move the user directories `C:\users\<username>\Documents` to this new disk, allowing depending AppVMs to have their own private volumes. Moving the hidden application directories `AppData`, however, is likely to invite trouble - the same trouble that occurs if, during QWT installation, the option `Move user profiles` is selected.

For Windows 10, configuration data like those stored in directories like `AppData` still remain in the TemplateVM, such that their changes are lost each time the AppVM shuts down. In order to make permanent changes to these configuration data, they have to be changed in the TemplateVM, meaning that applications have to be started there, which violates and perhaps even endangers the security of the TemplateVM. Such changes should be done only if absolutely necessary and with great care. It is a good idea to test them first in a cloned TemplateVM before applying them in the production VM.

AppVMs based on these templates can be created the normal way by using the Qube Manager or by specifying
~~~
qvm-create --class=AppVM --template=<VMname> 
~~~

On starting the AppVM, sometimes a message is displayed that the Xen PV Network Class needs to restart the system. This message can be safely ignored and closed by selecting "No".

**Caution:** These AppVMs must not be started while the corresponding TemplateVM is running, because they share the TemplateVM's license data. Even if this could work sometimes, it would be a violation of the license terms.

### Windows 10 Usage According to GDPR

If Windows 10 is used in the EU to process personal data, according to GDPR no automatic data transfer to countries outside the EU is allowed without explicit consent of the person(s) concerned, or other legal consent, as applicable. Since no reliable way is found to completely control the sending of telemetry from Windows 10, the system containing personal data must be completely shielded from the internet.

This can be achieved by installing Windows 10 on a TemplateVM with the user data directory moved to a separate drive (usually `D:`). Personal data must not be stored within the TemplateVM, but only in AppVMs depending on this TemplateVM. Network access by these AppVMs must be restricted to the local network and perhaps additional selected servers within the EU. Any data exchange of the AppVMs must be restricted to file and clipboard operations to and from other VMs in the same Qubes system.

Windows update
--------------

Depending on how old your installation media is, fully updating your Windows VM may take *hours* (this isn't specific to Xen/Qubes) so make sure you clone your VM between the mandatory reboots in case something goes wrong. This [comment](https://github.com/QubesOS/qubes-issues/issues/3585#issuecomment-366471111) provides useful links on updating a Windows 7 SP1 VM.

Note: if you already have Qubes Windows Tools installed the video adapter in Windows will be "Qubes video driver" and you won't be able to see the Windows Update process when the VM is being powered off because Qubes services would have been stopped by then. Depending on the size of the Windows update packs it may take a bit of time until the VM shutdowns by itself, leaving one wondering if the VM has crashed or still finalizing the updates (in dom0 a changing CPU usage - eg. shown with `xentop` - usually indicates that the VM hasn't crashed).
To avoid guessing the VM's state enable debugging (`qvm-prefs -s win7new debug true`) and in Windows' device manager (My computer -> Manage / Device manager / Display adapters) temporarily re-enable the standard VGA adapter and disable "Qubes video driver". You can disable debugging and revert to Qubes' display once the VM is updated.


Xen PV drivers and Qubes Windows Tools
------------------------------------

Installing Xen's PV drivers in the VM will lower its resources usage when using network and/or I/O intensive applications, but *may* come at the price of system stability (although Xen's PV drivers on a Win7 VM are usually very stable). There are two ways of installing the drivers:

1. installing the drivers independently, from Xen's [official site](https://www.xenproject.org/developers/teams/windows-pv-drivers.html)
2. installing Qubes Windows Tools (QWT), which bundles Xen's PV drivers.

Notes about using Xen's VBD (storage) PV driver:
- **Windows 7:** installing the driver requires a fully updated VM or else you'll likely get a BSOD and a VM in a difficult to fix state. Updating Windows takes *hours* and for casual usage there isn't much of a performance between the disk PV driver and the default one; so there is likely no need to go through the lengthy Windows Update process if your VM doesn't have access to untrusted networks and if you don't use I/O intensive apps. If you plan to update your newly installed Windows VM it is recommended that you do so *before* installing Qubes Windows Tools (QWT). If QWT are installed, you should temporarily re-enable the standard VGA adapter in Windows and disable Qubes' (see the section above).
- the option to install the storage PV driver is disabled by default in Qubes Windows Tools 
- in case you already had QWT installed without the storage PV driver and you then updated the VM, you may then install the driver from Xen's site (xenvbd.tar).

**Caution:** Installing the version 9.0.0 Xen drivers on Windows 7 (a system without QWT - QWT uninstalled) leads to an unbootable system. The drivers install without error, but after reboot, the system aborts the reboot saying ´Missing driver xenbus.sys´.

- **Windows 10:** The version 9.0.0 Xen drivers have to be installed before installing Qubes Windows Tools. Installing them on a system with QWT installed is likely to produce a system which crashes or has the tools in a non-functional state. Even if the tools were installed and then removed before installing the Xen drivers, they probably will not work as expected.


Installing Qubes Windows Tools:
- on R3.2: see [this page](/doc/windows-tools/)
- R4.0: you'll have to install QWT for Qubes R3.2. Be warned that QWT on R4.0 is a work in progress though (see [issue #3585](https://github.com/QubesOS/qubes-issues/issues/3585) for instructions and known issues).


With Qubes Windows Tools installed the early graphical console provided in debugging mode isn't needed anymore since Qubes' display driver will be used instead of the default VGA driver:

~~~
qvm-prefs -s win7new debug false
~~~


Further customization
---------------------

Please see the [Customizing Windows 7 templates](/doc/windows-template-customization/) page (despite the focus on preparing the VM for use as a template, most of the instructions are independent from how the VM will be used - ie. TemplateVM or StandaloneVM).

