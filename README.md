# kafka-jmx-grafana-docker

Docker-compose file for Confluent Kafka with configuration mounted as properties files. Brings up Kafka and components with JMX metrics exposed and visualized using Prometheus and Grafana

## Start

```
docker-compose up -d
```

## Usage

The docker-compose file brings up 3 node kafka cluster with security enabled. Each service in the compose file has its properties/configurations mounted as a volume from a directory with the same name as the service.

Check the kafka server.properties for more details about the Kafka setup.

### Health

Check if all components are up and running using

```bash
docker-compose ps -a
# Ensure there are no Exited services and all containers have the status `Up`
```


### Client

To use a kafka client, exec into the `kfkclient` container which contains the Kafka CLI and other tools necessary for troubleshooting Kafka. THe `kfkclient` container also contains a properties file mounted to `/opt/client`, which can be used to define the client properties for communicating with Kafka.

```
docker exec -it kfkclient bash
```

### Logs

Check the logs of the respective service by its container name.

```bash
docker logs <container_name> # docker logs kafka1
```

### Restarting services

To restart a particular service - 

```bash
docker-compose restart <service_name> # docker-compose restart kafka1
# OR
docker-compose up -d --force-recreate <service_name> # docker-compose up -d --force-recreate kafka1
```

# Scenario 12

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up and healthy `docker-compose ps`
3. Wait for the topics to be created. Check the control center(localhost:9021) to see if the topics - `clickstream` is created

## Problem Statement

The client is unable to produce to the `clickstream` topic using the command -

```
kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic clickstream
```

The client is using SASL/PLAIN over PLAINTEXT with the user `bob` and acks=1

###**Solution:**


![image](https://github.com/user-attachments/assets/d7893988-f722-40b8-8746-79a0bfec4edc)

Error in kafka 2 logs:

![image](https://github.com/user-attachments/assets/4298df1a-7117-4c4f-87a4-95295771213b)

To resolve this, comment the below line:

```
- kafka2data:/var/lib/kafka/data
```
  Reason:

  ```
volumes:
    prometheus_data: {}
    grafana_data: {}
    kafka2data:
      driver_opts:
        type: tmpfs
        o: size=3145728
        device: tmpfs
```
So comment these lines from volumes:

Corrected: 
```
volumes:
    prometheus_data: {}
    grafana_data: {}
      #kafka2data:
      #driver_opts:
        #type: tmpfs
          #o: size=3145728
          #device: tmpfs

```
the volume kafka2data is configured to use tmpfs as a storage driver. Let's break down what this means and how it relates to your disk space issue.

Understanding tmpfs Volume
tmpfs Storage:

A tmpfs volume is stored in the host's memory (RAM) instead of on the disk. This means that it is temporary and data stored in tmpfs is lost when the container stops or restarts.
The line o: size=3145728 indicates that the tmpfs volume is limited to approximately 3 MB (3145728 bytes). This is a very small size for Kafka, which typically requires more disk space to store log data.
Impact on Kafka:

Kafka relies heavily on disk storage to handle message logs. Limiting the Kafka data volume to 3 MB will quickly lead to issues, including the "No space left on device" error, especially if the service tries to write more data than that.
Since tmpfs uses memory, this also means that if your system runs out of RAM, the container could behave unexpectedly or crash.

Now all the services are up, but topic 'clickstream' is yet not created.

**Error in broker logs:**

```
[2024-11-01 09:20:56,200] INFO [MergedLog partition=_confluent-telemetry-metrics-0, dir=/var/lib/kafka/data] Truncating to 0 has no effect as the largest offset in the log is -1 (kafka.log.MergedLog)
[2024-11-01 09:20:56,206] WARN [ReplicaFetcher replicaId=1, leaderId=2, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-3. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,206] WARN [ReplicaFetcher replicaId=1, leaderId=2, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-6. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,206] WARN [ReplicaFetcher replicaId=1, leaderId=2, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-9. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,207] WARN [ReplicaFetcher replicaId=1, leaderId=2, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-0. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,224] INFO [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Truncating partition _confluent-telemetry-metrics-4 with TruncationState(offset=0, completed=true) due to local high watermark 0 (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,224] INFO [MergedLog partition=_confluent-telemetry-metrics-4, dir=/var/lib/kafka/data] Truncating to 0 has no effect as the largest offset in the log is -1 (kafka.log.MergedLog)
[2024-11-01 09:20:56,224] INFO [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Truncating partition _confluent-telemetry-metrics-7 with TruncationState(offset=0, completed=true) due to local high watermark 0 (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,224] INFO [MergedLog partition=_confluent-telemetry-metrics-7, dir=/var/lib/kafka/data] Truncating to 0 has no effect as the largest offset in the log is -1 (kafka.log.MergedLog)
[2024-11-01 09:20:56,224] INFO [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Truncating partition _confluent-telemetry-metrics-10 with TruncationState(offset=0, completed=true) due to local high watermark 0 (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,224] INFO [MergedLog partition=_confluent-telemetry-metrics-10, dir=/var/lib/kafka/data] Truncating to 0 has no effect as the largest offset in the log is -1 (kafka.log.MergedLog)
[2024-11-01 09:20:56,224] INFO [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Truncating partition _confluent-telemetry-metrics-1 with TruncationState(offset=0, completed=true) due to local high watermark 0 (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,224] INFO [MergedLog partition=_confluent-telemetry-metrics-1, dir=/var/lib/kafka/data] Truncating to 0 has no effect as the largest offset in the log is -1 (kafka.log.MergedLog)
[2024-11-01 09:20:56,237] WARN [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-4. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,237] WARN [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-7. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,237] WARN [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-10. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,237] WARN [ReplicaFetcher replicaId=1, leaderId=3, fetcherId=0] Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-1. This error may be returned transiently when the partition is being created or deleted, but it is not expected to persist. (kafka.server.ReplicaFetcherThread)
[2024-11-01 09:20:56,615] INFO Created telemetry topic _confluent-telemetry-metrics (io.confluent.telemetry.exporter.kafka.KafkaExporter)
[2024-11-01 09:20:56,616] INFO App info kafka.admin.client for confluent-telemetry-reporter-local-producer unregistered (org.apache.kafka.common.utils.AppInfoParser)
[2024-11-01 09:20:56,618] INFO Metrics scheduler closed (org.apache.kafka.common.metrics.Metrics)
[2024-11-01 09:20:56,618] INFO Closing reporter org.apache.kafka.common.metrics.JmxReporter (org.apache.kafka.common.metrics.Metrics)
```
Reason:
Message: Received UNKNOWN_TOPIC_ID from the leader for partition _confluent-telemetry-metrics-x
Meaning: This warning suggests that the replica fetcher is attempting to fetch data for a topic that it cannot find. This can happen transiently during the creation or deletion of topics. In your case, it seems to be related to the telemetry metrics topics being created or deleted. Since the warning states that this is not expected to persist, it should resolve itself once the topic is fully initialized.

in clients/setup.sh

correct topic creation:

correct:

```
kafka-topics --bootstrap-server kafka1:19092 --command-config /opt/client/client.properties --create --topic clickstream --if-not-exists --replica-assignment 2

for x in {1..100}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic clickstream

sleep infinity

```
  
