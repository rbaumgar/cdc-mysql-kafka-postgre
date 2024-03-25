# Debezium: Change Data Capture with Kafka

![](images/jaeger04.png)

*By Robert Baumgartner, Red Hat Austria, March 2024 (OpenShift 4.14)*

In this blog I will guide you on how to use Debezium - Change Data Capture (CDC) together with Red Hat AMQ Streams / Kafka. Data are streamed via Kafka from a MySQL database into a PostgreSQL database.

This document is based on online training [Change data capture with Debizium](https://developers.redhat.com/courses/camel-k/data-capture-debezium)
.

## Installed Operators
The *Red Hat AMQ Streams Operator* is pre-installed in my cluster. Because we don't have to install the Operators in this scenario, `admin` permissions are not required to complete the steps in this document.
In your actual deployment, to make an Operator available for all projects in a cluster, you must be logged in `cluster-admin` permission before you install the Operator.

## Required Tools
`jq` is required to format json data.

## Creating a Namespace
Create a namespace (project) with the name `debeziu` for the AMQ Streams Kafka Cluster Operator:
```shell
export PROJECT=debezium
oc new-project $PROJECT
Already on project "debezium" on server "https://api.example.com:6443".

You can add applications ...
```

## Creating a Kafka Cluster
Create a Kafka cluster named `my-cluster` that has one ZooKeeper node and one Kafka broker node. To simplify the deployment, the YAML file that we'll use to create the cluster specifies the use of ephemeral storage.

Create the Kafka cluster by applying the following command:
```shell
oc apply -f crd/kafka-cluster.yaml
kafka.kafka.strimzi.io/my-cluster created
```
## Checking the status of the Kafka cluster
Verify that the ZooKeeper and Kafka pods are deployed and running in the cluster.

Enter the following command to check the status of the pods:
```shell
oc get pods -w
NAME                                          READY   STATUS              RESTARTS   AGE
my-cluster-zookeeper-0                        1/1     Running             0          26s
my-cluster-kafka-0                            0/1     ContainerCreating   0          0s
my-cluster-kafka-0                            0/1     Running             0          1s
my-cluster-kafka-0                            1/1     Running             0          20s
my-cluster-entity-operator-86b499f8f4-wzvm5   0/2     ContainerCreating   0          1s
my-cluster-entity-operator-86b499f8f4-wzvm5   0/2     Running             0          2s
my-cluster-entity-operator-86b499f8f4-wzvm5   1/2     Running             0          21s
my-cluster-entity-operator-86b499f8f4-wzvm5   2/2     Running             0          21s
```
After a few minutes, the status of the pods for ZooKeeper, Kafka, and the Entity Operator changed to running. 

Notice that the Cluster Operator starts the ZooKeeper clusters, as well as the broker nodes and the Entity Operator.

Enter Ctrl+C to stop the pod monitoring.

The ZooKeeper and Kafka clusters are created in Kubernetes as StatefulSets.

## Verifying that the broker is running
Enter the following command to send a message to the broker that you just deployed on the topic `test`:

```shell
echo "Hello world" | oc exec -i -c kafka my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
[2024-02-23 12:16:20,489] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 1 : {test=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient)
```
The command does not return any output unless it fails. If you see warning messages like above you can ignore them.
The warning is generated when the producer requests metadata for the topic, because the producer wants to write to a topic and broker partition leader that does not exist yet.

Enter the following command to retrieve a message from the broker:

```shell
oc exec -c kafka my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning --max-messages 1
Hello world
Processed a total of 1 messages
```

The broker processes the message that you sent in the previous command, and it returns the `Hello world` string.

You have successfully deployed the Kafka broker service and made it available to clients to produce and consume messages.

## Deploying Kafka Connect with Debezium

After you set up a Kafka cluster, deploy Kafka Connect in a custom container image for Debezium. The Kafka Connect service provides a framework for managing Debezium connectors.

You can create a custom container image by downloading the Debezium MySQL connector archive from the Red Hat Integration download site and extracting it to create the directory structure for the connector plugin.

After you obtain the connector plugin, you can create and publish a custom Linux container image by running the docker build or podman build commands with a custom Dockerfile.

For detailed information about deploying Kafka Connect with Debezium, see the Debezium documentation.

To save some time, we have already created the required content in the `kafka-connect.yaml` CRD. The CRD will automatically create the image for you. The operator creates a buildconfig a starts an image build automatically.

To build and deploy the Kafka Connect with the custom image, enter the following command:
```shell
oc create is debezium-streams-connect
imagestream.image.openshift.io/debezium-streams-connect created
oc apply -f crd/kafka-connect.yaml
kafkaconnect.kafka.strimzi.io/debezium created
```
The Kafka Connect pod `debezium-connect` is deployed.

Check the pod status:
```shell
oc get pods -w -l app.kubernetes.io/name=kafka-connect
NAME                 READY   STATUS              RESTARTS   AGE
debezium-connect-0   0/1     ContainerCreating   0          0s
debezium-connect-0   1/1     Running             0          19s
```
After a couple of minutes, the pod status changes to Running. When the READY column shows 1/1, you are ready to continue.

Enter Ctrl+C to stop the pod monitoring.

## Verify that Kafka Connect is running with Debezium

After the Connect pod is running, you can verify that the Debezium connectors are available. Because AMQ Streams lets you manage most components of the Kafka ecosystem as Kubernetes custom resources, you can obtain information about Kafka Connect from the status of the KafkaConnect resource.

List the connector plugins that are available on the Kafka Connect node:
```shell
oc get kafkaconnect/debezium -o json | jq .status.connectorPlugins
[
  {
    "class": "io.debezium.connector.jdbc.JdbcSinkConnector",
    "type": "sink",
    "version": "2.3.7.Final-redhat-00001"
  },
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "2.3.7.Final-redhat-00001"
  },
  {
    "class": "io.debezium.connector.postgresql.PostgresConnector",
    "type": "source",
    "version": "2.3.7.Final-redhat-00001"
  },
...
]
```

You will see the list of available connectors (including MySQL, Postgres, and JdbcSink) and versions.
The Debezium connector is now available for use on the Connect node.

## Create MySQL database

Generate two random passwords with `openssl` command and a secret for the MySQL database.
```shell
ROOTPWD=`openssl rand -base64 12`
USERPWD=`openssl rand -base64 12`
oc create secret generic mysql-secret \
                         --from-literal=user=mysqluser \
                         --from-literal=password=$USERPWD \
                         --from-literal=root_password=$ROOTPWD
```

Create a simple MySQL database and add the secret with the following commands:
```shell
oc new-app -l app=mysql --name=mysql quay.io/debezium/example-mysql:latest 
--> Found container image 864478b (5 hours old) from quay.io for "quay.io/debezium/example-mysql:latest"

    * An image stream tag will be created as "mysql:latest" that will track this image

--> Creating resources with label app=mysql ...
    imagestream.image.openshift.io "mysql" created
    deployment.apps "mysql" created
    service "mysql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/mysql' 
    Run 'oc status' to view your app.

oc set env deployment/mysql --from secret/mysql-secret --prefix=MYSQL_    
```

Wait until the database is up and running.

```
oc get pod -l app=mysql -w
NAME                    READY   STATUS    RESTARTS   AGE
mysql-67d5bb6cb-7kn2b   1/1     Running   0          89m
```

## Grant permissions to the database user

Grant the required permissions for Debezium to the previously created database user `mysqluser` by the user `root`.

```shell
oc exec -i deploy/mysql -- bash -c 'mysql -t -u root -p$MYSQL_ROOT_PASSWORD -e "GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO mysqluser" mysql'
mysql: [Warning] Using a password on the command line interface can be insecure.
```

The warning can be ignored.

## Streaming data from a database

The final step is to create a link between Debezium and the MySQL source database.

As we mentioned in the previous step, we use AMQ Streams custom resources to interact with the Kafka Connect.

### Create a KafkaConnector resource

Create a KafkaConnector custom resource that will register the MySQL source database to the Debezium MySQL connector.

The KafkaConnector will use a ServiceAccount(SA) `debezium-connect`. This SA needs access to the MySql and Postgres secrets. They should only be referenced from the KafkaConnector.
To get access we create a role and a rolebinding for the SA.

```shell
cat connector.role.yaml | oc apply -f -
cat connector.rolebinding | oc apply -f -
```

Important the rolebinding contains the value of the current project in the `namespace`.

Use the following registration resource, which is included in the scenario environment: `mysql-connector.yaml`.

Register the database source:

```shell
oc apply -f crd/mysql-connector.yaml
kafkaconnector.kafka.strimzi.io/mysql-connector created

# Check the Kafka Connect log file to verify that the registration succeeded and Debezium started:
oc logs -f deploy/debezium-connect
Connection to the database is confirmed in the output as Connected to mysql.default.svc:3306:
...
Feb 23, 2024 1:00:05 PM com.github.shyiko.mysql.binlog.BinaryLogClient tryUpgradeToSSL
INFO: SSL enabled
Feb 23, 2024 1:00:05 PM com.github.shyiko.mysql.binlog.BinaryLogClient connect
INFO: Connected to mysql:3306 at mysql-bin.000003/441 (sid:184054, cid:12)
2024-02-23 13:00:05,630 WARN [mysql-connector|task-0] [Producer clientId=connector-producer-mysql-connector-0] Error while fetching metadata with correlation id 10 : {mysql.inventory.addresses=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient) [kafka-producer-network-thread | connector-producer-mysql-connector-0]
```

Important is the info message `Connected to mysql:3306`.

Warnings with `LEADER_NOT_AVAILABLE`can be ignored.

Enter Ctrl+C to stop the process.

### Inspect KafkaTopics
Now that the database is connected, as changes are committed to the database, Debezium emits change event messages to Kafka topics.

List the topics that Debezium has created:
```shell
oc get kafkatopics
NAME                                                                           CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
consumer-offsets---84e7a678d08f4bd226872e5cdd4eb527fadc1c6a                    my-cluster   50           1                    True
debezium-cluster-configs                                                       my-cluster   1            1                    True
debezium-cluster-offsets                                                       my-cluster   25           1                    True
debezium-cluster-status                                                        my-cluster   5            1                    True
mysql                                                                          my-cluster   1            1                    True
mysql.inventory.addresses                                                      my-cluster   1            1                    True
mysql.inventory.customers                                                      my-cluster   1            1                    True
mysql.inventory.geom                                                           my-cluster   1            1                    True
mysql.inventory.orders                                                         my-cluster   1            1                    True
mysql.inventory.products                                                       my-cluster   1            1                    True
mysql.inventory.products-on-hand---992ac432537e38a711e6ebf3                    my-cluster   1            1                    True
mysql.schema-changes.inventory                                                 my-cluster   1            1                    True
strimzi-store-topic---effb8e3e057afce1ecf67c3f5d8e4e3ff177fc55                 my-cluster   1            1                    True
strimzi-topic-operator-kstreams-topic-store-changelog---b75e702040b99be8a926   my-cluster   1            1                    True
test                                                                           my-cluster   1            1                    True
```

Remember that AMQ Streams Operators also manage the topics of the Apache Kafka ecosystem.

In the list `mysql.inventory.` are all tables available which are found in the source inventory database. 

### Verify that data is emitted from MySQL to Kafka
Check the content of the `customers` table in MySQL source database.

Display the source table:
```shell
oc exec -i deploy/mysql -- bash -c 'mysql -t -u $MYSQL_USER -p$MYSQL_PASSWORD -e "SELECT * from customers" inventory'
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne       | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
mysql: [Warning] Using a password on the command line interface can be insecure.
```

The topic mysql.inventory.customers on the Kafka broker contains messages that represent the data change events from this table in Debezium change event format

Read the messages from the Kafka topic for the customers table:

```shell
oc exec -c kafka my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mysql.inventory.customers --from-beginning --max-messages 4 | jq
Processed a total of 4 messages
...
{
  "schema": {
    "type": "struct",
    "fields": [
      {
        "type": "struct",
        "fields": [
          {
            "type": "int32",
            "optional": false,
            "field": "id"
          },
          {
            "type": "string",
            "optional": false,
            "field": "first_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "last_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "email"
          }
        ],
        "optional": true,
        "name": "mysql.inventory.customers.Value",
        "field": "before"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "int32",
            "optional": false,
            "field": "id"
          },
          {
            "type": "string",
            "optional": false,
            "field": "first_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "last_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "email"
          }
        ],
        "optional": true,
        "name": "mysql.inventory.customers.Value",
        "field": "after"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "version"
          },
          {
            "type": "string",
            "optional": false,
            "field": "connector"
          },
          {
            "type": "string",
            "optional": false,
            "field": "name"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "ts_ms"
          },
          {
            "type": "string",
            "optional": true,
            "name": "io.debezium.data.Enum",
            "version": 1,
            "parameters": {
              "allowed": "true,last,false,incremental"
            },
            "default": "false",
            "field": "snapshot"
          },
          {
            "type": "string",
            "optional": false,
            "field": "db"
          },
          {
            "type": "string",
            "optional": true,
            "field": "sequence"
          },
          {
            "type": "string",
            "optional": true,
            "field": "table"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "server_id"
          },
          {
            "type": "string",
            "optional": true,
            "field": "gtid"
          },
          {
            "type": "string",
            "optional": false,
            "field": "file"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "pos"
          },
          {
            "type": "int32",
            "optional": false,
            "field": "row"
          },
          {
            "type": "int64",
            "optional": true,
            "field": "thread"
          },
          {
            "type": "string",
            "optional": true,
            "field": "query"
          }
        ],
        "optional": false,
        "name": "io.debezium.connector.mysql.Source",
        "field": "source"
      },
      {
        "type": "string",
        "optional": false,
        "field": "op"
      },
      {
        "type": "int64",
        "optional": true,
        "field": "ts_ms"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "id"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "total_order"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "data_collection_order"
          }
        ],
        "optional": true,
        "name": "event.block",
        "version": 1,
        "field": "transaction"
      }
    ],
    "optional": false,
    "name": "mysql.inventory.customers.Envelope",
    "version": 1
  },
  "payload": {
    "before": null,
    "after": {
      "id": 1004,
      "first_name": "Anne",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "source": {
      "version": "2.3.7.Final-redhat-00001",
      "connector": "mysql",
      "name": "mysql",
      "ts_ms": 1708693203000,
      "snapshot": "last_in_data_collection",
      "db": "inventory",
      "sequence": null,
      "table": "customers",
      "server_id": 0,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 441,
      "row": 0,
      "thread": null,
      "query": null
    },
    "op": "r",
    "ts_ms": 1708693203330,
    "transaction": null
  }
}
```

The output shown is formatted to improve readability.

Under `Payload` are the data for one record.

Add a new customer record to the table:
```shell
oc exec -i deploy/mysql -- bash -c 'mysql -t -u $MYSQL_USER -p$MYSQL_PASSWORD -e "INSERT INTO customers VALUES(default,\"John\",\"Doe\",\"john.doe@example.org\")" inventory'
```

Update an existing record:
```shell
oc exec -i deploy/mysql -- bash -c 'mysql -t -u $MYSQL_USER -p$MYSQL_PASSWORD -e "UPDATE customers SET first_name=\"Anne Marie\" WHERE id=1004;" inventory'
```

After adding and updating the record to the database, Debezium emits a new message to the associated topic.

View the messages in the topic again and show only the payload after the change:
```shell
oc exec -c kafka my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mysql.inventory.customers --from-beginning --max-messages 6 | jq .payload.after
{
  "id": 1001,
  "first_name": "Sally",
  "last_name": "Thomas",
  "email": "sally.thomas@acme.com"
}
{
  "id": 1002,
  "first_name": "George",
  "last_name": "Bailey",
  "email": "gbailey@foobar.com"
}
{
  "id": 1003,
  "first_name": "Edward",
  "last_name": "Walker",
  "email": "ed@walker.com"
}
{
  "id": 1004,
  "first_name": "Anne",
  "last_name": "Kretchmar",
  "email": "annek@noanswer.org"
}
{
  "id": 1005,
  "first_name": "John",
  "last_name": "Doe",
  "email": "john.doe@example.org"
}
{
  "id": 1004,
  "first_name": "Anne Marie",
  "last_name": "Kretchmar",
  "email": "annek@noanswer.org"
}

Processed a total of 6 messages
```
The command returns output that includes the newly added record for the user John Doe and the updated record for Kretchmar.


You have completed a basic deployment of Debezium to stream changes from a database. Now we add, change, or remove data from MySQL and see how the changes are reflected on the Kafka broker.

In the next step, we will create another database and create a copy of the table `customers`.
The content of this table will be synchronized by Debezium - Change Data Capture.

## Create Postgres database
Generate a random password and create a simple Postgres database with the following command:
```shell
POSTGRESPWD=`openssl rand -base64 12`
oc process -n openshift postgresql-persistent \
              POSTGRESQL_USER=postgresuser \
              POSTGRESQL_PASSWORD=$POSTGRESPWD \
              POSTGRESQL_DATABASE=postgres \
              DATABASE_SERVICE_NAME=postgres | oc apply -f -
secret/postgres created
service/postgres created
Warning: apps.openshift.io/v1 DeploymentConfig is deprecated in v4.14+, unavailable in v4.10000+
deploymentconfig.apps.openshift.io/postgres created
```

Wait until the Postgres database is up and running.
```bash
oc get pod -l name=postgres -w
NAME              READY   STATUS    RESTARTS   AGE
postgres-1-r76dg  1/1     Running   8s         2m2s
```

## Add Readiness Check for Postgres Database

    readinessProbe:
      exec:
        command:
        - psql
        - -U
        - postgres
        - -c
        - 'select 1'## Create data JDBC sink      

oc set probe dc/postgres --readiness \
                         -- 'psql -U $POSTGRESQL_USER -c select 1' \
                         --initial-delay-seconds=10 \
                         --timeout-seconds=5 \
                         --success-threshold=1 \
                         --failure-threshold=3

## Create a JDBC Connector resource
Create a JDBC Connector resource that will register the Postgres target database to the Kafka topic.

The JDBC Connector will use secret of the Postgres database.

```shell
oc apply -f jdbc-connector.yaml
```


## Check Postgres database

```shell
oc exec -it $(oc get pods -o custom-columns=NAME:.metadata.name --no-headers -l deploymentconfig=postgres) -- bash -c 'psql -U $POSTGRESQL_USER -d $POSTGRESQL_DATABASE -c \\dt'
                     List of relations
 Schema |           Name            | Type  |    Owner     
--------+---------------------------+-------+--------------
 public | mysql_inventory_customers | table | postgresuser
(1 row)

oc exec -it $(oc get pods -o custom-columns=NAME:.metadata.name --no-headers -l deploymentconfig=postgres) -- bash -c 'psql -U $POSTGRESQL_USER -d $POSTGRESQL_DATABASE -c "select * from mysql_inventory_customers"'
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1003 | Edward     | Walker    | ed@walker.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
(4 rows)
```

With the first command, we see that only one table exists in the Postgres database.
With the second command, we get the same records as in the source database.

## Sync the next table
Now we have a working Change Data Capture system running between a MySQL and a Postgres database.
We can now easily add another table which should be syncronised between the two databases.

Add another table name to the topic list. See `jdbc-connector-2.yaml`.

Update the JDBC Connector and it's done.

```shell
oc apply -f jdbc-connector-2.yaml
```

Check the table list in the Postgres database and the content of the products table.
```shell
oc exec -it $(oc get pods -o custom-columns=NAME:.metadata.name --no-headers -l deploymentconfig=postgres) -- bash -c 'psql -U $POSTGRESQL_USER -d $POSTGRESQL_DATABASE -c \\dt'
                     List of relations
 Schema |           Name            | Type  |    Owner     
--------+---------------------------+-------+--------------
 public | mysql_inventory_customers | table | postgresuser
 public | mysql_inventory_products  | table | postgresuser
(2 rows)

oc exec -it $(oc get pods -o custom-columns=NAME:.metadata.name --no-headers -l deploymentconfig=postgres) -- bash -c 'psql -U $POSTGRESQL_USER -d $POSTGRESQL_DATABASE -c "select * from mysql_inventory_products"'
 id  |        name        |                       description                       | weight 
-----+--------------------+---------------------------------------------------------+--------
 101 | scooter            | Small 2-wheel scooter                                   |   3.14
 102 | car battery        | 12V car battery                                         |    8.1
 103 | 12-pack drill bits | 12-pack of drill bits with sizes ranging from #40 to #3 |    0.8
 104 | hammer             | 12oz carpenter's hammer                                 |   0.75
 105 | hammer             | 14oz carpenter's hammer                                 |  0.875
 106 | hammer             | 16oz carpenter's hammer                                 |      1
 107 | rocks              | box of assorted rocks                                   |    5.3
 108 | jacket             | water resistent black wind breaker                      |    0.1
 109 | spare tire         | 24 inch spare tire                                      |   22.2
(9 rows)
```

## Congratulations

You are done!

## Additional features of the Kafka Connect

## Clean up
```shell
oc delete kafkaconnector jdbc-connector, mysql-connector
oc delete kafkaconnect debezium
oc delete bc debezium-build-connect
oc delete is debezium-streams-connect
oc delete dc,svc,secret postgres
oc delete deployment, svc, ic mysql
oc delete secret mysql-secret
oc delete rolebindig connector-role-binding
oc delete role connector-role
oc delete project $PROJECT
```