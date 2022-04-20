---
title: Deploying Apache Kafka Part 1
categories: [Knowledge Base]
tags: [kafka, cluster setup]
date: 2022-04-21 00:08 +0530
---
## What is Apache Kafka
Apache Kafka is an open source event streaming tool that is similar to multiple publish / subscribe tools available but its popularity has been growing due to multiple large scale organizations using Kafka to drive their event driven systems. 
This article won't be covering details of Apache Kafka and will concentrate on deploying Apache kafka on linux systems in cluster mode with some basic configurations.

## Setting up the infrastructure
I will be using Google Cloud VM instances to create a 3 node cluster setup with 1 node for ansible controller which can be easily done even if you have Free Credits from Google. Steps for creating a GCP account with free credits can be easily found on multiple websites. 

Below is the mentioned configuration of VM instances (total 4 VMs):  
Machine Series: N2  
Machine Type: n2-standard-2  
vCPU: 2  
Memory: 8 GB  
Operating System: CentOS 7  

Once you have 4 VM instances deployed either on GCP or platform of your choice, lets start with some steps:
1. Setup Passwordless SSH from ansible controller node to other nodes.
2. Install Ansible controller node
   ```bash
   ssh node4
   yum install -y ansible
   ```
3. Create hosts file for ansible inventory as below
   ```bash
   [kafka]
   node1
   node2
   node3

   [zoo]
   node1
   node2
   node3

   [ansible]
   node4
   ```
4. Install Java
   ```bash
   ansible -i hosts kafka -m yum -a "name=java-11-openjdk state=present" -b
   ```

