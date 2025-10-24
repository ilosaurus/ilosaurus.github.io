---
title: "EFK Stack - Deploying EFK Stack"
date: 2020-04-12T16:03:28+07:00
draft: false
categories: ["Logging"]
tags: ["EFK", "Elasticsearch", "Kibana", "Deployment"]
description: "A step-by-step guide to deploying the EFK (Elasticsearch, Fluentd, Kibana) stack, starting with the installation and configuration of Elasticsearch and Kibana on CentOS 8."
---
![EFK Arch](/images/efk-arch.png "EFK Arch")

Elasticsearch provides real-time search and analytics for all types of data. Whether you have structured or unstructured text, numerical data, or geospatial data, Elasticsearch can efficiently store and index it in a way that supports fast searches. You can go far beyond simple data retrieval and aggregate information to discover trends and patterns in your data. And as your data and query volume grows, the distributed nature of Elasticsearch enables your deployment to grow seamlessly right along with it. 

<!-- more -->

Elasticsearch is commonly deployed alongside Kibana, a powerful data visualization  and dashboard for Elasticsearch. Kibana allows you to explore your Elasticsearch log data through a web interface, and build dashboards and queries to quickly answer questions and gain insight into your Kubernetes applications.

## EFK Stack

* Elasticsearch Version 7.6  
* Fluentd Version 2.6
* Kibana Version 7.6  

## Install Elasticsearch

Install Elasticsearch on node `efk-elastic-kibana`

### Update & Upgrade packages

```bash
yum -y update
yum -y upgrade
yum -y install epel-release
reboot
```

### Set Timezone

```bash
timedatectl set-timezone Asia/Jakarta
```

### Install & Start Chronyd

```bash
yum -y install chrony
systemctl enable --now chronyd
systemctl status chronyd
```

### Install OpenJDK

```bash
yum install java-1.8.0-openjdk
```

### Add repository Elasticsearch

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
yum clean all
yum makecache
yum repolist
```

### Install Elasticsearch

```bash
yum -y install elasticsearch
```

### Config Elasticsearch, edit file `/etc/elasticsearch/elasticsearch.yml`

```bash
cp /etc/elasticsearch/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml.original
```

```bash
vi /etc/elasticsearch/elasticsearch.yml
```
```yaml
node.name: efk-elastic-kibana
discovery.zen.minimum_master_nodes: 1
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["efk-elastic-kibana"]
```

### Enable & Start Elasticsearch Service

```bash
systemctl daemon-reload
systemctl enable --now elasticsearch
systemctl status elasticsearch
```

### Verify Elasticsearch

```bash
netstat -tulpn | grep 9200
curl http://127.0.0.1:9200/_cluster/health?pretty
curl http://127.0.0.1:9200/_cat/nodes?v
```

Example Output 

```bash
[root@efk-elastic-kibana ~]# curl http://10.199.199.123:9200/_cluster/health?pretty
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 6,
  "active_shards" : 6,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 3,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 66.66666666666666
}
[root@efk-elastic-kibana ~]# curl http://10.199.199.123:9200/_cat/indices?v
health status index                    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_task_manager_1   c2A6LOb9QQySpcvfY2r_RQ   1   0          2            1     29.8kb         29.8kb
green  open   .apm-agent-configuration EM_3qi6yQOmP9unMRtFJ2g   1   0          0            0       283b           283b
green  open   .kibana_1                bsbGNLfDTdqjALyJwBCkIQ   1   0          9            1     44.7kb         44.7kb
[root@efk-elastic-kibana ~]# 
```

## Install Kibana

Install Kibana on node `efk-elastic-kibana`  

### Install kibana

```bash
yum -y install kibana
```

### Config Kibana, edit file `/etc/kibana/kibana.yml`

```bash
cp /etc/kibana/kibana.yml /etc/kibana/kibana.yml.original
```

```bash
vi /etc/kibana/kibana.yml
```
```yaml
elasticsearch.hosts: ["http://localhost:9200"]
```

### Enable & Start Kibana

```bash
systemctl enable --now kibana
systemctl status kibana
```

### Install nginx as reverse proxy

```bash
yum -y install nginx apache2-utils  
```

### Generate credential for basic http auth

```bash
# kibana password
echo "kibana:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.kibana
```

### Config nginx, edit file `/etc/nginx/nginx.conf`

```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.original
```

```bash
vi /etc/nginx/nginx.conf
```

```conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;
events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    include /etc/nginx/conf.d/*.conf;

    server {
        listen 80;
        server_name _;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/htpasswd.kibana;

        location / {
           proxy_pass http://localhost:5601;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
       }
    } 
}
```

### Verify config nginx

```bash
nginx -t
```

### Enable & Restart nginx

```bash
systemctl enable nginx 
systemctl restart nginx
systemctl status nginx
```

### Access kibana via browser

![Kibana Dashboard](/images/kibana.png "Kibana Dashboard")

---  

  


#### `Next chapter :`
#### `Introduction, Install & parsing simple log on fluentd`
#### `Parsing Apps Logs from Docker Container`
#### `EFK Stack Integration with Kubernetes CLuster`