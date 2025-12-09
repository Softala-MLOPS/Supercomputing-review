# Simplified instructions for OSS Platform

This document contains instructions based on [tutorial part 5](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_5.ipynb). The purpose is to provide an easy step-by-step guide to getting the platform working.

## Index

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
4. Make sure from the top-left corner that you are doing this in the right project. You can confirm the correct project number from MyCSC project page. If you can't find the right project number, you may need to wait a bit longer.
5. First we'll create an instance. Go to Compute > Instances and press "Launch Instance"
6. Give the instance a name and a description you like, to help you identify it better. When writing the name, preferably use lower case letters and substitute spaces with "_"
7. Go to Source and choose "Yes" to creating a new volume. Give it at least 100
8. 
