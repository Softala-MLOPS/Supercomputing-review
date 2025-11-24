# Part 5 setup Installation problems
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
