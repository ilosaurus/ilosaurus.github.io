---
title: "EFK Stack - Parsing Log with Fluentd"
date: 2020-04-13T16:03:00+07:00
draft: false
categories: ["Logging"]
tags: ["EFK", "Fluentd", "Elasticsearch", "Parsing"]
description: "Learn how to parse simple, JSON, XML, and multiline logs using Fluentd within an EFK (Elasticsearch, Fluentd, Kibana) stack to make centralized logs more human-readable and insightful."
---

In this session I will share about something that is quite important in the production environment, namely logging. Logs help us in analyzing and identifying the conditions of an application.  

The challenge is when we try to identify logs in each application on a different server or location that effort or spend a lot of time. The solution is by applying a Log Server so that the log in each application can be centralized.  

<!-- more -->

The next challenge is how the centralized log is Human readable or can be identified more easily by humans to provide stunning insight or summary. One of them is by parsing raw logs into smaller pieces.  

In this session I will share how to :
* Parsing simple raw log
* Parsing json raw log
* Parsing raw log xml
* Parsing multiline log  

Then the stack is used :
* CentOS 8 as Operating System.
* Elasticsearch as log storage.  
* Fluentd as a log aggregation, parser, and agent.  
* Kibana as a visualize tool integrated with Elasticsearch.  

To do a parsing log on fluentd, I use the parser plugin provided by fluentd, including regexp, apache2, apache_error, nginx, syslog, csv, tsv, ltsv, json, msgpack, multiline, none and third party parser grok.

## Simple parse raw log using fluentd regexp parser.  

Lets start running python apps to generate dummy logs.  
```
import datetime, time, random, string

def randomString(stringLength=10):
    letters = string.ascii_uppercase
    return ''.join(random.choice(letters) for i in range(stringLength))

while True:
    time.sleep(20)
    print(datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")+" - SIMPLEAPPS ORDER-"+randomString(14)+" "+str(random.randrange(10,99,1)))

```

```
mkdir /var/log/simple
python3 simple-python.py >> /var/log/simple/python-simple.log
```  
And we can tail this log file

```
[root@efk-fluentd simple]# pwd
/var/log/simple
[root@efk-fluentd simple]# tree
.
├── python-simple.log
├── python-simple.py
0 directories, 4 files
[root@efk-fluentd simple]# python3 python-simple.py >> /var/log/simple/python-simple.log &
[1] 8362
[root@efk-fluentd simple]# tail -f /var/log/simple/python-simple.log
2020-04-13 20:39:48 - SIMPLEAPPS ORDER-NNZWQAIAGQCVJM 15
2020-04-13 20:40:18 - SIMPLEAPPS ORDER-YTHWQVIKHZUSFE 46
2020-04-13 20:40:48 - SIMPLEAPPS ORDER-JWTDMRKLONUDNT 54
2020-04-13 20:41:18 - SIMPLEAPPS ORDER-JKDSUVPFLUTPZW 67
```

Edit file `/etc/td-agent/td-agent.conf` , add line `@include config.d/*.conf` for directives in separate configuration files.  

```
vi /etc/td-agent/td-agent.conf
mkdir /etc/td-agent/config.d
```

Create config file for parsing string log from file `/var/log/simple/python-simple.log.`  

```
vi /etc/td-agent/config.d/simple-parsing.conf
systemctl restart td-agent
systemctl status td-agent
```

```
<source>
  @type tail
  @id python_simple
  <parse>
    @type regexp
    expression (?<logtime>[^ ]* [^ ]*) - (?<apps-name>[^ ]*) (?<invoice>[^ ]*) (?<count>[^ ]*) 
    time_key logtime
    time_format %Y-%m-%d %H:%M:%S
    types count:integer
  </parse>
  path /var/log/simple/python-simple.log
  tag simple_parsing
</source>

<match simple_parsing>
   @type stdout
</match>
```

```
#2020-04-13 21:05:49 - SIMPLEAPPS ORDER-EUJPYYAYEOJQKW 92
expression (?<logtime>[^ ]* [^ ]*) - (?<apps-name>[^ ]*) (?<invoice>[^ ]*) (?<count>[^ ]*)
```

In config file simple-parsing.log using plugin parser regexp to parse raw log from /var/log/simple/python-simple.log. And in match section using output plugin stdout so logs from file /var/log/simple/python-simple.log will be forwarded to /var/log/td-agent/td-agent.log as stdout logs.
Now, just tail file /var/log/td-agent/td-agent. log to check the parsing result.  

```
2020-04-13 21:05:49.000000000 +0700 simple_parsing: {"apps-name":"SIMPLEAPPS","invoice":"ORDER-EUJPYYAYEOJQKW","count":92}
2020-04-13 21:06:19.000000000 +0700 simple_parsing: {"apps-name":"SIMPLEAPPS","invoice":"ORDER-KRCSOOQVYGTKMW","count":68}
2020-04-13 21:07:19.000000000 +0700 simple_parsing: {"apps-name":"SIMPLEAPPS","invoice":"ORDER-EXKBWKNDKLXGZS","count":61}
```
  
```
time:
2020-04-13 21:05:49.000000000 +0700 simple_parsing:
record: 
{
   "apps-name":"SIMPLEAPPS",
   "invoice":"ORDER-EUJPYYAYEOJQ",
   "count":"92"
}
```

## Simple parse raw json log using fluentd json parser.  
In this section, we will parsing raw json log using fluentd json parser and output to stdout. Ok lets start with create generator log using simple python script.  

```
import datetime, sys, time, random, string

def randomString(stringLength=10):
    letters = string.ascii_uppercase
    return ''.join(random.choice(letters) for i in range(stringLength))

while True:
    time.sleep(30)
    print('{"order_time":"'+datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")+'","order_id":"ORDER-'+randomString(14)+'","order_amount":"'+str(random.randrange(10,99,1)*1000)+'"}', flush=True)

```