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
// HERE WE NEED THE PICTURE
Python version:	3.12.9
Ray version:	2.49.2
Dashboard:	http://172.29.0.15:8265

If you want to return to the original venv, deactivate the new venv and conda:
```
deactivate
conda deactivate
conda deactivate
```
As we have now demonstrated, we should always use JobsAPI to interact with Ray to ensure interopeability, while letting only Ray clusters to use Ray.init() to control each other. We can interact with the JobsAPI with the following functions:
```
import requests

def test_url(
    target_url: str,
    timeout: int
) -> bool:
    try:
        response = requests.head(
            url = target_url, 
            timeout = timeout
        )
        if response.status_code == 200:
            return True
        return False
    except requests.ConnectionError:
        return False
```
```
from ray.job_submission import JobSubmissionClient
import time as t

def ray_setup_client(
    dashboard_address: str,
    timeout: int
):
    start = t.time()
    ray_client = None
    ray_dashboard_url = 'http://' + dashboard_address
    while t.time() - start <= timeout:
        ray_exists = test_url(
            target_url = ray_dashboard_url,
            timeout = 5
        )
        if ray_exists:
            ray_client = JobSubmissionClient(
                address = ray_dashboard_url
            )
            break
        t.sleep(5)
    return ray_client
```
```
import requests
from ray.job_submission import JobStatus
import json
import time as t

def ray_submit_job(
    ray_client: any,
    ray_parameters: any,
    ray_job_file: any,
    ray_directory: str,
    ray_job_envs: any,
    ray_job_packages: any
) -> any:
    command = "python " + str(ray_job_file)
    if 0 < len(ray_parameters):
        command = command + " '" + json.dumps(ray_parameters) + "'"
    job_id = ray_client.submit_job(
        entrypoint = command,
        runtime_env = {
            'working_dir': str(ray_directory),
            'env_vars': ray_job_envs,
            'pip': ray_job_packages
        }
    )
    return job_id

def ray_wait_job(
    ray_client: any,
    ray_job_id: int, 
    timeout: int
) -> any:
    start = t.time()
    job_status = None
    job_logs = None
    waited_status = [
        JobStatus.SUCCEEDED, 
        JobStatus.STOPPED, 
        JobStatus.FAILED
    ]
    while t.time() - start <= timeout:
        status = ray_client.get_job_status(ray_job_id)
        print(f"status: {status}")
        if status in waited_status:
            job_status = status
            job_logs = ray_client.get_job_logs(ray_job_id)
            break
        t.sleep(5)
    return job_status, job_logs

def ray_serve_route(
    address: str,
    port: str,
    route_path: str,
    route_type: str,
    route_input: any,
    timeout: any
) -> any:
    full_url = 'http://' + address + ':' + port + route_path
    print(full_url)
    if route_type == 'POST':
        route_response = requests.post(
            url = full_url,
            json = route_input
        )
    if route_type == 'GET':
        route_response = requests.get(
            url = full_url
        )

    route_status_code = None
    route_returned_text = {}
    if not route_response is None:
        route_status_code = route_response.status_code
        if route_status_code == 200:
            route_returned_text = json.loads(route_response.text)
    return route_status_code, route_returned_text
```
We can now send jobs to Ray, which are self contained Python scripts stored in a folder with a main and functions. You already find a provided one at applications/ray/processing, which you can submit begin submitting by first getting the client:
```
ray_client = ray_setup_client(
    dashboard_address = '127.0.0.1:8265',
    timeout = 5
)
```
Then defining the amount of pararellism used in the process. Here worker number is the amount of temporary tasks used to divide the work, while actor number is the amount of consistent claims to specific resources to run heavy code.
```
process_parameters = {
    'worker-number': 2,
    'actor-number': 2
}
```
Then leaving this empty, since we will use external storage mediums later.
```
storage_parameters = {}
```
Then defining the work settings. Here amount, lengths, extensions and priority affect how much, how long, what types and what priority tuples are generated, while tuple-batch and stats-batch control the amount of units sent to actors for processing.
```
data_parameters = {
    'amount': 50,
    'lengths': [
        2,
        3,
        4,
        5,
        6,
        7
    ],
    'extensions': [
        'txt',
        'py',
        'md',
        'ipynb',
        'yaml',
        'sh'
    ],
    'priority': {
        'txt': 1,
        'py': 2,
        'md': 3,
        'ipynb': 4,
        'yaml': 5,
        'sh': 6
    },
    'tuple-batch': 5,
    'stats-batch': 5
}
```
Then we unify all of them into a single dict.
```
job_parameters = {
    'process-parameters': process_parameters,
    'storage-parameters': storage_parameters,
    'data-parameters': data_parameters
}
```
Finally, we give the client connections, parameters, job file name, job file directory, enviromental variables and used packages.
```
job_id = ray_submit_job(
    ray_client = ray_client,
    ray_parameters = job_parameters,
    ray_job_file = 'parallel_processing.py',
    ray_directory = '(fill_path)/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/applications/ray/processing',
    ray_job_envs = {},
    ray_job_packages = [
        'numpy'
    ]
)
```
If there were no errors, you can check the running job at http://localhost:8265. To see the logs, do the following
- Click first job under recent jobs
- Scroll down to see logs

