# Lab 0: Pre-work

## 1. Setup IBM Cloud and Kubernetes environment

* Using grant cluster app [here](https://ibm.github.io/workshop-setup/GRANTCLUSTER/)

## 2. CLI environment

* Using IBM Cloud shell [here](https://ibm.github.io/workshop-setup/CLOUDSHELL/)
* Using Cognitive Class [here](https://ibm.github.io/workshop-setup/COGNITIVECLASS/)

## 3. Docker hub account

Create a [dockerhub](https://hub.docker.com/) user and set the environment variable.

```bash
DOCKERUSER=<dockerhub useid>
```

Ensure the docker engine is up in your terminal enviroment by running the command.
```bash
docker version
```
```
$ docker version
Client:
 Version:           19.03.6
 API version:       1.40
 Go version:        go1.12.17
 Git commit:        369ce74a3c
 Built:             Wed Oct 14 19:00:27 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.5
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.12
  Git commit:       633a0ea838
  Built:            Wed Nov 13 07:28:45 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.2.10
  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
 runc:
  Version:          1.0.0-rc8+dev
  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```
## 4. Set the cluster name

```bash
ibmcloud ks clusters

OK
Name                               ID                     State     Created        Workers   Location          Version                 Resource Group Name        Provider
user001-workshop                      bseqlkkd0o1gdqg4jc10   normal    3 months ago   5         Dallas            4.3.38_1544_openshift   default                    classic


CLUSTERNAME=use001-workshop
```

or

```bash
CLUSTERNAME=`ibmcloud ks clusters | grep Name -A 1 | awk '{print $1}' | grep -v Name`

echo $CLUSTERNAME
user001-workshop
```
