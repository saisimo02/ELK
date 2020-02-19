In this section, I will explain how to install and set-up a Kafka cluster on Centos/RedHat. 
# Introduction
Apache Kafka is a scalable and high-throughtput messaging system designed to handle a huge amount of real-time data. You can deploy Kafka on a single server or multiple servers to build a distributed Kafka cluster for greater performance. We will show how to deploy a Kafka cluster with a minimum of 3 instances.

#  Architecture
1. Zookeeper : Which is used by Kafka to maintain state between the nodes of the cluster.
2. Kafka brokers : The “pipes” in our pipeline, which store and emit data.
3. Producers : That insert data into the cluster. (Filebeat for example)
4. Consumers : That read data from the cluster. (Logstash for example)


# Prerequisites:
There is 3 interestion paths to know:
- 3 CentOs servers and a user with sudo privileges.
- Large memory capacity, at least 4 GB of RAM
- OpenJDK installed

# Download Apache Kafka
Choose a kafka version in this website https://downloads.apache.org/kafka/  
For our example, we have chosen kafka 2.1.1

Create a new directory in `/opt`
```
$ sudo mkdir /opt/kafka && cd /opt/kafka
```
Download kafka and rename the file
``` 
$ sudo curl "https://downloads.apache.org/kafka/2.1.1/kafka_2.11-2.1.1.tgz" -o kafka.tgz
```
Extract the archive you downloaded
```
$ sudo tar -xvzf ~/Downloads/kafka.tgz --strip 1
```
Remove the archive
```
$ sudo rm -f kafka.tgz
```
* you have to do the same thing on all three instances.


# The key folders

- **/opt/kafka/config/:** In this folder, you can find the configuration files for zookeeper and kafka
- **/opt/kafka/bin/:** This folder contains .sh files used to run zookeeper, kafka, to add a new topic...
- **/opt/kafka/zookeeper/:** the zookeeper data directory.

# Configure the Kafka Server
open the kafka properties file:
```
$ sudo vi /opt/kafka/config/server.properties
```
Edit the `broker.id`, `advertised.host.name` and `zookeeper.connect`

For Example, for the first server:

```
broker.id=1
advertised.host.name=server1_ip
zookeeper.connect=server1_ip:2181,server2_ip:2181,server3_ip:2181
```
Do the same for the other instances by modifying the `broker.id` and the `advertised.host.name` and keep the other fields by default.

# Configure the Zookeeper server:

Add a new file myid for your zookeeper server. The value after `echo` should be different for each instance.  
Server1
```
$ sudo echo '1' > /opt/kafka.zookeeper/myid
```
Server2
```
$ sudo echo '2' > /opt/kafka.zookeeper/myid
```
...    
open the zookeeper properties file:
```
$ sudo vi /opt/kafka/config/zookeeper.properties
```
For each instance, put the appropriate ip address  
An example of the config file for server 1
```
dataDir=/opt/kafka/zookeeper
# the port at which the clients will connect
clientPort=2181
# disable the per-ip limit on the number of connections since this is a non-production config
maxClientCnxns=0

clientPortAddress=server1_ip
# disable the per-ip limit on the number of connections since this is a non-production config
tickTime=2000
initLimit=5
syncLimit=2
server.1=server1_ip:2888:3888
server.2=server2_ip:2888:3888
server.3=server3_ip:2888:3888

```
# Creating Systemd Unit Files for Kafka & Zookeeper
Create unit file `zookeeper.service`
```
$ sudo vi /etc/systemd/system/zookeeper.service
```
Enter the following unit definition into the file:
```
[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=root
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

```

Create unit file `kafka.service`
```
$ sudo vi /etc/systemd/system/kafka.service
```
Enter the following unit definition into the file:
```
[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=root
ExecStart=/bin/sh -c '/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties > /opt/kafka/kafka.log 2>&1'
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
Do the same for the other instances

# Start your kafka cluster
Start Zookeeper and kafka on each server:

```
$ sudo systemctl start zookeeper && sudo systemctl start kafka
```
To ensure that the service is properly started:

```
$ sudo systemctl status zookeeper
$ sudo systemctl status kafka
```
Ensure that your kafka servers communicate well with each other. 
```
$ cd /opt/kafka && ./bin/zookeeper-shell.sh server1_ip:2181 ls /brokers/ids
```
The output should be:
```
Connecting to server1_ip:2181

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[1, 2, 3] <----

```
# Create a new topics

Create two topics `csv-topic` and `log-topic`

```
$ cd /opt/kafka/bin && ./kafka-topics.sh --create --zookeeper server_ip:2181 --topic csv-topic --partitions 3 --replication-factor 3
$ cd /opt/kafka/bin && ./kafka-topics.sh --create --zookeeper server_ip:2181 --topic log-topic --partitions 3 --replication-factor 3
```