When you see a row similar to
```
INFO 2025-10-08 04:28:20,679 serve 1321 -- Application 'requests' is ready at http://0.0.0.0:8350/.
```
you can run the following block to request created output:
```
serve_status, serve_output = ray_serve_route(
    address = '127.0.0.1',
    port = '8350',
    route_path = '/output',
    route_type = 'GET',
    route_input = {},
    timeout = 5
)
```
http://127.0.0.1:8350/output<br></br>
This block enables you to wait until the Ray job is complete, which enables you to provide conditions for future actions and get the created logs:
```
job_status, job_logs = ray_wait_job(
    ray_client = ray_client,
    ray_job_id = job_id, 
    timeout = 300
)
print(job_logs)
```
```
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: RUNNING
status: SUCCEEDED
2025-10-10 01:57:19,317	INFO job_manager.py:531 -- Runtime env is setting up.
2025-10-10 01:57:23,062	INFO worker.py:1630 -- Using address 172.29.0.15:6379 set in the environment variable RAY_ADDRESS
2025-10-10 01:57:23,065	INFO worker.py:1771 -- Connecting to existing Ray cluster at address: 172.29.0.15:6379...
2025-10-10 01:57:23,081	INFO worker.py:1942 -- Connected to Ray cluster. View the dashboard at http://172.29.0.15:8265 
(ProxyActor pid=1762) INFO 2025-10-10 01:57:26,313 proxy 172.29.0.15 -- Proxy starting on node 370b18789d8a31f5481d561e8af93c558346f38b913cb235d3966aa6 (HTTP port: 8350).
INFO 2025-10-10 01:57:26,392 serve 1631 -- Started Serve in namespace "serve".
Starting Ray job
Python version is:3.12.9 | packaged by conda-forge | (main, Mar  4 2025, 22:48:41) [GCC 13.3.0]
Ray version is:2.49.2
NumPy version is:1.26.4
Getting and loading input
Running parallel processing
Generating file batches for 2 workers
Storing batches
Creating 2 provider actors
Starting preprocess tasks
Waiting preprocess tasks and starting calculator tasks
(ProxyActor pid=1762) INFO 2025-10-10 01:57:26,387 proxy 172.29.0.15 -- Got updated endpoints: {}.
Waiting calculator tasks
Starting Serve
INFO 2025-10-10 01:57:27,529 serve 1631 -- Connecting to existing Serve app in namespace "serve". New http options will not be applied.
WARNING 2025-10-10 01:57:27,530 serve 1631 -- The new client HTTP config differs from the existing one in the following fields: ['host', 'port', 'location']. The new HTTP config is ignored.
(ServeController pid=1700) INFO 2025-10-10 01:57:27,557 controller 1700 -- Deploying new version of Deployment(name='Requests', app='requests') (initial target replicas: 1).
(ProxyActor pid=1762) INFO 2025-10-10 01:57:27,567 proxy 172.29.0.15 -- Got updated endpoints: {Deployment(name='Requests', app='requests'): EndpointInfo(route='/', app_is_cross_language=False)}.
(ProxyActor pid=1762) INFO 2025-10-10 01:57:27,577 proxy 172.29.0.15 -- Started <ray.serve._private.router.SharedRouterLongPollClient object at 0x7f3b9c2766f0>.
(ServeController pid=1700) INFO 2025-10-10 01:57:27,668 controller 1700 -- Adding 1 replica to Deployment(name='Requests', app='requests').
INFO 2025-10-10 01:57:29,659 serve 1631 -- Application 'requests' is ready at http://0.0.0.0:8350/.
Waiting for interactions
(ServeReplica:requests:Requests pid=2056) INFO 2025-10-10 01:57:41,328 requests_Requests 9ag4tpqp 8a7ee04c-3160-4d03-ab96-072ee6bef35c -- GET /output 200 3.0ms
Stopping Serve
(ServeController pid=1700) INFO 2025-10-10 01:59:29,718 controller 1700 -- Removing 1 replica from Deployment(name='Requests', app='requests').
(ServeController pid=1700) INFO 2025-10-10 01:59:31,795 controller 1700 -- Replica(id='9ag4tpqp', deployment='Requests', app='requests') is stopped.
Job success:True
Ray job Complete
```
The created output is a summary on the means and variances created by the generator with id, worker, actor and batches metadata to show the chain of events. Here the values mean the following:
- ID -> worker, actor and batch index number joined
- Workers -> List of workers that created the batches
- Actors -> List of actors that processes the batches
- Batches -> List of index numbers for the batch of a worker
- Seed -> Mean and variance of the seeds used in creating the tuples
- Name -> Mean and variance of the characters in alphabetical numbers used in tuples
- Lengths -> Mean and variance of the name lengths used in tuples
- Priority -> Mean and variance of the priority used in sorting tuples for round robin
```
print(serve_output['output'][0])
```
{'id': '1-1-1', 'workers': [1, 1, 1, 1, 1], 'actors': [1, 1, 1, 1, 1], 'batches': [1, 2, 3, 4, 5], 'seed': {'mean': [20.8, 26.0, 32.2, 26.4, 21.4], 'variance': [185.76000000000002, 154.8, 204.56, 220.64000000000001, 125.44000000000001]}, 'name': {'mean': [15.088235294117647, 14.441176470588236, 12.975609756097562, 13.849056603773585, 13.441176470588236], 'variance': [52.551038062283745, 61.83477508650518, 55.048185603807255, 54.354574581701684, 45.59948096885813]}, 'priority': {'mean': [1.2, 2.2, 3.8, 4.4, 5.8], 'variance': [0.16, 0.15999999999999998, 0.15999999999999998, 0.24000000000000005, 0.15999999999999998]}, 'length': {'mean': [4.0, 4.8, 3.8, 6.0, 4.4], 'variance': [2.8, 1.7600000000000002, 1.3599999999999999, 1.2, 3.44]}}<br></br>
Run these blocks if you want to see the created output in scatter plots:
```
collector_output = serve_output['output'] 
relevant_values = [
    'seed',
    'name',
    'priority',
    'length'
]
scatter_plot_data = {}
for summary in collector_output:
    group_name = summary['id']
    for value in relevant_values:
        if not value in scatter_plot_data:
            scatter_plot_data[value] = {}
        scatter_plot_data[value][group_name] = summary[value] 
```
```
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.colors as mcolors

for case, data in scatter_plot_data.items():
    color_groups = []
    means = []
    variances = []
    for group, values in data.items():
        color_groups.append(group)
        means.append(values['mean'])
        variances.append(values['variance'])
    group_amount = len(color_groups)
    cmap = plt.colormaps['tab10'].resampled(group_amount)
    color_map = {group: cmap(i) for i, group in enumerate(list(color_groups))}
    index = 0
    plt.figure(figsize = (7,5))
    for group in color_groups:
        plt.scatter(means[index], variances[index], color = color_map[group], label = group, alpha = 0.7)
        index += 1
    plt.title('Parallel processing summary scatter plot for ' + str(case) + ' values')
    plt.xlabel('Mean')
    plt.ylabel('Variance')
    plt.legend(title='ID')
    plt.grid(True)
    plt.show()
```
Parallel processing summary pictures x 4 HERE</br></br>
With this demonstration of Ray, let's now check the actual code to understand the critical components of the Ray job stored at applications/ray/processing. Inside the folder we see that the files are structures in:
- parallel_processing.py
- functions
  - division.py
  - generator.py
