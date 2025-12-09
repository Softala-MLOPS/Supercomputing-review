# Simplified instructions for OSS Platform

This document contains instructions based on [tutorial part 5](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_5.ipynb). A slightly better formatted version can be found [here](https://github.com/Softala-MLOPS/Supercomputing-review/blob/main/Markdown%20Files/tutorial_for_integration_part_5.md). The purpose is to provide an easy step-by-step guide to getting the platform working.

## Index

- [Notes](#Notes)
- [Pre-installments](#Pre-installments)
- [Create a project in CSC](#Create-a-project-in-CSC)
- [Create cPouta virtual machine](#Create-cPouta-virtual-machine)
- [About Volumes on virtual machine](#About-Volumes-on-virtual-machine)
- [Installing prerequisites in virtual machine](#Installing-prerequisites-in-virtual-machine)
- [Troubleshooting](#Troubleshooting)

## Notes

At the time of making these instructions (9.12.2025), we have unfortunately not found an effective way to fix any mistakes done while creating the virtual machine. This means that if you get any of the steps wrong, you may need to delete everything and start over. Even if you try to manually delete something specific and install it again, the system may recognize the leftover files and refuse to proceed as intended. I know, it sucks.

## Pre-installments

1. Before you begin creating the project, you need to make sure you have the necessary pre-installments. If you have done the initial practice installation (CLI-tool from oss-mlops-platform), then you can skip this section.
2. Otherwise refer to "Step 0: Checking all necessary pre-installments" of [this](https://github.com/OSS-MLOPS-PLATFORM/oss-mlops-platform/blob/main/tools/CLI-tool/Installations,%20setups%20and%20usage.md) document depending on whether you're using Windows or MacOS. These instructions are written based on Windows.
3. Follow "For Windows - before Ubuntu installation" to get WSL and Docker working correctly
4. We will get back to "For Linux - Ubuntu (and WSL)" when the virtual machine is set up in the later parts. Ignore it for now.

## Create a project in CSC

1. First you will need to log into your CSC account using HAKA. You can access the page from [here](https://my.csc.fi/welcome).
2. Go to your "Projects" and create a new one. Alternatively have the course teacher invite you as a collaborator to the project (e.g OSS MLOps and LLMOps for EuroHPC - non-LUMI part). If you're doing the latter, you can skip the rest of the steps in this section.
3. Name your project and give it a description you like. It's best to name it so that it's easy to recognize for you.
4. Pick "Natural Sciences" as the primary field of science and "Computer and information sciences" as secondary field of science
5. Choose end date (At least till the end of the course).
6. Choose at least cPouta when picking services. Allas and Puhti may be needed later but they can be chosen when required from the project page once it has been created.
7. The page will show you granted resources, proceed to the next phase.
8. Accept the Terms of Use. Check everything needed.
9. It will take approx. 30 min for CSC to grant you permission to use services like cPouta. You should get a confirmation in your registered email when it's done.
11. Meanwhile you should go to your "Projects", click on the project you just created and add your team members in the "Members" section.

## Create cPouta virtual machine

1. When you have the project page open, scroll down a bit until you see a section called "Services"
2. Log in to cPouta and authenticate using HAKA
3. You should be taken to an "Overview" page of all your virtual machines.
4. Make sure from the top-left corner that you are doing this in the right project. You can confirm the correct project number from MyCSC project page. If you can't find the right project number, you may need to wait a bit longer to be granted access.
5. First we'll adjust security groups in order to make SSH connection work. We'll need this to get access to the VM (virtual machine). Go to Network > Security Groups and create a new Security Group.
6. Give it a name and description you like, to help you identify it better. When writing the name, preferably use lower case letters and substitute spaces with "_" or "-".
7. You will be moved to the Group Rules. Next press "Add rule"
8. In the "Rule" box, it most likely says "Custom TCP Rule". Change that to SSH and add as it is. You should get the port range 22.
9. Go to Network > Floating IPs and press "Allocate IP to Project". Give it a suitable description for easier recognition.
10. Now we'll create a key pair, which we'll use to log in. Go to Compute > Key Pairs and create a new one.
11. Give the key pair a suitable name and choose SSH Key as the key type.
12. Download the newly created PEM-file. It's a RSA file that you will need in order to log in to the VM using SSH.
13. Go to your file explorer and move the PEM file to Linux (Below "This computer" and looks like a penguin) > Ubuntu > home > user. The "Ubuntu" or "user" may wary depending on what you've set as your username
14. Change the permissions on the PEM-file using `chmod 400 <your_key_name>.pem`
15. Next we'll create an instance. Go to Compute > Instances and press "Launch Instance"
16. Give the instance a name and a description you like
17. Go to Source and choose "Yes" to creating a new volume. Give it e.g 200 GB space
18. Scroll down and choose at least Ubuntu-22.04 as an image by clicking the up arrow next to it. May work with a newer version of Ubuntu but this has not been tested as of 9.12.2025
19. Move on to Flavor and choose standard.xxlarge VM that has 31.25 GB RAM. You will need the most powerful VM you can get because the project is heavy-duty and this is the best one available on a student account.
20. Go to Security Groups and choose the new security group you created. It doesn't matter if it's together with the default one or not.
21. Go to Key Pair and choose the key pair that you created
22. Launch Instance
23. When the instance has finished loading the blue bar, press the down arrow next to the "Create Snapshot" button and choose "Associate Floating IP". Select the Floating IP you created earlier.
24. Open your WSL terminal and write `sudo ssh -L 8080:localhost:8080 ubuntu@<floating ip> -i <the_name_of_your_PEM-file>.pem` without the "<" and ">" symbols.
25. Enter your password for Ubuntu. May be the same as the username on default.
26. The terminal will ask you of the authenticity of the floating IP. Choose Yes to connect.
27. You're in!
28. To make sure you have the latest version of applications etc. inside the virtual machine use `sudo apt update` and `sudo apt upgrade`

## About Volumes on virtual machine

- In the previous case, we have already created a volume when we created the VM. It will appear as "vda" when you use the command `lsblk`.
- If the virtual machine was an electrical appliance like a digital camera or a phone, you can think of volume as the memory card
- The volume is needed so that whatever we do inside the virtual machine will be saved. Otherwise, every time we exit the VM, we will need to start over.
-  In case you forgot to add the volume or just need some extra space, we can manually attach a separate volume, which will appear as "vdb", "vdc" and so on. In this case, we will also need to format, mount and add to fstab-file because new volumes do not have a file system on default. Here's how to do it:

1. In the cPouta Dashboard, go to Volumes > Volumes and create a new one
2. Give it a suitable name and description
3. In Volume Source, choose Image
4. In "Use image as a source", choose "Ubuntu-22.04 (2.2 GB)"
5. Give it a size you want
6. Create it
7. Go back to Compute > Instances, press the down arrow next to "Create Snapshot" and choose "Attach Volume"
8. Check with `lsblk` that the volume exists as vdb. Notice that it has nothing on mountpoint.
9. To create a file system, we need to format it using the command `sudo mkfs.ext4 /dev/vdb`. Choose Yes when asked about gpt partition table.
10. Next create the directory to which the volume will be attached to by using the command `sudo mkdir /mnt/data`
11. Now mount the volume with `sudo mount /dev/vdb /mnt/data`
12. You can check with `lsblk` to see that the file system has been created (size changed) and the mountpoint has been set
13. Now in order to make the volume persistent (mounts automatically after reboot), we'll first check the UUID of the volume with `sudo blkid /dev/vdb`
14. Copy the UUID. It looks something like `UUID="4ad9ce03-a78f-44bf-a932-3e002324da75" BLOCK_SIZE="4096" TYPE="ext4"`
15. Add UUID to the end of the fstab-file using nano. The command is `sudo nano /etc/fstab`
16. Save using `ctrl+x`, "y" and enter
17. We're done!

## Installing prerequisites in virtual machine

1. Let's go back to the Linux part of [pre-installation](https://github.com/OSS-MLOPS-PLATFORM/oss-mlops-platform/blob/main/tools/CLI-tool/Installations,%20setups%20and%20usage.md). Follow the instructions.
2. Install Docker using `sudo apt install docker.io` and check whether it succeeded with `docker version`

## Troubleshooting

There's a chance you may encounter several problems during the installation.

### Some known problems considering Kind, Kustomize etc.

Here's a document with some of the known problems considering [Kind, kustomize etc.](https://github.com/Softala-MLOPS/Supercomputing-review/blob/main/Installation.md) and some possible solutions

### Jupyter Notebook can't be installed

If installing Jupyter notebook doesn't work because of global environment issues, you can use these instead:

```
sudo apt update
sudo apt install jupyter-notebook
```

### Pip3 won't install

If Pip3 cannot be used due to global environment issues, you may need to install pipx instead.

### Docker can't find images in the installation

If Docker cannot find images in the first part of the installation, you may need to use the following commands:

```
sudo usermod -aG docker $USER
newgrp docker
docker version
```
