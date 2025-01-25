### MUXLESS GPU PASSTHROUGH GUIDE BY ArshamEbr ###

# 1. Check Bios for these three options and turn them on: 

A. Intel vt-d*

B. Virtualization*

C. UEFI Support ( pretty much everything from 2014 till now has it)

D. Secure Boot !!!DISABLE THIS ONE!!!

heres an example of my bios: https://download.lenovo.com/bsco/index.html#/textsimulator/IdeaPad%203-15ITL6%20(82H8)

Note: 'You don't need your CPU to support nither sr-iov nor Gvt-g 
My CPU is and 11th gen Intel and it doesn't support both ```LOL```

# 2. Iommu checking via the command `lspci -nnk` like this:

Do `lspci -nnk` and find your dgpu named 3D controller

```
0000:01:00.0 3D controller [0302]: NVIDIA Corporation GP107M [GeForce MX350] [10de:1c94] (rev a1)
	Subsystem: Lenovo Device [17aa:3f9b]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau, nvidia_drm, nvidia
```
A. Rememeber `0000:01:00.0` & `[10de:1c94]` & `[17aa:3f9b]`  (it's something else for you!) ... write them somewhere!

B. Also dont forget that it has to be a `3D controller` to be considered as muxless !!! if its named `VGA compatible` then you have to follow the Muxed laptop guide! `Bland Man Studio` https://www.youtube.com/watch?v=m8xj2Py8KPc&t=485s

# 3 Loading up VFIO stuff to stop the host from using it:

A. under NixOS we simply add these lines and we're good to go! (Due to your linux distro this part is different for you!)
```
 config = {
    boot = {
      kernelParams = [
      "intel_iommu=on"          
      "iommu=pt"                
      "vfio-pci.ids=10de:1c94" ### change it to your dgpu vendor id !!! take a look at Part 2.A for more info ###
      ];

      initrd.kernelModules = [
        "vfio_pci"          
        "vfio"
        "vfio_iommu_type1"
      ];
    }
 }
```
# 4. Download all of these files:

A. virtio-drivers.iso (for better performance with virtio devices) https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

B. spice-guest-tools-latest.exe https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe

C. Fake display driver (a task requiered under windows to run at startup) https://www.amyuni.com/downloads/usbmmidd_v2.zip

D. WinFSP for virtiofs to make a shared folder between windows vm and linux (download msi version) https://github.com/winfsp/winfsp/releases/tag/v2.0

E. Fake battery for vm (for bypassing error 43) https://lantian.pub/usr/uploads/202007/ssdt1.dat

F. download lookingGlass Version: B7-rc1 (windows host binary) https://looking-glass.io/downloads

G. Your desired windows .iso file https://www.microsoft.com/en-us/software-download/windows11

H. Patched file for your ease of mind: https://drive.google.com/file/d/1iocVGH2RTIolUIhUVtLH_05mCTjayY0q/view?usp=sharing

I. Make sure all virtualization package stuffs are installed on your desired linux distro like libvirt, qemu and virtiofs!

J. Heres the ready to go xml config for the vm: https://github.com/ArshamEbr/Nixo/blob/main/scripts/win11.xml !!! Dont FORGET TO CHECK PART 6.D to modify it for your system !!!

# 5. Let's extract your vbios safely:

A. Download your bios update file (mostly an .exe file) from your laptop's manufacturer website (for example for me it's lenovo)

B. Download VbiosFinder: `git clone https://github.com/coderobe/VBiosFinder.git`

B1: copy your bios.exe to vbiosfinder folder: `mv bios.exe VBiosFinder/`

D. Install these dependencies based on your distro!: "ruby ruby-bundler innoextract p7zip upx"

E. Install rom-parser (Run these command after eachother in sequance!):

E1: Download Rom-Parser `git clone https://github.com/awilliam/rom-parser.git`

E2: loacte into there: `cd rom-parser`

E3: build it: `make`

E4: move it over to VbiosFinder's 3rdparty folder: `mv rom-parser ../VBiosFinder/3rdparty`

E5: go back to where you was: `cd ..`

F. Install UEFIExtract (Run these command after eachother in sequance!):

F1: Download UEFIExtract `git clone https://github.com/LongSoft/UEFITool.git -b new_engine`

