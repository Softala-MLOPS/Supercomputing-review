# Tutorial for integrated distributed computing in MLOps (4/10)
This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.
## Compose Apache Airflow Setup
Useful material:
- https://github.com/apache/airflow
- https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html
- https://pypi.org/project/apache-airflow-client/
- https://airflow.apache.org/docs/docker-stack/index.html
- https://docs.docker.com/compose/how-tos/use-secrets/
- https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#connections
- https://docs.csc.fi/cloud/pouta/launch-vm-from-web-gui/
- https://docs.csc.fi/cloud/pouta/connecting-to-vm/
- https://docs.csc.fi/computing/connecting/https://github.com/apache/airflow
- https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html
- https://pypi.org/project/apache-airflow-client/
- https://airflow.apache.org/docs/docker-stack/index.html
- https://docs.docker.com/compose/how-tos/use-secrets/
- https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#connections
- https://docs.csc.fi/cloud/pouta/launch-vm-from-web-gui/
- https://docs.csc.fi/cloud/pouta/connecting-to-vm/
- https://docs.csc.fi/computing/connecting/

To remove the need to hard code automation and constantly rebuild FastAPI frontends and Celery backends to enable changes in the workflow, we will use Apache Airflow to provide a way to create Python based workflows via both JupyterLab and IDE of your choosing. You find the initial things at applications/airflow that require setup. We need to create folders
```
!mkdir applications/submitter/airflow/dags
!mkdir applications/submitter/airflow/logs
!mkdir applications/submitter/airflow/plugins
!mkdir applications/submitter/airflow/config
```
create the .env file
```
!echo -e "AIRFLOW_UID=$(id -u)" > applications/submitter/airflow/.env
```
Building a airflow docker image. Be aware that you can modify and add pip packages into the image using the packages.txt file. You can also find and modify the used image at
```
x-airflow-common:
    image: submitter-airflow:3.0.4
```
```
!docker build -t submitter-airflow:3.0.4 ./applications/submitter/airflow
```
create default configuration
```
!docker compose -f applications/submitter/airflow/docker-compose.yaml run airflow-cli airflow config list
```
Modify the configuration by using CTRL + F to find refresh_interval and change it from 300 to 30, which speeds up the time for Airflow to add new DAGs. Then setup database:
```
!docker compose -f applications/submitter/airflow/docker-compose.yaml up airflow-init
```
If the following shows exited with code 0, you can then make Airflow run with
```
cd applications/submitter/airflow
docker compose up
```
You can find the UI at http://127.0.0.1:8080, where you need to give the credentials airflow-airflow. There the pages of our intrest are
- dags
- security

We will go into their details later. Now we only need to add secrets into the Airflow. Put the following YAML into your .ssh file and fill the necessery details:
```
platforms:
  cloud:
    cpouta-1:
      address: '(fill)'
      user: '(fill)'
      type: 'ssh'
      key: 'local-cloud'
  hpc:
    puhti:
      address: 'puhti.csc.fi'
      user: '(fill)'
      type: 'ssh'
      key: 'local-hpc'
    mahti:
      address: 'mahti.csc.fi'
      user: '(fill)'
      type: 'ssh'
      key: 'local-hpc'
    lumi:
      address: 'lumi.csc.fi'
      user: '(fill)'
      type: 'ssh'
      key: 'local-hpc'
connections:
  ssh:
    local-cloud:
      path: '/run/secrets/local-cloud-ssh'
      phrase: '(fill)'
    local-hpc:
      path: '/run/secrets/local-hpc-ssh'
      phrase: '(fill)'
    cloud-hpc:
      path: '/run/secrets/cloud-hpc-ssh'
      phrase: 'empty'
```
You also need to generate the necessery SSH private keys for local-cloud, local-hpc and cloud-hpc interactions. In the case of CSC ecosystem, generate two key pairs to CPouta and generate a key pair to MyCSC to access puhti, mahti and LUMI. When you have moved the private keys into your .ssh folder, you can move them into a docker container with the following YAML modifications:
```
x-airflow-common:
    secrets:
        - local-cloud-ssh
        - local-hpc-ssh
        - cloud-hpc-ssh

secrets:
  local-cloud-ssh:
    file: /home/(fill)/.ssh/local-cpouta.pem
  local-hpc-ssh:
    file: /home/(fill)/.ssh/local-hpc.pem
  cloud-hpc-ssh:
    file: /home/(fill)/.ssh/cpouta-hpc.pem
```
After you have added secrets at the end of the yaml and list of secrets at the end of the container configuration, read the secrets with the following:
```
import json
import yaml

secret_yaml_path = '/home/(fill)/.ssh/airflow-secrets.yaml'
secret_yaml_dict = None
with open(secret_yaml_path, 'r') as f:
    secret_yaml_dict = yaml.safe_load(f)
```
Generate connection CLI commands with this function:
```
import os

def airflow_cli_connections(
    secret_dict: any
): 
    secret_platforms = secret_dict['platforms']
    secret_connections = secret_dict['connections']
    
    airflow_connections = []
    for p_type, values in secret_platforms.items():
        for name, parameters  in values.items():
            connection_type = parameters['type']
            if connection_type == 'ssh':
                connection_host = parameters['address']
                connection_user = parameters['user']
                connection_key = parameters['key']
                key_parameters = secret_connections[connection_type][connection_key]
                connection_path = key_parameters['path']
                connection_phrase = key_parameters['phrase']
                connection_id = p_type + '-' + name
    
                values = {
                    'conn_type': 'ssh',
                    'login': connection_user,
                    'password': connection_phrase,
                    'host': connection_host,
                    'port': 22,
                    'extra': {
                        'key_file': connection_path
                    }
                }
    
                compose_command = "-f " + os.path.abspath('applications/submitter/airflow/docker-compose.yaml')
                connection_name = "'" + connection_id + "'"
                connection_json = "'" + json.dumps(values) + "'"
                cli_command = [
                    "docker", 
                    "compose", 
                    compose_command,
                    "run", 
                    "airflow-worker",
                    "airflow", 
                    "connections", 
                    "add",
                    connection_name,
                    "--conn-json",
                    connection_json
                ]
            
                airflow_connections.append(' '.join(cli_command))
    return airflow_connections
```
The commands are run with the following function:
```
import subprocess

def subprocess_run_command(
    command: any
) -> any: 
    resulted_print = subprocess.run(
        command,
        shell = True,
        stdout = subprocess.PIPE,
        stderr = subprocess.PIPE
    )

    print_output = resulted_print.stdout.decode('utf-8').split('\n')
    print_errors = resulted_print.stderr.decode('utf-8').split('\n')
    return print_output, print_errors
```
Confirm that you have shutdown possible airflow containers and run the following blocks
```
airflow_connections = airflow_cli_connections(
    secret_dict = secret_yaml_dict
)
```
```
for connection in airflow_connections:
    result_print, error_print = subprocess_run_command(
        command = connection
    )
    print(result_print)
```
You can confirm that the secrets were loaded by going to http://localhost:8080/connections, which shows a list of available secrets. You can also edit or delete them as needed.
## Airflow DAGs
Useful material:
- Packages:
    - https://pypi.org/project/apache-airflow/
    - https://pypi.org/project/paramiko/#history
    - https://pypi.org/project/apache-airflow-providers-sftp/
    - https://pypi.org/project/apache-airflow-client/
