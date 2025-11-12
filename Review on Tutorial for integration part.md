This review covers the tutorials on integrating distributed computing in MLOps. There are 10 parts.

- [Move to part 1](#part-1)
- [Move to part 2](#part-2)
- [Move to part 3](#part-3)
- [Move to part 4](#part-4)
- [Move to part 5](#part-5)
- [Move to part 6](#part-6)
- [Move to part 7](#part-7)
- [Part 8](#part-8) coming soon!
- [Part 9](#part-9) coming soon!
- [Part 10](#part-10) coming soon!

# Part 1

Estimated time:
H: 4h
J: 2,5h
Together: 1,5h

[Link to part 1](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_1.ipynb)

## Summary

- Defining concepts being used and the terminology behind every important concept
- Tells the meaning behind the documentation
- Explained why Python was chosen as the coding language
- Explaining the use of JupyterLab
- Explaining the use of Pydantic
- Explaining the use of YAML
- Docker setup

## What works

Huang:
- Defining concepts at the start clearly (Allas, Mycsc account etc.)
- Specific versions of dependencies mentioned in case of problems and solutions to them
- Examples of dictionaries

Jukka:
- The lists of separated system integrations and enviroment tools are really good
- The conclusion to choose certain technologies (for example Python) were clearly justifyed in the document
- Examples of usage of technologies in the document were good

## Suggested improvements

Huang:
- Perhaps naming the hyperlinks? (The useful links)
- Small typos like "cloud enviroment"
- Commands highlighted
- If abbreviations are used, would be nice to be included in the defining part
- If there is time, screenshots or guidevideos usually help visually, especially to complete newbies
- [The installation instructions](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/tree/development/tutorials/integration) are outdated

Jukka:
- Naming could be done on the links in the notebook file of the useful material lists
- Not all of the examples aren't completely clear in every step (Pydantic and YAML)
- Simplified and more visual presentations or animations could work for the data scientist who are going to be using these platforms
- We could have a hierarcical view of the technologies and the correlations between them

## Questions
- Why do we need YAML & Pydantic in the project (What does the JSON validation and YAML's pydantic validation mean?)?

# Part 2

Estimated time:
H: 2h
J: 1,5h
Together: 1h

[Link to part 2](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_2.ipynb)

## Summary
- Explanation of FastAPI and how Submitter and Kubernetes Forwarder communicate through the Uvicorn server
- Explanation of the FastApi logs, application file structure and important parts of the fastApi. Document also tells how APIRouters and their routes work
- Docker Compose explained and how to use it
- Redis explained and how to use it
- Examples of using the created/stored Redis pickled data 

## What works

Huang:
- The structure of separating the application into smaller files, easier to pinpoint possible issues
- The useful materials themselves are nice and clear

Jukka: 
- How to check localhost and the fastapi logs is valuable help
- Explaining of the fastapi is done very well. The important parts list is good
- Examples of how to use the commands are clear
- The entire document is much more clearer than the first one because there are fewer new technologies and the correlation between everything is much more clearer

## Suggested improvements

Huang:
- A bit confusing at first where commands should be typed, what directory etc. (initial installation or some separate app?)
- Concrete examples or more explanation on different factors, difficult to imagine without prior experience
- Naming conventions on "useful material", step-by-step guiding because initial assumption is just "extra material"
- What are the things in braces for? (Between the Redis configurations)
- Re-enacting steps should bring similar results each time

Jukka:
- The examples could include videos and animations
- Summary of the most important things in the end of the document

## Questions
- Is it okay to assume that data scientist are able to use the technologies only by reading this documentary if they don't have any prior experience with software development?
- Could there be a better alternative for the examples that show the coding in practice? Could the examples be better explained in the documentary? (Redis in the end of this document)

# Part 3

Estimated time:
H:
J:
Together:

[Link to part 3](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_3.ipynb)

## Summary

- Celery with Redis works as a message broker, automating interactions between separate containers and systems
- Flower enables easier monitoring and debugging of Celery tasks
- Celery Beat enables scheduling of Celery tasks
- Explaining the use of Microservice Architectures

## What works
Jukka:
- All of the technologies in this document are clearly linked to one another so understanding was much easier this time. The document wasn't also too long.
- The celery related technologies are much easier to understand in general

## Suggested improvements

Jukka:
- New supporting learning material (for example youtube)
- Some of the useful material links don't work properly (last link in the microservice section)

## Questions

# Part 4

Estimated time:
H:
J:
Together:

[Link to part 4](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_4.ipynb)

## Summary
- Apache Airflow is used as a platform to programmatically monitor workflows. We can also create and schedule the workflows with it.
- The workflow blueprint are called DAGs and they are run similarly to kubeflow pipelines. When the workflow develops, we can modity the Submitter and Forwarder automation by modifying the DAGs.
- TriggerDagRUnOperator is a operator that we can use to create a complex DAG that triggers smaller DAGs by defining the sub DAGs.
- Hooks in Airflow are high-level interfaces to external platforms that remove the need to write low-level code.
- Most of the documentation is showing the logs and code in practice that is necessary for the creation of the Airflow integration.

## What works
Jukka:
- Lots of good log information and documentation that is usable specifically for our project.
  
## Suggested improvements
Jukka:
- Separating the theory parts of the technologies and the practice of showing coding and logs to separate blocks. Other idea is to have the concrete coding done in separate blocks of the documentation.
- Separating the DAGs and coding into their own documentations.

## Questions

# Part 5

Estimated time:
H:
J:
Together:

[Link to part 5](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_5.ipynb)

## Summary

## What works

## Suggested improvements
# Kind / Kubeflow Ympäristön Asennusdokumentaatio

## Ympäristön kuvaus

* Käyttöjärjestelmä: Ubuntu (pilvi-VM: "ubuntu@oss-platform")
* Työkalut: Docker, kind, kubectl, Kubeflow integration setup script
* Tarkoitus: pystyttää kind-klusteri ja ajaa `integration-setup.sh` Kubeflow/KFP-asennusta varten

---

## 1. Kind-komento ei toiminut

**Virheilmoitus:**

```
kind: command not found
```

**Syy:**
Kind ei ollut asennettuna tai PATH-muuttujassa.

**Ratkaisu:**

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Tulos:** `kind version` toimii

---

## 2. Docker-permission denied -virhe

**Virheilmoitus:**

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

**Syy:** Käyttäjä `ubuntu` ei kuulunut `docker`-ryhmään.

**Ratkaisu:**

```bash
sudo usermod -aG docker ubuntu
sudo reboot
```

**Varmistus:**

```bash
groups  # pitäisi sisältää 'docker'
docker ps  # toimii ilman sudo
```

---

## 3. Fstab-virhe – NFS usage parsing

**Virheilmoitus:**

```
cannot parse /etc/fstab: expected at least 3 fields, found 1
```

**Syy:** `/etc/fstab` tiedoston viimeinen rivi oli virheellinen.

**Korjaus:** Kommentoi rivin alkuun `#` tai poista se

```bash
# UUID="046ec14e-b98f-499f-865b-7cb5ebc872bf"
```

**Varmistus:**

```bash
sudo mount -a  # ei virheilmoitusta
```

---

## 4. Kustomize not found

**Virheilmoitus:**

```
./integration-setup.sh: line 241: kustomize: command not found
```

**Syy:** Kustomize ei ole asennettuna eikä PATHissa.

**Ratkaisu:**

```bash
cd ~
curl -s https://api.github.com/repos/kubernetes-sigs/kustomize/releases/latest \
| grep browser_download_url \
| grep linux_amd64.tar.gz \
| cut -d '"' -f 4 \
| xargs curl -Lo kustomize.tar.gz

tar -xzf kustomize.tar.gz
sudo mv kustomize /usr/local/bin/
rm kustomize.tar.gz
```

**Varmistus:**

```bash
kustomize version
```

---

## Yhteenveto

| Vaihe | Virheilmoitus / Ongelma                  | Syy                         | Ratkaisu                                 |
| ----- | ---------------------------------------- | --------------------------- | ---------------------------------------- |
| 1     | `kind: command not found`                | Kind ei asennettu           | Asenna kind ja lisää PATHiin             |
| 2     | `permission denied /var/run/docker.sock` | Käyttäjä ei docker-ryhmässä | `sudo usermod -aG docker $USER` + reboot |
| 3     | `cannot parse /etc/fstab`                | Virheellinen rivi fstabissa | Kommentoi ylimääräinen UUID-rivi         |
| 4     | `kustomize: command not found`           | Kustomize ei asennettu      | Lataa ja asenna manuaalisesti            |

**Lopputulos:**

* Docker toimii ilman sudoa
* kind-klusteri pystyssä
* fstab-virhe korjattu
* kustomize asennettu ja PATHissa
* `integration-setup.sh` voi jatkaa Kubeflow / KFP + KServe -ympäristön asennusta

---

## Suositellut tarkistuskomennot

```bash
docker ps
kind get clusters
kubectl version --client
kustomize version
```

## Questions

# Part 6

Estimated time:
H:
J:
Together:

[Link to part 6](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_6.ipynb)

## Summary

## What works

## Suggested improvements

## Questions

# Part 7

Estimated time:
H:
J:
Together:

[Link to part 7](https://github.com/K123AsJ0k1/multi-cloud-hpc-oss-mlops-platform/blob/studying/tutorials/integration/studying/tutorial_for_integration_part_7.ipynb)

## Summary

## What works

## Suggested improvements

## Questions

# Part 8

Estimated time:
H:
J:
Together:

Coming soon!

## Summary

## What works

## Suggested improvements

## Questions


# Part 9

Estimated time:
H:
J:
Together:

Coming soon!

## Summary

## What works

## Suggested improvements

## Questions


# Part 10

Estimated time:
H:
J:
Together:

Coming soon!

## Summary

## What works

## Suggested improvements

## Questions