F2: locate into there: `cd UEFITool`

F3: run the script: `./unixbuild.sh`

F4: move it over to VbiosFinder's 3rdparty folder: `mv UEFIExtract/UEFIExtract ../VBiosFinder/3rdparty`

F5: go back to where you was: `cd ..`

G. Extract vBIOS

G1: go to Downloaded VbiosFinder dir: `cd VBiosFinder`

G2: run this bundler: `bundle update --bundler`

G3: run this bundle installer: `bundle install --path=vendor/bundle`

G4: Extracting your vbios from the copied bios.exe: `./vbiosfinder extract bios.exe`

G5: print the output: "ls output" And Note!!!
There will be one or a few files in the output folder for example:
### vbios_10de_1c94.rom
### vbios_10de_1c8d.rom
### vbios_10de_1c8e.rom
### ...
Going back to the Part 2.A! which we will use the rom file that has the same subsystem id and vendor id name as our Dgpu! for me was "10de:1c94" so Congratulation you found your Vbios!

G6: copy your vbios file to your home dir and rename it to vbios.rom: "cp vbios_10de_1c94.rom ~/vbios.rom"
now we can continue to the next part!

# 6. Adding the Extracted Vbios.rom file to VM's OVMF!:
Note: Based on some reports, UEFI firmware should NEVER be moved once built!!!

A. go somewhere to permanently place the files i'd recommand '/opt' so lets do it:

A1: go to '/opt' by running 'sudo su' first then : `cd /opt`

A2: Download edk2: `git clone https://github.com/tianocore/edk2.git`

A3: get inside: `cd edk2`

A4: Download all the requirments: `git submodule update --init`

B. Download these dependencies packages due to your desired linux distro: "git python2 iasl nasm subversion perl-libwww vim dos2unix gcc5"

C. get inside: `cd edk2/OvmfPkg/AcpiPlatformDxe`

D. Convert that vbios.rom that we copied earlier to home dir: `xxd -i ~/vbios.rom vrom.h`

E. Now we have to edit that converted vrom.h file!: use a text editor that your like to open it... i use vim `Vim vrom.h`

E1: rename the unsigned char array to VROM_BIN

E2: modify the length variable at the end to VROM_BIN_LEN, and memorize the number, for example mine was 238080!

E3: save and exit the editor! :wq :))

F. Download this file right there: `wget https://github.com/jscinoz/optimus-vfio-docs/files/1842788/ssdt.txt -O ssdt.asl`

F1. edit the ssdt.asl file: `Vim ssdt.asl`

F2. change line 37 to match VROM_BIN_LEN that we memorized earlier!:

`Name (RVBS, 238080) // size of ROM in bytes`

G. Run the following commands in sequance and dont mind the errors they're fine as long as Ssdt.aml is created!

G1: `iasl -f ssdt.asl`

G2: `xxd -c1 Ssdt.aml | tail -n +37 | cut -f2 -d' ' | paste -sd' ' | sed 's/ //g' | xxd -r -p > vrom_table.aml`

G3: `xxd -i vrom_table.aml | sed 's/vrom_table_aml/vrom_table/g' > vrom_table.h`

G4: copy both vrom.h and vrom_table.h to here:

`cp vrom.h /opt/edk2/OvmfPkg/Library/AcpiPlatformLib/`

`cp vrom_table.h /opt/edk2/OvmfPkg/Library/AcpiPlatformLib/`

H. lets go back to the main edk2 dir: `cd ../..`

I. ok since i already patched the QemuFwCfgAcpi.c file for ya'll you can just copy the downloaded file with yours!:

I1: i assume your downloaded files are in your Downloads folder so you have to copy that file over... so do:
`cp /home/YOUR_USER/Downloads/QemuFwCfgAcpi.c /opt/edk2/OvmfPkg/Library/AcpiPlatformLib/`

J. now run these commnads in sequance: 

J1: `make -C BaseTools`

J2: `. ./edksetup.sh BaseTools`

K. now we have to set these variables in Conf/target.txt:

K1:  ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc

K2:  TARGET_ARCH           = X64

K3:  TOOL_CHAIN_TAG        = GCC5

L: Run `build`

