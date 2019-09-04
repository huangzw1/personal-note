## kong学习笔记
---
### 基础
- kong是一个云原生、高效、可扩展的分布式api网关
- 最核心的对象：
1. upstream是对上游服务器的抽象；
2. target代表一个物理服务，是ip+port的抽象；
3. service是抽象层面的服务，可以直接映射到一个物理服务（host指向ip+port），也可以指向一个upstream来做loadbalancer，在一个服务中无法同时存在http和https，如果上游提供http和https服务，同时需要Kong代理，那必须要设置两个服务
4. route是路由的抽象，负责将实际的请求映射到service
- 对应关系：
1. upstream和target：1对n
2. service和upstream：1对1或1对0（service也可以指向具体的target，相当于不做负载均衡）
3. service和route：1对n
4. Routes specify how(and if)requests are sent to their Services after they reach Kong.A single Service can have many Routes.
- By default Kong listens on the following ports:

`:8000 on which Kong listens for incoming HTTP traffic from your clients, and forwards it to your upstream services.`监听http请求，并将其转发到上游服务

`:8443 on which Kong listens for incoming HTTPS traffic. This port has a similar behavior as the :8000 port, except that it expects HTTPS traffic only. This port can be disabled via the configuration file.`监听https请求，作用同8000，仅协议不同，可在配置中禁用

`:8001 on which the Admin API used to configure Kong listens.`

`:8444 on which the Admin API listens for HTTPS traffic.`
### Kong配置
- 通常配置文件在/etc/kong/kong.conf
- Kong也会读取环境变量，所以也可以通过环境变量进行配置，形式：`KONG_XXXX`,XXXX为配置的名称，对应kong.conf
- 调整nginx配置:向kong.conf加入配置项如`nginx_http_`,`nginx_proxy_`,or`nginx_admin_`
- 也可以通过创建一个nginx配置文件，再在文件中指定，如`nginx_http_include = /path/to/your/my-server.kong.conf`，或者通过环境变量配置`export KONG_NGINX_HTTP_INCLUDE="/path/to/your/my-server.kong.conf"`

### 部署
- 常规部署

- 集群化部署(因没有多台机器，故此处测试采用docker)
1. need a load-balancer in front of Kong cluster to distribute traffic across your available nodes.
2. a Kong cluster means that those nodes will share the same configuration
### 插件
- 对于部分插件，影响范围可能是整个kong服务，但大多数插件都是装在service或者route上，这使得插件的影响范围可以被控制。在多数情况下，我们可能只需要对核心接口进行限流控制，只需要对部分接口进行权限控制，这时只对特定的service和route进行定向的配置即可。
- 例子：
#### 服务
1. 为hello服务添加50次/秒的限流：
```
curl -X POST http://localhost:8001/services/hello/plugins \
--data "name=rate-limiting" \
--data "config.second=50"
```
2. 为hello服务添加jwt插件：
```
curl -X POST http://localhost:8001/services/login/plugins \
--data "name=jwt"
```
3. 插件也可以安装在route上：
```
curl -X POST http://localhost:8001/routes/{routeId}/plugins \
--data "name=rate-limiting" \
--data "config.second=50"

curl -X POST http://localhost:8001/routes/{routeId}/plugins \
--data "name=jwt"
```
4. 更新服务：
```
curl -s -X PATCH --url http://192.168.0.184:8001/services/linuxops_server \
-d 'name=linuxops_server_patch' \
-d 'protocol=http' \
-d 'host=www.baidu.com' \
 | python -m json.tool
 #修改服务名称，但服务id不变
```
5. 更新或创建服务：
```
curl -s -X PUT --url http://192.168.0.184:8001/services/linuxops_server_put \
-d 'name=linuxops_server_patch' \
-d 'protocol=http' \
-d 'host=www.baidu.com' \
 | python -m json.tool
 #若服务存在，则更新，若服务不存在，则创建
 ```
 6. 删除服务：
 ```
 curl -i  -X DELETE --url http://192.168.0.184:8001/services/b6094754-07da-4c31-bb95-0a7caf5e6c0b
 ```
 #### 路由
 1. 查看路由
 ```
 curl -s http://192.168.0.184:8001/routes/6c6b7863-9a05-4d51-bf7e-8e4e5866a131 | python -m json.tool
 ```
 2. 添加路由
 ```
 curl -s -X POST --url http://192.168.0.184:8001/routes \
-d 'protocols=http' \
-d 'methods=GET'  \
-d 'paths=/weather' \
-d 'service.id=43921b23-65fc-4722-a4e0-99bf84e26593' \
| python -m json.tool
#上面的路由关联了一个服务，methods、hosts、paths必须有一个才能创建路由
 ```
 3. 更新路由
 ```
 curl -s -X PATCH --url http://192.168.0.184:8001/routes/6c6b7863-9a05-4d51-bf7e-8e4e5866a131 \
-d 'protocols=http' \
-d 'methods=GET'  \
-d 'paths=/weather' \
-d 'service.id=43921b23-65fc-4722-a4e0-99bf84e26593' \
| python -m json.tool
#路由没有name字段，所有都以id匹配
 ```
 4. 更新或添加路由
 ```
 curl -s -X PUT --url http://192.168.0.184:8001/routes/6c6b7863-9a05-4d51-bf7e-2962c1d6b0e6 \
-d 'protocols=http' \
-d 'methods=GET'  \
-d 'paths=/weather' \
-d 'service.id=43921b23-65fc-4722-a4e0-99bf84e26593' \
| python -m json.tool
 ```
 5. 查看和服务关联的路由
 ```
 curl -s http://192.168.0.184:8001/services/wechat/routes | python -m json.tool
 ```
 6. 查看和路由关联的服务
 ```
 curl -s http://192.168.0.184:8001/routes/f8ef8876-9681-4629-a2ee-d7fac8a8094a/service | python -m json.tool
 ```
 7. 删除路由
 ```
 curl -s http://192.168.0.184:8001/routes/f8ef8876-9681-4629-a2ee-d7fac8a8094a/service | python -m json.tool
 ```
- HMAC:
```
#Enable the plugin on service
$ curl -X POST http://kong:8001/services/{service}/plugins \
    --data "name=hmac-auth" 
#Enable the plugin on route
$ curl -X POST http://kong:8001/routes/{route}/plugins \
    --data "name=hmac-auth"
#Usage
#Create a Consumer
$ curl -d "username=user123&custom_id=SOME_CUSTOM_ID" http://kong:8001/consumers/

#Create a Credential,a consumer can have many credentials
$ curl -X POST http://kong:8001/consumers/{consumer}/hmac-auth \
    --data "username=bob" \
    --data "secret=secret456"
    
#
```
- JWT:
```

```
- OAUTH2:
```

```

- kong从1.1版本开始支持声明式配置（通过yaml或json）以及无数据库配置
### postgresql增删改查
```
\l  #查看已有数据库
\c database_name #进入指定数据库
```




- 附：也可使用konga作为kong的gui管理界面，则不需要通过命令行管理，（容器部署）参考：https://juejin.im/post/5d09c307e51d4510a73280c4
