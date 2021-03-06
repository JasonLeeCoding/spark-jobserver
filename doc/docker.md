<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Docker](#docker)
  - [Configuration](#configuration)
  - [Jars / Passing Arguments to the Start Script](#jars--passing-arguments-to-the-start-script)
  - [Database, Persistence, Logs](#database-persistence-logs)
  - [Marathon](#marathon)
  - [Building docker image from the master branch](#building-docker-image-from-the-master-branch)
  - [Issues](#issues)
    - [I can't access a textfile](#i-cant-access-a-textfile)
    - [Timeouts](#timeouts)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Docker

Spark-jobserver is available as a Docker container!  This might be the easiest way to get started and deploy.

To get started:

    docker run -d -p 8090:8090 sparkjobserver/spark-jobserver:0.7.0.mesos-0.25.0.spark-1.6.2 

This will start job server on port 8090 in a container, with H2 database and Mesos support, and expose that port to the host on which you run the container.

If you would like to debug job server using JMX / VisualVM etc., then also expose port 9999.

## Configuration

By default, the container has an embedded Spark distro and runs using Spark local mode (`local[4]`).

To change the spark master the container runs against, set SPARK_MASTER when you start the container:

    docker run -d -p 8090:8090 -e SPARK_MASTER=mesos://zk://mesos.master:5050 sparkjobserver/spark-jobserver:0.7.0.mesos-0.25.0.spark-1.6.2

You can easily change the amount of memory job server uses with `JOBSERVER_MEMORY`, or replace the entire config job server uses at startup with `JOBSERVER_CONFIG`.

The standard way to replace the config is to derive a custom Docker image from the job server one by overwriting the default config at `app/docker.conf`.  The Dockerfile would look like this:

    from sparkjobserver/spark-jobserver:0.7.0.mesos-0.25.0.spark-1.6.2
    add /path/to/my/jobserver.conf /app/docker.conf

Similarly, to change the logging configuration, inherit from this container and overwrite `/app/log4j-server.properties`.

## Jars / Passing Arguments to the Start Script

Any `spark-submit` arguments can be passed to the tail of the `docker run` command.  A very common use of this is to add custom jars to your Spark job environment.  For example, to add the Datastax Spark-Cassandra Connector to your job:

    docker run -d -p 8090:8090 sparkjobserver/spark-jobserver:0.7.0.mesos-0.25.0.spark-1.6.2 --packages com.datastax.spark:spark-cassandra-connector_2.10:1.6.1

## Database, Persistence, Logs

Docker containers are usually stateless, but it wouldn't be very useful to have the jars and job config reset every time you had to kill and restart a container.

The job server docker image is configured to use H2 database by default and to write the database to a Docker volume at `/database`, which will be persisted between container restarts, and can even be shared amongst multiple job server containers on the same host. Note that in order to persist them to new containers, you need to create a local directory, something like this:

    docker run -d -p 8090:8090 -v /opt/job-server-db:/database sparkjobserver/spark-jobserver:0.7.0.mesos-0.25.0.spark-1.6.2

See the [Docker Volumes Guide](http://docs-stage.docker.com/userguide/dockervolumes/#volume) for more info.

Another option is to configure job server to persist metadata in PostGres, MySQL, or similar database.  To do that, create a new config, pass it into the docker container as above using `JOBSERVER_CONFIG` and the `/config` volume, and point to your shared database, perhaps using `--link` to a PostGres or MySQL container.

Logging goes to stdout, as per standard Docker conventions.  Therefore:

* Use `docker logs -f <containerHash>` to follow logs
* Use `docker logs --tail=100 <containerHash>` to list the last 100 lines
* Use Docker logging drivers to redirect logs to syslog, SumoLogic, etc.

## Marathon

Example Marathon config, thanks to @peterklipfel:

```json
{
  "id": "spark.jobserver",
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "sparkjobserver/spark-jobserver:0.7.0.mesos-0.25.0.spark-1.6.2",
      "network": "BRIDGE",
      "portMappings": [{
        "containerPort": 8090,
        "hostPort": 0,
        "protocol": "tcp"
      }],
      "privileged": false
    }
  },
  "args": [
    "--packages", "com.datastax.spark:spark-cassandra-connector_2.10:1.6.1,com.github.sstone:amqp-client_2.10:1.5,com.rabbitmq:amqp-client:3.2.1, com.typesafe.akka:akka-actor_2.10:2.3.11,com.github.nscala-time:nscala-time_2.10:1.6.0,com.fasterxml.jackson.core:jackson-core:2.2.2, com.fasterxml.jackson.core:jackson-databind:2.2.2,com.fasterxml.jackson.module:jackson-module-scala_2.10:2.2.2,org.scalaj:scalaj-http_2.10:1.1.4,org.elasticsearch:elasticsearch-spark_2.10:2.1.0.Beta3,spark.jobserver:job-server-api:0.6.2,spark.jobserver:job-server-extras:0.6.2"
  ],
  "env": {
    "SPARK_MASTER": "mesos.ourcluster.internal:5050"
  },
  "cpus": 0.5,
  "mem": 100,
  "instances": 1
}
```

## Building docker image from the master branch

If you want to build your docker version based on current master branch:

```
sbt docker
```

## Issues

### I can't access a textfile

You might not be able to access HDFS, or whatever shared file system your data files are on.  Make sure the right ports are open.  Also see https://github.com/spark-jobserver/spark-jobserver/issues/243#issuecomment-168242345.

### Timeouts

You may need to enable host-only networking to get Docker to work in AWS.
