## 配置文件
docker-compose.yaml

```docker-compose
version: '3'
services:
  zookeeper:
    image: zookeeper:3.4.9
    hostname: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_PORT: 2181
      ZOO_SERVERS: server.1=zookeeper:2888:3888
    volumes:
      - ../../data/zookeeper/data:/data
      - ../../data/zookeeper/datalog:/datalog


  kafka1:
    image: confluentinc/cp-kafka:5.3.0
    hostname: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka1:19092,LISTENER_DOCKER_EXTERNAL://192.168.0.106:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_BROKER_ID: 1
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - ../../data/kafka1/data:/var/lib/kafka/data
    depends_on:
      - zookeeper

  kafka2:
    image: confluentinc/cp-kafka:5.3.0
    hostname: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka2:19093,LISTENER_DOCKER_EXTERNAL://192.168.0.106:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_BROKER_ID: 2
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - ../../data/kafka2/data:/var/lib/kafka/data
    depends_on:
      - zookeeper

  kafka3:
    image: confluentinc/cp-kafka:5.3.0
    hostname: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka3:19094,LISTENER_DOCKER_EXTERNAL://192.168.0.106:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_BROKER_ID: 3
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - ../../data/kafka3/data:/var/lib/kafka/data
    depends_on:
      - zookeeper

  phoenix:
    image: phoenix:v0
    hostname: phoenix
    container_name: hbase-phoenix
    depends_on:
      - zookeeper
    ports:
      - "8765:8765"


  kafdrop:
    image: obsidiandynamics/kafdrop
    restart: "no"
    ports:
      - "9000:9000"
    environment:
      KAFKA_BROKERCONNECT: "kafka1:19092"
    depends_on:
      - kafka1
      - kafka2
      - kafka3





```

docker ps 信息:
```
31a0677e3c9f        obsidiandynamics/kafdrop      "/kafdrop.sh"            About an hour ago   Up About an hour    0.0.0.0:9000->9000/tcp                       kafak_kafdrop_1
c9e9c973f5b8        phoenix:v0                    "./start-hbase-phoen…"   About an hour ago   Up About an hour    0.0.0.0:8765->8765/tcp                       hbase-phoenix
f69c8653db89        confluentinc/cp-kafka:5.3.0   "/etc/confluent/dock…"   About an hour ago   Up About an hour    0.0.0.0:9092->9092/tcp                       kafak_kafka1_1
4ab008dfe38a        confluentinc/cp-kafka:5.3.0   "/etc/confluent/dock…"   About an hour ago   Up About an hour    9092/tcp, 0.0.0.0:9094->9094/tcp             kafak_kafka3_1
1b2abcb4bf1d        confluentinc/cp-kafka:5.3.0   "/etc/confluent/dock…"   About an hour ago   Up About an hour    9092/tcp, 0.0.0.0:9093->9093/tcp             kafak_kafka2_1
9765e443644e        zookeeper:3.4.9               "/docker-entrypoint.…"   About an hour ago   Up About an hour    2888/tcp, 0.0.0.0:2181->2181/tcp, 3888/tcp   kafak_zookeeper_1

```

测试:
(1) 进入phoenix 环境内部
./sqlline.py  zookeeper:2181 可以连接成功

(2)外部 假设有一个phoenix的安装包
执行:
./sqlline-thin.py http://localhost:8765
可以通过8765 来访问phoenix,zookeeper:2181 无法连接

(3) JDBC URL
pom文件:
```
<dependency>
            <groupId>com.aliyun.phoenix</groupId>
            <artifactId>ali-phoenix-core</artifactId>
            <version>5.1.0-HBase-2.0</version>
            <exclusions>
                <exclusion>
                     <groupId>org.glassfish</groupId>
                     <artifactId>javax.el</artifactId>
                </exclusion>
            </exclusions>
        </dependency>



        <!-- https://mvnrepository.com/artifact/com.aliyun.phoenix/ali-phoenix-shaded-thin-client -->
        <dependency>
            <groupId>com.aliyun.phoenix</groupId>
            <artifactId>ali-phoenix-shaded-thin-client</artifactId>
            <version>5.1.0-HBase-2.0.0.2</version>
        </dependency>
```

代码:
```java
package com.atguigu.gmall.realtime.app;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.util.Properties;

public class PhoenixExample {
    public static void main(String args[]) throws Exception {
        Connection conn;
        Properties prop = new Properties();
        Class.forName("org.apache.phoenix.jdbc.PhoenixDriver");
        conn =  DriverManager.getConnection("jdbc:phoenix:thin:url=http://localhost:8765;serialization=protobuf");
        System.out.println("got connection");
        ResultSet rst = conn.createStatement().executeQuery("select * from student");
        while (rst.next()) {
            System.out.println(rst.getString(1) + ":" + rst.getString(2));
        }
    }
}

```



