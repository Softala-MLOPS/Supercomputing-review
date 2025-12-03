# Tutorial for integrated distributed computing in MLOps (6/10)
This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.
## Local compose Ray
Useful material
- https://docs.docker.com/desktop/features/gpu/
- https://github.com/ray-project/ray
- https://pypi.org/project/ray/
- https://docs.ray.io/en/latest/ray-overview/getting-started.html
- https://github.com/MarvinSt/ray-docker-compose
- https://hub.docker.com/layers/rayproject/ray/2.49.0.5531bc-py312-cpu/images/sha256-71094320d36004a7a12181bb930b3c08ebaf1c1594587737a115f279ebb9e153
- https://hub.docker.com/layers/rayproject/ray/2.49.0.5531bc-py312-cu128/images/sha256-ee4be232c6bc0bb7e9cabb3c36c211d11a15304a7790ce2f9512f65a8a916b8e
- https://docs.ray.io/en/latest/ray-core/starting-ray.html
- https://docs.ray.io/en/latest/cluster/vms/user-guides/launching-clusters/on-premises.html#on-prem
- https://docs.ray.io/en/latest/ray-core/configure.html
- https://docs.docker.com/engine/network/
- https://docs.docker.com/compose/how-tos/networking/
- https://docs.docker.com/reference/compose-file/deploy/

In order to enable interoperable code execution between local, cloud and HPC enviroments we will now setup Ray. Ray is a Python based computation framework that provides an easy way to create pararellized code for different scales. To begin we will setup a local Ray cluster consisting of a single head node and single worker node. You find the prepared YAMLs at deployments/ray/compose, where you can run either the cpu or GPU option with:
```
docker compose -f cpu-ray-cluster.yaml up
docker compose -f gpu-ray-cluster.yaml up
```
In these YAMLs the values of intrest are the following:
```
networks:
  study_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.29.0.0/16

services:
    ray-head:
        image: rayproject/ray:2.49.2.7b0af3-py312-cpu
        networks:
          study_network:
            ipv4_address: 172.29.0.15
    ray-worker:
        image: rayproject/ray:2.49.2.7b0af3-py312-cpu
        command: bash -c "ray start --address=ray-head:6379 --block"
        deploy:
          resources:
            limits:
              cpus: '3'
              memory: '4g'
        networks:
          - study_network
```
In this yaml we see the following new things:
- networks -> defines a custom network for containers
- command -> enables running commands inside a container
- resources -> enables requesting resources for container

The network definition makes it possible to specify internal IP addresses for each container. Commands enable specifying application configuration. Resources give access to CPUs, memory, disk and GPUs. Docker compose enables GPU reservations with:
```
resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```
Besides new docker compose concpets we also see things Ray provides us such as:
- CPU and GPU images -> cpu vs gpu
- Images with specific python versions -> py312
- Head and worker -> --head
- Resource reservation -> --num-cpus=1 --num-gpus=1
- Head port -> --port=6379
- Head address -> --address=ray-head:6379
- Dashboard port -> --dashboard-host=0.0.0.0 --dashboard-port=8265
- Prometheus port -> --metrics-export-port=8500
- Client port -> --ray-client-server-port=10001

From these you will most likely change image, python version and resource reservations the most depending on the use case. Be aware that some Python packages require either older or newer versions of Python. Additionally Ray requires head and worker to have the same version, while not allowing large version differences between clusters, which requires image testing especially if you plan to use GPUs. In the case of addresses you only need to figure out the numbers you best remember and don't overlap with other address in order to enable their use in development. To begin the use of Ray we only need to keep in mind the following addresses:
- Dashboard -> http://127.0.0.1:8265
- Client -> http://127.0.0.1:10001

Provided that your Ray cluster is running, if you go to http://127.0.0.1:8265, you should see an overview page. This page enables fast checking of available resources, resource consumption, submitted jobs, running jobs and failed jobs. This dashboard provides many tools for performance analysis and code debugging, which you can find in the Jobs, Cluster and Logs pages. We will go into their details once we start running jobs.
## Ray Jobs
Useful material:
- https://docs.ray.io/en/latest/cluster/running-applications/job-submission/ray-client.html
- https://docs.ray.io/en/latest/cluster/running-applications/job-submission/doc/ray.job_submission.JobSubmissionClient.html
- https://docs.ray.io/en/latest/cluster/running-applications/job-submission/doc/ray.job_submission.JobSubmissionClient.submit_job.html
- https://docs.conda.io/projects/conda/en/stable/user-guide/getting-started.html
- https://stackoverflow.com/questions/45197777/how-do-i-update-anaconda
- https://docs.conda.io/projects/conda/en/stable/user-guide/tasks/manage-conda.html
- https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-python.html
- https://docs.ray.io/en/latest/cluster/running-applications/job-submission/index.html
- https://docs.ray.io/en/latest/ray-core/key-concepts.html
- https://docs.ray.io/en/latest/ray-core/tasks.html
- https://docs.ray.io/en/latest/ray-core/actors.html
- https://docs.ray.io/en/latest/serve/index.html
- https://pypi.org/project/numpy/
- https://pypi.org/project/matplotlib/
- https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)
- https://en.wikipedia.org/wiki/Round-robin_scheduling

We can interact with Ray though two ways, which are JobsAPI via dashboard connection and Ray client via client connection. The first is done with:
```
from ray.job_submission import JobSubmissionClient

ray_job_client = JobSubmissionClient(
    address = 'http://127.0.0.1:8265'
)
```
2025-10-10 11:55:43,954	INFO util.py:154 -- Missing packages: ['ipywidgets']. Run `pip install -U ipywidgets`, then restart the notebook server for rich notebook output.2025-10-10 11:55:43,954	INFO util.py:154 -- Missing packages: ['ipywidgets']. Run `pip install -U ipywidgets`, then restart the notebook server for rich notebook output.<br></br>
The second one requires more configuration, because Ray is sensitive to package and python versions. Most likely you will have mismatches in both, where the former can be fixed by specifying the version and the latter by using conda to get the specific python version. Since the first has already been covered by simply using == in package.txt, we will now use conda to create a suitable python enviroment for the following code. First, check if conda is already installed with:
```
conda --version
```
If you want to update your version, run
```
conda update conda
```
In my case I have 24.5.0, which is good enough. We can create a conda enviroment with python 3.12.9 with:
```
conda create -n study_p3129 python=3.12.9
```
You can activate it with
```
# Activate env
conda init
conda activate study_p3129

# Confirm version
python --version
```
Now create a new venv to run this Notebook:
```
python3 -m venv tutorial_venv_py3129
source tutorial_venv_py3129/bin/activate
pip install -r packages.txt
```
If you now activate jupyterlab, you should be able to run the following code without ray complaining about package or python versions.
```
import ray

ray.init(
    address="ray://127.0.0.1:10001"
)
```
2025-10-10 11:55:48,110	INFO client_builder.py:242 -- Passing the following kwargs to ray.init() on the server: log_to_driver
SIGTERM handler is not set because current thread is not the main thread.<br></br>
