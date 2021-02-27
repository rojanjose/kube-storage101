# Lab 4. Storage with Kubernetes Stateful Sets

## Introduction

When looking at what kind of storage class you would like to use in Kubernetes, there are are a few choices such as file storage, block storage, object storage, etc. If your use case requires fast and reliable data access then consider block storage.

[Block storage](https://www.ibm.com/cloud/learn/block-storage) is a storage option that breaks data into "blocks" and stores those blocks across a Storage Area Network (SAN). These smaller blocks are faster to store and retrieve than large data objects. For this reason, block storage is primarily used as a backing storage for databases.


## Mongodb install as ReplicaSet configuration

In Lab 3, we installed Mongodb
```
                ┌────────────────┐
                │   MongoDB&reg; │
                |      svc       │
                └───────┬────────┘
                        │
                        ▼
                  ┌────────────┐
                  │MongoDB&reg;│
                  │   Server   │
                  │    Pod     │
                  └────────────┘
```

ReplicaSet without the [MongoDB® Arbiter](https://docs.mongodb.com/manual/core/replica-set-arbiter/) component
```
    ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
    │ MongoDB&reg; 0 │ │ MongoDB&reg; 1 │ │ MongoDB&reg; N │
    |  external svc  │ |  external svc  │ |  external svc  │
    └───────┬────────┘ └───────┬────────┘ └───────┬────────┘
            │                  │                  │
            ▼                  ▼                  ▼
    ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
    │ MongoDB&reg; 0 │ │ MongoDB&reg; 1 │ │ MongoDB&reg; N │
    │    Server      │ │     Server     │ │     Server     │
    │     Pod        │ │      Pod       │ │      Pod       │
    └────────────────┘ └────────────────┘ └────────────────┘
         primary           secondary         secondary
```

In this lab we will deploy a Mongo database on top of block storage on Kubernetes.

The workflow this lab is as follows:

1. When we install MongoDB with the helm chart, a `Persistent Volume Claim` (PVC) is created on the cluster. This PVC is a request for storage to be used by the application.

2. In IBM Cloud, the request goes to the IBM Cloud storage provider which then provisions a physical storage device within IBM Cloud.

3. A `Persistent Volume` (PV) is then created which acts as a reference to the physical storage device created earlier. This PV is then mounted as a directory in a container's file system.

4. The guestbook application receives requests to store guestbook entries from the user which the guestbook pod then sends to the MongoDB pod to store.

5. The MongoDB pod receives the request to store information and persists the data to the mounted directory from the Persistent Volume.

## Setup

Before we get into the lab we first need to do some setup to ensure that the lab will flow smoothly.

1. In your terminal, navigate to where you would like to store the files used in this lab and run the following.
    ```bash
    WORK_DIR=`pwd`
    ```
1. Ensure that you have run through the prerequistes in [Lab0](../Lab0/README.md)

## Using IBM Cloud Block Storage with Kubernetes

Log into the Kubernetes cluster and create a project where we want to deploy the application with stateful database.

```bash
kubectl create namespace mongo-statefulset
```
```
$ kubectl create namespace mongo-statefulset
namespace/mongo-statefulset created
```

## Setup Block Storage Plugin & MongoDB Helm Repo

If you have not completed lab 3, install the Block Strogage plugin as descibed [here](../Lab3/#install-block-storage-plugin).

Verify that the plugin installation was successful by retrieving the list of storage classes in the cluster:

```bash
kubectl get storageclasses | grep block
```

You should notice a few options that start with `ibmc-block` as seen below.

```
$ kubectl get storageclasses | grep block

ibmc-block-bronze          ibm.io/ibmc-block   Delete          Immediate           true                   9m32s
ibmc-block-custom          ibm.io/ibmc-block   Delete          Immediate           true                   9m32s
ibmc-block-gold            ibm.io/ibmc-block   Delete          Immediate           true                   9m32s
ibmc-block-retain-bronze   ibm.io/ibmc-block   Retain          Immediate           true                   9m32s
ibmc-block-retain-custom   ibm.io/ibmc-block   Retain          Immediate           true                   9m32s
...
```

Setup the repo that holds the MongoDB chart by following the steps descibed [here](../Lab3/#install-block-storage-plugin).


### Installation Dry Run

Like the previous lab,let's do a dry run of the installation using the `--dry-run` flag to see what the chart will create. Certain input parameter changes are required for the chat to order to install MongoDB as stateful sets.

Dryrun:

```bash
helm install mongo bitnami/mongodb --set architecture=replicaset,arbiter.enabled=false,replicaCount=3,global.storageClass=ibmc-block-gold,auth.password=testing,auth.username=guestbookAdmin,auth.database=guestbook -n mongo-statefulset --dry-run > mongdb-statefuleset-install-dryrun.yaml
```
The addictional parameters used in this install command:

- architecture=replicaset, `replicaset` value will force the installation in master/slave configuration using the Kubernetes StatefulSet manifests. Default value for `architecture` is `standalone` and was used in the previous lab.
- arbiter.enabled=false, disables the installation of arbiter component.
- replicaCount=3, specifies the number of slave nodes for the MongoDB database.

Review the manifests generated in the file `mongdb-statefuleset-install-dryrun.yaml`. 
Check out the file in your code editor and take a look at the `StatefulSet` object. Scroll down futher, the property named `storageClassName` with a value of `ibmc-block-gold` is defined under the section `volumeClaimTemplates`.

```yaml
# Source: mongodb/templates/replicaset/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo-mongodb
  namespace: mongo-statefulset
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-10.7.1
    app.kubernetes.io/instance: mongo
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mongodb
spec:
  serviceName: mongo-mongodb-headless
  podManagementPolicy: OrderedReady
  replicas: 3
.....
.....
.....
  volumeClaimTemplates:
    - metadata:
        name: datadir
      spec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: "8Gi"
        storageClassName: ibmc-block-gold
......
......
```

### Install Mongodb

Before we install MongoDB we need to generate a password for our database credentials. These credentials will be used in the application to authenticate with the database.

Generate the password for the user we will use for the `guestbook` database.

```bash
USER_PASS=`openssl rand -base64 12 | tr -d "=+/"`
```

Now we can install MongoDB using the command we tried for the dry run and supply the password that we just generated.

```bash
helm install mongo bitnami/mongodb --set architecture=replicaset,arbiter.enabled=false,replicaCount=3,global.storageClass=ibmc-block-gold,auth.password=$USER_PASS,auth.username=guestbookAdmin,auth.database=guestbook -n mongo-statefulset 
```

Here's an explanation of the above command:

- `helm install mongo bitnami/mongo`: Install the MongoDB bitnami helm chart and name the release "mongo".
- `--set global.storageClass=ibmc-block-gold`: Set the storage class to block storage rather than the default file storage.
- ...`auth.password=$USER_PASS`: Create a custom user with this password (Which we generated earlier).
- ...`auth.username=guestbook-admin`: Create a custom user with this username.
- ...`auth.database=guestbook`: Create a database named `guestbook` that the custom user can authenticate to.
- `-n mongo-statefulset`: Install this release in the `mongo-statefulset` namespace.

Expected output:

```
$ helm install mongo bitnami/mongodb --set architecture=replicaset,arbiter.enabled=false,replicaCount=3,global.storageClass=ibmc-block-gold,auth.password=$USER_PASS,auth.username=guestbookAdmin,auth.database=guestbook -n mongo-statefulset

NAME: mongo
LAST DEPLOYED: Fri Feb 26 18:06:46 2021
NAMESPACE: mongo-statefulset
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

MongoDB(R) can be accessed on the following DNS name(s) and ports from within your cluster:

    mongo-mongodb-0.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017
    mongo-mongodb-1.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017
    mongo-mongodb-2.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017

To get the root password run:

    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongo-statefulset mongo-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

To get the password for "guestbookAdmin" run:

    export MONGODB_PASSWORD=$(kubectl get secret --namespace mongo-statefulset mongo-mongodb -o jsonpath="{.data.mongodb-password}" | base64 --decode)

To connect to your database, create a MongoDB(R) client container:

    kubectl run --namespace mongo-statefulset mongo-mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:4.4.4-debian-10-r0 --command -- bash

Then, run the following command:
    mongo admin --host "mongo-mongodb-0.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017,mongo-mongodb-1.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017,mongo-mongodb-2.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```

Copy and save the commands displayed under the `NOTES` section. We will use these commands later in the lab to access the data directly from the database.


Wait for the some time and then view the objects created by the helm chart.

```bash
kubectl get all -n mongo-statefulset
```

```
$ kubectl get all -n mongo-statefulset

NAME                  READY   STATUS    RESTARTS   AGE
pod/mongo-mongodb-0   1/1     Running   0          6m8s
pod/mongo-mongodb-1   1/1     Running   0          4m14s
pod/mongo-mongodb-2   1/1     Running   0          2m26s

NAME                             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/mongo-mongodb-headless   ClusterIP   None         <none>        27017/TCP   6m9s

NAME                             READY   AGE
statefulset.apps/mongo-mongodb   3/3     6m9s
```

Unlike in the prvious lab where MongoDB installed in standalone more, the pod had the name with the `mongo-mongodb-<random-string>` format. The pods in stateful set have sticky identities with the format `mongo-mongodb-n` where n = 0 for the master and 1,2,3.. for slave nodes.

Check the storage allocation for the database. PVs are allocated for each replica sets and `mongo-mongodb` is now bound to volumes.

```bash
kubectl get pvc -n mongo-statefulset
```
```
$ kubectl get pvc -n mongo-statefulset
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
datadir-mongo-mongodb-0   Bound    pvc-2339ad4f-c6ba-4280-8888-d13058da7d0a   20Gi       RWO            ibmc-block-gold   14m
datadir-mongo-mongodb-1   Bound    pvc-d5b9630c-ef27-438e-986e-301e26d11329   20Gi       RWO            ibmc-block-gold   12m
datadir-mongo-mongodb-2   Bound    pvc-038fea42-d456-416d-b5b4-db038c6f0a8d   20Gi       RWO            ibmc-block-gold   10m
```

With MongoDB deployed now we need to deploy an application that will utilize it as a datastore.

### Building Guestbook

Build the Guestbook application as described in lab 3 [here](../Lab3/#building-guestbook).

### Deploying Guestbook

Now that we have built our application, let's deploy the application the same way it was done in Lab 3.

1. Navigate to the configuration repo that we cloned earlier.
    ```bash
    cd $WORK_DIR/guestbook-nodejs-config
    ```
    
    This repo contains 3 manifests that we will be deploying to our cluster today:

        - A deployment manifest
        - A service manifest
        - A configMap manifest


1. Ensure the image name is set corerctly in the deployment yaml `guestbook-deployment.yaml` as described in Lab 3

    For example, my Docker username is `odrodrig` so line 25 in my `guestbook-deployment.yaml` file would look like this:

    ```yaml
    ...
    image: odrodrig/guestbook-nodejs:mongo
    ...
    ```

    Create the kubernetes secret for database user name (`MONGO_USER`) and password (`MONGO_PASS`) 

    ```bash
    kubectl create secret generic mongodb --from-literal=username=guestbook-admin --from-literal=password=$USER_PASS -n mongo-statefulset
    ```
    ```
    $ kubectl create secret generic mongodb --from-literal=username=guestbook-admin --from-literal=password=$USER_PASS -n mongo-statefulset
    secret/mongodb created
    ```

1. Now we are ready to deploy the application. Run the following commands:

    ```bash
    cd $WORK_DIR/guestbook-nodejs-config/
    kubectl apply -f . -n mongo-statefulset
    ```

    Ensure that the application pod is running:

    ```bash
    kubectl get pods -n mongo-statefulset
    ```

    You should see both the mongo pod and the guestbook pod running now:

    ```
    $ kubectl get pods -n mongo-statefulset
    NAME                           READY   STATUS    RESTARTS   AGE
    guestbook-v1-9668f9585-dp9r8   1/1     Running   0          3m24s
    mongo-mongodb-0                1/1     Running   0          79m
    mongo-mongodb-1                1/1     Running   0          78m
    mongo-mongodb-2                1/1     Running   0          76m
    ```

### Test out the application

Now that we have deployed the application, let's test it out.

1. Find the URL for the guestbook application by joining the worker node external IP and service node port. Run the following to get the IP and service node port of the application:

    ```bash
    HOSTNAME=`kubectl get nodes -ojsonpath='{.items[0].metadata.labels.ibm-cloud\.kubernetes\.io\/external-ip}'`
    SERVICEPORT=`kubectl get svc guestbook -n mongo-statefulset -o=jsonpath='{.spec.ports[0].nodePort}'`
    echo "http://$HOSTNAME:$SERVICEPORT"
    ```

1. In your browser, open up the address that was output as part of the previous command. Type in a few test entries in the text box and press enter to submit them.

    ![testEntries](./images/testEntries.png)

    These entries are now saved in the Mongo database. Let's take down the application and see if the data will truly persist.

1. Find the name of the pod that is running our application:

    ```bash
    kubectl get pods -n mongo-statefulset
    ```

    Copy the name of the pod that starts with `guestbook`. For me, the pod is named `guestbook-v1-9465dcbb4-f6s9h`.

    ```bash
    NAME                             READY   STATUS    RESTARTS   AGE
    guestbook-v1-9465dcbb4-f6s9h     1/1     Running   0          4m7s
    mongo-mongodb-757d9777d7-q64lg   1/1     Running   0          5m47s
    ```

1. Let's connect to the database to view the records.

    ```bash
        export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace mongo-statefulset mongo-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)
        export MONGODB_PASSWORD=$(kubectl get secret --namespace mongo-statefulset mongo-mongodb -o jsonpath="{.data.mongodb-password}" | base64 --decode)
        kubectl run --namespace mongo-statefulset mongo-mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:4.4.4-debian-10-r0 --command -- bash
    ```

    Then, run the following command:
    ```bash
        mongo admin --host "mongo-mongodb-0.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017,mongo-mongodb-1.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017,mongo-mongodb-2.mongo-mongodb-headless.mongo-statefulset.svc.cluster.local:27017" --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
    ```

    Query the data in the `guestbook` database:
    ```
    > use guestbook
    switched to db guestbook
    > show collections
    entry
    >
    > db.entry.find()
    { "_id" : ObjectId("603955c28f6077001501e239"), "message" : "Hello", "timestamp" : ISODate("2021-02-26T20:10:42Z") }
    { "_id" : ObjectId("603955c58f6077001501e23a"), "message" : "Holla", "timestamp" : ISODate("2021-02-26T20:10:45Z") }
    { "_id" : ObjectId("603955c98f6077001501e23b"), "message" : "Namaste", "timestamp" : ISODate("2021-02-26T20:10:49Z") }
    >
    ```

## Summary

In this lab we used block storage to run our own database on Kubernetes. Block storage allows for fast I/O operations making it ideal for our application database. We utilized configMaps and secrets to store the database configuration making it easy to use this application with different database configurations without making code changes.

## Cleanup (Optional)

This part of the lab desrcibes the steps to delete what was built in the lab.

### Deleting the application

```bash
cd $WORK_DIR/guestbook-nodejs-config
kubectl delete -f . -n mongo-statefulset
```

### Uninstalling Mongo

```bash
helm uninstall mongo -n mongo-statefulset
```

### Remove namespace

```bash
kubectl delete namespace mongo-statefulset
```
