# Nameko-web
python微服务框架

### Python 微服务框架 Nmaeko

#### 1. What?

 		让程序员关注应用逻辑个测试的微服务框架

#### 安装

```
pip install nameko
```

安装nameko需要的依赖

```
sudo apt install rabbitmq-server
```

#### hello world!

#### 服务端

```
# helloworld.py

from nameko.rpc import rpc

class GreetingService:
	name = "greeting_service"# 自定义服务名称
	
	@rpc #入口标记
	def hello(self, name):
		return "Hello,{}!".format(name)
	
```

运行服务端

```
nameko run helloworld
```

正常输出

```/
starting service: greeting_service
Connted to amqp://guest:**@127.0.0.1:5672//
```

#### 进入命令行客户端

```
nameko shell 
```

#### 发起客户端请求

```
n.rpc.greeting_service.hello(name='world')
```

输出

```
'Hello, world!'
```

## 2. 核心

### 服务

​	一个服务就是一个Python类，这个类吧服务逻辑封装到方法中，而且把任何的依赖都作为方法的参数。

### 进入点

​	进入点可以简单理解为@rpc标记的方法所关联的服务入口，在方法上使用了@rpc修饰的方法都就是将暴露给外部业务。

这些进入点会监视外部事件， 将触发进入点修饰的方法执行并返回结果.

### 依赖

​	官方提出所有非服务核心逻辑的实现最好都以依赖的形式实现

​	依赖其实是隐藏代码的一种很好的方式

​	使用依赖时应该把所有的依赖都进行声明	

### 工作器

​	工作器就是进入点发放被触发的时候差生的微服务实例，但是如果有依赖，那么就会被依赖的实例代替。

​	一个工作器实例就只处理以此请求

​	一个服务可以同时运行多个工作器，但最多只能是用户预定义的并发值。

### 依赖注入

​	服务类的依赖添加是声明式的，声明时不是使用接口，而是通过使用参数进行声明。

​	这个参数是一个DependencyProvider。这个东西负责提供注入到服务工作器的对象

​	所有的provider都要提供get_dependency()方法生产要注入到工作器中的对象。

​	工作器生命周期:

```
1. 进入点触发
2. 通过服务类初始化工作器
3. 依赖注入到工作器
4. 执行方法
5. 工作器销毁
```

依赖提供者在服务的持续时间内存活， 而注入的依赖项对每个工作器来说都是唯一的。

### 同步

Nameko基于eventlet库， 这个库实现的同步模型是基于隐式yield模式的协程，通过“绿色的线程”提供同步功能.

隐式的yield基于monkey pathing 基础库。当一个线程等待IO时就会触发yield. 通过 nameko run 启的服务将会应用者个模式。

每一个工作器都有自己的线程，最大的同步工作器数量可以基于每个工作器等待Io时间的总量来动态调整。

工作器都是无状态的所以线程安全。但是 外部依赖应该 确保他们每个工作线程都是用同一个依赖或者多个工作器都能安全地同步访问。

然而c扩展系都使用socket通信， 他们通常都不认为绿色线程(协程)的工作能满足线程安全。其中就包括 

librabbitmq, MySQLdb等 。

### 扩展

所以有的入口点和依赖提供者都作为“扩展”实现。 因为他们存在于服务代码之外，又不是所有服务都需要的。

### 运行服务

运行服务需要的所有东西： 服务类和有关的配置

最简单的运行一个或多个服务的方法是使用Nameko命令行：

​	运行某module下的所有服务或者运行某module下的特定的ServiceClass服务	

```
nameko run module:[ServiceClass] 
```

### 服务容器

每个服务类都委托给一个ServiceContainer.这个容器封装了所有需要运行一个服务的方法，而且装载了在服务类上的任何扩展。

使用ServiceContainer运行单个服务：

```
from nameko.containers import ServiceContainer

class Service:
	name = "service"
	
#create a container 
container = ServiceContainer(Service, config={})

#"container.extensions" exposes all extensions used by the service 
service_extensions = list(container.extensions)

# start service
container.start()

# stop service 
container.stop()
```

### 服务运行器

ServiceRunner 是多个服务容器的简单包装，同时提供启动和停止所有包装容器的方法。 是nameko run内部使用的方法，但也可以实现程序化控制。

```
from nameko.runners import ServiceRunner
from nameko.testing.utils import get_container

class ServiceA:
	name = "service_a"
	
class ServiceB:
	name = "sevice_b"
	
#create a runner for ServiceA and ServiceB
runner = ServiceRunner(config={})
runner.add_service(ServiceA)
runner.add_service(ServiceB)

# "get_container" will return the container for a particular service 
container_a = get_container(runner, ServiceA)

# start both servies
runner.start()

# stop both services
runner.stop()

```

### 3. 命令行接口