## Download and distribute software package
Apache Kafka's latest release as well as older releases can be downloaded from the below link:
[Apache Kafka Downloads](https://kafka.apache.org/downloads)

We will be using Apache Kafka 3.1.0 for our setup. Once the pre-requisites are completed, proceed with the below steps:
1. Download archive
   ```bash
   wget https://dlcdn.apache.org/kafka/3.1.0/kafka_2.13-3.1.0.tgz
   ```
2. Distribute and extract the package on kafka nodes
   ```bash
   ansible -i hosts kafka -m unarchive -a "src=kafka_2.13-3.1.0.tgz dest=/opt/" -b
   ansible -i hosts kafka -m shell -a "cd /opt/; ln -s kafka_2.13-3.1.0 kafka" -b
   ```

## Configuration setup for Zookeeper
With Kafka 2.8 Kraft was introduced to replace Zookeeper's requirement in setting up the Kafka clusters but it is still not advised to be used in Production clusters. So, we will be going ahead with setup of zookeeper for storing Kafka's metadata.
1. Create zookeeper user
   ```bash
   ansible -i hosts zoo -m shell -a "groupadd kafka" -b
   ansible -i hosts zoo -m shell -a "useradd zookeeper -g kafka" -b
   ```
2. Create zookeeper.properties file on the ansible node
   ```bash
   dataDir=/zoodata/zookeeper
   clientPort=2181
   maxClientCnxns=0
   admin.enableServer=true
   admin.serverPort=4888
   initLimit=5
   syncLimit=2
   tickTime=2000
   server.1=node1:2888:3888
   server.2=node2:2888:3888
   server.3=node3:2888:3888
   ```
3. Distribute the zookeeper.properties file to all the zookeeper nodes
   ```bash
   ansible -i hosts zoo -m copy -a "src=zookeeper.properties dest=/etc/zookeeper/conf/ mode=0644 owner=zookeeper group=kafka" -b
   ```
4. Create the required log and data directories
   ```bash
   ansible -i hosts zoo -m shell -a "mkdir -p /zoodata/zookeeper" -b
   ansible -i hosts zoo -m shell -a "mkdir -p /var/log/zookeeper" -b
   ```
   /zoodata symbolises a mount point that is used to store zookeeper znodes' data.
5. Create the myid file on each node and add unique id on each hosts
   ```bash
   ansible -i hosts zoo -m shell -a "touch /zoodata/zookeeper/myid" -b
   ansible -i hosts zoo -m shell -a "echo 1 > /zoodata/zookeeper/myid" -b --limit node1
   ansible -i hosts zoo -m shell -a "echo 2 > /zoodata/zookeeper/myid" -b --limit node2
   ansible -i hosts zoo -m shell -a "echo 3 > /zoodata/zookeeper/myid" -b --limit node3
   ```
6. Create a systemd file zookeeper.service with the below contents
   ```bash
   [Unit]
   Description=Apache ZooKeeper Service
   Documentation=http://zookeeper.apache.org
   Requires=network.target
   After=network.target
   
   [Service]
   Type=simple
   User=zookeeper
   Group=kafka
   Environment="KAFKA_HEAP_OPTS=-Xmx1g -Xms1g"
   Environment=LOG_DIR=/var/log/zookeeper
   ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /etc/zookeeper/conf/zookeeper.properties
   ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh /etc/zookeeper/conf/zookeeper.properties
   
   [Install]
   WantedBy=multi-user.target
   ```
7. Distribute the systemd file on all hosts
   ```bash
   ansible -i hosts zoo -m copy -a "src=zookeeper.service dest=/etc/systemd/system/ mode=0644" -b
   ```
8. Change Ownership of required files and directories
   ```bash
   ansible -i hosts zoo -m shell -a "chown -R zookeeper:kafka /zoodata/zookeeper" -b
   ansible -i hosts zoo -m shell -a "chown -R zookeeper:kafka /var/log/zookeeper" -b
   ansible -i hosts kafka -m shell -a "chown -R kafka:kafka /opt/kafka/*" -b
   ```
9.  Start the zookeeper service
   ```bash
   ansible -i hosts zoo -m shell -a "systemctl enable zookeeper; systemctl start zookeeper"
   ```

## Configuration setup for Kafka
Once the zookeeper quorum is achieved and we are able to test its functionality , we will be proceeding with Kafka's configuration settings and systemd service creation for easy management of the cluster.
1. Create Kafka user
   ```bash
   ansible -i hosts kafka -m shell -a "useradd kafka -g kafka" -b
   ```
2. Create server.properties file on ansible node
   ```bash
   broker.id.generation.enable=true
   port=9092
   controlled.shutdown.enable=true
   controlled.shutdown.max.retries=2
   group.max.session.timeout.ms=1800000
   group.min.session.timeout.ms=6000
   listeners=PLAINTEXT://:9092
   num.network.threads=3
   num.io.threads=8
   socket.send.buffer.bytes=102400
   socket.receive.buffer.bytes=102400
   socket.request.max.bytes=104857600
   log.dirs=/kafka/kafka-logs
   num.partitions=1
   num.recovery.threads.per.data.dir=1
   offsets.topic.replication.factor=3
   transaction.state.log.replication.factor=3
   transaction.state.log.min.isr=2
   log.retention.hours=168
   log.segment.bytes=1073741824
   log.retention.check.interval.ms=300000
   zookeeper.connect=node1:2181,node2:2181,node3:2181
   zookeeper.connection.timeout.ms=18000
   group.initial.rebalance.delay.ms=0
   ```
3. Distribute the server.properties file to all the kafka broker nodes
   ```bash
   ansible -i hosts kafka -m copy -a "src=server.properties dest=/etc/kafka/conf/ mode=0644 owner=kafka group=kafka" -b
   ```
4. Create the log and data directories
   ```bash
   ansible -i hosts kafka -m shell -a "mkdir -p /var/log/kafka" -b
   ansible -i hosts kafka -m shell -a "mkdir -p /kafka/kafka-logs" -b
   ```
   /kafka symbolises a mount point that is used to store kafka message log files. We can have multiple such mounts and each mount needs to be added in the server.properties file.
5. Create systemd file kafka.service with the below contents
   ```bash
   [Unit]
   Description=Apache Kafka Broker
   Documentation=https://kafka.apache.org/documentation.html
   Requires=network.target
   After=network.target
   
   [Service]
   Type=simple
   User=kafka
   Group=kafka
   Environment="JMX_PORT=9999"
   Environment="KAFKA_HEAP_OPTS=-Xmx4g -Xms4g"
   Environment=LOG_DIR=/var/log/kafka
   Environment="KAFKA_JMX_OPTS=-Djava.rmi.server.hostname=0.0.0.0 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.rmi.port=9999 -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.net.preferIPv4Stack=true"
   ExecStart=/opt/kafka/bin/kafka-server-start.sh /etc/kafka/conf/server.properties
   ExecStop=/opt/kafka/bin/kafka-server-stop.sh
   LimitNOFILE=100000
   TimeoutStopSec=180
   Restart=on-failure
   
   [Install]
   WantedBy=multi-user.target
   ```
6. Distribute the systemd file on all hosts
   ```bash
   ansible -i hosts kafka -m copy -a "src=kafka.service dest=/etc/systemd/system/ mode=0644" -b
   ```
7. Change the ownership of files and directories
   ```bash
   ansible -i hosts kafka -m shell -a "chown -R kafka:kafka /var/log/kafka" -b
   ansible -i hosts kafka -m shell -a "chown -R kafka:kafka /kafka/kafka-logs" -b
   ```
8. Start the Kafka service
   ```bash
   ansible -i hosts kafka -m shell -a "systemctl enable kafka; systemctl start kafka" -b
   ```

## Test the kafka cluster setup 
We can use below set of commands to test if we are able to produce and consume messages from the newly created kafka cluster.
1. Create test topic
   ```bash
   kafka-topics --bootstrap-server node1:9092,node2:9092,node3:9092 --create --topic testTopic --replication-factor 3 --partitions 4
   ```
2.  Start a kafka console producer
   ```bash
   kafka-console-producer --bootstrap-server node1:9092,node2:9092,node3:9092 --topic testTopic
   ```
   Enter some messages on the console and then exit the console with Ctrl+C
3. Start a kafka console consumer
   ```bash
   kafka-console-consumer --bootstrap-server node1:9092,node2:9092,node3:9092 --topic testTopic --from-beginning
   ```
   You should be able to see the messages that you entered in producer
   
Now we have a fully functional Kafka cluster. All the above steps can also be completed using ansible playbook easily to setup the cluster.

In the next part of this blog we will be installing MIT KDC, securing the Kafka clusters with kerberos, Enabling ACLs on topics etc. 


NOTE: I do not own any copyrights to mention any trademark products so any such reference should only be considered as part of educational references only.
