# Geni on Kubernetes

Geni works on any Spark cluster, including Spark on Kubernetes.
There are a lot of different approaches to running Spark on Kubernetes.

Here, we will show a simple approach, which can be replicated on a single computer, using Minikube.

## Prerequisites

The following steps assume a Linux based OS having [Docker](https://docs.docker.com/engine/install/), [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed. You may also need to install [rlwrap](https://linux.die.net/man/1/rlwrap), as we will be using the `clj` command.

## Install spark distribution

Running spark on Kubernetes requires creating Docker images for the nodes.

The Spark distribution contains tools to ease the creation of suitable Docker images,
so we need to download it first.

```bash
wget https://downloads.apache.org/spark/spark-3.0.1/spark-3.0.1-bin-hadoop2.7.tgz
tar xzf spark-3.0.1-bin-hadoop2.7.tgz
cd spark-3.0.1-bin-hadoop2.7/
```

## Minikube

We will use Minikube, a Kubernetes variant running on a single computer.
This creates a full featured Kubernetes cluster inside a single computer.

### Start Minikube

We need some RAM for Spark.

```bash
minikube --memory 8192 --cpus 3 start
```

Minikube should be running now, and kubectl is configured to work with the local Minikube cluster.

Now we look for the mater URL, to be used later.

```bash
kubectl cluster-info
```

### Create docker images

The `bin/docker-imagetool.sh` script can be used for creating Docker images.

The `-m` switch makes the images directly accessible to Minikube.

It needs to be run in the directory where Spark was extracted into.

```bash

bin/docker-image-tool.sh -m -t v3.0.1 build

```

### Creating ServiceAccount and ClusterRoleBinding for Spark

We need to create a role to allow the creation of Kubernetes resources by the Spark library.

```bash
kubectl create namespace spark
kubectl create serviceaccount spark-serviceaccount --namespace spark
kubectl create clusterrolebinding spark-rolebinding --clusterrole=edit --serviceaccount=spark:spark-serviceaccount --namespace=spark
```

## Setup clojure dependencies

Spark on Kubernetes requires one specific spark dependency to be added, namely `org.apache.spark/spark_kubernetes_2.12`.
The minimal Clojure dependencies to get the following code work are these:

```bash
GENI_VERSION=$(wget -qO- https://raw.githubusercontent.com/zero-one-group/geni/develop/resources/GENI_REPL_RELEASED_VERSION)
clj -Sdeps "{:deps {zero.one/geni {:mvn/version \"$GENI_VERSION\" :exclusions [reply/reply]} org.apache.spark/spark-core_2.12 {:mvn/version \"3.0.1\" } org.apache.spark/spark-mllib_2.12 {:mvn/version \"3.0.1\"} org.apache.spark/spark-kubernetes_2.12 {:mvn/version \"3.0.1\"}}}"
```

This will start a Clojure REPL including the needed dependencies of Geni and Spark on Kubernetes.

## Create Spark session

The Spark session needs to be configured correctly in order to work with Spark on Kubernetes.
The minimal information needed is:

* a link to Kubernetes cluster API  "master";
* the name of Docker image to use to create the workers (local name for Minikube, else it needs to be prefixed with the Docker registry);
* a namespace to be used to create the workers;
* a service account with sufficient permissions; and
* the number of instance (or the Kubernetes pods to be launched).

In this configuration, the worker pods get started when the session gets created and teared down, when the session stops - typically when the REPL closes.

All possible Kubernetes related properties are listed [here](https://spark.apache.org/docs/latest/running-on-kubernetes.html#spark-properties).

Running the following code in the REPL, will create a Spark session and a Spark driver.
The driver will then launch three pods in Kubernetes using the provided Docker images.
Note that you will have to **change the `:spark.master` address** according to `kubectl cluster-info`.

```clojure
(require '[zero-one.geni.core :as g])

;; This should be the first function executed in the repl.

(def spark-session
  (g/create-spark-session
   {:app-name "my-app"
    :log-level "INFO" ;; default is WARN
    :configs
    {:spark.master "k8s://https://172.17.0.3:8443" ;;  might differ for you, its the output of kubecl cluster-info
     :spark.kubernetes.container.image "spark:v3.0.1" ;; this is for local docker images, works for minikube
     :spark.kubernetes.namespace "spark"
     :spark.kubernetes.authenticate.serviceAccountName "spark" ;; created above
     :spark.executor.instances 3}}))
```

Verify that `(g/spark-conf spark-session)` returns a map with the `:spark.master` key pointing to `k8s`.

## List running pods

We can list the pods started, like this:


```bash
kubectl get pods -n spark

NAME                             READY   STATUS    RESTARTS   AGE
my-app-a9f44174fa720b22-exec-1   1/1     Running   0          89s
my-app-a9f44174fa720b22-exec-2   1/1     Running   0          89s
my-app-a9f44174fa720b22-exec-3   1/1     Running   0          89s
```

## Shutting down

The moment we close the Clojure REPL session, the worker pods get deleted in Kubernetes.

## Reading files into Spark on Kubernetes

We have a lot of different options for reading files into Spark in a Kubernetes setup.
The main requirement is, that the files can be addressed under a common URL in the driver and all workers.

In Geni all file reading happens via functions of form  `g/read_xxx_!`. We can use different (eventually remote) file sytems by using different forms of the URL (specifically by using different prefixes).

The reading of  files by Geni can be achieved in 4 different ways:

0. __Copy the data files__ into the same path __in all Kubernetes pods__ and the driver machine. This is very adhoc and likely not suitable in most scenarios, as the data files would be lost at the end of the REPL session due to pods being terminated.

1. __Add data files to custom  Docker images__ used for Driver and Worker nodes. This is rather inconvenient as well, as any change in the data requires different Docker images

2. __Mount remote filesystem__ into the pod on __operating system level__. This makes them accessible as local files. See for example: [blobfuse](https://github.com/Azure/azure-storage-fuse). This has direct support for Kubenetes as well through [kubernetes volume driver](https://github.com/Azure/kubernetes-volume-drivers) (This is then the same as 3.)

3. __Mount Kubernetes data volumes into pods:__ Using a __local__ file path __and__ make sure that these path exist and work in all workers __and__ the driver. In Kubernets/Docker we can do this by __mounting__ (eventually remote) filesystems into the pods at certain location. Doing this, the files would look like local files and urls would not have a specific prefix, and be for example `(g/read "/data/test.csv")` This approach can eventually become easier if we make sure, that the driver is running as well inside Kubernetes. In the minikube setup above, the driver runs __outside__ Kubernetes.

4. __Using remote file systems:__ Spark can work with various types of remoe filesystem (hdfs, Amazon S3, Azure Storage based, others). This needs to be setup correctly (inside or outside Kubernetes). This might require 3 changes to the basic Minikube setup:

  * Create and use a Docker image which has all needed jars to support the remote file system

  * Use the correct options for the filesystem in use (like credentials) in the call to `g/create-spark-session`

  * Reading the files and using a file system specific prefix, like: `(g/read-csv! "wasbs://my_container@myblobstorage.blob.core.windows.net/my/file.csv")` for Azure blob Storage

 In this context we need to think about performance as well. For large data files, the different options might have very different performance characteristics. The cluster nodes should be as close (= as fast) as possible to the place of storage of the data files.
