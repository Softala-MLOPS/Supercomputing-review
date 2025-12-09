# Tutorial for integrated distributed computing in MLOps (7/10)
This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.

## Allas with Swift
Useful material:

- https://docs.csc.fi/data/Allas/
- https://docs.csc.fi/data/Allas/accessing_allas/
- https://docs.csc.fi/data/Allas/using_allas/python_swift/
- https://pypi.org/project/python-keystoneclient/
- https://pypi.org/project/keystoneauth1/
- https://pypi.org/project/python-swiftclient/
- https://pypi.org/project/python-decouple/
- https://docs.openstack.org/keystoneauth/latest/api/keystoneauth1.session.html
- https://docs.openstack.org/python-swiftclient/pike/client-api.html
- https://docs.openstack.org/keystoneauth/latest/api/keystoneauth1.identity.html
- https://docs.openstack.org/python-swiftclient/newton/swiftclient.html
- https://pypi.org/project/pandas/
- https://pypi.org/project/pyarrow/

To provide a global storage that enables data transfers between separated systems and a more permanent storage with vendor handeled fault tolerance, we will now go through how to use CSC Allas object storage through Python Swiftclient. Assuming that you have a MyCSC project with access to Allas, we will only need to provide our credentials to get a access token that we can provide as a input for separated systems. We can do that with the following function:
```
from decouple import Config,RepositoryEnv
from keystoneauth1 import loading, session

def access_swift_parameters(
    env_path: str,
    auth_url: str,
    pre_auth_url: str,
    auth_version: str
):
    env_config = Config(RepositoryEnv(env_path))
    swift_auth_url = auth_url
    swift_user = env_config.get('CSC_USERNAME')
    swift_key = env_config.get('CSC_PASSWORD')
    swift_project_name = env_config.get('CSC_PROJECT_NAME')
    swift_user_domain_name = env_config.get('CSC_USER_DOMAIN_NAME')
    swift_project_domain_name = env_config.get('CSC_USER_DOMAIN_NAME')

    loader = loading.get_plugin_loader('password')
    auth = loader.load_from_options(
        auth_url = swift_auth_url,
        username = swift_user,
        password = swift_key,
        project_name = swift_project_name,
        user_domain_name = swift_user_domain_name,
        project_domain_name = swift_project_domain_name
    )

    keystone_session = session.Session(
        auth = auth
    )
    swift_token = keystone_session.get_token()
    swift_pre_auth_url = pre_auth_url
    swift_auth_version = auth_version

    swift_parameters = {
        'pre-auth-url': str(swift_pre_auth_url),
        'pre-auth-token': str(swift_token),
        'user-domain-name': str(swift_user_domain_name),
        'project-domain-name': str(swift_project_domain_name),
        'project-name': str(swift_project_name),
        'auth-version': str(swift_auth_version)
    }

    return swift_parameters
```
To use it we only need to create .env file at .ssh with the following:
```
# Go to folder
cd .ssh
nano .env

# Add with filled values
CSC_USERNAME = "(Fill)"
CSC_PASSWORD = "(Fill)"
CSC_USER_DOMAIN_NAME = "(Fill)"
CSC_PROJECT_NAME = "(Fill)"

CTRL + X
Y
```
Here user domain name is usually 'Default', while the project is shown in MyCSC in the form project_(id). With these set we can get the storage parameters by giving the following values to the function:
- env_path -> Absolute path to the .env
- auth_url -> CPouta API identity service point
- pre_auth_url -> CPouta API Object Store service endpoint
- auth_version -> Client API version
```
workflow_swift_parameters = access_swift_parameters(
    env_path = '/home/()/.ssh/.env',
    auth_url = 'https://pouta.csc.fi:5001/v3',
    pre_auth_url = 'https://a3s.fi:443/swift/v1',
    auth_version = '3'
)
```
These parameters can be used by any system with the swiftclient package installed. Be aware that the token has a time span of around 2-6 hours, which means longer operations require token renewal using the token. The client functions is:
```
import swiftclient as sc

def swift_client_check(
    storage_client: any
) -> any:
    return isinstance(storage_client, sc.Connection)

def swift_setup_client(
    swift_parameters: any
) -> any:
    swift_client = sc.Connection(
        preauthurl = swift_parameters['pre-auth-url'],
        preauthtoken = swift_parameters['pre-auth-token'],
        os_options = {
            'user_domain_name': swift_parameters['user-domain-name'],
            'project_domain_name': swift_parameters['project-domain-name'],
            'project_name': swift_parameters['project-name']
        },
        auth_version = swift_parameters['auth-version']
    )
    return swift_client
```
We simply need to provide the correct parameters to create the client:
```
workflow_swift_client = swift_setup_client(
    swift_parameters = workflow_swift_parameters
)
```
Now, we can manipulate the project Allas containers with suitable functions. Here are the utility functions:
```
def swift_set_encoded_metadata(
    metadata: any
) -> any:
    encoded_metadata = {}
    key_initial = 'x-object-meta'
    for key, value in metadata.items():
        encoded_key = key_initial + '-' + key
        if isinstance(value, list):
            encoded_metadata[encoded_key] = 'list=' + ','.join(map(str, value))
            continue
        encoded_metadata[encoded_key] = str(value)
    return encoded_metadata

def swift_get_general_metadata(
    metadata: any
) -> any:
    general_metadata = {}
    key_initial = 'x-object-meta'
    for key, value in metadata.items():
        if not key_initial == key[:len(key_initial)]:
            general_metadata[key] = value
    return general_metadata

def swift_get_decoded_metadata(
    metadata: any
) -> any:
    decoded_metadata = {}
    key_initial = 'x-object-meta'
    for key, value in metadata.items():
        if key_initial == key[:len(key_initial)]:
            decoded_key = key[len(key_initial) + 1:]
            if 'list=' in value:
                string_integers = value.split('=')[1]
                values = string_integers.split(',')
                if len(values) == 1 and values[0] == '':
                    decoded_metadata[decoded_key] = []
                else:
                    try:
                        decoded_metadata[decoded_key] = list(map(int, values))
                    except:
                        decoded_metadata[decoded_key] = list(map(str, values))
                continue
            if value.isnumeric():
                decoded_metadata[decoded_key] = int(value)
                continue
            decoded_metadata[decoded_key] = value
    return decoded_metadata

def swift_get_bucket_metadata(
    metadata: any
):
    relevant_values = {
        'x-container-object-count': 'object-count',
        'x-container-bytes-used-actual': 'used-bytes',
        'last-modified': 'date',
        'content-type': 'type'
    }
    formatted_metadata = {}
    for key,value in metadata.items():
        if key in relevant_values:
            formatted_key = relevant_values[key]
            formatted_metadata[formatted_key] = value
    return formatted_metadata

def swift_format_bucket_objects(
    objects: any
) -> any:
    formatted_objects = {}
    for bucket_object in objects:
        formatted_object_metadata = {
            'hash': 'id',
            'bytes': 'used-bytes',
            'last_modified': 'date'
        }
        object_key = None
        object_metadata = {}
        for key, value in bucket_object.items():
            if key == 'name':
                object_key = value
            if key in formatted_object_metadata:
                formatted_key = formatted_object_metadata[key]
                object_metadata[formatted_key] = value
        formatted_objects[object_key] = object_metadata
    return formatted_objects
```
Here are the interaction functions:
```
def swift_create_or_update_object( 
    swift_client: any,
    bucket_name: str, 
    object_path: str, 
    object_data: any,
    object_metadata: any
) -> any:
    bucket_info = {}
    
    try:
        given_info = swift_client.get_container(
                container = bucket_name
            )
        bucket_info = {
            'metadata': given_info[0],
            'objects': given_info[1]
        }
    except Exception as e:
        pass

    if len(bucket_info) == 0:
        try:
            swift_client.put_container(
                container = bucket_name
            )
        except Exception as e:
            return False
    
    object_info = {}
    try:
        object_info = swift_client.head_object(
            container = bucket_name,
            obj = object_path
        )       
    except Exception as e:
        pass
    
    if not len(object_info) == 0:
        try:
            swift_client.delete_object(
                container = bucket_name, 
                obj = object_path
            )
        except Exception as e:
            return False
        
    formatted_metadata = swift_set_encoded_metadata(
        metadata = object_metadata
    )
    
    try:
        swift_client.put_object(
            container = bucket_name,
            obj = object_path,
            contents = object_data,
            headers = formatted_metadata
        )
        return True
    except Exception as e:
        return False
    
def swift_check_object_metadata(
    swift_client: any,
    bucket_name: str, 
    object_path: str
) -> any:
    all_metadata = {}
    try:
        all_metadata = swift_client.head_object(
            container = bucket_name,
            obj = object_path
        )       
    except Exception as e:
        pass

    object_metadata = {
        'general-meta': {},
        'custom-meta': {}
    }
    if not len(all_metadata) == 0:
        general_metadata = swift_get_general_metadata(
            metadata = all_metadata
        )
        custom_metadata = swift_get_decoded_metadata(
            metadata = all_metadata
        )

        object_metadata['general-meta'] = general_metadata
        object_metadata['custom-meta'] = custom_metadata
    
    return object_metadata

def swift_get_object_content(
    swift_client: any,
    bucket_name: str,
    object_path: str
) -> any:
    try:
        response = swift_client.get_object(
            container = bucket_name,
            obj = object_path 
        )
        general_meta = swift_get_general_metadata(
            metadata = response[0]
        )
        custom_meta = swift_get_decoded_metadata(
            metadata = response[0]
        )
        object_content = {
            'data': response[1],
            'general-meta': general_meta,
            'custom-meta': custom_meta
        }
        return object_content
    except Exception as e:
        return {}

def swift_remove_object(
    swift_client: any,
    bucket_name: str,
    object_path: str   
) -> bool:
    try:
        swift_client.delete_object(
            container = bucket_name, 
            obj = object_path
        )
        return True
    except Exception as e:
        return False
    
def swift_get_bucket_info(
    swift_client: any,
    bucket_name: str   
) -> any:
    try:
        bucket_info = swift_client.get_container(
            container = bucket_name
        )
        bucket_metadata = swift_get_bucket_metadata(
            metadata = bucket_info['metadata']
        )
        bucket_objects = swift_format_bucket_objects(
            objects = bucket_info['objects']
        )
        return {'metadata': bucket_metadata, 'objects': bucket_objects} 
    except Exception as e:
        return {} 

def swift_get_container_info(
    swift_client: any
) -> any:
    try:
        account_buckets = swift_client.get_account()[1]
        container_info = {}
        for bucket in account_buckets:
            bucket_name = bucket['name']
            bucket_count = bucket['count']
            bucket_size = bucket['bytes']
            container_info[bucket_name] = {
                'amount': bucket_count,
                'size': bucket_size
            }
        return container_info 
    except Exception as e:
        return {}
```
These functions are meant to serve as abstraction functions that can be moved between remote systems. Be aware that the container is the place where all project buckets are stored, while buckets container all the objects with specific paths stored specifically into them. Since these functions handle the actual interactions, the repeating interaction details such as bucket names need to be handeled by further functions. The purpose of these functions is to point toward the correct place, prevent path mishaps such as inputing DATA//2 (these are not handeled by the UI), handle serialization and enable us to simply call the functions in the code as needed. For our use case we need to standardize the following things for the entire workflow:
- Bucket names
- Object paths
- Object metadata
- Object serialization

For our use case these will be:
- Names (identity)-(name)-(User)
    - Identity: mlch
    - Name: for-air, sub-air, pipe and exp
    - User: User@example.com
- Paths:
    - for-air
        - ARTIFACTS
            - TASKS
        - DEPLOYMENTS
            - FORWARD
                - IMPORTS
        - MONITOR
            - TIMES
            - SACCT
    - sub-air
        - ARTIFACTS
            - SACCT
            - TASKS
            - TIMES
        - MANAGMENT
    - pipe
        - ARTIFACTS
        - CODE
            - RAY
        - LOGS
        - TIMES
    - exp
        - ARTIFACTS
            - SACCT
        - TIMES
- Metadata:
    - Version: Incremented number that starts from 1
- Serialization
    - Pickle: Used for communication dicts
    - Parquet: Used for tabular system and model data
    - None: Used for ML model formats
    - Other: Used for model data unsuitavle for Picke (big data images and audio)

The reasoning for these rules is storage isolation and solidification of paths to ensure that distributed system manipulate only the storage space they are allowed to. The main idea is to keep objects read-write rights for their owner with other systems only having read rights to ensure that there is no write conflicts, which in the case of the owners can be ensured with redis locks. With these rules we get the following utility functions:
```
import re

def set_formatted_user(
    user: str   
) -> any:
    return re.sub(r'[^a-z0-9]+', '-', user)
```
Here are the inferaction functions
```
def set_bucket_name(
    bucket_target: str,
    bucket_prefix: str,
    bucket_user: str
) -> any:
    bucket_names = {
        'forwarder': bucket_prefix + '-for-air',
        'submitter': bucket_prefix + '-sub-air-' + set_formatted_user(user = bucket_user),
        'pipeline': bucket_prefix + '-pipe-' + set_formatted_user(user = bucket_user),
        'experiment': bucket_prefix + '-exp-' + set_formatted_user(user = bucket_user)
    }
    bucket_name = bucket_names[bucket_target]
    print('User object bucket: ' + str(bucket_name))
    return bucket_name

def set_object_path(
    object_name: str,
    path_replacers: any,
    path_names: any
):
    object_paths = {
        'root': 'name',
        'code': 'CODE/name',
        'data': 'DATA/name',
        'arti': 'ARTIFACTS/name',
        'mana': 'MANAGEMENT/name',
        'time': 'TIMES/name',
        'logs': 'LOGS/name'
    }

    i = 0
    path_split = object_paths[object_name].split('/')
    for name in path_split:
        if name in path_replacers:
            replacer = path_replacers[name]
            if 0 < len(replacer):
                path_split[i] = replacer
        i = i + 1
    
    if not len(path_names) == 0:
        path_split.extend(path_names)

    object_path = '/'.join(path_split)
    print('Used object path: ' + str(object_path))
    return object_path

def object_storage_interaction(
    storage_client: any,
    parameters: any,
    object_data: any,
    object_metadata: any
) -> any:
    output = None
    if swift_client_check(storage_client = storage_client):
        bucket_name = set_bucket_name(
            bucket_target = parameters['bucket-target'],
            bucket_prefix = parameters['bucket-prefix'],
            bucket_user = parameters['bucket-user']
        )
        object_path = set_object_path(
            object_name = parameters['object-name'],
            path_replacers = parameters['path-replacers'],
            path_names = parameters['path-names']
        )
        if parameters['mode'] == 'check' or parameters['mode'] == 'send':
            output = swift_check_object_metadata(
                swift_client = storage_client,
                bucket_name = bucket_name,
                object_path = object_path
            )
        if parameters['mode'] == 'send':
            perform = True
            if not len(output['general-meta']) == 0 and not parameters['overwrite']:
                perform = False
            if perform:
                output = swift_create_or_update_object(
                    swift_client = storage_client,
                    bucket_name = bucket_name,
                    object_path = object_path,
                    object_data = object_data,
                    object_metadata = object_metadata
                )
        if parameters['mode'] == 'get':
            temporary = swift_get_object_content(
                swift_client = storage_client,
                bucket_name = bucket_name,
                object_path = object_path
            )
            if 'data' in temporary:
                output = (
                    temporary['data'],
                    temporary['general-meta'], 
                    temporary['custom-meta']
                )
    return output
```
Here are the parquet serialization functions:
```
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from io import BytesIO

def pyarrow_serialize_dataframe(
    dataframe: any
) -> any:
    buffer = BytesIO()
    dataframe.to_parquet(buffer, index = True)
    buffer.seek(0)
    serialized_data = buffer.getvalue()
    return serialized_data

def pyarrow_deserialize_dataframe(
    serialized_dataframe: any
) -> any:
    deserialized_data = BytesIO(serialized_dataframe)
    restored_dataframe = pd.read_parquet(deserialized_data)
    return restored_dataframe
```
## Interaction Objects
Useful material:
- https://docs.pydantic.dev/2.3/usage/types/number_types/
- https://docs.pydantic.dev/latest/concepts/fields/#inspecting-model-fields
- https://stackoverflow.com/questions/76466468/pydantic-model-fields-with-typing-optional-vs-typing-optional-none
- https://stackoverflow.com/questions/69578431/how-to-fix-streamlitapiexception-expected-bytes-got-a-int-object-conver

We now have the necessery functions to store and get data, which means we only need to define the structures of dicts and dataframes. For our use case the important dictionaries are found in
- MANAGEMENT -> Enables HPC orchestration
- TIMES -> Enables workflow monitoring
- SACCT -> Enables HPC monitoring
- LOGS -> Enables HPC debugging

