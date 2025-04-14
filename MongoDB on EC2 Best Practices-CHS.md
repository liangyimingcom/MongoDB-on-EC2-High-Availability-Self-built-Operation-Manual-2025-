
# MongoDB on EC2 最佳实践

## 目录

- [MongoDB on EC2 最佳实践](#mongodb-on-ec2-最佳实践)
  - [目录](#目录)
  - [MongoDB 5.0副本集架构on EC2安装](#mongodb-50副本集架构on-ec2安装)
    - [副本集架构](#副本集架构)
    - [安装主节点](#安装主节点)
    - [安装副本节点](#安装副本节点)
    - [创建MongoDB 5.0副本集架构](#创建mongodb-50副本集架构)
  - [MongoDB on EC2数据库参数模版](#mongodb-on-ec2数据库参数模版)
    - [数据库参数设置](#数据库参数设置)
      - [net.maxIncomingConnections](#netmaxincomingconnections)
      - [net.bindIp](#netbindip)
      - [net.ipv6](#netipv6)
      - [storage.wiredTiger.engineConfig.cacheSizeGB](#storagewiredtigerengineconfigcachesizegb)
      - [storage.directoryPerDB](#storagedirectoryperdb)
      - [replication.oplogSizeMB](#replicationoplogsizemb)
      - [systemLog.logAppend](#systemloglogappend)
    - [mongod.conf](#mongodconf)
  - [使用prometheus与grafana监控MongoDb](#使用prometheus与grafana监控mongodb)
    - [前提](#前提)
    - [部署说明](#部署说明)
    - [部署指南](#部署指南)
      - [1.监控机安装Docker](#1监控机安装docker)
      - [2.监控机上安装mongodb-exporter](#2监控机上安装mongodb-exporter)
      - [3.安装mongodb宿主机 go环境](#3安装mongodb宿主机-go环境)
      - [4.mongodb宿主机安装node\_exporter](#4mongodb宿主机安装node_exporter)
      - [5.安装prometheus收集mongodb-exporter接口的数据](#5安装prometheus收集mongodb-exporter接口的数据)
      - [6.安装与使用Granfara](#6安装与使用granfara)
      - [7.查看监控数据](#7查看监控数据)
      - [8.告警配置](#8告警配置)
  - [Mongodb on EC2备份还原](#mongodb-on-ec2备份还原)
    - [建议备份还原架构](#建议备份还原架构)
    - [MongoDB on EC2备份方案](#mongodb-on-ec2备份方案)
    - [MongoDB on EC2还原方案](#mongodb-on-ec2还原方案)
    - [Mongodb备份和还原常用命令](#mongodb备份和还原常用命令)
  - [MongoDB 副本集架构on EC2扩展](#mongodb-副本集架构on-ec2扩展)
    - [副本集架构](#副本集架构-1)
    - [存储扩展](#存储扩展)
    - [实例扩展](#实例扩展)
    - [副本集成员扩展](#副本集成员扩展)

## MongoDB 5.0副本集架构on EC2安装

安装MongoDB 5.0 在三个EC2节点（R6i.2xLarge）三个节点：

### 副本集架构

![副本集架构图](https://example.com/placeholder-image)

*注：原文档中包含图片，转换为Markdown格式时图片链接已替换为占位符*

每个节点配置如下：

- Security group: mongodb-sg（安全组内各节点网络互通）
- Storage: GP3 2T (存放MongoDB 数据文件)
- 网络：三个节点跨三个AZ部署

### 安装主节点

1. 安装mongodb 5.0 软件

```bash
sudo vi /etc/yum.repos.d/mongodb-org-5.0.repo
```

添加以下内容：

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

2. 查看修改配置文件

```bash
sudo vim /etc/mongod.conf
```

修改绑定IP：

```
bindIp: 127.0.0.1  改为 bindIp: 0.0.0.0
```

（注意冒号与ip之间需要一个空格）

3. 启动MongoDB

```bash
sudo systemctl start mongod.service
```

或者采用另外一种方式启动：

```bash
sudo mongod --config /etc/mongod.conf&
```

停止mongodb：

```bash
sudo systemctl stop mongod.service
```

4. 设置开机启动

```bash
sudo systemctl enable mongod.service
```

5. 修改MongoDB数据文件到2T GP3存储

查看raw device:

```bash
lsblk
```

创建MongoDB文件系统：

```bash
sudo mkfs.ext4 /dev/nvme1n1
sudo mkdir /mongodata
sudo mount /dev/nvme1n1 /mongodata
df -hT
```

6. 使MongoDB data 所在的存储卷在EC2主机重启后自动挂载：

```bash
sudo cp /etc/fstab /etc/fstab.orig
sudo blkid
```

输出示例：
```
/dev/nvme1n1: UUID="03a545aa-3662-4137-ac44-f436098d26bc" TYPE="ext4"
```

编辑fstab文件：

```bash
sudo vim /etc/fstab
```

增加一行：

```
UUID=03a545aa-3662-4137-ac44-f436098d26bc /mongodata ext4 defaults,nofail 0 2
```

7. 创建MongoDB日志目录：

```bash
sudo mkdir /mongodata/log
sudo systemctl stop mongod.service
```

8. 修改Mongod Data的路径

```bash
sudo vim /etc/mongod.conf
```

修改配置文件内容，数据文件目录指向：/mongodata; 日志文件指向:/mongodata/log

9. 拷贝原有mongodb 数据文件到新数据文件目录：

检查/mongodata是否为单独的mount point:

```bash
df -hT
```

拷贝数据并设置权限：

```bash
sudo cp -R /var/lib/mongo /mongodata
sudo chmod 777 -R /mongodata
sudo chown mongod:mongod /mongodata
sudo service mongod start
```

### 安装副本节点

依据主节点安装步骤安装（选用同一个security group和存储2TB GP3）

### 创建MongoDB 5.0副本集架构

1. 修改三个MongoDB 5.0节点，在/etc/mongod.conf中增加以下内容（保持缩进格式）：

```yaml
replication:
  replSetName: "rs0"
```

2. 重启MongoDB服务：

```bash
sudo service mongod restart
```

3. 在三个节点上的任意节点上运行Mongo执行(添加ip地址均为各节点私有ip地址)：

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

4. 查看副本集状态：

```javascript
rs.status()
rs.conf()
```

5. 设置副本集可读（在两个副本集secondary节点设置）：

```javascript
rs.secondaryOk()
```

6. 连接到主节点，创建admin用户：

```javascript
use admin
db.createUser({
  user:"admin",
  pwd:"password",
  roles:[{role:"root", db:"admin"}]
})
```

## MongoDB on EC2数据库参数模版

### 数据库参数设置

#### net.maxIncomingConnections

默认值为65536，建议根据实例内存大小对连接数参数进行设置

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

假设内存大小为 X GB
- X < 128：log2(X / 2) * 3000
- X >= 128：log2(X / 64) * 21000

参考文档：[Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-net.maxIncomingConnections)

#### net.bindIp

默认值为 localhost 建议设置为 ::,0.0.0.0

参考文档：[Configuration Options](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.bindIp)

#### net.ipv6

默认值为false 建议设置为 true

参考文档：[Configuration Options](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-net.ipv6)

#### storage.wiredTiger.engineConfig.cacheSizeGB

按默认值设置即可： max(50% of (RAM - 1 GB), 256 MB)

参考文档：[Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-storage.wiredTiger.engineConfig.cacheSizeGB)

#### storage.directoryPerDB

默认值为false 建议设置为 true

参考文档：[Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-storage.directoryPerDB)

#### replication.oplogSizeMB

默认值为5%磁盘空间，建议在需要更大范围的数据变更记录时，改为10%的磁盘空间

参考文档：[Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-replication.oplogSizeMB)

#### systemLog.logAppend

默认值为false 建议设置为 true

参考文档：[Configuration Options](https://www.mongodb.com/docs/v5.0/reference/configuration-options/#mongodb-setting-systemLog.logAppend)

### mongod.conf

在2c4g，挂载磁盘为80G的实例部署时的参考配置如下

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

## 使用prometheus与grafana监控MongoDb

本章节主要描述如何使用prometheus，结合mongodb-exporter对mongoDB进行监控数据采集，告警配置和告警查看；以及使用grafana进行监控metrics展示。以上三个组件都是开源工具，并广泛使用于各领域监控。其中，mongodb-exporter选择了percona开源的版本，专门针对mongoDB进行了体系化的监控数据采集，并提供http接口给prometheus进行metrics入库和监控数据管理。

### 前提

- MongoDB 5.0实例已部署
- 一台EC2服务器已部署

### 部署说明

- EC2用于部署 mongodb-exporter, prometheus与grafana
- MongoDB 5.0实例的宿主机部署node-exporter

![监控架构图](https://example.com/placeholder-image)

*注：原文档中包含图片，转换为Markdown格式时图片链接已替换为占位符*

### 部署指南

#### 1.监控机安装Docker

```bash
# 安装最新的 Docker Community Edition 程序包
sudo yum install -y docker

# 启动 Docker 服务
sudo service docker start

# 将 ec2-user 添加到 docker 组，以便无需使用 sudo 执行 Docker 命令
sudo usermod -a -G docker ec2-user
```

#### 2.监控机上安装mongodb-exporter

准备镜像percona/mongodb_exporter:

```bash
# 拉取percona镜像
docker pull percona/mongodb_exporter:0.38

# 查看镜像是否拉取成功
docker images
```

启动容器:

```bash
# 注意监听端口9104:9216，前面的9104是服务器监听端口，后面的9216是容器内端口
docker run -d -p 9104:9216 --restart=always --name=mongodb-exporter \
  percona/mongodb_exporter:0.38 --compatible-mode --discovering-mode \
  --collect-all --mongodb.uri=mongodb://admin:password@10.0.0.1:27017/admin?replicaSet=rs0
```

验证容器是否启动:

```bash
docker ps
netstat -ntlp | grep 9104

# 检查数据采集
curl http://127.0.0.1:9104/metrics
```

#### 3.安装mongodb宿主机 go环境

```bash
# 下载Go
wget https://go.dev/dl/go1.20.1.linux-amd64.tar.gz

# 解压到/usr/local目录
sudo tar -C /usr/local -xzf go1.20.1.linux-amd64.tar.gz

# 设置环境变量
sudo vi ~/.bash_profile
# 添加以下行
export PATH=$PATH:/usr/local/go/bin

# 重载配置文件
source ~/.bash_profile

# 验证Go版本
go version
```

#### 4.mongodb宿主机安装node_exporter

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
tar -xzf node_exporter-1.1.2.linux-amd64.tar.gz
sudo cp node_exporter-1.1.2.linux-amd64/node_exporter /usr/local/bin/
node_exporter &

# 验证node_exporter是否启动
netstat -lntp | grep 9100
```

#### 5.安装prometheus收集mongodb-exporter接口的数据

```bash
# 安装Go环境
yum install go
go version

# 安装Prometheus
cd ~/mongo_monitor
wget https://github.com/prometheus/prometheus/releases/download/v2.11.0-rc.0/prometheus-2.11.0-rc.0.linux-amd64.tar.gz
sudo tar xvf prometheus-2.11.0-rc.0.linux-amd64.tar.gz -C /usr/local/
sudo ln -sv /usr/local/prometheus-2.11.0-rc.0.linux-amd64/ /usr/local/prometheus

cd /usr/local/prometheus
```

修改配置文件prometheus.yml:

```yaml
scrape_configs:
  # 监控MongoDB
  - job_name: 'mongodb-monitor'
    static_configs:
    - targets: ['127.0.0.1:9104']
```

启动Prometheus服务:

```bash
nohup /usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --web.listen-address=:9090 --web.enable-lifecycle &
ps -ef | grep prome
```

访问Prometheus界面:
http://your-prometheus-server:9090/graph

#### 6.安装与使用Granfara

```bash
# 安装Grafana
cd /home/ec2-user/mongo_monitor
wget https://dl.grafana.com/oss/release/grafana-6.2.5-1.x86_64.rpm
sudo yum localinstall grafana-6.2.5-1.x86_64.rpm

# 启动Grafana服务
sudo service grafana-server start
```

访问Grafana前端页面:
http://your-grafana-server:3000
默认用户名/密码: admin/admin

在Data Sources添加Prometheus数据源，并创建Dashboard。
推荐模板ID:
- node-exporter: 10795
- mongodb-exporter-0.38: 7353

#### 7.查看监控数据

通过Grafana界面查看MongoDB监控数据。

#### 8.告警配置

配置Prometheus告警规则：

```bash
# 创建告警规则文件夹
cd /usr/local/prometheus
mkdir -p rules

# 创建告警规则文件
cd rules
vim alter_op.rules
```

添加告警规则内容:

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

将规则文件加入Prometheus主配置:

```bash
cd /usr/local/prometheus
cp prometheus.yml prometheus.yml.bak
vim prometheus.yml
```

添加规则文件配置:

```yaml
rule_files:
- /usr/local/prometheus/rules/*.rules
```

重启Prometheus:

```bash
ps -ef | grep prometheus
kill {pid}
nohup /usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --web.listen-address=:9090 --web.enable-lifecycle &
```

查看告警规则和告警状态可通过Prometheus界面的Alerts和Rules选项卡。

## Mongodb on EC2备份还原

### 建议备份还原架构

![备份还原架构图](https://example.com/placeholder-image)

*注：原文档中包含图片，转换为Markdown格式时图片链接已替换为占位符*

### MongoDB on EC2备份方案

1. EC2需要有SSM权限(AmazonSSMManagedInstanceCore)，非Amazon Linux的AMI需要自行安装SSM代理。

2. 创建SSM Document，内容示例:

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

3. 使用CloudWatch创建定时触发规则，在MongoDB集群节点执行备份脚本。

### MongoDB on EC2还原方案

使用mongorestore对特定库进行还原或全库还原:

```bash
# 还原特定数据库
mongorestore --host 10.0.0.1 -d test /path/to/backup/test

# 全库还原
mongorestore --host 10.0.0.1 /path/to/backup/
```

### Mongodb备份和还原常用命令

```bash
# 全库备份
mongodump --readPreference secondary --host 10.0.0.1,10.0.0.2 -o /path/to/backup/

# 备份特定数据库
mongodump --readPreference secondary --host 10.0.0.1,10.0.0.2 --db test -o /path/to/backup/test

# 全库还原
mongorestore --host 10.0.0.1 /path/to/backup/

# 还原特定数据库
mongorestore --host 10.0.0.1 -d test /path/to/backup/test
```

## MongoDB 副本集架构on EC2扩展

### 副本集架构

MongoDB on EC2基本配置:
- MongoDB 5.0
- 三个节点 - R6i.2xLarge
- 节点跨三个AZ部署
- 节点同属一个Security Group
- 存储 – 每个节点的存储为GP3 2TB

### 存储扩展

增加存储的步骤示例(将MongoDB节点的2TB存储扩展至3TB):

1. 在EC2控制台修改存储卷大小从2TB至3TB
2. 登录实例扩展文件系统:

```bash
# 查看当前文件系统大小
df -hT

# 扩展文件系统
sudo resize2fs /dev/nvme1n1

# 再次查看文件系统大小
df -hT
```

注意：整个存储扩展过程中，MongoDB进程可以保持运行状态。

### 实例扩展

逐一更改副本成员实例类型:

1. 关闭副本集成员:

```javascript
mongo
use admin
db.shutdownServer()
```

2. 停止EC2实例，更改实例类型(例如从R6i.2xlarge升级到R6i.4xlarge)
3. 启动EC2实例
4. 登录EC2实例并启动MongoDB进程:

```bash
sudo mongod --config /etc/mongod.conf&
```

5. 查询副本集状态:

```javascript
rs.status()
```

### 副本集成员扩展

从3个成员扩展至5个成员的步骤:

1. 检查oplog大小:

```javascript
rs.printReplicationInfo()
```

2. 如需更改oplog大小:

```javascript
use local
db.adminCommand({replSetResizeOplog: 1, size: 16000})
```

3. 增加两个副本集成员，先设置新节点不能选主且不能投票:

```javascript
rs.add({ host: "10.0.0.4:27017", priority: 0, votes: 0 })
rs.add({ host: "10.0.0.5:27017", priority: 0, votes: 0 })
```

4. 等待新节点同步完成，状态变为SECONDARY

5. 在新节点上启用读取数据:

```javascript
rs.secondaryOk()
```

6. 重置新节点属性，让它们可以选主和投票:

```javascript
// 修改第4个节点
var cfg = rs.conf()
cfg.members[3].priority = 1
cfg.members[3].votes = 1
rs.reconfig(cfg)

// 修改第5个节点
var cfg = rs.conf()
cfg.members[4].priority = 1
cfg.members[4].votes = 1
rs.reconfig(cfg)
```

7. 验证配置:

```javascript
rs.config()
