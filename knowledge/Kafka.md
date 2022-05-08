## Kafka

### 配置Kafka服务器

- 下载kafka包:

  - 地址：https://kafka.apache.org/downloads
  - 需要注意的点： 下载的包不要使用source源码包，需要选择编译过后的包。

- 放到服务器后，执行解压操作

  ```shell
  > tar -zxf kafka.tgz.tar
  ```

- 解压完成后，进入kafka目录，修改配置文件：

  ```shell
  > cd kafka_2.13-3.1.0/config/
  > vim server.properties
  ```

  - 配置文件中，需要注意主要的配置项：
    - broker.id: 这个是唯一标志此broker，id不可重复。
    - advertised.listeners: 如果我们需要把kafka服务器对外提供服务，则需要配置此项，填写本机的ip和端口号。需要注意，必须保证防火墙策略是关掉的，才能接受外部连接。（外网连接）如果是租的服务器，需要去对应的服务器配置页面设置防火墙策略，否则会被拦截。
    - listener: 这个是内网访问kafka服务的时候需要配置ip；
    - zookeeper.connecct: 表示kafka需要连接的zookeeper集群。

- 启动kafka:

  - 进入bin目录下，执行kafka-server-start.sh脚本。

    ```shell
    > bin/kafka-server-start.sh -daemon conf/server.properties
    ```

    -daemon表示作为后台进程启动。

  - 启动完成后，可以通过jps命令查看当前进程。

  - 测试是否可以连接：

    - 可以使用kafka-topics.sh脚本查看topic，来确认kafka正确启动。

      ```
      > bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test
      ```

      如果能创建成功，表示kafka无问题。

    - kafka topic相关命令：

      - --create：创建topic。
      - --list：列出当前kafka上所有的topic。
      - --describe：描述topic具体信息。
      - --alter：修改topic相关配置。
      - --delete：删除指定的topic。
      - --topic：指定topic。
      - --partitions：指定当前kafka分区数量，默认位1个。
      - --replication-factor：指定每个分区的复制因子个数。

    - 