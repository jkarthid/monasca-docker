Apache Kafka on Docker
======================

This holds a build definition and supporting files for building a
[Docker] image to run [Kafka] in containers. It is based on 'ches/kafka' in 
Docker hub, but modified to handle 8.1.1 kafka version and 2.9.2 scala versions.
Added missing log4j jar to kafka/libs, and modified the server.properties.template
for some of our settings.
 
  - Configuration is parameterized, enabling a Kafka cluster to be run from
    multiple container instances.
  - Kafka data and logs can be handled outside the container(s) using volumes.
  - JMX is exposed, for Kafka and JVM metrics visibility.


Usage Quick Start
-----------------

Here is a minimal-configuration example running the Kafka broker service, then
using the container as a client to run the basic producer and consumer example
from [the Kafka Quick Start]:

```
$ docker build -t monasca/kafka:V1 .
$ docker run -d --name zookeeper jplock/zookeeper:3.4.6
$ docker run -d --name kafka --link zookeeper:zookeeper monasca/kafka:V1

$ export ZK_IP=$(docker inspect --format '{{ .NetworkSettings.IPAddress }}' zookeeper)
$ export KAFKA_IP=$(docker inspect --format '{{ .NetworkSettings.IPAddress }}' kafka)

$ docker run --rm monasca/kafka:V1 \
>   kafka-topics.sh --create --topic test --replication-factor 1 --partitions 1 --zookeeper $ZK_IP:2181
Created topic "test".

In this terminal:
$ docker run --rm --interactive monasca/kafka:V1 \
>   kafka-console-producer.sh --topic test --broker-list $KAFKA_IP:9092
<type some messages followed by newline>

In a separate (second) terminal:
$ export ZK_IP=$(docker inspect --format '{{ .NetworkSettings.IPAddress }}' zookeeper)
$ docker run --rm monasca/kafka:V1 \
>   kafka-console-consumer.sh --topic test --from-beginning --zookeeper $ZK_IP:2181
```

### Volumes

The container exposes two volumes that you may wish to bind-mount, or process
elsewhere with `--volumes-from`:

- `/data`: Path where Kafka's data is stored (`log.dirs` in Kafka configuration)
- `/logs`: Path where Kafka's logs (`INFO` level) will be written, via log4j

### Ports and Linking

The container publishes two ports:

- `9092`: Kafka's standard broker communication
- `7203`: JMX publishing, for e.g. jconsole or VisualVM connection

Kafka requires Apache ZooKeeper. You can satisfy the dependency by simply
linking another container that exposes ZooKeeper on its standard port of 2181,
as shown in the above example, **ensuring** that you link using an alias of
`zookeeper`.

Alternatively, you may configure a specific address for Kafka to find ZK. See
the Configuration section below.

### A more complex local development setup

This example shows more configuration options and assumes that you wish to run a
development environment with Kafka ports mapped directly to localhost, for
instance if you're writing a producer or consumer and want to avoid rebuilding a
container for it to run in as you iterate. This requires that localhost is your
Docker host, i.e. your workstation runs Linux. If you're using something like
boot2docker, substitute the value of `boot2docker ip` below.

```bash
$ mkdir -p kafka-ex/{data,logs} && cd kafka-ex
$ docker run -d --name zookeeper --publish 2181:2181 jplock/zookeeper:3.4.6
$ docker run -d \
    --hostname localhost \
    --name kafka \
    --volume ./data:/data --volume ./logs:/logs \
    --publish 9092:9092 --publish 7203:7203 \
    --env KAFKA_ADVERTISED_HOST_NAME=127.0.0.1 --env ZOOKEEPER_IP=127.0.0.1 \
    ches/kafka
```

Configuration
-------------

Some parameters of Kafka configuration can be set through environment variables
when running the container (`docker run -e VAR=value`). These are shown here
with their default values, if any:

- `KAFKA_BROKER_ID=0`

  Maps to Kafka's `broker.id` setting. Must be a unique integer for each broker
  in a cluster.
- `KAFKA_PORT=9092`

  Maps to Kafka's `port` setting. The port that the broker service listens on.
  You will need to explicitly publish a new port from container instances if you
  change this.
