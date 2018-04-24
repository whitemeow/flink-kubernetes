# Flink on Kubernetes

Forked from https://github.com/melentye/flink-kubernetes.  

Upgrade to latest flink image and upgrade the kubernetes configuration.

Add flink image building script changed based on [flink source](https://github.com/apache/flink/tree/master/flink-contrib/docker-flink)

## Sources

```console
./build.sh --from-release --flink-version 1.4.2 --hadoop-version 2.8 --scala-version 2.11 --image-name flink-pdd:v1.4.2

or

# download flink package from http://mirror.bit.edu.cn/apache/flink/flink-1.4.2/ to flik-docker/images
./build.sh --from-local-dist --image-name flink-pdd:v1.4.2
```

# Setup

## Step Zero: Prerequisites

- You have a pre installed Kubernetes cluster installed and running. 

## Step One: Create namespace

```console
$ kubectl create -f namespace.yaml
```

Now list all namespaces:

```console
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    3h
flink         Active    20m
kube-system   Active    3h
```

In order not to pass the `--namespace flink` argument each time, we define a context and use it:

```console
$ kubectl config set-context flink --namespace=flink --cluster=${YOUR_CLUSTER_NAME} --user=${YOUR_USER_NAME}
$ kubectl config use-context flink
```

You can find your cluster name and user name in kubernetes config in ~/.kube/config.

## Step Two: Start your Job Manager service

The Job Manager [service](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/services.md) is the master service for a Flink cluster.

Use the
[`jobmanager-controller.yaml`](jobmanager-deployment.yaml)
file to create a
[replication controller](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/replication-controller.md)
running the Flink Job Manager.

```console
$ kubectl create -f jobmanager-controller.yaml
replicationcontroller "jobmanager-controller" created
```

Then, use the
[`jobmanager-service.yaml`](jobmanager-service.yaml) file to
create a logical service endpoint that Flink Task Managers can use to access the Job Manager pod. It fact, even the Job
Manager pods will fail to start before executing this step, since they attempt to use the resulting DNS record on startup.

```console
$ kubectl create -f jobmanager-service.yaml
service "jobmanager" created
```

You can then create a service for the Flink Job Manager WebUI:

```console
$ kubectl create -f jobmanager-webui-service.yaml
service "jobmanager-webui" created
```

### Check to see if Job Manager is running and accessible

```console
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
jobmanager-controller-5u0q5     1/1       Running   0          8m
```

Check logs to see the status of the Job Manager. (Use the pod name retrieved on the previous step.)

```console
$ kubectl logs jobmanager-controller-5u0q5
[...]
--------------------------------------------------------------------------------
2016-11-12 21:34:32,100 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  Starting JobManager (Version: 1.1.3, Rev:3c95f71, Date:19.10.2016 @ 17:54:57 CEST)
2016-11-12 21:34:32,100 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  Current user: root
2016-11-12 21:34:32,100 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  JVM: OpenJDK 64-Bit Server VM - Oracle Corporation - 1.8/25.66-b17
2016-11-12 21:34:32,101 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  Maximum heap size: 245 MiBytes
2016-11-12 21:34:32,101 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  JAVA_HOME: /usr/lib/jvm/java-1.8-openjdk
2016-11-12 21:34:32,103 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  Hadoop version: 2.3.0
2016-11-12 21:34:32,103 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  JVM Options:
2016-11-12 21:34:32,103 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     -Xms256m
2016-11-12 21:34:32,103 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     -Xmx256m
2016-11-12 21:34:32,103 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     -Dlog.file=/opt/flink-1.1.3-custom-akka3/log/flink--jobmanager-0-jobmanager-controller-i4oc9.log
2016-11-12 21:34:32,103 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     -Dlog4j.configuration=file:/opt/flink-1.1.3-custom-akka3/conf/log4j.properties
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     -Dlogback.configurationFile=file:/opt/flink-1.1.3-custom-akka3/conf/logback.xml
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  Program Arguments:
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     --configDir
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     /opt/flink-1.1.3-custom-akka3/conf
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     --executionMode
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -     cluster
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                -  Classpath: /opt/flink-1.1.3-custom-akka3/lib/flink-dist_2.10-1.1.3.jar:/opt/flink-1.1.3-custom-akka3/lib/flink-python_2.10-1.1.3.jar:/opt/flink-1.1.3-custom-akka3/lib/log4j-1.2.17.jar:/opt/flink-1.1.3-custom-akka3/lib/slf4j-log4j12-1.7.7.jar:::
2016-11-12 21:34:32,104 INFO  org.apache.flink.runtime.jobmanager.JobManager                - --------------------------------------------------------------------------------
2016-11-12 21:34:32,106 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Registered UNIX signal handlers for [TERM, HUP, INT]
2016-11-12 21:34:32,222 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Loading configuration from /opt/flink-1.1.3-custom-akka3/conf
2016-11-12 21:34:32,232 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager without high-availability
2016-11-12 21:34:32,236 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager on 0.0.0.0:6123 with execution mode CLUSTER
2016-11-12 21:34:32,258 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Security is not enabled. Starting non-authenticated JobManager.
2016-11-12 21:34:32,265 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager
2016-11-12 21:34:32,266 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager actor system at 0.0.0.0:6123
2016-11-12 21:34:32,330 INFO  org.apache.flink.runtime.akka.AkkaUtils$                      - Using listening address "0.0.0.0":6123 and external address "10.0.0.240":6123
2016-11-12 21:34:32,577 INFO  akka.event.slf4j.Slf4jLogger                                  - Slf4jLogger started
2016-11-12 21:34:32,615 INFO  Remoting                                                      - Starting remoting
2016-11-12 21:34:32,740 INFO  Remoting                                                      - Remoting started; listening on addresses :[akka.tcp://flink@10.0.0.240:6123]
2016-11-12 21:34:32,745 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager web frontend
2016-11-12 21:34:32,765 INFO  org.apache.flink.runtime.webmonitor.WebMonitorUtils           - Determined location of JobManager log file: /opt/flink-1.1.3-custom-akka3/log/flink--jobmanager-0-jobmanager-controller-i4oc9.log
2016-11-12 21:34:32,765 INFO  org.apache.flink.runtime.webmonitor.WebMonitorUtils           - Determined location of JobManager stdout file: /opt/flink-1.1.3-custom-akka3/log/flink--jobmanager-0-jobmanager-controller-i4oc9.out
2016-11-12 21:34:32,808 INFO  org.apache.flink.runtime.webmonitor.WebRuntimeMonitor         - Using directory /tmp/flink-web-42636f9e-2e7b-468b-9a50-f3bf2a3ad196 for the web interface files
2016-11-12 21:34:32,808 INFO  org.apache.flink.runtime.webmonitor.WebRuntimeMonitor         - Using directory /tmp/flink-web-upload-7af4968b-96cd-4e3e-8fec-2430bdf32e09 for web frontend JAR file uploads
2016-11-12 21:34:33,020 INFO  org.apache.flink.runtime.webmonitor.WebRuntimeMonitor         - Web frontend listening at 0:0:0:0:0:0:0:0:8081
2016-11-12 21:34:33,021 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager actor
2016-11-12 21:34:33,026 INFO  org.apache.flink.runtime.blob.BlobServer                      - Created BLOB server storage directory /tmp/blobStore-66dae3f5-eabd-4f11-bbae-35a0222dd192
2016-11-12 21:34:33,027 INFO  org.apache.flink.runtime.blob.BlobServer                      - Started BLOB server at 0.0.0.0:40125 - max concurrent requests: 50 - max backlog: 1000
2016-11-12 21:34:33,031 INFO  org.apache.flink.runtime.checkpoint.savepoint.SavepointStoreFactory  - Using job manager savepoint state backend.
2016-11-12 21:34:33,034 INFO  org.apache.flink.runtime.metrics.MetricRegistry               - No metrics reporter configured, no metrics will be exposed/reported.
2016-11-12 21:34:33,039 INFO  org.apache.flink.runtime.jobmanager.MemoryArchivist           - Started memory archivist akka://flink/user/archive
2016-11-12 21:34:33,042 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager at akka.tcp://flink@10.0.0.240:6123/user/jobmanager.
2016-11-12 21:34:33,044 INFO  org.apache.flink.runtime.webmonitor.WebRuntimeMonitor         - Starting with JobManager akka.tcp://flink@10.0.0.240:6123/user/jobmanager on port 8081
2016-11-12 21:34:33,044 INFO  org.apache.flink.runtime.webmonitor.JobManagerRetriever       - New leader reachable under akka.tcp://flink@10.0.0.240:6123/user/jobmanager:null.
2016-11-12 21:34:33,051 INFO  org.apache.flink.runtime.clusterframework.standalone.StandaloneResourceManager  - Trying to associate with JobManager leader akka.tcp://flink@10.0.0.240:6123/user/jobmanager
2016-11-12 21:34:33,079 INFO  org.apache.flink.runtime.jobmanager.JobManager                - JobManager akka.tcp://flink@10.0.0.240:6123/user/jobmanager was granted leadership with leader session ID None.
2016-11-12 21:34:33,085 INFO  org.apache.flink.runtime.clusterframework.standalone.StandaloneResourceManager  - Resource Manager associating with leading JobManager Actor[akka://flink/user/jobmanager#1759222010] - leader session null
```

After you know the master is running, you can use the [cluster
proxy](http://kubernetes.io/docs/user-guide/accessing-the-cluster/#using-kubectl-proxy) to
connect to the Flink WebUI:

```console
kubectl proxy --port=8080 &
```

At which point the UI will be available at
[http://localhost:8080/api/v1/proxy/namespaces/flink/services/jobmanager-webui/](http://localhost:8080/api/v1/proxy/namespaces/flink/services/jobmanager-webui/).

## Step Three: Start your Flink Task Managers

Use the [`taskmanager-controller.yaml`](taskmanager-controller.yaml) file to create a
[replication controller](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/replication-controller.md) that manages the Task Manager pods.

```console
$ kubectl create -f taskmanager-controller.yaml
replicationcontroller "taskmanager-controller" created
```

### Check to see if the Task Managers are running

If you launched the Flink WebUI, your Task Managers should just appear in the UI when
they're ready. (It may take a little bit to pull the images and launch the
pods.) You can also interrogate the status in the following way:

```console
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
jobmanager-controller-5u0q5     1/1       Running   0          25m
taskmanager-controller-e8otp    1/1       Running   0          6m
taskmanager-controller-fiivl    1/1       Running   0          6m

$ kubectl logs jobmanager-controller-5u0q5
[...]
2016-11-12 21:36:25,651 INFO  org.apache.flink.runtime.instance.InstanceManager             - Registered TaskManager at taskmanager-controller-ypz70 (akka.tcp://flink@172.17.0.5:36580/user/taskmanager) as c23e0d5439461e2a8e50494974763c0c. Current number of registered hosts is 1. Current number of alive task slots is 2.
2016-11-12 21:36:25,653 INFO  org.apache.flink.runtime.clusterframework.standalone.StandaloneResourceManager  - TaskManager ResourceID{resourceId='4aa8f076fb7fa6f726d9292e56c9da0c'} has started.
2016-11-12 21:36:25,666 INFO  org.apache.flink.runtime.clusterframework.standalone.StandaloneResourceManager  - TaskManager ResourceID{resourceId='12ad33ad4d6c98a02081935133336500'} has started.
2016-11-12 21:36:25,667 INFO  org.apache.flink.runtime.instance.InstanceManager             - Registered TaskManager at taskmanager-controller-2chea (akka.tcp://flink@172.17.0.6:42142/user/taskmanager) as 6ba1e278196cab39238e407e8fee3833. Current number of registered hosts is 2. Current number of alive task slots is 4.
2016-11-12 21:36:25,968 INFO  org.apache.flink.runtime.instance.InstanceManager             - Registered TaskManager at taskmanager-controller-q8aei (akka.tcp://flink@172.17.0.4:36143/user/taskmanager) as 889f76511e7f778f18c2d4a9c75e6920. Current number of registered hosts is 3. Current number of alive task slots is 6.
```

Assuming you still have the `kubectl proxy` running from the previous section,
you should now see the Task Managers in the UI as well.

# Usage

## Connecting to Job Manager Web UI

See above for `kubectl proxy` examples. It works well on real Kubernetes cluster but if you are running 'minikube' you need to run this command that works for me:
```console
$ kubectl.exe  port-forward jobmanager-controller-ppnmx  8090:8081 
```
You need to adjust the pods of the jobmanager and the port you would like to forward to.

## Submitting Jobs

One option is to use Web UI, upload a JAR and submit a job from there.



### Flink Compatibility

[FLINK-2821](https://issues.apache.org/jira/browse/FLINK-2821) which is strictly speaking not a Flink bug, seems to be preventing Task Managers from talking to Job Managers because of the recipient IP address mismatch. Therefore we're using a Docker image with a custom Akka 3 build of Flink 1.1.3 instead of the vanilla 1.1.3.
