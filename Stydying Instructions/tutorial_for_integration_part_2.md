# Tutorial for integrated distributed computing in MLOps (2/10)

This Jupyter notebook goes through the basic ideas with practical examples for integrating
distributed computing systems for MLOps systems.

## Index

[FastAPI](##FastAPI)


## FastAPI

### Useful links

[Python Package Index for FastAPI](https://pypi.org/project/fastapi/)
<br>
[Python Package Index for Uvicorn](https://pypi.org/project/uvicorn/)
<br>
[Tutorial for FastAPI](https://fastapi.tiangolo.com/tutorial/)
<br>
[FastAPI - Running a server manually](https://fastapi.tiangolo.com/deployment/manually/#install-the-server-program)
<br>
[Running Uvicorn](https://uvicorn.dev/#uvicornrun)

### Info

To provide an interaction interface for both docker compose Submitter and Kubernetes Forwarder
that can be used both by humans and machines, we will create FastAPI frontend run in a
Uvicorn server. Without going into details, FastAPI provides a easy way to create APIs for
interactions, while Uvicorn provides an programmatic way to run the server in a container.
You can find a reduced variant of Submitter frontend in the applications/submitter/fastapi
folder that you can run with:

```
cd applications/submitter/fastapi
python3 -m venv front_venv
source front_venv/bin/activate
pip install -r packages.txt
python3 run_fastapi.py
```

You can now check the available routes at http://localhost:7500/docs and check the
produced logs via the docs by doing the following:

   1. Click /interaction/arti/{type}/{target} window
   2. Click try it out
   3. Write logs in type
   4. Write frontend in target
   5. Click execute
   6. Go down to see the logs

The shown logs are gathered at fastapi/logs/frontend.log. This API interaction used
the following things:

  - function logger_get_logs() found at fastapi/functions/platforms/logger/use
  - route artifact_interaction() found at fastapi/routes/interaction
  - inclusion via include_route() found at fastapi/setup_fastapi
  - app initilization via FastAPI() found at fastapi/setup_fastapi
  - app setup via setup_fastapi_app() found at fastapi/run_fastapi
  - server run via run() found at fastapi/run_fastapi

As you have seen, the whole application is separated into smaller files in the following way:

  - run_fastapi
  - setup_fastapi
  - logs
  - routes
    - general
    - interaction
    - setup
  - functions
    - models
    - platforms
      - celery
        - setup
        - use
      - logger
        - setup
        - use
      - redis
        - setup
        - use
       
The reason for this is to enable easier understandablity and development by dividing the
necessery code groups into their smallest practical form. This does unfortunately make
the file structure more complex, which can increase the time needed to conceptualize
what is happening in the application, but in return in divides the application
responsiblity into clear areas that in this case are import functions, use functions,
routes and setup. This results in depth abstraction development, where you only need to 
focus on routes provided that functions use import functions properly. For this reason we
can focus on actually understanding the important parts of FastAPI that are:

  - State
  - Router
  - Route
  - Path
  - Path Parameters
  - Request body

When you create a instance with FastAPI(), you can start to store different objects and 
data by using:

```
from fastapi import FastAPI
fastapi_app = FastAPI()
fastapi_app.state.log_path = 'str'
fastapi_app.state.logger = object
```

These are treated as application level context that can be used later. You can include 
in the main FastAPI instance smaller instances called APIRouters that consists of
multiple routes via the following pattern:

```
from fastapi import APIRouter
interaction_fastapi = APIRouter(prefix = '/interaction')
fastapi_app.include_router(interaction_fastapi)
```

Here the prefix creates group paths, where all the paths inside the router get 
the '/interaction' prefix at the start. The routes in them are created with:

```
from fastapi import Request
@interaction_fastapi.get("/task/{type}/{identity}")
async def task_interaction(
    request: Request,
    type: str,
    identity: str,
    input: Dict = {}
):
    request.app.state.logger.info('Task interaction')
    return_dict = {
        'output': None
    }
    print('Hello')
    return return_dict
```

Here a route consists of:

  - Router -> @interaction_fastapi
  - HTTP request -> get()
  - Path -> '/task/'
  - Path parameters -> '/{type}/{identity}'
  - Application context -> request: Request
  - Async -> async def
  - Function parameters -> type: str, identity: str
  - Request body -> input: Dict = {}
  - Logger use -> request.app.state.logger.info('Task interaction')
  - Default values -> return_dict = {'output': None}
  - Execution code -> print('Hello')
  - Value return -> return return_dict

Be aware that FastAPI validates data request body, which enables us to use Pydantic 
to specify what we want from a API request in the following way:

```
from functions.models.parameters import Parameters
@setup_fastapi.post("/config")
async def configuration_setup(
    request: Request,
    parameters: Parameters
): 
```

This isn't the only way to structure a FastAPI routes, but this is one of the most 
suitable one for API that requires concurrent requests to the backend. When all the
routers are given to the instance, it is then given to uvicorn in the following way:

```
fastapi = setup_fastapi_app()
uvicorn.run(
    app = fastapi, 
    host = '0.0.0.0', 
    port = 7500
)
```

with address and port configuration for the server to run at http://0.0.0.0:7500

## Docker Compose

### Useful material

[Redis tags from Docker Hub](https://hub.docker.com/_/redis/tags)
<br>
[Docker Compose Quickstart](https://docs.docker.com/compose/gettingstarted/)
<br>
[How Docker Compose works](https://docs.docker.com/compose/intro/compose-application-model/)
<br>
[Running Redis Open Source on Docker](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/)

### Info

Provided that your docker is setup, you can start running different docker containers
with provided YAML files. Please replace (path_here) with absolute path to applications folder:

In [ ]:
```
%%writefile (path_here)/submitter/redis.yaml
services:
  tutorial-redis:
    image: redis:8.2.1/
    restart: no
    ports:
      - '127.0.0.1:6379:6379'
```

by running the following command:

```
docker compose -f (name).yaml up
```

You can later stop it with:

```
CTRL + C
```

The most important things in our case in using provided single containers are the 
provided image and ports. In this case we used the newest redis image found in DockerHub
and ordered Docker to expose a connection to the container at 127.0.0.1:6379. Most open-
source software has official documentation and even container images that we can use to 
create suitable YAML files.

## Redis

### Useful material

[redis-py guide](https://redis.io/docs/latest/develop/clients/redis-py/)
<br>
[Python Redis: A beginner's guide](https://www.datacamp.com/tutorial/python-redis-beginner-guide?dc_referrer=https%3A%2F%2Fnotebooks.githubusercontent.com%2F)

To provide a cache for application storage, we will use Redis to store pickled data via
frontend that backend can use later as parameters. Run the redis container with:

```
cd applications/submitter
docker compose -f redis.yaml up
```

You can now create a client with the following:

In [2]:
```
import redis

redis_endpoint = '127.0.0.1'
redis_port = '6379'
redis_db = '0'
redis_client = redis.Redis(
    host = redis_endpoint,
    port = int(redis_port),
    db = redis_db
)
```

We can now store the following dictionary:

In [3]:
```
storage_parameters = {
    'bucket-prefix': 'multi-cloud-hpc',
    'ice-id': 's0-c0-u1',
    'user': 'user@example.com',
    'used-client': 'swift',
    'pre-auth-url': 'empty',
    'pre-auth-token': 'empty',
    'user-domain-name': 'empty',
    'project-domain-name': 'empty',
    'project-name': 'empty',
    'auth-version': 'empty'
}
```

We can use:

```
hset()
hgetall()
```

to get following functions:

In [7]:
```
def redis_store_regular_dict(
    redis_client: any,
    dict_name: str,
    regular_dict: any
) -> bool:
    try:
        result = redis_client.hset(
            dict_name, 
            mapping = regular_dict
        )
        return result
    except Exception as e:
        return False

def redis_get_regular_dict(
    redis_client: any,
    dict_name: str,
) -> any:
    try:
        byte_dict = redis_client.hgetall(dict_name)
        regular_dict = {k.decode('utf-8'): v.decode('utf-8') for k, v in byte_dict.items()}
        return regular_dict
    except Exception as e:
        return None
```

In [5]:
```
response = redis_store_regular_dict(
    redis_client = redis_client,
    dict_name = 'tutorial-parameters',
    regular_dict = storage_parameters
)
print(response)
```

Out [5]:
10

In [8]:
```
stored_dict = redis_get_regular_dict(
    redis_client = redis_client,
    dict_name = 'tutorial-parameters'
)
print(stored_dict)
```

{'bucket-prefix': 'multi-cloud-hpc', 'ice-id': 's0-c0-u1', 'user': 'user@example.com', 
'used-client': 'swift', 'pre-auth-url': 'empty', 'pre-auth-token': 'empty', 
'user-domain-name': 'empty', 'project-domain-name': 'empty', 'project-name': 'empty',
'auth-version': 'empty'}

If for some reason your application has small nested dicts and you want a quick way
(not the best way) of storing them in redis, use pickle. For example:

In [12]

```
nested_example_dict = {
    'item-1': {
        'nest-1': {
            'value': 'empty'
        },
        'nest-2': {
            'value': 'empty'
        }
    },
    'item-2': {
        'nest-1': {
            'value': 'empty'
        },
        'nest-2': {
            'value': 'empty'
        }
    }
}
```

We can implement pickle storage with:

```
set()
get()
```

by chancing the previous functions:

In [10]:
```
import pickle

def redis_store_nested_dict(
    redis_client: any,
    dict_name: str,
    nested_dict: any
) -> bool:
    try:
        formatted_dict = pickle.dumps(nested_dict)
        result = redis_client.set(dict_name, formatted_dict)
        return result
    except Exception as e:
        return False

def redis_get_nested_dict(
    redis_client: any,
    dict_name: str
) -> any:
    try:
        pickled_dict = redis_client.get(dict_name)
        unformatted_dict = pickle.loads(pickled_dict)    
        return unformatted_dict
    except Exception as e:
        return None
```

In [13]:

```
response = redis_store_nested_dict(
    redis_client = redis_client,
    dict_name = 'tutorial-nested',
    nested_dict = nested_example_dict
)
print(response)
```

True

In [14]:
```
stored_nested_dict = redis_get_nested_dict(
    redis_client = redis_client,
    dict_name = 'tutorial-nested',
) 
print(stored_nested_dict)
```

{'item-1': {'nest-1': {'value': 'empty'}, 'nest-2': {'value': 'empty'}}, 
'item-2': {'nest-1': {'value': 'empty'}, 'nest-2': {'value': 'empty'}}}

One can find further discussion on how to implement a better system:

[Store Nested data structures and Python objects in Redis Cache](https://medium.com/@vickypalaniappan12/store-nested-data-structures-and-python-objects-in-redis-cache-814f03436d89)
[Is it possible to store nested dict in Redis as hash structure](https://stackoverflow.com/questions/70203777/is-it-possible-to-store-nested-dict-in-redis-as-hash-structure)

Besides being a storage Redis is used for handling concurrent locking to prevent 
race conditions. We can use:

```
lock()
acquire(blocking = True)
exists()
locked()
release()
```

to create the following function:

In [15]:
```
def redis_lock_interaction(
    redis_client: any,
    redis_lock: any,
    mode: str,
    lock_name: str,
    timeout: int
) -> any:
    first_output = False
    second_output = None
    try:
        if mode == 'check':
            first_output = bool(redis_client.exists(lock_name))
        if mode == 'release':
            if redis_lock.locked():
                redis_lock.release()
                first_output = True
        if mode == 'get':
            lock_instance = redis_client.lock(
                lock_name,
                timeout = timeout
            )

            lock_aquired = lock_instance.acquire(blocking = True)
            
            if lock_aquired:
                first_output = True
                second_output = lock_instance
        return first_output, second_output
    except Exception as e:
        return first_output, second_output
```

This allows us to create the following sequence for each concurrent task:

In [16]:
```
lock_name = 'lock-tutorial'
redis_lock = None
lock_active, empty_1 = redis_lock_interaction(
    redis_client = redis_client,
    redis_lock = None,
    mode = 'check',
    lock_name = lock_name,
    timeout = 200
)

if not lock_active:
    print('No lock')
    lock_created, redis_lock = redis_lock_interaction(
        redis_client = redis_client,
        redis_lock = None,
        mode = 'get',
        lock_name = lock_name,
        timeout = 200
    )
    if lock_created:
        print('Lock created')
        
        print('Code block')
    
        lock_released, empty_2 = redis_lock_interaction(
            redis_client = None,
            redis_lock = redis_lock,
            mode = 'release',
            lock_name = None,
            timeout = None
        )
        print('Lock released:' + str(lock_released))
```

No lock
Lock created
Code block
Lock released:True
