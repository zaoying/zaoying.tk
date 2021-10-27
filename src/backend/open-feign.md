# 如何使用OpenFeign+WebClient实现非阻塞的接口聚合

随着微服务的遍地开花，越来越多的公司开始采用 `SpringCloud` 用于公司内部的微服务框架。

按照微服务的理念，每个单体应用的功能都应该按照功能正交，也就是功能相互独立的原则，划分成一个个功能独立的微服务（模块），再通过接口聚合的方式统一对外提供服务！

然而随着微服务模块的不断增多，通过接口聚合对外提供服务的中层服务需要聚合的接口也越来越多！慢慢地，接口聚合就成分布式微服务架构里一个非常棘手的性能瓶颈！

举个例子，有个聚合服务，它需要聚合 `Service`、`Route` 和 `Plugin` 三个服务的数据才能对外提供服务：

```java
@Headers({ "Accept: application/json" })
public interface ServiceClient {
    @RequestLine("GET /")
    List<Service> list();
}

@Headers({ "Accept: application/json" })
public interface RouteClient {
    @RequestLine("GET /")
    List<Route> list();
}

@Headers({ "Accept: application/json" })
public interface PluginClient {
    @RequestLine("GET /")
    List<Plugin> list();
}
```

使用声明式的 `OpenFeign` 代替 `HTTP Client` 进行网络请求

编写单元测试

```java

public class SyncFeignClientTest {
    public static final String SERVER = "http://devops2:8001";
    private ServiceClient serviceClient;
    private RouteClient routeClient;
    private PluginClient pluginClient;
    @Before
    public void setup(){
        BasicConfigurator.configure();
        Logger.getRootLogger().setLevel(Level.INFO);

        String service = SERVER + "/services";
        serviceClient = Feign.builder()
                .target(ServiceClient.class, service);

        String route = SERVER + "/routes";
        routeClient = Feign.builder()
                .target(RouteClient.class, route);

        String plugin = SERVER + "/plugins";
        pluginClient = Feign.builder()
                .target(PluginClient.class, plugin);
    }

    @Test
    public void aggressionTest() {
        long current = System.currentTimeMillis();
        System.out.println("开始调用聚合查询");

        serviceTest();

        routeTest();

        pluginTest();

        System.out.println("调用聚合查询结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
    }

    @Test
    public void serviceTest(){
        long current = System.currentTimeMillis();
        System.out.println("开始获取Service");

        String service = serviceClient.list();
        System.out.println(service);
        System.out.println("获取Service结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
    }

    @Test
    public void routeTest(){
        long current = System.currentTimeMillis();
        System.out.println("开始获取Route");

        String route = routeClient.list();
        System.out.println(route);
        System.out.println("获取Route结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
    }

    @Test
    public void pluginTest(){
        long current = System.currentTimeMillis();
        System.out.println("开始获取Plugin");

        String plugin = pluginClient.list();
        System.out.println(plugin);
        System.out.println("获取Plugin结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
    }
}
```

测试结果：

```
开始调用聚合查询
开始获取Service
{"next":null,"data":[]}
获取Service结束！耗时：134毫秒
开始获取Route
{"next":null,"data":[]}
获取Route结束！耗时：44毫秒
开始获取Plugin
{"next":null,"data":[]}
获取Plugin结束！耗时：45毫秒
调用聚合查询结束！耗时：223毫秒
Process finished with exit code 0
```

可以明显看出：聚合查询查询所用的时间 `223毫秒 = 134毫秒 + 44毫秒 + 45毫秒`

也就是聚合服务的 `请求时间` 与 `接口数量` 成*正比关系*，这种做法显然不能接受！

而解决这种问题的最常见做法就是预先创建线程池，通过多线程并发请求接口进行接口聚合！

这种方案在网上随便百度一下就能找到好多，今天我就不再把它的代码贴出来！而是说一下这个方法的缺点：

原本 `JavaWeb` 的主流 `Servlet容器` 采用的方案是一个HTTP请求就使用一个线程和一个Servlet进行处理！

这种做法在并发量不高的情况没有太大问题，但是由于摩尔定律失效了，单台机器的线程数量仍旧停留在一万左右，在网站动辄上千万点击量的今天，单机的线程数量根本无法应付上千万级的并发量！

而为了解决接口聚合的耗时过长问题，采用线程池多线程并发网络请求的做法，更是火上浇油！原本只需一个线程就搞定的请求，通过多线程并发进行接口聚合，就把处理每个请求所需要的线程数量给放大了，急速降低系统可用线程的数量，自然也降低系统的并发数量！

这时，人们想起从 `Java5` 开始就支持的 `NIO` 以及它的开源框架 `Netty` ！基于 `Netty` 以及 `Reactor模式` ，Java生态圈出现了`SpringWebFlux` 等异步非阻塞的 `JavaWeb` 框架！`Spring5` 也是基于SpringWebFlux进行开发的！有了异步非阻塞服务器，自然也有异步非阻塞网络请求客户端 `WebClient`！

今天我就使用 `WebClient` 和 `ReactiveFeign` 做一个异步非阻塞的接口聚合教程：

