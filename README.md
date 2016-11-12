# Flink on Kubernetes

Very much influenced by [Spark Example](https://github.com/kubernetes/application-images/tree/master/spark)

## Sources

The Docker images are based on https://github.com/melentye/flink-docker

## Step Zero: Prerequisites

- You have a Kubernetes cluster installed and running. Use [Minikube](https://github.com/kubernetes/minikube) for local testing.
- You have the ```kubectl``` command line tool somewhere in your path.

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
[`flink-jobmanager-controller.yaml`](flink-jobmanager-controller.yaml)
file to create a
[replication controller](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/replication-controller.md)
running the Flink Job Manager.

```console
$ kubectl create -f jobmanager-controller.yaml
replicationcontroller "jobmanager-controller" created
```

Then, use the
[`jobmanager-service.yaml`](jobmanager-service.yaml) file to
create a logical service endpoint that Flink Task Managers can use to access the Job Manager pod.

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
$ kubectl logs flink-master-controller-5u0q5
2016-06-19 21:48:52,039 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager actor
2016-06-19 21:48:52,042 INFO  org.apache.flink.runtime.blob.BlobServer                      - Created BLOB server storage directory /tmp/blobStore-c6d0abf6-b64b-47a4-9c9b-7e3b8ef7e53f
2016-06-19 21:48:52,050 INFO  org.apache.flink.runtime.blob.BlobServer                      - Started BLOB server at 0.0.0.0:33556 - max concurrent requests: 50 - max backlog: 1000
2016-06-19 21:48:52,061 INFO  org.apache.flink.runtime.checkpoint.SavepointStoreFactory     - Using job manager savepoint state backend.
2016-06-19 21:48:52,070 INFO  org.apache.flink.runtime.jobmanager.MemoryArchivist           - Started memory archivist akka://flink/user/archive
2016-06-19 21:48:52,072 INFO  org.apache.flink.runtime.jobmanager.JobManager                - Starting JobManager at akka.tcp://flink@192.168.137.3:6123/user/jobmanager.
2016-06-19 21:48:52,076 INFO  org.apache.flink.runtime.jobmanager.JobManager                - JobManager akka.tcp://flink@192.168.137.3:6123/user/jobmanager was granted leadership with leader session ID None.
2016-06-19 21:48:52,078 INFO  org.apache.flink.runtime.webmonitor.WebRuntimeMonitor         - Starting with JobManager akka.tcp://flink@192.168.137.3:6123/user/jobmanager on port 8080
2016-06-19 21:48:52,079 INFO  org.apache.flink.runtime.webmonitor.JobManagerRetriever       - New leader reachable under akka.tcp://flink@192.168.137.3:6123/user/jobmanager:null.
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

Use the [`taskmanager-controller.yaml`](taskmanager-worker-controller.yaml) file to create a
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