We can define their structures using pydantic.
### Management
Management object enables the communication between forwarder that creates the object by providing inputs, which are then used by Submitter to monitor Forwarder orders and HPC platforms. We define it using multiple parts:
```
from pydantic import BaseModel, Field
from typing import List, Dict
```
```
class Venv(BaseModel):
    name: str = Field(alias = 'name')
    path: str = Field(alias = 'path')
    modules: List[str] = Field(default_factory = list, alias = 'modules')
    packages: List[str] = Field(default_factory = list, alias = 'packages')

class Configs(BaseModel):
    modules: List[str] = Field(default_factory = list, alias = 'modules')
    venv: Venv = Field(alias = 'venv')
```
```
venv_test = {
    'name': 'exp-venv',
    'path': '/users/csc',
    'modules': [
        'pytorch'
    ],
    'packages': [
        'ray',
        'python-swiftclient',
        'torch',
        'torchmetrics'
    ]
}

validated_venv = Venv(**venv_test)

validated_venv.model_dump()
```
```
{'name': 'exp-venv',
 'path': '/users/csc',
 'modules': ['pytorch'],
 'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}
```
```
configs_test = {
    'modules': [
        'gcc',
        'openmpi',
        'openblas',
        'csc-tools',
        'StdEnv'
    ],
    'venv': venv_test
}

validated_config = Configs(**configs_test)

validated_config.model_dump()
```
```
{'modules': ['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'],
 'venv': {'name': 'exp-venv',
  'path': '/users/csc',
  'modules': ['pytorch'],
  'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}}
```
```
class Status(BaseModel):
    start: bool = Field(False, alias = 'start')
    config: bool = Field(False, alias = 'config'),
    submit: bool = Field(False, alias = 'submit'),
    pending: bool = Field(False, alias = 'pending'),
    running: bool = Field(False, alias = 'running')
    failed: bool = Field(False, alias = 'failed')
    complete: bool = Field(False, alias = 'complete')
    cancel: bool = Field(False, alias = 'cancel')
    stopped: bool = Field(False, alias = 'stopped')
    stored: bool = Field(False, alias = 'stored')

class Job(BaseModel):
    path: str = Field(alias = 'path')
    id: str = Field('None', alias = 'id')
    status: Status = Field(alias = 'status')
```
```
status_test = {
    'start': False,
    'config': False,
    'submit': False,
    'pending': False,
    'running': False,
    'failed': False,
    'complete': False,
    'cancel': False,
    'stopped': False,
    'stored': False
}

validated_status = Status(**status_test)

validated_status.model_dump()
```
```
{'start': False,
 'config': False,
 'submit': False,
 'pending': False,
 'running': False,
 'failed': False,
 'complete': False,
 'cancel': False,
 'stopped': False,
 'stored': False}
```
```
job_test = {
    'path': '/users/csc/ray-cluster.sh',
    'id': 'None',
    'status': status_test
}

validated_job = Job(**job_test)

validated_job.model_dump()
```
```
{'path': '/users/csc/ray-cluster.sh',
 'id': 'None',
 'status': {'start': False,
  'config': False,
  'submit': False,
  'pending': False,
  'running': False,
  'failed': False,
  'complete': False,
  'cancel': False,
  'stopped': False,
  'stored': False}}
```
```
class Direction(BaseModel):
    place: str = Field(alias = 'place')
    path: str = Field(alias = 'path')
    value: str = Field(alias = 'value')

class Transfer(BaseModel):
    source: Direction = Field(alias = 'source')
    target: Direction = Field(alias = 'target')

class Send(BaseModel):
    transfer: Transfer = Field(alias = 'transfer')
    overwrite: bool = Field(False, alias = 'overwrite')

class Get(BaseModel):
    transfer: Transfer = Field(alias = 'transfer')
    remove: bool = Field(False, alias = 'remove') 

class Files(BaseModel):
    send: List[Send] = Field(default_factory = list, alias = 'send')
    get: List[Get] = Field(default_factory = list, alias = 'get')
```
```
direction_test = {
    'place': 'local/submitter',
    'path': '/run/secrets/parameters',
    'value': 'integration/cloud-hpc/key'
}

validated_direction = Direction(**direction_test)

validated_direction.model_dump()
```
```
{'place': 'local/submitter',
 'path': '/run/secrets/parameters',
 'value': 'integration/cloud-hpc/key'}
```
```
transfer_test = {
    'source': {
        'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'
    },
    'target': {
        'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'
    } 
}

validated_transfer = Transfer(**transfer_test)

validated_transfer.model_dump()
```
```
{'source': {'place': 'local/submitter',
  'path': '/run/secrets/parameters',
  'value': 'integration/cloud-hpc/key'},
 'target': {'place': 'hpc/mahti',
  'path': '/users/csc/cloud-hpc.pem',
  'value': 'None'}}
```
```
send_test = {
    'transfer': transfer_test,
    'overwrite': False
}

validated_send = Send(**send_test)

validated_send.model_dump()
```
```
{'transfer': {'source': {'place': 'local/submitter',
   'path': '/run/secrets/parameters',
   'value': 'integration/cloud-hpc/key'},
  'target': {'place': 'hpc/mahti',
   'path': '/users/csc/cloud-hpc.pem',
   'value': 'None'}},
 'overwrite': False}
```
```
get_test = {
    'transfer': transfer_test,
    'remove': False
}

validated_get = Get(**get_test)

validated_get.model_dump()
```
```
{'transfer': {'source': {'place': 'local/submitter',
   'path': '/run/secrets/parameters',
   'value': 'integration/cloud-hpc/key'},
  'target': {'place': 'hpc/mahti',
   'path': '/users/csc/cloud-hpc.pem',
   'value': 'None'}},
 'remove': False}
```
```
files_test = {
    'send': [
        {
            'transfer': {
                'source': {
                    'place': 'local/submitter',
                    'path': '/run/secrets/parameters',
                    'value': 'integration/cloud-hpc/key'
                },
                'target': {
                    'place': 'hpc/mahti',
                    'path': '/users/csc/cloud-hpc.pem',
                    'value': 'None'
                },
            },
            'overwrite': False
        },
        {
            'transfer': {
                'source': {
                    'place': 'storage/allas/pipeline',
                    'path': '/CODE/SLURM/ray-cluster.sh',
                    'value': 'None'
                },
                'target': {
                    'place': 'hpc/mahti',
                    'path': '/users/csc/ray-cluster.sh',
                    'value': 'None'
                },
            },
            'overwrite': True
        }
    ],
    'get': [
        {
            'transfer': {    
                'source': {
                    'place': 'hpc/mahti',
                    'path': '/users/csc/slurm-{0}.out',
                    'value': 'None'
                },
                'target': {
                    'place': 'storage/allas/pipeline',
                    'path': 'LOGS/{0}',
                    'value': 'None'
                },
            },
            'remove': False
        }
    ]
}

validated_files = Files(**files_test)

validated_files.model_dump()
```
```
{'send': [{'transfer': {'source': {'place': 'local/submitter',
     'path': '/run/secrets/parameters',
     'value': 'integration/cloud-hpc/key'},
    'target': {'place': 'hpc/mahti',
     'path': '/users/csc/cloud-hpc.pem',
     'value': 'None'}},
   'overwrite': False},
  {'transfer': {'source': {'place': 'storage/allas/pipeline',
     'path': '/CODE/SLURM/ray-cluster.sh',
     'value': 'None'},
    'target': {'place': 'hpc/mahti',
     'path': '/users/csc/ray-cluster.sh',
     'value': 'None'}},
   'overwrite': True}],
 'get': [{'transfer': {'source': {'place': 'hpc/mahti',
     'path': '/users/csc/slurm-{0}.out',
     'value': 'None'},
    'target': {'place': 'storage/allas/pipeline',
     'path': 'LOGS/{0}',
     'value': 'None'}},
   'remove': False}]}<br></br>
We can the validate them with:
```
```
class Language(BaseModel):
    name: str = Field(alias = 'name')
    version: str = Field(alias = 'version')

class Workspace(BaseModel):
    path: str = Field(alias = 'path')
    used_capacity: str = Field(alias = 'used-capacity')
    max_capacity: str = Field(alias = 'max-capacity')
    used_files: str = Field(alias = 'used-files') 
    max_files: str = Field(alias = 'max-files')

class Properties(BaseModel):
    directory: str = Field(alias = 'directory')
    languages: List[Language] = Field(alias = 'languages')
    workspaces: List[Workspace] = Field(alias = 'workspaces')
```
```
language_test = {
    'name': 'Python',
    'version': '3.6.8'
}

validated_language = Language(**language_test)

validated_language.model_dump()
```
```
{'name': 'Python', 'version': '3.6.8'}
```
The next validation uses modified puhti, mahti and lumi file system prints found at logs folder, which we parse with:
```
def parse_hpc_workspaces(
    given_print: str
) -> any:
    row_format = given_print.split('\n')
    workspaces = []
    if 0 < len(row_format):
        valid_rows = [
            '/users/',
            '/project_'
        ]
        placement = [
            'path',
            'capacity',
            'files',
            'none'
        ]
        for row in row_format:
            if any(sub in row for sub in valid_rows):
                space = {
                    'path': None,
                    'capacity': None,
                    'files': None,
                }
                empty_split = row.split(' ')
                index = 0
                for case in empty_split:
                    if len(case) == 0:
                        continue
                    if index < 4:
                        place = placement[index]
                        if not place == 'none':
                            space[place] = case
                            if place == 'capacity' or place == 'files':
                                path_split = case.split('/')
                                used_key = 'used-' + place
                                max_key = 'max-' + place
                                space[used_key] = path_split[0]
                                space[max_key] = path_split[1]
                                del space[place]
                        index += 1
                workspaces.append(space)
    return workspaces
```
```
def hpc_format_workspace(
    workspace_path: str
) -> any:
    workspace_data = None
    with open(
        workspace_path, 
        'r'
    ) as f:
        workspace_data = f.read()
    workspace = parse_hpc_workspaces(
        given_print = workspace_data
    )
    return workspace
```
```
puhti_workspaces = hpc_format_workspace(
    workspace_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/puhti-workspaces.txt'
)
```
```
space_test_1 = puhti_workspaces[0]

validated_space_1 = Workspace(**space_test_1)

validated_space_1.model_dump()
```
```
{'path': '/users/csc',
 'used_capacity': '2.2M',
 'max_capacity': '10G',
 'used_files': '37',
 'max_files': '100K'}
```
```
properties_test_1 = {
    'directory': '/users/csc',
    'languages': [language_test],
    'workspaces': puhti_workspaces
}

validated_properties_1 = Properties(**properties_test_1)

validated_properties_1.model_dump()
```
```
{'directory': '/users/csc',
 'languages': [{'name': 'Python', 'version': '3.6.8'}],
 'workspaces': [{'path': '/users/csc',
   'used_capacity': '2.2M',
   'max_capacity': '10G',
   'used_files': '37',
   'max_files': '100K'},
  {'path': '/projappl/project_csc',
   'used_capacity': '120M',
   'max_capacity': '50G',
   'used_files': '265',
   'max_files': '100K'},
  {'path': '/scratch/project_csc',
   'used_capacity': '164M',
   'max_capacity': '1.0T',
   'used_files': '4',
   'max_files': '1.0M'},
  {'path': '/projappl/project_csc',
   'used_capacity': '4.8G',
   'max_capacity': '50G',
   'used_files': '19K',
   'max_files': '100K'},
  {'path': '/scratch/project_csc',
   'used_capacity': '32K',
   'max_capacity': '1.0T',
   'used_files': '8',
   'max_files': '1.0M'},
  {'path': '/projappl/project_csc',
   'used_capacity': '13M',
   'max_capacity': '50G',
   'used_files': '974',
   'max_files': '100K'},
  {'path': '/scratch/project_csc',
   'used_capacity': '4.0K',
   'max_capacity': '1.0T',
   'used_files': '1',
   'max_files': '1.0M'}]}
```
```
mahti_workspaces = hpc_format_workspace(
    workspace_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/mahti-workspaces.txt'
)
```
```
space_test_2 = mahti_workspaces[0]

validated_space_2 = Workspace(**space_test_2)

validated_space_2.model_dump()
```
```
{'path': '/users/csc',
 'used_capacity': '3.045G',
 'max_capacity': '10G',
 'used_files': '2.42k',
 'max_files': '100k'}
```
```
properties_test_2 = {
    'directory': '/users/csc',
    'languages': [language_test],
    'workspaces': mahti_workspaces
}

validated_properties_2 = Properties(**properties_test_2)

validated_properties_2.model_dump()
```
```
{'directory': '/users/csc',
 'languages': [{'name': 'Python', 'version': '3.6.8'}],
 'workspaces': [{'path': '/users/csc',
   'used_capacity': '3.045G',
   'max_capacity': '10G',
   'used_files': '2.42k',
   'max_files': '100k'},
  {'path': '/projappl/project_csc',
   'used_capacity': '4k',
   'max_capacity': '50G',
   'used_files': '0k',
   'max_files': '100k'},
  {'path': '/projappl/project_csc',
   'used_capacity': '4k',
   'max_capacity': '50G',
   'used_files': '0k',
   'max_files': '100k'},
  {'path': '/projappl/project_csc',
   'used_capacity': '12.86M',
   'max_capacity': '50G',
   'used_files': '.97k',
   'max_files': '100k'},
  {'path': '/scratch/project_csc',
   'used_capacity': '36.43G',
   'max_capacity': '1T',
   'used_files': '.64k',
   'max_files': '1000k'},
  {'path': '/scratch/project_csc',
   'used_capacity': '4k',
   'max_capacity': '1T',
   'used_files': '0k',
   'max_files': '1000k'},
  {'path': '/scratch/project_csc',
   'used_capacity': '4k',
   'max_capacity': '1T',
   'used_files': '0k',
   'max_files': '1000k'}]}
```
```
lumi_workspaces = hpc_format_workspace(
    workspace_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/lumi-workspaces.txt'
)
```
```
space_test_3 = lumi_workspaces[0]

validated_space_3 = Workspace(**space_test_3)

validated_space_3.model_dump()
```
```
{'path': '/users/csc',
 'used_capacity': '16M',
 'max_capacity': '22G',
 'used_files': '17',
 'max_files': '100K'}
```
```
properties_test_3 = {
    'directory': '/users/csc',
    'languages': [language_test],
    'workspaces': mahti_workspaces
}

validated_properties_3 = Properties(**properties_test_3)

validated_properties_3.model_dump()
```
```
{'directory': '/users/csc',
 'languages': [{'name': 'Python', 'version': '3.6.8'}],
 'workspaces': [{'path': '/users/csc',
   'used_capacity': '3.045G',
   'max_capacity': '10G',
   'used_files': '2.42k',
   'max_files': '100k'},
  {'path': '/projappl/project_csc',
   'used_capacity': '4k',
   'max_capacity': '50G',
   'used_files': '0k',
   'max_files': '100k'},
  {'path': '/projappl/project_csc',
   'used_capacity': '4k',
   'max_capacity': '50G',
   'used_files': '0k',
   'max_files': '100k'},
  {'path': '/projappl/project_csc',
   'used_capacity': '12.86M',
   'max_capacity': '50G',
   'used_files': '.97k',
   'max_files': '100k'},
  {'path': '/scratch/project_csc',
   'used_capacity': '36.43G',
   'max_capacity': '1T',
   'used_files': '.64k',
   'max_files': '1000k'},
  {'path': '/scratch/project_csc',
   'used_capacity': '4k',
   'max_capacity': '1T',
   'used_files': '0k',
   'max_files': '1000k'},
  {'path': '/scratch/project_csc',
   'used_capacity': '4k',
   'max_capacity': '1T',
   'used_files': '0k',
   'max_files': '1000k'}]}
```
```
class Platform(BaseModel):
    properties: Properties = Field(alias = 'properties')
    configs: Configs = Field(alias = 'configs')
    files: Files = Field(default_factory = list, alias = 'files')
    jobs: List[Job] = Field(default_factory = list, alias = 'jobs') 
    
class Orchestration(BaseModel):
    name: str = Field(alias = 'name')
    platforms: Dict[str, Platform] = Field(alias = 'platforms')
```
```
platform_test_1 = {
    'properties': properties_test_1,
    'configs': configs_test,
    'files': files_test,
    'jobs': [job_test] 
}

validation_platfrom_1 = Platform(**platform_test_1)

validation_platfrom_1.model_dump()
```
```
{'properties': {'directory': '/users/csc',
  'languages': [{'name': 'Python', 'version': '3.6.8'}],
  'workspaces': [{'path': '/users/csc',
    'used_capacity': '2.2M',
    'max_capacity': '10G',
    'used_files': '37',
    'max_files': '100K'},
   {'path': '/projappl/project_csc',
    'used_capacity': '120M',
    'max_capacity': '50G',
    'used_files': '265',
    'max_files': '100K'},
   {'path': '/scratch/project_csc',
    'used_capacity': '164M',
    'max_capacity': '1.0T',
    'used_files': '4',
    'max_files': '1.0M'},
   {'path': '/projappl/project_csc',
    'used_capacity': '4.8G',
    'max_capacity': '50G',
    'used_files': '19K',
    'max_files': '100K'},
   {'path': '/scratch/project_csc',
    'used_capacity': '32K',
    'max_capacity': '1.0T',
    'used_files': '8',
    'max_files': '1.0M'},
   {'path': '/projappl/project_csc',
    'used_capacity': '13M',
    'max_capacity': '50G',
    'used_files': '974',
    'max_files': '100K'},
   {'path': '/scratch/project_csc',
    'used_capacity': '4.0K',
    'max_capacity': '1.0T',
    'used_files': '1',
    'max_files': '1.0M'}]},
 'configs': {'modules': ['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'],
  'venv': {'name': 'exp-venv',
   'path': '/users/csc',
   'modules': ['pytorch'],
   'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
 'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
      'path': '/run/secrets/parameters',
      'value': 'integration/cloud-hpc/key'},
     'target': {'place': 'hpc/mahti',
      'path': '/users/csc/cloud-hpc.pem',
      'value': 'None'}},
    'overwrite': False},
   {'transfer': {'source': {'place': 'storage/allas/pipeline',
      'path': '/CODE/SLURM/ray-cluster.sh',
      'value': 'None'},
     'target': {'place': 'hpc/mahti',
      'path': '/users/csc/ray-cluster.sh',
      'value': 'None'}},
    'overwrite': True}],
  'get': [{'transfer': {'source': {'place': 'hpc/mahti',
      'path': '/users/csc/slurm-{0}.out',
      'value': 'None'},
     'target': {'place': 'storage/allas/pipeline',
      'path': 'LOGS/{0}',
      'value': 'None'}},
    'remove': False}]},
 'jobs': [{'path': '/users/csc/ray-cluster.sh',
   'id': 'None',
   'status': {'start': False,
    'config': False,
    'submit': False,
    'pending': False,
    'running': False,
    'failed': False,
    'complete': False,
    'cancel': False,
    'stopped': False,
    'stored': False}}]}
```
```
platform_test_2 = {
    'properties': properties_test_2,
    'configs': configs_test,
    'files': files_test,
    'jobs': [job_test] 
}

validation_platfrom_2 = Platform(**platform_test_2)

validation_platfrom_2.model_dump()
```
```
{'properties': {'directory': '/users/csc',
  'languages': [{'name': 'Python', 'version': '3.6.8'}],
  'workspaces': [{'path': '/users/csc',
    'used_capacity': '3.045G',
    'max_capacity': '10G',
    'used_files': '2.42k',
    'max_files': '100k'},
   {'path': '/projappl/project_csc',
    'used_capacity': '4k',
    'max_capacity': '50G',
    'used_files': '0k',
    'max_files': '100k'},
   {'path': '/projappl/project_csc',
    'used_capacity': '4k',
    'max_capacity': '50G',
    'used_files': '0k',
    'max_files': '100k'},
   {'path': '/projappl/project_csc',
    'used_capacity': '12.86M',
    'max_capacity': '50G',
    'used_files': '.97k',
    'max_files': '100k'},
   {'path': '/scratch/project_csc',
    'used_capacity': '36.43G',
    'max_capacity': '1T',
    'used_files': '.64k',
    'max_files': '1000k'},
   {'path': '/scratch/project_csc',
    'used_capacity': '4k',
    'max_capacity': '1T',
    'used_files': '0k',
    'max_files': '1000k'},
   {'path': '/scratch/project_csc',
    'used_capacity': '4k',
    'max_capacity': '1T',
    'used_files': '0k',
    'max_files': '1000k'}]},
 'configs': {'modules': ['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'],
  'venv': {'name': 'exp-venv',
   'path': '/users/csc',
   'modules': ['pytorch'],
   'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
 'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
      'path': '/run/secrets/parameters',
      'value': 'integration/cloud-hpc/key'},
     'target': {'place': 'hpc/mahti',
      'path': '/users/csc/cloud-hpc.pem',
      'value': 'None'}},
    'overwrite': False},
   {'transfer': {'source': {'place': 'storage/allas/pipeline',
      'path': '/CODE/SLURM/ray-cluster.sh',
      'value': 'None'},
     'target': {'place': 'hpc/mahti',
      'path': '/users/csc/ray-cluster.sh',
      'value': 'None'}},
    'overwrite': True}],
  'get': [{'transfer': {'source': {'place': 'hpc/mahti',
      'path': '/users/csc/slurm-{0}.out',
      'value': 'None'},
     'target': {'place': 'storage/allas/pipeline',
      'path': 'LOGS/{0}',
      'value': 'None'}},
    'remove': False}]},
 'jobs': [{'path': '/users/csc/ray-cluster.sh',
   'id': 'None',
   'status': {'start': False,
    'config': False,
    'submit': False,
    'pending': False,
    'running': False,
    'failed': False,
    'complete': False,
    'cancel': False,
    'stopped': False,
    'stored': False}}]}
```
```
platform_test_3 = {
    'properties': properties_test_3,
    'configs': configs_test,
    'files': files_test,
    'jobs': [job_test] 
}

validation_platfrom_3 = Platform(**platform_test_3)

validation_platfrom_3.model_dump()
```
```
{'properties': {'directory': '/users/csc',
  'languages': [{'name': 'Python', 'version': '3.6.8'}],
  'workspaces': [{'path': '/users/csc',
    'used_capacity': '3.045G',
    'max_capacity': '10G',
    'used_files': '2.42k',
    'max_files': '100k'},
   {'path': '/projappl/project_csc',
    'used_capacity': '4k',
    'max_capacity': '50G',
    'used_files': '0k',
    'max_files': '100k'},
   {'path': '/projappl/project_csc',
    'used_capacity': '4k',
    'max_capacity': '50G',
    'used_files': '0k',
    'max_files': '100k'},
   {'path': '/projappl/project_csc',
    'used_capacity': '12.86M',
    'max_capacity': '50G',
    'used_files': '.97k',
    'max_files': '100k'},
   {'path': '/scratch/project_csc',
    'used_capacity': '36.43G',
    'max_capacity': '1T',
    'used_files': '.64k',
    'max_files': '1000k'},
   {'path': '/scratch/project_csc',
    'used_capacity': '4k',
    'max_capacity': '1T',
    'used_files': '0k',
    'max_files': '1000k'},
   {'path': '/scratch/project_csc',
    'used_capacity': '4k',
    'max_capacity': '1T',
    'used_files': '0k',
    'max_files': '1000k'}]},
 'configs': {'modules': ['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'],
  'venv': {'name': 'exp-venv',
   'path': '/users/csc',
   'modules': ['pytorch'],
   'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
 'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
      'path': '/run/secrets/parameters',
      'value': 'integration/cloud-hpc/key'},
     'target': {'place': 'hpc/mahti',
      'path': '/users/csc/cloud-hpc.pem',
      'value': 'None'}},
    'overwrite': False},
   {'transfer': {'source': {'place': 'storage/allas/pipeline',
      'path': '/CODE/SLURM/ray-cluster.sh',
      'value': 'None'},
     'target': {'place': 'hpc/mahti',
      'path': '/users/csc/ray-cluster.sh',
      'value': 'None'}},
    'overwrite': True}],
  'get': [{'transfer': {'source': {'place': 'hpc/mahti',
      'path': '/users/csc/slurm-{0}.out',
      'value': 'None'},
     'target': {'place': 'storage/allas/pipeline',
      'path': 'LOGS/{0}',
      'value': 'None'}},
    'remove': False}]},
 'jobs': [{'path': '/users/csc/ray-cluster.sh',
   'id': 'None',
   'status': {'start': False,
    'config': False,
    'submit': False,
    'pending': False,
    'running': False,
    'failed': False,
    'complete': False,
    'cancel': False,
    'stopped': False,
    'stored': False}}]}
```
```
orhestration_test = {
    'name': 'multi-cloud-hpc-integration',
    'platforms': {
        'puhti': platform_test_1,
        'mahti': platform_test_2,
        'lumi': platform_test_3
    }
}

validation_orchestration = Orchestration(**orhestration_test)

validation_orchestration.model_dump()
```
```
{'name': 'multi-cloud-hpc-integration',
 'platforms': {'puhti': {'properties': {'directory': '/users/csc',
    'languages': [{'name': 'Python', 'version': '3.6.8'}],
    'workspaces': [{'path': '/users/csc',
      'used_capacity': '2.2M',
      'max_capacity': '10G',
      'used_files': '37',
      'max_files': '100K'},
     {'path': '/projappl/project_csc',
      'used_capacity': '120M',
      'max_capacity': '50G',
      'used_files': '265',
      'max_files': '100K'},
     {'path': '/scratch/project_csc',
      'used_capacity': '164M',
      'max_capacity': '1.0T',
      'used_files': '4',
      'max_files': '1.0M'},
     {'path': '/projappl/project_csc',
      'used_capacity': '4.8G',
      'max_capacity': '50G',
      'used_files': '19K',
      'max_files': '100K'},
     {'path': '/scratch/project_csc',
      'used_capacity': '32K',
      'max_capacity': '1.0T',
      'used_files': '8',
      'max_files': '1.0M'},
     {'path': '/projappl/project_csc',
      'used_capacity': '13M',
      'max_capacity': '50G',
      'used_files': '974',
      'max_files': '100K'},
     {'path': '/scratch/project_csc',
      'used_capacity': '4.0K',
      'max_capacity': '1.0T',
      'used_files': '1',
      'max_files': '1.0M'}]},
   'configs': {'modules': ['gcc',
     'openmpi',
     'openblas',
     'csc-tools',
     'StdEnv'],
    'venv': {'name': 'exp-venv',
     'path': '/users/csc',
     'modules': ['pytorch'],
     'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
   'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'}},
      'overwrite': False},
     {'transfer': {'source': {'place': 'storage/allas/pipeline',
        'path': '/CODE/SLURM/ray-cluster.sh',
        'value': 'None'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/ray-cluster.sh',
        'value': 'None'}},
      'overwrite': True}],
    'get': [{'transfer': {'source': {'place': 'hpc/mahti',
        'path': '/users/csc/slurm-{0}.out',
        'value': 'None'},
       'target': {'place': 'storage/allas/pipeline',
        'path': 'LOGS/{0}',
        'value': 'None'}},
      'remove': False}]},
   'jobs': [{'path': '/users/csc/ray-cluster.sh',
     'id': 'None',
     'status': {'start': False,
      'config': False,
      'submit': False,
      'pending': False,
      'running': False,
      'failed': False,
      'complete': False,
      'cancel': False,
      'stopped': False,
      'stored': False}}]},
  'mahti': {'properties': {'directory': '/users/csc',
    'languages': [{'name': 'Python', 'version': '3.6.8'}],
    'workspaces': [{'path': '/users/csc',
      'used_capacity': '3.045G',
      'max_capacity': '10G',
      'used_files': '2.42k',
      'max_files': '100k'},
     {'path': '/projappl/project_csc',
      'used_capacity': '4k',
      'max_capacity': '50G',
      'used_files': '0k',
      'max_files': '100k'},
     {'path': '/projappl/project_csc',
      'used_capacity': '4k',
      'max_capacity': '50G',
      'used_files': '0k',
      'max_files': '100k'},
     {'path': '/projappl/project_csc',
      'used_capacity': '12.86M',
      'max_capacity': '50G',
      'used_files': '.97k',
      'max_files': '100k'},
     {'path': '/scratch/project_csc',
      'used_capacity': '36.43G',
      'max_capacity': '1T',
      'used_files': '.64k',
      'max_files': '1000k'},
     {'path': '/scratch/project_csc',
      'used_capacity': '4k',
      'max_capacity': '1T',
      'used_files': '0k',
      'max_files': '1000k'},
     {'path': '/scratch/project_csc',
      'used_capacity': '4k',
      'max_capacity': '1T',
      'used_files': '0k',
      'max_files': '1000k'}]},
   'configs': {'modules': ['gcc',
     'openmpi',
     'openblas',
     'csc-tools',
     'StdEnv'],
    'venv': {'name': 'exp-venv',
     'path': '/users/csc',
     'modules': ['pytorch'],
     'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
   'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'}},
      'overwrite': False},
     {'transfer': {'source': {'place': 'storage/allas/pipeline',
        'path': '/CODE/SLURM/ray-cluster.sh',
        'value': 'None'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/ray-cluster.sh',
        'value': 'None'}},
      'overwrite': True}],
    'get': [{'transfer': {'source': {'place': 'hpc/mahti',
        'path': '/users/csc/slurm-{0}.out',
        'value': 'None'},
       'target': {'place': 'storage/allas/pipeline',
        'path': 'LOGS/{0}',
        'value': 'None'}},
      'remove': False}]},
   'jobs': [{'path': '/users/csc/ray-cluster.sh',
     'id': 'None',
     'status': {'start': False,
      'config': False,
      'submit': False,
      'pending': False,
      'running': False,
      'failed': False,
      'complete': False,
      'cancel': False,
      'stopped': False,
      'stored': False}}]},
  'lumi': {'properties': {'directory': '/users/csc',
    'languages': [{'name': 'Python', 'version': '3.6.8'}],
    'workspaces': [{'path': '/users/csc',
      'used_capacity': '3.045G',
      'max_capacity': '10G',
      'used_files': '2.42k',
      'max_files': '100k'},
     {'path': '/projappl/project_csc',
      'used_capacity': '4k',
      'max_capacity': '50G',
      'used_files': '0k',
      'max_files': '100k'},
     {'path': '/projappl/project_csc',
      'used_capacity': '4k',
      'max_capacity': '50G',
      'used_files': '0k',
      'max_files': '100k'},
     {'path': '/projappl/project_csc',
      'used_capacity': '12.86M',
      'max_capacity': '50G',
      'used_files': '.97k',
      'max_files': '100k'},
     {'path': '/scratch/project_csc',
      'used_capacity': '36.43G',
      'max_capacity': '1T',
      'used_files': '.64k',
      'max_files': '1000k'},
     {'path': '/scratch/project_csc',
      'used_capacity': '4k',
      'max_capacity': '1T',
      'used_files': '0k',
      'max_files': '1000k'},
     {'path': '/scratch/project_csc',
      'used_capacity': '4k',
      'max_capacity': '1T',
      'used_files': '0k',
      'max_files': '1000k'}]},
   'configs': {'modules': ['gcc',
     'openmpi',
     'openblas',
     'csc-tools',
     'StdEnv'],
    'venv': {'name': 'exp-venv',
     'path': '/users/csc',
     'modules': ['pytorch'],
     'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
   'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'}},
      'overwrite': False},
     {'transfer': {'source': {'place': 'storage/allas/pipeline',
        'path': '/CODE/SLURM/ray-cluster.sh',
        'value': 'None'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/ray-cluster.sh',
        'value': 'None'}},
      'overwrite': True}],
    'get': [{'transfer': {'source': {'place': 'hpc/mahti',
        'path': '/users/csc/slurm-{0}.out',
        'value': 'None'},
       'target': {'place': 'storage/allas/pipeline',
        'path': 'LOGS/{0}',
        'value': 'None'}},
      'remove': False}]},
   'jobs': [{'path': '/users/csc/ray-cluster.sh',
     'id': 'None',
     'status': {'start': False,
      'config': False,
      'submit': False,
      'pending': False,
      'running': False,
      'failed': False,
      'complete': False,
      'cancel': False,
      'stopped': False,
      'stored': False}}]}}}
```
As we have now seen, the orchestration dict is very long and mixed human and machine inputet parts. For clarity, here are the parts that are given when a management instnace is created:
```
provided_orch_input = {
    'name': 'multi-cloud-hpc-integration',
    'platforms': {
        'puhti': {
            'configs': configs_test,
            'files': files_test,
            'jobs': [job_test] 
        },
        'mahti': {
            'configs': configs_test,
            'files': files_test,
            'jobs': [job_test] 
        },
        'lumi': {
            'configs': configs_test,
            'files': files_test,
            'jobs': [job_test] 
        }
    }
}
```
This input is provided by the forwarder, who stores the input in sub-air/MANAGEMENT in the following way:
```
import pickle
formatted_orch_input = pickle.dumps(provided_orch_input)
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'submitter',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'mana',
        'path-replacers': {
            'name': '1.pkl'
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = formatted_orch_input,
    object_metadata = {'version': 1}
)
```
```
User object bucket: mlch-sub-air-user-example-com
Used object path: MANAGEMENT/1.pkl
```
Now, if you go into https://pouta.csc.fi/, you should see this container and file appear by doing the following:
- Click object storage
- Click containers
- Click mlch-sub-air-user-example-com
- Click MANAGEMENT We can get the dictionary with the following way:
```
get_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'get',
        'bucket-target': 'submitter',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'mana',
        'path-replacers': {
            'name': '1.pkl'
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = None,
    object_metadata = None
)
```
```
User object bucket: mlch-sub-air-user-example-com
Used object path: MANAGEMENT/1.pkl
```
```
formatted_stored_orch = pickle.loads(get_interaction_output[0])
stored_general_metadata = get_interaction_output[1]
stored_custom_metadata = get_interaction_output[2]
```
```
formatted_stored_orch
```
```
{'name': 'multi-cloud-hpc-integration',
 'platforms': {'puhti': {'configs': {'modules': ['gcc',
     'openmpi',
     'openblas',
     'csc-tools',
     'StdEnv'],
    'venv': {'name': 'exp-venv',
     'path': '/users/csc',
     'modules': ['pytorch'],
     'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
   'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'}},
      'overwrite': False},
     {'transfer': {'source': {'place': 'storage/allas/pipeline',
        'path': '/CODE/SLURM/ray-cluster.sh',
        'value': 'None'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/ray-cluster.sh',
        'value': 'None'}},
      'overwrite': True}],
    'get': [{'transfer': {'source': {'place': 'hpc/mahti',
        'path': '/users/csc/slurm-{0}.out',
        'value': 'None'},
       'target': {'place': 'storage/allas/pipeline',
        'path': 'LOGS/{0}',
        'value': 'None'}},
      'remove': False}]},
   'jobs': [{'path': '/users/csc/ray-cluster.sh',
     'id': 'None',
     'status': {'start': False,
      'config': False,
      'submit': False,
      'pending': False,
      'running': False,
      'failed': False,
      'complete': False,
      'cancel': False,
      'stopped': False,
      'stored': False}}]},
  'mahti': {'configs': {'modules': ['gcc',
     'openmpi',
     'openblas',
     'csc-tools',
     'StdEnv'],
    'venv': {'name': 'exp-venv',
     'path': '/users/csc',
     'modules': ['pytorch'],
     'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
   'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'}},
      'overwrite': False},
     {'transfer': {'source': {'place': 'storage/allas/pipeline',
        'path': '/CODE/SLURM/ray-cluster.sh',
        'value': 'None'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/ray-cluster.sh',
        'value': 'None'}},
      'overwrite': True}],
    'get': [{'transfer': {'source': {'place': 'hpc/mahti',
        'path': '/users/csc/slurm-{0}.out',
        'value': 'None'},
       'target': {'place': 'storage/allas/pipeline',
        'path': 'LOGS/{0}',
        'value': 'None'}},
      'remove': False}]},
   'jobs': [{'path': '/users/csc/ray-cluster.sh',
     'id': 'None',
     'status': {'start': False,
      'config': False,
      'submit': False,
      'pending': False,
      'running': False,
      'failed': False,
      'complete': False,
      'cancel': False,
      'stopped': False,
      'stored': False}}]},
  'lumi': {'configs': {'modules': ['gcc',
     'openmpi',
     'openblas',
     'csc-tools',
     'StdEnv'],
    'venv': {'name': 'exp-venv',
     'path': '/users/csc',
     'modules': ['pytorch'],
     'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}},
   'files': {'send': [{'transfer': {'source': {'place': 'local/submitter',
        'path': '/run/secrets/parameters',
        'value': 'integration/cloud-hpc/key'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/cloud-hpc.pem',
        'value': 'None'}},
      'overwrite': False},
     {'transfer': {'source': {'place': 'storage/allas/pipeline',
        'path': '/CODE/SLURM/ray-cluster.sh',
        'value': 'None'},
       'target': {'place': 'hpc/mahti',
        'path': '/users/csc/ray-cluster.sh',
        'value': 'None'}},
      'overwrite': True}],
    'get': [{'transfer': {'source': {'place': 'hpc/mahti',
        'path': '/users/csc/slurm-{0}.out',
        'value': 'None'},
       'target': {'place': 'storage/allas/pipeline',
        'path': 'LOGS/{0}',
        'value': 'None'}},
      'remove': False}]},
   'jobs': [{'path': '/users/csc/ray-cluster.sh',
     'id': 'None',
     'status': {'start': False,
      'config': False,
      'submit': False,
      'pending': False,
      'running': False,
      'failed': False,
      'complete': False,
      'cancel': False,
      'stopped': False,
      'stored': False}}]}}}
```
You can also check the custom metadata with:
```
stored_custom_metadata
```
```
{'version': 1}
```
### Times
Times object enables easy efficency monitoring of workflow components by enabling time monitoring in the run integration pipelines and Ray jobs. It is defined:
```
class Time(BaseModel):
    begin: int = Field(0, alias = 'begin')
    end: int = Field(0, alias = 'end')
    total: int = Field(0, alias = 'total')

class Times(BaseModel):
    name: str = Field(alias = 'name')
    cases: Dict[str, Time] = Field(alias = 'cases')
```
```
time_test = {
    'begin': 0,
    'end': 0,
    'total': 0
}

validation_time = Time(**time_test)

validation_time.model_dump()
```
```
{'begin': 0, 'end': 0, 'total': 0}
```
```
times_test = {
    'name': 'run-1',
    'cases': {
        'start': {
            'begin': 0,
            'end': 0,
            'total': 0
        },
        'config': {
            'begin': 0,
            'end': 0,
            'total': 0
        },
        'run': {
            'begin': 0,
            'end': 0,
            'total': 0
        },
        'cancel': {
            'begin': 0,
            'end': 0,
            'total': 0
        },
        'store': {
            'begin': 0,
            'end': 0,
            'total': 0
        }
    }
}

validation_times = Times(**times_test)

validation_times.model_dump()
```
```
{'name': 'run-1',
 'cases': {'start': {'begin': 0, 'end': 0, 'total': 0},
  'config': {'begin': 0, 'end': 0, 'total': 0},
  'run': {'begin': 0, 'end': 0, 'total': 0},
  'cancel': {'begin': 0, 'end': 0, 'total': 0},
  'store': {'begin': 0, 'end': 0, 'total': 0}}}
```
We can store these into pipeline/TIMES in the following way:
```
import pandas as pd

file_name = times_test['name'] + '.parquet'
times_df = pd.DataFrame.from_dict(times_test['cases'], orient = 'index')
```
```
times_df
```
```
        begin	end	    total
start	0	    0	    0
config	0	    0	    0
run	    0	    0	    0
cancel	0	    0	    0
store	0	    0	    0
```
```
formatted_times_data = pyarrow_serialize_dataframe(
    dataframe = times_df
)
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'time',
        'path-replacers': {
            'name': file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = formatted_times_data,
    object_metadata = {'version': 1}
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: TIMES/run-1.parquet
```
Now, if you go into https://pouta.csc.fi/, you should see this container and file appear by doing the following:
- Click object storage
- Click containers
- Click mlch-pipe-user-example-com
- Click TIMES We can get the dictionary with the following way:
```
get_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'get',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'time',
        'path-replacers': {
            'name': file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = None,
    object_metadata = None
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: TIMES/run-1.parquet
```
```
formatted_stored_times = pyarrow_deserialize_dataframe(serialized_dataframe = get_interaction_output[0])
stored_general_metadata = get_interaction_output[1]
stored_custom_metadata = get_interaction_output[2]
```
```
formatted_stored_times
```
```
        begin   end     total
start	0	    0	    0
config	0	    0	    0
run	    0	    0	    0
cancel	0	    0	    0
store	0	    0	    0
```
```
You can again check the custom metadata with:
```
```
stored_custom_metadata
```
```
{'version': 1}
```
### SACCT
Sacct object enables the monitoring of HPC job performane by enabling parsing of SLURM SACCT inputs. We define it with:
```
from typing import Optional

class Row(BaseModel):
    job_id: Optional[str] = Field(required=False, default = 'empty', alias = 'job-id') 
    job_name: Optional[str] = Field(required=False, default = 'empty',alias = 'job-name') 
    account: Optional[str] = Field(required=False, default = 'empty', alias = 'account') 
    parition: Optional[str] = Field(required=False, default = 'empty', alias = 'partition') 
    req_cpus: Optional[int] = Field(required=False, default = 0, alias = 'req-cpus') 
    alloc_cpus: Optional[int] = Field(required=False, default = 0, alias = 'alloc-cpus') 
    req_nodes: Optional[int] = Field(required=False, default = 0, alias = 'req-nodes') 
    alloc_nodes: Optional[int] = Field(required=False, default = 0,alias = 'alloc-nodes') 
    state: Optional[str] = Field(required=False, default = 'empty',alias = 'state') 
    ave_cpu: Optional[float] = Field(required=False, default = 0, alias = 'ave-cpu-seconds') 
    ave_cpu_freq: Optional[float] = Field(required=False, default = 0, alias = 'ave-cpu-freq-khz') 
    ave_disk_read: Optional[float] = Field(required=False, default = 0, alias = 'ave-disk-read-bytes') 
    ave_disk_write: Optional[float] = Field(required=False, default = 0, alias = 'ave-disk-write-bytes') 
    timelimit: Optional[float] = Field(required=False, default = 0, alias = 'timelimit-seconds') 
    submit: Optional[float] = Field(required=False, default = 0, alias = 'submit-time') 
    start: Optional[float] = Field(required=False, default = 0,alias = 'start-time') 
    elapsed: Optional[float] = Field(required=False, default = 0,alias = 'elapsed-seconds') 
    planned: Optional[float] = Field(required=False, default = 0,alias = 'planned-seconds') 
    end: Optional[float] = Field(required=False, default = 0,alias = 'end-time') 
    planned_cpu: Optional[float] = Field(required=False, default = 0,alias = 'planned-cpu-seconds') 
    cpu_time: Optional[float] = Field(required=False, default = 0, alias = 'cpu-time-seconds') 
    total_cpu: Optional[float] = Field(required=False, default = 0, alias = 'total-cpu-seconds') 
   
class Sacct(BaseModel):
    name: str = Field(alias = 'name')
    rows: Dict[int, Row] = Field(alias = 'rows')
```
```
from datetime import timedelta

def unit_converter(
    value: str,
    bytes: bool
) -> any:
    units = {
        'K': {
            'normal': 1000,
            'bytes': 1024
        },
        'M': {
            'normal': 1000**2,
            'bytes': 1024**2
        },
        'G': {
            'normal': 1000**3,
            'bytes': 1024**3
        },
        'T': {
            'normal': 1000**4,
            'bytes': 1024**4
        },
        'P': {
            'normal': 1000**5,
            'bytes': 1024**5
        }
    }
    
    converted_value = 0
    unit_letter = ''

    character_index = 0
    for character in value:
        if character.isalpha():
            unit_letter = character
            break
        character_index += 1
    
    if 0 < len(unit_letter):
        if not bytes:
            converted_value = int(float(value[:character_index]) * units[unit_letter]['normal'])
        else:
            converted_value = int(float(value[:character_index]) * units[unit_letter]['bytes'])
    else:
        converted_value = value
    return converted_value

def convert_into_seconds(
    given_time: str
) -> int:
    days = 0
    hours = 0
    minutes = 0
    seconds = 0
    milliseconds = 0

    day_split = given_time.split('-')
    if '-' in given_time:
        days = int(day_split[0])

    millisecond_split = day_split[-1].split('.')
    if '.' in given_time:
        milliseconds = int(millisecond_split[1])
    
    hour_minute_second_split = millisecond_split[0].split(':')

    if len(hour_minute_second_split) == 3:
        hours = int(hour_minute_second_split[0])
        minutes = int(hour_minute_second_split[1])
        seconds = int(hour_minute_second_split[2])
    else:
        minutes = int(hour_minute_second_split[0])
        seconds = int(hour_minute_second_split[1])
    
    result = timedelta(
        days = days,
        hours = hours,
        minutes = minutes,
        seconds = seconds,
        milliseconds = milliseconds
    ).total_seconds()
    return result
```
```
from datetime import datetime

def format_slurm_sacct(
    file_path: str
) -> any: 
    sacct_text = None
    with open(file_path, 'r') as f:
        sacct_text = f.readlines()

    values_dict = {}
    if not sacct_text is None:
        header = sacct_text[0].split(' ')[:-1]
        columns = [s for s in header if s != '']

        spaces = sacct_text[1].split(' ')[:-1]
        space_sizes = []
        for space in spaces:
            space_sizes.append(len(space))

        rows = sacct_text[2:]
        if 1 < len(sacct_text):
            for i in range(1,len(rows) + 1):
                values_dict[str(i)] = {} 
                for column in columns:
                    values_dict[str(i)][column] = ''
            i = 1
            for row in rows:
                start = 0
                end = 0
                j = 0
                
                for size in space_sizes:
                    end += size
                    formatted_value = [s for s in row[start:end].split(' ') if s != '']
                    column_value = None
                    if 0 < len(formatted_value):
                        column_value = formatted_value[0] 
                    column = columns[j]
                    values_dict[str(i)][column] = column_value
                    start = end
                    end += 1
                    j += 1
                i += 1
    return values_dict

def sacct_metric_formatting(
    metric: str
) -> any:
    formatted_name = ''
    first = True
    index = -1
    for character in metric:
        index += 1
        if character.isupper():
            if first:
                first = False
                formatted_name += character
                continue
            if index + 1 < len(metric): 
                if metric[index - 1].islower():
                    formatted_name += '-' + character
                    continue
                if metric[index - 1].isupper() and metric[index + 1].islower():
                    formatted_name += '-' + character
                    continue
        formatted_name += character 
    return formatted_name

def parse_sacct_dict(
    sacct_data:any
):
    formatted_data = {}

    metric_units = {
        'ave-cpu': 'seconds',
        'ave-cpu-freq': 'khz',
        'ave-disk-read': 'bytes',
        'ave-disk-write': 'bytes',
        'timelimit': 'seconds',
        'elapsed': 'seconds',
        'planned': 'seconds',
        'planned-cpu': 'seconds',
        'cpu-time': 'seconds',
        'total-cpu': 'seconds',
        'submit': 'time',
        'start': 'time',
        'end': 'time'
    }
   
    for key,value in sacct_data.items():
        spaced_key = sacct_metric_formatting(
            metric = key
        )
        formatted_key = spaced_key.lower()
        
        if formatted_key in metric_units:
            formatted_key += '-' + metric_units[formatted_key]
        
        formatted_data[formatted_key] = value

    ignore = [
        'account'
    ]
    
    metadata = [
        'job-name',
        'job-id',
        'partition',
        'state'
    ]
    
    parsed_metrics = {}
    parsed_metadata = {}
    
    for key in formatted_data.keys():
        key_value = formatted_data[key]

        if key_value is None:
            continue

        if key in ignore:
            continue

        if key in metadata:
            if key == 'job-id':
                key_value = key_value.split('.')[0]
            parsed_metadata[key] = key_value
            continue
        
        if ':' in key_value:
            if 'T' in key_value:
                format = datetime.strptime(key_value, '%Y-%m-%dT%H:%M:%S')
                key_value = round(format.timestamp())
            else:
                key_value = convert_into_seconds(
                    given_time = key_value
                )
        else:
            if 'bytes' in key_value:
                key_value = unit_converter(
                    value = key_value,
                    bytes = True
                )
            else:
                key_value = unit_converter(
                    value = key_value,
                    bytes = False
                )
        parsed_metrics[key] = key_value
    return parsed_metrics, parsed_metadata
```
You find example Sacct samples in logs folder, which we can use in the following way:
```
sacct_dict = format_slurm_sacct(
    file_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/sacct_1.txt'
)
```
```
rows = {}
i = 0
for sample in sacct_dict.values():
    metrics, metadata = parse_sacct_dict(
        sacct_data = sample
    )
    row = metrics | metadata
    validation_row = Row(**row)
    rows[i] = row
    i += 1
```
```
sacct_test = {
    'name': 'run-1',
    'rows': rows
}

validation_sacct = Sacct(**sacct_test)

validation_sacct.model_dump()
```
```
{'name': 'run-1',
 'rows': {0: {'job_id': '3470718',
   'job_name': 'ray-clust+',
   'account': 'empty',
   'parition': 'medium',
   'req_cpus': 2,
   'alloc_cpus': 512,
   'req_nodes': 2,
   'alloc_nodes': 2,
   'state': 'CANCELLED+',
   'ave_cpu': 0,
   'ave_cpu_freq': 0,
   'ave_disk_read': 0,
   'ave_disk_write': 0,
   'timelimit': 1200.0,
   'submit': 1716736668.0,
   'start': 1716736669.0,
   'elapsed': 100.0,
   'planned': 1.0,
   'end': 1716736769.0,
   'planned_cpu': 2.0,
   'cpu_time': 51200.0,
   'total_cpu': 18.362},
  1: {'job_id': '3470718',
   'job_name': 'batch',
   'account': 'empty',
   'parition': 'empty',
   'req_cpus': 256,
   'alloc_cpus': 256,
   'req_nodes': 1,
   'alloc_nodes': 1,
   'state': 'CANCELLED',
   'ave_cpu': 2.0,
   'ave_cpu_freq': 665330000.0,
   'ave_disk_read': 20340000.0,
   'ave_disk_write': 80000.0,
   'timelimit': 0,
   'submit': 1716736669.0,
   'start': 1716736669.0,
   'elapsed': 101.0,
   'planned': 0,
   'end': 1716736770.0,
   'planned_cpu': 0,
   'cpu_time': 25856.0,
   'total_cpu': 2.222},
  2: {'job_id': '3470718',
   'job_name': 'extern',
   'account': 'empty',
   'parition': 'empty',
   'req_cpus': 512,
   'alloc_cpus': 512,
   'req_nodes': 2,
   'alloc_nodes': 2,
   'state': 'COMPLETED',
   'ave_cpu': 0.0,
   'ave_cpu_freq': 5200000000.0,
   'ave_disk_read': 10000.0,
   'ave_disk_write': 0.0,
   'timelimit': 0,
   'submit': 1716736669.0,
   'start': 1716736669.0,
   'elapsed': 100.0,
   'planned': 0,
   'end': 1716736769.0,
   'planned_cpu': 0,
   'cpu_time': 51200.0,
   'total_cpu': 0.002},
  3: {'job_id': '3470718',
   'job_name': 'hostname',
   'account': 'empty',
   'parition': 'empty',
   'req_cpus': 2,
   'alloc_cpus': 2,
   'req_nodes': 1,
   'alloc_nodes': 1,
   'state': 'COMPLETED',
   'ave_cpu': 0.0,
   'ave_cpu_freq': 2600000000.0,
   'ave_disk_read': 0.0,
   'ave_disk_write': 0.0,
   'timelimit': 0,
   'submit': 1716736681.0,
   'start': 1716736681.0,
   'elapsed': 0.0,
   'planned': 0,
   'end': 1716736681.0,
   'planned_cpu': 0,
   'cpu_time': 0.0,
   'total_cpu': 0.044},
  4: {'job_id': '3470718',
   'job_name': 'singulari+',
   'account': 'empty',
   'parition': 'empty',
   'req_cpus': 2,
   'alloc_cpus': 2,
   'req_nodes': 1,
   'alloc_nodes': 1,
   'state': 'CANCELLED',
   'ave_cpu': 11.0,
   'ave_cpu_freq': 327530000.0,
   'ave_disk_read': 106440000.0,
   'ave_disk_write': 560000.0,
   'timelimit': 0,
   'submit': 1716736681.0,
   'start': 1716736681.0,
   'elapsed': 91.0,
   'planned': 0,
   'end': 1716736772.0,
   'planned_cpu': 0,
   'cpu_time': 182.0,
   'total_cpu': 10.045},
  5: {'job_id': '3470718',
   'job_name': 'singulari+',
   'account': 'empty',
   'parition': 'empty',
   'req_cpus': 2,
   'alloc_cpus': 2,
   'req_nodes': 1,
   'alloc_nodes': 1,
   'state': 'CANCELLED',
   'ave_cpu': 6.0,
   'ave_cpu_freq': 88320000.0,
   'ave_disk_read': 58540000.0,
   'ave_disk_write': 150000.0,
   'timelimit': 0,
   'submit': 1716736686.0,
   'start': 1716736686.0,
   'elapsed': 86.0,
   'planned': 0,
   'end': 1716736772.0,
   'planned_cpu': 0,
   'cpu_time': 172.0,
   'total_cpu': 6.047}}}
```
We can again use parquet to store these into for-air/ARTIFACTS/SACCT in the following way:
```
sacct_df = pd.DataFrame.from_dict(sacct_test['rows'], orient = 'index')
column_order = [
    'job-id',
    'job-name',
    'partition', 
    'req-cpus', 
    'alloc-cpus', 
    'req-nodes', 
    'alloc-nodes',
    'state',
    'ave-cpu-seconds',
    'ave-disk-read-bytes', 
    'ave-disk-write-bytes',
    'ave-pages',
    'ave-rss',
    'ave-vm-size',
    'consumed-energy-raw', 
    'timelimit-seconds', 
    'submit-time', 
    'start-time',
    'elapsed-seconds', 
    'planned-seconds', 
    'end-time', 
    'planned-cpu-seconds',
    'cpu-time-seconds', 
    'total-cpu-seconds'
]
sacct_df = sacct_df.reindex(columns = column_order)
```
```
def pandas_modify_dataframe(
    column_types: any,
    df: any
) -> any:
    for name in df.columns:
        wanted_type = column_types[name]
        if df[name].isna().any():
            if wanted_type == 'object':
                df[name] = df[name].fillna('null')
            if wanted_type == 'int64' or wanted_type == 'float64':
                df[name] = df[name].fillna(0)
    df = df.astype(column_types)
    return df
```
```
column_types = {
    'job-id': 'object',
    'job-name': 'object',
    'partition': 'object', 
    'req-cpus': 'int64',
    'alloc-cpus': 'int64', 
    'req-nodes': 'int64', 
    'alloc-nodes': 'int64',
    'state': 'object',
    'ave-cpu-seconds': 'float64',
    'ave-disk-read-bytes': 'int64',
    'ave-disk-write-bytes': 'int64',
    'ave-pages': 'int64',
    'ave-rss': 'int64',
    'ave-vm-size': 'int64',
    'consumed-energy-raw': 'int64', 
    'timelimit-seconds': 'float64', 
    'submit-time': 'int64', 
    'start-time': 'int64',
    'elapsed-seconds': 'float64', 
    'planned-seconds': 'float64', 
    'end-time': 'int64', 
    'planned-cpu-seconds': 'float64',
    'cpu-time-seconds': 'float64', 
    'total-cpu-seconds': 'float64',
}

modified_sacct_df = pandas_modify_dataframe(
    column_types = column_types,
    df = sacct_df
)
```
```
modified_sacct_df
```
```
OUT 67 PICTURE HERE
```
```
formatted_sacct_data = pyarrow_serialize_dataframe(
    dataframe = modified_sacct_df
)
```
```
file_name = set_formatted_user(user = 'user@example.com') + '-' + sacct_test['name'] + '.parquet'
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'forwarder',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'arti',
        'path-replacers': {
            'name': 'SACCT'
        },
        'path-names': [
            file_name
        ],
        'overwrite': True
    },
    object_data = formatted_sacct_data,
    object_metadata = {'version': 1}
)
```
```
User object bucket: mlch-for-air
Used object path: ARTIFACTS/SACCT/user-example-com-run-1.parquet
```
Now, if you go into https://pouta.csc.fi/, you should see this container and file appear by doing the following:
- Click object storage
- Click containers
- Click mlch-for-air
- Click ARTIFACTS
- Click SACCT

