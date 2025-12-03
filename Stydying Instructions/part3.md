# Tutorial for integrated distributed computing in MLOps (3/10)
This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.
## Celery
Useful materials
- https://pypi.org/project/celery/
- https://pypi.org/project/requests/
- https://docs.celeryq.dev/en/latest/userguide/index.html
- https://docs.celeryq.dev/en/3.1/getting-started/first-steps-with-celery.html

To automate interactions between separate containers and systems, we choose Celery with Redis as the message broker to handle automation in Docker compose Submitter and Kubernetes Forwarder. Celery provides a distributed task queue that can execute requested or scheduled python code in pararell, while Redis handles double duty as the cache storage and message broker. You can find the reduced Submitter backend variant at applications/submitter/celery that you can run with
```
cd applications/celery
python3 -m venv back_venv
source back_venv/bin/activate
pip install -r packages.txt
python3 run_celery.py
```
You can now send tasks to be run using the FastAPI frontend by going to http://localhost:7500/docs and check the produced logs via the docs by doing the following:
- click /interaction/arti/{type}/{target} window
- click try it out
- Write logs in type
- Write backend in target
- Click execute
- Go down to see the logs

The shown logs are gathered at celery/logs/backend.log. This task interaction used the following things:
- Function celery_get_logs() found at celery/functions/platforms/celery/use.py
- Task defined submitter_requests() found at celery/tasks/interactive/requests.py
- Registration with task() found at celery/setup_celery.py
- Defined instance with celery_setup_instance() found at celery/setup_celery.pu
- Celery connection with Celery() found at celery/functions/platforms/celery/user.pu
- Celery runner with worker_main() found at celery/run_celery.py

As you have seen, the whole application is separated into smaller files in the following way:
- run_celery.py
- setup_celery.py
- logs
- functions
    - general
    - platforms
        - airflow
            - setup
            - use
        - celery
            - setup
            - use
        - flower
            - use
            - utility
        - redis
            - setup
            - use
- tasks
    - interactive
        - requests
    - scheduled
        - trigger

The reason is the same as for FastAPI aka depth abstraction development that enables focus on import functions, use functions and tasks. For this reason we can focus on the important parts of Celery that are:
- Module
- Tasks
- Signatures

When you create a instance with Celery(), you need to connect to Redis in the following way:
```
redis_endpoint = '127.0.0.1'
redis_port = '6379'
redis_db = '0'

name = 'tasks'
redis_connection = 'redis://' + redis_endpoint + ':' + str(redis_port) + '/' + str(redis_db)

celery_app = Celery(
    main = name,
    broker = redis_connection,
    backend = redis_connection
)
```
Here we establish to celery that we want it to save all tasks into a module called tasks, which enables FastAPI to send requests using the correct signatures. This enables it to be treated as a reusble context that can be used to create a main instance and sub instances with tasks
```
tasks_celery = celery_setup_instance()

@tasks_celery.task( 
    bind = False, 
    max_retries = 0,
    soft_time_limit = 120,
    time_limit = 240,
    rate_limit = '2/m',
    name = 'tasks.submitter-requests'
) 
def submitter_requests(  
    payload: any
):
    return_dict = {}
    print('task')
    return return_dict
```
that consist of
- Instance -> @tasks_celery
- Task configuration -> task()
- Task signature -> name = 'tasks.submitter-requests'
- Task functions -> def submitter_requests()
- Task input -> payload: any
- Default values -> return_dict = {}
- Executed code -> print('task')
- Value return -> return return_dict

These are added into the main instance with
```
celery_app = celery_setup_instance()
celery_app.task(submitter_requests)
```
that we then run with
```
celery_app.worker_main(
    argv = [
        'worker', 
        used_concurrency, 
        used_loglevel, 
        used_logpath
    ]
)
```
where the argv for them following start up command
```
worker --concurrency=8 --loglevel=info --logfile=logs/backend.log
```
## Flower
Useful materials:
- https://pypi.org/project/flower/
- https://github.com/mher/flower

