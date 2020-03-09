# Coffeeshop Demo with Quarkus
> 中文版说明
这个项目是来自于 cescoffier/quarkus-coffeeshop-demo 的一个用于演示如何在Quarkus中使用Kafka _reactive_ 来完成一个真实的电商业务的一组演示。它通过 Streamzi 项目（商业化产品为 Redhat AMQ Streams）简化了分布式实现 Kafka 的过程，并且展示了基于分布式 kafka 实现在可靠数据下实现系统的弹性和弹性的能力。本项目不仅适合于传统开发，适合于 Kubernetes 也适合于 Redhat Openshift。 在为这个项目添加更详细的同时，我计划为这个项目增加 OpenShift 部署流程，从而让这个项目可以直接使用 Redhat 商业化版本的 Operator 和 AMQ Streams。 这将更有助于帮助用户学习如何在 OpenShift 快速构建一个商业化应用。

## Build
构建过程需要准备一些基础环境，首先服务器需要具备 maven2 、docker、docker-compose、kafka客户端，如果在 Redhat Linux 上我们可以如下来准备基础环境
```bash
yum install maven2 docker docker-compose
```
至于kafka的客户端则需要去 Apache 网站下载：
###### kafka quickstart
 https://kafka.apache.org/quickstart 。
这个项目中使用了 Streamzi 的 Kafka 镜像，然后使用 docker-compose 统一构建，这样使用有两个好处。一个是它可以快速的基于 Streamzi 的 kafka 管理能力移植到 kubernetes/Openshift 上。 另一个在单机构建用于开发调试的时候用户不用话费太多精力来构建、维护 kafka 。甚至不用非常理解 kafka 的知识，直接使用 docker-compose 就能构建出一个包含了 zookeeper 和 分布式 kafka 的基础运行环境。
因此在使用上面 Apache kafka 的 quickstart 时，只需要下载，解压，并将 kafka 命令行加入 PATH 就可以了，其它都可以不管。
```bash
cd /opt
tar -xzf kafka_2.12-2.4.0.tgz
export PATH=$PATH:/opt/kafka_2.12-2.4.0/bin
```
这样将 kafka bin 加入到 PATH 下后，后面的 create-topics.sh 就可以运行了。
> 在宿主机上下载 kafka 其实就是为了执行这个脚本建立 Topic 如果有其它环境可以访问由 docker-compose 拉起来的 streamzi 镜像，也可以不安装 kafka。

有了基础环境就可以在宿主机上编译代码构建 Demo 了：
> 你可以从这里 clone 代码 `git clone https://github.com/zhaoxiyi/quarkus-coffeeshop-demo.git` 也可以到源去 clone `git clone https://github.com/cescoffier/quarkus-coffeeshop-demo.git`
Clone 代码后进入代码目录编译
```bash
cd ./quarkus-coffeeshop-demo
mvn clean package
```

