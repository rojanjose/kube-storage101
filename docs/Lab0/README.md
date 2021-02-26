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
