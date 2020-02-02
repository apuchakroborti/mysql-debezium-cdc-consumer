# mysql-debezium-cdc-consumer

Kafka, Zookeeper, etc starting using confluent:
Download the Confluent 5.4.0 platform and extract it to any directory
then extract the file
For 'name*.tar.gz' file
$ tar -xvzf /path/to/the/confluent-5.4.0
$ cd confluent-5.4.0
$ vi ~/.bashrc
then add the following lines ebd of that file's
export CONFLUENT_HOME=/path/to/the/confluent-5.4.0
export PATH=$CONFLUENT_HOME/bin:$PATH

$ source ~/.bashrc
check the confluent version
$  confluent --version
$ confluent local start

Setting up MySQL server:
https://debezium.io/documentation/reference/assemblies/cdc-mysql-connector/as_setup-the-mysql-server.html

Create a new MySql user:
$ mysql -u root -p
Mysql > CREATE USER 'apu'@'localhost' IDENTIFIED BY 'tigerit';
Mysql > GRANT ALL PRIVILEGES ON * . * TO 'apu'@'localhost';
Mysql $ FLUSH PRIVILEGES;

Enabling the MySQL binlog for Debezium:
$ sudo vi /etc/mysql/my.cnf
Add the following lines: 
[mysqld]
server-id         = 223344
log_bin           = mysql-bin
binlog_format     = row
binlog_row_image  = full
expire_logs_days  = 10
performance_schema = ON
show_compatibility_56 = On
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_rows_query_log_events = ON

$ sudo systemctl restart mysql.service
$ sudo systemctl status mysql.service

Checking:
1.Check if the log-bin option is already on or not.
Mysql > SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM information_schema.global_variables WHERE variable_name='log_bin';
2.Confirm your changes by checking the binlog status once more.
Mysql > SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM information_schema.global_variables WHERE variable_name='log_bin';

Confirm the GTID changes:
Mysql> show global variables like '%GTID%';

Debezium MySql Connector Installation:
Download the MySql-Debezium Connector and execute the followings command:
$ cp /yourpath/mysql-plugin/debezium-connector-mysql-1.0.0.Final-plugin.tar.gz /yourpath/confluent-5.4.0/share/java/
$ cd /yourpath/confluent-5.4.0/share/java/
$ tar -xvzf debezium-connector-mysql-1.0.0.Final-plugin.tar.gz
$ rm debezium-connector-mysql-1.0.0.Final-plugin.tar.gz

Start Connector:
Using PostMan: 
Headers:
Key: ContentType
Value: application/json
Body:
raw:
{
"name":"dbserver1",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "127.0.0.1",
    "database.port": "3306",
    "database.user": "apu",
    "database.password": "password",
    "database.server.id": "223344",
    "database.server.name": "mysqldb",
    "database.whitelist": "studentdb",
    "database.history.kafka.bootstrap.servers": "localhost:9092",
    "database.history.kafka.topic": "kafka-connect-test",
    "include.schema.changes": "true",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"

  }
}
Or
Using Curl Command:
$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-mysql.json

Note: Keep the above connector-info JSON text to the 'register-mysql.json' and place it to the directory, from where you will execute the curl command.
Check the connector running or not:
$ curl -s "http://localhost:8083/connectors" | jq '.[]' | xargs -I{dbserver1} curl -s "http://localhost:8083/connectors/"{dbserver1}"/status" | jq -c -M '[.name,.connector.state,.tasks[].state] |  
join(":|:")'| column -s : -t| sed 's/"//g'| sort

Output:
dbserver1 | RUNNING | RUNNING

Start the console consumer:
$ cd /path/confluent-5.4.0
$ ./bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 --property schema.registry.url=http://localhost:8081 --topic mysqldb.studentdb.customers --from-beginning | jq '.'

List the topics:
$ ./bin/kafka-topics --list --zookeeper localhost:2181

Use the group-id or topic name 
mysqldb.studentdb.customers
[From connector JSON File,
"database.server.name"."database.whitelist"."table_name"]

Start the SpringBoot Consumer:
$ cd /path/to/project
$ mvn spring-boot:run

Change the MySql Database:
$ mysql -u apu -p
password
$ show databases;
$ CREATE DATABASE  studentdb;
$ USE studentdb;
$ CREATE TABLE Customers (
   id int NOT NULL PRIMARY KEY AUTO_INCREMENT,
   first_name varchar(255) NOT NULL,
   last_name varchar(255) NOT NULL,
   age int NOT NULL,   
address varchar(255)NOT NULL
);
$ INSERT INTO Customers (first_name, last_name, age, address)
VALUES ('Apu', 'Chakroborti', 26, 'abc');
Then check the spring boot consumer
Clean up resources:
$ curl -X DELETE localhost:8083/connectors/dbserver1
$ confluent local stop
