# Part 5 Setup Installation Problems
This installation file is for the creator of the original repository as part of the review because the steps documented here were missing from the original installation. The step-by-step installation in this repository ([linked here](https://github.com/Softala-MLOPS/Supercomputing-review/blob/main/Simplified%20instructions%20for%20OSS%20Platform.md)) contains all of the parts mentioned here
# Kind / Kubeflow Environment Installation Documentation

## Environment Description

* Operating System: Ubuntu (cloud VM: "ubuntu@oss-platform")
* Tools: Docker, kind, kubectl, Kubeflow integration setup script
* Purpose: Set up a kind cluster and run `integration-setup.sh` for Kubeflow/KFP installation

---

## 1. Kind Command Not Working

**Error Message:**

```
kind: command not found
```

**Cause:**
Kind was not installed or not in the PATH variable.

**Solution:**

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Result:** `kind version` works

---

## 2. Docker Permission Denied Error

**Error Message:**

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

**Cause:** User `ubuntu` was not part of the `docker` group.

**Solution:**

```bash
sudo usermod -aG docker ubuntu
sudo reboot
```

**Verification:**

```bash
groups  # should contain 'docker'
docker ps  # works without sudo
```

---

## 3. Fstab Error – NFS Usage Parsing

**Error Message:**

```
cannot parse /etc/fstab: expected at least 3 fields, found 1
```

**Cause:** The last line in the `/etc/fstab` file was invalid.

**Fix:** Comment out the beginning of the line with `#` or remove it

```bash
# UUID="046ec14e-b98f-499f-865b-7cb5ebc872bf"
```

**Verification:**

```bash
sudo mount -a  # no error message
```

---

## 4. Kustomize Not Found

**Error Message:**

```
./integration-setup.sh: line 241: kustomize: command not found
```

**Cause:** Kustomize is not installed and not in PATH.

**Solution:**

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

**Verification:**

```bash
kustomize version
```

---

## Summary

| Step | Error Message / Problem                  | Cause                              | Solution                                 |
| ---- | ---------------------------------------- | ---------------------------------- | ---------------------------------------- |
| 1    | `kind: command not found`                | Kind not installed                 | Install kind and add to PATH             |
| 2    | `permission denied /var/run/docker.sock` | User not in docker group           | `sudo usermod -aG docker $USER` + reboot |
| 3    | `cannot parse /etc/fstab`                | Invalid line in fstab              | Comment out extra UUID line              |
| 4    | `kustomize: command not found`           | Kustomize not installed            | Download and install manually            |

**Final Result:**

* Docker works without sudo
* kind cluster is up
* fstab error fixed
* kustomize installed and in PATH
* `integration-setup.sh` can continue with Kubeflow / KFP + KServe environment installation

---

## Recommended Verification Commands

```bash
docker ps
kind get clusters
kubectl version --client
kustomize version
```