- Basics:
    - https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html
    - https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/params.html
    - https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html
    - https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html
- Docker
    - https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html
    - https://airflow.apache.org/docs/docker-stack/build.html#extending-the-image
    - https://stackoverflow.com/questions/66699394/airflow-how-to-get-pip-packages-installed-via-their-docker-compose-yml
    - https://docs.docker.com/compose/how-tos/use-secrets/
- CLI
    - https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#connections
    - https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/mask-sensitive-values.html
Connections:
    - https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/connections.html
- Operators
    - https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html
    - https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/operators/index.html
    - https://stackoverflow.com/questions/77357880/airflow-triggerdagrunoperator-does-nothing
- SSH
    - https://github.com/paramiko/paramiko/issues/2537
- Hooks
    - https://airflow.apache.org/docs/apache-airflow/stable/operators-and-hooks-ref.html
    - https://airflow.apache.org/docs/apache-airflow-providers-ssh/stable/_api/airflow/providers/ssh/hooks/ssh/index.html
    - https://airflow.apache.org/docs/apache-airflow-providers-sftp/stable/_api/airflow/providers/sftp/hooks/sftp/index.html
- API
    - https://hevodata.com/learn/airflow-rest-api/
    - https://stackoverflow.com/questions/60055151/how-to-trigger-an-airflow-dag-run-from-within-a-python-script
    - https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#operation/get_list_dag_runs_batch
    - https://github.com/apache/airflow-client-python/blob/main/docs/DAGApi.md#get_dags
    - https://github.com/apache/airflow-client-python/blob/main/docs/DAGRunApi.md
    - https://github.com/apache/airflow-client-python/blob/main/docs/TaskInstanceApi.md#get_log
    - https://github.com/apache/airflow-client-python/blob/main/docs/TriggerDAGRunPostBody.md

Directed acyclic graph (DAG) is a workflow that executes code step by step in a similar manner to orhestration platforms such as Kubeflow pipelines. For our use case airflow provides a easy way to modify the automation used by both Submitter and Forwarder as the workflow develops, since it removes the need to constantly rebuild Submitter and Forwarder docker images to update their hard coded automation by moving that automation into easily edited execution steps. The most important consepts are:
- DAG creation
- Function importing
- DAG configuration
- DAG input
- Code parameters
- Tasks
- Task depedencies
- TaskFlow
- Context
- Operators
- DAG triggering
- Hooks
- SSH
- Airflow API

We can create DAGs by writing a .py file in the airflow /dags folder in the following way:
```
%%writefile (fill)/studying/applications/submitter/airflow/dags/hello_world_dag.py
from airflow.sdk import DAG, task

with DAG(
    dag_id = "tutorial-hello-world", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    tags = ["integration"]
) as dag:
    @task()
    def hello_world():
        print("airflow")

    hello_world()
```
In this DAG we see the following critical components
- Airflow imports -> from airflow.sdk import DAG, task
- Context manager DAG -> with DAG() as dag
- DAG name -> dag_id = "tutorial-hello-world"
- DAG scheduling start -> start_date = None
- DAG scheduling amount -> schedule = None
- DAG backfilling -> catchup = False
- DAG automatic running -> is_paused_upon_creation = False

Be aware that there are many ways to declare DAGs with the shown way being the widely used for simple DAGs. You can make this DAG run by going to http://localhost:8080/dags and doing the following:
- Click filter by tag
- Find or write 'integration' tag
- Select tutorial-hello-world
- Click the blue trigger button on the right top corner
- Wait, click runs, click the first instance and click hello_world
- Click the task logs to see the following:
    ```
    [2025-09-23, 14:09:32] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
    [2025-09-23, 14:09:32] INFO - Filling up the DagBag from /opt/airflow/dags/hello_world_dag.py: source="airflow.models.dagbag.DagBag"
    [2025-09-23, 14:09:33] INFO - Task instance is in running state: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO - Current task name:hello_world: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
    [2025-09-23, 14:09:33] INFO - Dag name:tutorial-hello-world: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO - airflow: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO - Task instance in success state: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
    [2025-09-23, 14:09:33] INFO - Task operator:<Task(_PythonDecoratedOperator): hello_world>: chan="stdout": source="task"
    ```