- tasks
  - preprocess
  - collector
- actors
  - provider
- serve
  - requests
Compared to Airflow we cleary see that Ray has more flexible approach to separating code and reserving resources for their execution. This job specifically uses data pararellism to divide the work between tasks, which further divide the work into batches that are sent to actors, until all results are joined and given to serve for interactions. If we go step by step we notice the different patterns. The first execution pattern is
```
from importlib.metadata import version

if __name__ == "__main__":
    print('Python version is:' + str(sys.version))
    print('Ray version is:' + version('ray'))
    print('NumPy version is:' + version('numpy'))
```
This is used to confirm the available packages, which can forcibly change between enviroments. The second execution pattern is:
```
import json
job_input = json.loads(sys.argv[1])
process_parameters = job_input['process-parameters']
storage_parameters = job_input['storage-parameters']
data_parameters = job_input['data-parameters']
```
This enables us to give dictionary arguments to the executed file in the same way as 'python3 file_name args'. The final execution pattern is:
```
def parallel_processing(
    process_parameters: any,
    storage_parameters: any,
    data_parameters: any
):
    try: 
        return True
    except Exception as e:
        print('Parallel processing error')
        print(e)
        return False
        
if __name__ == "__main__":
    parallel_hello_status = parallel_processing(
        process_parameters = process_parameters,
        storage_parameters = storage_parameters,
        data_parameters = data_parameters
    )
```
This is meant to ensure that you will be able to debug what errors happened from the logs, while making it faster for Ray to handle the faulty job. When we check this function, the first paralleism pattern is:
```
def division_round_robin(
    target_list: any, 
    number: int
) -> any:
    lists = [[] for _ in range(number)]
    i = 0
    sorted_list = sorted(target_list, key = lambda x: (x[-2], x[-1]))
    for elem in sorted_list:
        lists[i].append(elem)
        i = (i + 1) % number
    return lists

def generate_file_batches(
    data_parameters: any,
    number: int
) -> any:
    file_tuples = []
    for file_seed in range(0, data_parameters['amount']):
        file_tuple = generate_file_tuple(
            seed = file_seed,
            lengths = data_parameters['lengths'],
            extensions = data_parameters['extensions'],
            priority = data_parameters['priority']
        ) 
        file_tuples.append(file_tuple)
    file_batches = division_round_robin(
        target_list = file_tuples, 
        number = number
    )
    return file_batches

file_batches = generate_file_batches( 
    data_parameters = data_parameters,
    number = worker_number
)

file_batch_refs = []
for file_batch in file_batches:
    file_batch_refs.append(ray.put(file_batch))
```
Here we see that the generator creates a list of items, which are then divided into separate lists using round robin. The created lists are then stored as Ray objects. The second parallelism pattern is:
```
@ray.remote(
    num_cpus = 1,
    memory = 0.2 * 1024 * 1024 * 1024
)
class Provider:
    def __init__(
        self
    ):
        import numpy as np
        self.np = np
        def batch_stats(
            data: list
        ) -> dict:
            mean = float(self.np.mean(data))
            variance = float(self.np.var(data))
            results = {
                'mean': mean,
                'variance': variance
            }
            return results
        def letter_numbers(
            text: str
        ) -> list:
            return [ord(c.lower()) - ord('a') + 1 for c in text if c.isalpha()]
        self.letter_numbers = letter_numbers
        self.batch_stats = batch_stats
    
actor_refs = []
for i in range(0, actor_number):
    actor_refs.append(Provider.remote())
```
Here we order Ray to create n actors, which are global classes that tasks can use to run specific functions with provided resources as long as they know actor reference. Actors aim to be self contained units, which is why the imports and helper functions are setup in initilization function. The third parallelism pattern is:
```
import ray

@ray.remote(
    num_cpus = 1,
    memory = 0.2 * 1024 * 1024 * 1024
)
def preprocess(
    worker_index: int,
    actor_index: int,
    actor_ref: any,
    data_parameters: any,
    file_tuples: any
) -> any:
    tuple_batch = []
    index = 0
    batch_index = 1
    provider_task_refs = []
    for file_tuple in file_tuples: 
        tuple_batch.append(file_tuple)
        index += 1
        if data_parameters['tuple-batch'] <= index:
            batched_tuples_ref = ray.put(tuple_batch)
            provider_task_refs.append(actor_ref.batch_process_tuples.remote(
                worker_index = worker_index,
                actor_index = actor_index,
                batch_index = batch_index,
                tuples = batched_tuples_ref
            ))
            tuple_batch = []
            index = 0
            batch_index += 1
    batch_stats = []
    while len(provider_task_refs):
        done_task_refs, provider_task_refs = ray.wait(provider_task_refs)
        for output_ref in done_task_refs: 
            stats = ray.get(output_ref)
            batch_stats.append(stats)
    return batch_stats

task_1_refs = [] 
worker_index = 1
actor_index = 0
for file_batch_ref in file_batch_refs:
    actor_ref = actor_refs[actor_index]
    task_1_refs.append(preprocess.remote(
        worker_index = worker_index,
        actor_index = actor_index + 1,
        actor_ref = actor_ref,
        data_parameters = data_parameters,
        file_tuples = file_batch_ref
    ))
    worker_index += 1
    actor_index = (actor_index + 1) % actor_number
```
Here we divide the list of Ray objects first come first serve and actors in a round robin to tasks, while creating a list of their references. The fourth parallelism pattern is:
```
task_2_refs = []
worker_index = 1
actor_index = 0
while len(task_1_refs):
    done_task_1_refs, task_1_refs = ray.wait(task_1_refs)
    for output_ref in done_task_1_refs:
        actor_ref = actor_refs[actor_index]
        task_2_refs.append(collector.remote(
            worker_index = worker_index,
            actor_index = actor_index + 1,
            actor_ref = actor_ref,
            data_parameters = data_parameters,
            batch_stats = output_ref
        ))
        worker_index += 1
        actor_index = (actor_index + 1) % actor_number
```
Here we chain tasks by waiting them to complete one by one to forward their outputs first come first serve and actors in a round robin into the next task. The final parallelism pattern is:
```
stat_summaries = []
while len(task_2_refs):
    done_task_2_refs, task_2_refs = ray.wait(task_2_refs)
    for output_ref in done_task_2_refs:
        stat_summaries.extend(ray.get(output_ref))
stat_summaries_ref = ray.put(stat_summaries)
```
Here we simply wait for the tasks complete, while collecting the outputs into a list, which we then put into a object for further utilization. If we go inside the tasks, we notice first batch pattern:
```
@ray.remote(
    num_cpus = 1,
    memory = 0.2 * 1024 * 1024 * 1024
)
class Provider:
    def batch_process_tuples(
        self,
        worker_index: int,
        actor_index: int,
        batch_index: int,
        tuples: any
    ) -> any:
        seeds = []
        names = []
        priorities = []
        lengths = []
        for tuple in tuples:
            seeds.append(tuple[0])
            numbers = self.letter_numbers(
                text = tuple[1]
            )
            names.extend(numbers)
            priorities.append(tuple[2])
            lengths.append(tuple[3])
        seed_stats = self.batch_stats(
            data = seeds
        )
        name_stats = self.batch_stats(
            data = names
        )
        priority_stats = self.batch_stats(
            data = priorities
        )
        lengths_stats = self.batch_stats(
            data = lengths
        )
        stats = {
            'worker': worker_index,
            'actor': actor_index,
            'batch': batch_index,
            'seed': seed_stats,
            'name': name_stats,
            'priority': priority_stats,
            'length': lengths_stats
        }
        return stats

for file_tuple in file_tuples: 
tuple_batch.append(file_tuple)
index += 1
if data_parameters['tuple-batch'] <= index:
    batched_tuples_ref = ray.put(tuple_batch)
    provider_task_refs.append(actor_ref.batch_process_tuples.remote(
        worker_index = worker_index,
        actor_index = actor_index,
        batch_index = batch_index,
        tuples = batched_tuples_ref
    ))
    tuple_batch = []
    index = 0
    batch_index += 1
```
Here we send lists of n size to actor function using provided reference, while collecting the actor call references. Notice how the actor function uses self to use the initilized functions. The final batch pattern is:
```
batch_stats = []
while len(provider_task_refs):
    done_task_refs, provider_task_refs = ray.wait(provider_task_refs)
    for output_ref in done_task_refs: 
        stats = ray.get(output_ref)
        batch_stats.append(stats)
return batch_stats
```
Here we await for the actor calls to be completed, while collecting the batches into a list, which the task provides as a output. If we go back to the main file, we notice the first serve pattern:
```
serve.start(
    http_options = {
        'host':'0.0.0.0',
        'port': 8350
    }
)
```
This is used to setup serve for later use by configuring it to create address at 0.0.0.0:8350, which enables us to use it by simply opening port 127.0.0.1:8350:8350 in docker compose. The second serve pattern is:
```
app = FastAPI()

@serve.deployment(
    num_replicas = 1,
    ray_actor_options = {
        'num_cpus': 1,
        'memory': 0.5 * 1024 * 1024 * 1024
    }
)
@serve.ingress(app)
class Requests:
    def __init__(
        self,
        data_ref
    ):
        self.data_ref = data_ref

    @app.get("/output")
    async def output_route(
        self
    ):
        output_data = ray.get(self.data_ref)
        return {'output': output_data} 

print('Starting Serve')
serve.run(
    Requests.bind(
        data_ref = stat_summaries_ref
    ), 
    name = 'requests', 
    route_prefix = '/'
)
```
Here we create a Ray serve deployment that enables us to get the processed data from localhost:8350/output by making the FASTAPI instance use the provided object reference to get the data and put it into a JSON. The final serve pattern is:
```
print('Waiting for interactions')
time.sleep(120)    

print('Stopping Serve')
serve.shutdown()
```
This keeps the Ray serve deployment up for 120 seconds until Ray serve is ordered to shutdown to ensure proper deployment clean up. Be aware that these patterns aren't necesserily the best, but at the very least they provide us a default way of creating pararellized Ray code with the ability to create services. Usually the main problem with jobs, actors and serve is the lack of resources. Specifically out of memory problems are the most common, since there isn't a good way to know how much memory code needs unless you manage to run it few times. For this reason local Ray should be used for quick development and testing, since it does not reflect upon the resources and constraints set by cloud and HPC enviroments.
## Cloud OSS KubeRay
Useful material:
- https://github.com/ray-project/kuberay
- https://docs.ray.io/en/latest/cluster/kubernetes/index.html