We can get the dataframe with the following way:
```
get_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'get',
        'bucket-target': 'forwarder',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'arti',
        'path-replacers': {
            'name': 'SACCT'
        },
        'path-names': [
            file_name
        ],
        'overwrite': True
    },
    object_data = None,
    object_metadata = None
)
```
```
User object bucket: mlch-for-air
Used object path: ARTIFACTS/SACCT/user-example-com-run-1.parquet
```
```
formatted_stored_sacct = pyarrow_deserialize_dataframe(serialized_dataframe = get_interaction_output[0])
stored_general_metadata = get_interaction_output[1]
stored_custom_metadata = get_interaction_output[2]
```
```
formatted_stored_sacct
```
OUT 73 HERE
### Logs
Logs objects provide accurate prints created by SLURM batch jobs. We define them with:
```
class Logs(BaseModel):
    name: str = Field(alias = 'name')
    rows: list[str] = Field(alias = 'rows')class Logs(BaseModel):
    name: str = Field(alias = 'name')
    rows: list[str] = Field(alias = 'rows')
```
```
def format_slurm_logs(
    file_path: str
) -> any:
    log_text = None
    with open(file_path, 'r') as f:
        log_text = f.readlines()
    
    row_list = []
    if not log_text is None:
        for line in log_text:
            filter_1 = line.replace('\n', '')
            filter_2 = filter_1.replace('\t', ' ')
            filter_3 = filter_2.replace('\x1b', ' ')
            if not filter_3.isspace() and 0 < len(filter_3):
                row_list.append(filter_3)
    return row_list 
```
```
puhti_logs = format_slurm_logs(
    file_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/puhti-logs.txt'
)
```
```
puhti_logs_test = {
    'name': 'puhti-run-1',
    'rows': puhti_logs
}

validation_puhti_logs = Logs(**puhti_logs_test)

validation_puhti_logs.model_dump()
```
```
{'name': 'puhti-run-1',
 'rows': ['Loaded modules:',
  'Currently Loaded Modules:',
  '  1) gcc/11.3.0                  3) openmpi/4.1.4       5) StdEnv',
  '  2) intel-oneapi-mkl/2022.1.0   4) csc-tools     (S)   6) pytorch/2.6',
  '  Where:',
  '   S:  Module is Sticky, requires --force to unload or purge',
  'Activating venv',
  'Venv active',
  'Installed packages',
  'Package                            Version',
  '---------------------------------- -------------------',
  'absl-py                            1.4.0',
  'accelerate                         1.6.0',
  'affine                             2.4.0',
  'aiohappyeyeballs                   2.6.1',
  'aiohttp                            3.11.16',
  'aiohttp-cors                       0.8.1',
  'aiosignal                          1.3.2',
  'airportsdata                       20250224',
  'alembic                            1.15.2',
  'annotated-types                    0.7.0',
  'ansicolors                         1.1.8',
  'anyio                              4.9.0',
  'apex                               0.1',
  'argon2-cffi                        23.1.0',
  'argon2-cffi-bindings               21.2.0',
  'array_record                       0.7.1',
  'arrow                              1.3.0',
  'assertpy                           1.1',
  'astor                              0.8.1',
  'asttokens                          3.0.0',
  'async-lru                          2.0.5',
  'attrs                              25.3.0',
  'audioread                          3.0.1',
  'babel                              2.17.0',
  'beautifulsoup4                     4.13.3',
  'bitsandbytes                       0.45.5',
  'blake3                             1.0.4',
  'bleach                             6.2.0',
  'blinker                            1.9.0',
  'blosc2                             3.3.0',
  'cachetools                         5.5.2',
  'certifi                            2025.1.31',
  'cffi                               1.17.1',
  'cftime                             1.6.4.post1',
  'charset-normalizer                 3.4.1',
  'click                              8.1.8',
  'click-plugins                      1.1.1',
  'cligj                              0.7.2',
  'cloudpickle                        3.1.1',
  'cmake                              3.31.6',
  'colorama                           0.4.6',
  'colorful                           0.5.6',
  'comm                               0.2.2',
  'compressed-tensors                 0.9.2',
  'contourpy                          1.3.1',
  'cupy-cuda12x                       13.4.1',
  'cycler                             0.12.1',
  'Cython                             3.0.12',
  'dash                               3.0.2',
  'dask                               2025.3.0',
  'dask-jobqueue                      0.9.0',
  'databricks-sdk                     0.49.0',
  'datasets                           3.5.0',
  'debtcollector                      3.0.0',
  'debugpy                            1.8.13',
  'decorator                          5.2.1',
  'deepspeed                          0.16.5',
  'deepspeed-kernels                  0.0.1.dev1698255861',
  'defusedxml                         0.7.1',
  'Deprecated                         1.2.18',
  'depyf                              0.18.0',
  'diffusers                          0.32.2',
  'dill                               0.3.8',
  'diskcache                          5.6.3',
  'distlib                            0.3.9',
  'distributed                        2025.3.0',
  'distro                             1.9.0',
  'dm-tree                            0.1.9',
  'dnspython                          2.7.0',
  'docker                             7.1.0',
  'docstring_parser                   0.16',
  'einops                             0.8.1',
  'email_validator                    2.2.0',
  'entrypoints                        0.4',
  'et_xmlfile                         2.0.0',
  'etils                              1.12.2',
  'evaluate                           0.4.3',
  'executing                          2.2.0',
  'faiss                              1.10.0',
  'fastapi                            0.115.12',
  'fastapi-cli                        0.0.7',
  'fastargs                           1.2.0',
  'fastjsonschema                     2.21.1',
  'fastrlock                          0.8.3',
  'ffcv                               1.0.2',
  'filelock                           3.18.0',
  'flash_attn                         2.7.4.post1',
  'Flask                              3.0.3',
  'fonttools                          4.57.0',
  'fqdn                               1.5.1',
  'frozenlist                         1.5.0',
  'fsspec                             2024.12.0',
  'fvcore                             0.1.5.post20221221',
  'gensim                             4.3.3',
  'geopandas                          1.0.1',
  'gguf                               0.10.0',
  'gitdb                              4.0.12',
  'GitPython                          3.1.44',
  'google-api-core                    2.24.2',
  'google-auth                        2.38.0',
  'googleapis-common-protos           1.69.2',
  'gpytorch                           1.14',
  'graphene                           3.4.3',
  'graphql-core                       3.2.6',
  'graphql-relay                      3.2.0',
  'graphviz                           0.20.3',
  'greenlet                           3.1.1',
  'grpcio                             1.71.0',
  'gunicorn                           23.0.0',
  'gym                                0.26.2',
  'gym-notices                        0.0.8',
  'h11                                0.14.0',
  'h5py                               3.13.0',
  'hf-xet                             1.0.2',
  'hjson                              3.1.0',
  'httpcore                           1.0.7',
  'httptools                          0.6.4',
  'httpx                              0.28.1',
  'huggingface-hub                    0.30.2',
  'idna                               3.10',
  'imageio                            2.37.0',
  'imbalanced-learn                   0.13.0',
  'immutabledict                      4.2.1',
  'importlib_metadata                 8.6.1',
  'importlib_resources                6.5.2',
  'iniconfig                          2.1.0',
  'interegular                        0.3.3',
  'iopath                             0.1.10',
  'ipykernel                          6.29.5',
  'ipython                            9.1.0',
  'ipython-genutils                   0.2.0',
  'ipython_pygments_lexers            1.1.1',
  'ipywidgets                         8.1.5',
  'iso8601                            2.1.0',
  'isoduration                        20.11.0',
  'itsdangerous                       2.2.0',
  'jaxtyping                          0.3.1',
  'jedi                               0.19.2',
  'Jinja2                             3.1.6',
  'jiter                              0.9.0',
  'joblib                             1.4.2',
  'json5                              0.12.0',
  'jsonpatch                          1.33',
  'jsonpointer                        3.0.0',
  'jsonschema                         4.23.0',
  'jsonschema-specifications          2024.10.1',
  'jupyter_client                     8.6.3',
  'jupyter_core                       5.7.2',
  'jupyter-events                     0.12.0',
  'jupyter-lsp                        2.2.5',
  'jupyter_server                     2.15.0',
  'jupyter-server-mathjax             0.2.6',
  'jupyter_server_proxy               4.4.0',
  'jupyter_server_terminals           0.5.3',
  'jupyterlab                         4.3.6',
  'jupyterlab-dash                    0.1.0a3',
  'jupyterlab_git                     0.51.1',
  'jupyterlab_pygments                0.3.0',
  'jupyterlab_server                  2.27.3',
  'jupyterlab_widgets                 3.0.13',
  'jupytext                           1.17.0',
  'kagglehub                          0.3.11',
  'keopscore                          2.2.3',
  'keras                              3.9.2',
  'keras-core                         0.1.7',
  'keras-cv                           0.9.0',
  'keystoneauth1                      5.10.0',
  'kiwisolver                         1.4.8',
  'lark                               1.2.2',
  'lazy_loader                        0.4',
  'librosa                            0.11.0',
  'lightning                          2.5.1',
  'lightning-utilities                0.14.3',
  'linear-operator                    0.6',
  'lit                                18.1.8',
  'llguidance                         0.7.13',
  'llvmlite                           0.44.0',
  'lm-format-enforcer                 0.10.11',
  'lmdb                               1.6.2',
  'locket                             1.0.0',
  'lxml                               5.3.2',
  'Mako                               1.3.9',
  'Markdown                           3.7',
  'markdown-it-py                     3.0.0',
  'MarkupSafe                         3.0.2',
  'matplotlib                         3.10.1',
  'matplotlib-inline                  0.1.7',
  'mdit-py-plugins                    0.4.2',
  'mdurl                              0.1.2',
  'mistral_common                     1.5.4',
  'mistune                            3.1.3',
  'ml_dtypes                          0.5.1',
  'mlflow                             2.21.3',
  'mlflow-skinny                      2.21.3',
  'mpi4py                             4.0.3',
  'mpmath                             1.3.0',
  'msgpack                            1.1.0',
  'msgspec                            0.19.0',
  'multidict                          6.3.2',
  'multiprocess                       0.70.16',
  'mysql-connector-python             9.2.0',
  'namex                              0.0.8',
  'nanobind                           2.6.1',
  'narwhals                           1.34.0',
  'nbclassic                          1.2.0',
  'nbclient                           0.10.2',
  'nbconvert                          7.16.6',
  'nbdime                             4.0.2',
  'nbformat                           5.10.4',
  'ndindex                            1.9.2',
  'nest-asyncio                       1.6.0',
  'netaddr                            1.3.0',
  'netCDF4                            1.7.2',
  'networkx                           3.4.2',
  'ninja                              1.11.1.4',
  'nltk                               3.9.1',
  'notebook                           7.3.3',
  'notebook_shim                      0.2.4',
  'numba                              0.61.0',
  'numexpr                            2.10.2',
  'numpy                              1.26.4',
  'nvidia-cublas-cu12                 12.4.5.8',
  'nvidia-cuda-cupti-cu12             12.4.127',
  'nvidia-cuda-nvrtc-cu12             12.4.127',
  'nvidia-cuda-runtime-cu12           12.4.127',
  'nvidia-cudnn-cu12                  9.1.0.70',
  'nvidia-cufft-cu12                  11.2.1.3',
  'nvidia-curand-cu12                 10.3.5.147',
  'nvidia-cusolver-cu12               11.6.1.9',
  'nvidia-cusparse-cu12               12.3.1.170',
  'nvidia-cusparselt-cu12             0.6.2',
  'nvidia-nccl-cu12                   2.21.5',
  'nvidia-nvjitlink-cu12              12.4.127',
  'nvidia-nvtx-cu12                   12.4.127',
  'odfpy                              1.4.1',
  'openai                             1.71.0',
  'opencensus                         0.11.4',
  'opencensus-context                 0.1.3',
  'opencv-python                      4.11.0.86',
  'opencv-python-headless             4.11.0.86',
  'openpyxl                           3.1.5',
  'opentelemetry-api                  1.31.1',
  'opentelemetry-sdk                  1.31.1',
  'opentelemetry-semantic-conventions 0.52b1',
  'optree                             0.15.0',
  'os-service-types                   1.7.0',
  'oslo.config                        9.7.1',
  'oslo.i18n                          6.5.1',
  'oslo.serialization                 5.7.0',
  'oslo.utils                         8.2.0',
  'outlines                           0.1.11',
  'outlines_core                      0.1.26',
  'overrides                          7.7.0',
  'packaging                          24.2',
  'pandas                             2.2.3',
  'pandocfilters                      1.5.1',
  'papermill                          2.6.0',
  'parso                              0.8.4',
  'partd                              1.4.2',
  'partial-json-parser                0.2.1.1.post5',
  'pbr                                6.1.1',
  'peft                               0.15.1',
  'pexpect                            4.9.0',
  'pillow                             11.1.0',
  'pip                                25.0.1',
  'platformdirs                       4.3.7',
  'plotly                             6.0.1',
  'pluggy                             1.5.0',
  'pooch                              1.8.2',
  'portalocker                        3.1.1',
  'prometheus_client                  0.21.1',
  'prometheus-fastapi-instrumentator  7.1.0',
  'promise                            2.3',
  'prompt_toolkit                     3.0.50',
  'propcache                          0.3.1',
  'proto-plus                         1.26.1',
  'protobuf                           3.20.3',
  'psutil                             7.0.0',
  'ptyprocess                         0.7.0',
  'pure_eval                          0.2.3',
  'py-cpuinfo                         9.0.0',
  'py-spy                             0.4.0',
  'pyarrow                            19.0.1',
  'pyasn1                             0.6.1',
  'pyasn1_modules                     0.4.2',
  'pybind11                           2.13.6',
  'pycodestyle                        2.13.0',
  'pycountry                          24.6.1',
  'pycparser                          2.22',
  'pydantic                           2.11.3',
  'pydantic_core                      2.33.1',
  'pydot                              3.0.4',
  'pyflakes                           3.3.2',
  'pyglet                             1.5.15',
  'Pygments                           2.19.1',
  'pykeops                            2.2.3',
  'pynndescent                        0.5.13',
  'pyogrio                            0.10.0',
  'pyparsing                          3.2.3',
  'pyproj                             3.7.1',
  'pyprojroot                         0.3.0',
  'pysqlite3                          0.5.4',
  'pytest                             8.3.5',
  'python-dateutil                    2.9.0.post0',
  'python-dotenv                      1.1.0',
  'python-json-logger                 3.3.0',
  'python-keystoneclient              5.6.0',
  'python-multipart                   0.0.20',
  'python-swiftclient                 4.7.0',
  'pytorch-lightning                  2.5.1',
  'pytorch-pfn-extras                 0.8.2',
  'pytz                               2025.2',
  'PyYAML                             6.0.2',
  'pyzmq                              26.4.0',
  'rasterio                           1.4.3',
  'ray                                2.43.0',
  'referencing                        0.36.2',
  'regex                              2024.11.6',
  'requests                           2.32.3',
  'retrying                           1.3.4',
  'rfc3339-validator                  0.1.4',
  'rfc3986                            2.0.0',
  'rfc3986-validator                  0.1.1',
  'rich                               14.0.0',
  'rich-toolkit                       0.14.1',
  'rpds-py                            0.24.0',
  'rsa                                4.9',
  'safetensors                        0.5.3',
  'scikit-image                       0.25.2',
  'scikit-learn                       1.6.1',
  'scipy                              1.13.1',
  'seaborn                            0.13.2',
  'Send2Trash                         1.8.3',
  'sentencepiece                      0.2.0',
  'setuptools                         78.1.0',
  'shapely                            2.1.0',
  'shellingham                        1.5.4',
  'simpervisor                        1.0.0',
  'simple-parsing                     0.1.7',
  'six                                1.17.0',
  'sklearn-compat                     0.1.3',
  'smart-open                         7.1.0',
  'smmap                              5.0.2',
  'sniffio                            1.3.1',
  'sortedcontainers                   2.4.0',
  'soundfile                          0.13.1',
  'soupsieve                          2.6',
  'soxr                               0.5.0.post1',
  'SQLAlchemy                         2.0.40',
  'sqlparse                           0.5.3',
  'stack-data                         0.6.3',
  'starlette                          0.46.1',
  'stevedore                          5.4.1',
  'sympy                              1.13.1',
  'tables                             3.10.2',
  'tabulate                           0.9.0',
  'tblib                              3.1.0',
  'tenacity                           9.1.2',
  'tensorboard                        2.19.0',
  'tensorboard-data-server            0.7.2',
  'tensorboardX                       2.6.2.2',
  'tensorflow-datasets                4.9.8',
  'tensorflow-metadata                1.14.0',
  'termcolor                          3.0.1',
  'terminado                          0.18.1',
  'terminaltables                     3.1.10',
  'threadpoolctl                      3.6.0',
  'tifffile                           2025.3.30',
  'tiktoken                           0.9.0',
  'timm                               1.0.15',
  'tinycss2                           1.4.0',
  'tokenizers                         0.21.1',
  'toml                               0.10.2',
  'toolz                              1.0.0',
  'torch                              2.6.0+cu124',
  'torch-geometric                    2.6.1',
  'torch-tb-profiler                  0.4.3',
  'torchaudio                         2.6.0+cu124',
  'torchinfo                          1.8.0',
  'torchmetrics                       1.7.1',
  'torchvision                        0.21.0+cu124',
  'tornado                            6.4.2',
  'tqdm                               4.67.1',
  'traitlets                          5.14.3',
  'transformers                       4.51.1',
  'triton                             3.2.0',
  'trl                                0.16.1',
  'typer                              0.15.2',
  'types-python-dateutil              2.9.0.20241206',
  'typing_extensions                  4.13.1',
  'typing-inspection                  0.4.0',
  'tzdata                             2025.2',
  'umap-learn                         0.5.7',
  'uri-template                       1.3.0',
  'urllib3                            2.3.0',
  'uvicorn                            0.34.0',
  'uvloop                             0.21.0',
  'virtualenv                         20.30.0',
  'visdom                             0.2.4',
  'vllm                               0.8.3',
  'wadler_lindig                      0.1.4',
  'watchfiles                         1.0.5',
  'wcwidth                            0.2.13',
  'webcolors                          24.11.1',
  'webencodings                       0.5.1',
  'websocket-client                   1.8.0',
  'websockets                         15.0.1',
  'Werkzeug                           3.0.6',
  'wheel                              0.45.1',
  'widgetsnbextension                 4.0.13',
  'wrapt                              1.17.2',
  'xformers                           0.0.29.post2',
  'xgboost                            3.0.0',
  'xgrammar                           0.1.17',
  'xlwt                               1.3.0',
  'xxhash                             3.5.0',
  'yacs                               0.1.8',
  'yarl                               1.19.0',
  'zict                               3.0.0',
  'zipp                               3.21.0',
  'Packages listed',
  'Setting connection variables',
  'Setting Ray variables',
  'Setting up Ray head',
  'IP Head: ():8265',
  'Starting HEAD at r02c17',
  'Setting up SSH tunnel',
  'Reverse port forward running',
  'Setting up Ray workers',
  'Starting WORKER 1 at r02c18',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  'connect_to () port 8280: failed.',
  '2025-04-18 11:38:00,439 - INFO - Note: NumExpr detected 40 cores but "NUMEXPR_MAX_THREADS" not set, so enforcing safe limit of 16.',
  '2025-04-18 11:38:00,439 - INFO - NumExpr defaulting to 16 threads.',
  '2025-04-18 11:38:02,292 INFO usage_lib.py:467 -- Usage stats collection is enabled by default without user confirmation because this terminal is detected to be non-interactive. To disable this, add `--disable-usage-stats` to the command that starts the cluster, or run the following command: `ray disable-usage-stats` before starting the cluster. See https://docs.ray.io/en/master/cluster/usage-stats.html for more details.',
  '2025-04-18 11:38:02,292 INFO scripts.py:865 -- Local node IP: ()',
  '2025-04-18 11:38:06,106 SUCC scripts.py:902 -- --------------------',
  '2025-04-18 11:38:06,106 SUCC scripts.py:903 -- Ray runtime started.',
  '2025-04-18 11:38:06,106 SUCC scripts.py:904 -- --------------------',
  '2025-04-18 11:38:06,107 INFO scripts.py:906 -- Next steps',
  '2025-04-18 11:38:06,107 INFO scripts.py:909 -- To add another node to this Ray cluster, run',
  "2025-04-18 11:38:06,107 INFO scripts.py:912 --   ray start --address='():8265'",
  '2025-04-18 11:38:06,107 INFO scripts.py:921 -- To connect to this Ray cluster:',
  '2025-04-18 11:38:06,107 INFO scripts.py:923 -- import ray',
  "2025-04-18 11:38:06,107 INFO scripts.py:924 -- ray.init(_node_ip_address='()')",
  '2025-04-18 11:38:06,107 INFO scripts.py:936 -- To submit a Ray job using the Ray Jobs CLI:',
  "2025-04-18 11:38:06,107 INFO scripts.py:937 --   RAY_ADDRESS='http://():8280' ray job submit --working-dir . -- python my_script.py",
  '2025-04-18 11:38:06,107 INFO scripts.py:946 -- See https://docs.ray.io/en/latest/cluster/running-applications/job-submission/index.html ',
  '2025-04-18 11:38:06,107 INFO scripts.py:950 -- for more information on submitting Ray jobs to the Ray cluster.',
  '2025-04-18 11:38:06,107 INFO scripts.py:955 -- To terminate the Ray runtime, run',
  '2025-04-18 11:38:06,107 INFO scripts.py:956 --   ray stop',
  '2025-04-18 11:38:06,107 INFO scripts.py:959 -- To view the status of the cluster, use',
  '2025-04-18 11:38:06,107 INFO scripts.py:960 --   ray status',
  '2025-04-18 11:38:06,107 INFO scripts.py:964 -- To monitor and debug Ray, view the dashboard at ',
  '2025-04-18 11:38:06,107 INFO scripts.py:965 --   ():8280',
  '2025-04-18 11:38:06,107 INFO scripts.py:972 -- If connection to the dashboard fails, check your firewall settings and network configuration.',
  '2025-04-18 11:38:06,107 INFO scripts.py:1076 -- --block',
  '2025-04-18 11:38:06,107 INFO scripts.py:1077 -- This command will now block forever until terminated by a signal.',
  '2025-04-18 11:38:06,107 INFO scripts.py:1080 -- Running subprocesses are monitored and a message will be printed if any of them terminate unexpectedly. Subprocesses exit with SIGTERM will be treated as graceful, thus NOT reported.',
  '2025-04-18 11:38:13,966 - INFO - Note: NumExpr detected 40 cores but "NUMEXPR_MAX_THREADS" not set, so enforcing safe limit of 16.',
  '2025-04-18 11:38:13,966 - INFO - NumExpr defaulting to 16 threads.',
  '[2025-04-18 11:38:15,762 W 3878629 3878629] global_state_accessor.cc:429: Retrying to get node with node ID 9a2ab1623d258f5f5f7e0012764e2a28556643cce30b12d87174244b',
  '2025-04-18 11:38:15,429 INFO scripts.py:1047 -- Local node IP: ()',
  '2025-04-18 11:38:16,777 SUCC scripts.py:1063 -- --------------------',
  '2025-04-18 11:38:16,777 SUCC scripts.py:1064 -- Ray runtime started.',
  '2025-04-18 11:38:16,777 SUCC scripts.py:1065 -- --------------------',
  '2025-04-18 11:38:16,777 INFO scripts.py:1067 -- To terminate the Ray runtime, run',
  '2025-04-18 11:38:16,777 INFO scripts.py:1068 --   ray stop',
  '2025-04-18 11:38:16,778 INFO scripts.py:1076 -- --block',
  '2025-04-18 11:38:16,778 INFO scripts.py:1077 -- This command will now block forever until terminated by a signal.',
  '2025-04-18 11:38:16,778 INFO scripts.py:1080 -- Running subprocesses are monitored and a message will be printed if any of them terminate unexpectedly. Subprocesses exit with SIGTERM will be treated as graceful, thus NOT reported.',
  'srun: Job step aborted: Waiting up to 62 seconds for job step to finish.',
  'srun: Job step aborted: Waiting up to 62 seconds for job step to finish.',
  'slurmstepd: error: *** STEP 27587022.2 ON r02c18 CANCELLED AT 2025-04-18T11:39:08 ***',
  'slurmstepd: error: *** JOB 27587022 ON r02c17 CANCELLED AT 2025-04-18T11:39:08 ***',
  'slurmstepd: error: *** STEP 27587022.1 ON r02c17 CANCELLED AT 2025-04-18T11:39:08 ***']}
```
```
puhti_file_name = puhti_logs_test['name'] + '.parquet'
puhti_logs_df = pd.DataFrame({'rows': puhti_logs_test['rows']})
print(puhti_file_name)
```
```
puhti-run-1.parquet
```
```
puhti_logs_df
```
```

rows
0	Loaded modules:
1	Currently Loaded Modules:
2	1) gcc/11.3.0 3) openmpi/4....
3	2) intel-oneapi-mkl/2022.1.0 4) csc-tools ...
4	Where:
...	...
492	srun: Job step aborted: Waiting up to 62 secon...
493	srun: Job step aborted: Waiting up to 62 secon...
494	slurmstepd: error: *** STEP 27587022.2 ON r02c...
495	slurmstepd: error: *** JOB 27587022 ON r02c17 ...
496	slurmstepd: error: *** STEP 27587022.1 ON r02c...

497 rows × 1 columns
```
```
formatted_puhti_logs_data = pyarrow_serialize_dataframe(
    dataframe = puhti_logs_df
)
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': puhti_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = formatted_puhti_logs_data,
    object_metadata = {'version': 1}
)
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': puhti_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = formatted_puhti_logs_data,
    object_metadata = {'version': 1}
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: LOGS/puhti-run-1.parquet
```
```
mahti_logs = format_slurm_logs(
    file_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/mahti-logs.txt'
)
```
```
mahti_logs_test = {
    'name': 'mahti-run-1',
    'rows': mahti_logs
}

validation_mahti_logs = Logs(**mahti_logs_test)

validation_mahti_logs.model_dump()
```
```
{'name': 'mahti-run-1',
 'rows': ['Loaded modules:',
  'Currently Loaded Modules:',
  '  1) gcc/11.2.0      3) openblas/0.3.18-omp       5) StdEnv',
  '  2) openmpi/4.1.2   4) csc-tools           (S)   6) pytorch/2.6',
  '  Where:',
  '   S:  Module is Sticky, requires --force to unload or purge',
  'Activating venv',
  'Venv active',
  'Installed packages',
  'Package                            Version',
  '---------------------------------- -------------------',
  'absl-py                            1.4.0',
  'accelerate                         1.6.0',
  'affine                             2.4.0',
  'aiohappyeyeballs                   2.6.1',
  'aiohttp                            3.11.16',
  'aiohttp-cors                       0.8.1',
  'aiosignal                          1.3.2',
  'airportsdata                       20250224',
  'alembic                            1.15.2',
  'annotated-types                    0.7.0',
  'ansicolors                         1.1.8',
  'anyio                              4.9.0',
  'apex                               0.1',
  'argon2-cffi                        23.1.0',
  'argon2-cffi-bindings               21.2.0',
  'array_record                       0.7.1',
  'arrow                              1.3.0',
  'assertpy                           1.1',
  'astor                              0.8.1',
  'asttokens                          3.0.0',
  'async-lru                          2.0.5',
  'attrs                              25.3.0',
  'audioread                          3.0.1',
  'babel                              2.17.0',
  'beautifulsoup4                     4.13.3',
  'bitsandbytes                       0.45.5',
  'blake3                             1.0.4',
  'bleach                             6.2.0',
  'blinker                            1.9.0',
  'blosc2                             3.3.0',
  'cachetools                         5.5.2',
  'certifi                            2025.1.31',
  'cffi                               1.17.1',
  'cftime                             1.6.4.post1',
  'charset-normalizer                 3.4.1',
  'click                              8.1.8',
  'click-plugins                      1.1.1',
  'cligj                              0.7.2',
  'cloudpickle                        3.1.1',
  'cmake                              3.31.6',
  'colorama                           0.4.6',
  'colorful                           0.5.6',
  'comm                               0.2.2',
  'compressed-tensors                 0.9.2',
  'contourpy                          1.3.1',
  'cupy-cuda12x                       13.4.1',
  'cycler                             0.12.1',
  'Cython                             3.0.12',
  'dash                               3.0.2',
  'dask                               2025.3.0',
  'dask-jobqueue                      0.9.0',
  'databricks-sdk                     0.49.0',
  'datasets                           3.5.0',
  'debtcollector                      3.0.0',
  'debugpy                            1.8.13',
  'decorator                          5.2.1',
  'deepspeed                          0.16.5',
  'deepspeed-kernels                  0.0.1.dev1698255861',
  'defusedxml                         0.7.1',
  'Deprecated                         1.2.18',
  'depyf                              0.18.0',
  'diffusers                          0.32.2',
  'dill                               0.3.8',
  'diskcache                          5.6.3',
  'distlib                            0.3.9',
  'distributed                        2025.3.0',
  'distro                             1.9.0',
  'dm-tree                            0.1.9',
  'dnspython                          2.7.0',
  'docker                             7.1.0',
  'docstring_parser                   0.16',
  'einops                             0.8.1',
  'email_validator                    2.2.0',
  'entrypoints                        0.4',
  'et_xmlfile                         2.0.0',
  'etils                              1.12.2',
  'evaluate                           0.4.3',
  'executing                          2.2.0',
  'faiss                              1.10.0',
  'fastapi                            0.115.12',
  'fastapi-cli                        0.0.7',
  'fastargs                           1.2.0',
  'fastjsonschema                     2.21.1',
  'fastrlock                          0.8.3',
  'ffcv                               1.0.2',
  'filelock                           3.18.0',
  'flash_attn                         2.7.4.post1',
  'Flask                              3.0.3',
  'fonttools                          4.57.0',
  'fqdn                               1.5.1',
  'frozenlist                         1.5.0',
  'fsspec                             2024.12.0',
  'fvcore                             0.1.5.post20221221',
  'gensim                             4.3.3',
  'geopandas                          1.0.1',
  'gguf                               0.10.0',
  'gitdb                              4.0.12',
  'GitPython                          3.1.44',
  'google-api-core                    2.24.2',
  'google-auth                        2.38.0',
  'googleapis-common-protos           1.69.2',
  'gpytorch                           1.14',
  'graphene                           3.4.3',
  'graphql-core                       3.2.6',
  'graphql-relay                      3.2.0',
  'graphviz                           0.20.3',
  'greenlet                           3.1.1',
  'grpcio                             1.71.0',
  'gunicorn                           23.0.0',
  'gym                                0.26.2',
  'gym-notices                        0.0.8',
  'h11                                0.14.0',
  'h5py                               3.13.0',
  'hf-xet                             1.0.2',
  'hjson                              3.1.0',
  'httpcore                           1.0.7',
  'httptools                          0.6.4',
  'httpx                              0.28.1',
  'huggingface-hub                    0.30.2',
  'idna                               3.10',
  'imageio                            2.37.0',
  'imbalanced-learn                   0.13.0',
  'immutabledict                      4.2.1',
  'importlib_metadata                 8.6.1',
  'importlib_resources                6.5.2',
  'iniconfig                          2.1.0',
  'interegular                        0.3.3',
  'iopath                             0.1.10',
  'ipykernel                          6.29.5',
  'ipython                            9.1.0',
  'ipython-genutils                   0.2.0',
  'ipython_pygments_lexers            1.1.1',
  'ipywidgets                         8.1.5',
  'iso8601                            2.1.0',
  'isoduration                        20.11.0',
  'itsdangerous                       2.2.0',
  'jaxtyping                          0.3.1',
  'jedi                               0.19.2',
  'Jinja2                             3.1.6',
  'jiter                              0.9.0',
  'joblib                             1.4.2',
  'json5                              0.12.0',
  'jsonpatch                          1.33',
  'jsonpointer                        3.0.0',
  'jsonschema                         4.23.0',
  'jsonschema-specifications          2024.10.1',
  'jupyter_client                     8.6.3',
  'jupyter_core                       5.7.2',
  'jupyter-events                     0.12.0',
  'jupyter-lsp                        2.2.5',
  'jupyter_server                     2.15.0',
  'jupyter-server-mathjax             0.2.6',
  'jupyter_server_proxy               4.4.0',
  'jupyter_server_terminals           0.5.3',
  'jupyterlab                         4.3.6',
  'jupyterlab-dash                    0.1.0a3',
  'jupyterlab_git                     0.51.1',
  'jupyterlab_pygments                0.3.0',
  'jupyterlab_server                  2.27.3',
  'jupyterlab_widgets                 3.0.13',
  'jupytext                           1.17.0',
  'kagglehub                          0.3.11',
  'keopscore                          2.2.3',
  'keras                              3.9.2',
  'keras-core                         0.1.7',
  'keras-cv                           0.9.0',
  'keystoneauth1                      5.10.0',
  'kiwisolver                         1.4.8',
  'lark                               1.2.2',
  'lazy_loader                        0.4',
  'librosa                            0.11.0',
  'lightning                          2.5.1',
  'lightning-utilities                0.14.3',
  'linear-operator                    0.6',
  'lit                                18.1.8',
  'llguidance                         0.7.13',
  'llvmlite                           0.44.0',
  'lm-format-enforcer                 0.10.11',
  'lmdb                               1.6.2',
  'locket                             1.0.0',
  'lxml                               5.3.2',
  'Mako                               1.3.9',
  'Markdown                           3.7',
  'markdown-it-py                     3.0.0',
  'MarkupSafe                         3.0.2',
  'matplotlib                         3.10.1',
  'matplotlib-inline                  0.1.7',
  'mdit-py-plugins                    0.4.2',
  'mdurl                              0.1.2',
  'mistral_common                     1.5.4',
  'mistune                            3.1.3',
  'ml_dtypes                          0.5.1',
  'mlflow                             2.21.3',
  'mlflow-skinny                      2.21.3',
  'mpi4py                             4.0.3',
  'mpmath                             1.3.0',
  'msgpack                            1.1.0',
  'msgspec                            0.19.0',
  'multidict                          6.3.2',
  'multiprocess                       0.70.16',
  'mysql-connector-python             9.2.0',
  'namex                              0.0.8',
  'nanobind                           2.6.1',
  'narwhals                           1.34.0',
  'nbclassic                          1.2.0',
  'nbclient                           0.10.2',
  'nbconvert                          7.16.6',
  'nbdime                             4.0.2',
  'nbformat                           5.10.4',
  'ndindex                            1.9.2',
  'nest-asyncio                       1.6.0',
  'netaddr                            1.3.0',
  'netCDF4                            1.7.2',
  'networkx                           3.4.2',
  'ninja                              1.11.1.4',
  'nltk                               3.9.1',
  'notebook                           7.3.3',
  'notebook_shim                      0.2.4',
  'numba                              0.61.0',
  'numexpr                            2.10.2',
  'numpy                              1.26.4',
  'nvidia-cublas-cu12                 12.4.5.8',
  'nvidia-cuda-cupti-cu12             12.4.127',
  'nvidia-cuda-nvrtc-cu12             12.4.127',
  'nvidia-cuda-runtime-cu12           12.4.127',
  'nvidia-cudnn-cu12                  9.1.0.70',
  'nvidia-cufft-cu12                  11.2.1.3',
  'nvidia-curand-cu12                 10.3.5.147',
  'nvidia-cusolver-cu12               11.6.1.9',
  'nvidia-cusparse-cu12               12.3.1.170',
  'nvidia-cusparselt-cu12             0.6.2',
  'nvidia-nccl-cu12                   2.21.5',
  'nvidia-nvjitlink-cu12              12.4.127',
  'nvidia-nvtx-cu12                   12.4.127',
  'odfpy                              1.4.1',
  'openai                             1.71.0',
  'opencensus                         0.11.4',
  'opencensus-context                 0.1.3',
  'opencv-python                      4.11.0.86',
  'opencv-python-headless             4.11.0.86',
  'openpyxl                           3.1.5',
  'opentelemetry-api                  1.31.1',
  'opentelemetry-sdk                  1.31.1',
  'opentelemetry-semantic-conventions 0.52b1',
  'optree                             0.15.0',
  'os-service-types                   1.7.0',
  'oslo.config                        9.7.1',
  'oslo.i18n                          6.5.1',
  'oslo.serialization                 5.7.0',
  'oslo.utils                         8.2.0',
  'outlines                           0.1.11',
  'outlines_core                      0.1.26',
  'overrides                          7.7.0',
  'packaging                          24.2',
  'pandas                             2.2.3',
  'pandocfilters                      1.5.1',
  'papermill                          2.6.0',
  'parso                              0.8.4',
  'partd                              1.4.2',
  'partial-json-parser                0.2.1.1.post5',
  'pbr                                6.1.1',
  'peft                               0.15.1',
  'pexpect                            4.9.0',
  'pillow                             11.1.0',
  'pip                                25.0.1',
  'platformdirs                       4.3.7',
  'plotly                             6.0.1',
  'pluggy                             1.5.0',
  'pooch                              1.8.2',
  'portalocker                        3.1.1',
  'prometheus_client                  0.21.1',
  'prometheus-fastapi-instrumentator  7.1.0',
  'promise                            2.3',
  'prompt_toolkit                     3.0.50',
  'propcache                          0.3.1',
  'proto-plus                         1.26.1',
  'protobuf                           3.20.3',
  'psutil                             7.0.0',
  'ptyprocess                         0.7.0',
  'pure_eval                          0.2.3',
  'py-cpuinfo                         9.0.0',
  'py-spy                             0.4.0',
  'pyarrow                            19.0.1',
  'pyasn1                             0.6.1',
  'pyasn1_modules                     0.4.2',
  'pybind11                           2.13.6',
  'pycodestyle                        2.13.0',
  'pycountry                          24.6.1',
  'pycparser                          2.22',
  'pydantic                           2.11.3',
  'pydantic_core                      2.33.1',
  'pydot                              3.0.4',
  'pyflakes                           3.3.2',
  'pyglet                             1.5.15',
  'Pygments                           2.19.1',
  'pykeops                            2.2.3',
  'pynndescent                        0.5.13',
  'pyogrio                            0.10.0',
  'pyparsing                          3.2.3',
  'pyproj                             3.7.1',
  'pyprojroot                         0.3.0',
  'pysqlite3                          0.5.4',
  'pytest                             8.3.5',
  'python-dateutil                    2.9.0.post0',
  'python-dotenv                      1.1.0',
  'python-json-logger                 3.3.0',
  'python-keystoneclient              5.6.0',
  'python-multipart                   0.0.20',
  'python-swiftclient                 4.7.0',
  'pytorch-lightning                  2.5.1',
  'pytorch-pfn-extras                 0.8.2',
  'pytz                               2025.2',
  'PyYAML                             6.0.2',
  'pyzmq                              26.4.0',
  'rasterio                           1.4.3',
  'ray                                2.43.0',
  'referencing                        0.36.2',
  'regex                              2024.11.6',
  'requests                           2.32.3',
  'retrying                           1.3.4',
  'rfc3339-validator                  0.1.4',
  'rfc3986                            2.0.0',
  'rfc3986-validator                  0.1.1',
  'rich                               14.0.0',
  'rich-toolkit                       0.14.1',
  'rpds-py                            0.24.0',
  'rsa                                4.9',
  'safetensors                        0.5.3',
  'scikit-image                       0.25.2',
  'scikit-learn                       1.6.1',
  'scipy                              1.13.1',
  'seaborn                            0.13.2',
  'Send2Trash                         1.8.3',
  'sentencepiece                      0.2.0',
  'setuptools                         78.1.0',
  'shapely                            2.1.0',
  'shellingham                        1.5.4',
  'simpervisor                        1.0.0',
  'simple-parsing                     0.1.7',
  'six                                1.17.0',
  'sklearn-compat                     0.1.3',
  'smart-open                         7.1.0',
  'smmap                              5.0.2',
  'sniffio                            1.3.1',
  'sortedcontainers                   2.4.0',
  'soundfile                          0.13.1',
  'soupsieve                          2.6',
  'soxr                               0.5.0.post1',
  'SQLAlchemy                         2.0.40',
  'sqlparse                           0.5.3',
  'stack-data                         0.6.3',
  'starlette                          0.46.1',
  'stevedore                          5.4.1',
  'sympy                              1.13.1',
  'tables                             3.10.2',
  'tabulate                           0.9.0',
  'tblib                              3.1.0',
  'tenacity                           9.1.2',
  'tensorboard                        2.19.0',
  'tensorboard-data-server            0.7.2',
  'tensorboardX                       2.6.2.2',
  'tensorflow-datasets                4.9.8',
  'tensorflow-metadata                1.14.0',
  'termcolor                          3.0.1',
  'terminado                          0.18.1',
  'terminaltables                     3.1.10',
  'threadpoolctl                      3.6.0',
  'tifffile                           2025.3.30',
  'tiktoken                           0.9.0',
  'timm                               1.0.15',
  'tinycss2                           1.4.0',
  'tokenizers                         0.21.1',
  'toml                               0.10.2',
  'toolz                              1.0.0',
  'torch                              2.6.0+cu124',
  'torch-geometric                    2.6.1',
  'torch-tb-profiler                  0.4.3',
  'torchaudio                         2.6.0+cu124',
  'torchinfo                          1.8.0',
  'torchmetrics                       1.7.1',
  'torchvision                        0.21.0+cu124',
  'tornado                            6.4.2',
  'tqdm                               4.67.1',
  'traitlets                          5.14.3',
  'transformers                       4.51.1',
  'triton                             3.2.0',
  'trl                                0.16.1',
  'typer                              0.15.2',
  'types-python-dateutil              2.9.0.20241206',
  'typing_extensions                  4.13.1',
  'typing-inspection                  0.4.0',
  'tzdata                             2025.2',
  'umap-learn                         0.5.7',
  'uri-template                       1.3.0',
  'urllib3                            2.3.0',
  'uvicorn                            0.34.0',
  'uvloop                             0.21.0',
  'virtualenv                         20.30.0',
  'visdom                             0.2.4',
  'vllm                               0.8.3',
  'wadler_lindig                      0.1.4',
  'watchfiles                         1.0.5',
  'wcwidth                            0.2.13',
  'webcolors                          24.11.1',
  'webencodings                       0.5.1',
  'websocket-client                   1.8.0',
  'websockets                         15.0.1',
  'Werkzeug                           3.0.6',
  'wheel                              0.45.1',
  'widgetsnbextension                 4.0.13',
  'wrapt                              1.17.2',
  'xformers                           0.0.29.post2',
  'xgboost                            3.0.0',
  'xgrammar                           0.1.17',
  'xlwt                               1.3.0',
  'xxhash                             3.5.0',
  'yacs                               0.1.8',
  'yarl                               1.19.0',
  'zict                               3.0.0',
  'zipp                               3.21.0',
  'Packages listed',
  'Setting connection variables',
  'Setting Ray variables',
  'Setting up Ray head',
  'IP Head: ():8265',
  'Starting HEAD at c3262',
  'Setting up SSH tunnel',
  'Reverse port forward running',
  'Setting up Ray workers',
  'Starting WORKER 1 at c3263',
  '2025-04-18 12:38:41,084 - INFO - Note: detected 256 virtual cores but NumExpr set to maximum of 64, check "NUMEXPR_MAX_THREADS" environment variable.',
  '2025-04-18 12:38:41,084 - INFO - Note: NumExpr detected 256 cores but "NUMEXPR_MAX_THREADS" not set, so enforcing safe limit of 16.',
  '2025-04-18 12:38:41,084 - INFO - NumExpr defaulting to 16 threads.',
  '2025-04-18 12:38:44,600 - INFO - Note: detected 256 virtual cores but NumExpr set to maximum of 64, check "NUMEXPR_MAX_THREADS" environment variable.',
  '2025-04-18 12:38:44,600 - INFO - Note: NumExpr detected 256 cores but "NUMEXPR_MAX_THREADS" not set, so enforcing safe limit of 16.',
  '2025-04-18 12:38:44,600 - INFO - NumExpr defaulting to 16 threads.',
  '[2025-04-18 12:38:45,983 W 1172662 1172662] global_state_accessor.cc:429: Retrying to get node with node ID 780537062d6cf92ec0ce5c7ad9a424ed448623055e51a52a5d0d9f01',
  '2025-04-18 12:38:41,971 INFO usage_lib.py:467 -- Usage stats collection is enabled by default without user confirmation because this terminal is detected to be non-interactive. To disable this, add `--disable-usage-stats` to the command that starts the cluster, or run the following command: `ray disable-usage-stats` before starting the cluster. See https://docs.ray.io/en/master/cluster/usage-stats.html for more details.',
  '2025-04-18 12:38:41,971 INFO scripts.py:865 -- Local node IP: ()',
  '2025-04-18 12:38:46,073 SUCC scripts.py:902 -- --------------------',
  '2025-04-18 12:38:46,073 SUCC scripts.py:903 -- Ray runtime started.',
  '2025-04-18 12:38:46,073 SUCC scripts.py:904 -- --------------------',
  '2025-04-18 12:38:46,073 INFO scripts.py:906 -- Next steps',
  '2025-04-18 12:38:46,073 INFO scripts.py:909 -- To add another node to this Ray cluster, run',
  "2025-04-18 12:38:46,073 INFO scripts.py:912 --   ray start --address='():8265'",
  '2025-04-18 12:38:46,073 INFO scripts.py:921 -- To connect to this Ray cluster:',
  '2025-04-18 12:38:46,073 INFO scripts.py:923 -- import ray',
  "2025-04-18 12:38:46,073 INFO scripts.py:924 -- ray.init(_node_ip_address='()')",
  '2025-04-18 12:38:46,073 INFO scripts.py:936 -- To submit a Ray job using the Ray Jobs CLI:',
  "2025-04-18 12:38:46,073 INFO scripts.py:937 --   RAY_ADDRESS='http://():8280' ray job submit --working-dir . -- python my_script.py",
  '2025-04-18 12:38:46,073 INFO scripts.py:946 -- See https://docs.ray.io/en/latest/cluster/running-applications/job-submission/index.html ',
  '2025-04-18 12:38:46,073 INFO scripts.py:950 -- for more information on submitting Ray jobs to the Ray cluster.',
  '2025-04-18 12:38:46,073 INFO scripts.py:955 -- To terminate the Ray runtime, run',
  '2025-04-18 12:38:46,073 INFO scripts.py:956 --   ray stop',
  '2025-04-18 12:38:46,073 INFO scripts.py:959 -- To view the status of the cluster, use',
  '2025-04-18 12:38:46,073 INFO scripts.py:960 --   ray status',
  '2025-04-18 12:38:46,073 INFO scripts.py:964 -- To monitor and debug Ray, view the dashboard at ',
  '2025-04-18 12:38:46,073 INFO scripts.py:965 --   ():8280',
  '2025-04-18 12:38:46,073 INFO scripts.py:972 -- If connection to the dashboard fails, check your firewall settings and network configuration.',
  '2025-04-18 12:38:46,074 INFO scripts.py:1076 -- --block',
  '2025-04-18 12:38:46,074 INFO scripts.py:1077 -- This command will now block forever until terminated by a signal.',
  '2025-04-18 12:38:46,074 INFO scripts.py:1080 -- Running subprocesses are monitored and a message will be printed if any of them terminate unexpectedly. Subprocesses exit with SIGTERM will be treated as graceful, thus NOT reported.',
  '2025-04-18 12:38:45,253 INFO scripts.py:1047 -- Local node IP: ()',
  '2025-04-18 12:38:46,998 SUCC scripts.py:1063 -- --------------------',
  '2025-04-18 12:38:46,998 SUCC scripts.py:1064 -- Ray runtime started.',
  '2025-04-18 12:38:46,998 SUCC scripts.py:1065 -- --------------------',
  '2025-04-18 12:38:46,998 INFO scripts.py:1067 -- To terminate the Ray runtime, run',
  '2025-04-18 12:38:46,998 INFO scripts.py:1068 --   ray stop',
  '2025-04-18 12:38:46,998 INFO scripts.py:1076 -- --block',
  '2025-04-18 12:38:46,998 INFO scripts.py:1077 -- This command will now block forever until terminated by a signal.',
  '2025-04-18 12:38:46,998 INFO scripts.py:1080 -- Running subprocesses are monitored and a message will be printed if any of them terminate unexpectedly. Subprocesses exit with SIGTERM will be treated as graceful, thus NOT reported.',
  'srun: Job step aborted: Waiting up to 62 seconds for job step to finish.',
  'srun: Job step aborted: Waiting up to 62 seconds for job step to finish.',
  'slurmstepd: error: *** STEP 4393900.2 ON c3263 CANCELLED AT 2025-04-18T12:39:37 ***',
  'slurmstepd: error: *** STEP 4393900.1 ON c3262 CANCELLED AT 2025-04-18T12:39:37 ***',
  'slurmstepd: error: *** JOB 4393900 ON c3262 CANCELLED AT 2025-04-18T12:39:37 ***']}
```
```
mahti_file_name = mahti_logs_test['name'] + '.parquet'
mahti_logs_df = pd.DataFrame({'rows': mahti_logs_test['rows']})
print(mahti_file_name)
```
```
mahti-run-1.parquet
```
```
mahti_logs_df
```
```
rows
0	Loaded modules:
1	Currently Loaded Modules:
2	1) gcc/11.2.0 3) openblas/0.3.18-omp ...
3	2) openmpi/4.1.2 4) csc-tools (S...
4	Where:
...	...
482	srun: Job step aborted: Waiting up to 62 secon...
483	srun: Job step aborted: Waiting up to 62 secon...
484	slurmstepd: error: *** STEP 4393900.2 ON c3263...
485	slurmstepd: error: *** STEP 4393900.1 ON c3262...
486	slurmstepd: error: *** JOB 4393900 ON c3262 CA...

487 rows × 1 columns
```
```
formatted_mahti_logs_data = pyarrow_serialize_dataframe(
    dataframe = mahti_logs_df
)
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': mahti_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = formatted_mahti_logs_data,
    object_metadata = {'version': 1}
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: LOGS/mahti-run-1.parquet
```
```
lumi_logs = format_slurm_logs(
    file_path = '/home/()/multi-cloud-hpc-oss-mlops-platform/tutorials/integration/development/studying/logs/lumi-logs.txt'
)
```
```
lumi_logs_test = {
    'name': 'lumi-run-1',
    'rows': lumi_logs
}

validation_lumi_logs = Logs(**lumi_logs_test)

validation_lumi_logs.model_dump()
```
```
{'name': 'lumi-run-1',
 'rows': ['NOTE: This module uses Singularity. Some commands execute inside the container',
  '(e.g. python3, pip3).',
  'This module has been installed by CSC.',
  'Documentation: https://docs.csc.fi/apps/pytorch/',
  'Support: https://docs.csc.fi/support/contact/',
  'Loaded modules:',
  'Currently Loaded Modules:',
  '  1) craype-x86-rome                        9) cray-mpich/8.1.29',
  '  2) libfabric/1.15.2.0                    10) cray-libsci/24.03.0',
  '  3) craype-network-ofi                    11) PrgEnv-cray/8.5.0',
  '  4) perftools-base/24.03.0                12) ModuleLabel/label   (S)',
  '  5) xpmem/2.8.2-1.0_5.1__g84a27a5.shasta  13) lumi-tools/24.05    (S)',
  '  6) cce/17.0.1                            14) init-lumi/0.2       (S)',
  '  7) craype/2.7.31.11                      15) pytorch/2.7',
  '  8) cray-dsmml/0.3.0',
  '  Where:',
  '   S:  Module is Sticky, requires --force to unload or purge',
  'Activating venv',
  'Venv active',
  'Installed packages',
  'Package                                  Version',
  '---------------------------------------- -------------------------',
  'absl-py                                  1.4.0',
  'accelerate                               1.7.0',
  'affine                                   2.4.0',
  'aiohappyeyeballs                         2.6.1',
  'aiohttp                                  3.11.18',
  'aiohttp-cors                             0.8.1',
  'aiosignal                                1.3.2',
  'airportsdata                             20250224',
  'alembic                                  1.15.2',
  'amdsmi                                   24.6.3+9578815',
  'annotated-types                          0.7.0',
  'ansicolors                               1.1.8',
  'anyio                                    4.9.0',
  'apex                                     1.7.0a0',
  'argon2-cffi                              23.1.0',
  'argon2-cffi-bindings                     21.2.0',
  'array_record                             0.7.2',
  'arrow                                    1.3.0',
  'astor                                    0.8.1',
  'asttokens                                3.0.0',
  'async-lru                                2.0.5',
  'async-timeout                            5.0.1',
  'attrs                                    25.3.0',
  'audioread                                3.0.1',
  'autocommand                              2.2.2',
  'awscli                                   1.40.21',
  'babel                                    2.17.0',
  'backports.tarfile                        1.2.0',
  'beautifulsoup4                           4.13.4',
  'bitsandbytes                             0.43.3.dev0',
  'blake3                                   1.0.5',
  'bleach                                   6.2.0',
  'blinker                                  1.9.0',
  'blosc2                                   3.3.3',
  'boto3                                    1.38.22',
  'botocore                                 1.38.22',
  'cachetools                               5.5.2',
  'certifi                                  2025.4.26',
  'cffi                                     1.17.1',
  'cftime                                   1.6.4.post1',
  'charset-normalizer                       3.4.2',
  'clang                                    20.1.5',
  'click                                    8.1.8',
  'click-plugins                            1.1.1',
  'cligj                                    0.7.2',
  'cloudpickle                              3.1.1',
  'cmake                                    4.0.2',
  'colorama                                 0.4.6',
  'colorful                                 0.5.6',
  'comm                                     0.2.2',
  'compressed-tensors                       0.10.1',
  'contourpy                                1.3.2',
  'cycler                                   0.12.1',
  'Cython                                   3.1.1',
  'dask                                     2025.5.1',
  'dask-jobqueue                            0.9.0',
  'databricks-sdk                           0.53.0',
  'datasets                                 3.6.0',
  'debtcollector                            3.0.0',
  'debugpy                                  1.8.14',
  'decorator                                5.2.1',
  'deepspeed                                0.17.1+2ce55057',
  'defusedxml                               0.7.1',
  'Deprecated                               1.2.18',
  'depyf                                    0.18.0',
  'diffusers                                0.33.1',
  'dill                                     0.3.8',
  'diskcache                                5.6.3',
  'distlib                                  0.3.9',
  'distributed                              2025.5.1',
  'distro                                   1.9.0',
  'dm-tree                                  0.1.9',
  'dnspython                                2.7.0',
  'docker                                   7.1.0',
  'docstring_parser                         0.16',
  'docutils                                 0.19',
  'einops                                   0.8.1',
  'email_validator                          2.2.0',
  'entrypoints                              0.4',
  'et_xmlfile                               2.0.0',
  'etils                                    1.12.2',
  'evaluate                                 0.4.3',
  'executing                                2.2.0',
  'faiss                                    1.11.0',
  'fastapi                                  0.115.12',
  'fastapi-cli                              0.0.7',
  'fastjsonschema                           2.21.1',
  'filelock                                 3.18.0',
  'flash-attn                               2.7.4.post1',
  'Flask                                    3.1.1',
  'fonttools                                4.58.0',
  'fqdn                                     1.5.1',
  'frozenlist                               1.6.0',
  'fsspec                                   2025.3.0',
  'fvcore                                   0.1.5.post20221221',
  'gensim                                   4.3.3',
  'geopandas                                1.0.1',
  'gguf                                     0.16.3',
  'gitdb                                    4.0.12',
  'GitPython                                3.1.44',
  'google-api-core                          2.24.2',
  'google-auth                              2.40.1',
  'googleapis-common-protos                 1.70.0',
  'gpytorch                                 1.14',
  'graphene                                 3.4.3',
  'graphql-core                             3.2.6',
  'graphql-relay                            3.2.0',
  'graphviz                                 0.20.3',
  'greenlet                                 3.2.2',
  'grpcio                                   1.71.0',
  'gunicorn                                 23.0.0',
  'gym                                      0.26.2',
  'gym-notices                              0.0.8',
  'h11                                      0.16.0',
  'h5py                                     3.13.0',
  'hf_transfer                              0.1.9',
  'hf-xet                                   1.1.2',
  'hiredis                                  3.1.1',
  'hjson                                    3.1.0',
  'httpcore                                 1.0.9',
  'httptools                                0.6.4',
  'httpx                                    0.28.1',
  'huggingface-hub                          0.33.0',
  'humanize                                 4.12.3',
  'idna                                     3.10',
  'imageio                                  2.37.0',
  'imbalanced-learn                         0.13.0',
  'immutabledict                            4.2.1',
  'importlib_metadata                       8.0.0',
  'importlib_resources                      6.5.2',
  'inflect                                  7.3.1',
  'iniconfig                                2.1.0',
  'inquirerpy                               0.3.4',
  'interegular                              0.3.3',
  'iopath                                   0.1.10',
  'ipykernel                                6.29.5',
  'ipython                                  9.2.0',
  'ipython-genutils                         0.2.0',
  'ipython_pygments_lexers                  1.1.1',
  'ipywidgets                               8.1.7',
  'iso8601                                  2.1.0',
  'isoduration                              20.11.0',
  'itsdangerous                             2.2.0',
  'jaraco.collections                       5.1.0',
  'jaraco.context                           5.3.0',
  'jaraco.functools                         4.0.1',
  'jaraco.text                              3.12.1',
  'jaxtyping                                0.3.2',
  'jedi                                     0.19.2',
  'Jinja2                                   3.1.6',
  'jiter                                    0.10.0',
  'jmespath                                 1.0.1',
  'joblib                                   1.5.0',
  'json5                                    0.12.0',
  'jsonpatch                                1.33',
  'jsonpointer                              3.0.0',
  'jsonschema                               4.23.0',
  'jsonschema-specifications                2025.4.1',
  'jupyter_client                           8.6.3',
  'jupyter_core                             5.7.2',
  'jupyter-events                           0.12.0',
  'jupyter-lsp                              2.2.5',
  'jupyter_server                           2.16.0',
  'jupyter-server-mathjax                   0.2.6',
  'jupyter_server_terminals                 0.5.3',
  'jupyterlab                               4.4.2',
  'jupyterlab_git                           0.51.1',
  'jupyterlab_pygments                      0.3.0',
  'jupyterlab_server                        2.27.3',
  'jupyterlab_widgets                       3.0.15',
  'jupytext                                 1.17.1',
  'kagglehub                                0.3.12',
  'keopscore                                2.3',
  'keras                                    3.10.0',
  'keras-core                               0.1.7',
  'keras-cv                                 0.9.0',
  'keystoneauth1                            5.11.0',
  'kiwisolver                               1.4.8',
  'lark                                     1.2.2',
  'lazy_loader                              0.4',
  'libnacl                                  2.1.0',
  'librosa                                  0.11.0',
  'lightning                                2.5.1.post0',
  'lightning-utilities                      0.14.3',
  'linear-operator                          0.6',
  'lion-pytorch                             0.2.3',
  'lit                                      18.1.8',
  'llguidance                               0.7.21',
  'llvmlite                                 0.44.0',
  'lm-format-enforcer                       0.10.11',
  'lmdb                                     1.6.2',
  'locket                                   1.0.0',
  'lxml                                     5.4.0',
  'Mako                                     1.3.10',
  'Markdown                                 3.8',
  'markdown-it-py                           3.0.0',
  'MarkupSafe                               3.0.2',
  'matplotlib                               3.9.4',
  'matplotlib-inline                        0.1.7',
  'mdit-py-plugins                          0.4.2',
  'mdurl                                    0.1.2',
  'mistral_common                           1.5.5',
  'mistune                                  3.1.3',
  'ml_dtypes                                0.5.1',
  'mlflow                                   2.22.0',
  'mlflow-skinny                            2.22.0',
  'more-itertools                           10.3.0',
  'mpi4py                                   4.0.3',
  'mpmath                                   1.3.0',
  'msgpack                                  1.1.0',
  'msgspec                                  0.19.0',
  'multidict                                6.4.4',
  'multiprocess                             0.70.16',
  'mysql-connector-python                   9.3.0',
  'namex                                    0.0.9',
  'nbclassic                                1.3.1',
  'nbclient                                 0.10.2',
  'nbconvert                                7.16.6',
  'nbdime                                   4.0.2',
  'nbformat                                 5.10.4',
  'ndindex                                  1.9.2',
  'nest-asyncio                             1.6.0',
  'netaddr                                  1.3.0',
  'netCDF4                                  1.7.2',
  'networkx                                 3.4.2',
  'ninja                                    1.11.1.4',
  'nltk                                     3.9.1',
  'notebook                                 7.4.2',
  'notebook_shim                            0.2.4',
  'numba                                    0.61.2',
  'numexpr                                  2.10.2',
  'numpy                                    1.26.4',
  'nvidia-nccl-cu12                         2.26.5',
  'odfpy                                    1.4.1',
  'openai                                   1.79.0',
  'opencensus                               0.11.4',
  'opencensus-context                       0.1.3',
  'opencv-python                            4.11.0.86',
  'opencv-python-headless                   4.11.0.86',
  'openpyxl                                 3.1.5',
  'opentelemetry-api                        1.26.0',
  'opentelemetry-exporter-otlp              1.26.0',
  'opentelemetry-exporter-otlp-proto-common 1.26.0',
  'opentelemetry-exporter-otlp-proto-grpc   1.26.0',
  'opentelemetry-exporter-otlp-proto-http   1.26.0',
  'opentelemetry-proto                      1.26.0',
  'opentelemetry-sdk                        1.26.0',
  'opentelemetry-semantic-conventions       0.47b0',
  'opentelemetry-semantic-conventions-ai    0.4.9',
  'optree                                   0.15.0',
  'os-service-types                         1.7.0',
  'oslo.config                              9.8.0',
  'oslo.i18n                                6.5.1',
  'oslo.serialization                       5.7.0',
  'oslo.utils                               9.0.0',
  'outlines                                 0.1.11',
  'outlines_core                            0.1.26',
  'overrides                                7.7.0',
  'packaging                                24.2',
  'pandas                                   2.2.3',
  'pandocfilters                            1.5.1',
  'papermill                                2.6.0',
  'parso                                    0.8.4',
  'partd                                    1.4.2',
  'partial-json-parser                      0.2.1.1.post5',
  'pbr                                      6.1.1',
  'peft                                     0.15.2',
  'pexpect                                  4.9.0',
  'pfzy                                     0.3.4',
  'pillow                                   11.2.1',
  'pip                                      22.0.2',
  'platformdirs                             4.3.8',
  'pluggy                                   1.6.0',
  'pooch                                    1.8.2',
  'portalocker                              3.1.1',
  'prometheus_client                        0.22.0',
  'prometheus-fastapi-instrumentator        7.1.0',
  'promise                                  2.3',
  'prompt_toolkit                           3.0.51',
  'propcache                                0.3.1',
  'proto-plus                               1.26.1',
  'protobuf                                 4.25.7',
  'psutil                                   7.0.0',
  'ptyprocess                               0.7.0',
  'pure_eval                                0.2.3',
  'py-cpuinfo                               9.0.0',
  'py-spy                                   0.4.0',
  'pyarrow                                  19.0.1',
  'pyasn1                                   0.6.1',
  'pyasn1_modules                           0.4.2',
  'pybind11                                 2.13.6',
  'pycodestyle                              2.13.0',
  'pycountry                                24.6.1',
  'pycparser                                2.22',
  'pydantic                                 2.11.4',
  'pydantic_core                            2.33.2',
  'pydot                                    4.0.0',
  'pyflakes                                 3.3.2',
  'Pygments                                 2.19.1',
  'pykeops                                  2.3',
  'pynndescent                              0.5.13',
  'pyogrio                                  0.11.0',
  'PyOpenGL                                 3.1.5',
  'pyparsing                                3.2.3',
  'pyproj                                   3.7.1',
  'pyprojroot                               0.3.0',
  'pysqlite3                                0.5.4',
  'pytest                                   8.3.5',
  'pytest-asyncio                           0.26.0',
  'python-dateutil                          2.9.0.post0',
  'python-dotenv                            1.1.0',
  'python-json-logger                       3.3.0',
  'python-keystoneclient                    5.6.0',
  'python-multipart                         0.0.20',
  'python-swiftclient                       4.7.0',
  'pytorch-lightning                        2.5.1.post0',
  'pytorch-triton-rocm                      3.3.1',
  'pytz                                     2025.2',
  'PyYAML                                   6.0.2',
  'pyzmq                                    26.4.0',
  'rasterio                                 1.4.3',
  'ray                                      2.44.1',
  'redis                                    6.1.0',
  'referencing                              0.36.2',
  'regex                                    2024.11.6',
  'requests                                 2.32.3',
  'rfc3339-validator                        0.1.4',
  'rfc3986                                  2.0.0',
  'rfc3986-validator                        0.1.1',
  'rich                                     14.0.0',
  'rich-toolkit                             0.14.6',
  'rpds-py                                  0.25.0',
  'rsa                                      4.7.2',
  'runai-model-streamer                     0.11.0',
  'runai-model-streamer-s3                  0.11.0',
  's3transfer                               0.13.0',
  'safetensors                              0.5.3',
  'scikit-image                             0.25.2',
  'scikit-learn                             1.6.1',
  'scipy                                    1.14.1',
  'seaborn                                  0.13.2',
  'Send2Trash                               1.8.3',
  'sentencepiece                            0.2.0',
  'setuptools                               59.6.0',
  'shapely                                  2.1.1',
  'shellingham                              1.5.4',
  'simple-parsing                           0.1.7',
  'six                                      1.17.0',
  'sklearn-compat                           0.1.3',
  'smart-open                               7.1.0',
  'smmap                                    5.0.2',
  'sniffio                                  1.3.1',
  'sortedcontainers                         2.4.0',
  'soundfile                                0.13.1',
  'soupsieve                                2.7',
  'soxr                                     0.5.0.post1',
  'SQLAlchemy                               2.0.41',
  'sqlparse                                 0.5.3',
  'stack-data                               0.6.3',
  'starlette                                0.46.2',
  'stevedore                                5.4.1',
  'sympy                                    1.14.0',
  'tables                                   3.10.2',
  'tabulate                                 0.9.0',
  'tblib                                    3.1.0',
  'tenacity                                 9.1.2',
  'tensorboard                              2.19.0',
  'tensorboard-data-server                  0.7.2',
  'tensorboardX                             2.6.2.2',
  'tensorflow-datasets                      4.9.8',
  'tensorflow-metadata                      1.14.0',
  'tensorizer                               2.9.3',
  'termcolor                                3.1.0',
  'terminado                                0.18.1',
  'threadpoolctl                            3.6.0',
  'tifffile                                 2025.5.10',
  'tiktoken                                 0.9.0',
  'timm                                     1.0.15',
  'tinycss2                                 1.4.0',
  'tokenizers                               0.21.1',
  'toml                                     0.10.2',
  'tomli                                    2.0.1',
  'toolz                                    1.0.0',
  'torch                                    2.7.1+rocm6.2.4',
  'torch-tb-profiler                        0.4.3',
  'torchaudio                               2.7.1+rocm6.2.4',
  'torchinfo                                1.8.0',
  'torchmetrics                             1.7.1',
  'torchvision                              0.22.1+rocm6.2.4',
  'tornado                                  6.5',
  'tqdm                                     4.67.1',
  'traitlets                                5.14.3',
  'transformers                             4.52.4',
  'triton                                   3.3.0',
  'trl                                      0.17.0',
  'typeguard                                4.3.0',
  'typer                                    0.15.4',
  'types-python-dateutil                    2.9.0.20250516',
  'typing_extensions                        4.13.2',
  'typing-inspection                        0.4.0',
  'tzdata                                   2025.2',
  'umap-learn                               0.5.7',
  'uri-template                             1.3.0',
  'urllib3                                  2.4.0',
  'uvicorn                                  0.34.2',
  'uvloop                                   0.21.0',
  'virtualenv                               20.31.2',
  'visdom                                   0.2.4',
  'vllm                                     0.9.1+rocm624',
  'wadler_lindig                            0.1.6',
  'watchfiles                               1.0.5',
  'wcwidth                                  0.2.13',
  'webcolors                                24.11.1',
  'webencodings                             0.5.1',
  'websocket-client                         1.8.0',
  'websockets                               15.0.1',
  'Werkzeug                                 3.1.3',
  'wheel                                    0.43.0',
  'widgetsnbextension                       4.0.14',
  'wrapt                                    1.17.2',
  'xformers                                 0.0.31+39addc86.d20250521',
  'xgboost                                  3.0.1',
  'xgrammar                                 0.1.19',
  'xlwt                                     1.3.0',
  'xxhash                                   3.5.0',
  'yacs                                     0.1.8',
  'yarl                                     1.20.0',
  'zict                                     3.0.0',
  'zipp                                     3.21.0',
  'Packages listed',
  'Setting connection variables',
  'Setting Ray ports',
  'Setting Ray settings',
  'Setting up Ray head',
  'IP Head: ():8050',
  'Starting HEAD at nid002440',
  'Setting up SSH tunnel for head dash',
  'Setting up Ray workers',
  'Starting WORKER 1 at nid002441',
  'Starting wait',
  '2025-11-21 14:20:11,935 INFO usage_lib.py:467 -- Usage stats collection is enabled by default without user confirmation because this terminal is detected to be non-interactive. To disable this, add `--disable-usage-stats` to the command that starts the cluster, or run the following command: `ray disable-usage-stats` before starting the cluster. See https://docs.ray.io/en/master/cluster/usage-stats.html for more details.',
  '2025-11-21 14:20:11,936 INFO scripts.py:861 -- Local node IP: ()',
  '2025-11-21 14:20:16,017 SUCC scripts.py:897 -- --------------------',
  '2025-11-21 14:20:16,017 SUCC scripts.py:898 -- Ray runtime started.',
  '2025-11-21 14:20:16,017 SUCC scripts.py:899 -- --------------------',
  '2025-11-21 14:20:16,017 INFO scripts.py:901 -- Next steps',
  '2025-11-21 14:20:16,018 INFO scripts.py:904 -- To add another node to this Ray cluster, run',
  "2025-11-21 14:20:16,018 INFO scripts.py:907 --   ray start --address='():8050'",
  '2025-11-21 14:20:16,018 INFO scripts.py:916 -- To connect to this Ray cluster:',
  '2025-11-21 14:20:16,018 INFO scripts.py:918 -- import ray',
  "2025-11-21 14:20:16,018 INFO scripts.py:919 -- ray.init(_node_ip_address='()')",
  '2025-11-21 14:20:16,018 INFO scripts.py:931 -- To submit a Ray job using the Ray Jobs CLI:',
  "2025-11-21 14:20:16,018 INFO scripts.py:932 --   RAY_ADDRESS='http://():8265' ray job submit --working-dir . -- python my_script.py",
  '2025-11-21 14:20:16,018 INFO scripts.py:941 -- See https://docs.ray.io/en/latest/cluster/running-applications/job-submission/index.html ',
  '2025-11-21 14:20:16,018 INFO scripts.py:945 -- for more information on submitting Ray jobs to the Ray cluster.',
  '2025-11-21 14:20:16,018 INFO scripts.py:950 -- To terminate the Ray runtime, run',
  '2025-11-21 14:20:16,018 INFO scripts.py:951 --   ray stop',
  '2025-11-21 14:20:16,018 INFO scripts.py:954 -- To view the status of the cluster, use',
  '2025-11-21 14:20:16,018 INFO scripts.py:955 --   ray status',
  '2025-11-21 14:20:16,018 INFO scripts.py:959 -- To monitor and debug Ray, view the dashboard at ',
  '2025-11-21 14:20:16,018 INFO scripts.py:960 --   ():8265',
  '2025-11-21 14:20:16,018 INFO scripts.py:967 -- If connection to the dashboard fails, check your firewall settings and network configuration.',
  '2025-11-21 14:20:16,018 INFO scripts.py:1071 -- --block',
  '2025-11-21 14:20:16,018 INFO scripts.py:1072 -- This command will now block forever until terminated by a signal.',
  '2025-11-21 14:20:16,018 INFO scripts.py:1075 -- Running subprocesses are monitored and a message will be printed if any of them terminate unexpectedly. Subprocesses exit with SIGTERM will be treated as graceful, thus NOT reported.',
  '[2025-11-21 14:20:18,248 W 35846 35846] global_state_accessor.cc:429: Retrying to get node with node ID cb8468e80d9ea04b037d84c569b963ee914abc8b8ac933b41d4d852a',
  '[2025-11-21 14:20:19,249 W 35846 35846] global_state_accessor.cc:429: Retrying to get node with node ID cb8468e80d9ea04b037d84c569b963ee914abc8b8ac933b41d4d852a',
  '2025-11-21 14:20:17,994 INFO scripts.py:1042 -- Local node IP: ()',
  '2025-11-21 14:20:20,269 SUCC scripts.py:1058 -- --------------------',
  '2025-11-21 14:20:20,269 SUCC scripts.py:1059 -- Ray runtime started.',
  '2025-11-21 14:20:20,269 SUCC scripts.py:1060 -- --------------------',
  '2025-11-21 14:20:20,269 INFO scripts.py:1062 -- To terminate the Ray runtime, run',
  '2025-11-21 14:20:20,269 INFO scripts.py:1063 --   ray stop',
  '2025-11-21 14:20:20,269 INFO scripts.py:1071 -- --block',
  '2025-11-21 14:20:20,270 INFO scripts.py:1072 -- This command will now block forever until terminated by a signal.',
  '2025-11-21 14:20:20,270 INFO scripts.py:1075 -- Running subprocesses are monitored and a message will be printed if any of them terminate unexpectedly. Subprocesses exit with SIGTERM will be treated as graceful, thus NOT reported.',
  'srun: Job step aborted: Waiting up to 32 seconds for job step to finish.',
  'srun: Job step aborted: Waiting up to 32 seconds for job step to finish.',
  'slurmstepd: error: *** STEP 14838873.1 ON nid002440 CANCELLED AT 2025-11-21T14:20:55 ***',
  'slurmstepd: error: *** JOB 14838873 ON nid002440 CANCELLED AT 2025-11-21T14:20:55 ***']}
```
```
lumi_file_name = lumi_logs_test['name'] + '.parquet'
lumi_logs_df = pd.DataFrame({'rows': lumi_logs_test['rows']})
print(lumi_file_name)
```
```
lumi-run-1.parquet
```
```
lumi_logs_df
```
```
	rows
0	NOTE: This module uses Singularity. Some comma...
1	(e.g. python3, pip3).
2	This module has been installed by CSC.
3	Documentation: https://docs.csc.fi/apps/pytorch/
4	Support: https://docs.csc.fi/support/contact/
...	...
497	2025-11-21 14:20:20,270 INFO scripts.py:1075 -...
498	srun: Job step aborted: Waiting up to 32 secon...
499	srun: Job step aborted: Waiting up to 32 secon...
500	slurmstepd: error: *** STEP 14838873.1 ON nid0...
501	slurmstepd: error: *** JOB 14838873 ON nid0024...

502 rows × 1 columns
```
```
formatted_lumi_logs_data = pyarrow_serialize_dataframe(
    dataframe = lumi_logs_df
)
```
```
send_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'send',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': lumi_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = formatted_lumi_logs_data,
    object_metadata = {'version': 1}
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: LOGS/lumi-run-1.parquet
```
Now, if you go into https://pouta.csc.fi/, you should see this container and file appear by doing the following:
- Click object storage
- Click containers
- Click mlch-pipe-user-example-com
- Click LOGS

