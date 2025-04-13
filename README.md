# MongoDB on EC2高可用自建操作手册2025 – AWS 动手实验总结与操作重点截屏

## 目录

1. [模拟生产环境一致的VPC](#1模拟生产环境一致的vpc)
2. [在2个AZ的部署架构图](#2在2个az的部署架构图)
3. [EC2规划配置](#3ec2规划配置)
4. [如架构图，在VPC私有子网开启EC2，然后开启MongoDB](#4如架构图在vpc私有子网开启ec2然后开启mongodb-50)
5. [安装命令](#6安装命令)
6. [性能监控](#7性能监控)
7. [每日自动备份](#8每日自动备份)
8. [完成](#9完成)

## 1、模拟生产环境一致的VPC

![image-20250413162003516](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162004537.png)

*注：原文档中包含图片，转换为Markdown格式时图片未能保留*

## 2、在2个AZ的部署架构图

![image-20250413162057900](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162058523.png)



*注：原文档中包含图片，转换为Markdown格式时图片未能保留*

## 3、EC2规划配置：

原始需求：

> MongoDB自建这个架构是用3台EC2搭建的，3台服务器是跨可用区做高可用确保运行稳定。 但是这种高可用架构，提高性能和磁盘扩容比托管服务要费劲的多，建议您直接按照2年预期容量给配置，避免频繁扩容。
> 
> **我们应该需要按照4核8GB 硬盘100GB SSD的配置**

**AWS建议配置为：
3台（节点）\* m6i.xlarge , 4 vCPU 16 GiB Memory, GP3 60GB系统盘，独立数据盘 200GB**

单台EC2的关键配置如下:

1. 基本信息:
   * 实例名称: MongoDB Primary -m6i.xlarge
   * AMI: Amazon Linux 2 AMI (HVM) - Kernel 5.10
   * 架构: 64-bit (x86)

2. 实例规格:
   * 类型: m6i.xlarge
   * vCPU: 4核
   * 内存: 16 GiB

3. 网络配置:
   * VPC: 私有VPC (10.0.0.0/16)
   * 子网: private-subnet-1a (10.0.0.0/24)
   * 公网IP: 禁用
   * 安全组: MongoDB-SG

4. 存储配置:
   * 根卷: 60 GiB gp3, 3000 IOPS
   * 附加卷: 200 GiB gp3, 3000 IOPS
   * 总存储: 260 GiB

## 4、如架构图，在VPC私有子网开启EC2，然后开启MongoDB 50：

注意：需要使用Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type；

![image-20250413162213948](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162214797.png)

| 创建MongoDB安全组  1. 在VPC控制面板中，点击"安全组" 2. 点击"创建安全组" 3. 填写以下信息：    * **安全组名称**：MongoDB-SG    * **描述**：Security group for MongoDB cluster    * **VPC**：选择MongoDB-VPC 4. 配置入站规则：    * **类型**：SSH，**来源**：您的IP地址/32    * **类型**：Custom TCP，**端口**：27017-27019，**来源**：10.0.0.0/16（VPC CIDR）    * **类型**：All Traffic，**来源**：MongoDB-SG（自引用，允许集群内部通信） 5. 配置出站规则：    * **类型**：All Traffic，**目标**：0.0.0.0/0 6. 点击"创建安全组" |
| --- |

## 6、安装命令

### 1、创建MongoDB仓库配置

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

### 2、安装MongoDB

```bash
sudo yum install -y mongodb-org
```

### 3、编辑MongoDB配置文件

![image-20250413162611805](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162612443.png)

```bash
sudo vim /etc/mongod.conf
```

### 4、启动MongoDB服务

```bash
sudo systemctl start mongod.service
sudo systemctl enable mongod.service
```

### 5、准备数据目录

![image-20250413162510441](/Users/yiming/Library/Application Support/typora-user-images/image-20250413162510441.png)

```bash
lsblk
sudo mkfs.ext4 /dev/nvme1n1
sudo mkdir /mongodata
sudo mount /dev/nvme1n1 /mongodata
df -hT
```

### 6、配置开机自动挂载

![image-20250413162525289](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162525956.png)

```bash
sudo cp /etc/fstab /etc/fstab.orig
sudo blkid
```

编辑/etc/fstab文件：

```bash
sudo vim /etc/fstab
```

增加一行：

```
UUID=ce6fe2c6-81be-46f6-adca-739d934d1b68 /mongodata ext4 defaults,nofail 0 2
```

### 7、创建日志目录并停止MongoDB服务

```bash
sudo mkdir /mongodata/log
sudo systemctl stop mongod.service
```

### 8、更新MongoDB配置指向新数据目录

![image-20250413162639767](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162640423.png)

```bash
sudo vim /etc/mongod.conf
```

配置文件内容：

```yaml
# mongod.conf
# for documentation of all options, see:
# http://docs.mongodb.org/manual/reference/configuration-options/
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /mongodata/log/mongod.log

# Where and how to store data.
storage:
  dbPath: /mongodata
  journal:
    enabled: true
  # engine:
  # wiredTiger:

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0 # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
```

迁移数据并设置权限：

```bash
df -hT
sudo cp -R /var/lib/mongo /mongodata
sudo chmod 777 -R /mongodata
sudo chown mongod:mongod /mongodata
sudo service mongod start
```

![image-20250413162706918](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162707588.png)

### 10、监控MongoDB服务

```bash
sudo systemctl status mongod
sudo tail -f /mongodata/log/mongod.log
ls -l /mongodata
sudo journalctl -u mongod -n 50
```

验证MongoDB连接：

```bash
# 使用mongo shell命令行连接（本地）
mongosh

# 使用mongo shell指定主机和端口连接
mongosh --host 127.0.0.1 --port 27017

# 如果是远程连接，使用完整的URI
mongosh "mongodb://服务器IP:27017"
```

连接成功后，可以执行以下命令验证：

```javascript
// 显示所有数据库
show dbs

// 查看当前状态
db.serverStatus()

// 创建测试数据库
use testdb

// 插入测试数据
db.test.insertOne({name: "test", value: 123})

// 查询数据
db.test.find()
```

![image-20250413162805186](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162806115.png)

### 11、创建MongoDB副本集

获取全部的EC2IP后，分别为：
* MongoDB Primary -m6i.xlarge = 10.0.0.10
* MongoDB Secondary 1 -m6i.xlarge = 10.0.1.11
* MongoDB Secondary 2 -m6i.xlarge = 10.0.1.12

在三个节点上，各自修改MongoDB配置：

```bash
sudo vim /etc/mongod.conf
```

添加以下内容：

```yaml
replication:
  replSetName: "rs0"
```

重启MongoDB服务：

```bash
sudo service mongod restart
sudo systemctl status mongod
```

在Primary节点初始化副本集：

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "10.0.0.10:27017" },
    { _id: 1, host: "10.0.1.11:27017" },
    { _id: 2, host: "10.0.1.12:27017" }
  ]
})
```

检查副本集状态：

```javascript
rs.status()
rs.conf()
```

创建管理员用户：

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "admin",
  roles: [{role: "root", db: "admin"}]
})
db.auth("admin", "admin")
```

## 7、性能监控

### 1）创建 CloudWatch dashboard

![image-20250413162848629](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162849367.png)

### 2）使用MongoDB工具监控

使用mongostat查看实时操作统计：

```bash
mongostat --host <ip_address> --port 27017 --username admin --password admin --authenticationDatabase admin
```

使用mongotop查看集合级别的读写活动：

```bash
mongotop --host <ip_address> --port 27017 --username admin --password admin --authenticationDatabase admin
```

示例命令：

```bash
mongosh mongostat --host 10.0.0.10 --port 27017 --username admin --password admin --authenticationDatabase admin
mongosh mongostat --host 10.0.1.11 --port 27017 --username admin --password admin --authenticationDatabase admin
mongosh mongostat --host 10.0.1.12 --port 27017 --username admin --password admin --authenticationDatabase admin
```

## 8、每日自动备份

### 1、创建一个S3私有桶，用于放备份

```
s3://mongodb-backup-bucket-[account-id]
```

### 2、新开一台低配置的EC2专门用于备份，EBS容量与MongoDB存储容量一致

### 3、登录EC2之后配置访问S3桶的访问权限

```bash
aws configure
# 输入ak/sk, cn-northwest-1
aws s3 ls # 测试
```

### 4、对三个节点的MongoDB EC2打tag：tag:Database,Values=mongodb-cluster

![image-20250413162924366](https://raw.githubusercontent.com/liangyimingcom/storage/master/PicGo/20250413162925020.png)

### 5、在备份EC2上创建备份脚本

```bash
sudo vim /home/ec2-user/mongodb_backup.sh
```

脚本内容：

```bash
#!/bin/bash
# 设置日期格式
date=$(date +"%Y%m%d")
directory="/home/ec2-user/mongo_data_bak/$date"
s3_bucket="s3://mongodb-backup-bucket-[account-id]"

# 清理旧备份
echo "Cleaning up all previous backups..."
rm -rf /home/ec2-user/mongo_data_bak/*
echo "All previous backups removed"

# 创建新备份目录
mkdir -p "$directory"
echo "Directory $directory created."

# 获取MongoDB节点IP
ips=$(aws ec2 describe-instances --filters "Name=tag:Database,Values=mongodb-cluster" --region cn-northwest-1 --query "Reservations[].Instances[].PrivateIpAddress" --output text)
formatted_ips=""
for ip in $ips; do
  formatted_ips+="$ip,"
done
formatted_ips=${formatted_ips%,}

# 执行MongoDB备份
echo "Starting MongoDB backup..."
sudo mongodump --readPreference secondary --host $formatted_ips -o $directory

# 检查备份是否成功
if [ $? -eq 0 ]; then
  echo "MongoDB backup completed successfully"
  
  # 上传到S3
  echo "Uploading backup to S3..."
  aws s3 cp $directory $s3_bucket/$date --recursive
  
  # 检查S3上传是否成功
  if [ $? -eq 0 ]; then
    echo "S3 upload completed successfully"
    echo "Local backup is kept at $directory"
  else
    echo "ERROR: S3 upload failed, but local backup is kept at $directory"
  fi
else
  echo "ERROR: MongoDB backup failed"
fi
```

设置权限并创建必要目录：

```bash
sudo chmod +x /home/ec2-user/mongodb_backup.sh
sudo mkdir -p /home/ec2-user/mongo_data_bak
sudo chown ec2-user:ec2-user /home/ec2-user/mongo_data_bak
sudo mkdir -p /var/log/mongodb_backup
sudo chown ec2-user:ec2-user /var/log/mongodb_backup
```

设置定时任务：

```bash
crontab -e
# 添加以下内容（每天凌晨2点执行）
0 2 * * * /home/ec2-user/mongodb_backup.sh >> /var/log/mongodb_backup/backup_$(date +\%Y\%m\%d).log 2>&1
```

### 6、测试备份脚本

```bash
cd /home/ec2-user/
bash /home/ec2-user/mongodb_backup.sh
```

检查结果：

```bash
ls -l /home/ec2-user/mongo_data_bak
tail -f /var/log/mongodb_backup/backup_$(date +%Y%m%d).log
```

## 9、完成
```bash
按照要求修改了MongoDB自建操作手册文档，移除了所有敏感信息：
删除了企业名称
替换了原始IP地址为通用IP格式（10.0.0.x/10.0.1.x）
替换了实际VPC标识符和CIDR范围为通用格式
修改了S3桶名称为通用格式
更新了EC2标签名称为通用名称
处理后的文档保留了所有有价值的技术内容和操作步骤，但移除了任何可能识别特定客户或环境的信息。文档现在可以作为通用的MongoDB on EC2高可用集群搭建指南使用，不含任何敏感信息。
```
