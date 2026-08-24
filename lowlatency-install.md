The following script describes the commands to install a lowlatency system, which for example can be used to control a Franka Emika Panda robot arm with all control functionality being containerized in docker containers. 

Dockerfiles for this can be found in https://github.com/David0tt/ThesisInformation


```bash
# low-latency kernel
# sudo apt install linux-lowlatency-6.11
# sudo apt install linux-lowlatency-6.14
# sudo apt install linux-lowlatency-6.17
sudo apt install linux-lowlatency-7.0

# NVIDIA Drivers: (following https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html -> Ubuntu -> Network Repository Installation)
sudo apt install linux-headers-$(uname -r)
export distro="ubuntu2404"
export arch="x86_64"
wget https://developer.download.nvidia.com/compute/cuda/repos/$distro/$arch/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
# It is recommended to pin the driver to a specific version, so on normal system update no newer nvidia driver versions will be installed, which often can lead to broken systems
sudo apt install nvidia-driver-pinning-595
sudo apt install nvidia-open

# Install CUDA (following https://developer.nvidia.com/cuda-13-1-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_network)
# (or select a different version)
sudo apt-get -y install cuda-toolkit-13-3

# Install Docker (following https://docs.docker.com/engine/install/ubuntu/)
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify (Optional)
# sudo docker run hello-world

# NVIDIA Container Toolkit installation (following: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
# Optionally select a newer version
# export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.17.8-1
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.19.0-1
sudo apt-get install -y \
    nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}

# Configure
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify (Optional) -> should show nvidia-smi
# sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi

# Users that want to use Docker should be added to the docker group
# sudo adduser ott docker


# Network setup (Optional)
# This network setup is pecific to our setup, where the PC has a second network card, which is connected to the control box of a Franka Emika Panda robot
# Set up the network configuration (IP: 172.16.0.1 on the second network card which is connected to the Panda)
ifconfig # to show all the network interfaces, find the one that you want, in this case `enp11s0`
# Add the IP address to the network config by editing
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    enp12s0:
      dhcp4: true
    enp11s0:
      dhcp4: no
      addresses:
        - 172.16.0.1/24

sudo netplan apply 
```


# Cleanly uninstall NVIDIA Driver
It can often be required to fully cleanly uninstall the nvidia drivers before reinstalling the nvidia driver. Follow roughly the following instructions:

```bash
# Check what's currently installed
dpkg -l | grep -i nvidia
nvidia-smi

# Remove all NVIDIA packages
sudo apt remove --purge '^nvidia-.*' -y
sudo apt remove --purge '^libnvidia-.*' -y
sudo apt remove --purge '^cuda-.*' -y        # if CUDA is installed
sudo apt remove --purge '^libcuda.*' -y      # if CUDA libs present

# Remove leftover config and dependencies
sudo apt autoremove -y
sudo apt autoclean

# Check, that everything is removed:
dpkg -l | grep -i nvidia

# reboot
sudo reboot

# Verify
dpkg -l | grep -i nvidia      # should show nothing
nvidia-smi                    # should return "command not found"

# NVIDIA drivers are now cleanly fully uninstalled, you can now install new following the instructions above
```

# Cleanly Uninstall kernels
Over the course of running such a system for longer times, it can happen that multiple kernels get installed. This can be annoying, since for installations with dkms builds, dkms modules for all these kernels need to be built. Also, for a clean unified system it also makes sense to only have the kernels installed which are actually used. Here are instructions to uninstall kernels:

```bash
# See installed stuff  and list status of kernel modules managed by DKMS
sudo dkms status                      # show status of kernel modules managed by DKMS
uname -a                              # show the currently active kernel (sanity check)

dpkg --list | grep linux-image        # show installed kernel packages
dpkg --list | grep linux-headers      # show all kernel headers packages installed
dpkg --list | grep linux-lowlatency   # show lowlatency linux kernel packages (which might not be needed anymore)
dpkg --list | grep linux-modules   # show installed linux kernel modules

# remove all the kernel packages not wanted anymore, found from the `dkms --list` commands above e.g.:
sudo apt purge linux-image-6.11.0-1016-lowlatency linux-image-6.14.0-37-generic linux-image-lowlatency-6.11 linux-image-generic-6.14 linux-headers-6.11.0-1016-lowlatency linux-headers-6.11.0-29-generic linux-headers-6.14.0-37-generic linux-lowlatency-hwe-6.11-headers-6.11.0-1016 linux-lowlatency-hwe-6.11-tools-6.11.0-1016

# Remove leftover config and dependencies
sudo apt autoremove --purge
sudo apt autoclean

# Check in the following directories if there are any remnants of uninstalled kernels
ls /usr/src/
ls /lib/modules/
# If so, remove them, e.g.
# sudo rm -rf /usr/src/*6.11*
# sudo rm -rf /lib/modules/6.11.0-29-generic

# Check, if everything unwanted is removed:
dpkg --list | grep linux-image
dpkg --list | grep linux-headers
dpkg --list | grep linux-lowlatency
dpkg --list | grep linux-modules
sudo dkms status
# if `sudo dkms status` still shows something, you can further investigate, e.g. with 
# dpkg --list | grep 6.11 # e.g. if some kernel 6.11 still exists, show all packages with 6.11

# If `sudo dkms status` still reports some registered kernel modules, even after all corresponding dpkg packages are removed, e.g.:
# 
#        (base) ott@panda3gpu5090:~$ sudo dkms status
#        nvidia/595.91.07, 6.11.0-29-generic, x86_64: installed (Differences between built and installed modules)
#        nvidia/595.91.07, 7.0.0-30-generic, x86_64: installed
# 
# Normally, these should have been automatically removed, you can manually remove them with e.g.
# sudo dkms remove nvidia/595.91.07 -k 6.11.0-29-generic


# update initramfs and grub
sudo update-initramfs -u -k all
sudo update-grub

reboot

# Final check
sudo dkms status
uname -r

```



