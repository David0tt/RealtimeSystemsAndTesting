The following script describes the commands to install a lowlatency system, which for example can be used to control a Franka Emika Panda robot arm with all control functionality being containerized in docker containers. 

Dockerfiles for this can be found in https://github.com/David0tt/ThesisInformation


```bash
# low-latency kernel
# sudo apt install linux-lowlatency-6.11
# sudo apt install linux-lowlatency-6.14
sudo apt install linux-lowlatency-6.17

# NVIDIA Drivers: (following https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/index.html -> Ubuntu -> Network Repository Installation)
sudo apt install linux-headers-$(uname -r)
export distro="ubuntu2404"
export arch="x86_64"
wget https://developer.download.nvidia.com/compute/cuda/repos/$distro/$arch/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
# It is recommended to pin the driver to a specific version, so on normal system update no newer nvidia driver versions will be installed, which often can lead to broken systems
sudo apt install nvidia-driver-pinning-590
sudo apt install nvidia-open

# Install CUDA (following https://developer.nvidia.com/cuda-13-1-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_network)
# (or select a different version)
sudo apt-get -y install cuda-toolkit-13-1

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
