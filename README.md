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
* 1.) Install the kernel headers and development packages for the currently running kernel:
```bash
sudo yum install kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

* 2.) Download the [NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-10.2-download-archive?target_os=Linux&target_arch=x86_64&target_distro=CentOS&target_version=7&target_type=runfilelocal) and install CUDA: 
```bash
wget http://developer.download.nvidia.com/compute/cuda/10.2/Prod/local_installers/cuda_10.2.89_440.33.01_linux.run
sudo sh cuda_10.2.89_440.33.01_linux.run
```
> Disable the driver option during installation because we have already installed the NVIDIA driver.

* 3.) Export system path to NVIDIA CUDA binary executables. Open the `~/.bashrc`using your preferred text editor and add the following two lines: 
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
sudo wget https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL7/nvidia-container.pp
sudo semodule -i nvidia-container.pp
```
#### Check and restore the labels
The SELinux policy heavily relies on the correct labeling of the host. Therefore we have to make sure that the files that are needed have the correct SELinux label.
* 1.) Restorecon all files that the prestart hook will need:
```bash
sudo nvidia-container-cli -k list | restorecon -v -f -
```
* 2.) Restorecon all accessed devices:
```bash
sudo restorecon -Rv /dev
```
#### Verify functionality of SELinux and prestart hook
To verify that the drivers and container tools are configured correctly, try running a cuda-vector-add container. We can run the container with docker or podman.
```bash
sudo podman run --user 1000:1000 --security-opt=no-new-privileges --cap-drop=ALL \
--security-opt label=type:nvidia_container_t  \
docker.io/mirrorgooglecontainers/cuda-vector-add:v0.1
```
> If the test passes, the drivers, hooks and the container runtime are functioning correctly.

#### Try it out with GPU accelerated PyTorch (with root privileges)
* 1.) Download the python code into a directory
```bash
mkdir pytorch_mnist_ex && cd pytorch_mnist_ex
wget https://raw.githubusercontent.com/pytorch/examples/master/mnist/main.py
```
> As it is written, this example will try to find GPUs and if it does not, it will run on CPU. We want to make sure that it fails with a useful error if it cannot access a GPU, so we make the following modification to the file with sed:
In order to run it without root privileges, follow these instructions.
```bash
sed -i '98 s/("cuda.*$/("cuda")/' main.py
```

* 2.) Run the training
```bash
sudo podman run --rm --net=host -v $(pwd):/workspace:Z \
--security-opt=no-new-privileges \
--cap-drop=ALL --security-opt label=type:nvidia_container_t \
docker.io/pytorch/pytorch:latest \
python3 main.py --epochs=3
```
#### Running rootless Podman GPU Container
* 1.) You can run containers rootless with podman. To use GPUs in rootless containers you need to modify /etc/nvidia-container-runtime/config.toml and change these values:
```bash
sudo nano  /etc/nvidia-container-runtime/config.toml

# Add "no-cgroups = true"
...
[nvidia-container-cli]
...
#no-cgroups = false
no-cgroups = true
...

# Add "debug = "~/.local/nvidia-container-runtime.log""
...
[nvidia-container-runtime]
#debug = "/var/log/nvidia-container-runtime.log"
debug = "~/.local/nvidia-container-runtime.log"
...
```
* 2.) Enable user namespaces. We have a temporary solution for this. If you are looking for a long-term solution, you will not find one for now. Execute the following command from the user who should execute the GPU container using Podman:
```bash
sudo bash -c 'echo 10000 > /proc/sys/user/max_user_namespaces'
sudo bash -c "echo $(whoami):110000:65536 > /etc/subuid"
sudo bash -c "echo $(whoami):110000:65536 > /etc/subgid"

# And then if you encounter an errors related to lchown run the following:
sudo rm -rf ~/.{config,local/share}/containers /run/user/$(id -u)/{libpod,runc,vfs-*}
```

3.) Start a GPU Container rootless. As a non-root user the system hooks are not used by default, so you need to set the --hooks-dir option in the podman run command. The following should allow you to run nvidia-smi in a rootless podman container:
```bash
podman run --rm -it -p 6006:6006 --security-opt=label=disable --hooks-dir=/usr/share/containers/oci/hooks.d docker.io/gaussianer/fasterseg:1
```
> We use the FasterSeg container, which we have already uploaded to DockerHub. FasterSeg and the environment we train on are also the reason why we want to run a GPU container without root privileges. 
