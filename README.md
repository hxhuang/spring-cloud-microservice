# Environment
+ Java : 1.8+   
+ Spring Boot : 2.0.3.RELEASE   
+ Spring Cloud : Finchley.RELEASE     


# Components

- 服务注册发现：Eureka Server,Eureka Client    
- 服务消费负载均衡：Ribbon、Feign   
- 服务容错保护（降级、隔离、熔断）：Hystrix、Hystrix-Dashboard      
- 服务监控：Turbine,RabbitMQ       
- 服务网关：Spring-Cloud-Gateway,Netflix-Zuul   
- 服务链路追踪：Spring-Cloud-Sleuth、Zipkin(聚合系统调用数据)、RabbitMQ(系统调用数据传输)、 Elasticsearch(系统调用数据持久化)、Kibana(分析和可视化平台)  
- 日志分析系统：Logstash、Elasticsearch、Kibana

# Use       
1、启动eureka-server 服务注册中心，http://localhost:8080      
2、启动user-service 服务，http://localhost:9100,http://localhost:9101     
&emsp; 启动两个服务，用于测试负载均衡功能，启动命令：  
```jshelllanguage
java -jar user-service-0.0.1-SNAPSHOT.jar --server.port=9100     
java -jar user-service-0.0.1-SNAPSHOT.jar --server.port=9101  

java -jar order-service-0.0.1-SNAPSHOT.jar --server.port=9120     
java -jar order-service-0.0.1-SNAPSHOT.jar --server.port=9121  
```
3、启动eureka-ribbon: ribbon服务负载均衡，http://localhost:9200        
4、启动eureka-feign: feign服务负载均衡，http://localhost:9210       
5、启动eureka-feign-hystrix: feign+hystrix服务容错，http://localhost:9220   
6、启动hystrix-dashboard： 服务容错仪表盘监控，http://localhost:9230   
&emsp;&ensp;在hystrix-dashboard 界面输入： http://localhost:9220/actuator/hystrix.stream 监控服务  
7、启动hystrix-turbine：服务监控信息聚合，http://localhost:9240/turbine.stream ，服务监控信息  
8、启动turbine-amqp：服务监控信息收集，使用RabbitMQ消息代理异步实现，http://localhost:9260  
&emsp;&ensp;打开 http://localhost:9230/hystrix 界面输入：http://localhost:9240/turbine.stream 监控服务  
9、启动cloud-gateway： 服务网关转发请求:  
&emsp;&ensp;http://localhost:9300/actuator/gateway/routes/,查看所有Api路由信息  
&emsp;&ensp;http://localhost:9300/user/**,请求User-Service  
&emsp;&ensp;http://localhost:9300/order/**,请求Order-Service         
10、启动zuul-gateway: 服务网关转发请求:  
&emsp;&ensp;http://localhost:9310/v1/user/**,请求User-Service  
&emsp;&ensp;http://localhost:9310/v1/order/**,请求Order-Service  
11、spring cloud sleuth 服务链路追踪  
&emsp;&ensp;1、Centos7 安装 zipkin服务端，`注意：Zipkin 不再推荐我们来自定义 Server 端了，网上的各种教程基本都是之前版本的，基本都数互相抄的，请无视`      
&emsp;&ensp;2、使用命令启动 :  
```jshelllanguage
# 通过环境变量让 Zipkin 从 RabbitMQ 中读取信息，如下：  
java -jar zipkin.jar --RABBIT_ADDRESSES=localhost:5672 --RABBIT_USER=admin --RABBIT_PASSWORD=12345 --RABBIT_VIRTUAL_HOST=sleuth 

or

java -jar zipkin.jar --RABBIT_URI=amqp://admin:12345@localhost:5672/sleuth

``` 
&emsp;&ensp;3、user-service,order-service,zuul-gateway 添加依赖：    
```jshelllanguage
        <!--服务链路追踪-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
        <!--数据传输-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
        </dependency>   
```
&emsp;&ensp;4、user-service,order-service,zuul-gateway 的 application.yml 添加以下配置:   
&emsp;&ensp;&emsp;&ensp;zuul-gateway -> order-service -> user-service 形成调用链路  
```xml
  # 服务追踪配置 
  sleuth:
    web:
      client:
        enabled: true
    sampler:
      # 将采样比例设置为 1.0，也就是全部都需要。默认是 0.1 
      probability: 1.0
  # 指定 zipkin 服务器的地址
  zipkin:
    base-url: http://192.168.10.100:9411/
    enabled: true
    sender:
      type: rabbit
  rabbitmq:
    addresses: 192.168.10.100
    port: 15672
    username: admin
    password: 12345
    virtual-host: sleuth
``` 
&emsp;&ensp;5、http://192.168.10.100:9411/zipkin/ ，查看所有的 trace信息   
&emsp;&ensp;6、将服务追踪trace信息持久化到 Elasticsearch
>>>1、安装Elasticsearch,并启动 http://192.168.10.100:9200/            
>>>2、安装Elasticsearch-head，并启动  http://192.168.10.100:9100/        
>>>3、重新启动 Zipkin 服务，并指定存储方式为 Elasticsearch，数据传输是 RabbitMQ，启动命令如下：
```jshelllanguage
java -jar zipkin.jar --RABBIT_URI=amqp://admin:12345@localhost:5672/sleuth --STORAGE_TYPE=elasticsearch --ES_HOSTS=http://localhost:9200 --ES_HTTP_LOGGING=BASIC

前面是指定用 RabbitMQ,后面是指定存储方式 Elasticsearch
``` 
>>>4、zuul-gateway -> order-service -> user-service 形成调用链路，请求的数据使用RabbitMQ传输并存储在 Elasticsearch 中，浏览数据可查看  
6、使用 Kibana 查看分析 Elasticsearch存数的数据，http://192.168.10.100:5601，不需任何代码，安装 Kibana 配置下Elasticsearch地址即可  

12、搭建ELK日志分析系统，安装配置 Logstash,直接使用以上的 Elasticsearch和 Kibana, Logstash的配置如下：  
```jshelllanguage
input {
  tcp {
    host => "192.168.10.100"
    port => 6510
    mode => "server"
    tags => ["tags"]
    codec => json_lines
  }
}
output {
  stdout{codec =>rubydebug}
  elasticsearch {
   hosts => "127.0.0.1:9200"
  }
}
```
使用命令：`bin/logstash -f config/logstash_es.conf` 启动 logstash，然后在Kibana 创建索引，即可查看日志。










