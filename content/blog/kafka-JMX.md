---
title: "Kafka JMX"
date: 2018-05-04T14:30:33+01:00
tags: ["Kafka", "Java", "Monitoring", "JMX"] 
---

## How to: JMX and Kafka

There are many many ways to read from [JMX](https://en.wikipedia.org/wiki/Java_Management_Extensions) in general.

We will show here 2 ways:

1. using a GUI-based JMX-client: [jconsole](https://docs.oracle.com/javase/8/docs/technotes/guides/management/jconsole.html)
1. using a cli JMX-client: [jmxterm](https://cwiki.apache.org/confluence/display/KAFKA/jmxterm+quickstart)

## Running kafka

You can run kafka the way you want.

In order to get JMX, you just need to set the `JMX_PORT` environment variable to an open port (usually `9999`) before you start kafka.
 
If you use the [`iyp-uk.kafka` ansible role](https://galaxy.ansible.com/iyp-uk/kafka/), it's as simple as setting it in your role's variables like so:

```yaml
kafka_env:
  JMX_PORT: 9999
```

Or for a more complete typical example:

```yaml
- name: Installing Kakfa
  hosts: kafka
  become: yes
  roles:
    - role: kafka
      kafka_hosts: "{{groups['kafka']}}"
      kafka_zookeeper_hosts: "{{groups['zookeeper']}}"
      kafka_env:
        JMX_PORT: 9999
      kafka_config:
        server:
          num_partitions: 3
          offsets_topic_replication_factor: 3
          transaction_state_log_replication_factor: 3
          log_retention_hours: -1
```

> From now on, we will assume we have 3 brokers: `kaf1`, `kaf2` and `kaf3`, with JMX exported on port `9999`.

## JMX-client

### `jconsole`

With `jconsole`, you don't need to have anything sitting on the brokers, as you can do it remotely (as long as you can access the port remotely).

It comes with Java, so you don't need to install it specifically.

So, let's get started:

```console
$ jconsole
```

Select "remote" and supply your broker JMX endpoint such as `kaf1:9999`.

By default, authentication is **not** enabled, so you don't need to specify any username / password. 

You then get something like:

{{< figure src="/img/jconsole.png"  >}}

The part we're particularly interested in lives in the `MBeans` tab.

Example with under-replicated partitions - They should always be equal to 0.

The corresponding MBean is `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions`

So under the `MBean` tab, expand:

- `kafka.server`
- `ReplicaManager`
- `UnderReplicatedPartitions`
- `Attributes`
- `Value`

Check it's effectively 0.

{{< figure src="/img/jconsole-UnderReplicatedPartitions.png"  >}}

### `jmxterm`

As its name suggests, it's a terminal-based utility. Go and [download](http://wiki.cyclopsgroup.org/jmxterm/download.html) it. 

Let's start it, and connect to our remote broker:

```console
$ java -jar jmxterm-1.0.0-uber.jar --url kaf1:9999
Welcome to JMX terminal. Type "help" for available commands.
```

the help command shows what commands are available:

```console
$>help
#following commands are available to use:
about    - Display about page
bean     - Display or set current selected MBean. 
beans    - List available beans under a domain or all domains
bye      - Terminate console and exit
close    - Close current JMX connection
domain   - Display or set current selected domain. 
domains  - List all available domain names
exit     - Terminate console and exit
get      - Get value of MBean attribute(s)
help     - Display available commands or usage of a command
info     - Display detail information about an MBean
jvms     - List all running local JVM processes
open     - Open JMX session or display current connection
option   - Set options for command session
quit     - Terminate console and exit
run      - Invoke an MBean operation
set      - Set value of an MBean attribute
subscribe - Subscribe to the notifications of a bean
unsubscribe - Unsubscribe the notifications of an earlier subscribed bean
watch    - Watch the value of one MBean attribute constantly
```

An interesting command is `beans`, which shows all available beans.

Narrowing down to a specific domain might be more useful though:

```console
$>beans -d kafka.server
#domain = kafka.server:
kafka.server:broker-id=1,type=controller-channel-metrics
kafka.server:broker-id=2,fetcher-id=0,type=replica-fetcher-metrics
kafka.server:broker-id=2,type=controller-channel-metrics
kafka.server:broker-id=3,fetcher-id=0,type=replica-fetcher-metrics
kafka.server:broker-id=3,type=controller-channel-metrics
kafka.server:brokerHost=kaf2,brokerPort=9092,clientId=ReplicaFetcherThread-0-2,name=BytesPerSec,type=FetcherStats
kafka.server:brokerHost=kaf2,brokerPort=9092,clientId=ReplicaFetcherThread-0-2,name=RequestsPerSec,type=FetcherStats
kafka.server:brokerHost=kaf3,brokerPort=9092,clientId=ReplicaFetcherThread-0-3,name=BytesPerSec,type=FetcherStats
kafka.server:brokerHost=kaf3,brokerPort=9092,clientId=ReplicaFetcherThread-0-3,name=RequestsPerSec,type=FetcherStats
kafka.server:clientId=Replica,name=MaxLag,type=ReplicaFetcherManager
... long list ...
```

Let's now see the same MBean as previously (`kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions`):


First, get info about that bean:
```console
$>info -b kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
#mbean = kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
#class name = com.yammer.metrics.reporting.JmxReporter$Gauge
# attributes
  %0   - Value (java.lang.Object, r)
# operations
  %0   - javax.management.ObjectName objectName()
#there's no notifications
```

It has just one attribute, `Value`, which is the one we're interested in.

The `info` commands being:

```console
$>help info
[USAGE]
  info <OPTIONS>
[DESCRIPTION]
  Display detail information about an MBean
[OPTIONS]
  -b --bean     <value>    Name of MBean
  -d --domain   <value>    Domain for bean
  -h --help                Display usage
  -o --op       <value>    Show a single operation with more details (including parameters information)
  -e --detail              Show description
  -t --type     <value>    Types(a|o|u) to display, for example aon for all attributes, operations and notifications
[NOTE]
  If -b option is not specified, current selected MBean is applied
```

So what's the value of this `Value` attribute? Let's see:

```console
$>get -s -b kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions Value
#mbean = kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions:
0
```

All good, it's still `0`.


The `get` command being:
```console
$>help get
[USAGE]
  get <OPTIONS> <ARGS>
[DESCRIPTION]
  Get value of MBean attribute(s)
[OPTIONS]
  -b --bean     <value>    MBean name where the attribute is. Optional if bean has been set
  -l --delimite <value>    Sets an optional delimiter to be printed after the value
  -d --domain   <value>    Domain of bean, optional
  -h --help                Display usage
  -i --info                Show detail information of each attribute
  -q --quots               Quotation marks around value
  -s --simple              Print simple expression of value without full expression
  -n --singleLi            Prints result without a newline - default is false
[ARGS]
  <attr>... Name of attributes to select
[NOTE]
  * stands for all attributes. eg. get Attribute1 Attribute2 or get *
```

