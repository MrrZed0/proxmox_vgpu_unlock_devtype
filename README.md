How To Use Tesla P40 In Windows Without vGPU Unlock: https://github.com/MrrZed0/How-to-use-tesla-p40

Installing NVIDIA Driver On Proxmox:
-------------------------------------

1) Make sure to add the community pve repo and get rid of the enterprise repo (you can skip this step if you have a valid enterprise subscription)
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" >> /etc/apt/sources.list
rm /etc/apt/sources.list.d/pve-enterprise.list

2) Update and upgrade
apt update
apt dist-upgrade

3) We need to install a few more packages like git, a compiler and some other tools.
apt install -y git build-essential dkms pve-headers mdevctl




------------------
Add NVIDIA GPU To Blacklist:
----------------------------
nano /etc/modprobe.d/pve-blacklist.conf

blacklist nvidiafb
blacklist nouveau
blacklist nvidia*


GRUB CDLINE:
------------
nano /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"





-----------------------------
:Git repos and Rust compiler
-----------------------------

4) First, clone this repo to your home folder (in this case /root/)
git clone https://gitlab.com/polloloco/vgpu-proxmox.git

5) You also need the vgpu_unlock-rs repo
cd /opt
git clone https://github.com/mbilker/vgpu_unlock-rs.git

6) After that, install the rust compiler
curl https://sh.rustup.rs -sSf | sh -s -- -y --profile minimal

7) Now make the rust binaries available in your $PATH (you only have to do it the first time after installing rust)
source $HOME/.cargo/env

8) Enter the vgpu_unlock-rs directory and compile the library. Depending on your hardware and internet connection that may take a while
cd vgpu_unlock-rs/
cargo build --release





-----------------------------
:Create files for vGPU unlock
-----------------------------

9) First create the folder for your vgpu unlock config and create an empty config file
mkdir /etc/vgpu_unlock
touch /etc/vgpu_unlock/profile_override.toml

10) Then, create folders and files for systemd to load the vgpu_unlock-rs library when starting the nvidia vgpu services
mkdir /etc/systemd/system/{nvidia-vgpud.service.d,nvidia-vgpu-mgr.service.d}
echo -e "[Service]\nEnvironment=LD_PRELOAD=/opt/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" > /etc/systemd/system/nvidia-vgpud.service.d/vgpu_unlock.conf
echo -e "[Service]\nEnvironment=LD_PRELOAD=/opt/vgpu_unlock-rs/target/release/libvgpu_unlock_rs.so" > /etc/systemd/system/nvidia-vgpu-mgr.service.d/vgpu_unlock.conf

11) We have to load the vfio, vfio_iommu_type1, vfio_pci and vfio_virqfd kernel modules to get vGPU working
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules

12) Proxmox comes with the open source nouveau driver for nvidia gpus, however we have to use our patched nvidia driver to enable vGPU. The next line will prevent the nouveau driver from loading
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf

13) I'm not sure if this is needed, but it doesn't hurt :)
update-initramfs -u -k all

14) 
reboot

15) Obtaining the driver
Grab drivers from nvidia licensing portal website: https://nvid.nvidia.com/login
17.5 (550.144.02)
17.4 (550.127.06)
17.3 (550.90.05)
17.1 (550.54.16)
17.0 (550.54.10)
16.9 (535.230.02) !!! USE THIS IF YOU ARE ON PASCAL OR OLDER !!!
16.8 (535.216.01)
16.7 (535.183.04)
16.5 (535.161.05) the patch for this version is the same as for 16.4, the host driver wasnt changed in this release
16.4 (535.161.05)
16.2 (535.129.03)
16.1 (535.104.06)
16.0 (535.54.06)


15) Now, on the proxmox host, make the driver executable
chmod +x NVIDIA-Linux-x86_64-535.230.02-vgpu-kvm.run

16) And then patch it
./NVIDIA-Linux-x86_64-535.230.02-vgpu-kvm.run --apply-patch ~/vgpu-proxmox/535.230.02.patch

17) Installing the driver
./NVIDIA-Linux-x86_64-535.230.02-vgpu-kvm-custom.run --dkms -m=kernel

18) 
reboot

19) Wait for your server to reboot, then type this into the shell to check if the driver install worked
nvidia-smi

20) Click on vm and add the gpu to it with DevType 