​	Nameko 提供了一个命令行接口尽可能方便地托管服务及与服务进行交互。

### 运行服务

```
nameko run <module>[:<ServiceClass>]

```

发现并运行服务类， 这个命令将会在前台启动服务并且运行到进程终止

也可以用-config选项重写默认配置，并提供一个Yaml格式的配置文件

```
nameko run --config ./foobar.yaml <module>[:<ServiceClass>]

```

```
# foobar.yaml 

AMQP_URI: 'pyamqp://guest:guest@localhost'
WEB_SERVER_ADDRESS: '0.0.0.0:8000'
rpc_exchange: 'nameko-rpc'
max_workers: 10
parent_calls_tracked: 10

LOGGING:
	version: 1
	handler: 
		console:
			class: logging.StreamHandler
	root:
		level: DEBUG
		handler: [console]

```

LOGGING 项会传递到logging.config.dictConfig(), 而且会适应调用的样式

配置值可以通过内建的Config依赖提供器读取

### 环境变量解决方案

YAML配置文件为环境变量提供了基本的支持。 可以使用bash风格的语法： ${ENV_VAR}, 另外还可以提供默认值${ENV_VAR:default_value}

```
# foobar.yaml
AMQP_URI: pyamqp://${RABBITMQ_USER:guest}:${RABBITMQ_PASSWORD: password}@${RABBITMQ_HOST:localhost}


```

使用环境变量的运行方式示例

```
$ RABBITMQ_USER=user RABBITMQ_PASSWORD=password RABBITMQ_HOST=host nameko run --config ./foobar.yaml <module>[:<ServiceClass>]


```

如果需要在YAML文件里使用引号(引号里使用环境变量), 显式声明！ env_var处理器是必须的

```
$ RABBITMQ_USER=user RABBITMQ_PASSWORD=password RABBITMQ_HOST=host nameko run --config ./foobar.yaml <module>[:<ServiceClass>]


```

与运行的服务进行交互

```
nameko shell	

```

上述命令是为了与远程服务工作，运行了一个交互式python脚本环境。 这是规范的交互解释器，提供了一个添加到了内建命名空间的特殊的模块n，以支持RPC调用和分发事件

### 发起RPC到目标服务

```
nameko shell 
>>> n.rpc.target_service.target_method(...)
# RPC response

```

### 作为源服务分发事件	

```
nameko shell 
>>> n.dispatch_event("source_service", "event_type", "event_payload")

```

### 4. 内建的扩展

#### RPC

​	Nameko包含了一个基于AMQP的RPC实现。包括@rpc入口点， 一个与其他区啊服务对话的代理， 以及一个非RPC调用到集群的独立的代理

```
from nameko.rpc import rpc, RpcRroxy

class ServiceY:
	name = "sercvice_y"
	
	@rpc
	def append_identifier(self, value):
		return "{}-y".format(value)
		
class ServiceX:
	name = "service_x"
	
	y = RpcProxy("service_y")
	
	@rpc
	def remote_method(self,value):
		res = '{}-x'.format(value)
		return self.y.append_identifier(res)
		
from nameko.standlone.rpc import ClusterProxy

config = { 
	"AMQP_URI": AMQP_URI #e.g. "pyamqp://guest:guest@localhost"
}

with ClusterRpcProxy(config) as cluster_rpc:
	cluster_rpc.service_x.remote_method("hello") #"hello-x-y"


```

一般的RPC调用会一直阻塞直到远程方法完成为止，但是代理也提供了一个异步调用模式到后台或者并行化RPC调用.

```
with ClusterRpcProxy(config) as cluster_rpc:
	hello_res = cluster_rpc.service_x.remote_method.call_async("hello")
	world_res = cluster_rpc.service_x.remote_method.call_async("world")
	# do work while waiting 
	hello_res.result() #hello-x-y 
	world_res.result() #world-x-y
​		

```

### 事件(发布订阅)

Nameko事件是一个异步的消息系统，实现了发布订阅模式，。 服务分发事件， 而这些事件可以被0到多个其他服务接收。

```
from nameko.events import EventDispatcher, event_handler
from nameko.rpc import rpc

class ServiceA:
    """Event dispatching service"""
    name = "service_a"

    dispatch = EventDispatcher()

    @rpc
    def dispatching_method(self,payload):

        self.dispatch('event_type', payload)


class ServiceB:
    """Event listening service."""
    name = "service_b"

    @event_handler("service_a",'SERVICE_POOL')
    def handle_event(self,payload):
        print("service b recevied:", payload)

```

eventhandler进入点有是哪个处理器类型决定事件消息是如何被一个集群接收的：

​	SERVICE_POOL:  所有的事件处理器通过服务名称联合在一起，并且每个池中的只有一个实例接收事件，类似于RPC进入点的集群行为。这是默认的处理类型。

