# kubernetes-elasticsearch-cluster
Elasticsearch (6.0.0) cluster on top of Kubernetes made easy.

### Table of Contents

* [Important Notes](#important-notes)
* [Pre-Requisites](#pre-requisites)
* [Build-Images(optional)](#build-images)
* [Test (deploying & accessing)](#test)
* [Pod anti-affinity](#pod-anti-affinity)
* [Availability](#availability)
* [Deploying with Helm](#helm)
* [Install plug-ins](#plugins)
* [Clean up with Curator](#curator)
* [Deploy Kibana](#kibana)
* [FAQ](#faq)
* [Troubleshooting](#troubleshooting)

## Abstract

Elasticsearch best-practices recommend to separate nodes in three roles:
* `Master` nodes - intended for clustering management only, no data, no HTTP API
* `Client` nodes - intended for client usage, no data, with HTTP API
* `Data` nodes - intended for storing and indexing data, no HTTP API

Given this, I'm going to demonstrate how to provision a production grade scenario consisting of 3 master, 2 client and 2 data nodes.

<a id="important-notes">

## (Very) Important notes

* Elasticsearch pods need for an init-container to run in privileged mode, so it can set some VM options. For that to happen, the `kubelet` should be running with args `--allow-privileged`, otherwise
the init-container will fail to run.

* By default, `ES_JAVA_OPTS` is set to `-Xms256m -Xmx256m`. This is a *very low* value but many users, i.e. `minikube` users, were having issues with pods getting killed because hosts were out of memory.
One can change this in the deployment descriptors available in this repository.

* As of the moment, Kubernetes pod descriptors use an `emptyDir` for storing data in each data node container. This is meant to be for the sake of simplicity and should be adapted according to one's storage needs.

* The [stateful](stateful) directory contains an example which deploys the data pods as a `StatefulSet`. These use a `volumeClaimTemplates` to provision persistent storage for each pod.

<a id="pre-requisites">

## Pre-requisites

* Kubernetes cluster with **alpha features enabled** (tested with v1.7.2 on top of [Vagrant + CoreOS](https://github.com/pires/kubernetes-vagrant-coreos-cluster)), thas's because curator
 is a CronJob object which comes from batch/v2alpha1, to enable it, just add
 `--runtime-config=batch/v2alpha1=true` into your kube-apiserver options.
* `kubectl` configured to access the cluster master API Server

<a id="build-images">

## Build images (optional)

Providing one's own version of [the images automatically built from this repository](https://github.com/pires/docker-elasticsearch-kubernetes) will not be supported. This is an *optional* step. One has been warned.

## Test

### Deploy

```
kubectl create -f es-discovery-svc.yaml
kubectl create -f es-svc.yaml
kubectl create -f es-master.yaml
```

Wait until `es-master` deployment is provisioned, and
```
kubectl create -f es-client.yaml
kubectl create -f es-data.yaml
```

Wait for containers to be in the `Running` state and check one of the Elasticsearch master nodes logs:
```
$ kubectl get svc,deployment,pods
NAME                          CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
svc/elasticsearch             10.100.255.244   <none>        9200/TCP   3m
svc/elasticsearch-discovery   10.100.39.4      <none>        9300/TCP   3m
svc/kubernetes                10.100.0.1       <none>        443/TCP    11m

NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/es-client   2         2         2            2           2m
deploy/es-data     2         2         2            2           2m
deploy/es-master   3         3         3            3           3m

NAME                            READY     STATUS    RESTARTS   AGE
po/es-client-1608296982-h459v   1/1       Running   0          2m
po/es-client-1608296982-k81c0   1/1       Running   0          2m
po/es-data-217476735-jsg3d      1/1       Running   0          2m
po/es-data-217476735-stb6w      1/1       Running   0          2m
po/es-master-3685302915-9p50q   1/1       Running   0          3m
po/es-master-3685302915-b3r6p   1/1       Running   0          3m
po/es-master-3685302915-q9h27   1/1       Running   0          3m
```

```
$ kubectl logs es-master-3685302915-9p50q
[2017-11-29T07:58:43,817][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] initializing ...
[2017-11-29T07:58:43,961][INFO ][o.e.e.NodeEnvironment    ] [es-master-3685302915-9p50q] using [1] data paths, mounts [[/data (/dev/sda9)]], net usable_space [13.7gb], net total_space [15.5gb], types [ext4]
[2017-11-29T07:58:43,962][INFO ][o.e.e.NodeEnvironment    ] [es-master-3685302915-9p50q] heap size [247.5mb], compressed ordinary object pointers [true]
[2017-11-29T07:58:43,980][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] node name [es-master-3685302915-9p50q], node ID [wGjsPi7mThC-bsT9eKs4QA]
[2017-11-29T07:58:43,995][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] version[6.0.0], pid[10], build[8f0685b/2017-11-10T18:41:22.859Z], OS[Linux/4.14.0-rc8-coreos/amd64], JVM[Oracle Corporation/OpenJDK 64-Bit Server VM/1.8.0_131/25.131-b11]
[2017-11-29T07:58:43,996][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] JVM arguments [-XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction=75, -XX:+UseCMSInitiatingOccupancyOnly, -XX:+DisableExplicitGC, -XX:+AlwaysPreTouch, -Xss1m, -Djava.awt.headless=true, -Dfile.encoding=UTF-8, -Djna.nosys=true, -Djdk.io.permissionsUseCanonicalPath=true, -Dio.netty.noUnsafe=true, -Dio.netty.noKeySetOptimization=true, -Dlog4j.shutdownHookEnabled=false, -Dlog4j2.disable.jmx=true, -Dlog4j.skipJansi=true, -XX:+HeapDumpOnOutOfMemoryError, -Xms256m, -Xmx256m, -Des.path.home=/elasticsearch, -Des.path.conf=/elasticsearch/config]
[2017-11-29T07:58:46,528][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [aggs-matrix-stats]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [analysis-common]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [ingest-common]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [lang-expression]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [lang-mustache]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [lang-painless]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [parent-join]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [percolator]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [reindex]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [repository-url]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [transport-netty4]
[2017-11-29T07:58:46,537][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] loaded module [tribe]
[2017-11-29T07:58:46,538][INFO ][o.e.p.PluginsService     ] [es-master-3685302915-9p50q] no plugins loaded
[2017-11-29T07:58:51,282][INFO ][o.e.d.DiscoveryModule    ] [es-master-3685302915-9p50q] using discovery type [zen]
[2017-11-29T07:58:52,455][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] initialized
[2017-11-29T07:58:52,456][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] starting ...
[2017-11-29T07:58:52,882][INFO ][o.e.t.TransportService   ] [es-master-3685302915-9p50q] publish_address {10.244.100.2:9300}, bound_addresses {10.244.100.2:9300}
[2017-11-29T07:58:52,932][INFO ][o.e.b.BootstrapChecks    ] [es-master-3685302915-9p50q] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-11-29T07:58:56,242][INFO ][o.e.c.s.ClusterApplierService] [es-master-3685302915-9p50q] detected_master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300}, added {{es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300},}, reason: apply cluster state (from master [master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300} committed version [1]])
[2017-11-29T07:58:56,248][INFO ][o.e.n.Node               ] [es-master-3685302915-9p50q] started
[2017-11-29T07:59:02,539][INFO ][o.e.c.s.ClusterApplierService] [es-master-3685302915-9p50q] added {{es-master-3685302915-b3r6p}{tUpw_BSqT7SJCQpzh9HeQA}{9wVuSnD9TamxckgaZ0Sn5A}{10.244.19.3}{10.244.19.3:9300},}, reason: apply cluster state (from master [master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300} committed version [3]])
[2017-11-29T07:59:46,168][INFO ][o.e.c.s.ClusterApplierService] [es-master-3685302915-9p50q] added {{es-client-1608296982-h459v}{AleFXqSoTOGKtAGgocwVVw}{D5Qli2QdSHKMCI7R4G2lMw}{10.244.100.3}{10.244.100.3:9300},}, reason: apply cluster state (from master [master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300} committed version [4]])
[2017-11-29T07:59:46,401][INFO ][o.e.c.s.ClusterApplierService] [es-master-3685302915-9p50q] added {{es-data-217476735-stb6w}{DxuJBDzdQVuvYI3zbOHGcw}{W4cJwqqqSqW4RYWYoj9Ytw}{10.244.94.3}{10.244.94.3:9300},}, reason: apply cluster state (from master [master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300} committed version [5]])
[2017-11-29T07:59:54,592][INFO ][o.e.c.s.ClusterApplierService] [es-master-3685302915-9p50q] added {{es-data-217476735-jsg3d}{-OxX9CW2Sk6Km31QdLHNHg}{uY1aGuGiSHCy3GYtT4EEPQ}{10.244.19.5}{10.244.19.5:9300},}, reason: apply cluster state (from master [master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300} committed version [6]])
[2017-11-29T08:00:01,937][INFO ][o.e.c.s.ClusterApplierService] [es-master-3685302915-9p50q] added {{es-client-1608296982-k81c0}{98MsXSUWQzmMbVAVXIn2lQ}{RAXI3leASyazCRMZwcm75A}{10.244.19.4}{10.244.19.4:9300},}, reason: apply cluster state (from master [master {es-master-3685302915-q9h27}{1sVm5HVdTEeIFTP5jGLKUg}{VM0V9MGpSFquOhTOJlK1Nw}{10.244.94.2}{10.244.94.2:9300} committed version [7]])
```

As we can assert, the cluster is up and running. Easy, wasn't it?

### Access the service

*Don't forget* that services in Kubernetes are only acessible from containers in the cluster. For different behavior one should [configure the creation of an external load-balancer](http://kubernetes.io/v1.1/docs/user-guide/services.html#type-loadbalancer). While it's supported within this example service descriptor, its usage is out of scope of this document, for now.

*Note:* if you are using one of the cloud providers which support external load balancers, setting the type field to "LoadBalancer" will provision a load balancer for your Service. You can uncomment the field in [es-svc.yaml](https://github.com/pires/kubernetes-elasticsearch-cluster/blob/master/es-svc.yaml).
```
$ kubectl get svc elasticsearch
NAME            CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
elasticsearch   10.100.255.244   <none>        9200/TCP   4m
```

From any host on the Kubernetes cluster (that's running `kube-proxy` or similar), run:

```
$ curl http://10.100.255.244:9200
```

One should see something similar to the following:

```json
{
  "name" : "es-client-1608296982-h459v",
  "cluster_name" : "myesdb",
  "cluster_uuid" : "yere2SBDSXmU4Ap47Pav0w",
  "version" : {
    "number" : "6.0.0",
    "build_hash" : "8f0685b",
    "build_date" : "2017-11-10T18:41:22.859Z",
    "build_snapshot" : false,
    "lucene_version" : "7.0.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

Or if one wants to see cluster information:

```
$ curl http://10.100.255.244:9200/_cluster/health?pretty
```

One should see something similar to the following:

```json
{
  "cluster_name" : "myesdb",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 7,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
<a id="pod-anti-affinity">

## Pod anti-affinity

One of the main advantages of running Elasticsearch on top of Kubernetes is how resilient the cluster becomes, particularly during
node restarts. However if all data pods are scheduled onto the same node(s), this advantage decreases significantly and may even
result in no data pods being available.

It is then **highly recommended**, in the context of the solution described in this repository, that one adopts [pod anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity-beta-feature)
in order to guarantee that two data pods will never run on the same node.

Here's an example:
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: role
              operator: In
              values:
              - data
          topologyKey: kubernetes.io/hostname
  containers:
  - (...)
```

<a id="availability">

## Availability

If one wants to ensure that no more than `n` Elasticsearch nodes will be unavailable at a time, one can optionally (change and) apply the following manifests:
```
kubectl create -f es-master-pdb.yaml
kubectl create -f es-data-pdb.yaml
```

**Note:** This is an advanced subject and one should only put it in practice if one understands clearly what it means both in the Kubernetes and Elasticsearch contexts. For more information, please consult [Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions).

<a id="helm">

## Deploying with Helm

[Helm](https://github.com/kubernetes/helm) charts for a basic (non-stateful) ElasticSearch deployment are maintained at https://github.com/clockworksoul/helm-elasticsearch. With Helm properly installed and configured, standing up a complete cluster is almost trivial:

```
$ git clone https://github.com/clockworksoul/helm-elasticsearch.git
$ helm install helm-elasticsearch
```

<a id="plugins">

## Install plug-ins

The image used in this repo is very minimalist. However, one can install additional plug-ins at will by simply specifying the `ES_PLUGINS_INSTALL` environment variable in the desired pod descriptors. For instance, to install [Google Cloud Storage](https://www.elastic.co/guide/en/elasticsearch/plugins/5.6/repository-gcs.html) and [S3](https://www.elastic.co/guide/en/elasticsearch/plugins/5.6/repository-s3.html) plug-ins it would be like follows:
```yaml
- name: "ES_PLUGINS_INSTALL"
  value: "repository-gcs,repository-s3"
```

**Note:** The X-Pack plugin does not currently work with the `quay.io/pires/docker-elasticsearch-kubernetes` image. See Issue #102

<a id="curator">

## Clean up with Curator

Additionally, one can run a [CronJob](http://kubernetes.io/docs/user-guide/cron-jobs/) that will periodically run [Curator](https://github.com/elastic/curator) to clean up indices (or do other actions on the Elasticsearch cluster).

```
kubectl create -f es-curator-config.yaml
kubectl create -f es-curator.yaml
```

Please, confirm the job has been created.

```
$ kubectl get cronjobs
NAME      SCHEDULE    SUSPEND   ACTIVE    LAST-SCHEDULE
curator   1 0 * * *   False     0         <none>
```

The job is configured to run once a day at _1 minute past midnight and delete indices that are older than 3 days_.

**Notes**

- One can change the schedule by editing the cron notation in `es-curator.yaml`.
- One can change the action (e.g. delete older than 3 days) by editing the `es-curator-config.yaml`.
- The definition of the `action_file.yaml` is quite self-explaining for simple set-ups. For more advanced configuration options, please consult the [Curator Documentation](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html).

If one wants to remove the curator job, just run:

```
kubectl delete cronjob curator
kubectl delete configmap curator-config
```

Various parameters of the cluster, including replica count and memory allocations, can be adjusted by editing the `helm-elasticsearch/values.yaml` file. For information about Helm, please consult the [complete Helm documentation](https://github.com/kubernetes/helm/blob/master/docs/index.md).

<a id="kibana>

## Kibana

**ATTENTION**: This is community supported so it most probably is out-of-date.

Additionally, one can also add Kibana to the mix. In order to do so, one must use a container image of Kibana without x-pack,
as it's not supported by the Elasticsearch container images used in this repository.

An image is already provided but one can build their own like follows:

```
FROM docker.elastic.co/kibana/kibana:5.5.1
RUN bin/kibana-plugin remove x-pack
```

If ones does provide their own image, one must make sure to alter the following files before deploying:

```
kubectl create -f kibana.yaml
kubectl create -f kibana-svc.yaml
```

Kibana will be available through service `kibana`, and one will be able to access it from within the cluster or
proxy it through the Kubernetes API Server, as follows:

```
https://<API_SERVER_URL>/api/v1/proxy/namespaces/default/services/kibana/proxy
```

One can also create an Ingress to expose the service publicly or simply use the service nodeport.
In the case one proceeds to do so, one must change the environment variable `SERVER_BASEPATH` to the match their environment.

## FAQ

### Why does `NUMBER_OF_MASTERS` differ from number of master-replicas?
The default value for this environment variable is 2, meaning a cluster will need a minimum of 2 master nodes to operate. If a cluster has 3 masters and one dies, the cluster still works. Minimum master nodes are usually `n/2 + 1`, where `n` is the number of master nodes in a cluster. If a cluster has 5 master nodes, one should have a minimum of 3, less than that and the cluster _stops_. If one scales the number of masters, make sure to update the minimum number of master nodes through the Elasticsearch API as setting environment variable will only work on cluster setup. More info: https://www.elastic.co/guide/en/elasticsearch/guide/1.x/_important_configuration_changes.html#_minimum_master_nodes


### How can I customize `elasticsearch.yaml`?
Read a different config file by settings env var `path.conf=/path/to/my/config/`. Another option would be to build one's own image from  [this repository](https://github.com/pires/docker-elasticsearch-kubernetes)

## Troubleshooting

### No up-and-running site-local

One of the errors one may come across when running the setup is the following error:
```
[2016-11-29T01:28:36,515][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.IllegalArgumentException: No up-and-running site-local (private) addresses found, got [name:lo (lo), name:eth0 (eth0)]
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:116) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:103) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.SettingCommand.execute(SettingCommand.java:54) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:96) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.cli.Command.main(Command.java:62) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:80) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:73) ~[elasticsearch-5.0.1.jar:5.0.1]
Caused by: java.lang.IllegalArgumentException: No up-and-running site-local (private) addresses found, got [name:lo (lo), name:eth0 (eth0)]
	at org.elasticsearch.common.network.NetworkUtils.getSiteLocalAddresses(NetworkUtils.java:187) ~[elasticsearch-5.0.1.jar:5.0.1]
	at org.elasticsearch.common.network.NetworkService.resolveInternal(NetworkService.java:246) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.network.NetworkService.resolveInetAddresses(NetworkService.java:220) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.network.NetworkService.resolveBindHostAddresses(NetworkService.java:130) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.TcpTransport.bindServer(TcpTransport.java:575) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.netty4.Netty4Transport.doStart(Netty4Transport.java:182) ~[?:?]
 	at org.elasticsearch.common.component.AbstractLifecycleComponent.start(AbstractLifecycleComponent.java:68) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.transport.TransportService.doStart(TransportService.java:182) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.common.component.AbstractLifecycleComponent.start(AbstractLifecycleComponent.java:68) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.node.Node.start(Node.java:525) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Bootstrap.start(Bootstrap.java:211) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:288) ~[elasticsearch-5.0.1.jar:5.0.1]
 	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:112) ~[elasticsearch-5.0.1.jar:5.0.1]
 	... 6 more
[2016-11-29T01:28:37,448][INFO ][o.e.n.Node               ] [kIEYQSE] stopping ...
[2016-11-29T01:28:37,451][INFO ][o.e.n.Node               ] [kIEYQSE] stopped
[2016-11-29T01:28:37,452][INFO ][o.e.n.Node               ] [kIEYQSE] closing ...
[2016-11-29T01:28:37,464][INFO ][o.e.n.Node               ] [kIEYQSE] closed
```

This is related to how the container binds to network ports (defaults to ``_local_``). It will need to match the actual node network interface name, which depends on what OS and infrastructure provider one uses. For instance, if the primary interface on the node is `p1p1` then that is the value that needs to be set for the `NETWORK_HOST` environment variable.
Please see [the documentation](https://github.com/pires/docker-elasticsearch#environment-variables) for reference of options.

In order to workaround this, set `NETWORK_HOST` environment variable in the pod descriptors as follows:
```yaml
- name: "NETWORK_HOST"
  value: "_eth0_" #_p1p1_ if interface name is p1p1, _ens4_ if interface name is ens4, and so on.
```

### (IPv6) org.elasticsearch.bootstrap.StartupException: BindTransportException

Intermittent failures occur when the local network interface has both IPv4 and IPv6 addresses, and Elasticsearch tries to bind to the IPv6 address first.
If the IPv4 address is chosen first, Elasticsearch starts correctly.

In order to workaround this, set `NETWORK_HOST` environment variable in the pod descriptors as follows:
```yaml
- name: "NETWORK_HOST"
  value: "_eth0:ipv4_" #_p1p1:ipv4_ if interface name is p1p1, _ens4:ipv4_ if interface name is ens4, and so on.
```