21) Run the following command in proxmox shell for smbios info
dmidecode --type 1

22) Click on the vm you passing the gpu to and click on Options > SMBIOS settings

23) 
UUID: 00000000-0000-0000-0000-000000000111   
(Replace the 111 number with the vm number)
Manufacturer: GIGABYTE
Product: G340-G23-UI
Version: 0200
Serial: GCW1N3452A0429
SKU: 01237567890923431748GN
Family: Server

24) Change CPU Type To Host

25) Back in proxmox shell edit the vm config file 
nano /etc/pve/qemu-server/<VMID>.conf
[example nano /etc/pve/qemu-server/111.conf]

26) Add/change the following
args: -cpu host,-hypervisor,kvm=off, -smbios type=0,vendor="GIGABYTE",version=R13,date="12/05/2019"
cpu: host,hidden=1







---------------
:vGPU overrides
---------------

File Location: /etc/vgpu_unlock/profile_override.toml
PCI IDs
	- pci_id = 0x####@@@@ (Device ID followed by SubSystem ID)
	- pci_device_id = 0x#### (Device ID only)

		Architecture	Card		pci_device_id	pci_id
		- Maxwell	Quadro M6000	0x17F011A0	    0x17F0
		- Pascal	Quadro P6000	0x1B3011A0	    0x1B30
		- Volta		Quadro GV100	0x1DBA121A	    0x1DBA
		- Turing	Quadro RTX 6000	0x1E3012BA	    0x12BA
		- Kepler			(currently not supported)
		- Ampere 			(currently not supported)
        - Pascal        GTX 1070 8GB    0x1B8110DE          0x10DE
		
		
[profile.nvidia-259]
num_displays = 1          # Max number of virtual displays. Usually 1 if you want a simple remote gaming VM
display_width = 1920      # Maximum display width in the VM
display_height = 1080     # Maximum display height in the VM
max_pixels = 2073600      # This is the product of display_width and display_height so 1920 * 1080 = 2073600
cuda_enabled = 1          # Enables CUDA support. Either 1 or 0 for enabled/disabled
frl_enabled = 1           # This controls the frame rate limiter, if you enable it your fps in the VM get locked to 60fps. Either 1 or 0 for enabled/disabled
framebuffer = 0x74000000
framebuffer_reservation = 0xC000000   # In combination with the framebuffer size
                                      # above, these two lines will give you a VM
                                      # with 2GB of VRAM (framebuffer + framebuffer_reservation = VRAM size in bytes).
                                      # See below for some other sizes







----------------------------------
:Setting Up NVIDIA Licence Server:
-----------------------------------
Docker Version Can Be Found Here: https://hub.docker.com/r/collinwebdesigns/fastapi-dls

1) Setup FastAPI-DLS Server: 
Steps Are Found Here: 
https://git.collinwebdesigns.de/oscar.krause/fastapi-dls/-/blob/main/README.md?ref_type=heads

2) Create Ubuntu VM 

Debian / Ubuntu (using dpkg / apt)
Packages are available here:

GitLab-Registry

Successful tested with (LTS Version):


Debian 12 (Bookworm) (EOL: June 06, 2026)

Ubuntu 22.10 (Kinetic Kudu) (EOL: July 20, 2023)

Ubuntu 23.04 (Lunar Lobster) (EOL: January 2024)

Ubuntu 23.10 (Mantic Minotaur) (EOL: July 2024)

Ubuntu 24.04 (Noble Numbat) (EOL: Apr 2029)

Not working with:

Debian 11 (Bullseye) and lower (missing python-jose dependency)
Debian 13 (Trixie) (missing python-jose dependency)
Ubuntu 22.04 (Jammy Jellyfish) (not supported as for 15.01.2023 due to fastapi - uvicorn version missmatch)
Ubuntu 24.10 (Oracular Oriole) (missing python-jose dependency)

Run this on your server instance
First go to GitLab-Registry and select your
version. Then you have to copy the download link of the fastapi-dls_X.Y.Z_amd64.deb asset.

apt-get update
FILENAME=/opt/fastapi-dls.deb
wget -O $FILENAME <download-url>
dpkg -i $FILENAME
apt-get install -f --fix-missing


Start with systemctl start fastapi-dls.service and enable autostart with systemctl enable fastapi-dls.service.
Now you have to edit /etc/fastapi-dls/env as needed.