We can get the dataframes with the following way:
```
get_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'get',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': puhti_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = None,
    object_metadata = None
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: LOGS/puhti-run-1.parquet
```
```
formatted_stored_puhti_logs = pyarrow_deserialize_dataframe(serialized_dataframe = get_interaction_output[0])
stored_general_metadata = get_interaction_output[1]
stored_custom_metadata = get_interaction_output[2]
```
```
formatted_stored_puhti_logs
```
```
	rows
0	Loaded modules:
1	Currently Loaded Modules:
2	1) gcc/11.3.0 3) openmpi/4....
3	2) intel-oneapi-mkl/2022.1.0 4) csc-tools ...
4	Where:
...	...
492	srun: Job step aborted: Waiting up to 62 secon...
493	srun: Job step aborted: Waiting up to 62 secon...
494	slurmstepd: error: *** STEP 27587022.2 ON r02c...
495	slurmstepd: error: *** JOB 27587022 ON r02c17 ...
496	slurmstepd: error: *** STEP 27587022.1 ON r02c...

497 rows × 1 columns
```
```
stored_custom_metadata 
```
```
{'version': 1}
```
```
get_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'get',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': mahti_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = None,
    object_metadata = None
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: LOGS/mahti-run-1.parque
```
```
formatted_stored_mahti_logs = pyarrow_deserialize_dataframe(serialized_dataframe = get_interaction_output[0])
stored_general_metadata = get_interaction_output[1]
stored_custom_metadata = get_interaction_output[2]
```
```
formatted_stored_mahti_logs
```
```

rows
0	Loaded modules:
1	Currently Loaded Modules:
2	1) gcc/11.2.0 3) openblas/0.3.18-omp ...
3	2) openmpi/4.1.2 4) csc-tools (S...
4	Where:
...	...
482	srun: Job step aborted: Waiting up to 62 secon...
483	srun: Job step aborted: Waiting up to 62 secon...
484	slurmstepd: error: *** STEP 4393900.2 ON c3263...
485	slurmstepd: error: *** STEP 4393900.1 ON c3262...
486	slurmstepd: error: *** JOB 4393900 ON c3262 CA...

487 rows × 1 columns
```
```
stored_custom_metadata
```
```
{'version': 1}
```
```
get_interaction_output = object_storage_interaction(
    storage_client = workflow_swift_client,
    parameters = {
        'mode': 'get',
        'bucket-target': 'pipeline',
        'bucket-prefix': 'mlch',
        'bucket-user': 'user@example.com',
        'object-name': 'logs',
        'path-replacers': {
            'name': lumi_file_name
        },
        'path-names': [],
        'overwrite': True
    },
    object_data = None,
    object_metadata = None
)
```
```
User object bucket: mlch-pipe-user-example-com
Used object path: LOGS/lumi-run-1.parquet
```
```
formatted_stored_lumi_logs = pyarrow_deserialize_dataframe(serialized_dataframe = get_interaction_output[0])
stored_general_metadata = get_interaction_output[1]
stored_custom_metadata = get_interaction_output[2]
```
```
formatted_stored_lumi_logs
```
```

rows
0	NOTE: This module uses Singularity. Some comma...
1	(e.g. python3, pip3).
2	This module has been installed by CSC.
3	Documentation: https://docs.csc.fi/apps/pytorch/
4	Support: https://docs.csc.fi/support/contact/
...	...
497	2025-11-21 14:20:20,270 INFO scripts.py:1075 -...
498	srun: Job step aborted: Waiting up to 32 secon...
499	srun: Job step aborted: Waiting up to 32 secon...
500	slurmstepd: error: *** STEP 14838873.1 ON nid0...
501	slurmstepd: error: *** JOB 14838873 ON nid0024...

502 rows × 1 columns
```
```
stored_custom_metadata
```
```
{'version': 1}
```
With this we finally have the necessery understanding of storing data into Allas with interaction functions with pydantic standardized objects. Be aware that ML has different storage formats, which will most likely require you to modify the interaction and swift code to enable easier transfers of data. From this point forward we will start to use abstraction methods that enable us to hide the long object models and storage interaction functions due to them taking a lot of rows, which results in worse development and user experience. Both of these are the main problems we need to constantly find better ways to solve to ensure smoother integration and use of distributed systems.