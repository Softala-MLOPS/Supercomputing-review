# Tutorial for integrated distributed computing in MLOps (5/10)

This jupyter notebook goes through the basic ideas with practical examples for integrating
distributed computing systems for MLOps systems.

## Index

- [SSH Config](#SSH-Config)
- [CPouta Openstack](#CPouta-Openstack)
- [OSS Platform Setup](#OSS-Platform-Setup)
- [Kubernetes in Docker](#Kubernetes-In-Docker)
- [Helm Deployment](#Helm-Deployment)
- [YAML Deployment](#YAML-Deployment)

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

### Useful material

- [Kind](https://kind.sigs.k8s.io/)
- [Inception](https://guide.bash.academy/inception/)
- [Kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [Kind Configuration](https://kind.sigs.k8s.io/docs/user/configuration/)
- [Kind Ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
- [Nodeport](https://www.tkng.io/services/nodeport/)
- [?](https://web.archive.org/web/20240415022820/)
- [Kind with GPUs](https://www.substratus.ai/blog/kind-with-gpus)

The OSS MLOps platform uses kubernetes in docker (KinD), which enables portable utilization of Kubernetes in both local and cloud enviroments. KinD operates differently from regular Kubernetes due to the cluster being run in a Docker container, which makes interactive networking a bit more difficult. In return for these drawbacks KinD speeds up developemnt drastically in our use case, since we can automate the creation of clusters via unified Bash scripts and YAML deployment modifications as long as the host computer has enough resources to run the cluster. The most important things for OSS platform setup are:

   - Setup modificaton
   - Script addition
   - Kind configuration
   - ExtraPortMapping
   - ExtraMounts

When you run integration-setup.sh, it does the following things:

   1. Loads configration
   2. Gives you options
   3. Checks available resources
   4. Checks OS
   5. Runs setup scripts
   6. Applies YAMls
   7. Installs Helm if needed
   8. Installs GPU operator and nvshare if needed
   9. Installs Ray if needed
   10. Runs tests

From these areas you only need to modify 2, 5 and 7-10 to enable the necessery cluster setup by either modifying the setup code, modifying the deployment YAMLs or creating new scripts to run. The addition of new scripts is quite simple with only needing the following block in the beginning:

```
#!/bin/bash

set -eo pipefail
```

Usually the code you need to add after this block is the same code that you would run in a terminal to setup what you need. The exceptions are cases, where you need to install new software into the host system, but we will ignore those until there is actual need to use software outside of docker, kubectl and helm. For our use case the most important script modification is done at scripts/create_(gpu)_cluster.sh, because they enable modifying how the KinD cluster operates during runtime. For us the important configurations are ExtraPortMapping and ExtraMounts, where the former enables opening ports to the KinD Docker container and the latter enables using GPUs. An ExtraPortMapping is defined as:

```
nodes:
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
    - containerPort: 31001
      hostPort: 7001 
      protocol: TCP
```

Here ContainerPort is a NodePort used in Kubernetes and hostPort is the port opened by Docker. Be aware that ContainerPort should ideally be in the range 30000-32767 to enable connections to applications. A ExtraMount is defined as follows:

```
nodes:
    extraMounts:
        - hostPath: /dev/null
          containerPath: /var/run/nvidia-container-devices/all
```

In this case we enable kubernetes to use NVIDIA GPUs that Docker can access. In both cases be aware that you always need to delete the whole cluster to make changes to take effect, which is why one should think ahead about the configuration they need and confirm that they don't need any of the data stored by the cluster.

## Helm Deployment

### Useful material

- [Helm documentation](https://helm.sh/docs/)
- [RayCluster Quickstart](https://docs.ray.io/en/latest/cluster/kubernetes/getting-started/raycluster-quick-start.html#kuberay-raycluster-quickstart)
- [Ray Docker image](https://hub.docker.com/r/rayproject/ray)

### Info

The general idea of adding new applications into the OSS MLOps platform is to check if its easier to do it with a helm. In the case of Ray, we only need to use available helm charts with value replacement to get the cluster we want. For example, we get GPU Ray by creating this YAML file

```
# Create file
nano gpu-kuberay-cluster-values.yaml

# Change these values
image:
  repository: rayproject/ray
  tag: 2.38.0-py312-gpu
  pullPolicy: IfNotPresent

head:
  resources:
    limits:
      cpu: "3"
      memory: "30G"
      nvshare.com/gpu: "1"
    requests:
      cpu: "3"
      memory: "30G"
  volumes:
    - name: log-volume
      emptyDir: {}
  volumeMounts:
    - mountPath: /tmp/ray
      name: log-volume
 
worker:
  groupName: worker
  replicas: 1
  minReplicas: 1
  maxReplicas: 2
  resources:
    limits:
      cpu: "3"
      memory: "30G"
      nvshare.com/gpu: "1"
    requests:
      cpu: "3"
      memory: "30G"
  volumes:
    - name: log-volume
      emptyDir: {}
  volumeMounts:
    - mountPath: /tmp/ray
      name: log-volume
      
# Save the file
CTRL + X 
Y
```

and giving it to the helm to replace default values

```
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update
helm install kuberay-operator kuberay/kuberay-operator --version 1.0.0
helm install raycluster kuberay/ray-cluster --version 1.0.0 -f gpu-kuberay-cluster-values.yaml
```

In the same way you can make Ray only use CPUs with the following YAML

```
cpu-kuberay-cluster-values.yaml

image:
  repository: rayproject/ray
  tag: 2.44.1-py312
  pullPolicy: IfNotPresent

head:
  resources:
    limits:
      cpu: "1"
      memory: "10G"
    requests:
      cpu: "1"
      memory: "10G"
  volumes:
    - name: log-volume
      emptyDir: {}
  volumeMounts:
    - mountPath: /tmp/ray
      name: log-volume
 
worker:
  groupName: worker
  replicas: 1
  minReplicas: 1
  maxReplicas: 1
  resources:
    limits:
      cpu: "1"
      memory: "10G"
    requests:
      cpu: "1"
      memory: "10G"
  volumes:
    - name: log-volume
      emptyDir: {}
  volumeMounts:
    - mountPath: /tmp/ray
      name: log-volume
```

In both cases the critical cases are confirming that the Ray DockerHub images are suitable and that there are enough resources for Ray to use. If a pod requests too many resources, Kubernetes will keep it in a pending state until either there is enough resources or the requested amount is made smaller. You can start again by uninstalling:

```
helm list -A
helm uninstall raycluster
helm uninstall kuberay-operator
```

## YAML Deployment

### Useful material

- [Kustomize](https://kustomize.io/)
- [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Using Apache Airflow with Kubernetes](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/kubernetes.html)
- [Helm Chart for Apache Airflow](https://airflow.apache.org/docs/helm-chart/stable/index.html)
- [Configuring External Database](https://github.com/airflow-helm/charts/blob/main/charts/airflow/docs/faq/database/external-database.md)
- [Temporary fix to Bitnami psql chart licensing issues](https://github.com/apache/airflow/pull/55820)
- [Remove postgres subcharts from Airflow Helm Charts](https://github.com/apache/airflow/issues/55823)
- [Deployment fails with postgresql.enabled=true due to Bitnami image deprecation](https://github.com/apache/airflow/issues/56322)
- [Extending the Chart](https://airflow.apache.org/docs/helm-chart/stable/extending-the-chart.html)
- [Production Guide](https://airflow.apache.org/docs/helm-chart/stable/production-guide.html)
- [Unable to Connect to External Postgres with Airflow Community Helm Chart](https://stackoverflow.com/questions/75606678/unable-to-connect-to-external-postgres-with-airflow-community-helm-chart)
- [Getting started with Airflow: Deploying your first pipeline on Kubernetes](https://medium.com/@rupertarup/getting-started-with-airflow-deploying-your-first-pipeline-on-kubernetes-0014495e6c92)
- [Kubernetes Executor](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html)
- [Airflow with Git-Sync in Kubernetes](https://blog.devgenius.io/airflow-with-git-sync-in-kubernetes-0690460f67a5)
- [SSH Tunneling](https://www.ssh.com/academy/ssh/tunneling)
- [Manage DAGs files](https://airflow.apache.org/docs/helm-chart/1.5.0/manage-dags-files.html)
- [Apache Airflow Docker image](https://hub.docker.com/r/apache/airflow/tags?name=3.0.6)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Kubectl Apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubectl Delete](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_delete/)

### Info

In the cases where you don't have a suitable helm chart such as your own applicatons, you need to define the necessery YAML files in the repository deployment folder, define the kustomize files in the deployment/envs and possibly change the area in the integration setup script. To understand how this is done in practice, we will now deploy the Forwarder redis, frontend, backend, monitor and airflow. We will also deploy applications to handle LLM infernece and various storage applications to provide cache, graph, object, relational, structures and vector databases.

The forwarder deployment begins with image creation. We first need to check that everything works correctly by doing the following:

```
# activate redis
cd applications
docker compose -f redis.yaml up

# activate forwarder frontend
cd applications/forwarder/fastapi
python3 -m venv front_venv
source front_venv/bin/activate
pip install -r packages.txt
python3 run_fastapi.py

# activate forwarder backend
cd applications/forwarder/celery
python3 -m venv back_venv
source back_venv/bin/activate
pip install -r packages.txt
python3 run_celery.py

# activate forwarder monitor
cd applications/forwarder/flower
python3 -m venv monitor_venv
source monitor_venv/bin/activate
pip install -r packages.txt
python3 run_flower.py

cd applications/forwarder/beat
python3 -m venv scheduler_venv
source scheduler_venv/bin/activate
pip install -r packages.txt
python3 run_beat.py
```

When we have confirmed that there are no errors, we then need to comment the enviromental variables in the run files to prevent them from overriding the enviromental variables set by the docker image:

```
frontend
ENV REDIS_ENDPOINT=127.0.0.1
ENV REDIS_PORT=6379
ENV REDIS_DB=0

backend
ENV REDIS_ENDPOINT=127.0.0.1
ENV REDIS_PORT=6379
ENV REDIS_DB=0

ENV CELERY_CONCURRENCY=8
ENV CELERY_LOGLEVEL=info

ENV FLOWER_ENDPOINT=127.0.0.1
ENV FLOWER_PORT=7601
ENV FLOWER_USERNAME=flower123
ENV FLOWER_PASSWORD=flower456

ENV PROMETHEUS_PORT=7602 

ENV AIRFLOW_HOST=http://localhost:8090
ENV AIRFLOW_USERNAME=admin
ENV AIRFLOW_PASSWORD=admin

monitor
ENV REDIS_ENDPOINT=127.0.0.1
ENV REDIS_PORT=6379
ENV REDIS_DB=0

ENV FLOWER_USERNAME=flower123
ENV FLOWER_PASSWORD=flower456

scheduler
ENV REDIS_ENDPOINT=127.0.0.1
ENV REDIS_PORT=6379
ENV REDIS_DB=0

ENV SCHEDULER_TIMES=30
```

Besides enviromental variables, you should always check the ports you use and ports you have exposed in the image, since there can easily be a mismatch. In our case these are:

```
frontend
uvicorn.run(
    app = fastapi, 
    host = '0.0.0.0', 
    port = 7600
)
EXPOSE 7600

backend

ENV PROMETHEUS_PORT=7602 
start_http_server(
    port = wanted_port, 
    registry = global_registry
) 
EXPOSE 7602

monitor
endpoint = '0.0.0.0'
port = '7601'
flower.start(argv = ['flower', used_address, used_port, basic_authentication])
EXPOSE 7601
```

When you have confirmed that both code and image make sense, we can build images and send them to dockerhub:

```
cd applications/forwarder/fastapi
docker build -t (your repository):study_forwarder_frontend_v0.0.5 .
docker push (your repository):study_forwarder_frontend_v0.0.5

cd applications/forwarder/celery
docker build -t (your repository):study_forwarder_backend_v0.0.5 .
docker push (your repository):study_forwarder_backend_v0.0.5

cd applications/forwarder/flower
docker build -t (your repository):study_forwarder_monitor_v0.0.5 .
docker push (your repository):study_forwarder_monitor_v0.0.5

cd applications/forwarder/beat
docker build -t (your repository):study_forwarder_scheduler_v0.0.5 .
docker push (your repository):study_forwarder_flower_v0.0.5
```

Now that the images are in DockerHub, we can now run them in OSS by creating the necessery YAML files. You can find the already provided YAML files for deploying forwarder redis, frontend, backend, monitor and scheduler in deployments/forwarder with stack and scheduler folders. In these folders we find following YAML types:

   - Kustomization -> Defines building configuration
   - Namespace -> Defines the container space
   - Deployment -> Defines the container
   - Service -> Defines the container networking

When we deploy Kubernetes objects we can apply a single file in the following way:

```
cd deployments/forwarder/stack/fastapi
kubectl apply -f fastapi-deployment.yaml
```

This is fine for testing, but it should not be used to apply multiple files. For that we use Kustomize that enables us to apply the whole folder with a kustomization.yaml file in the following way

```
kubectl apply -k fastapi
```

This can be used to construct complex deployment configurations by simply putting kustomization.yamls in fitting places with correct file paths. In the case of this deployment we can apply the whole stack with the following:

```
namespace: forwarder 

resources:
  - forwarder-namespace.yaml
  - redis/
  - fastapi/
  - flower/
  - celery/
  - postgres/
```

Be aware that you should define a namespace before applying other files, which in this case is the first file defined with:

```
apiVersion: v1
kind: Namespace
metadata:
  name: forwarder 
```

The order of deployment should follow the depedency order to ensure that pods have the necessery resources to work properly, which is why redis and frontend are the first pods we deploy in this stack. All of these pods use a deployment and service object. The kubernetes deployment object for frontend is:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-server
  labels:
    app: fastapi-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi-server
  template:
    metadata:
      labels:
        app: fastapi-server
    spec:
      containers:
        - name: frontend-server
          image: t9k4b5ndjok1/multi-local-cloud-hpc-integration:study_forwarder_frontend_v0.0.5
          imagePullPolicy: Always
          ports:
            - containerPort: 7600
          env:
          - name: REDIS_ENDPOINT
            value: 'redis-service.forwarder.svc.cluster.local'
          - name: REDIS_PORT
            value: '6379'
          - name: REDIS_DB 
            value: '0'
```

In this YAML the critical things are the used names, image, containerPort and set enviromental variables. The service object for frontend is:

```
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector: 
    app: fastapi-server
  ports:
    - port: 7600
      targetPort: 7600
```

In this YAML the critical things are having matching names and port numbers. Both can be deployed at the same time with the following kustomization:

```
resources:
- fastapi-deployment.yaml
- fastapi-service.yaml
```

Be aware that nested kustomization are put into the same namespace as the activator unless specified. Now, provided that you have studying branch in your VM, you can make the stack run with

```
cd tutorials/integration/studying/deployments/forwarder
kubectl apply -k stack
```

This will cause the older forwarder pods to be replaced by newer ones that you can check with:

```
kubectl get pods -n forwarder
```

You can check that everything is running as indended by checking logs:

```
kubectl logs fastapi-server-(id) -n forwarder
kubectl logs celery-server-(id) -n forwarder
kubectl logs flower-server-(id) -n forwarder
kubectl logs redis-server-(id)-n forwarder
```

The first three should show the following logs:

```
fastapi
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:7600 (Press CTRL+C to quit)
celery
 
 -------------- celery@celery-server-8489d89f85-c6kwp v5.5.3 (immunity)
--- ***** ----- 
-- ******* ---- Linux-5.15.0-151-generic-x86_64-with-glibc2.36 2025-09-30 13:14:54
- *** --- * --- 
- ** ---------- [config]
- ** ---------- .> app:         tasks:0x7f584b5e3e50
- ** ---------- .> transport:   redis://redis-service.forwarder.svc.cluster.local:6379/0
- ** ---------- .> results:     redis://redis-service.forwarder.svc.cluster.local:6379/0
- *** --- * --- .> concurrency: 8 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** ----- 
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery
                

[tasks]
  . tasks.forwarder-requests
  . tasks.forwarder-trigger

monitor
[I 250930 13:14:42 command:168] Visit me at http://0.0.0.0:7601
[I 250930 13:14:42 command:176] Broker: redis://redis-service.forwarder.svc.cluster.local:6379/0
[I 250930 13:14:42 command:177] Registered tasks: 
    ['celery.accumulate',
     'celery.backend_cleanup',
     'celery.chain',
     'celery.chord',
     'celery.chord_unlock',
     'celery.chunks',
     'celery.group',
     'celery.map',
     'celery.starmap']
[I 250930 13:14:42 mixins:228] Connected to redis://redis-service.forwarder.svc.cluster.local:6379/0
[E 250930 13:15:11 events:191] Failed to capture events: 'Error 111 connecting to redis-service.forwarder.svc.cluster.local:6379. Connection refused.', trying again in 1 seconds.
[I 250930 13:15:12 mixins:228] Connected to redis://redis-service.forwarder.svc.cluster.local:6379/0
```

You can further confirm that everything works by port forwarding frontend and monitor:

```
# Check services
kubectl get service -n forwarder

# Get frontend
ssh -L 7600:localhost:7600 cpouta
kubectl port-forward svc/fastapi-service 7600:7600 -n forwarder

# Get monitor
ssh -L 7601:localhost:7601 cpouta
kubectl port-forward svc/flower-service 7601:7601 -n forwarder
```

To check the pages go to http://localhost:7600/docs and http://localhost:7601/. We can interact with forwarder using the following functions:

In [ ]:

```
import json
import requests

def get_forwarder_route(
    address: str,
    port: str,
    route_type: str,
    route_name: str,
    path_replacers: any,
    path_names: any
): 
    url_prefix = 'http://' + address + ':' + port

    routes = {
        'setup': 'POST:/setup/config/mode',
        'task': 'GET:/interaction/task/type/identity',
        'orch': 'POST:/interaction/orch/type/key',
        'forw': 'POST:/interaction/forw/type/user/key',
        'arti': 'GET:/interaction/arti/type/target'
    }

    route = None
    if route_name in routes:
        i = 0
        route = routes[route_name].split('/')
        for name in route:
            if name in path_replacers:
                replacer = path_replacers[name]
                if 0 < len(replacer):
                    route[i] = replacer
            i = i + 1

        if not len(path_names) == 0:
            route.extend(path_names)

        if not len(route_type) == 0:
            route[0] = route_type + ':'
        
        route = '/'.join(route)
    print('Used route: ' + str(route))
    route_split = route.split(':')
    url_type = route_split[0]
    used_path = route_split[1]
    full_url = url_prefix + used_path
    return url_type, full_url

def request_forwarder_route(
    address: str,
    port: str,
    route_type: str,
    route_name: str,
    path_replacers: any,
    path_names: any,
    route_input: any,
    timeout: any
) -> any:
    url_type, full_url = get_forwarder_route(
        address = address,
        port = port,
        route_type = route_type,
        route_name = route_name,
        path_replacers = path_replacers,
        path_names = path_names
    )

    if url_type == 'POST':
        route_response = requests.post(
            url = full_url,
            json = route_input
        )
    if url_type == 'GET':
        route_response = requests.get(
            url = full_url,
            json = route_input
        )

    route_status_code = None
    route_returned_text = {}
    if not route_response is None:
        route_status_code = route_response.status_code
        if route_status_code == 200:
            route_returned_text = json.loads(route_response.text)
    return route_status_code, route_returned_text
```

This setups the forwarder:

In [ ]:

```
route_code, route_text = request_forwarder_route(
    address = '127.0.0.1',
    port = '7600',
    route_type = '',
    route_name = 'setup',
    path_replacers = {
        'mode': 'init'
    },
    path_names = [],
    route_input = {
        'bucket-prefix': 'none',
        'ice-id': 'none',
        'user': 'none',
        'used-client': 'none',
        'pre-auth-url': 'none',
        'pre-auth-token': 'none',
        'user-domain-name': 'none',
        'project-domain-name': 'none',
        'project-name': 'none',
        'auth-version': 'none'
    },
    timeout = 240
)
print(route_code)
print(route_text)
```

```
Used route: POST:/setup/config/init
200
{'output': None}
```

We again get logs either through the frontend http://localhost:7600/docs or running the following:

In [ ]:

```
route_code, route_text = request_forwarder_route(
    address = '127.0.0.1',
    port = '7600',
    route_type = '',
    route_name = 'arti',
    path_replacers = {
        'type': 'logs',
        'target': 'frontend'
    },
    path_names = [],
    route_input = {},
    timeout = 240
)
print(route_code)
print(route_text)
```

```
Used route: GET:/interaction/arti/logs/frontend
200
{'output': {'logs': ['2025-09-30 13:31:39,009 - INFO - forwader-frontend - Creating FastAPI instance', '2025-09-30 13:31:39,009 - INFO - forwader-frontend - FastAPI created', '2025-09-30 13:31:39,009 - INFO - forwader-frontend - Connecting to redis', '2025-09-30 13:31:39,029 - INFO - forwader-frontend - Redis connected', '2025-09-30 13:31:39,029 - INFO - forwader-frontend - Defining Celery client', '2025-09-30 13:31:39,052 - INFO - forwader-frontend - Celery client defined', '2025-09-30 13:31:39,052 - INFO - forwader-frontend - Importing routes', '2025-09-30 13:31:39,076 - INFO - forwader-frontend - Including routers', '2025-09-30 13:31:39,078 - INFO - forwader-frontend - Routes implemented', '2025-09-30 13:31:39,078 - INFO - forwader-frontend - Frontend ready', '2025-09-30 13:31:53,917 - INFO - forwader-frontend - Artifact interaction']}}
```

In [ ]:

```
route_code, route_text = request_forwarder_route(
    address = '127.0.0.1',
    port = '7600',
    route_type = '',
    route_name = 'arti',
    path_replacers = {
        'type': 'logs',
        'target': 'backend'
    },
    path_names = [],
    route_input = {},
    timeout = 240
)
print(route_code)
print(route_text)
```

```
Used route: GET:/interaction/arti/logs/backend
200
{'output': {'logs': ['[2025-09-30 13:45:33,236: INFO/MainProcess] Connected to redis://127.0.0.1:6379/0', '[2025-09-30 13:45:33,244: INFO/MainProcess] mingle: searching for neighbors', '[2025-09-30 13:45:34,257: INFO/MainProcess] mingle: all alone', '[2025-09-30 13:45:34,275: INFO/MainProcess] celery@lx4-500-31309.ad.helsinki.fi ready.', '[2025-09-30 13:45:36,791: INFO/MainProcess] Events of group {task} enabled by remote.', '[2025-09-30 13:46:15,625: INFO/MainProcess] Task tasks.forwarder-requests[01b1db6d-b72e-4a90-a519-a36ed02dc75e] received', '[2025-09-30 13:46:15,627: WARNING/ForkPoolWorker-7] forwarder requests', '[2025-09-30 13:46:15,628: WARNING/ForkPoolWorker-7] Client', '[2025-09-30 13:46:15,628: WARNING/ForkPoolWorker-7] getting lock', '[2025-09-30 13:46:15,631: WARNING/ForkPoolWorker-7] False']}}
```

Be aware that if you see errors related to connecting to redis, these are caused by the fact that Redis reaches the running state later than the lighter backend and monitor. If you want to clean up these logs, you can do it by deleting the celery and flower pods in the following way:

```
kubectl delete pod celery-server-(id) -n forwarder
kubectl delete pod flower-server-(id) -n forwarder
```

The deletion of a pod caused kubernetes to recreating using a existing deployment, which you can check with:

```
kubectl get deployments -n forwarder
```

As we now see, the deletion of containers should be the primary way of fixing possible issues that happen during application use such as redudant logs. This additionally demonstrates the practical benefits of microservice architectures and modularity, since we didn't need to touch redis or frontend to handle problems in backend and monitor. Be aware that if forwarder for some reason is unable to give logs, you can check them by going inside the pods with:

```
kubectl exec -it celery-server-(id) -n forwarder -- /bin/bash
cd logs
cat backend.log
exit
```

We can now make the scheduler run by deploying it:

```
kubectl apply -k scheduler
```

Be aware that the scheduler will not give any logs, which is why you should instead check http://localhost:7601/ and http://localhost:8090/dags. When you have confirmed that it works properly, you can stop it by deleting the deployment with:

```
kubectl delete -k scheduler
```

Here we see the reason for separating the scheduler. By interacting with Kubernetes through kubectl and kustomize using a suitable YAMLs, we can start and stop the interval loop whenever we want, enabling us to keep forwarder static when we don't need automation. We will discuss this further in future parts. To deploy the forwarder airflow, we need to create its image and push it into dockerhub:

```
cd applications/forwarder/airflow
docker build -t (your repository):study_forwarder_airflow_v3.0.6 .
docker push (your repository):study_forwarder_airflow_v3.0.6
```

We can now deploy forwarder airflow by doing the following:

```
cd stack/airflow
helm upgrade --install airflow apache-airflow/airflow --namespace forwarder -f values.yaml
```

When this is complete after few minutes, you should get the following:

```
Release "airflow" does not exist. Installing it now.
NAME: airflow
LAST DEPLOYED: Mon Nov 17 12:45:59 2025
NAMESPACE: forwarder
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing Apache Airflow 3.0.6!

Your release is named airflow.
You can now access your dashboard(s) by executing the following command(s) and visiting the corresponding port at localhost in your browser:
Airflow API Server:     kubectl port-forward svc/airflow-api-server 8080:8080 --namespace forwarder
Default Webserver (Airflow UI) Login credentials:
    username: admin
    password: admin

You can get Fernet Key value by running the following:

    echo Fernet Key: $(kubectl get secret --namespace forwarder airflow-fernet-key -o jsonpath="{.data.fernet-key}" | base64 --decode)

###########################################################
#  WARNING: You should set a static webserver secret key  #
###########################################################

You are using a dynamically generated webserver secret key, which can lead to
unnecessary restarts of your Airflow components.

Information on how to set a static webserver secret key can be found here:
https://airflow.apache.org/docs/helm-chart/stable/production-guide.html#webserver-secret-key
```

We can get access to the Airflow dashboard with a SSH local forward and a kubernetes port forward:

```
ssh -L 8090:localhost:8080 cpouta
kubectl port-forward svc/airflow-api-server 8080:8080 --namespace forwarder
```

To enable easy package changes, development of DAGs and external database, we changed the following in values.yaml:

   - defaultAirflowRepository = t9k4b5ndjok1/multi-local-cloud-hpc-integration
   - defaultAirflowTag = "study_forwarder_airflow_v3.0.6"
   - airflowVersion = "3.0.6"
   - gitSync.Enabled = True
   - gitSync.repo = https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform.git
   - gitSync.branch = studying
   - gitSync.ref = studying
   - gitSync.wait = 15
   - postgresql.enabled = False

We can create a custom airflow image and send it to DockerHub

```
cd applications/forwarder/airflow
docker build -t (your repository):study_forwarder_airflow_v3.0.6 .
docker push (your repository):study_forwarder_airflow_v3.0.6
```

You can then update the deployment with:

```
helm upgrade airflow apache-airflow/airflow -n forwader -f updated-values.yaml
```

When you have confirmed that the pods run, open the UI with localforward to check http://localhost:8080/dags, which should show the tutorial-hello-world DAG. Be aware that you need to give airflow a GitHub token if your repository is private. It should show the following log:

```
Log message source details: sources=["http://airflow-worker-0.airflow-worker.airflow.svc.cluster.local:8793/log/dag_id=tutorial-hello-world/run_id=manual__2025-09-29T09:58:58.245270+00:00/task_id=hello_world/attempt=1.log"]
Log message source details: sources=["http://airflow-worker-0.airflow-worker.airflow.svc.cluster.local:8793/log/dag_id=tutorial-hello-world/run_id=manual__2025-09-29T09:58:58.245270+00:00/task_id=hello_world/attempt=1.log"]
[2025-09-29, 12:59:00] INFO - DAG bundles loaded: dags-folder: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-29, 12:59:00] INFO - Filling up the DagBag from /opt/airflow/dags/repo/tutorials/integration/studying/applications/forwarder/airflow/dags/hello_world_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-29, 12:59:00] INFO - airflow: chan="stdout": source="task"
[2025-09-29, 12:59:00] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperato
```

With all this setup, we have enabled interoperable development enviroment, where we can create and test DAGs locally before testing and running them in CPouta OSS. Be aware that if you do changes to the DAGs of your own repositories, it will take around 15 seconds for the dag processor to notice them. You can check processor logs with:

```
kubectl get pods -n airflow
kubectl logs airflow-dag-processor-(id) -n airflow
```

