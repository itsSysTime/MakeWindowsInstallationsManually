# Making Windows installations manually (from Windows or Windows PE)
This guide will help you create Windows installation USB drives, CDs, or DVDs manually. It is intended for power users, but any audience can give it a try.
## BIOS systems or MBR disk scheme
> [!Note]
> On systems that use BIOS or use an MBR disk scheme, please follow this subsection of the guide to ensure your target has compatibility with the specific system specs.
To begin creating your bootable media, please make sure your target follows these prerequisites:
* Is compatible with the target version of Windows. There may be bypasses that Microsoft officially supports, but it recommended you do not for security and compatibility purposes.
* Has a USB, CD, or DVD drive/port on the side of the laptop or on the tower of the desktop computer. Corresponding controllers or drivers for a USB, CD, or DVD device is important to make sure the internal partitions are recognized by Windows.
* Has enough system resources to install this version of Windows. If it is intended for another OS, such as MacOS or Linux, it may be harder to follow this guide due to hardware and system-level differences.

This guide is intended for Microsoft Windows specifically. If you would wish to run this on another OS, you may need to borrow a Windows computer or use Windows PE media (such as manually crafting it with the Windows ADK, or using a pre-built version such as Hiren's Boot CD).
### Step 1. Create partitions
> [!Warning]
> Any data on the USB drive will be erased in this step of the process to prepare for staging of installation files.

To prepare your USB drive, you must enter the Command Prompt or enter PowerShell with elevated privileges and enter `diskpart.exe`. This loads the *DiskPart* utility on Windows, and after that, you can use the series of commands:
```DiskPart
list disk
```
Depending on size or inside partitions, select the number of your USB drive's disk.
```DiskPart
select disk [num]
clean
```
Now, we are creating the main installation partition. It can be anywhere from 8-32GB in size, but you can make it any size you'd like!
```DiskPart
create partition primary size=[num in MiB]
format fs=ntfs [quick]
assign letter [any drive letter that is available, remember this]
active
```
### Step 2. Stage installation files and complete
While tools like Rufus, Balena Etcher, or Media Creation Tool are used, this a manual guide and this means that we have to stage the files to the USB drive partition manually.

This process assumed you have your ISO prepared and ready to mount. If not, *you can find an ISO online and download it to your local disk or external storage*.
Now, if you are ready to begin, you can continue:
1. Mount the ISO image you selected (Windows). Remember the drive letter.
2. You can either manually craft the first bootsector to the drive (by getting the ISO and writing the first 512 bytes to the USB drive's partition, however this guide does not include those steps, unfortunately) or wait until Step 4 for a safer option. To manually craft the first bootsector, you can use the series of commands:
3. Enter the Command Prompt or PowerShell and use the command `xcopy [mounted drive letter]:\* [USB drive letter]:\* /H /E /F /G /J`, this allows unbuffered I/O (Input/Output, useful for larger files), including hidden files, directories and subdirectories empty or not, shows files being copied, and allows encrypted files to be copied. If you are using a CD or DVD, you can use isoburn.exe instead.
4. Wait patiently for the copy process to finish. After this is done, apply the boot code using `bootsect /nt60 [USB DRIVE LETTER] /force` for Windows Vista and above. For versions Vista and below, use `bootsect.exe /nt52 [USB DRIVE LETTER] /force`
5. The drive should be ready.
#### Step 2.1: Add updates or drivers
If you would wish to add updates or drivers, you can use the DISM utility provided by Microsoft in the Command Prompt or Terminal.
To find the latest update packages for Windows, you can look at the [Microsoft Update Catalogue](https://catalog.update.microsoft.com/home.aspx) and download the latest Windows updates for that version and dependencies.
First, you need to mount install.wim (sometimes install.esd, if so, you can follow [this guide](https://www.thewindowsclub.com/how-to-convert-install-esd-to-install-wim-file-in-windows-10) on converting to .wim) or boot.wim, using:
`dism.exe /Mount-Image /ImageFile:[path to boot.wim or install.wim] /MountDir:[directory to mount install.wim] /Index:[any index you wish. Each index represents an edition of Windows]`
To extract index info, you can use:
`dism.exe /Get-WimInfo /Wimfile:[path to boot.wim or install.wim]`
This will show index info for each index, and you can choose an index you want. If you want multiple editions, you need to mount boot.wim or install.wim multiple times with the indexes you want for each attempt.

As for drivers, you will need to extract the files from a self-extracting archive using tools such as 7-Zip, but some OEMs include parameters to extract them to a specific directory. DISM accepts a folder with multiple driver files present (*.sys and *.inf)
Once you have a .cab or .msu file for the Windows Update package, you can run this command in your terminal:
`dism.exe /Image:[mounted install.wim using DISM] /Add-Package /PackagePath:[path to the Windows Update file]`

And for adding drivers to a mounted Windows install.wim image, you can use:
`dism.exe /Image:[mounted install.wim using DISM] /Add-Driver /Driver:[path that contains all drivers] /Recurse`

You can use boot.wim, present in \sources\, to load the drivers during Windows PE (which is what loads before Setup), but mounting install.wim ensures that the drivers stay persistent in a captured ISO file or installation medium.

After you are finished, you can use:
`dism.exe /Unmount-Image /MountDir:[mounted boot.wim or install.wim] /commit`
This will unmount and save the changes of install.wim or boot.wim.

Adding drivers or updates to install.wim is slower, but more persistent every time you use the installation medium or captured ISO file. Boot.wim only loads them during the Windows PE Setup environment.
## UEFI/GPT
> [!Note]
> This setup is quite different than the process for BIOS or MBR.
### Step 1. Creating partitions
For a installer ready to boot without an ESP, you should make your main partition set to FAT32 (but be aware that install.wim can be over 4GB and might require splitting into .swm files).
For the struggle of NOT splitting install.wim, you can use NTFS, but you must configure an ESP.

You can read the first step to MBR, which provides similar commands you can use for this step, as there are no major differences, EXCEPT the disk must be a GPT disk. If not already, after cleaning your disk, you can use `convert gpt`.
To create an ESP (EFI System Partition), you must use:
```DiskPart
list disk
select disk [num of USB disk]
create partition primary size=[anywhere from 260 to 512MiB, higher could be needed for other OSes]
format fs=FAT32 [quick]
set id=c12a7328-f81f-11d2-ba4b-00a0c93ec93b override
```
### Step 2. Staging installation files
This part may be slightly different. Before copying installation files and after mounting the ISO, if install.wim is over 4GB on a FAT32 partition, you need to copy it to a temporary directory and split it using DISM:
`dism.exe /Split-Image /ImageFile:[path to install.wim] /SWMFile:[parent directory of install.wim, which is \sources\]\install.swm /FileSize:4000`, but copy the ISO's files/contents recursively similar to the MBR/BIOS path, but remove install.wim from your USB drive and replace it with the install*.swm files that were created.

### Step 3. Continue with the EFI System Partition
We need to mount our ESP in order to actually access it.
To find the specific GUID of our ESP, we can use the PowerShell snippet:
`Get-Partition | Select-Object DiskNumber, PartitionNumber, Guid`
This will list each partition's (or volume) GUID, Disk number, and partition number. Once you find the partition number and disk number of your ESP, you can find the ESP quite quickly. Remember it.
To mount your ESP using the GUID, run the command:
`mountvol [drive letter]: \\?\Volume{GUID-HERE}\`
Since our ESP is not yet complete, you can either copy the UEFI:NTFS files provided in this repository to that drive and then unmount, copy the EFI files from the installation medium to the drive (including the EFI folder itself), or you can use bcdboot to apply the boot files from a mounted install.wim, unmount install.wim, and then unmount the ESP.

Mounting install.wim instructions are provided in Step 2.1 of BIOS/MBR setup. After mounting, you can use:
`bcdboot.exe [mounted install.wim drive letter]:\Windows /s [ESP drive letter]: /f ALL`
ALL is used for compatibility with BIOS and UEFI. After this, you can unmount install.wim (and discard changes with /discard instead of /commit), as the instructions are also in Step 2.1 of BIOS/MBR setup.

This ensures the BCD points to the proper boot entries, and that UEFI can find firmware applications to start the Windows Boot Manager and then load Windows PE, which loads Setup.
To unmount our ESP, the final step, we can simply use:
`mountvol [drive letter]: /d`

After that, make sure your ESP is marked as System in *DiskPart*, and your manual Windows USB should be ready!

## Important notes
* On fixed disks, you should use `create partition efi`, but for this case, we needed to set the ID to manually make it an EFI System Partition because it is not supported on removable media to format the ESP as FAT32.
* You should not forget to use the switch /commit after unmounting install.wim when applying updates or drivers. Forgetting to do so will discard any changes you made, including applying updates or installing drivers to the mounted install.wim image.
* You should also ensure that your Windows Update package has the dependencies it needs.
* On hosts where you are using a different OS, such as a Linux distro or MacOS, you can create a Windows VM or use Windows PE media to run these commands. You can either make Windows PE manually with the Windows ADK or use a pre-built like Hiren's Boot CD.

# Credits
ItsSysTime