We can use the way mentioned in part 5 to setup Ray cluster in OSS with ray cluster. You should first check the available VM resources with:
```
kubectl describe node
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests          Limits
  --------           --------          ------
  cpu                5385m (38%)       26800m (191%)
  memory             10211910912 (8%)  24272Mi (20%)
  ephemeral-storage  0 (0%)            0 (0%)
  hugepages-1Gi      0 (0%)            0 (0%)
  hugepages-2Mi      0 (0%)            0 (0%)
  nvidia.com/gpu     1                 1
  nvshare.com/gpu    1                 1
```
In my case this enables the following values:
```
gpu-kuberay-cluster-values.yaml
image:
  repository: rayproject/ray
  tag: 2.49.2.7b0af3-py312-gpu
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
```
This enables to do the following:
```
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update

# Install both CRDs and KubeRay operator v1.0.0.
helm install kuberay-operator kuberay/kuberay-operator --version 1.0.0

# wait for the operator to be ready
kubectl wait --for=condition=available --timeout=1200s deployment/kuberay-operator

# Install KubeRay cluster
helm install raycluster kuberay/ray-cluster --version 1.0.0 -f gpu-kuberay-values.yaml
```
You can check that they are running with
```
kubectl get pods
```
If you see errors, you can try again by uninstalling the raycluster
```
helm list
helm uninstall raycluster -n default
```
You can also check the logs with:
```
kubectl get pods -n default
kubectl logs raycluster-kuberay-head-(id) -n default
raycluster-kuberay-worker-worker-(id) -n default
```
If you now check available resources you should see:
```
kubectl describe nodes
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests           Limits
  --------           --------           ------
  cpu                11385m (81%)       32800m (234%)
  memory             70211910912 (56%)  83448278Ki (69%)
  ephemeral-storage  0 (0%)             0 (0%)
  hugepages-1Gi      0 (0%)             0 (0%)
  hugepages-2Mi      0 (0%)             0 (0%)
  nvidia.com/gpu     1                  1
  nvshare.com/gpu    3                  3
```
Now, we can get access to the dashboard at http://localhost:8090 by local forwarding:
```
ssh -L 127.0.0.1:8090:localhost:8265 GPU-cpouta
kubectl port-forward svc/raycluster-kuberay-head-svc 8265:8265 -n default
```
If you now go to the cluster page, the first thing you notice in the resource use is that cloud Ray has smaller memory consumption by default, since most of the VM resources are going to OSS cluster. Additionally, the VM itself can easily have more CPUs, memory, disk and GPUs than your local computers, which enable easier development of heavier jobs.
## Local-Cloud Ray setup
Useful material:
- https://www.ssh.com/academy/ssh/config
- https://www.ssh.com/academy/ssh/tunneling-example
- https://docs.docker.com/desktop/
- https://docs.ray.io/en/latest/cluster/metrics.html
- https://habr.com/en/articles/861626/
- https://nordvpn.com/ip-lookup/
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://www.tutorialworks.com/kubernetes-curl/
- https://github.com/ray-project/ray/releases

