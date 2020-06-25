
# Storing an IoT data into MinIO S3 Compatible storage using Apache NiFi

Th objective of the project is to collect the IoT sensed data from sensors using light weight MQTT protocol and insert into MinIO object storage using Apache NiFi processors.
 The following are the requirements to run the project:
 * Apache Nifi
 * MQTT server
 * MinIO Object storage server
 
 
## Installing an Apache NiFi 
Apache NiFi can be installed in the system using docker container :

```
sudo docker run --name nifi \
  -p 8080:8080 \
  -p 8081:8081 \
  -d \
  -e NIFI_WEB_HTTP_PORT='8080' \
  apache/nifi:latest
  ```
  
  ## Installing MQTT server 
MQTT server can be installed on the same machine :
Mosquitto runs at port number:1883, access url could be tcp://INSTANCE_IP:1883
 ```
 sudo apt-get install mosquitto
 ``` 
 ```
 sudo apt-get install mosquitto-clients
 ``` 
## Installing the MinIO server :
Ref: https://docs.min.io/docs/minio-docker-quickstart-guide.html
Install MinIO using docker compose, However there are multiple ways of deploying MinIO server using docker such as docker swarm, docker compose or single docker container.
* Download docker-compose.yml from above mentioned link. it looks like.
```

version: '3.7'

# starts 4 docker containers running minio server instances. Each
# minio server's web interface will be accessible on the host at port
# 9001 through 9004.
services:
  minio1:
    image: minio/minio:RELEASE.2020-06-22T03-12-50Z
    volumes:
      - data1-1:/data1
      - data1-2:/data2
    ports:
      - "9001:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio2:
    image: minio/minio:RELEASE.2020-06-22T03-12-50Z
    volumes:
      - data2-1:/data1
      - data2-2:/data2
    ports:
      - "9002:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio3:
    image: minio/minio:RELEASE.2020-06-22T03-12-50Z
    volumes:
      - data3-1:/data1
      - data3-2:/data2
    ports:
      - "9003:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio4:
    image: minio/minio:RELEASE.2020-06-22T03-12-50Z
    volumes:
      - data4-1:/data1
      - data4-2:/data2
    ports:
      - "9004:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

## By default this config uses default local driver,
## For custom volumes replace with volume driver configuration.
volumes:
  data1-1:
  data1-2:
  data2-1:
  data2-2:
  data3-1:
  data3-2:
  data4-1:
  data4-2:

```
* run docker compose yaml file.
```
docker-compose up
```
You will see the list of addresses, where MinIO servers are running. Choose any one server, for example "http://host-ip:9002", you should see MinIO running.
![MinIO login](/images/minio1.png)

Enter Access Key:minio

Enter secret key : minio123

* Create a bucket by clicking on + symbol
![Bucker Create](/images/minio2.png)

## Creating Data pipeline using Apache NiFi
Access the NiFi user interface using http://ip:8086/nifi. Here we are creating two processor groups 

* Generate a sample iot data and publish as a MQTT topic 
* Consume MQTT topic and insert in to influxDB
![Overview of two Processors ](/images/processors-nifi.png)

### Creating a processor group to generate a sample iot data and publish as a MQTT topic 
The overall processor group looks like:
![Overview of two Processors ](/images/minio4.png)


* The GenerateFlowFile processor can be used to generate a sample iot data. 
Modify the property:= Custom Text: external=${random():mod(50):plus(50)},internal=${random():mod(200):plus(150)}
![GenerateFlowFile ](/images/GenerateFlowFile.png)

* Publish flowfile through MQTT topic
Modify the properties:=
Broker URI:tcp://ip:1883,client ID: you can choose freely , Topic:Data
![GenerateFlowFile ](/images/PublishMQTT.png)

### Creating a processor group to consume MQTT topic and insert into MinIO server

* ConsumeMQTT processor can be used to consume the MQTT topic. Modify the following properties such as Broker URL:tcp://ip:1883,client ID: you can choose freely , Topic:Data

![ConsumeMQTT ](/images/ConsumeMQTT.png)

* Format the data according to the line protocol structure of influxDB.
We are using ReplaceText processor to append the measurement name, tag values to flowfile recieved in the pipeline.

![ReplaceText ](/images/ReplaceText.png)

* PutS3Object processor can be used to insert data in to MinIo object storage server and modify the properties such as 

**Bucket : Name of bucket created in earlier step, Ex: simple

**Access Key ID :minio(you can get this value in docker-compose.yaml file)

**Secret Key ID : minio123(you can get this value in docker-compose.yaml file)

**Endpoint override URL: http://host-ip:9002

![S3 Storage ](/images/minio3.png)



