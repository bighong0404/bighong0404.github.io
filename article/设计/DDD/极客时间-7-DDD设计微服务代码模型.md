DDD并没有给出标准的代码模型，不同的人可能会有不同理解。下面要说的这个微服务代码模型是我经过思考和实践后建立起来的，主要考虑的是微服务的边界、分层以及架构演进。



# DDD设计微服务代码模型





## 微服务代码模型

**Interfaces（用户接口层）：**它主要存放用户接口层与前端交互、展现数据相关的代码。前端应用通过这一层的接口，向应用服务获取展现所需的数据。这一层主要用来处理用户发送的Restful请求，解析用户输入的配置文件，并将数据传递给Application层。数据的组装、数据传输格式以及Facade接口等代码都会放在这一层目录里。

**Application（应用层）：**它主要存放应用层服务组合和编排相关的代码。应用服务向下基于微服务内的领域服务或外部微服务的应用服务完成服务的编排和组合，向上为用户接口层提供各种应用数据展现支持服务。应用服务和事件等代码会放在这一层目录里。

**Domain（领域层）：**它主要存放领域层核心业务逻辑相关的代码。领域层可以包含多个聚合代码包，它们共同实现领域模型的核心业务逻辑。聚合以及聚合内的实体、方法、领域服务和事件等代码会放在这一层目录里。

**Infrastructure（基础层）：**它主要存放基础资源服务相关的代码，为其它各层提供的通用技术能力、三方软件包、数据库服务、配置和基础资源服务的代码都会放在这一层目录里。

![img](img/915ad8d830d925a893cd09ff6cbdadb8.jpg)



## 从领域模型到微服务的设计



**服务封装与调用关系**

![img](img/eb626396fcb9f541ec46a799275e04b2-1589375074426.png)



**代码结构**

![img](img/84a486d4c0d9146462b31c7fcd5d835e.png)



注意: 

- 实体采用充血模型