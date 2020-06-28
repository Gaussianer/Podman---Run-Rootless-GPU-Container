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
wget https://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/cuda-repo-rhel7-10.1.105-1.x86_64.rpm
```

* 2.) Install the CUDA repository package. This will enable CUDA repository on your CentOS 7 Linux system: 
```bash
sudo rpm -i cuda-repo-rhel7-10.1.105-1.x86_64.rpm 
```
* 3.) Install the entire CUDA toolkit and driver packages: 
```bash
sudo yum install cuda -y
```
* 4.) Export system path to Nvidia CUDA binary executables. Open the `~/.bashrc`using your preferred text editor and add the following two lines: 
```bash
sudo nano ~/.bashrc

# Add this two lines:
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
```

* 5.) Re-login or read your updated ~/.bashrc file:
```bash
source ~/.bashrc
```

* 6.) Confirm the correct CUDA installation: 
```bash
nvcc --version
nvidia-smi
```
### Install Podman
Podman will be needed later for execution and setup. Install Podman with the command:
```bash
yum -y install podman
```
### Adding the nvidia-container-runtime-hook
* 1.) Add the NVIDIA-Container-Runtime Repository
```bash
curl -s -L https://nvidia.github.io/nvidia-container-runtime/centos7/nvidia-container-runtime.repo | sudo tee /etc/yum.repos.d/nvidia-container-runtime.repo
```
* 2.) Install the NVIDIA-Container-Runtime
```bash
sudo yum -y install nvidia-container-runtime
```
* 3.)  Install libnvidia-container and the nvidia-container-runtime repositories
```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | tee /etc/yum.repos.d/nvidia-docker.repo
```
* 4.) The next step will install an OCI prestart hook. The prestart hook is responsible for making NVIDIA libraries and binaries available in a container (by bind-mounting them in from the host). Without the hook, users would have to include libraries and binaries into each and every container image that might use a GPU. Hooks minimize container size and simplify management of container images by ensuring only a single copy of libraries and binaries are required. The prestart hook is triggered by the presence of certain environment variables in the container: NVIDIA_DRIVER_CAPABILITIES=compute,utility.
```bash
yum -y install nvidia-container-toolkit
```
#### Adding the SELinux policy module
NVIDIA provides a custom SELinux policy to make it easier to access GPUs from within containers, while still maintaining isolation. To run NVIDIA containers contained and not privileged, we have to install an SELinux policy tailored for running CUDA GPU workloads. The policy creates a new SELinux type (nvidia_container_t) with which the container will be running.

Furthermore, we can drop all capabilities and prevent privilege escalation. See the invocation below to have a glimpse of how to start a NVIDIA container.

* 1.) First install the SELinux policy modul:
```bash
wget https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL7/nvidia-container.pp
semodule -i nvidia-container.pp
```
#### Check and restore the labels
The SELinux policy heavily relies on the correct labeling of the host. Therefore we have to make sure that the files that are needed have the correct SELinux label.
* 1.) Restorecon all files that the prestart hook will need:
```bash
nvidia-container-cli -k list | restorecon -v -f -
```
* 2.) Restorecon all accessed devices:
```bash
restorecon -Rv /dev
```
* 3.) Verify functionality 
```bash
podman run --user 1000:1000 --security-opt=no-new-privileges --cap-drop=ALL \
--security-opt label=type:nvidia_container_t  \
docker.io/mirrorgooglecontainers/cuda-vector-add:v0.1
```
> Everything is now set up for running a GPU-enabled container on this host. However, so far this GPU container can only be executed with root privileges. In order to run it without root privileges, follow these instructions.
