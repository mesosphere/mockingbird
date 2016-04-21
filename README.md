# Tweeter

Tweeter is a sample service that demonstrates how easy it is to run a Twitter-like service on DCOS.

Capabilities:

* Stores tweets in Cassandra
* Streams tweets to Kafka as they come in
* Real time tweet analytics with Spark and Zeppelin


## Install and Configure Prerequisites on the Cluster

You'll need a DCOS cluster with one public node and at least five private nodes, DCOS CLI, and DCOS package CLIs.

Install package CLIs:

```base
$ dcos package install cassandra --cli
$ dcos package install kafka --cli
```

Look up the public agent IP in AWS. You need the IP of the EC2 host, not the ELB. Use this to replace `<agent_public_ip>` further down.

Look up the public master IP in AWS. You need the IP of the EC2 host, not the ELB. Use this to replace `<master_public_ip>` further down.


## Demo steps

Install packages for DCOS UI:
* kafka
* cassandra
* zeppelin
* marathon-lb

Wait until the Kafka and Cassandra services are healthly.

## Lookup Kafka broker addresses

Lookup the connection setting for Kafka:
    
```base
$ dcos kafka connection
```
    
The output should look similar:
```json
{
    "address": [
        "10.0.3.62:9557",
        "10.0.3.59:9757",
        "10.0.3.58:9504"
    ],
    "dns": [
        "broker-0.kafka.mesos:9557",
        "broker-1.kafka.mesos:9757",
        "broker-2.kafka.mesos:9504"
    ],
    "zookeeper": "master.mesos:2181/kafka"
}
```

## Edit the Tweeter Service Config

Edit the `KAFKA_BROKERS` environment variable in `tweeter.json` to match your environment. For example:

```bash
"KAFKA_BROKERS": "broker-0.kafka.mesos:9557"
```

## Run the Tweeter Service

Launch three instances of Tweeter on Marathon using the config file in this repo:

```bash
$ dcos marathon app add tweeter.json
```

The service talks to Cassandra via `node-0.cassandra.mesos:9042`, and Kafka via `broker-0.kafka.mesos:9557` in this example.

Traffic is routed to the service via marathon-lb. Navigate to `http://<agent_public_ip>:10000` to see the Tweeter UI and post a Tweet.


## Post a lot of Tweets

Post a lot of Shakespeare tweets from a file:

```bash
$ bin/tweet shakespeare-tweets.json http://<agent_public_ip>:10000
```

This will post more than 100k tweets one by one, so you'll see them coming in steadily when you refresh the page.


## Streaming Analytics

Next, we'll do real-time analytics on the stream of tweets coming in from Kafka.

Navigate to Zeppelin at `http://<master_public_ip>/service/zeppelin/`, click `Import note` and import `tweeter-analytics.json`. Zeppelin is preconfigured to execute Spark jobs on the DCOS cluster, so there is no further configuration or setup required.

Run the *Load Dependencies* step to load the required libraries into Zeppelin. Next, run the *Spark Streaming* step, which reads the tweet stream from Zookeeper, and puts them into a temporary table that can be queried using SparkSQL. Next, run the *Top Tweeters* SQL query, which counts the number of tweets per user, using the table created in the previous step. The table updates continuously as new tweets come in, so re-running the query will produce a different result every time.


NOTE: if /service/zeppelin is showing as Disconnected (and hence can’t load the notebook), you can instead redirect Zeppelin out the ELB using Marathon-LB. To do this, add the following labels to the zeppelin service and restart:


`HAPROXY_0_VHOST = [elb hostname]`

`HAPROXY_GROUP = external`

You can get the ELB hostname from the CCM “Public Server” link.  Once Zeppelin restarts, this should allow you to use that link to reach the Zeppelin GUI in “connected” mode.



## Developing Tweeter

You'll need Ruby and a couple of libraries on your local machine to hack on this service. If you just want to run the demo, you don't need this.

### Homebrew on Mac OS X

Using Homebrew, install `rbenv`, a Ruby version manager:

```bash
$ brew update
$ brew install rbenv
```

Run this command and follow the instructions to setup your environment:

```bash
$ rbenv init
```

To install the required Ruby version for Tweeter, run from inside this repo:

```bash
$ rbenv install
```

Then install the Ruby package manager and Tweeter's dependencies. From this repo run:

```bash
$ gem install bundler
$ bundle install
```
