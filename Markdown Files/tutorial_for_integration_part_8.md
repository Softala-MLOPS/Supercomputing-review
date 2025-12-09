# Tutorial for integrated distributed computing in MLOps (8/10)

This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.

## Index

- [CSC Supercomputers](#CSC-Supercomputers)
- [Account use](#Account Use)
- [SSH Key Management](#SSH-Key-Management)
- 

## CSC Supercomputers

### Useful material

- General
  - [Puhti](https://research.csc.fi/service/puhti/)
  - [Mahti](https://research.csc.fi/service/mahti/)
  - [Lumi](https://research.csc.fi/service/lumi-supercomputer/)
  - [Getting started with supercomputing at CSC](https://docs.csc.fi/support/tutorials/hpc-quick/)
  - [Technical details about Puhti](https://docs.csc.fi/computing/systems-puhti/)
  - [Technical details about Mahti](https://docs.csc.fi/computing/systems-mahti/)
  - [About Lumi supercomputer](https://lumi-supercomputer.eu/lumi_supercomputer/)
- SSH Key Management
  - [Setting up SSH keys](https://docs.csc.fi/computing/connecting/ssh-keys/)
- CPouta Connection Configuration
  - [Using Firewalls in Pukki](https://docs.csc.fi/cloud/dbaas/firewalls/)
- Lustre filesystem
  - [Lustre file system](https://docs.csc.fi/computing/lustre/)
  - [Disk areas](https://docs.csc.fi/computing/disk/)
  - [Lumi Supercomputer Storage Overview](https://docs.lumi-supercomputer.eu/storage/)
  - [Lustre - Parallel Filesystem](https://docs.lumi-supercomputer.eu/storage/parallel-filesystems/lustre/)
  - [Lumi Daily Management](https://docs.lumi-supercomputer.eu/runjobs/lumi_env/dailymanagement/)
- Lmod
  - [Lmod](https://github.com/TACC/Lmod)
  - [The module system](https://docs.csc.fi/computing/modules/)
  - [Modules Environment](https://docs.lumi-supercomputer.eu/runjobs/lumi_env/Lmod_modules/)
  - [PyTorch](https://docs.csc.fi/apps/pytorch/)
  - [CSC Installed Software Collection](https://docs.lumi-supercomputer.eu/software/local/csc/)
  - [What is the equivalent of `shell=True` in `paramiko`?](https://stackoverflow.com/questions/70415821/what-is-the-equivalent-of-shell-true-in-paramiko)
  - [Bash, Linux Manual Page](https://www.man7.org/linux/man-pages/man1/bash.1.html)
  - [Why is SSH not invoking .bash_profile?](https://superuser.com/questions/952084/why-is-ssh-not-invoking-bash-profile)
  - [Why does remote Bash source .bash_profile instead of .bashrc](https://unix.stackexchange.com/questions/332531/why-does-remote-bash-source-bash-profile-instead-of-bashrc)
  - [What is the difference between .bash_profile and .bash_login?](https://unix.stackexchange.com/questions/474080/what-is-the-difference-between-bash-profile-and-bash-login)
  - [What's the difference between .bashrc, .bash_profile, and .environment?](https://stackoverflow.com/questions/415403/whats-the-difference-between-bashrc-bash-profile-and-environment)
  - [What goes in ~/.profile and ~/.bashrc?](https://askubuntu.com/questions/1411833/what-goes-in-profile-and-bashrc)
  - [What do you use and configure? .bashrc or .profile or .login or .bash_login or something else](https://forum.endeavouros.com/t/what-do-you-use-and-configure-bashrc-or-profile-or-login-or-bash-login-or-something-else/70637/4)
- Python environments
  - [Using Python on CSC supercomputers](https://docs.csc.fi/support/tutorials/python-usage-guide/)
  - [Installing Python packages](https://docs.lumi-supercomputer.eu/software/installing/python/)
- Remote Forward
  - [SSH Tunneling: Examples, Command, Server Config](https://www.ssh.com/academy/ssh/tunneling-example)
  - [SSH commands](https://linuxcommand.org/lc3_man_pages/ssh1.html)
  - [How to disable strict host key checking in ssh?](https://askubuntu.com/questions/87449/how-to-disable-strict-host-key-checking-in-ssh)
  - [SSH StrictHostKeyChecking option](https://linux-audit.com/ssh/config/client/option-stricthostkeychecking/)
- SLURM Basics
  - [Running Jobs - Getting Started](https://docs.csc.fi/computing/running/getting-started/)
  - [Creating a Batch Job Script for Puhti](https://docs.csc.fi/computing/running/creating-job-scripts-puhti/)
  - [Submitting a batch job](https://docs.csc.fi/computing/running/submitting-jobs/)
  - [Available batch job partitions](https://docs.csc.fi/computing/running/batch-job-partitions/)
  - [sbatch](https://slurm.schedmd.com/sbatch.html)
  - [CSC Summer School in High-Performance Computing 2025](https://github.com/csc-training/summerschool)
  - [scontrol](https://slurm.schedmd.com/scontrol.html)
  - [srun](https://slurm.schedmd.com/srun.html)
  - [GPU Utilization](https://docs.csc.fi/support/tutorials/gpu-ml/#puhti)
- SLURM Ray
  - [Deploying on Slurm](https://docs.ray.io/en/latest/cluster/vms/user-guides/community/slurm.html)
- SLURM Sacct
  - [sacct](https://slurm.schedmd.com/sacct.html)
- General monitoring
  - [Performance analysis](https://docs.csc.fi/computing/performance/)
  - [Seff](https://lumi-supercomputer.github.io/LUMI-EasyBuild-docs/s/seff/)
  - [GPU Utilization](https://docs.csc.fi/support/tutorials/gpu-ml/#puhti)
  - [Performance analysis strategies](https://docs.lumi-supercomputer.eu/development/profiling/strategies/)
  - [Daily Management](https://docs.lumi-supercomputer.eu/runjobs/lumi_env/dailymanagement/)

### Info

To provide example high performance computing platforms with easy access for finnish public institutions, we will now go through the details of using CSC Puhti, Mahti and LUMI for cloud-hpc integration. The most important aspects of using these platforms are:

   - Account use
   - SSH key management
   - CPouta connection configuration
   - Luster Filesystem
   - Lmod modules
   - Python enviroments
   - Remote forwards
   - SLURM basics
   - SLURM Ray
   - SLURM SACCT
   - General monitoring

#### Account use

Before we begin, we need to set constraints. Puhti, Mahti and LUMI have strict rule of only individuals executing code on their platforms. For example, this is one of LUMI's did you know texts:

```
Sharing accounts is strictly forbidden on LUMI. When discovered, you will be 
banned from the system. This is not only about sharing with physical people,
but it is also forbidden to give third-party services access to your account or
software running under your account, which is, e.g., the case when you use
VSCode remote tunnels rather than the VSCode Remote ssh plugin, or when you
use Ngrok. This list is not exhaustive. There are many other such services,
e.g., some use cases of cloudflare or Gradio.

Also forbidden is to run a server on LUMI and make that server accessible 
to others than yourself, e.g., through an ssh tunnel from a publicly accessible
machine, as that effectively allows others to trigger the execution of code
on LUMI and hence is also a way of sharing accounts.

Internet-accessible services are typically run in a fully isolated environment
and often in a so-called "demilitarized zone" from which it is extremely hard
to get access to other computers at that site. This is exactly the opposite of
what a supercomputer is, as that is a "shared nearly everything" setup to enable
users to run large parallel jobs easily and efficiently, so opening up your
account to the outside world poses a high security risk, not only to you, but
also to other users of LUMI. LUMI is not a cloud infrastructure where virtual
machines and/or containers offer protection to other users and the system
itself.
```

These rules are not violated by the integration we show later, because the Ray cluster is made only accessible in our VM that we can only personally access. Our setup however can enable accidental violations if the users are not setting constraints on their development such as only giving access to their own IPs in the VM firewalls. Therefore, you must always be careful to uphold the permission constraints set by CSC by preventing yourself and other people from breaching terms of service. In later parts we will go into details on how we ensure both security and user isolation.

#### SSH Key Management

To begin, please use https://docs.csc.fi/computing/connecting/ssh-keys/ to setup MyCSC keys for puhti, mahti and LUMI access. When you have done that, please use https://docs.csc.fi/cloud/pouta/launch-vm-from-web-gui/ to create a SSH key pair without passphrase that you can put into puhti, mahti and LUMI. Store it in your .ssh folder with the name cpouta-hpc.pem to make its use in airflow easier as shown in the connection and secret setup of part 4.

#### CPouta connection configuration

In the similar ways we configured connections in part 5 and 6, we need to ensure that SSH configuration is correct and there are correct SSH security group rules. As we already did in part 6, we need to ensure that the SSH configurations have:

```
LogLevel DEBUG3
AllowTcpForwarding yes
GatewayPorts clientspecified
```

For the security group rules, please use https://docs.csc.fi/cloud/dbaas/firewalls/ to add the following rules:

   - SSH rule 1:
       - Description: Puhti login and compute nodes
       - CIDR: 86.50.164.176/28
   - SSH rule 2:
       - Description: Mahti login and compute nodes
       - CIDR: 86.50.165.192/27
   - SSH rule 3:
       - Description: LUMI login and compute nodes
       - CIDR: 193.167.209.160/28

With these the only SSH problems will be caused by bad commands. Remember from part 6 that you can debug VM SSH connections with:

```
sudo tail -f /var/log/auth.log
```

Key permission problems can be handled with:

```
chmod 600 (key_path)
```

Authorized keys problem can be handeled with:

```
# Check current keys
cat /.ssh/authorized_keys

# Add public key
echo "(public_key)" >> /.ssh/authorized_keys
nano /.ssh/authorized_keys (path might be different)
```

Known hosts problem can be handeled with:

```
# Check known hosts
cat /.ssh/known_hosts

# Remove old ip
nano /.ssh/known_hosts (paht might be different)
ssh-keygen -R (old ip) -f /.ssh/known_hosts (path might be different)

Be aware that any other problems can most likely be fixed by searching documentation, blogs and forums due to SSH maturity
```

#### Lustre filesystem

Please read the best practices of https://docs.csc.fi/computing/lustre/ and https://docs.csc.fi/computing/disk/ to understand the file systems of puhti and mahti. If you have LUMI access, check https://docs.lumi-supercomputer.eu/storage/parallel-filesystems/lustre/ and https://docs.lumi-supercomputer.eu/storage/. A summary of these is the following file management best practices:

   - Try to avoid creating or using large amounts of small files
   - Personal directories are used for credentials, configurations and logs
   - Project directories are used to store software for batch jobs
   - Scratch directories are used to store batch job data

A good rule of thumb, when working with HPC platforms is to consider and recheck your actions against available documentation to confirm that there will not be any effects on the cluster performance. Bad practices can and will effect other cluster users, which is why it is recommended to be more careful with HPC platforms than local and cloud platfroms, where your mistakes only effect you. This need to double check is especially important, when we start to create the automation for HPC interactions, because bad code that runs multiple times a second can slow down services and at worst crash the whole system. For us the most important command for understanding the file system are:

```
# Puhti and mahti filespaces
csc-workspaces
# Lumi filespaces
lumi-workspaces
```

#### Lmod modules

Please read https://docs.csc.fi/computing/modules/ and https://docs.lumi-supercomputer.eu/runjobs/lumi_env/Lmod_modules/ understand how to use Lmod module enviroments. The module enviroments reduces configuration via the use of CSC's pytorch module found here https://docs.csc.fi/apps/pytorch/ for all platforms. The most important commands for us are:

```
# List activated modules
module list

# Load specific module
module load (name)

# Unload specific module
module unload (name)

# LUMI addition
module use /appl/local/csc/modulefiles/ 
# All platfroms
module load pytorch
```

Be aware that for automation there is a difference between the normal interactive SSH terminal connection you create from your local computer and the SSH connection created by Paramiko into the HPC platform. In the latter there is no automatic enviroment configuration that enables Paramiko to use module tools, which is why we need to run extra commands before running module commands. This difference in enviroment activation is the difference between the terminal login shell and non interactive paramiko shell, which you can read about in here https://stackoverflow.com/questions/70415821/what-is-the-equivalent-of-shell-true-in-paramiko. Example of necessery enviroment activation is running the following command first in puhti and mahti:

```
source /appl/profile/zz-csc-env.sh
```

This same method does not work in LUMI and is not universal, which is why we will go later into the details of how .profile, .bashrc and .bash_profile can be used to fix these, when we create DAG automation.

#### Python environments

The use of Ray requires Python, but unfortunately Python venvs create and use multiple small files. This can be prevented by using CSC modules, but unfortunately they don't always contain the necessery packages, which is why the compromise solution is found here https://docs.csc.fi/support/tutorials/python-usage-guide/. With this we can create a module supported venv with:

```
module load python-data
python3 -m venv --system-site-packages <venv_name>
source <venv_name>/bin/activate
pip install whatshap
```

This can be then later activated with:

```
module load python-data
source /projappl/<your_project>/<venv_name>/bin/activate
```

As we can see, we should always prefer available modules before trying to setup anything ourselves. In fact the use of python enviroments is disgouraged in LUMI as seen in here https://docs.lumi-supercomputer.eu/software/installing/python/, where this problem would be solved with apptainer containers. Fortunately for our use case we can continue using the same method to add missing packages. However, if there are a lot of missing packages, apptainer containers should be created to lessen the load on the luster file system.

#### Remote Forward

As shown in https://www.ssh.com/academy/ssh/tunneling-example, we need to SSH remote forward at least the Ray cluster dashboard into CPouta VM using the following command:

```
ssh -f -o StrictHostKeyChecking=no -i $key_path -N \
-R $cloud_private_ip:$cloud_port:$head_node_ip:$hpc_dashboard_port \
$cloud_user@$cloud_public_ip
```

Here the details (check flags from https://linuxcommand.org/lc3_man_pages/ssh1.html) mean:


   - '-f' -> Make ssh run on background
   - '-o' -> Configure connection
   - 'StrictHostKeyChecking=no' -> Enables connections to systems with changed host keys
   - '-N' -> Removes ability to execute remote commands
   - '-R' -> Creates a remote forward connection

These commands can be duplicated for the amount of remote connections we want as long as we give unique ports, which means we can easily forward Ray serve and Prometheus ports as needed. For all intents and purposes we will continue to use port definitions used in part 6.

#### SLURM basics

Please read https://docs.csc.fi/computing/running/getting-started/ and https://docs.lumi-supercomputer.eu/runjobs/scheduled-jobs/slurm-quickstart/ to understand how to use SLURM. SLURM interactions can be divided into:

   - Resources -> Define the requested resources and time
   - Code -> Instructions executed with given resources
   - Commands -> Instructions for the SLURM system

In the case of resources SLURM HPC platforms have usually the same formats with small differences. For our use case the resource template is:

```
#!/bin/bash
#SBATCH --job-name=ray-cluster
#SBATCH --account=project_(your_project_code)
#SBATCH --partition=(suitable_parition)
#SBATCH --time=00:10:00
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=10GB
```

Here we see:

   - job-name -> Given name to the job
   - account -> Your MyCSC project with platform access
   - partition -> Node group with set amount of resources (puhti, mahti and LUMI have unique partitions)
   - time -> Given time for running the job
   - nodes -> Amount of compute nodes
   - ntasks-per-node -> Amount of invoked on node
   - cpus-per-task -> Amount of cpus on task
   - mem -> Amount of available RAM per node

As you can see, batch jobs are usually writen in bash .sh filess with execution instructions on configuration, used software and running code. These instructions are put after the last #SBATCH command in the following way:

```
#SBATCH --mem=10GB

module use /appl/local/csc/modulefiles/
module load pytorch

echo "Loaded modules:"

module list
```

These instructions can have SLURM commands with the most important ones being:

   - scontrol -> Enables checking and modification SLURM configuration and state
   - srun -> Enables running a job in parallel

The actual manual interactions with SLURM are done with:

   - sbatch -> Submit a batch job
   - squeue --me -> Check your submitted batch jobs
   - scancel (job_id) -> Cancel a specific batch job
   - sacct -> Get information about jobs

With these we can do all the necessery SLURM manipulation for our use case. If you are intrested in using SLURM for more that Python applications, please check the material provided by https://github.com/csc-training/summerschool.

#### SLURM Ray

Please check https://docs.ray.io/en/latest/cluster/vms/user-guides/community/slurm.html to understand how Ray can be run using SLURM. For our use case we will create a head-worker cluster with Nodes-1 amount of workers. The setup code is the same in puhti, mahti and LUMI with only module and venv activations creating the following differences:

```
# Puhti and Mahti setup start
module load pytorch
echo "Loaded modules:"
module list

# LUMI setup start
module use /appl/local/csc/modulefiles/
module load pytorch
echo "Loaded modules:"
module list
```

In both cases the following code is:

```
echo "Setting connection variables"

key_path="/users/(your_csc_user)/cpouta-hpc.pem"
cloud_private_ip="(your_vm_private_ip)"
cloud_port=8280
cloud_user="(your_vm_user)"
cloud_public_ip="(your_vm_public_ip)"

echo "Setting Ray variables"

hpc_head_port=8265
hpc_dashboard_port=8280 
nodes=$(scontrol show hostnames "$SLURM_JOB_NODELIST")
nodes_array=($nodes)
head_node=${nodes_array[0]}
head_node_ip=$(srun --nodes=1 --ntasks=1 -w "$head_node" hostname --ip-address)

echo "Setting up Ray head"

ip_head=$head_node_ip:$hpc_head_port
export ip_head
echo "IP Head: $ip_head"

echo "Starting HEAD at $head_node"
srun --nodes=1 --ntasks=1 -w "$head_node" \
    singularity_wrapper exec ray start --head --node-ip-address="$head_node_ip" \
    --port=$hpc_head_port --dashboard-host="$head_node_ip" --dashboard-port=$hpc_dashboard_port \
    --num-cpus "${SLURM_CPUS_PER_TASK}" --block &

echo "Setting up SSH tunnel"

ssh -f -o StrictHostKeyChecking=no -i $key_path -N \
-R $cloud_private_ip:$cloud_port:$head_node_ip:$hpc_dashboard_port \
$cloud_user@$cloud_public_ip

echo "Reverse port forward running"

sleep 5

echo "Setting up Ray workers"

worker_num=$((SLURM_JOB_NUM_NODES - 1))

for ((i = 1; i <= worker_num; i++)); do
    node_i=${nodes_array[$i]}
    echo "Starting WORKER $i at $node_i"
    srun --nodes=1 --ntasks=1 -w "$node_i" \
         singularity_wrapper exec ray start --address "$ip_head" \
	 --num-cpus "${SLURM_CPUS_PER_TASK}" --block &
done
sleep 240
```

In this script you should give special attention to the following:

   - key_path -> Private directory SSH key
   - cloud_private_ip -> CPouta VM internal ip
   - cloud_public_ip -> CPouta VM floating point
   - singularity_wrapper -> Uses apptainer for containerization
   - num-cpus -> Gives Ray cpus
   - sleep -> Keeps the batch job running as long as needed

Be aware that you will need to provide accurate details to ensure proper Ray cluster runs. Also remember to use part 6 to give the correct arguments for num-cpus and num-gpus to ensure that Ray actually uses provided resources. Besides these factors this same script only needs to be copied with the correct details in order to create suitable HPC Ray clusters.

#### SLURM Sacct

Please read https://csc-training.github.io/csc-env-eff/hands-on/batch_resources/tutorial_sacct_and_seff.html and https://slurm.schedmd.com/sacct.html to understand how SLURM Sacct can be used to monitor jobs. In our case we will collect data from jobs in the following way:

```
sacct -o (wanted_fields) -j (job_id)
```

The fields we are intrested in are:

   - JobID -> Batch job number identity
   - JobName -> Batch job name identityt
   - Account -> Used MyCSC account
   - Partition -> Used platform parition
   - ReqCPUS -> Number of requests CPUs
   - AllocCPUs -> Allocated CPUs
   - ReqNodes -> Number of requests nodes
   - AllocNodes -> Allocated nodes
   - State -> State of the batch job
   - AveCPU -> Average CPU time of all tasks
   - AveCPUFreq -> Average CPU frequency in kHz in all tasks
   - AveDiskRead -> Average number of bytes read by all tasks
   - AveDiskWrite -> Average number of bytes written by all tasks
   - Timelimit -> Given time limit of a batch jobs
   - Submit -> Time the job was submitted
   - Start -> Time the job was started
   - Elapsed -> The batch jobs elapsed time
   - Planned -> Amount of wall clock time that was used as planned
   - End -> Time the job was terminated
   - PlannedCPU -> CPU seconds that were used as planned
   - CPUTime -> Used time
   - TotalCPU -> SystemCPU and UserCPU time by the job

These fields enable us to get approximations about the performance of the Ray clusters, which we can later store, analyze and monitor with different OSS applications. The given print would then be stored into a file that is then stored into Allas with the method shown in part 7.

#### General monitoring

Please check https://docs.csc.fi/computing/performance/, https://lumi-supercomputer.github.io/LUMI-EasyBuild-docs/s/seff/, https://docs.csc.fi/support/tutorials/gpu-ml/#puhti, https://docs.lumi-supercomputer.eu/development/profiling/strategies/ and https://docs.lumi-supercomputer.eu/runjobs/lumi_env/dailymanagement/ to understand how general monitoring of batch job runs are done. For our use case, the most important factors to monitor are:

   - Logs
   - Billing units
   - Efficiency

In the case of logs we can simply store the created log files for further debugging by simply downloading them and storing them into Allas in the way shown in part 7. Billing units or efficency unfortunately does not have a standard way. In the case of puhti and mahti the easiest way of getting billing units and some what accurate efficency is with seff:

```
seff (job_id)
```

This will give the following print:

```
Job ID: 3460247
Cluster: mahti
User/Group: (your_user)/pepr_(your_user)
State: CANCELLED (exit code 0)
Nodes: 2
Cores per node: 256
CPU Utilized: 00:05:54
CPU Efficiency: 0.17% of 2-10:10:08 core-walltime
Job Wall-clock time: 00:06:49
Memory Utilized: 2.26 GB
Memory Efficiency: 11.31% of 20.00 GB
Job consumed 22.72 CSC billing units based on following used resources
Billed project: project_(your_project)
Non-Interactive BUs: 22.72
```

In the case of LUMI things get more complicated, because as shown in https://lumi-supercomputer.github.io/LUMI-EasyBuild-docs/s/seff/ seff does not work properly, which is why we need to get the billing unit difference from:

```
lumi-allocations --project project_(your_project)
```

Getting efficiency metrics is even more complicated for LUMI jobs without going into profilers, but https://docs.csc.fi/support/tutorials/gpu-ml/ and https://docs.lumi-supercomputer.eu/runjobs/scheduled-jobs/jobenergy/ show atleast the possibility of energy consumption monitoring, which we will consider later.

## Manual Cloud-HPC Integration

### Useful material

- [LUMI - Extending containers](https://462000265.lumidata.eu/ai-20241126/files/LUMI-ai-20241126-07-Extending_containers.pdf)
- [Setting up your own environment](https://github.com/Lumi-supercomputer/LUMI-AI-Guide/tree/main/2-setting-up-environment#readme)
- [Software on LUMI](https://lumi-supercomputer.github.io/lumi-self-learning/15-software-on-lumi/)
- [Moving AI jobs to LUMI](https://a3s.fi/mats/Moving%20AI%20jobs%20to%20LUMI.pdf)

To begin, let's make connecting to Puhti, Mahti and LUMI easier by configuring .ssh/config in the following way:

```
Host puhti
Hostname puhti.csc.fi
User (your_csc_user)
IdentityFile ~/.ssh/local-hpc.pem

Host mahti
Hostname mahti.csc.fi
User (your_csc_user)
IdentityFile ~/.ssh/local-hpc.pem

Host lumi
Hostname lumi.csc.fi
User (your_csc_user)
IdentityFile ~/.ssh/local-hpc.pem
```

Now, create two separate terminals for interacting with the cloud CPouta VM and HPC platforms. To begin, we first need to use information mentioned in part 6 to create headless services and virtualservices for Puhti, Mahti and LUMI ray clusters. We can use the services, virtual services and gateway hosts are templates to create HPC dashboard, client, metrics and serve forwards in ranges:

   - Ray dashboards (cluster port 8265)
       - 8125-8150
   - Ray client (cluster port 10001)
       - 8176-8200
   - Ray metrics (cluster ports 8500 and 8501)
       - 8251-8300
   - Ray serve (cluster port 8350)
       - 8351-8400

With these details we modify deployments/ray/kubernetes/services, which we deploy with:

```
cd deployments/ray/kubernetes/services,
kubectl apply -k services
```

Now we only need to edit deployments/networking/ray with the following hosts:

- "ray.hpc.dash-1.oss"
- "ray.hpc.serve-1.oss"
- "ray.hpc.dash-2.oss"
- "ray.hpc.serve-2.oss"
- "ray.hpc.dash-3.oss"
- "ray.hpc.serve-3.oss"

We can deploy these changes with:

```
cd deployments/networking
kubectl apply -k http
```

Finally, we now only need to update /etc/hosts with the following:

```
(vm_floating_ip) ray.hpc.dash-1.oss
(vm_floating_ip) ray.hpc.serve-1.oss
(vm_floating_ip) ray.hpc.dash-2.oss
(vm_floating_ip) ray.hpc.serve-2.oss
(vm_floating_ip) ray.hpc.dash-3.oss
(vm_floating_ip) ray.hpc.serve-3.oss
```

We are now ready to setup and run SLURM Ray clusters, which we can start by connecting to puhti, mahti and LUMI:

```
# Connect to puhti
ssh puhti

# Connect to mahti 
ssh mahti

# Connect to LUMI
ssh lumi
```

Check the available projects:

```
# Puhti and mahti
csc-workspaces

# LUMI
lumi-workspaces
```

Select the suitable MyCSC project projappl folder:

```
# Puhti, Mahti and LUMI
cd /projappl/project_(your_csc_project)
```

Check loaded modules:

```
# Puhti, mahti and LUMI
module list
```

Load PyTorch module:

```
# Puhti and mahti
module load pytorch/2.7
# LUMI
module use /appl/local/csc/modulefiles/
module load pytorch/2.7
```

Confirm that the modules are loaded:

```
module list
```

Example puhti print:

```
Currently Loaded Modules:
  1) gcc/11.3.0   2) intel-oneapi-mkl/2022.1.0   3) openmpi/4.1.4   4) csc-tools (S)   5) StdEnv   6) pytorch/2.7

  Where:
   S:  Module is Sticky, requires --force to unload or purge
```

Example mahti print:

```
Currently Loaded Modules:
  1) gcc/11.2.0   2) openmpi/4.1.2   3) openblas/0.3.18-omp   4) csc-tools (S)   5) StdEnv   6) pytorch/2.7

  Where:
   S:  Module is Sticky, requires --force to unload or purge
```

Example LUMI print:

```
Currently Loaded Modules:
  1) craype-x86-rome                        6) cce/17.0.1           11) PrgEnv-cray/8.5.0
  2) libfabric/1.15.2.0                     7) craype/2.7.31.11     12) ModuleLabel/label (S)
  3) craype-network-ofi                     8) cray-dsmml/0.3.0     13) lumi-tools/24.05  (S)
  4) perftools-base/24.03.0                 9) cray-mpich/8.1.29    14) init-lumi/0.2     (S)
  5) xpmem/2.8.2-1.0_5.1__g84a27a5.shasta  10) cray-libsci/24.03.0  15) pytorch/2.7

  Where:
   S:  Module is Sticky, requires --force to unload or purge
```

Create a venv used by run Ray and execute code:

```
python3 -m venv --system-site-packages tutorial-venv
```

We can active the enviroment:

```
source tutorial-venv/bin/activate
```

In the case of puhti and mahti it is recommended to update pip with:

```
pip install --upgrade pip
```

We can then check the available packages with:

```
pip list
```

The example list for puhti is:

```
Package                                  Version
---------------------------------------- -------------------------------------
absl-py                                  1.4.0
accelerate                               1.8.1
affine                                   2.4.0
aiohappyeyeballs                         2.6.1
aiohttp                                  3.12.13
aiohttp-cors                             0.8.1
aiosignal                                1.3.2
airportsdata                             20250622
alembic                                  1.16.2
annotated-types                          0.7.0
ansicolors                               1.1.8
anthropic                                0.55.0
anyio                                    4.9.0
apex                                     0.1
argon2-cffi                              25.1.0
argon2-cffi-bindings                     21.2.0
array_record                             0.7.2
arrow                                    1.3.0
assertpy                                 1.1
astor                                    0.8.1
asttokens                                3.0.0
async-lru                                2.0.5
attrs                                    25.3.0
audioread                                3.0.1
babel                                    2.17.0
beautifulsoup4                           4.13.4
bitsandbytes                             0.46.0
blake3                                   1.0.5
bleach                                   6.2.0
blinker                                  1.9.0
blobfile                                 3.0.0
blosc2                                   3.5.0
cachetools                               5.5.2
certifi                                  2025.6.15
cffi                                     1.17.1
cftime                                   1.6.4.post1
charset-normalizer                       3.4.2
click                                    8.2.1
click-plugins                            1.1.1.2
cligj                                    0.7.2
cloudpickle                              3.1.1
cmake                                    3.31.6
colorama                                 0.4.6
colorful                                 0.5.6
comm                                     0.2.2
compressed-tensors                       0.10.1
contourpy                                1.3.2
cuda-bindings                            12.9.0
cuda-python                              12.9.0
cupy-cuda12x                             13.5.0
cycler                                   0.12.1
Cython                                   3.1.2
dash                                     3.1.0
dask                                     2025.5.1
dask-jobqueue                            0.9.0
databricks-sdk                           0.57.0
datasets                                 3.6.0
debtcollector                            3.0.0
debugpy                                  1.8.14
decorator                                5.2.1
decord                                   0.6.0
deepspeed                                0.17.1
deepspeed-kernels                        0.0.1.dev1698255861
defusedxml                               0.7.1
depyf                                    0.18.0
diffusers                                0.34.0
dill                                     0.3.8
diskcache                                5.6.3
distlib                                  0.3.9
distributed                              2025.5.1
distro                                   1.9.0
dm-tree                                  0.1.9
dnspython                                2.7.0
docker                                   7.1.0
docstring_parser                         0.16
einops                                   0.8.1
email_validator                          2.2.0
entrypoints                              0.4
et_xmlfile                               2.0.0
etils                                    1.12.2
evaluate                                 0.4.4
executing                                2.2.0
faiss                                    1.11.0
fastapi                                  0.115.14
fastapi-cli                              0.0.7
fastargs                                 1.2.0
fastjsonschema                           2.21.1
fastrlock                                0.8.3
ffcv                                     1.0.2
filelock                                 3.18.0
flash_attn                               2.8.0.post2
flashinfer-python                        0.2.6.post1
Flask                                    3.1.1
fonttools                                4.58.4
fqdn                                     1.5.1
frozenlist                               1.7.0
fsspec                                   2025.3.0
fvcore                                   0.1.5.post20221221
gensim                                   4.3.3
geopandas                                1.1.1
gguf                                     0.17.1
gitdb                                    4.0.12
GitPython                                3.1.44
google-api-core                          2.25.1
google-auth                              2.40.3
googleapis-common-protos                 1.70.0
gpytorch                                 1.14
graphene                                 3.4.3
graphql-core                             3.2.6
graphql-relay                            3.2.0
graphviz                                 0.21
greenlet                                 3.2.3
grpcio                                   1.73.1
gunicorn                                 23.0.0
gym                                      0.26.2
gym-notices                              0.0.8
h11                                      0.16.0
h5py                                     3.14.0
hf_transfer                              0.1.9
hf-xet                                   1.1.5
hjson                                    3.1.0
httpcore                                 1.0.9
httptools                                0.6.4
httpx                                    0.28.1
huggingface-hub                          0.33.1
idna                                     3.10
imageio                                  2.37.0
imbalanced-learn                         0.13.0
immutabledict                            4.2.1
importlib_metadata                       8.7.0
importlib_resources                      6.5.2
iniconfig                                2.1.0
interegular                              0.3.3
iopath                                   0.1.10
ipykernel                                6.29.5
ipython                                  9.3.0
ipython-genutils                         0.2.0
ipython_pygments_lexers                  1.1.1
ipywidgets                               8.1.7
iso8601                                  2.1.0
isoduration                              20.11.0
itsdangerous                             2.2.0
jaxtyping                                0.3.2
jedi                                     0.19.2
Jinja2                                   3.1.6
jiter                                    0.10.0
joblib                                   1.5.1
json5                                    0.12.0
jsonpatch                                1.33
jsonpointer                              3.0.0
jsonschema                               4.24.0
jsonschema-specifications                2025.4.1
jupyter_client                           8.6.3
jupyter_core                             5.8.1
jupyter-events                           0.12.0
jupyter-lsp                              2.2.5
jupyter_server                           2.16.0
jupyter-server-mathjax                   0.2.6
jupyter_server_proxy                     4.4.0
jupyter_server_terminals                 0.5.3
jupyterlab                               4.4.3
jupyterlab-dash                          0.1.0a3
jupyterlab_git                           0.51.2
jupyterlab_pygments                      0.3.0
jupyterlab_server                        2.27.3
jupyterlab_widgets                       3.0.15
jupytext                                 1.17.2
kagglehub                                0.3.12
keopscore                                2.3
keras                                    3.10.0
keras-core                               0.1.7
keras-cv                                 0.9.0
keystoneauth1                            5.11.1
kiwisolver                               1.4.8
lark                                     1.2.2
lazy_loader                              0.4
librosa                                  0.11.0
lightning                                2.5.2
lightning-utilities                      0.14.3
linear-operator                          0.6
lit                                      18.1.8
litellm                                  1.73.2
llguidance                               0.7.30
llvmlite                                 0.44.0
lm-format-enforcer                       0.10.11
lmdb                                     1.6.2
locket                                   1.0.0
lxml                                     6.0.0
Mako                                     1.3.10
Markdown                                 3.8.2
markdown-it-py                           3.0.0
MarkupSafe                               3.0.2
matplotlib                               3.10.3
matplotlib-inline                        0.1.7
mdit-py-plugins                          0.4.2
mdurl                                    0.1.2
mistral_common                           1.6.3
mistune                                  3.1.3
ml_dtypes                                0.5.1
mlflow                                   3.1.1
mlflow-skinny                            3.1.1
modelscope                               1.27.1
mpi4py                                   4.1.0
mpmath                                   1.3.0
msgpack                                  1.1.1
msgspec                                  0.19.0
multidict                                6.5.1
multiprocess                             0.70.16
mysql-connector-python                   9.3.0
namex                                    0.1.0
narwhals                                 1.44.0
nbclassic                                1.3.1
nbclient                                 0.10.2
nbconvert                                7.16.6
nbdime                                   4.0.2
nbformat                                 5.10.4
ndindex                                  1.10.0
nest-asyncio                             1.6.0
netaddr                                  1.3.0
netCDF4                                  1.7.2
networkx                                 3.5
ninja                                    1.11.1.4
nltk                                     3.9.1
notebook                                 7.4.3
notebook_shim                            0.2.4
numba                                    0.61.2
numexpr                                  2.11.0
numpy                                    1.26.4
nvidia-cublas-cu12                       12.6.4.1
nvidia-cuda-cupti-cu12                   12.6.80
nvidia-cuda-nvrtc-cu12                   12.6.77
nvidia-cuda-runtime-cu12                 12.6.77
nvidia-cudnn-cu12                        9.5.1.17
nvidia-cufft-cu12                        11.3.0.4
nvidia-cufile-cu12                       1.11.1.6
nvidia-curand-cu12                       10.3.7.77
nvidia-cusolver-cu12                     11.7.1.2
nvidia-cusparse-cu12                     12.5.4.2
nvidia-cusparselt-cu12                   0.6.3
nvidia-ml-py                             12.575.51
nvidia-nccl-cu12                         2.26.2
nvidia-nvjitlink-cu12                    12.6.85
nvidia-nvtx-cu12                         12.6.77
odfpy                                    1.4.1
openai                                   1.92.2
opencensus                               0.11.4
opencensus-context                       0.1.3
opencv-python                            4.11.0.86
opencv-python-headless                   4.11.0.86
openpyxl                                 3.1.5
opentelemetry-api                        1.34.1
opentelemetry-exporter-otlp              1.34.1
opentelemetry-exporter-otlp-proto-common 1.34.1
opentelemetry-exporter-otlp-proto-grpc   1.34.1
opentelemetry-exporter-otlp-proto-http   1.34.1
opentelemetry-exporter-prometheus        0.55b1
opentelemetry-proto                      1.34.1
opentelemetry-sdk                        1.34.1
opentelemetry-semantic-conventions       0.55b1
opentelemetry-semantic-conventions-ai    0.4.10
optree                                   0.16.0
orjson                                   3.10.18
os-service-types                         1.7.0
oslo.config                              9.8.0
oslo.i18n                                6.5.1
oslo.serialization                       5.7.0
oslo.utils                               9.0.0
outlines                                 0.1.11
outlines_core                            0.1.26
overrides                                7.7.0
packaging                                25.0
pandas                                   2.3.0
pandocfilters                            1.5.1
papermill                                2.6.0
parso                                    0.8.4
partd                                    1.4.2
partial-json-parser                      0.2.1.1.post6
pbr                                      6.1.1
peft                                     0.15.2
pexpect                                  4.9.0
pillow                                   11.2.1
pip                                      25.3
platformdirs                             4.3.8
plotly                                   6.2.0
pluggy                                   1.6.0
pooch                                    1.8.2
portalocker                              3.2.0
prometheus_client                        0.22.1
prometheus-fastapi-instrumentator        7.1.0
promise                                  2.3
prompt_toolkit                           3.0.51
propcache                                0.3.2
proto-plus                               1.26.1
protobuf                                 5.29.5
psutil                                   7.0.0
ptyprocess                               0.7.0
pure_eval                                0.2.3
py-cpuinfo                               9.0.0
py-spy                                   0.4.0
pyarrow                                  20.0.0
pyasn1                                   0.6.1
pyasn1_modules                           0.4.2
pybind11                                 2.13.6
pycodestyle                              2.14.0
pycountry                                24.6.1
pycparser                                2.22
pycryptodomex                            3.23.0
pydantic                                 2.11.7
pydantic_core                            2.33.2
pydot                                    4.0.1
pyflakes                                 3.4.0
pyglet                                   1.5.15
Pygments                                 2.19.2
pykeops                                  2.3
pynndescent                              0.5.13
pynvml                                   12.0.0
pyogrio                                  0.11.0
pyparsing                                3.2.3
pyproj                                   3.7.1
pyprojroot                               0.3.0
pysqlite3                                0.5.4
pytest                                   8.4.1
python-dateutil                          2.9.0.post0
python-dotenv                            1.1.1
python-json-logger                       3.3.0
python-keystoneclient                    5.6.0
python-multipart                         0.0.20
python-swiftclient                       4.8.0
pytorch-lightning                        2.5.2
pytorch-pfn-extras                       0.8.3
pytz                                     2025.2
PyYAML                                   6.0.2
pyzmq                                    27.0.0
rasterio                                 1.4.3
ray                                      2.47.1
referencing                              0.36.2
regex                                    2024.11.6
requests                                 2.32.4
retrying                                 1.4.0
rfc3339-validator                        0.1.4
rfc3986                                  2.0.0
rfc3986-validator                        0.1.1
rich                                     14.0.0
rich-toolkit                             0.14.8
rpds-py                                  0.25.1
rsa                                      4.9.1
safetensors                              0.5.3
scikit-image                             0.25.2
scikit-learn                             1.6.1
scipy                                    1.13.1
seaborn                                  0.13.2
Send2Trash                               1.8.3
sentencepiece                            0.2.0
sentry-sdk                               2.32.0
setproctitle                             1.3.6
setuptools                               79.0.1
setuptools-scm                           8.3.1
sgl-kernel                               0.1.9
sglang                                   0.4.8.post1
shapely                                  2.1.1
shellingham                              1.5.4
simpervisor                              1.0.0
simple-parsing                           0.1.7
six                                      1.17.0
sklearn-compat                           0.1.3
smart-open                               7.1.0
smmap                                    5.0.2
sniffio                                  1.3.1
sortedcontainers                         2.4.0
soundfile                                0.13.1
soupsieve                                2.7
soxr                                     0.5.0.post1
SQLAlchemy                               2.0.41
sqlparse                                 0.5.3
stack-data                               0.6.3
starlette                                0.46.2
stevedore                                5.4.1
sympy                                    1.14.0
tables                                   3.10.2
tabulate                                 0.9.0
tblib                                    3.1.0
tenacity                                 9.1.2
tensorboard                              2.19.0
tensorboard-data-server                  0.7.2
tensorboardX                             2.6.4
tensorflow-datasets                      4.9.9
tensorflow-metadata                      1.14.0
termcolor                                3.1.0
terminado                                0.18.1
terminaltables                           3.1.10
threadpoolctl                            3.6.0
tifffile                                 2025.6.11
tiktoken                                 0.9.0
timm                                     1.0.16
tinycss2                                 1.4.0
tokenizers                               0.21.2
toml                                     0.10.2
toolz                                    1.0.0
torch                                    2.7.1+cu126
torch-geometric                          2.6.1
torch_memory_saver                       0.0.8
torch-tb-profiler                        0.4.3
torchao                                  0.9.0
torchaudio                               2.7.1+cu126
torchinfo                                1.8.0
torchmetrics                             1.7.3
torchvision                              0.22.1+cu126
tornado                                  6.5.1
tqdm                                     4.67.1
traitlets                                5.14.3
transformers                             4.53.1
triton                                   3.3.1
trl                                      0.19.0
typer                                    0.16.0
types-python-dateutil                    2.9.0.20250516
typing_extensions                        4.14.0
typing-inspection                        0.4.1
tzdata                                   2025.2
umap-learn                               0.5.7
uri-template                             1.3.0
urllib3                                  2.5.0
uvicorn                                  0.34.3
uvloop                                   0.21.0
virtualenv                               20.31.2
visdom                                   0.2.4
vllm                                     0.9.2.dev0+gb6553be1b.d20250703.cu126
wadler_lindig                            0.1.7
wandb                                    0.20.1
watchfiles                               1.1.0
wcwidth                                  0.2.13
webcolors                                24.11.1
webencodings                             0.5.1
websocket-client                         1.8.0
websockets                               15.0.1
Werkzeug                                 3.1.3
wheel                                    0.45.1
widgetsnbextension                       4.0.14
wrapt                                    1.17.2
xformers                                 0.0.31
xgboost                                  3.0.2
xgrammar                                 0.1.19
xlwt                                     1.3.0
xxhash                                   3.5.0
yacs                                     0.1.8
yarl                                     1.20.1
zict                                     3.0.0
zipp                                     3.23.0
```

The example print for mahti is:

```
Package                                  Version
---------------------------------------- -------------------------------------
absl-py                                  1.4.0
accelerate                               1.8.1
affine                                   2.4.0
aiohappyeyeballs                         2.6.1
aiohttp                                  3.12.13
aiohttp-cors                             0.8.1
aiosignal                                1.3.2
airportsdata                             20250622
alembic                                  1.16.2
annotated-types                          0.7.0
ansicolors                               1.1.8
anthropic                                0.55.0
anyio                                    4.9.0
apex                                     0.1
argon2-cffi                              25.1.0
argon2-cffi-bindings                     21.2.0
array_record                             0.7.2
arrow                                    1.3.0
assertpy                                 1.1
astor                                    0.8.1
asttokens                                3.0.0
async-lru                                2.0.5
attrs                                    25.3.0
audioread                                3.0.1
babel                                    2.17.0
beautifulsoup4                           4.13.4
bitsandbytes                             0.46.0
blake3                                   1.0.5
bleach                                   6.2.0
blinker                                  1.9.0
blobfile                                 3.0.0
blosc2                                   3.5.0
cachetools                               5.5.2
certifi                                  2025.6.15
cffi                                     1.17.1
cftime                                   1.6.4.post1
charset-normalizer                       3.4.2
click                                    8.2.1
click-plugins                            1.1.1.2
cligj                                    0.7.2
cloudpickle                              3.1.1
cmake                                    3.31.6
colorama                                 0.4.6
colorful                                 0.5.6
comm                                     0.2.2
compressed-tensors                       0.10.1
contourpy                                1.3.2
cuda-bindings                            12.9.0
cuda-python                              12.9.0
cupy-cuda12x                             13.5.0
cycler                                   0.12.1
Cython                                   3.1.2
dash                                     3.1.0
dask                                     2025.5.1
dask-jobqueue                            0.9.0
databricks-sdk                           0.57.0
datasets                                 3.6.0
debtcollector                            3.0.0
debugpy                                  1.8.14
decorator                                5.2.1
decord                                   0.6.0
deepspeed                                0.17.1
deepspeed-kernels                        0.0.1.dev1698255861
defusedxml                               0.7.1
depyf                                    0.18.0
diffusers                                0.34.0
dill                                     0.3.8
diskcache                                5.6.3
distlib                                  0.3.9
distributed                              2025.5.1
distro                                   1.9.0
dm-tree                                  0.1.9
dnspython                                2.7.0
docker                                   7.1.0
docstring_parser                         0.16
einops                                   0.8.1
email_validator                          2.2.0
entrypoints                              0.4
et_xmlfile                               2.0.0
etils                                    1.12.2
evaluate                                 0.4.4
executing                                2.2.0
faiss                                    1.11.0
fastapi                                  0.115.14
fastapi-cli                              0.0.7
fastargs                                 1.2.0
fastjsonschema                           2.21.1
fastrlock                                0.8.3
ffcv                                     1.0.2
filelock                                 3.18.0
flash_attn                               2.8.0.post2
flashinfer-python                        0.2.6.post1
Flask                                    3.1.1
fonttools                                4.58.4
fqdn                                     1.5.1
frozenlist                               1.7.0
fsspec                                   2025.3.0
fvcore                                   0.1.5.post20221221
gensim                                   4.3.3
geopandas                                1.1.1
gguf                                     0.17.1
gitdb                                    4.0.12
GitPython                                3.1.44
google-api-core                          2.25.1
google-auth                              2.40.3
googleapis-common-protos                 1.70.0
gpytorch                                 1.14
graphene                                 3.4.3
graphql-core                             3.2.6
graphql-relay                            3.2.0
graphviz                                 0.21
greenlet                                 3.2.3
grpcio                                   1.73.1
gunicorn                                 23.0.0
gym                                      0.26.2
gym-notices                              0.0.8
h11                                      0.16.0
h5py                                     3.14.0
hf_transfer                              0.1.9
hf-xet                                   1.1.5
hjson                                    3.1.0
httpcore                                 1.0.9
httptools                                0.6.4
httpx                                    0.28.1
huggingface-hub                          0.33.1
idna                                     3.10
imageio                                  2.37.0
imbalanced-learn                         0.13.0
immutabledict                            4.2.1
importlib_metadata                       8.7.0
importlib_resources                      6.5.2
iniconfig                                2.1.0
interegular                              0.3.3
iopath                                   0.1.10
ipykernel                                6.29.5
ipython                                  9.3.0
ipython-genutils                         0.2.0
ipython_pygments_lexers                  1.1.1
ipywidgets                               8.1.7
iso8601                                  2.1.0
isoduration                              20.11.0
itsdangerous                             2.2.0
jaxtyping                                0.3.2
jedi                                     0.19.2
Jinja2                                   3.1.6
jiter                                    0.10.0
joblib                                   1.5.1
json5                                    0.12.0
jsonpatch                                1.33
jsonpointer                              3.0.0
jsonschema                               4.24.0
jsonschema-specifications                2025.4.1
jupyter_client                           8.6.3
jupyter_core                             5.8.1
jupyter-events                           0.12.0
jupyter-lsp                              2.2.5
jupyter_server                           2.16.0
jupyter-server-mathjax                   0.2.6
jupyter_server_proxy                     4.4.0
jupyter_server_terminals                 0.5.3
jupyterlab                               4.4.3
jupyterlab-dash                          0.1.0a3
jupyterlab_git                           0.51.2
jupyterlab_pygments                      0.3.0
jupyterlab_server                        2.27.3
jupyterlab_widgets                       3.0.15
jupytext                                 1.17.2
kagglehub                                0.3.12
keopscore                                2.3
keras                                    3.10.0
keras-core                               0.1.7
keras-cv                                 0.9.0
keystoneauth1                            5.11.1
kiwisolver                               1.4.8
lark                                     1.2.2
lazy_loader                              0.4
librosa                                  0.11.0
lightning                                2.5.2
lightning-utilities                      0.14.3
linear-operator                          0.6
lit                                      18.1.8
litellm                                  1.73.2
llguidance                               0.7.30
llvmlite                                 0.44.0
lm-format-enforcer                       0.10.11
lmdb                                     1.6.2
locket                                   1.0.0
lxml                                     6.0.0
Mako                                     1.3.10
Markdown                                 3.8.2
markdown-it-py                           3.0.0
MarkupSafe                               3.0.2
matplotlib                               3.10.3
matplotlib-inline                        0.1.7
mdit-py-plugins                          0.4.2
mdurl                                    0.1.2
mistral_common                           1.6.3
mistune                                  3.1.3
ml_dtypes                                0.5.1
mlflow                                   3.1.1
mlflow-skinny                            3.1.1
modelscope                               1.27.1
mpi4py                                   4.1.0
mpmath                                   1.3.0
msgpack                                  1.1.1
msgspec                                  0.19.0
multidict                                6.5.1
multiprocess                             0.70.16
mysql-connector-python                   9.3.0
namex                                    0.1.0
narwhals                                 1.44.0
nbclassic                                1.3.1
nbclient                                 0.10.2
nbconvert                                7.16.6
nbdime                                   4.0.2
nbformat                                 5.10.4
ndindex                                  1.10.0
nest-asyncio                             1.6.0
netaddr                                  1.3.0
netCDF4                                  1.7.2
networkx                                 3.5
ninja                                    1.11.1.4
nltk                                     3.9.1
notebook                                 7.4.3
notebook_shim                            0.2.4
numba                                    0.61.2
numexpr                                  2.11.0
numpy                                    1.26.4
nvidia-cublas-cu12                       12.6.4.1
nvidia-cuda-cupti-cu12                   12.6.80
nvidia-cuda-nvrtc-cu12                   12.6.77
nvidia-cuda-runtime-cu12                 12.6.77
nvidia-cudnn-cu12                        9.5.1.17
nvidia-cufft-cu12                        11.3.0.4
nvidia-cufile-cu12                       1.11.1.6
nvidia-curand-cu12                       10.3.7.77
nvidia-cusolver-cu12                     11.7.1.2
nvidia-cusparse-cu12                     12.5.4.2
nvidia-cusparselt-cu12                   0.6.3
nvidia-ml-py                             12.575.51
nvidia-nccl-cu12                         2.26.2
nvidia-nvjitlink-cu12                    12.6.85
nvidia-nvtx-cu12                         12.6.77
odfpy                                    1.4.1
openai                                   1.92.2
opencensus                               0.11.4
opencensus-context                       0.1.3
opencv-python                            4.11.0.86
opencv-python-headless                   4.11.0.86
openpyxl                                 3.1.5
opentelemetry-api                        1.34.1
opentelemetry-exporter-otlp              1.34.1
opentelemetry-exporter-otlp-proto-common 1.34.1
opentelemetry-exporter-otlp-proto-grpc   1.34.1
opentelemetry-exporter-otlp-proto-http   1.34.1
opentelemetry-exporter-prometheus        0.55b1
opentelemetry-proto                      1.34.1
opentelemetry-sdk                        1.34.1
opentelemetry-semantic-conventions       0.55b1
opentelemetry-semantic-conventions-ai    0.4.10
optree                                   0.16.0
orjson                                   3.10.18
os-service-types                         1.7.0
oslo.config                              9.8.0
oslo.i18n                                6.5.1
oslo.serialization                       5.7.0
oslo.utils                               9.0.0
outlines                                 0.1.11
outlines_core                            0.1.26
overrides                                7.7.0
packaging                                25.0
pandas                                   2.3.0
pandocfilters                            1.5.1
papermill                                2.6.0
parso                                    0.8.4
partd                                    1.4.2
partial-json-parser                      0.2.1.1.post6
pbr                                      6.1.1
peft                                     0.15.2
pexpect                                  4.9.0
pillow                                   11.2.1
pip                                      25.3
platformdirs                             4.3.8
plotly                                   6.2.0
pluggy                                   1.6.0
pooch                                    1.8.2
portalocker                              3.2.0
prometheus_client                        0.22.1
prometheus-fastapi-instrumentator        7.1.0
promise                                  2.3
prompt_toolkit                           3.0.51
propcache                                0.3.2
proto-plus                               1.26.1
protobuf                                 5.29.5
psutil                                   7.0.0
ptyprocess                               0.7.0
pure_eval                                0.2.3
py-cpuinfo                               9.0.0
py-spy                                   0.4.0
pyarrow                                  20.0.0
pyasn1                                   0.6.1
pyasn1_modules                           0.4.2
pybind11                                 2.13.6
pycodestyle                              2.14.0
pycountry                                24.6.1
pycparser                                2.22
pycryptodomex                            3.23.0
pydantic                                 2.11.7
pydantic_core                            2.33.2
pydot                                    4.0.1
pyflakes                                 3.4.0
pyglet                                   1.5.15
Pygments                                 2.19.2
pykeops                                  2.3
pynndescent                              0.5.13
pynvml                                   12.0.0
pyogrio                                  0.11.0
pyparsing                                3.2.3
pyproj                                   3.7.1
pyprojroot                               0.3.0
pysqlite3                                0.5.4
pytest                                   8.4.1
python-dateutil                          2.9.0.post0
python-dotenv                            1.1.1
python-json-logger                       3.3.0
python-keystoneclient                    5.6.0
python-multipart                         0.0.20
python-swiftclient                       4.8.0
pytorch-lightning                        2.5.2
pytorch-pfn-extras                       0.8.3
pytz                                     2025.2
PyYAML                                   6.0.2
pyzmq                                    27.0.0
rasterio                                 1.4.3
ray                                      2.47.1
referencing                              0.36.2
regex                                    2024.11.6
requests                                 2.32.4
retrying                                 1.4.0
rfc3339-validator                        0.1.4
rfc3986                                  2.0.0
rfc3986-validator                        0.1.1
rich                                     14.0.0
rich-toolkit                             0.14.8
rpds-py                                  0.25.1
rsa                                      4.9.1
safetensors                              0.5.3
scikit-image                             0.25.2
scikit-learn                             1.6.1
scipy                                    1.13.1
seaborn                                  0.13.2
Send2Trash                               1.8.3
sentencepiece                            0.2.0
sentry-sdk                               2.32.0
setproctitle                             1.3.6
setuptools                               79.0.1
setuptools-scm                           8.3.1
sgl-kernel                               0.1.9
sglang                                   0.4.8.post1
shapely                                  2.1.1
shellingham                              1.5.4
simpervisor                              1.0.0
simple-parsing                           0.1.7
six                                      1.17.0
sklearn-compat                           0.1.3
smart-open                               7.1.0
smmap                                    5.0.2
sniffio                                  1.3.1
sortedcontainers                         2.4.0
soundfile                                0.13.1
soupsieve                                2.7
soxr                                     0.5.0.post1
SQLAlchemy                               2.0.41
sqlparse                                 0.5.3
stack-data                               0.6.3
starlette                                0.46.2
stevedore                                5.4.1
sympy                                    1.14.0
tables                                   3.10.2
tabulate                                 0.9.0
tblib                                    3.1.0
tenacity                                 9.1.2
tensorboard                              2.19.0
tensorboard-data-server                  0.7.2
tensorboardX                             2.6.4
tensorflow-datasets                      4.9.9
tensorflow-metadata                      1.14.0
termcolor                                3.1.0
terminado                                0.18.1
terminaltables                           3.1.10
threadpoolctl                            3.6.0
tifffile                                 2025.6.11
tiktoken                                 0.9.0
timm                                     1.0.16
tinycss2                                 1.4.0
tokenizers                               0.21.2
toml                                     0.10.2
toolz                                    1.0.0
torch                                    2.7.1+cu126
torch-geometric                          2.6.1
torch_memory_saver                       0.0.8
torch-tb-profiler                        0.4.3
torchao                                  0.9.0
torchaudio                               2.7.1+cu126
torchinfo                                1.8.0
torchmetrics                             1.7.3
torchvision                              0.22.1+cu126
tornado                                  6.5.1
tqdm                                     4.67.1
traitlets                                5.14.3
transformers                             4.53.1
triton                                   3.3.1
trl                                      0.19.0
typer                                    0.16.0
types-python-dateutil                    2.9.0.20250516
typing_extensions                        4.14.0
typing-inspection                        0.4.1
tzdata                                   2025.2
umap-learn                               0.5.7
uri-template                             1.3.0
urllib3                                  2.5.0
uvicorn                                  0.34.3
uvloop                                   0.21.0
virtualenv                               20.31.2
visdom                                   0.2.4
vllm                                     0.9.2.dev0+gb6553be1b.d20250703.cu126
wadler_lindig                            0.1.7
wandb                                    0.20.1
watchfiles                               1.1.0
wcwidth                                  0.2.13
webcolors                                24.11.1
webencodings                             0.5.1
websocket-client                         1.8.0
websockets                               15.0.1
Werkzeug                                 3.1.3
wheel                                    0.45.1
widgetsnbextension                       4.0.14
wrapt                                    1.17.2
xformers                                 0.0.31
xgboost                                  3.0.2
xgrammar                                 0.1.19
xlwt                                     1.3.0
xxhash                                   3.5.0
yacs                                     0.1.8
yarl                                     1.20.1
zict                                     3.0.0
zipp                                     3.23.0
```

The example print for LUMI is:

```
Package                                  Version
---------------------------------------- -------------------------
absl-py                                  1.4.0
accelerate                               1.7.0
affine                                   2.4.0
aiohappyeyeballs                         2.6.1
aiohttp                                  3.11.18
aiohttp-cors                             0.8.1
aiosignal                                1.3.2
airportsdata                             20250224
alembic                                  1.15.2
amdsmi                                   24.6.3+9578815
annotated-types                          0.7.0
ansicolors                               1.1.8
anyio                                    4.9.0
apex                                     1.7.0a0
argon2-cffi                              23.1.0
argon2-cffi-bindings                     21.2.0
array_record                             0.7.2
arrow                                    1.3.0
astor                                    0.8.1
asttokens                                3.0.0
async-lru                                2.0.5
async-timeout                            5.0.1
attrs                                    25.3.0
audioread                                3.0.1
autocommand                              2.2.2
awscli                                   1.40.21
babel                                    2.17.0
backports.tarfile                        1.2.0
beautifulsoup4                           4.13.4
bitsandbytes                             0.43.3.dev0
blake3                                   1.0.5
bleach                                   6.2.0
blinker                                  1.9.0
blosc2                                   3.3.3
boto3                                    1.38.22
botocore                                 1.38.22
cachetools                               5.5.2
certifi                                  2025.4.26
cffi                                     1.17.1
cftime                                   1.6.4.post1
charset-normalizer                       3.4.2
clang                                    20.1.5
click                                    8.1.8
click-plugins                            1.1.1
cligj                                    0.7.2
cloudpickle                              3.1.1
cmake                                    4.0.2
colorama                                 0.4.6
colorful                                 0.5.6
comm                                     0.2.2
compressed-tensors                       0.10.1
contourpy                                1.3.2
cycler                                   0.12.1
Cython                                   3.1.1
dask                                     2025.5.1
dask-jobqueue                            0.9.0
databricks-sdk                           0.53.0
datasets                                 3.6.0
debtcollector                            3.0.0
debugpy                                  1.8.14
decorator                                5.2.1
deepspeed                                0.17.1+2ce55057
defusedxml                               0.7.1
Deprecated                               1.2.18
depyf                                    0.18.0
diffusers                                0.33.1
dill                                     0.3.8
diskcache                                5.6.3
distlib                                  0.3.9
distributed                              2025.5.1
distro                                   1.9.0
dm-tree                                  0.1.9
dnspython                                2.7.0
docker                                   7.1.0
docstring_parser                         0.16
docutils                                 0.19
einops                                   0.8.1
email_validator                          2.2.0
entrypoints                              0.4
et_xmlfile                               2.0.0
etils                                    1.12.2
evaluate                                 0.4.3
executing                                2.2.0
faiss                                    1.11.0
fastapi                                  0.115.12
fastapi-cli                              0.0.7
fastjsonschema                           2.21.1
filelock                                 3.18.0
flash-attn                               2.7.4.post1
Flask                                    3.1.1
fonttools                                4.58.0
fqdn                                     1.5.1
frozenlist                               1.6.0
fsspec                                   2025.3.0
fvcore                                   0.1.5.post20221221
gensim                                   4.3.3
geopandas                                1.0.1
gguf                                     0.16.3
gitdb                                    4.0.12
GitPython                                3.1.44
google-api-core                          2.24.2
google-auth                              2.40.1
googleapis-common-protos                 1.70.0
gpytorch                                 1.14
graphene                                 3.4.3
graphql-core                             3.2.6
graphql-relay                            3.2.0
graphviz                                 0.20.3
greenlet                                 3.2.2
grpcio                                   1.71.0
gunicorn                                 23.0.0
gym                                      0.26.2
gym-notices                              0.0.8
h11                                      0.16.0
h5py                                     3.13.0
hf_transfer                              0.1.9
hf-xet                                   1.1.2
hiredis                                  3.1.1
hjson                                    3.1.0
httpcore                                 1.0.9
httptools                                0.6.4
httpx                                    0.28.1
huggingface-hub                          0.33.0
humanize                                 4.12.3
idna                                     3.10
imageio                                  2.37.0
imbalanced-learn                         0.13.0
immutabledict                            4.2.1
importlib_metadata                       8.0.0
importlib_resources                      6.5.2
inflect                                  7.3.1
iniconfig                                2.1.0
inquirerpy                               0.3.4
interegular                              0.3.3
iopath                                   0.1.10
ipykernel                                6.29.5
ipython                                  9.2.0
ipython-genutils                         0.2.0
ipython_pygments_lexers                  1.1.1
ipywidgets                               8.1.7
iso8601                                  2.1.0
isoduration                              20.11.0
itsdangerous                             2.2.0
jaraco.collections                       5.1.0
jaraco.context                           5.3.0
jaraco.functools                         4.0.1
jaraco.text                              3.12.1
jaxtyping                                0.3.2
jedi                                     0.19.2
Jinja2                                   3.1.6
jiter                                    0.10.0
jmespath                                 1.0.1
joblib                                   1.5.0
json5                                    0.12.0
jsonpatch                                1.33
jsonpointer                              3.0.0
jsonschema                               4.23.0
jsonschema-specifications                2025.4.1
jupyter_client                           8.6.3
jupyter_core                             5.7.2
jupyter-events                           0.12.0
jupyter-lsp                              2.2.5
jupyter_server                           2.16.0
jupyter-server-mathjax                   0.2.6
jupyter_server_terminals                 0.5.3
jupyterlab                               4.4.2
jupyterlab_git                           0.51.1
jupyterlab_pygments                      0.3.0
jupyterlab_server                        2.27.3
jupyterlab_widgets                       3.0.15
jupytext                                 1.17.1
kagglehub                                0.3.12
keopscore                                2.3
keras                                    3.10.0
keras-core                               0.1.7
keras-cv                                 0.9.0
keystoneauth1                            5.11.0
kiwisolver                               1.4.8
lark                                     1.2.2
lazy_loader                              0.4
libnacl                                  2.1.0
librosa                                  0.11.0
lightning                                2.5.1.post0
lightning-utilities                      0.14.3
linear-operator                          0.6
lion-pytorch                             0.2.3
lit                                      18.1.8
llguidance                               0.7.21
llvmlite                                 0.44.0
lm-format-enforcer                       0.10.11
lmdb                                     1.6.2
locket                                   1.0.0
lxml                                     5.4.0
Mako                                     1.3.10
Markdown                                 3.8
markdown-it-py                           3.0.0
MarkupSafe                               3.0.2
matplotlib                               3.9.4
matplotlib-inline                        0.1.7
mdit-py-plugins                          0.4.2
mdurl                                    0.1.2
mistral_common                           1.5.5
mistune                                  3.1.3
ml_dtypes                                0.5.1
mlflow                                   2.22.0
mlflow-skinny                            2.22.0
more-itertools                           10.3.0
mpi4py                                   4.0.3
mpmath                                   1.3.0
msgpack                                  1.1.0
msgspec                                  0.19.0
multidict                                6.4.4
multiprocess                             0.70.16
mysql-connector-python                   9.3.0
namex                                    0.0.9
nbclassic                                1.3.1
nbclient                                 0.10.2
nbconvert                                7.16.6
nbdime                                   4.0.2
nbformat                                 5.10.4
ndindex                                  1.9.2
nest-asyncio                             1.6.0
netaddr                                  1.3.0
netCDF4                                  1.7.2
networkx                                 3.4.2
ninja                                    1.11.1.4
nltk                                     3.9.1
notebook                                 7.4.2
notebook_shim                            0.2.4
numba                                    0.61.2
numexpr                                  2.10.2
numpy                                    1.26.4
nvidia-nccl-cu12                         2.26.5
odfpy                                    1.4.1
openai                                   1.79.0
opencensus                               0.11.4
opencensus-context                       0.1.3
opencv-python                            4.11.0.86
opencv-python-headless                   4.11.0.86
openpyxl                                 3.1.5
opentelemetry-api                        1.26.0
opentelemetry-exporter-otlp              1.26.0
opentelemetry-exporter-otlp-proto-common 1.26.0
opentelemetry-exporter-otlp-proto-grpc   1.26.0
opentelemetry-exporter-otlp-proto-http   1.26.0
opentelemetry-proto                      1.26.0
opentelemetry-sdk                        1.26.0
opentelemetry-semantic-conventions       0.47b0
opentelemetry-semantic-conventions-ai    0.4.9
optree                                   0.15.0
os-service-types                         1.7.0
oslo.config                              9.8.0
oslo.i18n                                6.5.1
oslo.serialization                       5.7.0
oslo.utils                               9.0.0
outlines                                 0.1.11
outlines_core                            0.1.26
overrides                                7.7.0
packaging                                24.2
pandas                                   2.2.3
pandocfilters                            1.5.1
papermill                                2.6.0
parso                                    0.8.4
partd                                    1.4.2
partial-json-parser                      0.2.1.1.post5
pbr                                      6.1.1
peft                                     0.15.2
pexpect                                  4.9.0
pfzy                                     0.3.4
pillow                                   11.2.1
pip                                      22.0.2
platformdirs                             4.3.8
pluggy                                   1.6.0
pooch                                    1.8.2
portalocker                              3.1.1
prometheus_client                        0.22.0
prometheus-fastapi-instrumentator        7.1.0
promise                                  2.3
prompt_toolkit                           3.0.51
propcache                                0.3.1
proto-plus                               1.26.1
protobuf                                 4.25.7
psutil                                   7.0.0
ptyprocess                               0.7.0
pure_eval                                0.2.3
py-cpuinfo                               9.0.0
py-spy                                   0.4.0
pyarrow                                  19.0.1
pyasn1                                   0.6.1
pyasn1_modules                           0.4.2
pybind11                                 2.13.6
pycodestyle                              2.13.0
pycountry                                24.6.1
pycparser                                2.22
pydantic                                 2.11.4
pydantic_core                            2.33.2
pydot                                    4.0.0
pyflakes                                 3.3.2
Pygments                                 2.19.1
pykeops                                  2.3
pynndescent                              0.5.13
pyogrio                                  0.11.0
PyOpenGL                                 3.1.5
pyparsing                                3.2.3
pyproj                                   3.7.1
pyprojroot                               0.3.0
pysqlite3                                0.5.4
pytest                                   8.3.5
pytest-asyncio                           0.26.0
python-dateutil                          2.9.0.post0
python-dotenv                            1.1.0
python-json-logger                       3.3.0
python-keystoneclient                    5.6.0
python-multipart                         0.0.20
python-swiftclient                       4.7.0
pytorch-lightning                        2.5.1.post0
pytorch-triton-rocm                      3.3.1
pytz                                     2025.2
PyYAML                                   6.0.2
pyzmq                                    26.4.0
rasterio                                 1.4.3
ray                                      2.44.1
redis                                    6.1.0
referencing                              0.36.2
regex                                    2024.11.6
requests                                 2.32.3
rfc3339-validator                        0.1.4
rfc3986                                  2.0.0
rfc3986-validator                        0.1.1
rich                                     14.0.0
rich-toolkit                             0.14.6
rpds-py                                  0.25.0
rsa                                      4.7.2
runai-model-streamer                     0.11.0
runai-model-streamer-s3                  0.11.0
s3transfer                               0.13.0
safetensors                              0.5.3
scikit-image                             0.25.2
scikit-learn                             1.6.1
scipy                                    1.14.1
seaborn                                  0.13.2
Send2Trash                               1.8.3
sentencepiece                            0.2.0
setuptools                               59.6.0
shapely                                  2.1.1
shellingham                              1.5.4
simple-parsing                           0.1.7
six                                      1.17.0
sklearn-compat                           0.1.3
smart-open                               7.1.0
smmap                                    5.0.2
sniffio                                  1.3.1
sortedcontainers                         2.4.0
soundfile                                0.13.1
soupsieve                                2.7
soxr                                     0.5.0.post1
SQLAlchemy                               2.0.41
sqlparse                                 0.5.3
stack-data                               0.6.3
starlette                                0.46.2
stevedore                                5.4.1
sympy                                    1.14.0
tables                                   3.10.2
tabulate                                 0.9.0
tblib                                    3.1.0
tenacity                                 9.1.2
tensorboard                              2.19.0
tensorboard-data-server                  0.7.2
tensorboardX                             2.6.2.2
tensorflow-datasets                      4.9.8
tensorflow-metadata                      1.14.0
tensorizer                               2.9.3
termcolor                                3.1.0
terminado                                0.18.1
threadpoolctl                            3.6.0
tifffile                                 2025.5.10
tiktoken                                 0.9.0
timm                                     1.0.15
tinycss2                                 1.4.0
tokenizers                               0.21.1
toml                                     0.10.2
tomli                                    2.0.1
toolz                                    1.0.0
torch                                    2.7.1+rocm6.2.4
torch-tb-profiler                        0.4.3
torchaudio                               2.7.1+rocm6.2.4
torchinfo                                1.8.0
torchmetrics                             1.7.1
torchvision                              0.22.1+rocm6.2.4
tornado                                  6.5
tqdm                                     4.67.1
traitlets                                5.14.3
transformers                             4.52.4
triton                                   3.3.0
trl                                      0.17.0
typeguard                                4.3.0
typer                                    0.15.4
types-python-dateutil                    2.9.0.20250516
typing_extensions                        4.13.2
typing-inspection                        0.4.0
tzdata                                   2025.2
umap-learn                               0.5.7
uri-template                             1.3.0
urllib3                                  2.4.0
uvicorn                                  0.34.2
uvloop                                   0.21.0
virtualenv                               20.31.2
visdom                                   0.2.4
vllm                                     0.9.1+rocm624
wadler_lindig                            0.1.6
watchfiles                               1.0.5
wcwidth                                  0.2.13
webcolors                                24.11.1
webencodings                             0.5.1
websocket-client                         1.8.0
websockets                               15.0.1
Werkzeug                                 3.1.3
wheel                                    0.43.0
widgetsnbextension                       4.0.14
wrapt                                    1.17.2
xformers                                 0.0.31+39addc86.d20250521
xgboost                                  3.0.1
xgrammar                                 0.1.19
xlwt                                     1.3.0
xxhash                                   3.5.0
yacs                                     0.1.8
yarl                                     1.20.0
zict                                     3.0.0
zipp                                     3.21.0
```

If you notice that there is a missing package, we can install it with:

```
pip install (package_name)
```

We can deactivate the enviroments with:

```
deactivate
```

With this we now only need to provide a SSH key, create a Ray cluster job and submit it. Move to your personal folder by checking it from workspaces:

```
# Puhti and Mahti
csc-workspaces

# LUMI
lumi-workspaces
```

These give the following path:

```
cd /users/(your_csc_user)
```

Now, assuming that you have a ssh key for connections between cpouta and HPC, we can move the SSH key into personal directory in the following way:

```
# Local
pwd
cd .ssh
cat cloud-hpc.pem
CTRL + SHIFT + C (copy the private key)

# HPC
pwd (confirm that you are in the personal directory)
nano cloud-hpc.pem
CTRL + SHIFT + V
CTRL + X
Y
cat cloud-hpc.pem
chmod 600 cloud-hpc.pem
```

We now only need to create the ray SLURM job. Please check https://docs.csc.fi/computing/running/batch-job-partitions/ and https://docs.lumi-supercomputer.eu/runjobs/scheduled-jobs/partitions/ to confirm suitable paritions. The most important thing is to check if the paritions has atleast two nodes for normal head-worker cluster. For this reason the paritions are:

   - Puhti -> large
   - Mahti -> medium
   - LUMI -> small

You can find ray-cluster.sh templates at deployments/ray/slurm. When you have filled the details, copy them with:

```
# HPC 
pwd (check that you are in the personal directory)
nano tutorial-ray-cluster.sh
CTRL + SHIFT + V
CTRL + X
Y
cat tutorial-ray-cluster.sh
```

Before we submit it, remember that we can confirm that the dashboard connection works by local forwarding it:

```
# SSH local forward for Puhti
ssh -L 127.0.0.1:8125:(your_VM_private_ip):8125 cpouta
# SSH local forward for Mahti
ssh -L 127.0.0.1:8126:(your_VM_private_ip):8126 cpouta
# SSH local forward fpr LUMI
ssh -L 127.0.0.1:8127:(your_VM_private_ip):8127 cpouta
```

The local forward addresses are:

   - http://127.0.0.1:8125
   - http://127.0.0.1:8126
   - http://127.0.0.1:8127

After that, we can confirm it works inside cluster with a curl pod:

```
cd deployments/networking
kubectl apply -f curl.yaml
kubectl get pods
kubectl exec -it curl-pod -- /bin/sh

# Puhti
curl http://ray-hpc-1-dash.integration.svc.cluster.local:8125

# Mahti
curl http://ray-hpc-2-dash.integration.svc.cluster.local:8126

# LUMI
curl http://ray-hpc-3-dash.integration.svc.cluster.local:8127
```

To confirm it works inside VM through Istio we can use curl with:

```
# Puhti
curl -v -H "Host: ray.hpc.dash-1.oss" http://localhost:7001
# Mahti
curl -v -H "Host: ray.hpc.dash-2.oss" http://localhost:7001
# LUMI
curl -v -H "Host: ray.hpc.dash-3.oss" http://localhost:7001
```

To confirm it works via browser, you can click the following urls:

   - Puhti: http://ray.hpc.dash-1.oss:7001
   - Mahti: http://ray.hpc.dash-2.oss:7001
   - LUMI: http://ray.hpc.dash-3.oss:7001

Assuming that everything works, submit the job

```
sbatch tutorial-ray-cluster.sh
```

You can check that it runs with:

```
squeue --me
```

When the job has run around 60 seconds, we should see the dasbhoards with local, curl and browser with the mentioned methods. After confirming access via browser, cancel the job with:

```
scancel (job_id)
```

We can check the created logs with:

```
cat slurm-(job_id).out
```

In puhti and mahti, we can check the spent billing units with:

```
seff (job_id)
```

In all of them we can get job metrics using the follwing command:

```
sacct -j (job_id)
```

We can get specific fields with:

```
sacct -o jobid,jobname,account,partition,timelimit,submit,start -j (job_id)
```

If we assume that we do file transfers through SSH, we now know all the manual actions we need to consider in HPC interaction automation.
