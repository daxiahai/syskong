- [1. 简介](#1-简介)
- [2. 架构](#2-架构)
  - [2.1. 特性](#21-特性)
  - [2.2. 组件](#22-组件)
- [3. 插件](#3-插件)
- [4. 请求流程](#4-请求流程)
- [5. 基于postgresql的一键安装](#5-基于postgresql的一键安装)
- [6. kong简单使用](#6-kong简单使用)
  - [6.1. login](#61-login)
  - [6.2. 负载均衡配置](#62-负载均衡配置)
    - [6.2.1. 创建upstream](#621-创建upstream)
    - [6.2.2. 创建 service 和 route](#622-创建-service-和-route)
  - [6.3. 限速](#63-限速)
  - [6.4. JWT身份验证](#64-jwt身份验证)

# 1. 简介
Kong 是由 Mashape 公司开源的云原生、高性能、可扩展的微服务 API 网关。它基于 OpenResty 实现，使用 Cassandra 或 PostgreSQL 存储数据。

除了免费的开源版本，Mashape 还提供了付费的企业版，其中包括技术支持、使用培训服务以及 API 分析插件。

# 2. 架构
## 2.1. 特性
![kong架构](../media/kong-texing01.png)  
* Kong RESTful 管理 API，提供了 API、API消费者、插件、upstreams、证书等管理。
* Kong 插件拦截请求/响应，类似于 Java Servlet 中的过滤器，实现请求/响应的 AOP 处理。
* Kong 数据存储用于存储 Kong 集群节点信息、API、消费者、插件等信息，目前提供了 Cassandra 和 PostgreSQL 支持。如果需要高可用，建议使用 Cassandra。
* Kong 集群中的节点通过 gossip 协议自动发现其它节点。当通过一个 Kong 节点的管理 API 进行一些变更时，也会通知其他节点。每个 Kong 节点的配置信息是会缓存的，例如插件，那么当在某一个 Kong 节点修改了插件配置时，需要通知其他节点配置的变更。
* Kong 核心基于 OpenResty，使用 Lua 进行请求/响应的处理。

## 2.2. 组件
![kong组件图](../media/kong-jiagou01.png)  
Kong 是在客户端和（微）服务间转发API通信的API网关，通过插件扩展功能。  
Kong 主要组件：
- Kong Server ：基于nginx的服务器，用来接收 API 请求。
- Apache Cassandra：用来存储操作数据。
- kong dashboard: konga

Kong 中通过插件扩展已有功能，这些插件在 API 请求响应循环的生命周期中被执行，插件使用 Lua 编写。  
Kong还有如下几个基础功能：
- HTTP 基本认证
- 密钥认证
- CORS（ Cross-origin Resource Sharing，跨域资源共享）
- TCP、UDP请求转发
- API 请求限流
- 文件日志
- nginx 监控


# 3. 插件
Kong 内置了丰富的插件，提供了强大的功能以及集成能力，同时我们又可以基于插件机制自行进行拓展。  
![插件](../media/plugins01.png)
* Authentication 身份认证插件：Kong 提供了 Basic Authentication、Key authentication、OAuth2.0 authentication、HMAC authentication、JWT、LDAP authentication 等等实现。
* Security 安全控制插件：ACL（访问控制）、CORS（跨域资源共享）、动态SSL、IP 限制、爬虫检测等等实现。
* Traffic Control 流量控制插件：请求限流（基于请求计数限流）、上游响应限流（根据 upstream 响应计数限流）、请求大小限制等等实现。限流支持本地、Redis 和集群三种限流模式。
* Analytics & Monitoring 分析监控插件：对接 Datadog、Prometheus、Zipkin 等等监控系统的实现。
* Transformations 协议转换插件：请求转换（在转发到 upstream 之前修改请求）、响应转换（在 upstream 响应返回给客户端之前修改响应）。
* Logging 日志应用插件：支持 TCP、UDP、HTTP、File、Syslog、StatsD、Loggly 等等方式传输日志。
* Serverless 插件：提供对 AWS Lambda、Azure Functions、Apache OpenWhisk、Kong 自带 Serverless Functions 等等的 Serverless 解决方案的支持。
* Deployment 插件

# 4. 请求流程
![request](../media/request01.png)  
* 当 Kong 运行时，每个对 API 的请求将先被 Kong 命中，然后这个请求将会被代理转发到最终的 API 接口。
* 在请求（Requests）和响应（Responses）之间，Kong 将会执行已经事先安装和配置好的插件，授权 API 访问操作。

# 5. 基于postgresql的一键安装
参考文档 [kong一键安装](kong一键安装.md)

# 6. kong简单使用
## 6.1. login
* 使用浏览器，访问 <http://IP:1337/> 地址，进入 Konga 首页
* 输入「账号/密码」登陆 Konga。添加 Kong 的 Admin URL 地址

## 6.2. 负载均衡配置
负载到 192.168.15.228:10001 和 192.168.15.228:10002  
### 6.2.1. 创建upstream
```
curl -X POST http://127.0.0.1:8001/upstreams --data "name=demo-upstream"
curl -X POST http://127.0.0.1:8001/upstreams/demo-upstream/targets --data "target=192.168.15.228:10001" --data "weight=100"
curl -X POST http://127.0.0.1:8001/upstreams/demo-upstream/targets --data "target=192.168.15.228:10002" --data "weight=100"
```
 如上的配置，效果等同于如下 Nginx 的配置
 ```
 upstream demo-upstream {
    server 192.168.15.228:10001 weight=100;
    server 192.168.15.228:10002 weight=100;
}
```

### 6.2.2. 创建 service 和 route
```
curl -X POST http://127.0.0.1:8001/services --data "name=demo-service" --data "host=demo-upstream"
curl -X POST http://localhost:8001/services/demo-service/routes --data "name=demo-route" --data "paths[]=/demo-api"
```
如上的配置，效果等同于如下 Nginx 的配置
```
server {
    listen  80;
    location /demo-api/ {
        proxy_pass http://demo-upstream;
    }
}
```
整个流程如下图：  
![upstream](../media/upstream01.png)


## 6.3. 限速
```
curl -X POST http://127.0.0.1:8001/services/demo-service/plugins \
    --data "name=rate-limiting"  \
    --data "config.second=1" \
    --data "config.limit_by=ip
```
* name 参数，设置为 rate-limiting 表示使用 Rate Limiting 插件。
* config.second 参数，设置为 1 表示每秒允许 1 次请求。
* config.limit_by 参数，设置为 ip 表示使用 IP 基础维度的限流。  
  
Rate Limiting 插件还支持给 route、consumer 设置限流配置

## 6.4. JWT身份验证
```
curl -X POST http://127.0.0.1:8001/services/demo-service/plugins \
    --data "name=jwt"
# 再使用 curl http://127.0.0.1:8000/demo-api/ 会被拦截
# 创建用户
curl -i -X POST http://localhost:8001/consumers/ \
    --data "username=yujing"

curl -i -X POST http://localhost:8001/consumers/yujing/jwt
```
![jwt](../media/jwt01.png)  
用Encoded生成的值去访问：  
```
curl http://127.0.0.1:8000/demo-api/ \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJMbFpvNkdBUmYzaDZIaW1xMHRHZ0xsQnNPR3U4YUh3dyJ9.G5TapzuSDLBsjWvRu74KAn7s_YPX5wAjCuILO7FEdZE" 

```