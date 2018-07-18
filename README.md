# EDCOP-INGEST

Table of Contents
-----------------
 
* [Configuration Guide](#configuration-guide)
	* [Image Repository](#image-repository)
	* [Networks](#networks)
	* [Logstash Configuration](#logstash-configuration)
		* [Features](#features)
			* [Syslog](#syslog)
			* [Packetbeat](#packetbeat)
			* [Winlogbeat](#winlogbeat)
			* [Bro](#bro)
			* [Suricata](#suricata)
			* [Custom Features](#custom-features)
		* [Resources](#resources)
	* [Redis Configuration](#redis-configuration)
		* [High Availability](#high-availability)
		* [External Logs](#external-logs)
		* [Resources](#resources)
		
		
# Configuration Guide

Within this configuration guide, you will find instructions for modifying the Helm chart for EDCOP nodes with the label ```ingest=true```. All changes should be made in the ```values.yaml``` file.
Please share any bugs or feature requests via GitHub issues.
 
## Image Repository

By default, images are pulled from each tool's official repository and configuration is added on top. If you're providing a custom image, we cannot guarantee that it will be compatible. 
 
```
images:
  logstash: docker.elastic.co/logstash/logstash:6.3.1
  redis: redis:4.0.9
```
 
## Networks

Ingest nodes only require the default calico interface because they only recieve and send data across the cluster. No further interfaces are necessary to accept external log traffic. 

```
networks:
  overlay: calico
```
 
To find the names of your networks, use the following command:
 
```
# kubectl get networks
NAME		AGE
calico		1d
passive		1d
inline-1	1d
inline-2	1d
```

## Logstash Configuration

Logstash is currently deployed as a Daemonset to all ingest nodes which allows for automatic scaling when more ingest nodes are added.

### Features

Each feature (tool) contains its own configuration and can be disabled in the ```values.yaml``` file. You can prioritize resource allocation by giving certain tools more resources within the pipeline sections. 
*Note: at least one feature needs to be enabled for Logstash to work*

Each feature has a ```pipeline``` section in which you can set Logstash pipeline settings for that specific feature. In general, the higher these settings, the more effort Logstash puts into processing that pipeline. Keep in mind, Logstash is still limited by the resources you give it in the next section. 

*Please make sure to read the [Logstash Performance Tuning Guide](https://www.elastic.co/guide/en/logstash/current/performance-troubleshooting.html) for a better understanding of managing Logstash's resources.*

```
pipeline:
  threads: 2
  batchCount: 250
  pipelineWorkers: 2
  pipelineOutputWorkers: 2
  pipelineBatchSize: 150
``` 

#### Syslog

You can enable Syslog by setting ```enabled``` to ```true```, but remember that this exposes the specified port on ALL ingest nodes. This port must be within the default 30000 - 32767 Kubernetes NodePort range. 

*Please make sure that this port is usable across all ingest nodes, otherwise you will have issues.*

```
logstashConfig:
  features:
    syslog:
      enabled: true
      port: 30144
      pipeline:
        threads: 2
        batchCount: 250
        pipelineWorkers: 2
        pipelineOutputWorkers: 2
        pipelineBatchSize: 150
``` 

In order to send syslogs to your EDCOP cluster, you must forward them to the IP of one of your ingest nodes (preferably the master) while also specifying the port number. After entering the cluster, these logs will be loadbalanced across all Logstash instances. 

#### Packetbeat

You can enable the Packetbeat pipeline by setting ```enabled``` to ```true```, which allows Logstash to pull Packetbeat logs from Redis using the key *packetbeat*. 

*Note: the current EDCOP tools include their own instances of Redis + Logstash. If you want to point Packetbeat's logs to the ingest node instances, you will need to edit the* ```packetbeat.yml``` *file to send logs to* ```ingest-service``` *on port 6379.*

```
logstashConfig:
  features:
    packetbeat:
      enabled: true
      pipeline:
        threads: 2
        batchCount: 250
        pipelineWorkers: 2
        pipelineOutputWorkers: 2
        pipelineBatchSize: 150
``` 

#### Winlogbeat

You can enable the Winlogbeat pipeline by setting ```enabled``` to ```true```, which allows Logstash to pull Winlogbeat logs from Redis using the key *winlogbeat*. This **requires** the use of externally available Redis in order to ingest logs from hosts outside of EDCOP. 

```
logstashConfig:
  features:
    winlogbeat:
      enabled: true
      pipeline:
        threads: 2
        batchCount: 250
        pipelineWorkers: 2
        pipelineOutputWorkers: 2
        pipelineBatchSize: 150
``` 
Before installing Winlogbeat, you will need Windows logs forwarded to a centralized logging location. To do this, please follow Microsoft's instructions available from these guides: https://blogs.technet.microsoft.com/jepayne/2015/11/23/monitoring-what-matters-windows-event-forwarding-for-everyone-even-if-you-already-have-a-siem/ and https://www.syspanda.com/index.php/2017/03/01/setting-up-windows-event-forwarder-server-wef-domain-part-13/

When installing Winlogbeat on a Windows server, please use version **6.3.1** as it is the most recent version that works with EDCOP-INGEST. Download the zip file from https://www.elastic.co/downloads/past-releases/winlogbeat-6-2-4 and then follow the instructions available from https://www.elastic.co/guide/en/beats/winlogbeat/6.2/winlogbeat-installation.html. Be sure to use the aforementioned zip file as the download link in the install instructions points to the most current version (6.3.1). 

In order to point your Winlogbeat logs to Redis, you need to edit the ```winlogbeat.yml``` on the host system of where it lives. Please **disable** all other outputs and enable the Redis output as shown below. Remember to replace the ```$HOST-IP``` with the IP of one of the ingest nodes and the ```$REDIS-NODEPORT``` with the port you have chosen within the Redis section of this guide. After installing and configuring, you can start Winlogbeat from ```services.msc```. 

*For more information on configuring Winlogbeat, please refer to the [Configuring Winlogbeat Guide](https://www.elastic.co/guide/en/beats/winlogbeat/current/configuring-howto-winlogbeat.html).*

```
output.redis:
  hosts: ["$HOST-IP:$REDIS-NODEPORT"]
  #password: "my_password"
  key: "winlogbeat"
  db: 0
  timeout: 5
```

**All winlogbeat filters, dashboards, templates, and logstash config are provided by [HELK](https://github.com/Cyb3rWard0g/HELK) and adapted for EDCOP. All Credit goes to Cyb3rWard0g for his amazing ELK stack work for windows log filtering.**

#### Bro

You can enable the Bro pipeline by setting ```enabled``` to ```true```, which allows Logstash to pull Bro logs from Redis using the key *bro*. 

*Note: the current EDCOP tools include their own instances of Redis + Logstash. If you want to point Bro's logs to the ingest node instances, you will need to edit the* ```filebeat.yml``` *file to send the Bro logs Filebeat collects to* ```ingest-service``` *on port 6379.*

```
logstashConfig:
  features:
    bro:
      enabled: true
      pipeline:
        threads: 2
        batchCount: 250
        pipelineWorkers: 2
        pipelineOutputWorkers: 2
        pipelineBatchSize: 150
``` 

#### Suricata

You can enable the Suricata pipeline by setting ```enabled``` to ```true```, which allows Logstash to pull Suricata logs from Redis using the key *suricata*. 

*Note: the current EDCOP tools include their own instances of Redis + Logstash. If you want to point Suricata's logs to the ingest node instances, you will need to edit the* ```suricata.yaml``` *file to send logs to* ```ingest-service``` *on port 6379.*

```
logstashConfig:
  features:
    suricata:
      enabled: true
      pipeline:
        threads: 2
        batchCount: 250
        pipelineWorkers: 2
        pipelineOutputWorkers: 2
        pipelineBatchSize: 150
``` 

#### Custom Features

If you need additional feature capabilities, you can add your own pipeline + configuration in the ```logstash-pipelines.yaml``` and ```logstash-config``` files. Afterwards, you can define your settings in the ```values.yaml``` so they're configurable in the future. 

*You can also open a GitHub issue requesting we add a feature if it can be repeated across clusters easily. If you have developed a feature and would like to share it, please submit a merge request detailing what it is and how it works. Thanks!*

### Resources

The resources seciton of Logstash's configuration allows you to set limits on the amount of resources it can use across ALL pipelines. In general, the initialJvmHeap and maxJvmHeap should always be equal. Keep in mind the amount of logs you're ingesting and the number of pipelines you're running when deciding on the amount of resources Logstash is limited to. 

*Note: Request defaults are set to accomodate limited resource VMs*

```
logstashConfig:
  resources:
    initialJvmHeap: 4g
    maxJvmHeap: 4g
	requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 2
      memory: 8G
```

## Redis Configuration

Redis is also included as part of EDCOP ingest nodes due to its role as a buffer for Logstash. If you do not want HA Redis, the Redis servers will be deployed as a DaemonSet to all ingest nodes. 

### High Availability

Redis is capable of being deployed in HA mode for clusters with 2+ nodes, but before you do so please read the following for a conceptual overview: https://redislabs.com/redis-features/high-availability. 

In order to enable HA Redis for your EDCOP cluster, you will need to change the Redis image value at the top of the ```values.yaml``` file. The most recent version of HA Redis will be commented above this value.

```
images:
  logstash: docker.elastic.co/logstash/logstash:6.3.1
  redis: quay.io/smile/redis:4.0.9r0
```

Afterwards, you will need to enable it by setting the ```enabled``` option to ```true```. 

```
redisConfig:
  highAvailability:
    enabled: true
```

Since HA mode replicates data across Redis nodes, you do not want too many instances otherwise you're wasting resources and bandwidth. You can decide how many Redis servers and sentinels to deploy, but remember that you need at least two of each for it to work. 

```
redisConfig:
  highAvailability:
    enabled: true
	servers: 2
    sentinels: 2 
```

### External Logs

If you have applications running outside of the cluster, but would like to send logs to EDCOP, you can set ```external.enabled``` to ```true``` to expose a NodePort for external access. As previously stated with Syslog, this port must be within the default Kubernetes NodePort range (30000 - 32767) and must be usable across all ingest nodes. 

```
redisConfig:
  external:
    enabled: true
	port: 30379
```

### Resources

As with Logstash, you can set resource limits on Redis to ensure it doesn't hog all of your node's resources, especially if running in VMs. If you aren't deploying HA Redis, the sentinel portion is ignored. 

```
redisConfig:
  resources:
    server:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 2
        memory: 8G
    sentinel:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 2
        memory: 2G
```
