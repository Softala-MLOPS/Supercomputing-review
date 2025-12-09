# Tutorial for integrated distributed computing in MLOps (8/10)

This jupyter notebook goes through the basic ideas with practical examples for integrating distributed computing systems for MLOps systems.

## Index

- [CSC Supercomputers](#CSC-Supercomputers)
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