To enable easier monitoring and debugging of Celery tasks, we can run Flower for real time monitoring. You can find the regular Flower variant used in both Forwarder and Submitter in the applications/flower that you can run with:
```
cd applications/submitter/flower
python3 -m venv monitor_venv
source monitor_venv/bin/activate
pip install -r packages.txt
python3 run_flower.py
```
This opens a website at http://127.0.0.1:7501 that asks for the credentials flower123-flower456. In the view the relevant pages for us are
- workers -> shows celery workers
- tasks -> shows detected tasks

If you go the tasks, you can see accurately what tasks were run, their arguments and results, which can be used to debug what happened. This was enabled again by the following functions:
- function celery_setup_instance() in flower/run_flower.py
- Celery instance with Celery() in flower/setup_flower.py
- Flower start with start() in flower/run_flower.py

These enable the whole thing to be run with
```
flower = celery_setup_instance()
used_address = '--address=' + endpoint
used_port = '--port=' + port
basic_authentication = '--basic_auth=' + os.environ.get('FLOWER_USERNAME') + ':' + os.environ.get('FLOWER_PASSWORD')
flower.start(argv = ['flower', used_address, used_port, basic_authentication])
```
where again the argv gets the following command
```
flower --address=0.0.0.0 --port=7501 --basic_auth=flower123:flower456
```
Besides the UI Flower also provides an API that can be used to get the logged tasks with HTTP requests:
```
import requests
from requests.auth import HTTPBasicAuth

def flower_get_tasks(
    flower_address: str,
    flower_port: str,
    flower_username: str,
    flower_password: str
) -> any:
    flower_url = 'http://' + flower_address + ':' + flower_port + '/api/tasks'
    response = requests.get(
        url = flower_url, 
        auth = HTTPBasicAuth(
            username = flower_username, 
            password = flower_password
        )
    )
    tasks = {}
    if response.status_code == 200:
        tasks = response.json()
    return tasks

def flower_format_tasks(
    tasks: any
) -> any:
    relevant_keys = [
        'worker',
        'children',
        'state',
        'received',
        'started',
        'succeeded',
        'failed',
        'result',
        'timestamp',
        'runtime',
    ]
    # This doesn't gurantee 
    # a time stamp order
    # and it doesn't 
    # specify the children names
    formatted_flower_tasks = {}
    sorted_tasks = sorted(
        tasks.values(), 
        key=lambda x: float(x['received']) if not x['received'] is None else float('-inf')
    )
    for task_info in sorted_tasks:
        if not task_info['name'] is None:
            task_id = task_info['uuid']
            task_name = task_info['name'].split('.')[-1].replace('_','-')
            used_key = ''
            if not task_name in formatted_flower_tasks:
                used_key = '1/' + task_id
                formatted_flower_tasks[task_name] = {
                    used_key: {}
                }
            else:
                new_key = len(formatted_flower_tasks[task_name]) + 1
                new_key = str(new_key)
                used_key = new_key + '/' + task_id
                formatted_flower_tasks[task_name][used_key] = {}
            
            for task_key, task_value in task_info.items():
                if task_key in relevant_keys:
                    formatted_flower_tasks[task_name][used_key][task_key] = task_value  
    return formatted_flower_tasks   
```
```
flower_tasks = flower_get_tasks(
    flower_address = '127.0.0.1',
    flower_port = '7501',
    flower_username = 'flower123',
    flower_password = 'flower456'
)
print(flower_tasks)
```
```
formatted_flower_tasks = flower_format_tasks(
    tasks = flower_tasks
)
print(formatted_flower_tasks)
```
This enables collecting tasks logs into a permantent storage for later debugging.
## Celery beat
Useful materials:
- https://docs.celeryq.dev/en/latest/userguide/periodic-tasks.html
- https://docs.celeryq.dev/en/stable/reference/celery.beat.html

