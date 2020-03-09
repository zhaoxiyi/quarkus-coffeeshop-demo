 # barista-kafka model
 #### pom
 从模块的 pom.xml 文件来分析，这个程序使用 quarkus 的 plugin 比 barista-http 多了很多：
 除了
```
            <artifactId>quarkus-resteasy</artifactId>
            <artifactId>quarkus-resteasy-jsonb</artifactId>
            <artifactId>quarkus-jsonp</artifactId>
            <artifactId>quarkus-smallrye-health</artifactId>
```
还使用到
```
            <artifactId>quarkus-smallrye-reactive-streams-operators</artifactId>
            <artifactId>quarkus-vertx</artifactId>
            <artifactId>quarkus-jsonp</artifactId>
            <artifactId>quarkus-smallrye-reactive-messaging</artifactId>
            <artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
```
这几个 plugin 是为了引入 vertx 这个敏捷反应框架，Eclipse Vert.x 是一个事件驱动且无阻塞的 java 执行框架。这意味着您的应用程序可以使用少量内核线程来处理大量并发。为了配合 Vert.x 框架，quarkus plugin 还引入了配套的 reactive-messageing 处理框架和相应的 messageing-kafka 处理引擎。这样整套无阻碍事件处理框架就直接搭建好了。

其它在 [barista-http](!../barista-http/readme.md#pom) 模块里面解释过的内容就不再解释了。

#### Java Code
整个代码仍然是实现了4个类，``BaristaResource`` `Beverage` `Names` `Order` ，为了实现事件反序列化还加入了一个 `codecs/OrderDeserializer.java` 。
仍然从 Names 和 Order 基础对象开始：
Order用于映射订单，这次它使用了 quarkus runtime annotations 里的 RegisterForReflection 方法。 RegisterForReflection 方法用于注册为反馈模式。 通过反射使用的元素通常不是主调用树的一部分，因此可以消除死代码（如果在其他情况下不直接调用就会被消除）。 要将这些元素包含在本机可执行文件中，您需要对其进行注册以显式反映（register them for reflection explicitly）。这是一个非常常见的使用方法，因为JSON库通常使用反射将对象序列化为JSON：
> 参考 https://quarkus.io/guides/writing-native-applications-tips 来深入了解 RegisterForReflection
```
import io.quarkus.runtime.annotations.RegisterForReflection;

@RegisterForReflection。 
```
JSON序列化库使用Java反射来获取对象的属性并将其序列化。与GraalVM一起使用(Quarkus 不仅可以使用标准 JVM 也可以使用更轻量级更新的 GrallVM )本机可执行文件时，需要注册所有将与反射 Reflection 一起使用的类。 
> 好消息是Quarkus大部分时间都能为您自动完成这个工作，很多时候即使您没有进行反射使用，也会一切正常。不过对于会引入外部（kafka）工具来序列化时，标注清楚 RegisterForReflection 是一个好习惯。
Names 由于还是个模拟程序，没有什么需要序列化的所以没有改变，而 Beverage 也被注册了 RegisterForReflection 。

我们还是关注在 KafkaBarista 类，这回我们不在使用 resteasy/ JSON 通信了。而是直接接收外部事件
```code
import org.eclipse.microprofile.reactive.messaging.Incoming;
import org.eclipse.microprofile.reactive.messaging.Outgoing;
....
import javax.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class KafkaBarista {
...
    @Incoming("orders")
    @Outgoing("queue")
...
```
在类生命上面的 `@annotation` 声明了这个类是全局工作 `ApplicationScoped` ，因此在协调整个应用的过程中，这个类都可以全局实现序列化和反序列化以便与 kafka 实现通信。 而 `@Incoming("orders")` 表明应用接收外来的 orders 消息作为输入，因此他会进入到 `public CompletionStage<Beverage> process(Order order)` 这个公开类的输入位置，也就是输入参数 Order 类派生的 order 对象。没收到一个事件就会触发一次这个公开累的 process 方法（因为这个声明就在这个方法上面）。 而返回参数 return 的线程中最终 return 的结果 那个 coffee 对象，则会进入 @Outgoing("queue") 事件。
现在赶紧去看看 resources/application.properties 里写了什么
```
quarkus.http.port=8082

## Orders topic
mp.messaging.incoming.orders.connector=smallrye-kafka
mp.messaging.incoming.orders.value.deserializer=me.escoffier.quarkus.coffeeshop.codecs.OrderDeserializer
mp.messaging.incoming.orders.auto.offset.reset=earliest
mp.messaging.incoming.orders.group.id=baristas

## Queue topic
mp.messaging.outgoing.queue.connector=smallrye-kafka
mp.messaging.outgoing.queue.value.serializer=io.quarkus.kafka.client.serialization.JsonbSerializer
```
它不仅写了，quarkus 程序会监听 http 协议，8082 端口。还描述了 Order topic 和 Queue topic 是怎么映射的。`mp.messaging.incoming.orders.connector` 描述了当前 Vert.x 框架下 orders 这个事件流会对接到 kafka 上。这里是使用 `smallrye-kafka` 这个 plugin 来实现对接的。而从 kafka 接收订单时，转为消息需要反序列化，那么就需要指定反序列化的类 `mp.messaging.incoming.orders.value.deserializer=me.escoffier.quarkus.coffeeshop.codecs.OrderDeserializer` 
> 从这里我们可以学习到 Quarkus 的反序列化手段，从 codecs/OrderDeserializer.java 里我们可以看到，反序列化很简单 
```
@RegisterForReflection
public class OrderDeserializer extends JsonbDeserializer<Order> { 
```
> 直接扩展 JsonbDeserializer 方法，并将 Order 类作为 Jsonb 的承载者，它直接就完成了从 json-b 格式的反序列化过程。而这个类真正实现只需要一句话， ` super(Order.class);` 表示使用超类 JsonbDeserialier 来将消息反序列化到 Order.class 这个类上就好。

至于 `@outgoing` 的对接就更简单了，定义了 queue 这个队列的 connector ，确保之前 `@Outgoing("queue")` 这个annotation 声明的输出直接对接到 queue 这个队列上，然后指定序列化工具， 序列化工具因为不再牵扯输入给程序某个指定类的问题所以直接使用 `JsonbSerializer` 来实现序列化即可。
 
从这一段程序我们可以看出，利用 Quarkus plugin 引入 Vert.x 的事件流机制之后，我们可以直接让程序与 kafka 的消息队列进行对接，实现几乎无代码的消息与应用逻辑对接。不仅简化了实现过程，减少了维护对接机制带来的各种程序认为控制错误可能性。同时也实现了业务逻辑与消息队列的松耦合。是原本代码依赖的紧耦合变为了更加适合微服务对接的松耦合机制。使整体程序更加适合云化、容器化、微服务化。
