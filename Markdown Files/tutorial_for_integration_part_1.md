# Tutorial for integrated distributed computing in MLOps (1/10)
This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.
## Basics
Useful material:
- https://dl.acm.org/doi/10.1145/3672608.3707866
- https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/tree/development/tutorials/integration

The interactions between separated systems are enabled by the following:
- Platform access = MyCSC account with SSH keys
- Interaction interface = Linux terminal
- Global storage = CSC Allas object storage
- Interaction automation = Python code run Apache Airflow
- Cooperative communication = Stored objects in CSC Allas
- Action monitoring = Logs created by Airflow, Submitter and Forwarder

The utilization of the interacted system depends on the tools enabled by the enviroment:
- Local enviroment
    - Owned computers can run almost anything within available resources
- Cloud enviroment
    - Virtual machines can run anything within the limits of vendor resources, terms of service and available budget
- Storage enviroment
    - Object storages can store anything within the limits of vendor resources, terms of service and available budget
- High performance computing enviroment
    - Supercomputers enable program execution in vendor approved ways in a shared computing cluster

These result in the following enviroment trade offs:
- Local has easy usability, but limited resources (you have as much resources as you are willing to spend on hardware and maintanance)
- Cloud has flexible resources, but limited scalability (you have as much resources as the vendor and your budget allows)
- Storage has dedicated storage, but utilization dependencies (you can only store in predefined ways that the use enviroment must enable)
- HPC has scalable computing, but limited usability (there is curated access and you can only do things in the rules the vendor set)

For these reasons, we need to select and design software for the integration that are:
- mature
- abstracted
- interoperable
## Python
Useful material:
- https://pip.pypa.io/en/latest/user_guide/
- https://docs.conda.io/projects/conda/en/stable/user-guide/getting-started.html
- https://builtin.com/articles/pip-install-specific-version

Was choosen as the default language for integration for the following reasons:
- One of the most used programming languages in ML (mature)
- Syntax is simpler compared to options like C and C++ (abstracted)
- Widely supported by local, cloud and HPC platforms (interopeable)

These make Python the default language of the MLOps users, who are primarily data scientists that like to work with Jupyter Notebooks. For this reason the main concern of the integration is user experience, since the data scientists need to be able to understand and modify the existing integration code to make their minimum viable product applications. For our use case the most important things about python are
- Virtual enviroments

```
python3 -m venv (venv_name)
```

- Enviroment activation

```
source (venv_name)/bin/activate
```

- Package installation

```
pip install (package)

pip install -r requirements.txt
```

- Requirements creation

```
pip freeze >> requirements.txt
```

Be aware that pip freeze shows all the packages, creating a long list that is fine for full configuration, but usually difficult to ready. A solution to this is to just hand type a requirements.txt with specified versions. For example, check the following websites (15.9.2025)

- https://pypi.org/project/fastapi/
- https://pypi.org/project/uvicorn/
- https://pypi.org/project/celery/
- https://pypi.org/project/redis/

we get the following for submitter frontend

```
fastapi==0.116.1
uvicorn==0.35.0
redis==6.4.0
celery==5.5.3
```

This enables easier downgrades if any future package update causes depedency problems. If changes in these do not help, then debugging should be done with pip freeze. Fortunately pip provides good enough error messages that specify what packages are causing problems. These packages can be changed by specifying a version using:

```
pip uninstall (package)
pip install (package)==(version)
```

Be aware that your python version might also cause problems. In such cases I recommend using conda to get a suitable python version, activating it and then creating the virtual enviroment. It is a good idea to create a pip freeze requirements once you have a working application to enable locking back during future development.
## JupyterLab
Useful material:
- https://jupyter.org/install
- https://ipython.readthedocs.io/en/9.2.0/interactive/magics.html

In order to speed up development with easier documentation, HTTP interactions and python code testing, we need to setup JupyterLab. Run the following:

```
python3 -m venv tutorial_venv
source tutorial_venv/bin/activate 
pip install -r packages.txt
jupyter lab
```

When JupyterLab is running, it should create a window in the choosen default browser in the activation place. You should then just find this notebook to interact with it. You can shut it down with:

```
CTRL + C
Y
```

