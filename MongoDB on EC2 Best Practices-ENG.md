
# MongoDB on EC2 Best Practices

## Table of Contents

- [MongoDB on EC2 Best Practices](#mongodb-on-ec2-best-practices)
  - [Table of Contents](#table-of-contents)
  - [MongoDB 5.0 Replica Set Architecture on EC2 Installation](#mongodb-50-replica-set-architecture-on-ec2-installation)
    - [Replica Set Architecture](#replica-set-architecture)
    - [Installing the Primary Node](#installing-the-primary-node)
    - [Installing Secondary Nodes](#installing-secondary-nodes)
    - [Creating MongoDB 5.0 Replica Set Architecture](#creating-mongodb-50-replica-set-architecture)
  - [MongoDB on EC2 Database Parameter Templates](#mongodb-on-ec2-database-parameter-templates)
    - [Database Parameter Settings](#database-parameter-settings)
      - [net.maxIncomingConnections](#netmaxincomingconnections)
      - [net.bindIp](#netbindip)
      - [net.ipv6](#netipv6)
      - [storage.wiredTiger.engineConfig.cacheSizeGB](#storagewiredtigerengineconfigcachesizegb)
      - [storage.directoryPerDB](#storagedirectoryperdb)
      - [replication.oplogSizeMB](#replicationoplogsizemb)
      - [systemLog.logAppend](#systemloglogappend)
    - [mongod.conf](#mongodconf)
  - [Monitoring MongoDB with Prometheus and Grafana](#monitoring-mongodb-with-prometheus-and-grafana)
    - [Prerequisites](#prerequisites)
    - [Deployment Overview](#deployment-overview)
    - [Deployment Guide](#deployment-guide)
      - [1. Installing Docker on the Monitoring Server](#1-installing-docker-on-the-monitoring-server)
      - [2. Installing MongoDB Exporter on the Monitoring Server](#2-installing-mongodb-exporter-on-the-monitoring-server)
      - [3. Installing Go Environment on MongoDB Host](#3-installing-go-environment-on-mongodb-host)
      - [4. Installing Node Exporter on MongoDB Host](#4-installing-node-exporter-on-mongodb-host)
      - [5. Installing Prometheus to Collect Data from MongoDB Exporter](#5-installing-prometheus-to-collect-data-from-mongodb-exporter)
      - [6. Installing and Using Grafana](#6-installing-and-using-grafana)
      - [7. Viewing Monitoring Data](#7-viewing-monitoring-data)
      - [8. Alert Configuration](#8-alert-configuration)
  - [MongoDB on EC2 Backup and Restore](#mongodb-on-ec2-backup-and-restore)
    - [Recommended Backup and Restore Architecture](#recommended-backup-and-restore-architecture)
    - [MongoDB on EC2 Backup Solution](#mongodb-on-ec2-backup-solution)
    - [MongoDB on EC2 Restore Solution](#mongodb-on-ec2-restore-solution)
    - [Common MongoDB Backup and Restore Commands](#common-mongodb-backup-and-restore-commands)
  - [MongoDB Replica Set Architecture Expansion on EC2](#mongodb-replica-set-architecture-expansion-on-ec2)
    - [Replica Set Architecture](#replica-set-architecture-1)
    - [Storage Expansion](#storage-expansion)
    - [Instance Expansion](#instance-expansion)
    - [Replica Set Member Expansion](#replica-set-member-expansion)

## MongoDB 5.0 Replica Set Architecture on EC2 Installation

Installing MongoDB 5.0 on three EC2 nodes (R6i.2xLarge):

### Replica Set Architecture

![Replica Set Architecture Diagram](https://example.com/placeholder-image)

*Note: The original document contained images that have been replaced with placeholder links in the Markdown conversion*

Each node is configured as follows:

- Security group: mongodb-sg (network communication between nodes within the security group)
- Storage: GP3 2T (for storing MongoDB data files)
- Network: Three nodes deployed across three AZs

### Installing the Primary Node

1. Install MongoDB 5.0 software

```bash
sudo vi /etc/yum.repos.d/mongodb-org-5.0.repo
```

Add the following content:

```
[mongodb-org-5.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2/mongodb-org/5.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-5.0.asc
```

```bash
sudo yum install -y mongodb-org
```

2. View and modify the configuration file

```bash
sudo vim /etc/mongod.conf
```

Change the binding IP:

```
bindIp: 127.0.0.1  to  bindIp: 0.0.0.0
```

(Note: There should be a space between the colon and the IP address)

3. Start MongoDB

```bash
sudo systemctl start mongod.service
```

Or use another method to start:

```bash
sudo mongod --config /etc/mongod.conf&
```

To stop MongoDB:

```bash
sudo systemctl stop mongod.service
```

4. Set up MongoDB to start on boot

```bash
sudo systemctl enable mongod.service
```

5. Move MongoDB data files to 2T GP3 storage

Check raw device:

```bash
lsblk
```

Create MongoDB file system:

```bash
sudo mkfs.ext4 /dev/nvme1n1
sudo mkdir /mongodata
sudo mount /dev/nvme1n1 /mongodata
df -hT
```

6. Enable automatic mounting of the MongoDB data storage volume after EC2 host reboot:

```bash
sudo cp /etc/fstab /etc/fstab.orig
sudo blkid
```

Example output:
```
/dev/nvme1n1: UUID="03a545aa-3662-4137-ac44-f436098d26bc" TYPE="ext4"
```

Edit fstab file:

```bash
sudo vim /etc/fstab
```

Add a line:

```
UUID=03a545aa-3662-4137-ac44-f436098d26bc /mongodata ext4 defaults,nofail 0 2
```

7. Create MongoDB log directory:

```bash
sudo mkdir /mongodata/log
sudo systemctl stop mongod.service
```

8. Modify MongoDB Data path

```bash
sudo vim /etc/mongod.conf
```

Change the configuration file content to point data directory to: /mongodata; log file to: /mongodata/log

9. Copy existing MongoDB data files to the new data file directory:

Check if /mongodata is a separate mount point:

```bash
df -hT
```

Copy data and set permissions:

```bash
sudo cp -R /var/lib/mongo /mongodata
sudo chmod 777 -R /mongodata
sudo chown mongod:mongod /mongodata
sudo service mongod start
```

### Installing Secondary Nodes

Follow the same installation steps as the primary node (use the same security group and 2TB GP3 storage)

### Creating MongoDB 5.0 Replica Set Architecture

1. Modify the three MongoDB 5.0 nodes by adding the following to /etc/mongod.conf (maintain indentation format):

```yaml
replication:
  replSetName: "rs0"
```

2. Restart the MongoDB service:

```bash
sudo service mongod restart
```

3. Run MongoDB on any of the three nodes and execute (add the private IP addresses of each node):

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "10.0.0.1:27017" },
    { _id: 1, host: "10.0.0.2:27017" },
    { _id: 2, host: "10.0.0.3:27017" }
  ]
})
```

4. View replica set status:

```javascript
rs.status()
rs.conf()
```

5. Enable replica set reading (set on the two secondary nodes):

```javascript
rs.secondaryOk()
```

6. Connect to the primary node and create an admin user:

```javascript
use admin
db.createUser({
  user:"admin",
  pwd:"password",
  roles:[{role:"root", db:"admin"}]
})
```

## MongoDB on EC2 Database Parameter Templates

### Database Parameter Settings

#### net.maxIncomingConnections

The default value is 65536. It is recommended to set the connection parameters based on the instance memory size:

```
2c4g 3000
4c8g 6000
6c16g 9000
12c32g 12000
24c64g 16000
24c128g 21000
32c240g 42000
48c512g 60000
```

Assuming memory size is X GB:
- X < 128: log2(X / 2) * 3000
- X >= 128: log2(X / 64) * 21000

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-net.maxIncomingConnections)

#### net.bindIp

The default value is localhost. It is recommended to set it to ::,0.0.0.0

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.bindIp)

#### net.ipv6

The default value is false. It is recommended to set it to true

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.ipv6)

#### storage.wiredTiger.engineConfig.cacheSizeGB

Use the default value: max(50% of (RAM - 1 GB), 256 MB)

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-storage.wiredTiger.engineConfig.cacheSizeGB)

#### storage.directoryPerDB

The default value is false. It is recommended to set it to true

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-storage.directoryPerDB)

#### replication.oplogSizeMB

The default value is 5% of disk space. When a larger range of data change records is needed, it is recommended to change it to 10% of disk space

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-replication.oplogSizeMB)

#### systemLog.logAppend

The default value is false. It is recommended to set it to true

Reference documentation: [Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-systemLog.logAppend)

### mongod.conf

Reference configuration for a 2c4g instance with an 80GB mounted disk:

```yaml
# mongod.conf
# for documentation of all options, see:
# http://docs.mongodb.org/manual/reference/configuration-options/
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  logRotate: rename
  path: /var/log/mongodb/mongod.log
# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
  directoryPerDB: true
# engine:
# wiredTiger:
# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
# network interfaces
net:
  port: 27017
  bindIp: ::,0.0.0.0
  ipv6: true
  maxIncomingConnections: 3000
#security:
#operationProfiling:
replication:
  oplogSizeMB: 8196
#sharding:
## Enterprise-Only Options
#auditLog:
#snmp:
```

## Monitoring MongoDB with Prometheus and Grafana

This section describes how to use Prometheus, combined with mongodb-exporter, to collect MongoDB monitoring data, configure alerts, and view alerts; as well as using Grafana to display monitoring metrics. All three components are open-source tools widely used in various monitoring domains. The mongodb-exporter chosen is the Percona open-source version, which has been systematically designed for MongoDB monitoring data collection and provides an HTTP interface for Prometheus to import metrics and manage monitoring data.

### Prerequisites

- MongoDB 5.0 instance has been deployed
- An EC2 server has been deployed

### Deployment Overview

- EC2 is used to deploy mongodb-exporter, Prometheus, and Grafana
- Node-exporter is deployed on the MongoDB 5.0 instance host

![Monitoring Architecture Diagram](https://example.com/placeholder-image)

*Note: The original document contained images that have been replaced with placeholder links in the Markdown conversion*

### Deployment Guide

#### 1. Installing Docker on the Monitoring Server

```bash
# Install the latest Docker Community Edition package
sudo yum install -y docker

# Start Docker service
sudo service docker start

# Add ec2-user to the docker group to run Docker commands without sudo
sudo usermod -a -G docker ec2-user
```

#### 2. Installing MongoDB Exporter on the Monitoring Server

Prepare the percona/mongodb_exporter image:

```bash
# Pull Percona image
docker pull percona/mongodb_exporter:0.38

# Check if the image was successfully pulled
docker images
```

Start the container:

```bash
# Note the listening port 9104:9216, where 9104 is the server listening port and 9216 is the port inside the container
docker run -d -p 9104:9216 --restart=always --name=mongodb-exporter \
  percona/mongodb_exporter:0.38 --compatible-mode --discovering-mode \
  --collect-all --mongodb.uri=mongodb://admin:password@10.0.0.1:27017/admin?replicaSet=rs0
```

Verify if the container has started:

```bash
docker ps
netstat -ntlp | grep 9104

# Check data collection
curl http://127.0.0.1:9104/metrics
```

#### 3. Installing Go Environment on MongoDB Host

```bash
# Download Go
wget https://go.dev/dl/go1.20.1.linux-amd64.tar.gz

# Extract to /usr/local directory
sudo tar -C /usr/local -xzf go1.20.1.linux-amd64.tar.gz

# Set environment variables
sudo vi ~/.bash_profile
# Add the following line
export PATH=$PATH:/usr/local/go/bin

# Reload configuration file
source ~/.bash_profile

# Verify Go version
go version
```

#### 4. Installing Node Exporter on MongoDB Host

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
tar -xzf node_exporter-1.1.2.linux-amd64.tar.gz
sudo cp node_exporter-1.1.2.linux-amd64/node_exporter /usr/local/bin/
node_exporter &

# Verify node_exporter is running
netstat -lntp | grep 9100
```

#### 5. Installing Prometheus to Collect Data from MongoDB Exporter

```bash
# Install Go environment
yum install go
go version

# Install Prometheus
cd ~/mongo_monitor
wget https://github.com/prometheus/prometheus/releases/download/v2.11.0-rc.0/prometheus-2.11.0-rc.0.linux-amd64.tar.gz
sudo tar xvf prometheus-2.11.0-rc.0.linux-amd64.tar.gz -C /usr/local/
sudo ln -sv /usr/local/prometheus-2.11.0-rc.0.linux-amd64/ /usr/local/prometheus

cd /usr/local/prometheus
```

Modify the prometheus.yml configuration file:

```yaml
scrape_configs:
  # Monitor MongoDB
  - job_name: 'mongodb-monitor'
    static_configs:
    - targets: ['127.0.0.1:9104']
```

Start the Prometheus service:

```bash
nohup /usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --web.listen-address=:9090 --web.enable-lifecycle &
ps -ef | grep prome
```

Access the Prometheus interface:
http://your-prometheus-server:9090/graph

#### 6. Installing and Using Grafana

```bash
# Install Grafana
cd /home/ec2-user/mongo_monitor
wget https://dl.grafana.com/oss/release/grafana-6.2.5-1.x86_64.rpm
sudo yum localinstall grafana-6.2.5-1.x86_64.rpm

# Start Grafana service
sudo service grafana-server start
```

Access the Grafana frontend page:
http://your-grafana-server:3000
Default username/password: admin/admin

Add Prometheus as a data source in Data Sources and create a Dashboard.
Recommended template IDs:
- node-exporter: 10795
- mongodb-exporter-0.38: 7353

#### 7. Viewing Monitoring Data

View MongoDB monitoring data through the Grafana interface.

#### 8. Alert Configuration

Configure Prometheus alert rules:

```bash
# Create alert rules folder
cd /usr/local/prometheus
mkdir -p rules

# Create alert rules file
cd rules
vim alter_op.rules
```

Add alert rules content:

```yaml
groups:
  - name: mongoConnectionAlert
    rules:
    - alert: mongoConnectionAlert
      expr: sum(mongodb_op_counters_total[5m]) by (instance) > 100
      for: 1m
      labels:
        severity: page
      annotations:
        summary: "Instance {{ $labels.instance }} command operations high"
        description: "{{ $labels.instance }} command operations above 100 times (current value: {{$value }})"
```

Add rules file to Prometheus main configuration:

```bash
cd /usr/local/prometheus
cp prometheus.yml prometheus.yml.bak
vim prometheus.yml
```

Add rules file configuration:

```yaml
rule_files:
- /usr/local/prometheus/rules/*.rules
```

Restart Prometheus:

```bash
ps -ef | grep prometheus
kill {pid}
nohup /usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --web.listen-address=:9090 --web.enable-lifecycle &
```

Alert rules and alert status can be viewed through the Alerts and Rules tabs of the Prometheus interface.

## MongoDB on EC2 Backup and Restore

### Recommended Backup and Restore Architecture

![Backup and Restore Architecture Diagram](https://example.com/placeholder-image)

*Note: The original document contained images that have been replaced with placeholder links in the Markdown conversion*

### MongoDB on EC2 Backup Solution

1. EC2 needs to have SSM permissions (AmazonSSMManagedInstanceCore). For non-Amazon Linux AMIs, the SSM agent needs to be installed manually.

2. Create an SSM Document, example content:

```json
{
  "schemaVersion": "2.2",
  "description": "State Manager MongoDB Backup",
  "parameters": {},
  "mainSteps": [
    {
      "action": "aws:runShellScript",
      "name": "configureServer",
      "inputs": {
        "runCommand": [
          "#!/bin/bash",
          "date=$(date +\"%Y%m%d\")",
          "directory=\"/home/ec2-user/mongodb_backup/$date\"",
          "if [ ! -d \"$directory\" ]; then",
          "  mkdir -p \"$directory\"",
          "  echo \"Directory $directory created.\"",
          "else",
          "  echo \"Directory $directory already exists.\"",
          "fi",
          "ips=$(aws ec2 describe-instances --filters \"Name=tag:Database,Values=mongodb-cluster\" --region us-east-1 --query \"Reservations[].Instances[].PrivateIpAddress\" --output text)",
          "formatted_ips=\"\"",
          "for ip in $ips; do",
          "  formatted_ips+=\"$ip,\"",
          "done",
          "formatted_ips=${formatted_ips%,} # Remove the last comma",
          "sudo mongodump --readPreference secondary --host $formatted_ips -o $directory"
        ]
      }
    }
  ]
}
```

3. Create a CloudWatch scheduled trigger rule to execute the backup script on the MongoDB cluster node.

### MongoDB on EC2 Restore Solution

Use mongorestore to restore a specific database or all databases:

```bash
# Restore specific database
mongorestore --host 10.0.0.1 -d test /path/to/backup/test

# Restore all databases
mongorestore --host 10.0.0.1 /path/to/backup/
```

### Common MongoDB Backup and Restore Commands

```bash
# Full database backup
mongodump --readPreference secondary --host 10.0.0.1,10.0.0.2 -o /path/to/backup/

# Backup specific database
mongodump --readPreference secondary --host 10.0.0.1,10.0.0.2 --db test -o /path/to/backup/test

# Restore all databases
mongorestore --host 10.0.0.1 /path/to/backup/

# Restore specific database
mongorestore --host 10.0.0.1 -d test /path/to/backup/test
```

## MongoDB Replica Set Architecture Expansion on EC2

### Replica Set Architecture

MongoDB on EC2 basic configuration:
- MongoDB 5.0
- Three nodes - R6i.2xLarge
- Nodes deployed across three AZs
- Nodes belong to the same Security Group
- Storage – Each node has GP3 2TB storage

### Storage Expansion

Steps to increase storage (example: expanding MongoDB node storage from 2TB to 3TB):

1. Modify storage volume size from 2TB to 3TB in the EC2 console
2. Log in to the instance and expand the file system:

```bash
# Check current file system size
df -hT

# Expand file system
sudo resize2fs /dev/nvme1n1

# Check file system size again
df -hT
```

Note: Throughout the storage expansion process, the MongoDB process can remain running.

### Instance Expansion

Steps to change replica member instance type one by one:

1. Shut down the replica set member:

```javascript
mongo
use admin
db.shutdownServer()
```

2. Stop the EC2 instance, change the instance type (e.g., from R6i.2xlarge to R6i.4xlarge)
3. Start the EC2 instance
4. Log in to the EC2 instance and start the MongoDB process:

```bash
sudo mongod --config /etc/mongod.conf&
```

5. Check replica set status:

```javascript
rs.status()
```

### Replica Set Member Expansion

Steps to expand from 3 members to 5 members:

1. Check oplog size:

```javascript
rs.printReplicationInfo()
```

2. Change oplog size if needed:

```javascript
use local
db.adminCommand({replSetResizeOplog: 1, size: 16000})
```

3. Add two replica set members, initially setting the new nodes to not be eligible for primary election and not able to vote:

```javascript
rs.add({ host: "10.0.0.4:27017", priority: 0, votes: 0 })
rs.add({ host: "10.0.0.5:27017", priority: 0, votes: 0 })
```

4. Wait for the new nodes to synchronize and reach SECONDARY status

5. Enable reading data on new nodes:

```javascript
rs.secondaryOk()
```

6. Reset new node properties to allow them to be elected as primary and to vote:

```javascript
// Modify the 4th node
var cfg = rs.conf()
cfg.members[3].priority = 1
cfg.members[3].votes = 1
rs.reconfig(cfg)

// Modify the 5th node
var cfg = rs.conf()
cfg.members[4].priority = 1
cfg.members[4].votes = 1
rs.reconfig(cfg)
```

7. Verify configuration:

```javascript
rs.config()
