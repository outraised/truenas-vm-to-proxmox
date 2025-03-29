# truenas-vm-to-proxmox
A guide on converting a TrueNAS Scale Virtual Machine file into a disk image readable by Proxmox VE\
The TrueNAS version used in this guide is `TrueNAS-SCALE-24.10.2`\
The Proxmox version used in this guide is `ProxmoxVE 8.3.4`
# Step 1: SSH into your TrueNAS host
On Windows you can do this by opening terminal and typing: `ssh [USER]@[IPADDRESS]`\
<br>
<img src="https://github.com/user-attachments/assets/c351ac75-20f1-4cd5-b47a-e9907dd5c8d8" style="width:65%; height:auto;">

# Step 2: Navigate to your Virtual Machines
Once connected, change your directory to the zvol where your VM's are kept\
As virtual machines are considered devices, by default this should be /dev/zvol\
In my case, I will type: `cd /dev/zvol/ssd/virtualmachines`\
then type `ls` to find the exact name of the VM you wish to migrate\
<br>
<img src="https://github.com/user-attachments/assets/7b0d94ef-2505-40ae-b1d0-b1aa8061f98d" style="width:75%; height:auto;">\
<br>
In this example I will be migrating `Minecraft-10zxy`

# Step 3: Copy the Virtual Machine image file and convert to qcow2
To create a copy of the virtual machine image I will enter the command:\
`sudo dd if=[IMAGEFILE] of=[DESTINATION] bs=[BLOCKSIZE]`\
Or in my case: `sudo dd if=/dev/zvol/ssd/virtualmachines/Minecraft-10zxy of=/mnt/minecraftcopy bs=4M`\
<br>
<img src="https://github.com/user-attachments/assets/4478fb58-a919-4dfe-b3c0-3d1af3751905" style="width:75%; height:auto;">
- As the virtual machine file is a block device so we use `dd` rather than `cp`
- Copy the file to /mnt/ as it's a good location for storing files temporarily to be transferred
- In this example the newly copied file is named minecraftcopy
- Depending on the size of the virtual machine, this could take a while
- Please note the dataset must have enough free space to create a copy of this image, it is not recommended to convert the image without making a copy

# Step 4: Convert the image file into a qcow2 file
Now it's time to convert the newly copied file into a qcow2 file that Proxmox can read\
As I have space within the TrueNAS dataset I will convert it then copy over to truenas\
You can copy the file to a different location or PC if necessary\
The command is
`sudo qemu-img convert -f raw -O qcow2 [SOURCEFILE] [CONVERTEDFILE]`\
Or in my case: `sudo qemu-img convert -f raw -O qcow2 minecraftcopy minecraftproxmox`\
<br>
<img src="https://github.com/user-attachments/assets/388132ff-d1ea-4cc4-87fa-c48acbcb6cbf" style="width:75%; height:auto;">\
<br>
Once again, this may take a while depending on the size of the VM and your CPU performance\
I have named the converted .qcow2 file 'minecraftproxmox'

# Step 5: Copy the converted file to your Proxmox host
Firstly, create a directory to store the file in your Proxmox host\
Open up the shell in your Proxmox host and create a directory of your choice using `mkdir`\
I will make one called `imagefiles` within /mnt/\
<br>
<img src="https://github.com/user-attachments/assets/ca806c18-beb7-4bd9-8855-68e0b4db0537" style="width:75%; height:auto;">\
<br>
Using the command `scp` move the file into this newly created directory\
`sudo scp /mnt/minecraftproxmox root@192.168.1.50:/mnt/imagefiles/`
- You will need to enter your TrueNAS password and your Proxmox Host's password
<img src="https://github.com/user-attachments/assets/cb17873c-c05e-4f8e-b452-260214b36188" style="width:75%; height:auto;">

# Step 6: Create a Virtual Machine in Proxmox
Now, create a virtual machine in Proxmox making sure to not mount an iso or create a disk\
Take note of the VM ID. In this example it is `550`\
<br>
<img src="https://github.com/user-attachments/assets/8d2b1c29-0995-4d79-bc0e-ea7d7ba2be74" style="width:75%; height:auto;">

# Step 7: Import the image file to your new VM
Open up the shell within Proxmox and type the command\
`qm importdisk [VM_ID] [IMAGE_NAME] [VM_STORAGE]`\
<br>
In my case it will be `qm importdisk 550 /mnt/imagefiles/minecraftproxmox local-lvm`\
<br>
<img src="https://github.com/user-attachments/assets/6ca68557-2435-42a6-95f9-4d061bc93b08" style="width:75%; height:auto;">\
<br>
This might take a while...\
<br>
<img src="https://github.com/user-attachments/assets/6da9582e-8adb-4bf9-90e5-aadd3e40803d" style="width:75%; height:auto;">\
<br>
Once complete it should show in your VM's hardware list as an Unused Disk\
<br>
<img src="https://github.com/user-attachments/assets/cc60064f-2048-4f98-8e27-b56789bd1387" style="width:75%; height:auto;">

# Step 8: Final tweaks/cleanup
As the disk is unused, we need to attach it and select a device type\
Click on the disk then edit\
By default it should be SCSI\
I'm going to select `SSD emulation` as it's stored on an SSD\
Click add. This will mount the disk as a virtual HDD\
<br>
<img src="https://github.com/user-attachments/assets/04a5b012-455d-4d66-a83c-b6a8808953a9" style="width:75%; height:auto;">\
<br>
<img src="https://github.com/user-attachments/assets/a1236d16-131d-479b-ab98-586a434bbacf" style="width:75%; height:auto;">\
<br>
As my VM was originally created as a UEFI drive, I will change the BIOS type to `OVMF (UEFI)`\
<br>
<img src="https://github.com/user-attachments/assets/3c3a1878-ca4c-4cc1-b783-c3dfcb2e11ce" style="width:75%; height:auto;">\
<br>
By default Proxmox doesn't enable an imported drive in the boot order\
Click on options > boot order > enable the drive > drag and drop the drive to the top of the boot order and click OK\
<br>
<img src="https://github.com/user-attachments/assets/0f2a2b2a-e87a-4056-b2ea-c8d2dabad9f8" style="width:75%; height:auto;">\
<br>
- Depending on your orginial VM's BIOS, CPU, Memory, Machine type etc you may need to tweak some more options to get the VM to boot
- Once imported, you can navigate to the directories we created earlier and remove the image files to save space
<br>
<img src="https://github.com/user-attachments/assets/1e9b0539-01c6-4281-b1d2-164ae62a23d2" style="width:75%; height:auto;">
<br>
<br>
You should now (hopefully) be able to boot the VM and enjoy the benefits of Proxmox!