To enable a scheduling of Celery tasks, we will use Celery beat as the decoupled scheduler. You can find the variant used by Forwarder and Submitter in the applications/beat that you can run with:
```
cd applications/submitter/celery
python3 -m venv scheduler_venv
source scheduler_venv/bin/activate
pip install -r packages.txt
python3 run_beat.py
```
This causes the scheduler to run defined tasks at seconds. The relevant things in the files are
- Given SCHEDULER_TIMES at beat/run_beat.py
- Defined schedule at beat/setup_beat.py

The SCHEDULER_TIMES defined as a enviroment variable gives the relative seconds for running the same task with '|' used to separate tasks in order.
```
from celery.schedules import timedelta
task_times = os.environ.get('SCHEDULER_TIMES').split('|')
schedule = []
expires_seconds = []
for time in task_times:
    schedule.append(timedelta(seconds = int(time)))
    expires_seconds.append(int(time))
```
This means '30|40|50' caused the following:
- First task runs every 30 sec
- Second task runs every 40 sec
- Third task runs every 50 sec
```
beat_app.conf.beat_schedule = {
    'submitter-trigger': {
        'task': 'tasks.submitter-trigger',
        'schedule': schedule[0],
        'kwargs': {},
        'relative': True,
        'options': {
            'expire_seconds': expires_seconds[0]
        }
    }
}
```
that consists of
- Task name in the scheduler -> submitter-trigger
- Task signature in Celery -> task: tasks.submitter-trigger
- Timedelta time -> schedule: schedule
- Task arguments -> kwargs: {}
- Scheduling relative to current time -> relative: True
- Task experation -> expire_seconds: expires_seconds

These are again defined into the main instance we can run with the following:
```
beat = setup_beat_app()
beat.Beat(loglevel='warning').run()
```
Be aware that the scheduling stops the moment Beat is stopped, which enables us to always remove scheduling for debugging the application without affecting the backend.
## Microservices Architectures
Useful materials:
- https://www.ibm.com/think/topics/monolithic-architecture
- https://medium.com/@belemgnegreetienne/the-ultimate-guide-to-docker-benefits-architecture-and-practical-steps-02d0f02e7eee
- https://stackoverflow.com/questions/30534939/should-i-use-separate-docker-containers-for-my-web-app
- https://www.devzero.io/blog/docker-microservices
- https://medium.com/@shrinit.poojary12/microservices-with-fastapi-and-docker-a-step-by-step-guide-63edaadb65b2
- https://en.wikipedia.org/wiki/Design_science_(methodology)

What we see in the FastAPI Frontend, Celery backend, Flower monitoring, Beat scheduling and Redis caching is a Python focused microservice architecture created for Docker Compose and Kubernetes enviroments. This architecture is the default for containerized applications, because their major benefit besides their portability is their ability to separate responsiblity to ensure scalability, increase fault tolerance and ensure flexible development in return for complexity. With smarter coding the amount of frontend, backend, monitoring, scheduling and caching instances could be increased as the number of users increase without having to worry too much about users not getting a container in the case of failures, while enabling the replacement of broken container with different deployment strategies.<br></br>
For our use case microservice architecture enables the synergic utilization of design science research methodology in architecture theory and practical implementation, since instead of nailing down what software we exactly should use, we can simply abstract these into pipeline functions that generilize to most common MLOps workflows. When the theoretical pipelines look good enough, we can then simply search for software that fits to the functions and create software to fill the critical gaps between different software. When the minimum viable product of this implementation is working, we only need to evaluated it against our own objectives, constraints and metrics to gain insights for the next development iteration of the architecture and implementation. For these reasons, we need selected and created software to be mature, abstracted and interopeable to reduce the required development effort of each iteraction to produce increasingly better solutions for integrated distributed computing.