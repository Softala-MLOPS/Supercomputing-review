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
- [Application Deployment](#Application-Deployment)
- [Istio Networking](#Istio-Networking)

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

## Application Deployment

### Useful material

- [Redis](https://github.com/redis/redis)
- [Redis Docker Image](https://hub.docker.com/layers/library/redis/8.2.1/images/sha256-94d44595f2835e84a865267cff718799849c4feae6bec51794c12ed3fe4fc28b)
- [Minio](https://github.com/minio/minio)
- [Minio Docker Image](https://hub.docker.com/layers/minio/minio/RELEASE.2025-09-07T16-13-09Z/images/sha256-a1a8bd4ac40ad7881a245bab97323e18f971e4d4cba2c2007ec1bedd21cbaba2)
- [PostgreSQL Database Management System](https://github.com/postgres/postgres)
- [Postgres Docker Image](https://hub.docker.com/layers/library/postgres/18.0/images/sha256-631eaa51f4ddfafb7494ae4bddb1adc6c54866df0340d94b20d6866c20b20d35)
- [MongoDB](https://github.com/mongodb/mongo)
- [MongoDB Docker Image](https://hub.docker.com/layers/library/mongo/8.0.14/images/sha256-0de701cce332bf7c3b409d91cf2b725c8604c9294ffd00f6329727962105a053)
- [Mongo Express](https://github.com/mongo-express/mongo-express)
- [Qdrant](https://github.com/qdrant/qdrant)
- [Qdrant Docker Image](https://hub.docker.com/layers/qdrant/qdrant/v1.15/images/sha256-53d8c371bc3ae99e4353897ee594bfcfab7f72c4925f74471ebb34d9f6103a3e)
- [Neo4j Graph Database](https://github.com/neo4j/neo4j)
- [Neo4j Docker Image](https://hub.docker.com/layers/library/neo4j/5.26.12-community/images/sha256-a6a4c409d35939597968a5dea1768c92db1787cd6c25614349be5c782ee99e35)
- [Kubernetes ConfiMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Ollama](https://github.com/ollama/ollama)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [Ollama Docker Image](https://hub.docker.com/layers/ollama/ollama/0.12.3/images/sha256-36ccf1b4c161179a198a19e897aa0a1acd54cd5eeb6ab99c80f2ef50ba7fa106)
- [Ollama Search](https://ollama.com/search)

### Info

For our use case we now know all the basic things we need to deploy applications into OSS. As a demonstration we will now add the following applications into OSS:

   - Cache storage with redis
   - Object storage with MinIO
   - Relational storage with PostgreSQL
   - Structured storage with MongoDB
   - Vector storage with Qdrant
   - Graph storage with Neo4j
   - Large language model inference with Ollama
   - Large language model interface with WebUi

To begin go check the deployments/storage. If you open the folders, you see the following new file names:


   - Config.env
   - Secret.env
   - Config
   - PVC

The three first file names related to Kubernetes Config object that enables us to manage non sensitive and sensitive configuration data by keeping it separate from deployment, which enables its easier modification as needed. In this case these files handle user secrets in MinIO and PostgreSQL by using .env files to generate configMap and Secrets as shown in the kustomization.yaml

```
configMapGenerator:
- name: storage-configmap
  envs:
    - config.env

secretGenerator:
- name: storage-secrets
  envs:
    - secret.env
```

These are then utilized in the deployments in the following way:

```
minio-deployment.yaml
- name: MINIO_ROOT_USER
  valueFrom:
    configMapKeyRef:
      name: storage-configmap
      key: MINIO_ROOT_USER
- name: MINIO_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: storage-secrets
      key: MINIO_ROOT_PASSWORD

postgres-deployment.yaml
env:
- name: POSTGRES_PASSWORD 
  valueFrom:
    secretKeyRef:
      name: storage-secrets
      key: DB_PASSWORD
```

In the case of postgresql we also a specified config map to set database name and user

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  labels:
    app: postgres
data:
  POSTGRES_DB: "relationaldb"
  POSTGRES_USER: "relational"
```

This is used in the following way:

```
envFrom:
- configMapRef:
    name: postgres-config
```

The other necessery object is persistent volumes (PVC) that provide requested storage space in the disk the cluster has access to, which in our case is the mounted openstack volume of 500G. PVCs should be provided to databases that require a permanent placement for data, which is why we don't provide Redis a PVC, but give it to Neo4j, MinIO, Postgres, Mongo and Qdrant. Be aware that PVCs are destroyed when the cluster is deleted, which is why one should be careful with OSS deletion or backup data in global storage such as Allas. We define a PVC object in the following way for MongoDB:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-data-pvc
  labels:
    app: mongo-server
spec:
 accessModes:
   - ReadWriteOnce
 resources:
   requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-log-pvc
  labels:
    app: mongo-server
spec:
 accessModes:
   - ReadWriteOnce
 resources:
   requests:
      storage: 5Gi
```

Here we define PVCs both for data and the logs, which we then give to deployment in the following way:

```
spec:
  containers:
    volumeMounts:
    - name: mongo-data
      mountPath: /data/db/
    - name: mongo-log
      mountPath: /var/log/mongodb/
  volumes:
  - name: mongo-data
    persistentVolumeClaim:
      claimName: mongo-data-pvc
  - name: mongo-log
    persistentVolumeClaim:
      claimName: mongo-log-pvc
```

With everything covered, we can now deploy these storages with kustomize:

```
kubectl apply -k storage
```

If you check their pod logs, you should see the following rows:

```
# Redis
1:M 01 Oct 2025 09:54:24.878 * Server initialized
1:M 01 Oct 2025 09:54:24.878 * Ready to accept connections tcp

# Minio
API: http://10.244.0.111:9000  http://127.0.0.1:9000 
WebUI: http://10.244.0.111:9001 http://127.0.0.1:9001    
Docs: https://docs.min.io
WARN: Detected default credentials 'minioadmin:minioadmin', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables

# PostgreSQL
2025-10-01 09:58:54.175 UTC [1] LOG:  starting PostgreSQL 17.2 (Debian 17.2-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
2025-10-01 09:58:54.175 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2025-10-01 09:58:54.175 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2025-10-01 09:58:54.182 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2025-10-01 09:58:54.210 UTC [71] LOG:  database system was shut down at 2025-10-01 09:58:54 UTC
2025-10-01 09:58:54.220 UTC [1] LOG:  database system is ready to accept connections

# MongoDB
MongoDB init process complete; ready for start up.

# Express
No custom config.js found, loading config.default.js
Welcome to mongo-express 1.0.2
------------------------
Mongo Express server listening at http://0.0.0.0:8081
Server is open to allow connections from anyone (0.0.0.0)

# Qdrant
           _                 _    
  __ _  __| |_ __ __ _ _ __ | |_  
 / _` |/ _` | '__/ _` | '_ \| __| 
| (_| | (_| | | | (_| | | | | |_  
 \__, |\__,_|_|  \__,_|_| |_|\__| 
    |_|                           

Version: 1.15.5, build: 48203e41
Access web UI at http://localhost:6333/dashboard
2025-10-01T09:55:51.772871Z  INFO storage::content_manager::consensus::persistent: Initializing new raft state at ./storage/raft_state.json
2025-10-01T09:55:52.624224Z  INFO qdrant: Distributed mode disabled
2025-10-01T09:55:52.624432Z  INFO qdrant: Telemetry reporting enabled, id: 663d9a7b-dc14-4b37-9b98-71ef70b33d82

# Neo4j
2025-10-01 09:56:08.454+0000 INFO  ======== Neo4j 5.26.12 ========
2025-10-01 09:56:17.095+0000 INFO  Anonymous Usage Data is being sent to Neo4j, see https://neo4j.com/docs/usage-data/
2025-10-01 09:56:17.307+0000 INFO  Bolt enabled on 0.0.0.0:7687.
2025-10-01 09:56:19.704+0000 INFO  HTTP enabled on 0.0.0.0:7474.
2025-10-01 09:56:19.705+0000 INFO  Remote interface available at http://localhost:7474/
2025-10-01 09:56:19.708+0000 INFO  id: AD6E1B0E52FDF7233306135CB8D07FFE1AFB0ED9C02AB2BC34721DE6B976DF8F
2025-10-01 09:56:19.708+0000 INFO  name: system
2025-10-01 09:56:19.709+0000 INFO  creationDate: 2025-10-01T09:56:12.868Z
2025-10-01 09:56:19.709+0000 INFO  Started.
```

We can now create local forward to the dashboards of MinIO, Mongo, Qdrant and Neo4j by doing the following:

```
# MinIo backend
ssh -L 9100:localhost:9100 cpouta
kubectl port-forward svc/minio-service 9100:9100 -n storage

# MinIO frontend
ssh -L 9101:localhost:9101 cpouta
kubectl port-forward svc/minio-service 9101:9101 -n storage (user is minioadmin and password is minioadmin)

# Mongo frontend
ssh -L 7200:localhost:7200 cpouta
kubectl port-forward svc/express-service 7200:7200 -n storage
http://localhost:7200 (user is express123 and password is express456)

# Mongo backend
ssh -L 27017:localhost:27017 cpouta
kubectl port-forward svc/mongo-service 27017:27017 -n storage
http://localhost:27017

# Qdrant
ssh -L 7201:localhost:7201 cpouta
kubectl port-forward svc/qdrant-service 7201:7201 -n storage
http://localhost:7201/dashboard (API key is qdrant_key)

# Neo4j frontend
ssh -L 7474:localhost:7474 cpouta
kubectl port-forward svc/neo4j-service 7474:7474 -n storage
http://localhost:7474 (user is neo4j, password is password)

# Neo4j backend (required for interactions)
ssh -L 7687:localhost:7687 cpouta
kubectl port-forward svc/neo4j-service 7687:7687 -n storage
http://localhost:7687
```

We can also interact with redis and postgresql with:

```
# Redis
ssh -L 6379:localhost:6379 cpouta
kubectl port-forward svc/redis-service 6379:6379 -n storage

# Postgresql
ssh -L 5532:localhost:5532 cpouta
kubectl port-forward svc/postgres-service 5532:5532 -n storage
```

All of these backends can now be interacted by using suitable Python clients. This enables near production enviroment development without creating these same containers in Docker Compose that we will discuss in future parts. Now, we will deploy Ollama and WebUI to enable interactive testing of large language models, which you can find at deployments/language. There is nothing new to note besides the large PVC claims made by Ollama to store models and nvshare request. Based on tests the CPouta Tesla P100 16GB can handle models smaller than 20B with some model exceptions. Most of these models vary in 4GB-15GB sizes, which is why we need 40GB to enable swapping between models. In the case of GPU access we will use the mentioned request previousily mentioned in the Ray Helm example to give Ollama access to GPU. If you don't have a GPU, you can simply remove the request to force Ollama into CPU mode. We can again deploy these with:

```
kubectl apply -k language
```

If you check the pod logs, you should get the following rows:

```
# Ollama
[NVSHARE][INFO]: Successfully initialized nvshare GPU
[NVSHARE][INFO]: Client ID = 188e3c163b8cfd4f
time=2025-10-01T11:18:18.978Z level=INFO source=types.go:131 msg="inference compute" id=GPU-7d14824c-991c-23c3-7303-1ef28991f48e library=cuda variant=v12 compute=6.0 driver=12.2 name="Tesla P100-PCIE-16GB" total="15.9 GiB" available="14.4 GiB"
time=2025-10-01T11:18:18.978Z level=INFO source=routes.go:1569 msg="entering low vram mode" "total vram"="15.9 GiB" threshold="20.0 GiB"

# Webui

 ██████╗ ██████╗ ███████╗███╗   ██╗    ██╗    ██╗███████╗██████╗ ██╗   ██╗██╗
██╔═══██╗██╔══██╗██╔════╝████╗  ██║    ██║    ██║██╔════╝██╔══██╗██║   ██║██║
██║   ██║██████╔╝█████╗  ██╔██╗ ██║    ██║ █╗ ██║█████╗  ██████╔╝██║   ██║██║
██║   ██║██╔═══╝ ██╔══╝  ██║╚██╗██║    ██║███╗██║██╔══╝  ██╔══██╗██║   ██║██║
╚██████╔╝██║     ███████╗██║ ╚████║    ╚███╔███╔╝███████╗██████╔╝╚██████╔╝██║
 ╚═════╝ ╚═╝     ╚══════╝╚═╝  ╚═══╝     ╚══╝╚══╝ ╚══════╝╚═════╝  ╚═════╝ ╚═╝


v0.6.32 - building the best AI user interface.

https://github.com/open-webui/open-webui

Fetching 30 files: 100%|██████████| 30/30 [00:17<00:00,  1.72it/s]
INFO:     Started server process [1]
INFO:     Waiting for application startup.
2025-10-01 11:20:11.230 | INFO     | open_webui.utils.logger:start_logger:162 - GLOBAL_LOG_LEVEL: INFO
2025-10-01 11:20:11.230 | INFO     | open_webui.main:lifespan:553 - Installing external dependencies of functions and tools...
2025-10-01 11:20:11.247 | INFO     | open_webui.utils.plugin:install_frontmatter_requirements:283 - No requirements found in frontmatter.
```

You can now localforward these with:

```
# Ollama
ssh -L 7100:localhost:7100 cpouta
kubectl port-forward svc/ollama-service 7100:7100 -n language

# Webui
ssh -L 7101:localhost:7101 cpouta
kubectl port-forward svc/webui-service 7101:7101 -n language
```

When you open http://localhost:7101/ to get into webui, you can start using models with the following clicks

   - Top right gear
   - Menu admin panel
   - Top settings
   - Models
   - Click download top right
   - Go to https://ollama.com/search
   - Get suitable model such as mistral:7b
   - Write it into pull a model
   - Wait for it to download
   - When its downloaded, go to new chat
   - Click top left to select a model
   - Select mistral
   - Write a prompt and press enter

For example, if you write 'can you shortly explain python', you might get the following answer:

```
Python is a high-level, interpreted programming language that was created by Guido van Rossum and first released in 1991. It's known for its clean syntax, readability, and versatility, making it a popular choice among both beginners and experts in various domains such as web development, data analysis, machine learning, artificial intelligence, and scientific computing.

Python is an interpreted language, meaning that the code is not compiled into machine code before execution like some other languages (e.g., C++ or Java). Instead, Python's interpreter translates the code at runtime, which makes it quicker to develop and test applications compared to compiled languages.

Some key features of Python include:

1. Simple syntax: Python has a readable syntax with fewer keywords and fewer lines of code compared to other programming languages. This allows beginners to quickly learn the basics.

2. Large standard library: Python comes with a vast built-in set of modules that handle common programming tasks, such as networking, file I/O, string operations, and more.

3. Extensibility: Python can be extended with C or C++ code for better performance in demanding applications.

4. Open-source and cross-platform: Python is open-source, meaning that its source code is freely available to everyone. It also runs on various platforms including Windows, macOS, Linux, and others.

5. Dynamic typing and automatic memory management: In Python, variables don't have specific data types until they are assigned values during runtime. Additionally, Python takes care of managing memory for you, reducing the risk of memory leaks and simplifying development tasks.

6. Strong support for object-oriented programming (OOP): Python supports OOP principles like inheritance, encapsulation, polymorphism, and abstract classes, making it suitable for developing complex applications.

7. Large community: Python has a large, active community of developers who contribute to the language's growth by creating libraries, frameworks, and tools that help make development more efficient and enjoyable.

Python is an essential tool in many industries, including web development (Django, Flask, Pyramid), data science (NumPy, Pandas, Scikit-learn, TensorFlow, Keras), artificial intelligence (TensorFlow, PyTorch, OpenCV), automation (Robot Framework), and scientific computing (SciPy, Matplotlib).
```

This might take at first few second to minute as GPU warms up. You can check the speed by using webui statistics shown under the answer circle with 'i'. Be aware that different models can handle very differently especially in mininum VRAM consumption. This you can apporximate by using nvidia-smi during generation, which gives the following for mistral 7b:

```
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.247.01             Driver Version: 535.247.01   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla P100-PCIE-16GB           Off | 00000000:00:06.0 Off |                    0 |
| N/A   30C    P0             162W / 250W |   5108MiB / 16384MiB |     96%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A   1417505      C   /usr/bin/ollama                             370MiB |
+---------------------------------------------------------------------------------------+
```

Here we see that this generation took around 5.1GB of VRAM to complete. Without going into details, this amount increases as the prompt and answer length increase. This is difficult to predict, which is why large models usually fail to run due to the token requirements set to VRAM. We will go into further details on future parts.

## Istio Networking

### Useful material

- [Istio](https://istio.io/)
- [Kiali](https://kiali.io/)
- [Kiali Quick Start](https://kiali.io/docs/installation/quick-start/)
- [Istio Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/)
- [Istio Ingress Gateways](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/)
- [Istio Virtual Service](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [The /etc/hosts file](https://tldp.org/LDP/solrhe/Securing-Optimizing-Linux-RH-Edition-v1.3/chap9sec95.html)
- [Curl tutorial](https://curl.se/docs/tutorial.html)

### Info

As we have seen in the previous section, there is a quite a lot of manual actions required to form connections into Kubernetes services. In order to remove the need for local forwards for service access, we will use Istio service mesh to handle outside connections, Kiali to handle network debugging and Openstack security groups to handle restricted access. To begin, we first need to modify OSS istio in the following way:

```
# Get values
kubectl get deployment istiod -n istio-system -o yaml > istiod.yaml

# Modify values
nano istiod.yaml
set ENABLE_DEBUG_ON_HTTP to "true"

# Apply values
kubectl apply -f istiod.yaml
```

When the istio-system is running again, we can now deploy Kiali by using deployments/networking/kiali. Kiali is a dashboard console for Istio that enables to easily see what is happening in the network. Deploy it with:

```
cd deployments/networking
kubectl apply -k kiali
```

The created pod should give the following logs:

```
# Check kiali
kubectl get pods -n istio-system

kiali
c4jmd -n istio-system
{"level":"info","time":"2025-10-01T12:23:28Z","message":"Adding a RegistryRefreshHandler"}
2025-10-01T12:23:28Z INF Kiali: Version: v1.76.1, Commit: 0f96891475ab41b10821ae8e1850572fbcd1505f, Go: 1.20.10
2025-10-01T12:23:28Z INF Using authentication strategy [anonymous]
2025-10-01T12:23:28Z WRN Kiali auth strategy is configured for anonymous access - users will not be authenticated.
2025-10-01T12:23:28Z INF Some validation errors will be ignored [KIA1301]. If these errors do occur, they will still be logged. If you think the validation errors you see are incorrect, please report them to the Kiali team if you have not done so already and provide the details of your scenario. This will keep Kiali validations strong for the whole community.
2025-10-01T12:23:28Z INF Initializing Kiali Cache
2025-10-01T12:23:28Z INF Adding a RegistryRefreshHandler
2025-10-01T12:23:28Z INF [Kiali Cache] Waiting for cluster-scoped cache to sync
2025-10-01T12:23:28Z INF [Kiali Cache] Started
2025-10-01T12:23:28Z INF [Kiali Cache] Kube cache is active for cluster: [Kubernetes]
2025-10-01T12:23:28Z INF Server endpoint will start at [:20001/kiali]
2025-10-01T12:23:28Z INF Server endpoint will serve static content from [/opt/kiali/console]
2025-10-01T12:23:28Z INF Starting Metrics Server on [:9090]
```

You can get access to Kiali with:

```
ssh -L 20001:localhost:20001 cpouta
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```

Check the dashboard in http://localhost:20001/. Be aware that we can debug a lot of problems with Istio and Kiali using pod logs, which you can get with:

```
kubectl logs istiod-(id) -n istio-system
kubectl logs kiali-(id) -n istio-system
```

We now have everything we need to setup gateways and virtualservices. Gateways provide a entrypoint throug the Kind extraportmappings to the OSS cluster, while virtualservices route the traffic into specific services. We can check all gateways and virtualservices with:

```
kubectl get gateways -A
kubectl get virtualservices -A
```

These should give the following:

```
gateways
NAMESPACE         NAME                    AGE
istio-system      cluster-local-gateway   2d6h
istio-system      istio-ingressgateway    2d6h
knative-serving   knative-local-gateway   2d6h
kubeflow          kubeflow-gateway        2d6h

virtualservices 
NAMESPACE   NAME     GATEWAYS                        HOSTS   AGE
mlflow      mlflow   ["kubeflow/kubeflow-gateway"]   ["*"]   2d6h
```

What we first need to do is to map the wanted ports into a existing istio service, which in this case is istio-ingressgateway. We will modify it with the following:

```
kubectl get svc istio-ingressgateway -n istio-system -o yaml > study-istio-ingressgateway.yaml
nano study-istio-ingressgateway.yaml
```

Modify this YAML by adding the following:

```
ports:
  - name: status-port
    nodePort: 30900
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http2
    nodePort: 30901
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 30902
    port: 443
    protocol: TCP
    targetPort: 8443
  - name: http-dashboards
    nodePort: 31001
    port: 100
    protocol: TCP
    targetPort: 9501 
  - name: tcp-redis 
    nodePort: 31002
    port: 200
    protocol: TCP
    targetPort: 9502
  - name: tcp-minio
    nodePort: 31003
    port: 201
    protocol: TCP
    targetPort: 9503
  - name: tcp-postgres
    nodePort: 31004
    port: 202 
    protocol: TCP
    targetPort: 9504
  - name: tcp-mongo
    nodePort: 31005
    port: 203
    protocol: TCP
    targetPort: 9505
  - name: tcp-qdrant
    nodePort: 31006
    port: 204
    protocol: TCP
    targetPort: 9006
  - name: tcp-neo4j
    nodePort: 31007
    port: 205
    protocol: TCP
    targetPort: 9007
selector:
  app: istio-ingressgateway
  istio: ingressgateway
sessionAffinity: None
type: NodePort
```

After saving this, apply it with

```
kubectl apply -f study-istio-ingressgateway.yaml
```

Now we can create necessery gateways and virtual services. The gateway and virtualservice template for dashboards are:

```
# Gateway
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: dashboard-gateway-1
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      name: http
      number: 100
      protocol: HTTP
    hosts:
    - "kubeflow.oss"
    - "kubeflow.minio.oss"
    - "mlflow.oss"
    - "mlflow.minio.oss"
    - "prometheus.oss"
    - "grafana.oss"
    - "forwarder.frontend.oss"
    - "forwarder.monitor.oss"
    - "forwarder.airflow.oss"
    - "kiali.oss"
    - "minio.oss"
    - "mongo.oss"
    - "qdrant.oss"
    - "neo4j.oss"
    - "ollama.oss"
    - "webui.oss"
    
# Virtual service
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: (name)-virtualservice
spec:
  hosts:
  - "(gateway-host)"
  gateways:
  - dashboard-gateway-1
  http:
  - route:
    - destination:
        host: (service-name).(service-namespace).svc.cluster.local
        port:
          number: (service-port)
```

In these the critical things are matching gateway-host, service-name, service-namespace and service-port. For example, in the case of Kiali these are:

   - gateway-host = kiali-oss
   - service-name = kiali
   - service-namespace = istio-system
   - service-port = 20001

These have already been provided in deployments/networking/http, which you can deploy with:

```
kubectl apply -k http
```

You can check these with:

```
kubectl get gateways -A
kubectl get virtualservices -A
```

To test these, we can use curl on the host machine with the following command:

```
curl -v -H "Host: (host-name)" http://localhost:7001
```

You can see these curls in logs:

```
kubectl logs istio-ingressgateway-(id) -n istio-system
```

By using both of these on each host-name should give some of the following:

```
# kubeflow.oss
# curl -v -H "Host: kubeflow.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: kubeflow.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/html; charset=utf-8
< content-length: 670
< etag: W/"29e-J4oJkifJqN5fxHZn6TdXdlThLtw"
< date: Thu, 02 Oct 2025 09:05:20 GMT
< x-envoy-upstream-service-time: 3
< server: istio-envoy
# logs
[2025-10-02T09:07:57.462Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 670 2 1 "10.244.0.1" "curl/7.81.0" "e610ece0-b893-4f5b-9938-0d0532a7a71b" "kubeflow.oss" "10.244.0.15:3000" outbound|80||ml-pipeline-ui.kubeflow.svc.cluster.local 10.244.0.44:33098 10.244.0.44:9501 10.244.0.1:13975 - -

# kubeflow.minio.oss
# curl -v -H "Host: kubeflow.minio.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: kubeflow.minio.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< accept-ranges: bytes
< content-length: 226
< content-security-policy: block-all-mixed-content
< content-type: application/xml
< server: istio-envoy
< vary: Origin
< x-amz-request-id: 186AA21040701A82
< x-xss-protection: 1; mode=block
< date: Thu, 02 Oct 2025 09:19:55 GMT
< x-envoy-upstream-service-time: 2
# logs
[2025-10-02T09:19:55.123Z] "GET / HTTP/1.1" 403 - via_upstream - "-" 0 226 3 2 "10.244.0.1" "curl/7.81.0" "6da7001b-8773-4c44-841e-0e0cc0d9f2b8" "kubeflow.minio.oss" "10.244.0.35:9000" outbound|9000||minio-service.kubeflow.svc.cluster.local 10.244.0.44:57052 10.244.0.44:9501 10.244.0.1:22764 - -

# mlflow.oss
# curl -v -H "Host: mlflow.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: mlflow.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< server: istio-envoy
< date: Thu, 02 Oct 2025 09:21:12 GMT
< content-disposition: inline; filename=index.html
< content-type: text/html; charset=utf-8
< content-length: 615
< last-modified: Tue, 11 Mar 2025 21:54:22 GMT
< cache-control: no-cache
< etag: "1741730062.0-615-3334150879"
< x-envoy-upstream-service-time: 3
# logs
[2025-10-02T09:21:12.426Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 615 3 3 "10.244.0.1" "curl/7.81.0" "a74df38a-53a1-48a3-ae1a-47eaa8aa308a" "mlflow.oss" "10.244.0.22:5000" outbound|5000||mlflow.mlflow.svc.cluster.local 10.244.0.44:48148 10.244.0.44:9501 10.244.0.1:17215 - -

# mlflow.minio.oss
# curl -v -H "Host: mlflow.minio.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: mlflow.minio.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< accept-ranges: bytes
< content-length: 1310
< content-type: text/html
< last-modified: Thu, 02 Oct 2025 09:22:20 GMT
< server: istio-envoy
< x-content-type-options: nosniff
< x-frame-options: DENY
< x-xss-protection: 1; mode=block
< date: Thu, 02 Oct 2025 09:22:20 GMT
< x-envoy-upstream-service-time: 1
# logs
[2025-10-02T09:22:20.986Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 1310 2 1 "10.244.0.1" "curl/7.81.0" "47a0961d-93d9-4ea3-bad0-787986f68d75" "mlflow.minio.oss" "10.244.0.38:9001" outbound|9001||mlflow-minio-service.mlflow.svc.cluster.local 10.244.0.44:47716 10.244.0.44:9501 10.244.0.1:10680 - -

# prometheus.oss
# curl -v -H "Host: prometheus.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: prometheus.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 302 Found
< content-type: text/html; charset=utf-8
< location: /graph
< date: Thu, 02 Oct 2025 09:23:14 GMT
< content-length: 29
< x-envoy-upstream-service-time: 0
< server: istio-envoy

* Connection #0 to host localhost left intact
# logs
[2025-10-02T09:23:14.036Z] "GET / HTTP/1.1" 302 - via_upstream - "-" 0 29 1 0 "10.244.0.1" "curl/7.81.0" "d014945b-c413-49f6-9fd4-4734ec6643d3" "prometheus.oss" "10.244.0.36:9090" outbound|8080||prometheus-service.monitoring.svc.cluster.local 10.244.0.44:43144 10.244.0.44:9501 10.244.0.1:13318 - -

# grafana.oss
# curl -v -H "Host: grafana.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: grafana.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 302 Found
< cache-control: no-cache
< content-type: text/html; charset=utf-8
< expires: -1
< location: /login
< pragma: no-cache
< set-cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
< x-content-type-options: nosniff
< x-frame-options: deny
< x-xss-protection: 1; mode=block
< date: Thu, 02 Oct 2025 09:23:50 GMT
< content-length: 29
< x-envoy-upstream-service-time: 0
< server: istio-envoy
* Connection #0 to host localhost left intact
# logs
[2025-10-02T09:23:50.657Z] "GET / HTTP/1.1" 302 - via_upstream - "-" 0 29 1 0 "10.244.0.1" "curl/7.81.0" "66306981-3b51-4a32-adf7-4b7696871947" "grafana.oss" "10.244.0.32:3000" outbound|3000||grafana.monitoring.svc.cluster.local 10.244.0.44:57096 10.244.0.44:9501 10.244.0.1:55575 - -

# forwarder.frontend.oss
# curl -v -H "Host: forwarder.frontend.oss" http://localhost:7001/docs
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET /docs HTTP/1.1
> Host: forwarder.frontend.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< date: Thu, 02 Oct 2025 09:25:42 GMT
< server: istio-envoy
< content-length: 931
< content-type: text/html; charset=utf-8
< x-envoy-upstream-service-time: 1
* Connection #0 to host localhost left intact
# logs
[2025-10-02T09:25:42.668Z] "GET /docs HTTP/1.1" 200 - via_upstream - "-" 0 931 2 1 "10.244.0.1" "curl/7.81.0" "e8320ce5-256f-4679-a376-4d25f47dbd62" "forwarder.frontend.oss" "10.244.0.93:7600" outbound|7600||fastapi-service.forwarder.svc.cluster.local 10.244.0.44:57914 10.244.0.44:9501 10.244.0.1:16906 - -

# forwarder.monitor.oss
# curl -v -H "Host: forwarder.monitor.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: forwarder.monitor.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
< server: istio-envoy
< content-type: text/html; charset=UTF-8
< date: Thu, 02 Oct 2025 09:27:01 GMT
< www-authenticate: Basic realm="flower"
< content-length: 13
< x-envoy-upstream-service-time: 3
* Connection #0 to host localhost left intact
Access denied
# logs
[2025-10-02T09:27:01.567Z] "GET / HTTP/1.1" 401 - via_upstream - "-" 0 13 3 3 "10.244.0.1" "curl/7.81.0" "a2e25434-c423-4604-8d44-de80024d5415" "forwarder.monitor.oss" "10.244.0.99:7601" outbound|7601||flower-service.forwarder.svc.cluster.local 10.244.0.44:37690 10.244.0.44:9501 10.244.0.1:28145 - -

# forwarder.airflow.oss
# curl -v -H "Host: forwarder.airflow.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: forwarder.airflow.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< date: Fri, 03 Oct 2025 06:53:59 GMT
< server: istio-envoy
< content-length: 483
< content-type: text/html; charset=utf-8
< vary: Accept-Encoding
< x-envoy-upstream-service-time: 6
# logs
[2025-10-03T06:53:59.417Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 483 7 6 "10.244.0.1" "curl/7.81.0" "520a504a-2f7e-4383-812d-b6784d142216" "forwarder.airflow.oss" "10.244.0.87:8080" outbound|8080||airflow-api-server.airflow.svc.cluster.local 10.244.0.44:55830 10.244.0.44:9501 10.244.0.1:12338 - -

# kiali.oss
# curl -v -H "Host: kiali.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: kiali.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 302 Found
< content-type: text/html; charset=utf-8
< location: /kiali/
< vary: Accept-Encoding
< date: Thu, 02 Oct 2025 09:27:48 GMT
< content-length: 30
< x-envoy-upstream-service-time: 0
< server: istio-envoy

* Connection #0 to host localhost left intact
# logs
[2025-10-02T09:27:48.638Z] "GET / HTTP/1.1" 302 - via_upstream - "-" 0 30 1 0 "10.244.0.1" "curl/7.81.0" "191bef02-39e6-4015-9b13-9092ac777ae9" "kiali.oss" "10.244.0.123:20001" outbound|20001||kiali.istio-system.svc.cluster.local 10.244.0.44:39632 10.244.0.44:9501 10.244.0.1:58115 - -

# minio.oss
# curl -v -H "Host: minio.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: minio.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< accept-ranges: bytes
< content-length: 1309
< content-security-policy: default-src 'self' 'unsafe-eval' 'unsafe-inline'; script-src 'self' https://unpkg.com;  connect-src 'self' https://unpkg.com;
< content-type: text/html
< last-modified: Thu, 02 Oct 2025 09:28:42 GMT
< referrer-policy: strict-origin-when-cross-origin
< server: istio-envoy
< x-content-type-options: nosniff
< x-frame-options: DENY
< x-xss-protection: 1; mode=block
< date: Thu, 02 Oct 2025 09:28:42 GMT
< x-envoy-upstream-service-time: 1
# logs
[2025-10-02T09:28:42.772Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 1309 2 1 "10.244.0.1" "curl/7.81.0" "67147df3-58f2-457f-b416-e45a60dcfa8e" "minio.oss" "10.244.0.111:9001" outbound|9101||minio-service.storage.svc.cluster.local 10.244.0.44:51440 10.244.0.44:9501 10.244.0.1:64383 - -

# mongo.oss
# curl -v -H "Host: mongo.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: mongo.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
< x-powered-by: Express
< www-authenticate: Basic realm="Authorization Required"
< date: Thu, 02 Oct 2025 09:29:48 GMT
< content-length: 12
< x-envoy-upstream-service-time: 6
< server: istio-envoy
< 
* Connection #0 to host localhost left intact
Unauthorized
# logs
[2025-10-02T09:29:48.685Z] "GET / HTTP/1.1" 401 - via_upstream - "-" 0 12 6 6 "10.244.0.1" "curl/7.81.0" "cc9cdfb3-6ec8-4be3-943c-7de884ca0e1c" "mongo.oss" "10.244.0.103:8081" outbound|7200||express-service.storage.svc.cluster.local 10.244.0.44:57880 10.244.0.44:9501 10.244.0.1:19851 - -

# qdrant.oss
# curl -v -H "Host: qdrant.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: qdrant.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 112
< content-type: application/json
< vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
< date: Thu, 02 Oct 2025 09:30:44 GMT
< x-envoy-upstream-service-time: 1
< server: istio-envoy
< 
* Connection #0 to host localhost left intact
{"title":"qdrant - vector search engine","version":"1.15.5","commit":"48203e414e4e7f639a6d394fb6e4df695f808e51"}
# logs
[2025-10-02T09:30:44.955Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 112 1 1 "10.244.0.1" "curl/7.81.0" "1fc0fe41-b0ec-4958-a8f0-771b6bedfc21" "qdrant.oss" "10.244.0.116:6333" outbound|7201||qdrant-service.storage.svc.cluster.local 10.244.0.44:42788 10.244.0.44:9501 10.244.0.1:29970 - -

# neo4j.oss
# curl -v -H "Host: neo4j.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: neo4j.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< date: Thu, 02 Oct 2025 09:31:26 GMT
< access-control-allow-origin: *
< content-type: application/json
< vary: Accept
< content-length: 241
< x-envoy-upstream-service-time: 23
< server: istio-envoy
< 
* Connection #0 to host localhost left intact
{"bolt_routing":"neo4j://neo4j.oss:7687","query":"http://neo4j.oss/db/{databaseName}/query/v2","transaction":"http://neo4j.oss/db/{databaseName}/tx","bolt_direct":"bolt://neo4j.oss:7687","neo4j_version":"5.26.12","neo4j_edition":"community"}
# logs 
[2025-10-02T09:31:26.497Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 241 24 23 "10.244.0.1" "curl/7.81.0" "8bbbf8d8-4898-4790-ad17-af3235d94403" "neo4j.oss" "10.244.0.115:7474" outbound|7474||neo4j-service.storage.svc.cluster.local 10.244.0.44:58414 10.244.0.44:9501 10.244.0.1:47335 - -

# ollama.oss
# curl -v -H "Host: ollama.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: ollama.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: text/plain; charset=utf-8
< date: Thu, 02 Oct 2025 09:40:40 GMT
< content-length: 17
< x-envoy-upstream-service-time: 2
< server: istio-envoy
< 
* Connection #0 to host localhost left intact
Ollama is running
# logs
[2025-10-02T09:40:40.882Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 17 3 2 "10.244.0.1" "curl/7.81.0" "cb06046f-8695-4608-9216-438e88171fdd" "ollama.oss" "10.244.0.120:11434" outbound|7100||ollama-service.language.svc.cluster.local 10.244.0.44:33744 10.244.0.44:9501 10.244.0.1:16719 - -

# webui.oss
# curl -v -H "Host: webui.oss" http://localhost:7001
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: webui.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< date: Thu, 02 Oct 2025 09:41:22 GMT
< server: istio-envoy
< content-type: text/html; charset=utf-8
< accept-ranges: bytes
< content-length: 7009
< last-modified: Mon, 29 Sep 2025 15:10:52 GMT
< etag: "e916fcf4eefb35f418e8e4be845c0853"
< vary: Accept-Encoding
< x-process-time: 0
< x-envoy-upstream-service-time: 11
< 
* Connection #0 to host localhost left intact
# logs
[2025-10-02T09:41:23.596Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 7009 12 11 "10.244.0.1" "curl/7.81.0" "f8824ee6-8289-4b74-8747-35af29bc8ed0" "webui.oss" "10.244.0.121:8080" outbound|7101||webui-service.language.svc.cluster.local 10.244.0.44:33576 10.244.0.44:9501 10.244.0.1:9797 - -
```

With these test we know that these services can be accessed outside the docke containers, which means we only need to configure computer /etc/hosts and VM firewall access to use them in our browsers. The /etc/hosts template is:

```
(vm_floating_ip) kubeflow.oss
(vm_floating_ip) kubeflow.minio.oss
(vm_floating_ip) mlflow.oss
(vm_floating_ip) mlflow.minio.oss
(vm_floating_ip) prometheus.oss
(vm_floating_ip) grafana.oss
(vm_floating_ip) forwarder.frontend.oss
(vm_floating_ip) forwarder.monitor.oss
(vm_floating_ip) forwarder.airflow.oss
(vm_floating_ip) kiali.oss
(vm_floating_ip) minio.oss
(vm_floating_ip) mongo.oss
(vm_floating_ip) qdrant.oss
(vm_floating_ip) neo4j.oss
(vm_floating_ip) ollama.oss
(vm_floating_ip) webui.oss
```

You can modify the file with:

```
sudo nano /etc/hosts
CTRL + X
Y
```

Now you only need to add the following security group rule:

   - Custom TCP Rule
   - Ingress
   - Port = 7001
   - CIRD = your IP

You can then access the dashboards with your browser using the following links:

   - http://kubeflow.oss:7001
   - http://kubeflow.minio.oss:7001
   - http://mlflow.oss:7001
   - http://mlflow.minio.oss:7001
   - http://prometheus.oss:7001
   - http://grafana.oss:7001
   - http://forwarder.frontend.oss:7001/docs
   - http://forwarder.monitor.oss:7001
   - http://forwarder.airflow.oss:7001
   - http://kiali.oss:7001
   - http://minio.oss:7001
   - http://mongo.oss:7001
   - http://qdrant.oss:7001/dashboard
   - http://neo4j.oss:7001
   - http://webui.oss:7001

You can now use kiali to see the traffic in the network by doing the following:

   - Click graph
   - Select airflow, forwarder, kubeflow, language, mlflow, monitoring and storage namespaces
   - Connect to any dashboard or refresh a existing page
   - Click the refresh button on the top right corner
   - See the updated graph show green arrows to services

Kiali additionally provides a easy way to check the gateways and virtualservices by doing the following:

   - Click Istio config
   - Select default, airflow, forwarder, kubeflow, language, mlflow, monitoring and storage namespaces
   - Click either gateway or virtualservice
   - You will now see the used YAML

This can be used to modify istio yamls as needed, but we will talk about it future parts. The final preparation we need to do is to create gateways and virtualservice pairs for TCP connections. The difference between HTTP and TCP is that TCP does not allow the use of hosts, which means we need to create gateway-virtualservice pairs for available ports. This is the reason for soo many extramappingports in the Kind configuration, since TCP connections take a port each and it could be useful to have separate ports for HTTP connections. The gateway and virtualservice templates for TCP connections are:

```
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: (name)-gateway
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      name: tcp
      number: (ingress-port)
      protocol: TCP
    hosts:
    - '*'
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: (name)-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - (name)-gateway
  http:
  - match:
    - port: (ingress-port)
    route:
    - destination:
        host: (service-name).(namespace).svc.cluster.local
        port:
          number: (service-port)
```

These are again provided in deployments/networking/tcp, which we can deploy with:

```
kubectl apply -k tcp
```

You should now have the following gateways and virtualservices:

```
# kubectl get gateways -A
NAMESPACE         NAME                    AGE
default           dashboards-gateway-1    21h
default           minio-gateway           19h
default           mongo-gateway           19h
default           neo4j-gateway           19h
default           postgres-gateway        19h
default           qdrant-gateway          19h
default           redis-gateway           19h
istio-system      cluster-local-gateway   4d
istio-system      istio-ingressgateway    4d
knative-serving   knative-local-gateway   4d
kubeflow          kubeflow-gateway        4d

# kubectl get virtualservices -A
NAMESPACE   NAME                                GATEWAYS                        HOSTS                        AGE
default     express-virtualservice              ["dashboards-gateway-1"]        ["mongo.oss"]                21h
default     forwarder-airflow-virtualservice    ["dashboards-gateway-1"]        ["forwarder.airflow.oss"]    40s
default     forwarder-frontend-virtualservice   ["dashboards-gateway-1"]        ["forwarder.frontend.oss"]   21h
default     forwarder-monitor-virtualservice    ["dashboards-gateway-1"]        ["forwarder.monitor.oss"]    21h
default     grafana-virtualservice              ["dashboards-gateway-1"]        ["grafana.oss"]              21h
default     kiali-virtualservice                ["dashboards-gateway-1"]        ["kiali.oss"]                21h
default     kubeflow-minio-virtualservice       ["dashboards-gateway-1"]        ["kubeflow.minio.oss"]       21h
default     kubeflow-virtualservice             ["dashboards-gateway-1"]        ["kubeflow.oss"]             21h
default     minio-server-virtualservice         ["minio-gateway"]               ["*"]                        19h
default     minio-virtualservice                ["dashboards-gateway-1"]        ["minio.oss"]                21h
default     mlflow-minio-virtualservice         ["dashboards-gateway-1"]        ["mlflow.minio.oss"]         21h
default     mlflow-virtualservice               ["dashboards-gateway-1"]        ["mlflow.oss"]               21h
default     mongo-virtualservice                ["mongo-gateway"]               ["*"]                        19h
default     neo4j-bolt-virtualservice           ["neo4j-gateway"]               ["*"]                        19h
default     neo4j-virtualservice                ["dashboards-gateway-1"]        ["neo4j.oss"]                21h
default     ollama-virtualservice               ["dashboards-gateway-1"]        ["ollama.oss"]               21h
default     postgres-virtualservice             ["postgres-gateway"]            ["*"]                        19h
default     prometheus-virtualservice           ["dashboards-gateway-1"]        ["prometheus.oss"]           21h
default     qdrant-tcp-virtualservice           ["qdrant-gateway"]              ["*"]                        19h
default     qdrant-virtualservice               ["dashboards-gateway-1"]        ["qdrant.oss"]               21h
default     redis-virtualservice                ["redis-gateway"]               ["*"]                        19h
default     webui-virtualservice                ["dashboards-gateway-1"]        ["webui.oss"]                21h
mlflow      mlflow                              ["kubeflow/kubeflow-gateway"]   ["*"]                        4d
```

We can again test TCP connections with curl by removing the host part, which gives the following responses and logs:

```
# redis
# curl -v http://localhost:7002
*   Trying 127.0.0.1:7002...
* Connected to localhost (127.0.0.1) port 7002 (#0)
> GET / HTTP/1.1
> Host: localhost:7002
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Empty reply from server
* Closing connection 0
curl: (52) Empty reply from server
# logs
[2025-10-02T11:11:41.682Z] "- - -" 0 - - - "-" 78 0 1 - "-" "-" "-" "-" "10.244.0.106:6379" outbound|6379||redis-service.storage.svc.cluster.local 10.244.0.44:56624 10.244.0.44:9502 10.244.0.1:37386 - -

# minio
# curl -v http://localhost:7003
*   Trying 127.0.0.1:7003...
* Connected to localhost (127.0.0.1) port 7003 (#0)
> GET / HTTP/1.1
> Host: localhost:7003
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Accept-Ranges: bytes
< Content-Length: 254
< Content-Type: application/xml
< Server: MinIO
< Strict-Transport-Security: max-age=31536000; includeSubDomains
< Vary: Origin
< Vary: Accept-Encoding
< X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
< X-Amz-Request-Id: 186AA82F13D05A10
< X-Content-Type-Options: nosniff
< X-Ratelimit-Limit: 30984
< X-Ratelimit-Remaining: 30984
< X-Xss-Protection: 1; mode=block
< Date: Thu, 02 Oct 2025 11:12:04 GMT
# logs
[2025-10-02T11:12:04.590Z] "- - -" 0 - - - "-" 78 743 2 - "-" "-" "-" "-" "10.244.0.111:9000" outbound|9100||minio-service.storage.svc.cluster.local 10.244.0.44:39480 10.244.0.44:9503 10.244.0.1:48169 - -

# postgres
# curl -v http://localhost:7004
*   Trying 127.0.0.1:7004...
* Connected to localhost (127.0.0.1) port 7004 (#0)
> GET / HTTP/1.1
> Host: localhost:7004
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Empty reply from server
* Closing connection 0
curl: (52) Empty reply from server
# logs
[2025-10-02T11:12:41.637Z] "- - -" 0 - - - "-" 78 0 4 - "-" "-" "-" "-" "10.244.0.117:5432" outbound|5532||postgres-service.storage.svc.cluster.local 10.244.0.44:51944 10.244.0.44:9504 10.244.0.1:26201 - -

# mongo
# curl -v http://localhost:7005
*   Trying 127.0.0.1:7005...
* Connected to localhost (127.0.0.1) port 7005 (#0)
> GET / HTTP/1.1
> Host: localhost:7005
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Connection: close
< Content-Type: text/plain
< Content-Length: 85
< 
It looks like you are trying to access MongoDB over HTTP on the native driver port.
* Closing connection 0
# logs
*   Trying 127.0.0.1:7005...
* Connected to localhost (127.0.0.1) port 7005 (#0)
> GET / HTTP/1.1
> Host: localhost:7005
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Connection: close
< Content-Type: text/plain
< Content-Length: 85
< 
It looks like you are trying to access MongoDB over HTTP on the native driver port.
* Closing connection 0

# qdrant
# curl -v http://localhost:7006
*   Trying 127.0.0.1:7006...
* Connected to localhost (127.0.0.1) port 7006 (#0)
> GET / HTTP/1.1
> Host: localhost:7006
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 112
< vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
< content-type: application/json
< date: Thu, 02 Oct 2025 11:14:10 GMT
< 
* Connection #0 to host localhost left intact
{"title":"qdrant - vector search engine","version":"1.15.5","commit":"48203e414e4e7f639a6d394fb6e4df695f808e51"}
# logs
[2025-10-02T11:14:10.736Z] "- - -" 0 - - - "-" 78 298 3 - "-" "-" "-" "-" "10.244.0.116:6333" outbound|7201||qdrant-service.storage.svc.cluster.local 10.244.0.44:41798 10.244.0.44:9006 10.244.0.1:20259 - -

# neo4j
# curl -v http://localhost:7007
* Connected to localhost (127.0.0.1) port 7007 (#0)
> GET / HTTP/1.1
> Host: localhost:7007
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: application/json
< access-control-allow-origin: *
< vary: Accept
< date: Thu, 02 Oct 2025 11:14:52 GMT
< content-length: 0
< 
* Connection #0 to host localhost left intact
# logs
[2025-10-02T11:18:16.545Z] "- - -" 0 - - - "-" 78 153 4 - "-" "-" "-" "-" "10.244.0.115:7687" outbound|7687||neo4j-service.storage.svc.cluster.local 10.244.0.44:49162 10.244.0.44:9007 10.244.0.1:50454 - -
```

Finally, we only need to open VM firewalls to gain access to these services, which we can do with the following security group rule:

   - Custom TCP Rule
   - Ingress
   - Open Port = Port Range
   - From Port = 7002
   - To Port = 7007
   - CIDR = your ip

You can test these in your local computer with Curl again:

```
# redis
curl -v http://(vm_floating_ip):7002 

# minio
curl -v http://(vm_floating_ip):7003

# postgres
curl -v http://(vm_floating_ip):7004

# mongo
curl -v http://(vm_floating_ip):7005

# qdrant
curl -v http://(vm_floating_ip):7006

# neo4j
curl -v http://(vm_floating_ip):7007
```

The easiest way to confirm everything works properly is using neo4j dasbhoard to connect to TCP address. Do this with:

   - address = (vm_floating_ip):7007
   - username = neo4j
   - password = password

With this we finally have the necessery basic experience with Istio. Be aware that this should not be used in production due to the security risks created by HTTP and the only safe guard against application abuse being a IP restricted firewall. These problems could possibly be fixed by reconfiguring the already used Dex to provide a portal for authentication (this is the reason we didn't pick 1 or 2 during the OSS setup, since istio configuration would have become more complex). We will leave these security concernes and fixes for the future parts.
