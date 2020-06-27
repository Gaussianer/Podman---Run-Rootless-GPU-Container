# Podman - Run Rootless GPU Container

With this repository we show you a way to use Podman to run a GPU container without root privileges.
First of all, we'll show you the prerequisites. Then we will show you from scratch how to install the prerequisites. 

## Prerequisites
- CentOS
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