M. after it ran successfully verify file's existence in this dir: /opt/edk2/Build/OvmfX64/DEBUG_GCC5/FV 

M1: it should contain two files: OVMF_CODE.fd AND OVMF_VARS.fd !!!DO NOT MOVE THEM ANYWHERE!!!

# 7. Let's create our first VM :

A. open Virtual Machine Manager GUI

B: from the top select edit -> Preferences -> Tick the Xml Editing then close the tab

C. select the Create vm icon and follow the instructions... before completing it tick the edit config before running the vm!!!

D. inside the info section of the vm copy-paste my config.xml (DO NOT run or Apply yet!)

D1: scroll down til the `<devices>` section:

`<emulator>/run/libvirt/nix-emulators/qemu-system-x86_64</emulator>` ### Modify this path with the output of this command: `which qemu-system-x86_64` 
for example my output was 
`/run/current-system/sw/bin/qemu-system-x86_64`

D2. scroll down a bit under `<devices>` find:
```
   <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" discard="unmap"/>
      <source file="/var/lib/libvirt/images/Win11.qcow2"/> ### Replace this path with your own created virtual disk!!! ###
      <target dev="vda" bus="virtio"/>
      <boot order="1"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </disk>
```
D3. scroll down till you find `<hostdev>`
```
   <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
      </source>
      <rom bar="off"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0" multifunction="on"/> ### change this pci address to your dgpu iommu group!!! Check Part 2.A for more info
    </hostdev>
```
D4. scroll to the end section and you will see these lines:
```
  <qemu:commandline>
    <qemu:arg value="-acpitable"/>
    <qemu:arg value="file=/home/arsham/ssdt1.dat"/> ### change it to the path that you downloaded your fake battery file! ###
  </qemu:commandline>
```
D5. under that you'll see these lines change them ACCURATLY!!!
```
  <qemu:override>
    <qemu:device alias="hostdev0">
      <qemu:frontend>
        <qemu:property name="x-pci-vendor-id" type="unsigned" value="4318"/> ### '[10de:1c94]' so here you have to convert first half '10de' to integer and place the value here! ###
        <qemu:property name="x-pci-device-id" type="unsigned" value="7316"/> ### '[10de:1c94]' so here you have to convert second half '1c94' to integer and place the value here! ###
        <qemu:property name="x-pci-sub-vendor-id" type="unsigned" value="6058"/> ### '[17aa:3f9b]' convert first half '17aa' to integer and place the value here! ###
        <qemu:property name="x-pci-sub-device-id" type="unsigned" value="16283"/>### '[17aa:3f9b]' convert second half '3f9b' to integer and place the value here! ###
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
```
!!!NOTE:'[10de:1c94]' AND '[17aa:3f9b]' are from Part 2.A also to convert these values from hex to integer use this online converter: https://www.rapidtables.com/convert/number/hex-to-decimal.html 

D6. scroll up under `<devices>` find :
```
   <filesystem type="mount" accessmode="passthrough">
      <driver type="virtiofs"/>
      <binary path="/etc/profiles/per-user/arsham/bin/virtiofsd"/> ### change this path to the output of this command: "which virtiofsd" for example mine was: /etc/profiles/per-user/arsham/bin/virtiofsd ###
      <source dir="/home/arsham/"/> ### the path you want to be the shared directory between the vm and your linux disrto! ###
      <target dir="Home"/> ### the name of the mounted path ###
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </filesystem>
```
D7. scroll up under `<memory>` change the value to the desired ram that you want to pass to the vm!

D8. under cpu section pass 4 cores to the vm and tick the passthrough!

D9. attach both windows.iso (Part 4.G) and virtio.iso (Part 4.A) to the vm

D10. Dont forget to Add a Display since we have to set things up first!

D11. Remove network interface so we could skip win11 account creation

# 8. Run the Damn windows vm Now!

A. Bypass the requirments:

A1. SHIFT + F10 to open the command prompt while on the first page of installation

A2. Type `regedit` and press enter

A3. Navigate to `HKEY_LOCAL_MACHINE\SYSTEM\Setup`

A4. Right-click on the setup folder then select New -> Key And Name it `LabConfig`

A5. Inside the `LabConfig` folder, right-click in the right panel and select New -> DWORD (32-bit) value and name it `BypassTPMCheck` then set its value to `1`