Now that we know how to create, run and check DAGs, lets see how we can provide parameters, import functions, create multiple tasks, use context and pass task outputs. We first need to create a folder for the functions:
```
!mkdir applications/submitter/airflow/dags/functions
```
We can store functions with the following way:
```
%%writefile (fill)/applications/submitter/airflow/dags/functions/general.py
import re

def set_formatted_user(
    user: str   
) -> any:
    return re.sub(r'[^a-z0-9]+', '-', user)
```
We can finally create the following DAG:
```
%%writefile (fill)/applications/submitter/airflow/dags/dependency_dag.py

from airflow.sdk import DAG, task, get_current_context

from functions.general import set_formatted_user

with DAG(
    dag_id = "tutorial-dependency", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    params = {
        "token": "test",
        "user": "user@example.com",
        "message": "hello",
        "target": "airflow"
    },
    tags = ["integration"]
) as dag:
    @task()
    def setup(
        params: dict
    ):
        formatted_user = set_formatted_user(
            user = params['user']
        )
        message = formatted_user + ' used ' + params['token']
        return message

    @task()
    def preprocess(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' and said ' +  airflow_context['params']['message']
        return message
        
    @task()
    def utilizer(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' to ' + airflow_context['params']['target']
        print(message)

    initial_message = setup()
    partial_message = preprocess(
        message = initial_message
    )
    utilizer(
        message = partial_message
    )
```
When you again trigger this DAG, you are now given options to provide input parameters that are different from the set defaults. If we run it with defaults, we get the following logs for each task:
```
setup
[2025-09-23, 14:53:12] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 14:53:12] INFO - Filling up the DagBag from /opt/airflow/dags/dependency_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 14:53:13] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 14:53:13] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 14:53:13] INFO - Current task name:setup: chan="stdout": source="task"
[2025-09-23, 14:53:13] INFO - Dag name:tutorial-dependency: chan="stdout": source="task"
[2025-09-23, 14:53:13] INFO - Done. Returned value was: user-example-com used test: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 14:53:13] INFO - Pushing xcom: ti="RuntimeTaskInstance(id=UUID('0199766b-a98d-731a-8289-0fa0d1a2c515'), task_id='setup', dag_id='tutorial-dependency', run_id='manual__2025-09-23T11:53:11.288275+00:00', try_number=1, map_index=-1, hostname='483f2a5d7972', context_carrier={}, task=<Task(_PythonDecoratedOperator): setup>, bundle_instance=LocalDagBundle(name=dags-folder), max_tries=0, start_date=datetime.datetime(2025, 9, 23, 11, 53, 11, 966876, tzinfo=datetime.timezone.utc), end_date=None, state=<TaskInstanceState.RUNNING: 'running'>, is_mapped=False, rendered_map_index=None, log_url='http://localhost:8080/dags/tutorial-dependency/runs/manual__2025-09-23T11%3A53%3A11.288275%2B00%3A00/tasks/setup?try_number=1')": source="task"
[2025-09-23, 14:53:13] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 14:53:13] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 14:53:13] INFO - Task operator:<Task(_PythonDecoratedOperator): setup>: chan="stdout": source="task"
preprocess
[2025-09-23, 14:53:14] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 14:53:14] INFO - Filling up the DagBag from /opt/airflow/dags/dependency_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 14:53:14] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 14:53:14] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 14:53:14] INFO - Current task name:preprocess: chan="stdout": source="task"
[2025-09-23, 14:53:14] INFO - Dag name:tutorial-dependency: chan="stdout": source="task"
[2025-09-23, 14:53:14] INFO - Done. Returned value was: user-example-com used test and said hello: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 14:53:14] INFO - Pushing xcom: ti="RuntimeTaskInstance(id=UUID('0199766b-a98e-796f-bddf-bacd1ea231e7'), task_id='preprocess', dag_id='tutorial-dependency', run_id='manual__2025-09-23T11:53:11.288275+00:00', try_number=1, map_index=-1, hostname='483f2a5d7972', context_carrier={}, task=<Task(_PythonDecoratedOperator): preprocess>, bundle_instance=LocalDagBundle(name=dags-folder), max_tries=0, start_date=datetime.datetime(2025, 9, 23, 11, 53, 14, 128528, tzinfo=datetime.timezone.utc), end_date=None, state=<TaskInstanceState.RUNNING: 'running'>, is_mapped=False, rendered_map_index=None, log_url='http://localhost:8080/dags/tutorial-dependency/runs/manual__2025-09-23T11%3A53%3A11.288275%2B00%3A00/tasks/preprocess?try_number=1')": source="task"
[2025-09-23, 14:53:14] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 14:53:14] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 14:53:14] INFO - Task operator:<Task(_PythonDecoratedOperator): preprocess>: chan="stdout": source="task"
utilizer
[2025-09-23, 14:53:15] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 14:53:15] INFO - Filling up the DagBag from /opt/airflow/dags/dependency_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 14:53:15] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO - Current task name:utilizer: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO - Dag name:tutorial-dependency: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO - user-example-com used test and said hello to airflow: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 14:53:15] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 14:53:15] INFO - Task operator:<Task(_PythonDecoratedOperator): utilizer>: chan="stdout": source="task"
```
The way this DAG was constructed used TaskFlow API. The old way would have done the same in the following way:
```
from airflow.providers.standard.operators.python import PythonOperator

def setup(
    params: dict
):
    formatted_user = set_formatted_user(
        user = params['user']
    )
    message = formatted_user + ' used ' + params['token']
    return message

def preprocess(
    message: str
):
    airflow_context = get_current_context()
    message += ' and said ' +  airflow_context['params']['message']
    return message
    
def utilizer(
    message: str
):
    airflow_context = get_current_context()
    message += ' to ' + airflow_context['params']['target']
    print(message)

with DAG() as dag:
    setup_task = PythonOperator(
        task_id = "setup", 
        python_callable = setup
    )
    preprocess_task = PythonOperator(
        task_id = "preprocess", 
        python_callable = preprocess
    )
    utilizer_task = PythonOperator(
        task_id = "utilizer", 
        python_callable = utilizer
    )
    
    setup_task >> preprocess_task >> utilizer_task
```
We still need to use this way when using operators with tasks, which are task templates for specific operations. There are many types of operators ranging from HttpOperator to DockerOperator created by the community that provide mature ways to create complex DAGs. In the case of Python code it is recommended to use the TaskFlow way, while for executing code such as Bash one should use BashOperator. For our use case the most intresting operator is TriggerDagRUnOperator that we can use to create a complex DAG that triggers smaller DAGs by defining the sub DAGs:
```
%%writefile (fill)/applications/submitter/airflow/dags/sub_dag_1.py
from airflow.sdk import DAG, task, get_current_context

with DAG(
    dag_id = "tutorial-sub-1", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    params = {
        "message": "none",
        "package": "none",
        "target": "none"
    },
    tags = ["integration"]
) as dag:
    @task()
    def sub_setup(
        params: str
    ):
        message = params['message'] + ' to sub-1'
        return message
        
    @task()
    def sub_preprocess(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' that had ' +  airflow_context['params']['package']
        return message

    @task()
    def sub_utilizer(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' to ' + airflow_context['params']['target']
        print(message)

    initial_message = sub_setup()
    partial_message = sub_preprocess(
        message = initial_message
    )
    sub_utilizer(
        message = partial_message
    )
```
```
%%writefile (fill)/applications/submitter/airflow/dags/sub_dag_2.py
from airflow.sdk import DAG, task, get_current_context

with DAG(
    dag_id = "tutorial-sub-2", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    params = {
        "message": "none",
        "package": "none",
        "target": "none"
    },
    tags = ["integration"]
) as dag:
    @task()
    def sub_setup(
        params: str
    ):
        message = params['message'] + ' to sub-2'
        return message
        
    @task()
    def sub_preprocess(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' that had ' +  airflow_context['params']['package']
        return message

    @task()
    def sub_utilizer(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' to ' + airflow_context['params']['target']
        print(message)

    initial_message = sub_setup()
    partial_message = sub_preprocess(
        message = initial_message
    )
    sub_utilizer(
        message = partial_message
    )
```
```
%%writefile (fill)/applications/submitter/airflow/dags/sub_dag_3.py
from airflow.sdk import DAG, task, get_current_context

with DAG(
    dag_id = "tutorial-sub-3", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    params = {
        "message": "none",
        "package": "none",
        "target": "none"
    },
    tags = ["integration"]
) as dag:
    @task()
    def sub_setup(
        params: str
    ):
        message = params['message'] + ' to sub-3'
        return message
        
    @task()
    def sub_preprocess(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' that had ' +  airflow_context['params']['package']
        return message

    @task()
    def sub_utilizer(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' to ' + airflow_context['params']['target']
        print(message)

    initial_message = sub_setup()
    partial_message = sub_preprocess(
        message = initial_message
    )
    sub_utilizer(
        message = partial_message
    )
```
We can use them in the following way:
```
%%writefile (fill)/applications/submitter/airflow/dags/main_dag.py
from airflow.sdk import DAG, task, get_current_context

from airflow.providers.standard.operators.trigger_dagrun import TriggerDagRunOperator

from functions.general import set_formatted_user

with DAG(
    dag_id = "tutorial-main", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    params = {
        "token": "test",
        "user": "user@example.com",
        "message": "hello",
        "target": "airflow"
    },
    tags = ["integration"]
) as dag:
    @task()
    def main_setup(
        params: str
    ):
        formatted_user = set_formatted_user(
            user = params['user']
        )
        message = formatted_user + ' used ' + params['token']
        return message
        
    @task()
    def main_preprocess(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' and said ' +  airflow_context['params']['message']
        return message

    @task()
    def main_utilizer(
        message: str
    ):
        airflow_context = get_current_context()
        message += ' to ' + airflow_context['params']['target']
        print(message)

    initial_message = main_setup()
    partial_message = main_preprocess(
        message = initial_message
    )
    main_utilizer(
        message = partial_message
    )
    
    sub_1 = TriggerDagRunOperator(
        task_id = 'trigger_sub_1',
        trigger_dag_id = 'tutorial-sub-1', 
        conf = {
            'message': partial_message,
            'package': 'object',
            'target': 'Allas'
        },
        wait_for_completion = False,
        poke_interval = 20,
        reset_dag_run = True
    )

    sub_2 = TriggerDagRunOperator(
        task_id = 'trigger_sub_2',
        trigger_dag_id = 'tutorial-sub-2', 
        conf = {
            'message': partial_message,
            'package': 'job',
            'target': 'LUMI'
        },
        wait_for_completion = False,
        poke_interval = 20,
        reset_dag_run = True
    )

    sub_3 = TriggerDagRunOperator(
        task_id = 'trigger_sub_3',
        trigger_dag_id = 'tutorial-sub-3', 
        conf = {
            'message': partial_message,
            'package': 'Ray',
            'target': 'CPouta'
        },
        wait_for_completion = False,
        poke_interval = 20,
        reset_dag_run = True
    )

    initial_message >> partial_message >> sub_1 >> sub_2 >> sub_3
```
When you trigger this DAG, you notice that the logs are now separated between the DAGs, giving the following for utilizator functions:
```
tutorial-main
[2025-09-23, 15:54:17] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 15:54:17] INFO - Filling up the DagBag from /opt/airflow/dags/main_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 15:54:17] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO - Current task name:main_utilizer: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO - Dag name:tutorial-main: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO - user-example-com used test and said hello to airflow: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 15:54:17] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 15:54:17] INFO - Task operator:<Task(_PythonDecoratedOperator): main_utilizer>: chan="stdout": source="task"
tutorial-sub-1
[2025-09-23, 15:54:21] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 15:54:21] INFO - Filling up the DagBag from /opt/airflow/dags/sub_dag_1.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 15:54:21] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO - Current task name:sub_utilizer: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO - Dag name:tutorial-sub-1: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO - user-example-com used test and said hello to sub-1 that had object to Allas: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 15:54:21] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 15:54:21] INFO - Task operator:<Task(_PythonDecoratedOperator): sub_utilizer>: chan="stdout": source="task"
tutorial-sub-2
[2025-09-23, 15:54:22] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 15:54:22] INFO - Filling up the DagBag from /opt/airflow/dags/sub_dag_2.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 15:54:22] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO - Current task name:sub_utilizer: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO - Dag name:tutorial-sub-2: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO - user-example-com used test and said hello to sub-2 that had job to LUMI: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 15:54:22] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 15:54:22] INFO - Task operator:<Task(_PythonDecoratedOperator): sub_utilizer>: chan="stdout": source="task"
tutorial-sub-3
[2025-09-23, 15:54:23] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-23, 15:54:23] INFO - Filling up the DagBag from /opt/airflow/dags/sub_dag_3.py: source="airflow.models.dagbag.DagBag"
[2025-09-23, 15:54:24] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO - Current task name:sub_utilizer: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO - Dag name:tutorial-sub-3: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-23, 15:54:24] INFO - user-example-com used test and said hello to sub-3 that had Ray to CPouta: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-23, 15:54:24] INFO - Task operator:<Task(_PythonDecoratedOperator): sub_utilizer>: chan="stdout": source="task"
```
As we can see from the amount of logs, understanding and debugging DAGs can become complex quite fast. For this reason it is recommended to create prints with keywords to make their search easier. The final things we need to understand about airflow for our use case are hooks and API. Hooks are high-level interfaces to external platforms that remove the need to write low-level code. In our case the most intresting hook is the SSHHook that eanbles us to connect to a remote computers via SSH and SFTPHook that uses SSH to transfer files between computers. For example, we can get the list of files in a remote directory in the following way:
```
%%writefile (fill)/applications/submitter/airflow/dags/interaction_dag.py
from airflow.sdk import DAG, task, get_current_context

from airflow.providers.ssh.operators.ssh import SSHHook
from airflow.providers.sftp.hooks.sftp import SFTPHook

with DAG(
    dag_id = "tutorial-interaction", 
    start_date = None, 
    schedule = None,
    catchup = False,
    is_paused_upon_creation = False,
    params = {
        "connection": "cloud-cpouta-1",
        "remote-path": "/home/ubuntu"
    },
    tags = ["integration"]
) as dag:
    @task()
    def ssh_interaction(
        commands: any
    ):
        airflow_context = get_current_context()
        hook = SSHHook(
            ssh_conn_id = airflow_context['params']['connection']
        )
        client = hook.get_conn()
        command_results = []
        for command in commands:
            stdin, stdout, stdeer = client.exec_command(command)
            command_results.append(stdout.read().decode())
        client.close()
        print(command_results)

    @task()
    def sftp_interaction():
        airflow_context = get_current_context()
        sftp_hook = SFTPHook(
            ssh_conn_id = airflow_context['params']['connection']
        )
        files = sftp_hook.list_directory(
            path = airflow_context['params']['remote-path']
        )
        print(files)
        
    ssh_task = ssh_interaction(
        commands = [
            'pwd',
            'ls'
        ]
    )
    stfp_task = sftp_interaction()

    ssh_task >> stfp_task
```
When you check the task logs at http://localhost:8080/dags, you should see something like this:
```
ssh_interaction
[2025-09-24, 15:04:38] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-24, 15:04:38] INFO - Filling up the DagBag from /opt/airflow/dags/interaction_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-24, 15:04:39] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-24, 15:04:39] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-24, 15:04:39] INFO - Current task name:ssh_interaction: chan="stdout": source="task"
[2025-09-24, 15:04:39] INFO - Dag name:tutorial-interaction: chan="stdout": source="task"
[2025-09-24, 15:04:39] WARNING - /home/airflow/.local/lib/python3.12/site-packages/airflow/models/connection.py:471: DeprecationWarning: Using Connection.get_connection_from_secrets from `airflow.models` is deprecated.Please use `from airflow.sdk import Connection` instead
  warnings.warn(
: source="py.warnings"
[2025-09-24, 15:04:39] INFO - Connection Retrieved 'cloud-cpouta-1': source="airflow.hooks.base"
[2025-09-24, 15:04:39] WARNING - No Host Key Verification. This won't protect against Man-In-The-Middle attacks: source="airflow.task.hooks.airflow.providers.ssh.hooks.ssh.SSHHook"
[2025-09-24, 15:04:39] INFO - Connected (version 2.0, client OpenSSH_8.9p1): source="paramiko.transport"
[2025-09-24, 15:04:39] INFO - Authentication (publickey) successful!: source="paramiko.transport"
[2025-09-24, 15:04:40] INFO - ['/home/ubuntu\n', 'cloud-hpc-oss-mlops-platform\ncuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb\nistio-ingressgateway.yaml\nistiod.yaml\nmulti-cloud-hpc-oss-mlops-platform\noperator-test.yaml\n']: chan="stdout": source="task"
[2025-09-24, 15:04:40] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-24, 15:04:40] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-24, 15:04:40] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-24, 15:04:40] INFO - Task operator:<Task(_PythonDecoratedOperator): ssh_interaction>: chan="stdout": source="task"
sftp_interaction
[2025-09-24, 15:04:40] INFO - DAG bundles loaded: dags-folder, example_dags: source="airflow.dag_processing.bundles.manager.DagBundlesManager"
[2025-09-24, 15:04:40] INFO - Filling up the DagBag from /opt/airflow/dags/interaction_dag.py: source="airflow.models.dagbag.DagBag"
[2025-09-24, 15:04:41] INFO - Task instance is in running state: chan="stdout": source="task"
[2025-09-24, 15:04:41] INFO -  Previous state of the Task instance: TaskInstanceState.QUEUED: chan="stdout": source="task"
[2025-09-24, 15:04:41] WARNING - /home/airflow/.local/lib/python3.12/site-packages/airflow/models/connection.py:471: DeprecationWarning: Using Connection.get_connection_from_secrets from `airflow.models` is deprecated.Please use `from airflow.sdk import Connection` instead
  warnings.warn(
: source="py.warnings"
[2025-09-24, 15:04:41] INFO - Current task name:sftp_interaction: chan="stdout": source="task"
[2025-09-24, 15:04:41] INFO - Dag name:tutorial-interaction: chan="stdout": source="task"
[2025-09-24, 15:04:41] INFO - Connection Retrieved 'cloud-cpouta-1': source="airflow.hooks.base"
[2025-09-24, 15:04:41] WARNING - No Host Key Verification. This won't protect against Man-In-The-Middle attacks: source="airflow.task.hooks.airflow.providers.sftp.hooks.sftp.SFTPHook"
[2025-09-24, 15:04:41] INFO - Connected (version 2.0, client OpenSSH_8.9p1): source="paramiko.transport"
[2025-09-24, 15:04:41] INFO - Authentication (publickey) successful!: source="paramiko.transport"
[2025-09-24, 15:04:41] INFO - [chan 0] Opened sftp connection (server version 3): source="paramiko.transport.sftp"
[2025-09-24, 15:04:41] INFO - [chan 0] sftp session closed.: source="paramiko.transport.sftp"
[2025-09-24, 15:04:41] INFO - Done. Returned value was: None: source="airflow.task.operators.airflow.providers.standard.decorators.python._PythonDecoratedOperator"
[2025-09-24, 15:04:41] INFO - ['.bash_history', '.bash_logout', '.bashrc', '.cache', '.config', '.kube', '.local', '.profile', '.ssh', '.sudo_as_admin_successful', 'cloud-hpc-oss-mlops-platform', 'cuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb', 'istio-ingressgateway.yaml', 'istiod.yaml', 'multi-cloud-hpc-oss-mlops-platform', 'operator-test.yaml']: chan="stdout": source="task"
[2025-09-24, 15:04:41] INFO - Task instance in success state: chan="stdout": source="task"
[2025-09-24, 15:04:41] INFO -  Previous state of the Task instance: TaskInstanceState.RUNNING: chan="stdout": source="task"
[2025-09-24, 15:04:41] INFO - Task operator:<Task(_PythonDecoratedOperator): sftp_interaction>: chan="stdout": source="task"
```
As we can see i nthe logs, the connection to CPouta was a succcess. If you have other connections, you can test them simply by giving the correct name and checking the default path of the remote machine with
```
pwd
```
We now have all the necessery ideas for using Airflow for creating automation DAGs, which we now only need to learn to active with other software. This can be done through its API that requires aquiring a token for interactions. We can do that using the following functions:
```
import airflow_client.client
import requests
from pydantic import BaseModel

class Token(BaseModel):
    access_token: str

def airflow_get_token(
    host: str,
    username: str,
    password: str,
) -> any:        
    url = f"{host}/auth/token"
    payload = {
        "username": username,
        "password": password,
    }
    headers = {
        "Content-Type": "application/json"
    }
    response = requests.post(
        url, 
        json = payload, 
        headers = headers
    )
    if response.status_code != 201:
        return None
    response_success = Token(**response.json())
    return response_success.access_token

def airflow_setup_configuration(
    airflow_host: str,
    airflow_username: str,
    airflow_password: str
) -> any:
    
    configuration = airflow_client.client.Configuration(
        host = airflow_host
    )
    configuration.access_token = airflow_get_token(
        host = airflow_host,
        username = airflow_username,
        password = airflow_password
    )
    return configuration 
```
With these we setup the airflow configuration with the following:
```
airflow_config = airflow_setup_configuration(
    airflow_host = 'http://127.0.0.1:8080',
    airflow_username = 'airflow',
    airflow_password = 'airflow'
)
```
We can use it with the following functions:
```
import airflow_client.client

def airflow_get_dags(
    airflow_configuration: any,
    limit: int, 
    tags: list,
    order_by: str
) -> any:
    with airflow_client.client.ApiClient(airflow_configuration) as api_client:
        api_instance = airflow_client.client.DAGApi(api_client)
        
        try:
            dag_list = api_instance.get_dags(
                limit = limit,
                tags = tags,
                order_by = order_by
            )

            formated_list = {}
            for dag in dag_list.dags:
                dag_id = dag.dag_id
                dag_tags = []
                for tag in dag.tags:
                    dag_tags.append(tag.name)
                formated_list[dag_id] = {
                    'tags': dag_tags
                }

            return formated_list
        except Exception as e:
            return {}

def airflow_get_runs(
    airflow_configuration: any,
    dag_id: str,
    limit: int,
    order_by: str
) -> any:
    with airflow_client.client.ApiClient(airflow_configuration) as api_client:
        api_instance = airflow_client.client.DagRunApi(api_client)
        
        try:
            run_list = api_instance.get_dag_runs(
                dag_id = dag_id, 
                limit = limit,
                order_by = order_by
            )

            formated_list = []
            for run in run_list.dag_runs:
                time = (run.end_date-run.start_date).total_seconds()
                formated_list.append({
                    'id': run.dag_run_id,
                    'type': run.run_type.value,
                    'state': run.state.value,
                    'start': run.start_date.strftime(f'%f-%S-%M-%H-%d-%m-%Y'),
                    'end': run.end_date.strftime(f'%f-%S-%M-%H-%d-%m-%Y'),
                    'time': time,
                    'trigger': run.triggered_by.value
                })

            return formated_list
        except Exception as e:
            return {}

def airflow_get_tasks(
    airflow_configuration: any,
    dag_id: str,
    order_by: str
) -> any:
    with airflow_client.client.ApiClient(airflow_configuration) as api_client:
        api_instance = airflow_client.client.TaskApi(api_client)
        
        try:
            task_list = api_instance.get_tasks(
                dag_id = dag_id, 
                order_by = order_by
            )
            
            formated_list = []
            for task in task_list.tasks:
                formated_parameters = {}
                for key in task.params.keys():
                    formated_parameters[key] = task.params[key]['value']
                    
                formated_list.append({
                    'id': task.task_id,
                    'name': task.task_display_name,
                    'parameters': formated_parameters,
                    'downstream': task.downstream_task_ids
                })

            return formated_list
        except Exception as e:
            return {}

def airflow_dag_logs(
    airflow_configuration: any,
    dag_id: str,
    dag_run_id: str,
    task_id: str,
    try_number: int
) -> any:
    with airflow_client.client.ApiClient(airflow_configuration) as api_client:
        api_instance = airflow_client.client.TaskInstanceApi(api_client)
        
        try:
            task_logs = api_instance.get_log(
                dag_id = dag_id, 
                dag_run_id = dag_run_id, 
                task_id = task_id, 
                try_number = try_number
            )
            
            formatted_logs = []
            for tuple_value in task_logs.content:
                if tuple_value[0] == 'actual_instance':
                    for row in tuple_value[1]:
                        log = row.event
                        formatted_logs.append(log)
            return formatted_logs
        except Exception as e:
            return {}

def airflow_trigger_dag(
    airflow_configuration: any,
    dag_id: str,
    parameters: dict
) -> any:
    with airflow_client.client.ApiClient(airflow_configuration) as api_client:
        api_instance = airflow_client.client.DagRunApi(api_client)
        body = airflow_client.client.TriggerDAGRunPostBody(
            conf = parameters
        ) 
        
        try:
            triggered_dag = api_instance.trigger_dag_run(
                dag_id = dag_id, 
                trigger_dag_run_post_body = body
            )
            
            formated_dag = {
                'id': triggered_dag.dag_run_id,
                'parameters': triggered_dag.conf,
                'state': triggered_dag.state.value,
                'queue': triggered_dag.queued_at.strftime(f'%f-%S-%M-%H-%d-%m-%Y'),
                'after': triggered_dag.run_after.strftime(f'%f-%S-%M-%H-%d-%m-%Y')
            }

            return formated_dag
        except Exception as e:
            return {}
```
We can list the available dags with:
```
dag_list = airflow_get_dags(
    airflow_configuration = airflow_config,
    limit = 100, 
    tags = ['integration'],
    order_by = 'dag_id'
)
```
```
print(dag_list)
```
{'tutorial-dependency': {'tags': ['integration']}, 'tutorial-hello-world': {'tags': ['integration']}, 'tutorial-interaction': {'tags': ['integration']}, 'tutorial-main': {'tags': ['integration']}, 'tutorial-sub-1': {'tags': ['integration']}, 'tutorial-sub-2': {'tags': ['integration']}, 'tutorial-sub-3': {'tags': ['integration']}}<br></br>
We can get the list of runs and tasks with:
```
dag_runs = airflow_get_runs(
    airflow_configuration = airflow_config,
    dag_id = 'tutorial-interaction',
    limit = 100,
    order_by = 'id'
)
```
```
print(dag_runs)
```
[{'id': 'manual__2025-09-24T12:04:36.778213+00:00', 'type': 'manual', 'state': 'success', 'start': '027288-37-04-12-24-09-2025', 'end': '853600-41-04-12-24-09-2025', 'time': 4.826312, 'trigger': 'rest_api'}]
```
dag_tasks = airflow_get_tasks(
    airflow_configuration = airflow_config,
    dag_id = 'tutorial-interaction',
    order_by = 'task_id'
)
```
```
print(dag_tasks)
```
[{'id': 'sftp_interaction', 'name': 'sftp_interaction', 'parameters': {'connection': 'cloud-cpouta-1', 'remote-path': '/home/ubuntu'}, 'downstream': []}, {'id': 'ssh_interaction', 'name': 'ssh_interaction', 'parameters': {'connection': 'cloud-cpouta-1', 'remote-path': '/home/ubuntu'}, 'downstream': ['sftp_interaction']}]<br></br>
If we want the logs of a specific run, we can do it with:
```
dag_task_logs = airflow_dag_logs(
    airflow_configuration = airflow_config,
    dag_id = 'tutorial-interaction',
    dag_run_id = 'manual__2025-09-24T12:04:36.778213+00:00',
    task_id = 'sftp_interaction',
    try_number = 1,
    filtered_words = []
)
```
```
print(dag_task_logs)
```
['::group::Log message source details', '::endgroup::', 'DAG bundles loaded: dags-folder, example_dags', 'Filling up the DagBag from /opt/airflow/dags/interaction_dag.py', 'Task instance is in running state', ' Previous state of the Task instance: TaskInstanceState.QUEUED', '/home/airflow/.local/lib/python3.12/site-packages/airflow/models/connection.py:471: DeprecationWarning: Using Connection.get_connection_from_secrets from `airflow.models` is deprecated.Please use `from airflow.sdk import Connection` instead\n  warnings.warn(\n', 'Current task name:sftp_interaction', 'Dag name:tutorial-interaction', "Connection Retrieved 'cloud-cpouta-1'", "No Host Key Verification. This won't protect against Man-In-The-Middle attacks", 'Connected (version 2.0, client OpenSSH_8.9p1)', 'Authentication (publickey) successful!', '[chan 0] Opened sftp connection (server version 3)', '[chan 0] sftp session closed.', 'Done. Returned value was: None', "['.bash_history', '.bash_logout', '.bashrc', '.cache', '.config', '.kube', '.local', '.profile', '.ssh', '.sudo_as_admin_successful', 'cloud-hpc-oss-mlops-platform', 'cuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb', 'istio-ingressgateway.yaml', 'istiod.yaml', 'multi-cloud-hpc-oss-mlops-platform', 'operator-test.yaml']", 'Task instance in success state', ' Previous state of the Task instance: TaskInstanceState.RUNNING', 'Task operator:<Task(_PythonDecoratedOperator): sftp_interaction>']</br></br>
We can trigger DAGs with the following:
```
triggered_dag = airflow_trigger_dag(
    airflow_configuration = airflow_config,
    dag_id = 'tutorial-main',
    parameters = {
        "token": "test",
        "user": "user@example.com",
        "message": "hello",
        "target": "airflow"
    }
)
```
```
print(triggered_dag)
```
{'id': 'manual__2025-09-24T12:35:21.454610+00:00_SzSnzKfd', 'parameters': {'token': 'test', 'user': 'user@example.com', 'message': 'hello', 'target': 'airflow'}, 'state': 'queued', 'queue': '470331-21-35-12-24-09-2025', 'after': '454610-21-35-12-24-09-2025'}<br></br>
To finalize our utilization of Airflow, we will wrap up this part by using these functions in the submitter celery backend to request DAGs. Let's first send a request via frontend by uncommenting the following block at applications/celery/tasks/interactive/requests.py:
```
if payload['mode'] == 'task':
    airflow_configuration = airflow_setup_configuration()
    return_dict = airflow_trigger_dag(
        airflow_configuration = airflow_configuration,
        dag_id = 'tutorial-dependency',
        parameters = payload['input']
    )
```
We now only need to activate redis, frontend, backend and monitor to test the interaction using the following functions:
```
import json
import requests

def get_submitter_route(
    address: str,
    port: str,
    route_type: str,
    route_name: str,
    path_replacers: any,
    path_names: any
): 
    url_prefix = 'http://' + address + ':' + port

    routes = {
        'setup': 'POST:/setup/config',
        'task': 'GET:/interaction/task/type/identity',
        'orch': 'POST:/interaction/orch/type/key',
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

def request_submitter_route(
    address: str,
    port: str,
    route_type: str,
    route_name: str,
    path_replacers: any,
    path_names: any,
    route_input: any,
    timeout: any
) -> any:
    url_type, full_url = get_submitter_route(
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
Now we can send the following request
```
route_code, route_text = request_submitter_route(
    address = '127.0.0.1',
    port = '7500',
    route_type = '',
    route_name = 'task',
    path_replacers = {
        'type': 'run',
        'identity': 'tasks.submitter-requests'
    },
    path_names = [],
    route_input = {
        'payload': {
            'mode': 'task',
            'input': {
                'token': 'test',
                'user': 'user@example.com',
                'message': 'hello',
                'target': 'airflow'
            }
        }
    },
    timeout = 240
)
print(route_code)
print(route_text)
```
Used route: GET:/interaction/task/run/tasks.submitter-requests
200
{'output': '16d0f831-9a45-4878-95f3-fb2ef4b369e1'}</br></br>
We can use the given task identity to get the airflow response:
```
route_code, route_text = request_submitter_route(
    address = '127.0.0.1',
    port = '7500',
    route_type = '',
    route_name = 'task',
    path_replacers = {
        'type': 'get',
        'identity': '16d0f831-9a45-4878-95f3-fb2ef4b369e1'
    },
    path_names = [],
    route_input = {},
    timeout = 240
)
print(route_code)
print(route_text)
```
Used route: GET:/interaction/task/get/16d0f831-9a45-4878-95f3-fb2ef4b369e1
200
{'output': {'status': 'SUCCESS', 'result': {'id': 'manual__2025-09-25T06:49:49.127575+00:00_gnYDCZTl', 'parameters': {'token': 'test', 'user': 'user@example.com', 'message': 'hello', 'target': 'airflow'}, 'state': 'queued', 'queue': '153843-49-49-06-25-09-2025', 'after': '127575-49-49-06-25-09-2025'}}}</br></br>
Since we know that the request was a success, we can now check the logs at http://localhost:8080/dags/tutorial-dependency. Now that we know that the backend can send requests, let's use the scheduler to automatically run a DAG. Uncomment the following block at applications/celery/tasks/scheduled/trigger.py
```
airflow_configuration = airflow_setup_configuration()
return_dict = airflow_trigger_dag(
    airflow_configuration = airflow_configuration,
    dag_id = 'tutorial-main',
    parameters = {
        'token': 'test',
        'user': 'user@example.com',
        'message': 'hello', 
        'target': 'airflow'
    }
)
```
When we relod the backend with this change, save the necessery parameters into redis and start the scheduler, the DAG should be triggered every 30 seconds. We can save the parameters with the following:
```
route_code, route_text = request_submitter_route(
    address = '127.0.0.1',
    port = '7500',
    route_type = '',
    route_name = 'setup',
    path_replacers = {},
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
Used route: POST:/setup/config
200
{'output': None}</br></br>
When we active the scheduler, we now should see multiple runs with a 30 second difference at the following DAGs
- http://localhost:8080/dags/tutorial-main
- http://localhost:8080/dags/tutorial-sub-1
- http://localhost:8080/dags/tutorial-sub-2
- http://localhost:8080/dags/tutorial-sub-3

As we have now demonstrated, we can easily intregated Airflow to handle all the necessery automation that we setup and trigger using FastAPI frontend, Celery backend, Flower monitoring and Beat scheduling. We now have the necessery basic skills for creating complex automation DAGs, but there is much more necessery setup to enable proper integration.