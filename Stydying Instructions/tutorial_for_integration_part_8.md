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
  - 