If we assume that you have local resources such as GPUs that you would like to use to divide work or you want to develop jobs for constrained enviroments, we can connect remote local ray clusters into the cloud Kuberay to enable orhestracted use of Ray clusters from a single job. We first need to modify the SSH config at /home/.ssh of the remote computer to have the following:
```
Host rf-GPU-cpouta
Hostname (vm_floating_ip)
User (vm_user)
IdentityFile ~/.ssh/(vm_key_name)
RemoteForward (vm_private_ip):(suitable_port) 127.0.0.1:(suitable port)
```
In my case I have two more laptops that could provide nice CPU, memory and GPU resources, which I will also connect to the OSS by moving the SSH key I use to connect to CPouta via a storage medium. Be aware that these other computers don't need to be Linux systems, since docker desktop is easily installed into Mac and Windows 11 machines usually with GPU abilities available by default, which enables us to simply install docker desktop into them, clone the repository, go to study branch, modify resource requirements, run the the ray clusters, setup their SSH config and create a connection into the VM to forward the following

- Ray dashboard
- Ray client
- Ray metrics
- Ray serve

It is recommended to define suitable port ranges for dashboards, clients and serve that take into account local, hpc and service connections. We should first check the active port ranges in the VM with
```
ss -tuln
```
This allows us to get a list of system wide ports such as local forwards and Kind extraportmappings. We can also check the OSS cluster service ports with
```
kubectl get services -A
```
Based on these I define these port ranges to give 25 ports for local and HPC
- Ray dashboards (cluster port 8265) get range 8100-8150
  - Local range 8100-8124
  - HPC range 8125-8150
