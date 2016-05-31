# WildFly

标签（空格分隔）： 未分类

---
##整体感受：
    更加充分考虑运维需求。
WildFly 的新增主要功能
> * WildFly 8 jdk 要求7+，而WildFly 10 jdk 要求8+
> * web server 采用undertow
> * 端口变少了。只有2个了，一个8080，一个9990
> * 全部的configuration改变，都有审计日志。console界面你修改东西，都有审计日志了。
> * RAC权限的加入
> * WildFly 新的打patch熟悉，允许cli install and rollback

## jdk要求说明
wildfly最新的版本是10.0.0.CR4，jdk要求1.8+

## 端口只有2个了
以前jboss的端口很多，有http port，management port，jndi port，remote port，message port等等。
现在就2个。
> * 8080=http port+websocket port+  JNDI port + ejb invocations port
> * 9990=management port +jmx port + web console port
管理类的端口是9990，变的功能类端口8080

##审计日志的加入
你在console 界面修改，都会有设计日志

##RAC权限的加入
WildFly内置了7个RBAC角色：

> * 监视者：拥有最少的权限。能够读取配置和当前运行状态，不能读取敏感资源和数据，不能查看审计日志和相关资源。
操作员：除拥有监视者的所有权限外，能够修改运行时状态，重新加载或者关闭服务器，暂停/恢复JMS目标。操作员无法修改持久化配置。

> * 维护员：除拥有操作员的所有权限外，能够修改持久化配置，可以部署应用，增加JMS目标等等。维护员能够编辑几乎所有服务器和部署相关的配置。但是，维护员不能读取和修改敏感信息（例如密码），不能读取或修改审计信息。
部署员：很像维护员，但仅限于部署相关的修改。部署员不能修改通用服务配置。

> * 管理员：能够查看和修改敏感信息例如密码，安全域设置。但是对审计日志不能进行任何操作。
审计员：拥有监视者所有权限。绝大部分都是只读的，但是能够查看和修改审计日志系统相关的配置。
> * 超级用户：等同于AS 7的管理员，拥有所有权限。
RBAC数据可以存储在几乎所有LDAP服务器上，也包括活动目录。

##undertow
web server 改用undertow。以前是基于tomcat编写的一个 jboss web。

新写一个web server的原因：
> * tomcat 不是jboss开发那几个人写的，而且10年前就有了，体系结构已经不能很灵活的跟jboss结合了。
> * 嵌入式需求。不论是JBoss应用服务器本身，还是目前微服务(MicroService)的设计倾向，都希望能够web容器足够小而精悍，可以嵌入使用，而Tomcat很难满足这个需要。
> * 支持Websocket协议需求和NIO，异步化思路等的促进。异步通信框架在并发连接数，性能的优异表现促使开发人员想重新按照新的思路来设计一款Web服务器

#undertow 性能
http://www.techempower.com/benchmarks/#section=data-r6&hw=ec2&test=plaintext
排名第一。
是不是很奇怪，为什么跟netty这种框架一起比较。不是应该跟tomcat，jetty这种容器对比吗。
看下测试说明：

In this test, the framework responds with the simplest of responses: a "Hello, World" message rendered as plain text. The size of the response is kept small so that gigabit Ethernet is not the limiting factor for all implementations. HTTP pipelining is enabled and higher client-side concurrency levels are used for this test (see the "Data table" view).

Example response:
```xml
HTTP/1.1 200 OK
Content-Length: 15
Content-Type: text/plain; charset=UTF-8
Server: Example
Date: Wed, 17 Apr 2013 12:00:00 GMT

Hello, World!
```
只是一个http server 返回一个hello world就好，不需要java EE的一些功能，如 jsp 解析，单纯的一个http server。
就像，下面 undertow 代码
```java
public class HelloWorldServer {  
  
    public static void main(final String[] args) {  
        Undertow server = Undertow.builder()  
                .addHttpListener(8080, "localhost")  
                .setHandler(new HttpHandler() { //设置HttpHandler回调方法  
                    @Override  
                    public void handleRequest(final HttpServerExchange exchange) throws Exception {  
                        exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "text/plain");  
                        exchange.getResponseSender().send("Hello World");  
                    }  
                }).build();  
        server.start();  
    }  
}  
```

所以，可以跟netty这种比较，因为netty 也可以自己写个http server
undertow jar包介绍，都模块化，分jar了，这样也可以看出，undertow 适用用于嵌入式web。
undertow-core ：undertow 核心包。http server
undertow-jastow ：jsp 解析包
jaxws-undertow-httpspi ： jaxws 包
undertow-websockets-jsr ：websocket 包
undertow-servlet ： servlet容器包


##wildfly 结构
![此处输入图片的描述][1]


  [1]: http://ifeve.com/wp-content/uploads/2013/10/4.png
  
  配置文件存放地方的改变，
  1.以前bin目录、subsystem等一些文件都在build目录下面，现在转到了feature-pack
  
  看standlone.xml
  多了：
  ```
  <extension module="org.wildfly.extension.batch.jberet"/>
  <extension module="org.wildfly.extension.undertow"/>
  <extension module="org.wildfly.extension.messaging-activemq"/>
  ```
  > * jberet 为 batch的jboss 的实现
  > * undertow 为 web server
  > * activemq 定义在full里面，这跟jboss 7 一样
```xml
<audit-log>
            <formatters>
                <json-formatter name="json-formatter"/>
            </formatters>
            <handlers>
                <file-handler name="file" formatter="json-formatter" path="audit-log.log" relative-to="jboss.server.data.dir"/>
            </handlers>
            <logger log-boot="true" log-read-only="false" enabled="false">
                <handlers>
                    <handler name="file"/>
                </handlers>
            </logger>
        </audit-log>
```
审计日志模块

```xml
<concurrent>
                <context-services>
                    <context-service name="default" jndi-name="java:jboss/ee/concurrency/context/default" use-transaction-setup-provider="true"/>
                </context-services>
                <managed-thread-factories>
                    <managed-thread-factory name="default" jndi-name="java:jboss/ee/concurrency/factory/default" context-service="default"/>
                </managed-thread-factories>
                <managed-executor-services>
                    <managed-executor-service name="default" jndi-name="java:jboss/ee/concurrency/executor/default" context-service="default" hung-task-threshold="60000" keepalive-time="5000"/>
                </managed-executor-services>
                <managed-scheduled-executor-services>
                    <managed-scheduled-executor-service name="default" jndi-name="java:jboss/ee/concurrency/scheduler/default" context-service="default" hung-task-threshold="60000" keepalive-time="3000"/>
                </managed-scheduled-executor-services>
            </concurrent>
```

JEE concurrent 配置

##WildFly 新的打patch熟悉
jboss 官方有patch，可以按照patch方式升级。
以前都是整个jboss升级