​	BROADCAST： 每个监听服务实例都会被接受到事件。

​	SIGNALETON： 只有一个监听服务实例会接受到事件。

### HTTP  GET  & POST

Nameko的HTTP进入点支持简单的GET和POST

```
# http.py
import json
from nameko.web.handlers import http


class HttpService:
    name = "http_service"

    @http('GET', "/get/<int:value>")
    def get_methods(self, request, value):
        return json.dumps({'values': value})

    @http('POST', '/post')
    def do_post(self, request):
        return "received:{}".format(request.get_data(as_text=True))

```

启动服务： 

```
nameko run http
starting services: http_service

```

测试服务

```
$ curl -i localhost:8000/get/1000
HTTHTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 13
Date: Fri, 13 Feb 2015 14:51:18 GMT

{'value': 1000}

```

```
$ curl -i -d "post body" localhost:8000/post
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 19
Date: Fri, 13 Feb 2015 14:55:01 GMT

received: post body

```



HTTP进入点是基于werkzeug库的， 服务方法必须返回以下的任意值:

```
一个字符串，将变成响应实体response body
一个二元组，(status code, response body)
一个三元组，(status_code, headers dict, response body)
一个werkzeug.wrappers.Response实例

```

```
class HttpServiceOnes:
    name = 'http_services'


    @http('GET', '/priviledged')
    def forbidden(self, request):
        return 403,'Forbidden'

    @http('GET', '/headers')
    def redirect(self, request):
        return 201, {'Location': 'http://www.example.com/widget/1'}, " "

    @http("GET",'/custom')
    def custom(self,request):
        return Response("payload")

```

可以重写responser_from_exception()方法格式化控制的错误返回(服务器异常控制)

```
import json 
from nameko.web.handlers import HttpRequestHandler
from werkzeug.wrappers import Response
from nameko.exceptions import safe_for_serialization

class HttpError(Exception):
	error_code = "BAD_REQUEST"
	status_code = 400
	
	
class  InvalidArgumentsError(HttpError):
	error_code = 'INVALID_ARGUMENTS'
	
# 异常处理
class HttpEntrypoint(HttpRequestHandler):
	def response_from_exception(self,exc):
		if isinstance(exc, HttpError):
			response = Response(
			json.dumps({
				'error': exc.error_code,
				'message': safe_for_serialization(exc),
			}),
				status = exc.status_code,
				mimetype = 'application/json'
			)
				return response
		 return HttpRequestHandler.response_from_exveption(self,exc)
http = HttpEntrypoint.decorator


class Service:
	name = "http_service"
	
	# 抛出异常 
	@http('GET', '/custom_exception')
	def custom_exceptions(self, request):
		raise InvalidArgumentsError("Argument 'foo' is required")
	

```

结果：

```
curl -i http://localhost:8000/custom_exception
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json
Content-Length: 71
Date: Wed, 18 Sep 2019 06:34:18 GMT

{"error": "INVALID_ARGUMENTS", "message": "Argument 'foo' is required"}

```

### 计时器

计时器是一个美达到可配置的秒数时刻就触发的简单的入口点，计时器是非集群定制的，而且在所有的服务实例都会触发。

```
from nameko.timer import timer 

# 启动服务后打印,计时执行ping(),打印pong
class Service:
	name = "service"
	
	@timer(interval=1)
	def ping(self):
		# method executed every second 
		print("pong")

```

### 5. 内建的Dependency Providers

​		Nameko包含了一些常用的Dependency Providers

​		Config 

​		Config 是一个简单的依赖提供器，提供了在运行时只读取配置值的能力

```
from nameko.dependency_providers import Config
from nameko.web.handlers import http 

class Service:
	name = "test_config"
	
	config = Config()
	
	@ property
	def foo_enable(self):
		return self.config.get("FOO_FEATURE_ENABLED", False)
		
	@http('GET', '/foo')
	def foo(self, request):
		if not self.foo_enabled:
			return 403, "FeatureNotEnabled"
		
		return "foo"
		

```

### 6. 社区：

​		扩展：

#### nameko-sqlalchemy

#### nameko-sentry 

#### nameko-anqp-client

#### nameko-bayeux-client

#### nameko-redis-py

#### nameko-redis

#### nameko-statsd

#### 补充库

#### django-nameko

#### flask-nameko

#### nameko-proxy

### 7. 操作数据库 nameko-sqlalchemy

```
from name_sqlalchemy import Database
from .models import Model, DeclarativeBase

class Service(object):
	name = "service"
	
	db = Database(DeclarativeBase)
	
	@entrypoint
	def write_to_db(self):
		model = Model()
		with self.db.get_session() as session:
			session.add(model)
			
	@entrypoint
	def query_db(self):
		queryset = self.db.session.query(Model).filter(...)
		

```

​	  

















































