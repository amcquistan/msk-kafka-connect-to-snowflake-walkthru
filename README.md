# AWS MSK / Kafka Connect to Snowflake Pipeline

Walk thru of AWS Blog Post [Analyze Streaming Data from Amazon Managed Streaming for Apache Kafka Using Snowflake](https://aws.amazon.com/blogs/apn/analyze-streaming-data-from-amazon-managed-streaming-for-apache-kafka-using-snowflake/)

## Provisoning from CloudFormation Template

Create a EC2 SSH Key Pair

```
aws ec2 create-key-pair --key-name kafka-snowflake --query 'KeyMaterial' --output text --region us-east-1 > kafka-snowflake.pem

chmod 400 kafka-snowflake.pem
```

```
STACK_NAME=kafka-snowflake-pipeline-stack
aws cloudformation create-stack \
  --stack-name $STACK_NAME \
  --template-body file://template.yml \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=KeyName,ParameterValue=kafka-snowflake
```

## Kafka Client Instance Setup

SSH onto the Kafka KafkaClientEC2Instance.

Save MSK Cluster ARN to shell variable.

```
STACK_NAME=kafka-snowflake-pipeline-stack
REGION=us-east-1

sudo yum install jq -y

MSK_ARN=$(aws cloudformation describe-stacks --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`MSKClusterArn`].OutputValue | [0]' \
  --stack-name $STACK_NAME | jq -r '.')
```

Fetch Bootstrap Server Endpoints and Zookeeper Endpoint

```
ZK_SERVERS=$(aws kafka describe-cluster --region $REGION \
  --cluster-arn $MSK_ARN | jq -r '.ClusterInfo.ZookeeperConnectString')
  
BOOTSTRAP_SERVERS=$(aws kafka get-bootstrap-brokers --region $REGION \
  --cluster-arn $MSK_ARN | jq -r '.BootstrapBrokerString')
```

Create a Kafka Topic to Publish Messages to.

```
KAFKA_ROOT=$HOME/kafka/kafka_2.12-2.4.0

$KAFKA_ROOT/bin/kafka-topics.sh --create --zookeeper $ZK_SERVERS \
  --replication-factor 3 --partitions 1 --topic testtopic

$KAFKA_ROOT/bin/kafka-topics.sh --create --zookeeper $ZK_SERVERS \
  --replication-factor 3 --partitions 1 --topic MSKSnowflakeTestTopic
```

Verify the topics were created.

```
$KAFKA_ROOT/bin/kafka-topics.sh --list --bootstrap-server $BOOTSTRAP_SERVERS
```

Output.

```
MSKSnowflakeTestTopic
__amazon_msk_canary
__consumer_offse
testtopic
```

Install Maven

```
sudo su -
cd /opt
wget https://www.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar xzf apache-maven-3.6.3-bin.tar.gz
ln -s apache-maven-3.6.3 maven
ln -s apache-maven-3.6.3/bin/mvn mvn
exit

echo "export PATH=$PATH:/opt:" >> ~/.bash_profile
source ~/.bash_profile 
```

Fetch Snowflake and other Dependencies

```
mvn org.apache.maven.plugins:maven-dependency-plugin:2.8:get -Dartifact=com.snowflake:snowflake-kafka-connector:1.1.0:jar

mvn org.apache.maven.plugins:maven-dependency-plugin:2.8:get -Dartifact=org.bouncycastle:bcprov-jdk15on:1.64:jar

mvn org.apache.maven.plugins:maven-dependency-plugin:2.8:get -Dartifact=org.bouncycastle:bc-fips:1.0.1:jar

mvn org.apache.maven.plugins:maven-dependency-plugin:2.8:get -Dartifact=org.bouncycastle:bcpkix-fips:1.0.3:jar
```

Copy dependencies to Kafka libs directory

```
cp ~/.m2/repository/com/snowflake/snowflake-kafka-connector/1.1.0/snowflake-kafka-connector-1.1.0.jar $KAFKA_ROOT/libs/

cp ~/.m2/repository/net/snowflake/snowflake-ingest-sdk/0.9.6/snowflake-ingest-sdk-0.9.6.jar $KAFKA_ROOT/libs/

cp ~/.m2/repository/net/snowflake/snowflake-jdbc/3.11.1/snowflake-jdbc-3.11.1.jar $KAFKA_ROOT/libs/

cp ~/.m2/repository/org/bouncycastle/bcprov-jdk15on/1.64/bcprov-jdk15on-1.64.jar $KAFKA_ROOT/libs/

cp ~/.m2/repository/org/bouncycastle/bc-fips/1.0.1/bc-fips-1.0.1.jar $KAFKA_ROOT/libs/

cp ~/.m2/repository/org/bouncycastle/bcpkix-fips/1.0.3/bcpkix-fips-1.0.3.jar $KAFKA_ROOT/libs/
```

Generate private auth key for Snowflake (use password Develop3r)

```
cd ~
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8
```

Generate public auth key for Snowflake (use password Develop3r)

```
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
```

Create a connection properties file in Home directory named .snowflake-connection.properties

First grab private key contents

```
SNOWFLAKE_PRIVATE_KEY=$(echo `sed -e '2,$!d' -e '$d' -e 's/\n/ /g' ~/rsa_key.p8`|tr -d ' ') 
```

Then make connection file ~/.snowflake-connection.properties

```
name=MskTestData
connector.class=com.snowflake.kafka.connector.SnowflakeSinkConnector
tasks.max=8
buffer.count.records=100
buffer.flush.time=60
buffer.size.bytes=65536
snowflake.url.name=HERE.snowflakecomputing.com
snowflake.user.name= MY_USERNAME
snowflake.database.name=MSKTESTDB
snowflake.schema.name=PUBLIC
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=com.snowflake.kafka.connector.records.SnowflakeJsonConverter
snowflake.private.key=THE-PRIVATE-KEY-BODY (SNOWFLAKE_PRIVATE_KEY)
snowflake.private.key.passphrase=Develop3r
topics=MSKSnowflakeTestTopic
```

Install snowsql client

```
cd ~
curl -O https://sfc-repo.snowflakecomputing.com/snowsql/bootstrap/1.2/linux_x86_64/snowsql-1.2.3-linux_x86_64.bash

mkdir ~/.snowsql
chmod +x snowsql-1.2.3-linux_x86_64.bash
./snowsql-1.2.3-linux_x86_64.bash

export SNOWSQL_PRIVATE_KEY_PASSPHRASE=Develop3r
```

Create snowsql connection config file ~/.snowsql.config

```
[connections]
password=MyPassword
```

Grab public key for snowflake connection and add to snowflake account.

```
SNOWFLAKE_PUBLIC_KEY=$(echo `sed -e '2,$!d' -e '$d' -e 's/\n/ /g' ~/rsa_key.pub`|tr -d ' ')

snowsql -a somealphanumeric.us-east-2.aws -u MY_USERNAME -q "alter user MY_USERNAME set rsa_public_key='$SNOWFLAKE_PUBLIC_KEY'" -r ACCOUNTADMIN
```

Test private key works to auth with Snowflake using snowsql

```
snowsql -a somealphnumerica.us-east-2.aws -u MY_USERNAME --private-key-path=$HOME/rsa_key.p8
```

While in snowsql client create database

```
CREATE DATABASE MSKTESTDB;
```

Prove it.

```
MY_USERNAME#COMPUTE_WH@MSKTESTDB.PUBLIC>show databases;
+-------------------------------+-----------------------+------------+------------+-------------------------+--------------+---------------------------------------------------+---------+----------------+
| created_on                    | name                  | is_default | is_current | origin                  | owner        | comment                                           | options | retention_time |
|-------------------------------+-----------------------+------------+------------+-------------------------+--------------+---------------------------------------------------+---------+----------------|
| 2022-01-13 06:33:33.608 -0800 | FIRSTDB               | N          | N          |                         | ACCOUNTADMIN |                                                   |         | 1              |
| 2022-01-13 07:03:15.749 -0800 | MSKTESTDB             | N          | Y          |                         | ACCOUNTADMIN |                                                   |         | 1              |
| 2022-01-12 13:51:14.016 -0800 | SNOWFLAKE             | N          | N          | SNOWFLAKE.ACCOUNT_USAGE |              |                                                   |         | 1              |
| 2022-01-12 13:51:17.036 -0800 | SNOWFLAKE_SAMPLE_DATA | N          | N          | SFC_SAMPLES.SAMPLE_DATA | ACCOUNTADMIN | Provided by Snowflake during account provisioning |         | 1              |
+-------------------------------+-----------------------+------------+------------+-------------------------+--------------+---------------------------------------------------+---------+----------------+
4 Row(s) produced. Time Elapsed: 0.086s
```

In the UI give MODIFY grant to ACCOUNTADMIN user to database MSKTESTDB



Update config/connect-standalone.properties file to use the MSK cluster's bootstrap servers

Run the connect server in one terminal.

```
$KAFKA_ROOT/bin/connect-standalone.sh $KAFKA_ROOT/config/connect-standalone.properties $HOME/.snowflake-connection.properties
```

In a second terminal connected to Kafka Client EC2 configure Connection Properties

```
cp /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.amzn2.0.2.x86_64/jre/lib/security/cacerts $HOME/kafka.client.truststore.jks

echo "ssl.truststore.location=$HOME/kafka.client.truststore.jks" >> $HOME/kafka-client.properties

echo "security.protocol=ssl" >> $HOME/kafka-client.properties
```


Then use kafka-console-producer.sh to produce data

```
$KAFKA_ROOT/bin/kafka-console-producer.sh --broker-list $BOOTSTRAP_SERVERS \
  $HOME/kafka-client.properties --topic MSKSnowflakeTestTopic
>{"name": "Scooby Doo", "species": "Dog"}
>{"name": "Shaggy", "species": "Human"}
>{"name": "Scrappy Doo", "species": "Dog"}
>{"name": "Fred", "species": "Human"}
>^C[ec2-user@ip-10-0-0-211 ~]$
```

Verify data exists in snowsql

```
snowsql -a somealphanumeric.us.us-east-2.aws -u MY_USERNAME --private-key-path=$HOME/rsa_key.p8 -q "SELECT * FROM MSKTESTDB.PUBLIC.MSKSNOWFLAKETESTTOPIC;"
```

Output.

```
* SnowSQL * v1.2.21
Type SQL statements or !help
+------------------------------------+--------------------------+
| RECORD_METADATA                    | RECORD_CONTENT           |
|------------------------------------+--------------------------|
| {                                  | {                        |
|   "CreateTime": 1642089436995,     |   "name": "Fred",        |
|   "offset": 3,                     |   "species": "Human"     |
|   "partition": 0,                  | }                        |
|   "topic": "MSKSnowflakeTestTopic" |                          |
| }                                  |                          |
| {                                  | {                        |
|   "CreateTime": 1642089374093,     |   "name": "Scooby Doo",  |
|   "offset": 0,                     |   "species": "Dog"       |
|   "partition": 0,                  | }                        |
|   "topic": "MSKSnowflakeTestTopic" |                          |
| }                                  |                          |
| {                                  | {                        |
|   "CreateTime": 1642089397919,     |   "name": "Shaggy",      |
|   "offset": 1,                     |   "species": "Human"     |
|   "partition": 0,                  | }                        |
|   "topic": "MSKSnowflakeTestTopic" |                          |
| }                                  |                          |
| {                                  | {                        |
|   "CreateTime": 1642089419873,     |   "name": "Scrappy Doo", |
|   "offset": 2,                     |   "species": "Dog"       |
|   "partition": 0,                  | }                        |
|   "topic": "MSKSnowflakeTestTopic" |                          |
| }                                  |                          |
+------------------------------------+--------------------------+
4 Row(s) produced. Time Elapsed: 0.147s
Goodbye!
```

## Clean Up

Execute from local terminal 

```
aws cloudformation delete-stack --stack-name kafka-snowflake-pipeline-stack \
  ---region us-east-1
```

Check its status periodically to make sure its destroyed.

```
for i in {1..10}
  do 
  aws cloudformation describe-stacks --stack-name kafka-snowflake-pipeline-stack --region us-east-1 --query "Stacks[].StackStatus" --output table
  sleep 10
done
```