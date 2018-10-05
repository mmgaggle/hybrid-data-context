# Background


# Prequisites

This tutorial runs on OpenShift, the easiest way to experience OpenShift is
using Minishift. For details about how to get setup Minishift on your local
machine I suggest the [Minishift Getting Started](https://docs.okd.io/latest/minishift/getting-started/installing.html) guide.

# Tutorial

## Minishift

Startup Minishift, let's get this party started!

```
minishift start
```

## Ceph Nano

Assuming this repository is your current working directory, we'll start by creating a Ceph Nano service that will serve as a private shared data context.

First, you will need to add add a security context to your user to allow them to run the Ceph Nano service.

```
oc --as system:admin adm policy add-scc-to-user anyuid \
   system:serviceaccount:myproject:default
```

Next, well use OpenShift [secrets](https://docs.openshift.com/container-platform/3.10/dev_guide/secrets.html) to store a set of credentials that the Ceph Nano service will create during it's bootstraping process. Later we'll show how these secrets can be exposed to applications in OpenShift through environmental variables.

```
oc create -f ceph-rgw-keys.yml
```

Finally, we'll create the Ceph Nano service and create a route to the object gateway

```
oc create -f ceph-nano.yml
oc expose pod ceph-nano-0 --type=NodePort
```

## Building a OpenShift Spark Image

The [radanalytics.io](https://radanalytics.io) community is focused on empoowering intelligent application development on the OpenShift Platform. One of the artifacts maintained by the community is incomplete Openshift Spark builder images [(openshift-spark-inc)](https://hub.docker.com/r/radanalyticsio/openshift-spark-inc/) that can be combined with a Spark tarball to create usable OpenShift Spark images (openshift-spark). I've created a custom Spark 2.3.2 tarball that includes Hadoop 2.8.5 for the purposes of this tutorial, and the following commands will combine it with the [radanalyticsio](https://radanalytics.io) incomplete OpenShift Spark builder images.

```
oc new-build --name=openshift-spark \
             --docker-image=radanalyticsio/openshift-spark-inc:latest \
             -e SPARK_URL=http://mmgaggle-bd.s3.amazonaws.com/spark-2.3.2-bin-hadoop-2.8.5.tgz \
             -e SPARK_MD5_URL=http://mmgaggle-bd.s3.amazonaws.com/spark-2.3.2-bin-hadoop-2.8.5.tgz.md5 \
             --binary
oc start-build openshift-spark
```

You can observe the image building process by tailing the buildconfig log.

```
oc logs -f buildconfig/openshift-spark
```

If you're tailing the log and the build completes, then you can press ctl-c to drop back to the shell.

## Building a Jupyter Notebook Image

Once the openshift-spark image is built, we can use it as a base image for building a base-notebook image. We'll use the base-notebook repository from [radanalyticsio](https://radanalytics.io) as a starting point. You may want to consider forking this repository and using your own copy if you want to build a set of commonly used libraries into the image.

```
oc new-build https://github.com/radanalyticsio/base-notebook \
             --docker-image="172.30.1.1:5000/myproject/openshift-spark:latest" \
             --strategy=docker
```

Again, you can observe the image building process by tailing the buildconfig log.

```
oc logs -f buildconfig/base-notebook
```

## Creating a Jupyter Notebook application

Once the base-notebook image is built, we can use it to deploy a Jupyter notebook application.

```
oc new-app -i myproject/base-notebook:latest \
           -e JUPYTER_NOTEBOOK_PASSWORD=developer \
           -e RGW_API_ENDPOINT=$(minishift openshift service ceph-nano-0 --url) \
           -e JUPYTER_NOTEBOOK_X_INCLUDE=https://raw.githubusercontent.com/mmgaggle/hybrid-data-context/master/hybrid-data-context.ipynb
```

Now expose the Ceph Nano credentials via the OpenShift secret we created earlier to the Jupyter notebook application's environment.

```
oc env --from=secret/ceph-rgw-keys dc/base-notebook
```

Finally, expose a route to the Jupyter notebook so we can access it from our browser.

```
oc expose svc/base-notebook
```



