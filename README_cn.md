# Coffeeshop Demo with Quarkus
中文版说明
这个项目是来自于 cescoffier/quarkus-coffeeshop-demo 的一个用于演示如何在Quarkus中使用Kafka _reactive_ 来完成一个真实的电商业务的一组演示。它通过 Streamzi 项目（商业化产品为 Redhat AMQ Streams）简化了分布式实现 Kafka 的过程，并且展示了基于分布式 kafka 实现在可靠数据下实现系统的弹性和弹性的能力。本项目不仅适合于传统开发，适合于 Kubernetes 也适合于 Redhat Openshift。 在为这个项目添加更详细的同时，我计划为这个项目增加 OpenShift 部署流程，从而让这个项目可以直接使用 Redhat 商业化版本的 Operator 和 AMQ Streams。 这将更有助于帮助用户学习如何在 OpenShift 快速构建一个商业化应用。

## Build
构建过程需要准备一些基础环境，首先服务器需要具备 maven2 、docker、docker-compose、kafka客户端，如果在 Redhat Linux 上我们可以如下来准备基础环境
```bash
yum install maven2 docker docker-compose
```
至于kafka的客户端则需要去 Apache 网站下载 https://kafka.apache.org/quickstart 。
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

Install Kafka locally for the Kafka tools e.g.

```bash
brew install kafka
```

Run Kafka with:

```bash
docker-compose up
```

In case of previous run, you can clean the state with

```bash
docker-compose down
docker-compose rm
```

Then, create the `orders` topic with `./create-topics.sh`

# Run the demo

You need to run:

* the coffee shop service
* the HTTP barista
* the Kafka barista

Im 3 terminals: 

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