## Prerequisites
> 具备条件可以结合前面的介绍完成，在 RHEL 上通常不使用 brew 来安装，不过从 mac 上可以。例如，我个人是通过 Mac 在远程宿主机 RHEL 上安装这个例子的 ，所以我可以在 mac 使用 brew 安装 kafka ，然后远程使用脚本创建 Topic。
```bash
brew install kafka
```
在宿主机 RHEL 上执行 docker-compse，他会去调用 docker-compose.yaml 来构建两个镜像，并将之以yaml文件的要求形式在宿主机上启动:
```bash
docker-compose up
```
> 在 docker-compose.yaml 中包含了两个 service `zookeeper` 和 `kafka` 。 `zookeeper` 用于维护多个 kafak 服务之间的一致性服务能力， `kafka` 则是真正承担消息队列任务的服务容器。 但是我们可以看到两个 service 都使用的是 `strimzi/kafka:0.15.0-kafka-2.3.1` 镜像。这个镜像里包含了完整的基础容器和 kafka 安装。kafka 安装里则包含了自身所需的 zookeeper。 在独立安装 kafka 的时候（参考[kafka quickstart](#kafka quickstart)里的内容）是需要在宿主机上配置 `config/zookeeper.properties` 和 `config/server.properties` 并以此为基础启动多个 Java VM 来执行 kafka 的服务器的。 而通过 docker-compose 则只需要在 yaml 中指定已经匹配好的带有kafka指定版本的基础镜像 `strimzi/kafka:0.15.0-kafka-2.3.1` 然后在 yaml 文件里的 command 段中描述好每个容器需要启动的命令，例如 zookeeper 容器就执行如下配置： 
```bash
command: [
      "sh", "-c",
      "bin/zookeeper-server-start.sh config/zookeeper.properties"
    ]
    ports:
      - "2181:2181"
    environment:
      LOG_DIR: /tmp/logs
```
这里面指定了执行 zookeeper 的方式，和容器映射的端口 `2181` ,以及存放 Log 的目录 `LOG_DIR: /tmp/logs`。
而 kafka server 则要多些内容
```bash
   command: [
      "sh", "-c",
      "bin/kafka-server-start.sh config/server.properties --override listeners=$${KAFKA_LISTENERS} --override advertised.listeners=$${KAFKA_ADVERTISED_LISTENERS} --override zookeeper.connect=$${KAFKA_ZOOKEEPER_CONNECT}"
    ]
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      LOG_DIR: "/tmp/logs"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```
这里将 kafka 的 advertised_listeners/listeners 都通过环境变量定义，这也是通用的容器处理办法，这意味着将其改为kubernetes/OpenShift的时候，这套方案仍然直接可用。此外 zookeeper connect 也被设为环境变量，并且使用服务名 `zookeeper` 加其发布的端口的形式告知给 docker-compose 。docker-compose 则会依据这些信息将两个容器组合成为一个完整的 kafka 服务。在这里我们可以考虑再增加一个 kafka 容器，来模拟更多分布式 kafuka 在分布式环境中服务。

使用上述 docker-compose 的方式来启动服务具备累一键式，可配置，可调整的服务特性，它的服务模型与 kubernetes/Openshift非常接近，可以作为 kubernetes/OpenShift 项目在本地实现的轻量级模拟环境。
> 当然也可以考虑在本地直接使用 OpenShift 的开发版环境 CodeReadyContainer （CRC）来直接建立本地 OpenShift 环境，直接用于本地开发使用。使用CRC会在本地通过虚机直接建立OpenShift All-in-one 环境。并且会直接关联至 Redhat 官方镜像源，用户可以完全按照正式 OpenShift 环境来使用。当然由于它的镜像库在 Redhat 官方网站，你需要将CRC环境连接网络才可以正常使用。不过在一些特殊情况下，你可以先把所要运行的环境执行一遍，这样CRC会把所需的镜像下载并加载至镜像流中（Image Stream）这样就可以离线使用这些功能。通过这种手段，在特殊环境里，你可以吧用过的环境离线出来。

In case of previous run, you can clean the state with
如果需要清楚 docker-compose 建立的环境可以执行如下命令
```bash
docker-compose down
docker-compose rm
```

接下来可以使用 `./create-topics.sh` 来建立 `orders` topic 。其实 create-topic.sh 命令也可以在任何其它环境来执行，只要你能够访问由 docker-compose 启动的镜像所监听的 9092 端口即可。

```bash
kafka-topics --bootstrap-server localhost:9092 --create --partitions 4 --replication-factor 1 --topic orders
```
这里使用 kafka-topics 命令时使用的 localhost 换成 docker-compose 所在的宿主机 IP 就可以远程使用。
> 因为 docker-compose.yaml 里面  `ports: - "9092:9092"` 意味着将容器的 `9092` 端口映射给宿主机的 `9092` 端口。这样你访问宿主机的 `9092` 端口就会被转入容器中，并被容器中监听 `9092` 端口的 Kafka 接手。这也是为什么你从远程也可以使用宿主机的原因。
这里使用到的 `--partitions 4 --replication-factor 1` 意味着我们会给 kakfa 分为4个 partitions分区，当Message 发往 Topic 时，Kafka broker 就会将这个 Message 写一份主体外，在另一个分区 partitions 上再写一个副本，一旦主体失效，这个副本会生效，并会给自己再生成一个副本。
> 这里就涉及到为什么要使用 streamzi 了, streamzi 镜像里已经融合了如何给 kafka 建立 partitions ，broker 以及如何和 zookeeper 配合工作的管理逻辑。 docker-compose 只在一个简单的物理环境上模拟执行，这时存储还不是很难管理，反正都在一个服务器的目录里。当这个应用搬迁到 kubernetes/OpenShift 上的时候，就会牵扯到，我们究竟是让 Message 存储在容器里还是容器所映射的存储的（PV/PVC的申请与使用）问题，同样就会遇到 kafka 的 partitions 和这些容器映射的存储如何配合构建，如果复制 Meesage 如何保障我的 Message 没有存在一个物理硬件上（一个硬件坏了我丢了所有的副本怎么办？）。容器调度的时候，我新的kafka Broker 是重新申请存储，还是使用原有的存储（PV/PVC），恢复的时候我如何保证我的 Message 是有效的，它究竟是主版本还是复制版本，它是不是已经被其它 Broker 使用了，等等一系列的问题。再扩展一些，如果涉及到多 Kafka 集群间同步、复制等问题，这件事会更加复杂。 Streamzi （ Redhat 商业版本叫做 AMQ Streams）就是解决这些问题的完整方案。  
# Run the demo

当执行Demo时，需要分别执行以下模块:

* the coffee shop service
* the HTTP barista
* the Kafka barista

这3个模块可以分别从三个终端分别执行，他们并非紧耦合的，而是可以各自执行的松耦合模型: 

```bash
cd coffeeshop-service
mvn compile quarkus:dev
```

```bash
cd barista-http
java -jar target/barista-http-1.0-SNAPSHOT-runner.jar
```

```bash
cd barista-kafka
mvn compile quarkus:dev
```

# Execute with HTTP

The first part of the demo shows HTTP interactions:

* Barista code: `me.escoffier.quarkus.coffeeshop.BaristaResource`
* CoffeeShop code: `me.escoffier.quarkus.coffeeshop.CoffeeShopResource.http`
* Generated client: `me.escoffier.quarkus.coffeeshop.http.BaristaService`

Order coffees by opening `http://localhost:8080`. Select the HTTP method.

Stop the HTTP Barista, you can't order coffee anymore.

# Execute with Kafka

* Barista code: `me.escoffier.quarkus.coffeeshop.KafkaBarista`: Read from `orders`, write to `queue`
* Bridge in the CoffeeShop: `me.escoffier.quarkus.coffeeshop.messaging.CoffeeShopResource#messaging` just enqueue the orders in a single thread (one counter)
* Get prepared beverages on `me.escoffier.quarkus.coffeeshop.dashboard.BoardResource` and send to SSE

* Open browser to http://localhost:8080/
* Order coffee with Order coffees by opening `http://localhost:8080`. Select the HTTP method. Select the messaging method.

# Baristas do breaks

1. Stop the Kafka barista
1. Continue to enqueue order
1. On the dashboard, the orders are in the "IN QUEUE" state
1. Restart the barista
1. They are processed

# 2 baristas are better

1. Start a second barista with: 
```bash
./barista-kafka/target/barista-kafka-1.0-SNAPSHOT-runner -Dquarkus.http.port=9999
```
1. Order more coffee

The dashboard shows that the load is dispatched among the baristas.
