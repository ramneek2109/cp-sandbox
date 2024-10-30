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

# Scenario 9

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up

## Problem Statement

The client just performed a lift and shift on their entire platform to different machines. The brokers and several other components are down.

The brokers have the following error log - 

```
java.lang.RuntimeException: Received a fatal error while waiting for all of the authorizer futures to be completed.
	at kafka.server.KafkaServer.startup(KafkaServer.scala:950)
	at kafka.Kafka$.main(Kafka.scala:114)
	at kafka.Kafka.main(Kafka.scala)
Caused by: java.util.concurrent.CompletionException: org.apache.kafka.common.errors.SslAuthenticationException: SSL handshake failed
	at java.base/java.util.concurrent.CompletableFuture.encodeRelay(CompletableFuture.java:367)
	at java.base/java.util.concurrent.CompletableFuture.completeRelay(CompletableFuture.java:376)
	at java.base/java.util.concurrent.CompletableFuture$AnyOf.tryFire(CompletableFuture.java:1663)
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:506)
	at java.base/java.util.concurrent.CompletableFuture.completeExceptionally(CompletableFuture.java:2088)
	at io.confluent.security.auth.provider.ConfluentProvider.lambda$null$10(ConfluentProvider.java:543)
	at java.base/java.util.concurrent.CompletableFuture.uniExceptionally(CompletableFuture.java:986)
	at java.base/java.util.concurrent.CompletableFuture$UniExceptionally.tryFire(CompletableFuture.java:970)
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:506)
	at java.base/java.util.concurrent.CompletableFuture.completeExceptionally(CompletableFuture.java:2088)
	at io.confluent.security.store.kafka.clients.KafkaReader.lambda$start$1(KafkaReader.java:102)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:829)
```

###**Solution:**

Regenerate certificate using steps performed in previous scenarios.

Next step,

Incorrect User Entries 

In each broker’s server.properties file, the super.users and broker.users entries contain incorrect values. The entries use kafka-1, kafka-2, and kafka-3, but the correct identifiers are kafka1, kafka2, and kafka3. 

```
Incorrect lines: 

  

super.users=User:bob;User:kafka-1;User:kafka-2;User:kafka-3;User:mds;User:schemaregistryUser;User:controlcenterAdmin;User:connectAdmin 

    broker.users=User:kafka-1;User:kafka-2;User:kafka-3 
```

Corrected lines:

```
super.users=User:bob;User:kafka1;User:kafka2;User:kafka3;User:mds;User:schemaregistryUser;User:controlcenterAdmin;User:connectAdmin 

    broker.users=User:kafka1;User:kafka2;User:kafka3
```

Incorrect SSL Principal Mapping Rules  

Another error in the server.properties file is in the ssl.principal.mapping.rules configuration. The regular expression incorrectly includes special characters (._-) in the mapping. 

Incorrect line: 

```  

    ssl.principal.mapping.rules=RULE:^.*CN=([a-zA-Z0-9._-]*),.*$/$1/L,DEFAULT 
```
  

Corrected line: 

 ```
    ssl.principal.mapping.rules=RULE:^.*CN=([a-zA-Z0-9]*),.*$/$1/L,DEFAULT
```

Apply these changes to the server.properties file for all brokers (kafka1, kafka2, and kafka3). 

Issue resolved:

![image](https://github.com/user-attachments/assets/c41ca559-dd49-468f-acc5-cf2a14d0de68)

![image](https://github.com/user-attachments/assets/a907621b-17d6-4678-aeda-009fb0581d53)

![image](https://github.com/user-attachments/assets/fb1c7381-1adc-49fe-883e-1208ad067e85)