首先，打开 `pom.xml` 引入依赖

```xml
<dependency>
    <groupId>com.playtika.reactivefeign</groupId>
    <artifactId>feign-reactor-core</artifactId>
    <version>1.0.30</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.playtika.reactivefeign</groupId>
    <artifactId>feign-reactor-webclient</artifactId>
    <version>1.0.30</version>
    <scope>test</scope>
</dependency>
```

然而基于 `Reactor Core` 重写 `Feign` 客户端，就是把原本接口返回值：`List<实体>` 改成 `FLux<实体>` ，`实体` 改成 `Mono<实体>`

```java
@Headers({ "Accept: application/json" })
public interface ServiceClient {
    @RequestLine("GET /")
    Flux<Service> list();
}

@Headers({ "Accept: application/json" })
public interface RouteClient {
    @RequestLine("GET /")
    Flux<Service> list();
}

@Headers({ "Accept: application/json" })
public interface PluginClient {
    @RequestLine("GET /")
    Flux<Service> list();
}
```

然后编写单元测试

```java
public class AsyncFeignClientTest {
    public static final String SERVER = "http://devops2:8001";
    private CountDownLatch latch;
    private ServiceClient serviceClient;
    private RouteClient routeClient;
    private PluginClient pluginClient;

    @Before
    public void setup(){
        BasicConfigurator.configure();
        Logger.getRootLogger().setLevel(Level.INFO);
        latch= new CountDownLatch(3);

        String service= SERVER + "/services";
        serviceClient= WebReactiveFeign
                .<ServiceClient>builder()
                .target(ServiceClient.class, service);

        String route= SERVER + "/routes";
        routeClient= WebReactiveFeign
                .<RouteClient>builder()
                .target(RouteClient.class, route);

        String plugin= SERVER + "/plugins";
        pluginClient= WebReactiveFeign
                .<PluginClient>builder()
                .target(PluginClient.class, plugin);
}

    @Test
    public void aggressionTest() throws InterruptedException {
        long current= System.currentTimeMillis();
        System.out.println("开始调用聚合查询");

        serviceTest();

        routeTest();

        pluginTest();

        latch.await();
        System.out.println("调用聚合查询结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
}

    @Test
    public void serviceTest(){
        long current= System.currentTimeMillis();
        System.out.println("开始获取Service");

        serviceClient.list()
                .subscribe(result ->{
                    System.out.println(result);
                    latch.countDown();
                    System.out.println("获取Service结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
                });
    }

    @Test
    public void routeTest(){
        long current= System.currentTimeMillis();
        System.out.println("开始获取Route");

        routeClient.list()
                .subscribe(result ->{
                    System.out.println(result);
                    latch.countDown();
                    System.out.println("获取Route结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
                });

}

    @Test
    public void pluginTest(){
        long current= System.currentTimeMillis();
        System.out.println("开始获取Plugin");

        pluginClient.list()
                .subscribe(result ->{
                    System.out.println(result);
                    latch.countDown();
                    System.out.println("获取Plugin结束！耗时：" + (System.currentTimeMillis() - current) + "毫秒");
                });
    }
}
```

这里的关键点就在于原本 `同步阻塞` 的请求，现在改成 `异步非阻塞` 了，所以需要使用 `CountDownLatch` 来同步。
在获取到接口后调用 `CountDownLatch.coutdown()` ，在调用所有接口请求后调用 `CountDownLatch.await()` 等待所有的接口返回结果再进行下一步操作！

测试结果：

```java
开始调用聚合查询

开始获取Service

开始获取Route

开始获取Plugin

{"next":null,"data":[]}

{"next":null,"data":[]}

获取Plugin结束！耗时：215毫秒

{"next":null,"data":[]}

获取Route结束！耗时：216毫秒

获取Service结束！耗时：1000毫秒

调用聚合查询结束！耗时：1000毫秒

Process finished with exit code 0

```

显然，聚合查询所消耗的时间不再等于所有接口请求的时间之和，而是接口请求时间中的最大值！

下面开始性能测试：
普通Feign接口聚合测试调用1000次：

```
开始调用聚合查询
开始获取Service
{"next":null,"data":[]}
获取Service结束！耗时：169毫秒
开始获取Route
{"next":null,"data":[]}
获取Route结束！耗时：81毫秒
开始获取Plugin
{"next":null,"data":[]}
获取Plugin结束！耗时：93毫秒
调用聚合查询结束！耗时：343毫秒
summary: 238515, average: 238
```

使用WebClient进行接口聚合查询1000次：

```
开始调用聚合查询
开始获取Service
开始获取Route
开始获取Plugin
{"next":null,"data":[]}
{"next":null,"data":[]}
获取Route结束！耗时：122毫秒
{"next":null,"data":[]}
获取Service结束！耗时：122毫秒
获取Plugin结束！耗时：121毫秒
调用聚合查询结束！耗时：123毫秒
summary: 89081, average: 89
```

测试结果中，WebClient的测试结果恰好相当于普通FeignClient的三分之一！正好在意料之中！