# Install and configure Logstash on AWS
Logstash is an open source, server-side data processing pipeline that ingests data from a multitude of sources simultaneously, transforms it, and then sends it to your favorite “stash.” (Ours is Elasticsearch, naturally.)

Logstash has three main functions
 - **INPUTS**: Ingest Data of All Shapes, Sizes, and Sources
 - **FILTERS**: Parse & Transform Your Data On the Fly
 - **OUTPUTS**: Choose Your Stash, Transport Your Data

 Logstash Features:
 - **PLUG & PLAY**: Accelerated Time to Insight with the Elastic Stack
 - **EXTENSIBILITY**: Create and Configure Your Pipeline, Your Way
 - **DURABILITY & SECURITY**: Trust in a Pipeline Built to Deliver
 - **MONITORING**: Have Full Visibility into Your Deployments
 - **MANAGEMENT & ORCHESTRATION**: Centrally Manage Deployments With a Single UI

![](https://raw.githubusercontent.com/miztiik/elk-stack/master/images/elk.png)
## Install Logstash

### Pre-Requisites
Logstash requires Java 8. Java 9 is not supported. 
```sh
java --version
# Install Java v8 (if it is lesser than v8)
sudo yum -y install java-1.8.0-openjdk
sudo yum -y remove java-1.7.0-openjdk
java -version
echo $JAVA_HOME
# export PATH=$JAVA_HOME/bin:$PATH 
```

We are going to install ES in Amazon Linux 2, you should be able to adapt this procedure for say RHEL/CentOS/Fedora.
Elasticsearch requires at least Java 8.
```sh
# Accept the ElasticSearch PGP Key
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
### Installing from the RPM repository
Create a file called `logstash.repo` in the `/etc/yum.repos.d/` directory. The below repo is for Logstash version 6.x
```sh
echo '[logstash-6.x]
name=Elastic repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
' | sudo tee /etc/yum.repos.d/elasticsearch.repo

# Install from RPM Repo
sudo yum -y install logstash

# For manual installations visit
# https://www.elastic.co/downloads/elasticsearch
```

## Configure Logstash
The configuration above tells Logstash to collect all files with .log extention in `/var/log`, `/var/log/message`s and `/var/log/syslog`.

```sh
echo 'input {
  file {
    path => [ "/var/log/*.log", "/var/log/messages", "/var/log/syslog" ]
    type => "syslog"
  }
}
output {
    elasticsearch {
        hosts => ["ec2-18-197-152-116.eu-central-1.compute.amazonaws.com:9200"]
    }
    #stdout { codec => rubydebug }
}' | sudo tee /etc/logstash/conf.d/logstash-syslog.conf
```
Next, we will create a filter to prevent Elasticsearch to store logs in the message field and simplify the analysis.
```sh
echo 'filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}' | sudo tee /etc/logstash/conf.d/logstash-syslog-filter.conf
```
#### Permissions ERROR
Logstash needs permissions to access the log files, If they aren't provided you will receive errors like this,
```pre
  [2018-07-20T23:53:30,532][WARN ][logstash.inputs.file ] failed to open /var/log/messages: Permission denied - /var/log/messages
```

Add a `logstash` group and provide read permissions 
```sh
chgrp logstash /var/log/*.log /var/log/messages
chmod g+r /var/log/*.log /var/log/messages
```


#### Configure Logstash to run as service
```sh
/usr/share/logstash/bin/system-install /etc/logstash/startup.options sysv
```

Use the `chkconfig` command to configure Logstash to start automatically when the system boots up
```sh
sudo chkconfig --add logstash
```
Logstash can be started and stopped using the `service` command
```sh
# To Start/Stop Logstash 
sudo -i service logstash start
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/ &

# To Stop Logstash 
sudo -i service logstash stop
```

```sh
[root@ip-10-80-4-74 ~]# sudo -i service elasticsearch start
Starting elasticsearch:                                    [  OK  ]
```

#### Configuring Access to ElasticSearch through Public IP
Typically you add your corporate IP/private IP and allow the same in the firewall/security group to allow access to ElasticSearch. In our case, just to give a dirty hack to get going we will open to the internet.
DO THIS ONLY FOR LEARNING PURPOSES - NOT FOR BUSINESS WORKLOADS CONFIGURATIONS
```sh
echo "network.host: 0.0.0.0" >> /etc/elasticsearch/elasticsearch.yml
curl localhost:9200
```
We should see something like this:
```sh

[root@ip-10-80-4-74 elasticsearch]# curl localhost:9200
{
  "name" : "Gr7qXpl",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "XouOh_loQ62c3nHcDxbDOQ",
  "version" : {
    "number" : "6.3.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "eb782d0",
    "build_date" : "2018-06-29T21:59:26.107521Z",
    "build_snapshot" : false,
    "lucene_version" : "7.3.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
It is my strong recommendation that you put your servers in a load balacer. Check out how to setup [ELB here](https://www.youtube.com/watch?v=QyjDktNxdQg)