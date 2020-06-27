# Podman - Run Rootless GPU Container

With this repository we show you a way to use Podman to run a GPU container without root privileges.
First of all, we'll show you the prerequisites. Then we will show you from scratch how to install the prerequisites. 

## Prerequisites
- CentOS 7
- CUDA supported NVIDIA GPU

## Installation

### NVIDIA Driver
* 1.) Identify your Nvidia graphic card model by executing: 
```bash
lshw -numeric -C display
```
* 2.) Download the **Nvidia driver package** from [nvidia.com](https://www.nvidia.com/Download/index.aspx) using search criteria based on your Nvidia card model and Linux operating system. 
* 3.) Install all prerequisites for a successful Nvidia driver compilation and installation. 
```bash
sudo yum groupinstall "Development Tools"
sudo yum install kernel-devel epel-release
sudo yum install dkms
```
> Note: The **dkms** package is optional. However, this package will ensure continuous Nvidia kernel module compilation and installation in the event of new kernel update.
* 4.) Disable **nouveau driver** by changing the configuration `/etc/default/grub` file. Add the **nouveau.modeset=0** into line starting with **GRUB_CMDLINE_LINUX**. Below you can find example of grub configuration file reflecting the previously suggested change
```bash
nano /etc/default/grub

# Add nouveau.modeset=0 to the end of GRUB_CMDLINE_LINUX as shown in the example below
GRUB_TIMEOUT=5                                                                                                                                      
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"                                                                                   
GRUB_DEFAULT=saved                                                                                                                                  
GRUB_DISABLE_SUBMENU=true                                                                                                                           
GRUB_TERMINAL_OUTPUT="console"                                                                                                                      
GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet nouveau.modeset=0"                                                                                  
GRUB_DISABLE_RECOVERY="true"
```
Execute the following command to apply the new GRUB configuration change:
```bash
# BIOS:
$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg
# EFI:
$ sudo grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
```
* 5.) Reboot your  System.
```bash
reboot
```
* 6.) The Nvidia drivers must be installed while Xorg server is stopped. Switch to text mode by: 
```bash
systemctl isolate multi-user.target
```
* 7.) Install the Nvidia driver by executing the following command:
```bash
sudo bash NVIDIA-Linux-x86_64-*
```
> Hint: When prompted answer YES to installation of NVIDIA's 32-bit compatibility libraries and automatic update of your X configuration file.

* 8.) After the installation is complete, reboot your system
```bash
reboot
```

* 9.) Validate the installation and check if the correct driver is installed:
```bash
nvidia-smi
```

### CUDA
* 1.) Download the latest [Nvidia CUDA repository package](https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/) cuda-repo-rhel7-*.rpm. For example use the wget command to download the latest CUDA package: 
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/cuda-repo-rhel7-10.0.130-1.x86_64.rpm
```

* 2.) Install the CUDA repository package. This will enable CUDA repository on your CentOS 7 Linux system: 
```bash
sudo rpm -i cuda-repo-rhel7-10.0.130-1.x86_64.rpm
```
* 3.) Install the entire CUDA toolkit and driver packages: 
```bash
sudo yum install cuda -y
```