A6. Create another DWORD (32-bit) value named `BypassSecureBootCheck` and set it to `1` too

A7. Close the window and continue the installation

B. We have to load the virtio drivers first:

B1. click on Load Drivers -> virtio.iso you mounted the load them up and voila now it showed up!

# 9. After the windows installation inside the vm

A. Install these things:

A1. Spice drivers

A2. LookingGlass (windows binary)

A3. Fake Display driver and copy that folder somewhere like `C:\Program Files\SOME_NAME`

A4. Virtio drivers

A5. your dgpu drivers

A6. WinFSD (set its service to automatic from manually)

B. Shutdown the vm and add the network interface and run this command `sudo virsh net-start default` for auto starting network thing, then start the vm again

C. we have to create a .bat file and copy it over to the fake display driver folder!

C1. Do Right-click and create a .txt file

C2. copy the following commands inside it (COPY PASTE)

```
@cd /d "%~dp0"

@goto %PROCESSOR_ARCHITECTURE%
@exit

:AMD64
@cmd /c deviceinstaller64.exe enableidd 1
@goto end
:end
exit
```
C3. Save it and Rename the end of the file from `.txt` to `.bat` for example: DGPU`.txt` -> DGPU`.bat`

C4. copy the file over to the Fake display driver

D. Creating some tasks with task scheduler to make things easier for us!

D1. Open task scheduler and create a new basic task and follow:

D2. Give it anyname you want then click next

D3. select `When i log on` go next

D4. Select `Start a Program` click next

D5. Browse to the `.bat` file we created and finish

D6. Click on the new task that we just created and tick the `Run with Highest Privileges` and done!

D69. what you just did creates a fake display at startup so we could use lookingGlass without needing a Real display!!!

E. now we create another task for LookingGlass itself... 

E1. run CMD with admin permissions

E2. Run the following command: `SCHTASKS /Create /TN "Looking Glass" /SC  ONLOGON /RL HIGHEST /TR 'C:\Program Files\Looking Glass (host)\looking-glass-host.exe'` Done


# 10. Finishing touches

A. shutdown the vm and remove that display from vm's info cuz we're now using the LookingGlass

B. (Optional) Adding Hyprland Keybinds for ease of access!

`$Primary$Alternate, p, exec, virsh -c qemu:///system start Win11 ; looking-glass-client -F`
`$Primary$Alternate, o, exec, virsh -c qemu:///system shutdown Win11`

B1. Replace `Win11` with your vm name that you created!

B2. also `$Primary$Alternate` is my personal combo so you can change them to your liking!

# FAQ

# I'm not using NixOS how to use LookingGlass?!
go here https://blandmanstudios.medium.com/tutorial-the-ultimate-linux-laptop-for-pc-gamers-feat-kvm-and-vfio-dee521850385 skip to the phase 5 and build LookingGlass from Source!
i would recommnad Version: B7-rc1 cuz we downloaded this version's windows binary earlier!

# I got error shm permission denied!
in NixOs we add this 
```
    systemd.tmpfiles.rules = [
    "f /dev/shm/looking-glass 0660 arsham qemu-libvirtd -"
  ];
```
so do something similar in your desired linux distro!

# Do i have to change the shm Size?
Most likely no... cuz i already set that to 64MB so its sufficient for most resolutions! but you can check LookingGlass website for more info on this

# I got error 23 no drivers found in device Manager
Run your gpu driver installer and it'll install the drivers for you

# I got error 43 in device manager for my dgpu
COPY-PASTE my config.xml and THEN modify it.. cuz theres always something that we miss to add or remove

# My system Freezes and i have to force reboot after running the vm!
Modify the Ram section and give only 80 percent of the ram that your system have

# I got evdev device error!
Just remove the evdev mouse and keyborad that i added... also check BlandManStudios video for more info about evdev devices  

# Sources

https://lantian.pub/en/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/

https://looking-glass.io/

https://blandmanstudios.medium.com/tutorial-the-ultimate-linux-laptop-for-pc-gamers-feat-kvm-and-vfio-dee521850385

https://github.com/tianocore/edk2

https://kilo.bytesize.xyz/gpu-passthrough-on-nixos