- Ray clients (cluster port 10001) get range 8151-8200
  - Local range 8151-8175
  - HPC range 8176-8200
- Ray metrics (cluster ports 8500 and 8501) get range 8201-8301
  - Local range 8201-8250
  - HPC range 8251-8300
- Ray serve (cluster port 8350) get range 8301-8400
  - Local range 8301-8350
  - HPC range 8351-8400

With these I can define the following configs for my three computers:
```
Computer 1
Host rf-GPU-cpouta
Hostname (vm_floating_ip)
User (vm_user)
IdentityFile ~/.ssh/(vm_key_name)
RemoteForward (vm_private_ip):8100 127.0.0.1:8265
RemoteForward (vm_private_ip):8151 127.0.0.1:10001
RemoteForward (vm_private_ip):8201 127.0.0.1:8500
RemoteForward (vm_private_ip):8202 127.0.0.1:8501
RemoteForward (vm_private_ip):8301 127.0.0.1:8350

Computer 2
Host rf-GPU-cpouta
Hostname (vm_floating_ip)
User (vm_user)
IdentityFile ~/.ssh/(vm_key_name)
RemoteForward (vm_private_ip):8101 127.0.0.1:8265
RemoteForward (vm_private_ip):8152 127.0.0.1:10001
RemoteForward (vm_private_ip):8203 127.0.0.1:8500
RemoteForward (vm_private_ip):8204 127.0.0.1:8501
RemoteForward (vm_private_ip):8302 127.0.0.1:8350

Computer 3
Host rf-GPU-cpouta
Hostname (vm_floating_ip)
User (vm_user)
IdentityFile ~/.ssh/(vm_key_name)
RemoteForward (vm_private_ip):8102 127.0.0.1:8265
RemoteForward (vm_private_ip):8153 127.0.0.1:10001
RemoteForward (vm_private_ip):8205 127.0.0.1:8500
RemoteForward (vm_private_ip):8206 127.0.0.1:8501
RemoteForward (vm_private_ip):8303 127.0.0.1:8350
```
These also enable us to create local forwards to test that all dashboards work:
```
Host lf-GPU-cpouta
Hostname (vm_floating_ip)
User (vm_user)
IdentityFile ~/.ssh/(vm_key_name)
LocalForward 127.0.0.1:8100 (vm_private_ip):8100
LocalForward 127.0.0.1:8101 (vm_private_ip):8101
LocalForward 127.0.0.1:8102 (vm_private_ip):8102
```
Before creating these connections we first need to modify the configuration of the VM SSH. We can confirm that SSH is running with:
```
ssh -V (version)
dpkg -l | grep openssh-server (running program)
sudo systemctl status ssh (program status)
sudo ss -tuln | grep :22 (listening connections)
```
If these don't give results, install SSH with:
```
sudo apt update
sudo apt install openssh-server
```
When SSH is installed, we can reconfigure it with:
```
cat /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config
```
You need to find and set the following in that file:
```
LogLevel DEBUG3
AllowTcpForwarding yes
GatewayPorts clientspecified
```
Here loglevel enables debuggin, AllowTcpForwarding enables remote forwarding and GatewayPorts enables specifying addresses outside of localhost. After saving it, make it go effect by running:
```
sudo systemctl restart ssh
```
Be aware that you can debug SSH connections by watching the following logs:
```
sudo tail -f /var/log/auth.log
```
Now, we need to confirm the VM private ip by running the following:
```
ip a
```
This shows the host network interfaces and from them we want the address of ens3 inet interface (with BROADCAST,MULTICAST,UP,LOWER_UP), which you can double check from the openstack interface in instances and ip addresses. Be aware that you can always get port and connection specifics with:
```
# listened ports
sudo ss -tulnp

# active connections
sudo lsof -i -P -n 
```
SSH can be tricky to figure out, but usually problems related to it are caused by the used commands, key permissions, authorized_keys and known_hosts. In the case of commands we will be using local and remote forwarding in the following ways:
```
ssh computer_user@computer_public_ip
-L connector_address:connector_port:computer_private_address_computer_private_port
-R computer_private_address:computer_private_port:connector_address:connector_port 
```
In the case of key permissions, SSH usually requires them to be private to a user, which can be done with:
```
chmod 600 (key_path)
```
In the case of authorized they can be fixed by:
```
# Check current keys
cat /.ssh/authorized_keys

# Add public key
echo "(public_key)" >> /.ssh/authorized_keys
nano /.ssh/authorized_keys (path might be different)
```
In the case of known hosts they can be fixed by
```
# Check known hosts
cat /.ssh/known_hosts

# Remove old ip
nano /.ssh/known_hosts (paht might be different)
ssh-keygen -R (old ip) -f /.ssh/known_hosts (path might be different)
```
With these covered, we are now ready to create connections into the VM. Be aware in the case of GPUs that you might need to update your drivers, which in my case could be done with GeForce Experience to get the following versions:
```
Computer 2
NVIDIA-SMI 581.42 
Driver Version: 581.42
CUDA Version: 13.0

Computer 3
NVIDIA-SMI 58.1.29
Driver Version: 581.29
CUDA Version 13.0
```
You can check that Ray is working correctly in remote machines, if you can see chanching resource metrics at localhost:8265 cluster page. In the case of GPUs they should show available devices and their VRAM, which in my case is
- NVIDIA GeForce RTX 4070 Laptop GPU with 8188MiB
- NVIDIA GeForce RTX 4090 Laptop GPU with 16376MiB