Be aware that package updates require shutting down to take affect, which is why it is recommeded to save a lot. Besides just running python code in the code block, Jupyter notebooks enable running different magic commands. In our case the most important ones are:
- %%writefile = writes a file
- %run = runs a python file
## Pydantic
Useful material:
- https://docs.pydantic.dev/latest/
- https://pypi.org/project/pydantic/

Pydantic is data validation tool that is mainly used to build and validate JSONs in frontend and backend interactions. To begin, run the following commands
```
python3 -m venv tutorial_venv
source tutorial_venv/bin/activate 
pip install -r packages.txt
```
With pydantic, we can validate the following dictionary:
```
venv_test = {
    'name': 'exp-venv',
    'path': '/users/(csc-user)',
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
```
```
from pydantic import BaseModel, Field
from typing import List
 
class Venv(BaseModel):
    name: str = Field(alias = 'name')
    path: str = Field(alias = 'path')
    modules: List[str] = Field(default_factory = list, alias = 'modules')
    packages: List[str] = Field(default_factory = list, alias = 'packages')
```
```
validated_venv = Venv(**venv_test)
# Model form
print(validated_venv)
# JSON form
print(validated_venv.model_dump())
```
name='exp-venv' path='/users/(csc-user)' modules=['pytorch'] packages=['ray', 'python-swiftclient', 'torch', 'torchmetrics']
{'name': 'exp-venv', 'path': '/users/(csc-user)', 'modules': ['pytorch'], 'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}

Models can be used by other models, allowing step by step validation:
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
```
```
class Configs(BaseModel):
    modules: List[str] = Field(default_factory = list, alias = 'modules')
    venv: Venv = Field(alias = 'venv')
```
```
validated_config = Configs(**configs_test)
# Model form
print(validated_config)
# JSON form
print(validated_config.model_dump())
```
modules=['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'] venv=Venv(name='exp-venv', path='/users/(csc-user)', modules=['pytorch'], packages=['ray', 'python-swiftclient', 'torch', 'torchmetrics'])
{'modules': ['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'], 'venv': {'name': 'exp-venv', 'path': '/users/(csc-user)', 'modules': ['pytorch'], 'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}}
### YAML
Useful material:
- https://pypi.org/project/PyYAML/

Unfortunately defining dicts that configure applications gets unreadable with enough rows in jupyterlab, which is why we can instead create a YAML that we read and validate with pydantic. The previous configs dict can be written into a YAML. Please replace (path_here) with a absolute path for the applications folder:
```
%%writefile (path_here)/configs.yaml
modules:
  - 'gcc'
  - 'openmpi'
  - 'openblas'
  - 'csc-tools'
  - 'StdEnv'
venv:
  name: 'exp-venv'
  path: '/users/(csc-user)'
  modules:
    - 'pytorch'
  packages: 
    - 'ray'
    - 'python-swiftclient'
    - 'torch'
    - 'torchmetrics'
```
We can read it into a dict with the following:
```
import yaml

yaml_paths = [
    '(path_here)/configs.yaml'
]

yaml_dicts = {}
for path in yaml_paths:
    with open(path, 'r') as f:
        file_name = path.split('/')[-1].split('.')[0]
        yaml_dict = yaml.safe_load(f)
        pydantic_validation = Configs.model_validate(yaml_dict)
        print(pydantic_validation)
        yaml_dicts[file_name] = yaml_dict

print(yaml_dicts)
```
modules=['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'] venv=Venv(name='exp-venv', path='/users/(csc-user)', modules=['pytorch'], packages=['ray', 'python-swiftclient', 'torch', 'torchmetrics'])
{'configs': {'modules': ['gcc', 'openmpi', 'openblas', 'csc-tools', 'StdEnv'], 'venv': {'name': 'exp-venv', 'path': '/users/(csc-user)', 'modules': ['pytorch'], 'packages': ['ray', 'python-swiftclient', 'torch', 'torchmetrics']}}}
## Docker setup
Useful material:
- https://docs.docker.com/desktop/
- https://docs.docker.com/engine/install/linux-postinstall/

To begin proper development, you first need to install either docker engine or docker desktop on your machine. My recommendation is to use docker desktop both on linux and windows laptops with docker engine used in ubuntu virtual machines, since it reduces the required configuration. Please follow the Docker docs for setting the most suitable option.