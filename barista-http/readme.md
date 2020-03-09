 # barista-http model
 #### pom
 从模块的 pom.xml 文件来分析，这个程序使用 quarkus 的以下 plugin：
 ```bash
            <artifactId>quarkus-resteasy</artifactId>
            <artifactId>quarkus-resteasy-jsonb</artifactId>
            <artifactId>quarkus-jsonp</artifactId>
            <artifactId>quarkus-smallrye-health</artifactId>
```
Build plugin 使用了：
```bash
                <artifactId>quarkus-maven-plugin</artifactId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${surefire-plugin.version}</version>
                <configuration>
                    <systemProperties>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                    </systemProperties>
                </configuration>
 ```
 这样 barista-http 模块就自动使用了 resteasy、json 来实现 json rest 通信。和容器所需的 smallrye-health 来保证识别容器自己的健康状态。 在 surefire-plugin 里指定了使用jboss logmanager来管理 log 输出。

#### Java Code
整个代码实现了4个类，``BaristaResource`` `Beverage` `Names` `Order` 其中 Names 和 Order 明显是基础对象，用于映射订单和咖啡名称。 这里的标准 Java 程序与 Qurkus 基本没什么关系，非常简单，在 Names 里映射了一个固定的咖啡名 List 对象，使用 Random 来随机返回一个咖啡名。  `Beverage` 和 `Order` 都是没有逻辑的标准 Session Bean ， Order用于映射订单的信息，Beverage 用于反应是饮料类。都没有持久化。
这里主要讲 BaristaResource 这是一个基于 Quarkus 能力的公开类，用于接受 resteasy/ JSON 格式的通信。
```code
@Path("/barista")
@Produces(MediaType.APPLICATION_JSON)
public class BaristaResource {
```
在类生命上面的两个 `@annotation` 分别声明了这个类接受 `容器对外IP监听/barista` 这个位置的 rest 通信。并且接收到信息后会通过 `MediaType.APPLICATION_JSON` 格式来进行报文处理 `@Produces` 。
> 从这里我们可以学习到 Quarkus 的 `@annotation` 与 Springboot 非常接近（基本上目前 Quarkus 可以兼容 90% 的 Springboot annotation）这使得 Quarkus 的开发非常简练，而且学习途径非常短。基本上熟悉 Springboot 的同学直接就可以使用 Quarkus 来开发了。而另一方面的好处是，基于 Quarkus 开发的代码相对于 Springboot 来说会轻量级很多，它只会为编译结果加载必须的类库。这使得便有过程可以缩短，编译结果可以更小。并且 Quarkus 的编译选项中有 doker native 选项。这样的编译会自动生成 docker 目录及相应的 docker file 并生成可运行的镜像。这使得开发者节省更多时间免于花费精力去管理如何生成镜像。
```   
    @POST
    public CompletionStage<Beverage> process(Order order) {
```
这里的 `@POST` 解决了如果接收到 HTTP 的 POST 通信会被这里接收，并会直接使用 process 来处理参数中的 Order 订单。
> 不过这个类里其实还有一个经典使用方法，就是对 Java thread 的使用。 Java 初学者如果没有很多使用 Java 多线程的经验可以参考这个例子。 这里面引入了下面几个类
```
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
```
> 这里 java.util.concurrent.Executors framework 很好的封装了 java.lang.thread 方法。你不再需要写一系列复杂的逻辑来维护自己的 java thread 线程池，而是直接利用 Executors 的各种方法来维护一套复杂的多线程逻辑。本例子中通过 `private ExecutorService queue = Executors.newSingleThreadExecutor();` 这个私有对象声明了一个新线程对象，每次 REST 收到 POST 请求处理 process 方法的时候，都会通过 CompletableFuture.supplyAsync(() ,queue) 来生成一个新的处理线程。 而处理线程会构建一个 Beverage 类的 coffee 对象，并通过 prepare 方法来处理以下输出参数 order 订单参数（prepare 其实就是用线程的 sleep 方法模拟了一下处理，并没有做任何事）而 Coffee 对象就会处理一下 order订单。  