Be aware that CUDA can cause unknown problems, which can make the cluster metrics stay blank. The fix for this is to switch either the used cuda version in the Ray image or change the version. Now, we only need to check firewalls to enable all remote laptops to connect via SSH. Use any IP look up to get the addresses right. When they are setup, you can confirm remote forwards are working with
```
ssh lf-GPU-cpouta
```
and going to http://localhost:8100, http://localhost:8101 and http://localhost:8102, where cluster should show the same views as in the http://localhost:8265 of the host machine. With this we now only need to setup headless services and use Istio to connect into dashboards and serve. A headless service is a kubernetes object that enables us to forward a connection in host into the OSS cluster. A example template is:
```
apiVersion: v1
kind: Endpoints
metadata:
   name: (suitable_name)
subsets:
  - addresses:
    - ip: '(vm_private_ip)'
    ports: 
      - port: (connection_port)
'''
apiVersion: v1
kind: Service
metadata:
   name: (suitable_name) 
spec:
   type: ClusterIP
   ports:
   - protocol: TCP
     port: (connection_port)
     targetPort: (connection_port)
```
These have been already provided in deployments/ray/kubernetes/services, where you only need to change the private ip. When you have done that, deploy these services with:
```
kubectl apply -k services
```
In my case this produces the following:
```
kubectl get services -n integration
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
ray-local-1-client           ClusterIP   10.96.244.12    <none>        8151/TCP   30m
ray-local-1-dash             ClusterIP   10.96.248.34    <none>        8100/TCP   30m
ray-local-1-metrics-head     ClusterIP   10.96.200.194   <none>        8201/TCP   30m
ray-local-1-metrics-worker   ClusterIP   10.96.145.118   <none>        8202/TCP   30m
ray-local-1-serve            ClusterIP   10.96.170.20    <none>        8301/TCP   30m
ray-local-2-client           ClusterIP   10.96.133.42    <none>        8152/TCP   30m
ray-local-2-dash             ClusterIP   10.96.113.220   <none>        8101/TCP   30m
ray-local-2-metrics-head     ClusterIP   10.96.9.168     <none>        8203/TCP   30m
ray-local-2-metrics-worker   ClusterIP   10.96.45.177    <none>        8204/TCP   30m
ray-local-2-serve            ClusterIP   10.96.150.197   <none>        8302/TCP   30m
ray-local-3-client           ClusterIP   10.96.133.8     <none>        8153/TCP   30m
ray-local-3-dash             ClusterIP   10.96.89.139    <none>        8102/TCP   30m
ray-local-3-metrics-head     ClusterIP   10.96.86.183    <none>        8205/TCP   30m
ray-local-3-metrics-worker   ClusterIP   10.96.252.53    <none>        8206/TCP   30m
ray-local-3-serve            ClusterIP   10.96.108.106   <none>        8303/TCP   30m
```
You might have noticed that the metrics services have the following addition:
```
annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port: '(connection_port)'
```
This is metadata necessery for Prometheus to automatically scrape the data generated by the clusters. You can confirm this being the case by updating your ip in the dashboard firewall rule and going to http://prometheus.oss:7001 and doing the following:
- Click status
- Select targets
- Write -metrics
- You should now see the UP state

