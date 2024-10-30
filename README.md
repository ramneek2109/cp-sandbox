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

# Scenario 10

> **Before starting ensure that there are no other versions of the sandbox running**
> Run `docker-compose down -v` before starting

1. Start the scenario with `docker-compose up -d`
2. Wait for all services to be up and healthy `docker-compose ps`
3. Wait for the topics to be created. Check the control center(localhost:9021) to see if the topics - `domestic_orders` and `international_orders` are created

## Problem Statement

The client is able to produce to the `international_orders` topic but receives an error while producing and consuming from `domestic_orders` using the following commands -

```
kafka-console-producer --bootstrap-server kafka1:19092 --producer.config /opt/client/client.properties --topic <topic>

kafka-console-consumer --bootstrap-server kafka1:19092 --consumer.config /opt/client/client.properties --from-beginning --topic <topic> 
```

The client is using SASL/PLAIN over PLAINTEXT with the user `bob` and acks=1

The error message seen in the console producer and consumer fro `domestic_orders` - 

```
WARN [Producer clientId=console-producer] Connection to node 2 (kafka2/172.20.0.6:19093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
```

##**Solution**


![image](https://github.com/user-attachments/assets/aa53ab57-8847-4fc3-9040-1feab3e3b595)

Regenerate the certificates using steps mentioned in previous scenarios.

All the containers are up now, let's produce and consume 

![image](https://github.com/user-attachments/assets/587f7b68-9f39-419e-a75c-0607f696fb8f)


couln't see the topics and error is there in kfkclient

![image](https://github.com/user-attachments/assets/5f04c665-022b-4e2f-8008-9d4220bb9af0)

![image](https://github.com/user-attachments/assets/f13dcaf3-5427-4a04-85fb-5d2266400232)

The logs indicate that the Kafka client is experiencing issues connecting to a specific Kafka broker, identified as node 2 (at IP 172.18.0.8:19093)

**Cause**: The broker might not be configured to listen on the correct address or port. This could be due to settings in the server.properties file.
**Solution**: Check the listeners and advertised.listeners configurations in the broker's server.properties file. Ensure that the advertised listener matches the client's connection address.

In kafka2, server.properties below is the incorrect configuration:

```
listeners=CLIENT://:29092,BROKER://:29093,TOKEN://:29094

# Listener name, hostname and port the broker will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=CLIENT://kafka2:19093,BROKER://kafka2:29093,TOKEN://kafka2:29094
```
corrrect:

```
listeners=CLIENT://:29092,BROKER://:29093,TOKEN://:29094

# Listener name, hostname and port the broker will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=CLIENT://kafka2:29093,BROKER://kafka2:29093,TOKEN://kafka2:29094

```

The main issue lies in the advertised listeners. The port for the CLIENT listener in the advertised listeners (19093) does not match the port it is listening on (29092). This mismatch can prevent clients from connecting properly, especially if they are trying to connect to the advertised port.

We see, topics are created now,


![image](https://github.com/user-attachments/assets/21539023-fa7e-4356-bfaf-24c7831290ce)

We are able to produce and consume from the topic domestic_orders now,

![image](https://github.com/user-attachments/assets/b509f9a8-0632-4089-a304-7c387787ebe5)

![image](https://github.com/user-attachments/assets/4b6b5167-cd83-4007-af4d-4b8c7b5d9149)


