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
OUT 773 HERE
### Logs
```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