If you now go to graph, you should be able to query different ray related metrics. For example, if you write ray_node_cpu_count, you get something like this:
```
ray_node_cpu_count{IsHeadNode="true", RayNodeType="head", SessionName="session_2025-10-10_01-47-08_123275_1", Version="2.49.2", instance="192.168.1.19:8201", ip="172.29.0.15", job="ray-local-1-metrics-head"}
```
This data can be displayed and analyze using Grafana, which we will talk about in future parts. Before full local-cloud integration we now need to reconfigure istio to remove the need for constant local forwards, which we can do again by updating the used dashboard gateway and adding virtual services. We will add the following hosts:
```
- "ray.cloud.dash-1.oss"
- "ray.cloud.serve-1.oss"
- "ray.local.dash-1.oss"
- "ray.local.serve-1.oss"
- "ray.local.dash-2.oss"
- "ray.local.serve-2.oss"
- "ray.local.dash-3.oss"
- "ray.local.serve-3.oss"
```
The gateway and virtual services are provided at deployments/networking/http, which means we can add them with:
```
cd deployments/networking
kubectl apply -k http
```
This should give the following virtual services:
```
kubectl get virtualservices -A
NAMESPACE   NAME                                GATEWAYS                        HOSTS                        AGE
default     express-virtualservice              ["dashboards-gateway-1"]        ["mongo.oss"]                10d
default     forwarder-airflow-virtualservice    ["dashboards-gateway-1"]        ["forwarder.airflow.oss"]    10d
default     forwarder-frontend-virtualservice   ["dashboards-gateway-1"]        ["forwarder.frontend.oss"]   10d
default     forwarder-monitor-virtualservice    ["dashboards-gateway-1"]        ["forwarder.monitor.oss"]    10d
default     grafana-virtualservice              ["dashboards-gateway-1"]        ["grafana.oss"]              10d
default     kiali-virtualservice                ["dashboards-gateway-1"]        ["kiali.oss"]                10d
default     kubeflow-minio-virtualservice       ["dashboards-gateway-1"]        ["kubeflow.minio.oss"]       10d
default     kubeflow-virtualservice             ["dashboards-gateway-1"]        ["kubeflow.oss"]             10d
default     minio-server-virtualservice         ["minio-gateway"]               ["*"]                        10d
default     minio-virtualservice                ["dashboards-gateway-1"]        ["minio.oss"]                10d
default     mlflow-minio-virtualservice         ["dashboards-gateway-1"]        ["mlflow.minio.oss"]         10d
default     mlflow-virtualservice               ["dashboards-gateway-1"]        ["mlflow.oss"]               10d
default     mongo-virtualservice                ["mongo-gateway"]               ["*"]                        10d
default     neo4j-bolt-virtualservice           ["neo4j-gateway"]               ["*"]                        10d
default     neo4j-virtualservice                ["dashboards-gateway-1"]        ["neo4j.oss"]                10d
default     ollama-virtualservice               ["dashboards-gateway-1"]        ["ollama.oss"]               10d
default     postgres-virtualservice             ["postgres-gateway"]            ["*"]                        10d
default     prometheus-virtualservice           ["dashboards-gateway-1"]        ["prometheus.oss"]           10d
default     qdrant-tcp-virtualservice           ["qdrant-gateway"]              ["*"]                        10d
default     qdrant-virtualservice               ["dashboards-gateway-1"]        ["qdrant.oss"]               10d
default     ray-cloud-1-dash-virtualservice     ["dashboards-gateway-1"]        ["ray.cloud.dash-1.oss"]     2d21h
default     ray-cloud-1-serve-virtualservice    ["dashboards-gateway-1"]        ["ray.cloud.serve-1.oss"]    3s
default     ray-local-1-dash-virtualservice     ["dashboards-gateway-1"]        ["ray.local.dash-1.oss"]     2d21h
default     ray-local-1-serve-virtualservice    ["dashboards-gateway-1"]        ["ray.local.serve-1.oss"]    2d21h
default     ray-local-2-dash-virtualservice     ["dashboards-gateway-1"]        ["ray.local.dash-2.oss"]     2d21h
default     ray-local-2-serve-virtualservice    ["dashboards-gateway-1"]        ["ray.local.serve-2.oss"]    2d21h
default     ray-local-3-dash-virtualservice     ["dashboards-gateway-1"]        ["ray.local.dash-3.oss"]     2d21h
default     ray-local-3-serve-virtualservice    ["dashboards-gateway-1"]        ["ray.local.serve-3.oss"]    2d21h
default     redis-virtualservice                ["redis-gateway"]               ["*"]                        10d
default     webui-virtualservice                ["dashboards-gateway-1"]        ["webui.oss"]                10d
mlflow      mlflow                              ["kubeflow/kubeflow-gateway"]   ["*"]                        14d
```
To confirm that forwarding is working, we can deploy a curl pod found at deployments/networking inside the cluster with:
```
kubectl apply -f curl.yaml
```
To use it go inside the container with:
```
# Check its running
kubectl get pods

# Open container shell
kubectl exec -it curl-pod -- /bin/sh

# Run command
curl http://ray-local-1-dash.integration.svc.cluster.local:8100

# Close the shell
exit
```
These should give the following prints for the local clusters:
```
curl http://ray-local-1-dash.integration.svc.cluster.local:8100
curl http://ray-local-2-dash.integration.svc.cluster.local:8101
curl http://ray-local-3-dash.integration.svc.cluster.local:8102
<!doctype html><html lang="en"><head><meta charset="utf-8"/><link rel="shortcut icon" href="./favicon.ico"/><meta name="viewport" content="width=device-width,initial-scale=1"/><title>Ray Dashboard</title><script defer="defer" src="./static/js/main.b3a3eb48.js"></script><link href="./static/css/main.388a904b.css" rel="stylesheet"></head><body><noscript>You need to enable JavaScript to run this app.</noscript><div id="root"></div></body></html>
```
We can confirm the virtual services work with the following curls:
```
# VM Curl
curl -v -H "Host: ray.cloud.dash-1.oss" http://localhost:7001
curl -v -H "Host: ray.local.dash-1.oss" http://localhost:7001
curl -v -H "Host: ray.local.dash-2.oss" http://localhost:7001
curl -v -H "Host: ray.local.dash-3.oss" http://localhost:7001
# logs
*   Trying 127.0.0.1:7001...
* Connected to localhost (127.0.0.1) port 7001 (#0)
> GET / HTTP/1.1
> Host: ray.local.dash-3.oss
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< cache-control: no-store
< content-type: text/html
< etag: "1866f1aa3cb53400-1be"
< last-modified: Sat, 20 Sep 2025 08:53:38 GMT
< content-length: 446
< accept-ranges: bytes
< date: Fri, 10 Oct 2025 11:28:45 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 31
< 
* Connection #0 to host localhost left intact
```
Now we only need to update /etc/hosts to have the following:
```
(vm_floating_ip) ray.cloud.dash-1.oss
(vm_floating_ip) ray.cloud.serve-1.oss
(vm_floating_ip) ray.local.dash-1.oss
(vm_floating_ip) ray.local.serve-1.oss
(vm_floating_ip) ray.local.dash-2.oss
(vm_floating_ip) ray.local.serve-2.oss
(vm_floating_ip) ray.local.dash-3.oss
(vm_floating_ip) ray.local.serve-3.oss
```
You should now be able to see dashboards at:
- http://ray.cloud.dash-1.oss:7001
- http://ray.local.dash-1.oss:7001
- http://ray.local.dash-2.oss:7001
- http://ray.local.dash-3.oss:7001

With these we are now ready to send jobs into Kuberay cluster that uses Docker compose Ray clusters.
## Local-Cloud Ray Orhestraction

```

```

```

```
