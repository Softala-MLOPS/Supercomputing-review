# Tutorial for integrated distributed computing in MLOps (5/10)

This jupyter notebook goes through the basic ideas with practical examples for integrating
distributed computing systems for MLOps systems.

## Index

- [SSH Config](#SSH-Config)
- [CPouta Openstack](#CPouta-Openstack)
- [OSS Platform Setup](#OSS-Platform-Setup)
- [Kubernetes in Docker](#Kubernetes-In-Docker)

## SSH Config

### Useful material

- [SSH simplified using SSH Config](https://blog.tarkalabs.com/ssh-simplified-using-ssh-config-161406ba75d7)

### Info

To enable easier manual use of SSH, we recommend creating .ssh/Config files in your local machines. You can edit them with the following:

```
cd .ssh
nano config
```

There we recommend having the following block:

```
Host cpouta
Hostname (your_vm_floating_ip)
User (your_vm_user)
IdentityFile ~/.ssh/local-cpouta.pem
```

Be aware that later SSH commands will use different host names, such as:

```
Host GPU-cpouta
Hostname (your_vm_floating_ip)
User (your_vm_user)
IdentityFile ~/.ssh/local-cpouta.pem
```

## CPouta Openstack

### Useful material

- [CSC Accounts](https://docs.csc.fi/accounts/)
- [Creating virtual machine in Pouta](https://docs.csc.fi/cloud/pouta/launch-vm-from-web-gui/)
- [Connecting to your virtual machine](https://docs.csc.fi/cloud/pouta/connecting-to-vm/)
- [Virtual machine flavors and Billing Unit rates](https://docs.csc.fi/cloud/pouta/vm-flavors-and-billing/)
- [Managing service quotas](https://research.csc.fi/resources/quotas/)
- [Ip-lookup](https://nordvpn.com/fi/ip-lookup/)
- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Linux post-installation steps for Docker Engine](https://docs.docker.com/engine/install/linux-postinstall/)
- [Persistent volumes on virtual machine](https://docs.csc.fi/cloud/pouta/persistent-volumes/)
- [How to increase Docker Disk Memory](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/documentation/docker-storage.md)
- [Installing CUDA on Ubuntu](https://forums.developer.nvidia.com/t/installing-cuda-on-ubuntu-22-04-rxt4080-laptop/292899)
- [Installing the NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [Adding GPU support to kind](https://jacobtomlinson.dev/posts/2022/quick-hack-adding-gpu-support-to-kind/)
- [Running kind clusters with GPUs using nvkind](https://github.com/NVIDIA/nvkind)
- [Kind with GPUs](https://web.archive.org/web/20240415022820/https://www.substratus.ai/blog/kind-with-gpus)
- [How to setup GPUs for containers](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/documentation/gpu-setup.md)

### Info

To enable a constant reserve of computing resources, we will use the Center of Scientific Computing (CSC) CPouta cloud platform to run an open-source MLOps platfrom. To begin, you must have a MyCSC account that is in a project with access to CPouta and billing units. When you have logged in to the CPouta Dashboard and selected the suitable project, follow the CSC documentation to create a suitable ubuntu virtual machine. For our use case the default VM image choice is ubuntu 22.04 or newer. Our use case additionally benefits from having cloud GPUs that you can aquire by requesting quota increase from service desk and running atleast gpu.1.1.gpu flavor.

This isn't however guranteed due to GPUs being a rare resource, which is why we recommend using standard.xxlarge flavor to setup a basic platform. When you have reached setting up a SSH security group rule for your own SSH connections, be aware that you might need to recreate that rule as your computer changes their IP address. Use your favorite ip lookup website to confirm if you IP has changed. When you have gained access to your VM, please follow the Docker docs for installing Docker engine and removing the need for Sudo. It might additionally be worthwhile to store docker data in a persistent volume due to the 80GB limit of the flavor and the ability to transfer that volume between VMs and projects. In such cases please follow the CSC docs to attach a volume and run the following commands:

```
# Check root directory
docker info 

# Create a folder in volume
cd /media/volume 
mkdir docker

# Get its path
cd docker
pwd

# Check docker daemon.json
cat /etc/docker/daemon.json

# Shutdown docker
sudo systemctl stop docker
sudo systemctl stop docker.socket
sudo systemctl stop containerd

# Edit to have data-root: '/media/volume/docker'
sudo nano /etc/docker/daemon.json

# Confirm path
cat /etc/docker/daemon.json

# Move docker data
sudo rsync -axPS /var/lib/docker/ /media/volume/docker

# Restart docker
sudo systemctl start docker

# Check docker 
docker info

# Run a hello world
docker run hello-world

# Check file system
df -h
```

If you managed to get resources to run GPU flavors, do the following to setup the necessery drivers, CUDA, container kit, Docker GPU, Kubernetes NVIDIA GPU operator and NVshare:

```
# Update the enviroment. Press enter, when you get a list
sudo apt update
sudo apt upgrade 

# Check available GPUs
lspci | grep -i nvidia

# Get driver installers
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt install ubuntu-drivers-common
ubuntu-drivers devices

# Install a suitable driver
sudo apt install nvidia-driver-535

# Reboot VM
Sudo reboot

# Confirm drivers
nvidia-smi

# Download and Install CUDA repository pin
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600

# Download CUDA repository Package
wget https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb

# Install CUDA repository package
sudo dpkg -i cuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb

# Install GPG key
sudo cp /var/cuda-repo-ubuntu2204-12-2-local/cuda-216F19BD-keyring.gpg /usr/share/keyrings/

# Update package lists
sudo apt-get update

# Install CUDA toolkit
sudo apt-get -y install cuda

# Set envs
echo 'export PATH=/usr/local/cuda-12.2/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.2/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

# Verify installation
nvidia-smi
nvcc --version

# Configure production repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Update packages
sudo apt-get update

# Install NVIDIA Container Toolkit
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.17.8-1
  sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}

# Configure docker
sudo nvidia-ctk runtime configure --runtime=docker

# Restart docker daemon
sudo systemctl restart docker

# Check daemon.json
cat /etc/docker/daemon.json

# Configure kubernetes
sudo nvidia-ctk runtime configure --runtime=containerd

# Restart containerd
sudo systemctl restart containerd

# Check config.toml
cat /etc/containerd/config.toml

# Check container toolkit
dpkg -l | grep nvidia-container-toolkit

# Test docker GPU
docker run --rm -it --gpus=all nvcr.io/nvidia/k8s/cuda-sample:nbody nbody -gpu -benchmark

# Configure daemon.json and config.toml
sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo systemctl restart docker
sudo sed -i '/accept-nvidia-visible-devices-as-volume-mounts/c\accept-nvidia-visible-devices-as-volume-mounts = true' /etc/nvidia-container-runtime/config.toml
```

## OSS Platform Setup

### Useful material

- [NVIDIA device plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin)
- [Installing the NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html)
- [Deploy on Kubernetes](https://github.com/grgalex/nvshare?tab=readme-ov-file#deploy_k8s)
- [How to setup GPUs for containers](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/documentation/gpu-setup.md)

### Info

To enable easier MLOps development, we will now setup the OSS MLOps platform. Regardless if you have GPU access, run the setup with

```
git clone https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform.git
cd multi-cloud-hpc-oss-mlops-platform
git checkout studying
./integration-setup.sh
```

You are then given options, where the recommendation for development is:

   - Setup integration = y
   - Please choose deployment option = 4
   - Install local Docker registry = y
   - Setup GPU cluster = n (y if you have GPUs)
   - Install Ray = n

When the setup is complete (can take 10-15 mins), you can check the available pods with

```
kubectl get pods -A
```

When you no longer see error or container creating status in any pod, you can test using GPUs with the following comamnds:

```
kubectl apply -f https://raw.githubusercontent.com/grgalex/nvshare/main/tests/kubernetes/manifests/nvshare-tf-pod-1.yaml && \
kubectl apply -f https://raw.githubusercontent.com/grgalex/nvshare/main/tests/kubernetes/manifests/nvshare-tf-pod-2.yaml
```

Check that they are running with

```
kubectl get pods
kubectl logs nvshare-tf-matmul-1 -f
kubectl logs nvshare-tf-matmul-2 -f
```

You can also check the GPU utilization with

```
nvidia-smi
```

When the pods show complete, you can get the pod logs with:

```
kubectl logs nvshare-tf-matmul-1 -n default
kubectl logs nvshare-tf-matmul-2 -n default
```

They should show the following last lines:

```
nvshare-tf-matmul-1
PASS
--- 238.66402554512024 seconds ---
nvshare-tf-matmul-2
PASS
--- 218.99518394470215 seconds ---
```

With NVIDIA GPU operator and NVshare you can now use the same GPU in 10 pods by giving the following configuration in their YAMLs:

```
resources:
  limits:
    nvshare.com/gpu: 1
```

It is recommended to record the used NVIDIA driver, CUDA, container toolkit, Operator and NVshare versions to make future debugging easier with the following commands:

```
nvidia-smi
NVIDIA-SMI 535.247.01             Driver Version: 535.247.01   CUDA Version: 12.2

nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Tue_Jun_13_19:16:58_PDT_2023
Cuda compilation tools, release 12.2, V12.2.91
Build cuda_12.2.r12.2/compiler.32965470_0

dpkg -l | grep nvidia-container-toolkit
ii  nvidia-container-toolkit         1.17.8-1                                amd64        NVIDIA Container toolkit
ii  nvidia-container-toolkit-base    1.17.8-1                                amd64        NVIDIA Container Toolkit Base

kubectl get pods -n gpu-operator
kubectl -n gpu-operator describe pod gpu-operator-(fill) | grep "Image:"
    Image:         nvcr.io/nvidia/gpu-operator:v24.9.0

kubectl -n nvshare-system describe pod nvshare-scheduler-s649x  | grep "Image:"
    Image:          docker.io/grgalex/nvshare:nvshare-scheduler-v0.1-8c2f5b90
```

Finally, you can check the used and available cluster resources with:

```
kubectl describe nodes
```

This shows the following resources for me:

```
Resource           Requests         Limits
  --------           --------         ------
  cpu                5025m (35%)      26700m (190%)
  memory             9339495680 (7%)  22736Mi (19%)
  ephemeral-storage  0 (0%)           0 (0%)
  hugepages-1Gi      0 (0%)           0 (0%)
  hugepages-2Mi      0 (0%)           0 (0%)
  nvidia.com/gpu     1                1
  nvshare.com/gpu    0                0
```

## Kubernetes in Docker