- `KAFKA_ADVERTISED_HOST_NAME=<container's IP within docker0's subnet>`

  Maps to Kafka's `advertised.host.name` setting. Kafka brokers gossip the list
  of brokers in the cluster to relieve producers from depending on a ZooKeeper
  library. This setting should reflect the address at which producers can reach
  the broker on the network, i.e. if you build a cluster consisting of multiple
  physical Docker hosts, you will need to set this to the hostname of the Docker
  *host's* interface where you forward the container `KAFKA_PORT`.
- `KAFKA_ADVERTISED_PORT=9092`

  As above, for the port part of the advertised address. Maps to Kafka's
  `advertised.port` setting. If you run multiple broker containers on a single
  Docker host and need them to be accessible externally, this should be set to
  the port that you forward to on the Docker host.
- `JAVA_RMI_SERVER_HOSTNAME=$KAFKA_ADVERTISED_HOST_NAME`

  Maps to the `java.rmi.server.hostname` JVM property, which is used to bind the
  interface that will accept remote JMX connections. Like
  `KAFKA_ADVERTISED_HOST_NAME`, it may be necessary to set this to a reachable
  address of *the Docker host* if you wish to connect a JMX client from outside
  of Docker.
- `ZOOKEEPER_IP=<taken from linked "zookeeper" container, if available>`

  **Required** if no container is linked with the alias "zookeeper" and
  publishing port 2181, or not using `ZOOKEEPER_CONNECTION_STRING` instead. Used
  in constructing Kafka's `zookeeper.connect` setting.
- `ZOOKEEPER_PORT=2181`

  Used in constructing Kafka's `zookeeper.connect` setting.
- `ZOOKEEPER_CONNECTION_STRING=<comma separated string of host:port pairs>`

  Set a string with host:port pairs for connecting to a ZooKeeper Cluster. This
  setting overrides `ZOOKEEPER_IP` and `ZOOKEEPER_PORT`.
- `ZOOKEEPER_CHROOT`, ex: `/v0_8_1`

  ZooKeeper root path used in constructing Kafka's `zookeeper.connect` setting.
  This is blank by default, which means Kafka will use the ZK `/`. You should
  set this if the ZK instance/cluster is shared by other services, or to
  accommodate Kafka upgrades that change schema. Starting in Kafka 0.8.2, it
  will create the path in ZK automatically; with earlier versions, you must
  ensure it is created before starting brokers.

JMX
---

Remote JMX access can be a bit of a pain to set up. The start script for this
container tries to make it as painless as possible, but it's important to
understand that if you want to connect a client like VisualVM from outside other
Docker containers (e.g. directly from your host OS in development), then you'll
need to configure RMI to be addressed *as the Docker host IP or hostname*. If
you have set `KAFKA_ADVERTISED_HOST_NAME`, that value will be used and is
probably what you want. If not (you're only using other containers to talk to
Kafka brokers) or you need to override it for some reason, then you can instead
set `JAVA_RMI_SERVER_HOSTNAME`.

For example in practice, if your Docker host is VirtualBox run by Docker
Machine, a `run` command like this should allow you to connect VisualVM from
your host OS to `$(docker-machine ip docker-vm):7203`:

    $ docker run -d --name kafka -p 7203:7203 \
        --link zookeeper:zookeeper \
        --env JAVA_RMI_SERVER_HOSTNAME=$(docker-machine ip docker-vm) \
        ches/kafka

Note that it is fussy about port as well---it may not work if the same port
number is not used within the container and on the host (any advice for
workarounds is welcome).

Finally, please note that by default remote JMX has authentication and SSL
turned off (these settings are taken from Kafka's own default start scripts). If
you expose the JMX hostname/port from the Docker host in a production
environment, you should make make certain that access is locked down
appropriately with firewall rules or similar. A more advisable setup in a Docker
setting would be to run a metrics collector in another container, and link it to
the Kafka container(s).

If you need finer-grained configuration, you can totally control the relevant
Java system properties by setting `KAFKA_JMX_OPTS` yourself---see `start.sh`.

[Docker]: http://www.docker.io
[Kafka]: http://kafka.apache.org
[on the Docker registry]: https://registry.hub.docker.com/u/ches/kafka/
[relateiq/kafka]: https://github.com/relateiq/docker-kafka
[the Kafka Quick Start]: http://kafka.apache.org/documentation.html#quickstart
