用DDD领域建模的方法来构建中台业务模型。你可以选择两种建模策略：**自顶向下**和**自底向上**的策略。



**1. 自顶向下的策略**

这种策略是先做顶层设计，从最高领域逐级分解为中台，分别建立领域模型，根据业务属性分为通用中台或核心中台。领域建模过程主要基于业务现状，暂时不考虑系统现状。自顶向下的策略适用于全新的应用系统建设，或旧系统推倒重建的情况。

由于这种策略不必受限于现有系统，你可以用DDD领域逐级分解的领域建模方法。从下面这张图我们可以看出它的主要步骤：第一步是将领域分解为子域，子域可以分为核心域、通用域和支撑域；第二步是对子域建模，划分领域边界，建立领域模型和限界上下文；第三步则是根据限界上下文进行微服务设计。

![img](img/e665d85381a9b2c599555cac6a06deda.jpg)



**2. 自底向上的策略**

这种策略是基于业务和系统现状完成领域建模。首先分别完成系统所在业务域的领域建模；然后对齐业务域，找出具有同类或相似业务功能的领域模型，对比分析领域模型的差异，重组领域对象，重构领域模型。这个过程会沉淀公共和复用的业务能力，会将分散的业务模型整合。**自底向上策略适用于遗留系统业务模型的演进式重构**。

主要分为三个步骤:

**第一步：锁定系统所在业务域，构建领域模型。**

锁定系统所在的业务域，采用事件风暴，找出领域对象，构建聚合，划分限界上下文，建立领域模型。看一下下面这张图，我们选取了传统核心应用的用户、客户、传统收付和承保四个业务域以及互联网电商业务域，共计五个业务域来完成领域建模。

![img](img/f537a7a43e77212c8a85241439b2f246.jpg)



**第二步：对齐业务域，构建中台业务模型。**

![img](img/25cd1e7fe14bfa22a752c1b184b9c91d.jpg)



**第三步：中台归类，根据领域模型设计微服务。**

完成中台业务建模后，我们就有了下面这张图。可以看到总共构建了多少个中台，中台下面有哪些领域模型，哪些中台是通用中台，哪些中台是核心中台，中台的基本信息等等，都一目了然。中台下的领域模型就可以设计微服务了。

![img](img/a88e9695c7198a1f88f537564ada0bc5.jpg)