# Third-Party Services

ConductR's method of delivering third-party service support embraces the [Open Container Initiative](https://www.opencontainers.org/) specifications. A vendor simply needs to provide a Docker or OCI image and ConductR can then take care of deploying and scaling it across your cluster.

We've spent a lot of effort ensuring that essential services will work correctly in a ConductR cluster. Below, you'll find a curated list of common applications and the required arguments to load them into your cluster.

* [Elasticsearch](#Elasticsearch)
* [Kafka](#Kafka)
* [MySQL](#MySQL)
* [PostgreSQL](#PostgreSQL)
* [ZooKeeper](#ZooKeeper)

#### Elasticsearch

[Elasticsearch](https://www.elastic.co/products/elasticsearch) is a search and analytics engine. The example below deploys [elastic](https://www.elastic.co/)'s image into your ConductR cluster. For more information on this image and its arguments, refer to [Install Elasticsearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html).


```bash
conduct load docker.elastic.co/elasticsearch/elasticsearch:5.4.0 \
    --start-command '[
        "bin/elasticsearch", 
        "-Enetwork.bind_host=$ELASTICSEARCH_TCP_9200_BIND_IP", 
        "-Ehttp.port=$ELASTICSEARCH_TCP_9200_BIND_PORT", 
        "-Enetwork.publish_host=$BUNDLE_HOST_IP", 
        "-Expack.security.enabled=false", 
        "-Ecluster.name=conductr"
    ]' \
    --volume es-data=/usr/share/elasticsearch/data \
    --endpoint elasticsearch-tcp-9200 \
        --service-name elastic-search \
        --bind-port 0 \
        --bind-protocol http
```

#### Kafka

[Kafka](https://kafka.apache.org/) is a distributed streaming platform that can be used to publish, subscribe, and process streams of data in real-time. The example below deploys [Confluent, Inc](https://www.confluent.io/)'s Kafka image into your ConductR cluster. For more information on this image and its arguments, refer to [Confluent's Documentation](http://docs.confluent.io/current/cp-docker-images/docs/configuration.html#confluent-kafka-cp-kafka).

*Please note that Kafka relies on [ZooKeeper](#ZooKeeper) for coordination. Ensure that it is available in your ConductR cluster via the service name `zookeeper`.*  

```bash
conduct load confluentinc/cp-kafka \
    --endpoint cp-kafka-tcp-9092 \
        --service-name kafka \
        --bind-port 0 \
    --volume kafka-data=/var/lib/kafka/data \
    --volume kafka-secrets=/etc/kafka/secrets \
    --env 'KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://$CP_KAFKA_TCP_9092_BIND_IP:$CP_KAFKA_TCP_9092_BIND_PORT' \
    --env 'KAFKA_ZOOKEEPER_CONNECT=$(curl -s -o /dev/null -w %{redirect_url} $SERVICE_LOCATOR/zookeeper | sed s@^tcp://@@)'
```

#### MySQL

[MySQL](https://www.mysql.com/) is an open-source Relational Database Management System (RDBMS). The example below deploys the MySQL team's image to your ConductR cluster. For more information on this image and its arguments, refer to [MySQL](https://hub.docker.com/_/mysql/) on Docker Hub.
 
 ```bash
conduct load mysql:5.7 \
    --endpoint mysql-tcp-3306 \
        --service-name mysql \
        --bind-port 3306 \
    --volume mysql-data=/var/lib/mysql \
    --env MYSQL_ROOT_PASSWORD=my-secret-pw
 ```

#### PostgreSQL

[PostgreSQL](https://www.postgresql.org/) is an open-source Relational Database Management System (RDBMS). The example below deploys the PostgreSQL community's image into your ConductR cluster. For more information on this image and its arguments, refer to [PostgreSQL](https://hub.docker.com/_/postgres/) on Docker Hub.

```bash
conduct load postgres:9.6.3 \
    --endpoint postgres-tcp-5432 \
        --service-name postgres \
        --bind-port 5432 \
    --volume postgres-data=/var/lib/postgresql/data \
    --env POSTGRES_PASSWORD=mysecretpassword
```

#### ZooKeeper

[ZooKeeper](https://zookeeper.apache.org/) is a distributed coordination service. The example below deploys [Confluent, Inc](https://www.confluent.io/)'s ZooKeeper image into your ConductR cluster. For more information on this image and its arguments, refer to [Confluent's Documentation](http://docs.confluent.io/current/cp-docker-images/docs/quickstart.html#zookeeper).
 
 ```bash
conduct load confluentinc/cp-zookeeper \
    --endpoint cp-zookeeper-tcp-2181 \
        --service-name zookeeper \
        --bind-port 0 \
    --volume zookeeper-data=/var/lib/zookeeper/data \
    --volume zookeeper-secrets=/etc/zookeeper/secrets \
    --volume zookeeper-log=/var/lib/zookeeper/log \
    --env 'ZOOKEEPER_CLIENT_PORT=$CP_ZOOKEEPER_TCP_2181_BIND_PORT'
 ```