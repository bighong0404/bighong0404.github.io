> 转自:
>
> https://juejin.cn/post/6844903625513238541



# TCP 建立连接 - 三次握手

TCP 三次握手就好比两个人在街上隔着50米看见了对方，但是因为雾霾等原因不能100%确认，所以要通过招手的方式相互确定对方是否认识自己。



![img](img/1643a1dd6df4813b)



张三首先向李四招手(**syn**)，李四看到张三向自己招手后，向对方点了点头挤出了一个微笑(**ack**)。张三看到李四微笑后确认了李四成功辨认出了自己(进入**estalished**状态)。

但是李四还有点狐疑，向四周看了一看，有没有可能张三是在看别人呢，他也需要确认一下。所以李四也向张三招了招手(**syn**)，张三看到李四向自己招手后知道对方是在寻求自己的确认，于是也点了点头挤出了微笑(**ack**)，李四看到对方的微笑后确认了张三就是在向自己打招呼(进入**established**状态)。

于是两人加快步伐，走到了一起，相互拥抱。



![img](img/1643a1f3fa6c21b0)



我们看到这个过程中一共是四个动作，张三招手 -- 李四点头+微笑 -- 李四招手 -- 张三点头微笑。其中李四连续进行了2个动作，先是点头微笑(回复对方)，然后再次招手(寻求确认)，实际上可以将这两个动作合一，招手的同时点头和微笑(**syn+ack**)。于是四个动作就简化成了三个动作，张三招手--李四点头微笑并招手--张三点头微笑。这就是**三次握手的本质，中间的一次动作是两个动作的合并**。

我们看到有两个中间状态，**syn_sent**和**syn_rcvd**，这两个状态叫着「半打开」状态，就是向对方招手了，但是还没来得及看到对方的点头微笑。**syn_sent**是主动打开方的「半打开」状态，**syn_rcvd**是被动打开方的「半打开」状态。客户端是主动打开方，服务器是被动打开方。

- syn_sent: syn package has been sent
- syn_rcvd: syn package has been received

# TCP 数据传输

TCP 数据传输就是两个人隔空对话，差了一点距离，所以需要对方反复确认听见了自己的话。



![img](img/1643a1f92f5af34a)



张三喊了一句话(data)，李四听见了之后要向张三回复自己听见了(ack)。

如果张三喊了一句，半天没听到李四回复，张三就认为自己的话被大风吹走了，李四没听见，所以需要重新喊话，这就是tcp重传。

也有可能是李四听到了张三的话，但是李四向张三的回复被大风吹走了，以至于张三没听见李四的回复。张三并不能判断究竟是自己的话被大风吹走了还是李四的回复被大风吹走了，张三也不用管，重传一下就是。

既然会重传，李四就有可能同一句话听见了两次，这就是「去重」。「重传」和「去重」工作操作系统的网络内核模块都已经帮我们处理好了，用户层是不用关心的。



![img](img/1643a1fc9435605c)



张三可以向李四喊话，同样李四也可以向张三喊话，因为tcp链接是「双工的」，双方都可以主动发起数据传输。不过无论是哪方喊话，都需要收到对方的确认才能认为对方收到了自己的喊话。

张三可能是个高射炮，一说连说了八句话，这时候李四可以不用一句一句回复，而是连续听了这八句话之后，一起向对方回复说前面你说的八句话我都听见了，这就是批量ack。但是张三也不能一次性说了太多话，李四的脑子短时间可能无法消化太多，两人之间需要有协商好的合适的发送和接受速率，这个就是「TCP窗口大小」。

网络环境的数据交互同人类之间的对话还要复杂一些，它存在数据包乱序的现象。同一个来源发出来的不同数据包在「网际路由」上可能会走过不同的路径，最终达到同一个地方时，顺序就不一样了。操作系统的网络内核模块会负责对数据包进行排序，到用户层时顺序就已经完全一致了。



# TCP 断开连接 - 四次挥手

TCP断开链接的过程和建立链接的过程比较类似，只不过中间的两部并不总是会合成一步走，所以它分成了4个动作，张三挥手(fin)——李四伤感地微笑(ack)——李四挥手(fin)——张三伤感地微笑(ack)。



![img](img/1643a20296de1ff0)



之所以中间的两个动作没有合并，是因为tcp存在「半关闭」状态，也就是单向关闭。张三已经挥了手，可是人还没有走，只是不再说话，但是耳朵还是可以继续听，李四呢继续喊话。等待李四累了，也不再说话了，朝张三挥了挥手，张三伤感地微笑了一下，才彻底结束了。



![img](img/1643b1147fbbc5e7)



上面有一个非常特殊的状态`time_wait`，它是主动关闭的一方在回复完对方的挥手后进入的一个长期状态，这个状态标准的持续时间是4分钟，4分钟后才会进入到closed状态，释放套接字资源。不过在具体实现上这个时间是可以调整的。

它就好比主动分手方要承担的责任，是你提出的要分手，你得付出代价。这个后果就是持续4分钟的`time_wait`状态，不能释放套接字资源(端口)，就好比守寡期，这段时间内套接字资源(端口)不得回收利用。

它的作用是重传最后一个ack报文，确保对方可以收到。因为如果对方没有收到ack的话，会重传fin报文，处于time_wait状态的套接字会立即向对方重发ack报文。

同时在这段时间内，该链接在对话期间于网际路由上产生的残留报文(因为路径过于崎岖，数据报文走的时间太长，重传的报文都收到了，原始报文还在路上)传过来时，都会被立即丢弃掉。**4分钟的时间足以使得这些残留报文彻底消逝**。不然当新的端口被重复利用时，这些残留报文可能会干扰新的链接。

4分钟就是2个MSL，每个MSL是2分钟。MSL就是`maximium segment lifetime`——最长报文寿命。这个时间是由官方RFC协议规定的。至于为什么是2个MSL而不是1个MSL，我还没有看到一个非常满意的解释。

四次挥手也并不总是四次挥手，中间的两个动作有时候是可以合并一起进行的，这个时候就成了三次挥手，主动关闭方就会从`fin_wait_1`状态直接进入到`time_wait`状态，跳过了`fin_wait_2`状态。



# 拓展

## 1. TCP建立连接为什么是三次握手, 而不是2次或者4次? 

​		假设两次握手就ok了，要是客户端第一次握手过去，结果卡在某个地方了，没到服务端；客户端等待超时后再次重试发送了第一次握手过去，服务端收到了回复第二次握手，ok两次握手建立连接了。

​		这时候，那个卡着的老的第一次握手发到了服务器，服务器直接就返回一个第二次握手，这个时候服务器开辟了资源准备客户端发送数据啥的. 但结果呢？客户端不会理睬这个发回来的二次握手(异常的数据包会被丢弃.)，因为之前都通信过了。 

​		但是如果是三次握手，那个二次握手发回去，客户端发现根本不对，就会回复个**复位报文**过去，让服务器撤销开辟的资源。

​		因此, 三次握手实际上是**互相发送请求, 要求回复确认**的具有**容错性**的过程. 

​		而再多次握手, 则纯粹浪费资源了. 



## 2. 为什么关闭连接是四次挥手而不是三次呢?

​		因为服务器端的SOCKET当收到`SYN`报文建连接请求后，它可以把ACK和SYN(ACK起应答作用，而SYN起同步作用)放在一个报文时发送。

​		但是关闭连接场景，当服务端收到对应`FIN`报文通知时，它仅仅表示客户端没有数据发送给你了，但服务端有可能数据还没全部发送给客户端了，因此服务端不能马上关闭SOCKET，需要数据全部客户端后再发送FIN报文给对方表示同间现在可以关闭连接了，所以这里的ACK报文和FIN报文多数情况下都是分开发送的, 不会跟建立连接一样, 把`ACK`+`FIN`合并一起回复.



## 3. 为什么`TIME_WAIT`状态还需要等`2MSL`后才能返回到`CLOSED`状态？

​		因为虽然服务器同意关闭连接了，而且挥手的4个报文也都协调和发送完毕，按理可以直接回到`CLOSED`状态（就好比建立连接时候, 收到`ACK`报文后, 从`SYN_SEND`状态到`ESTABLISHED`状态）. 但是因为网络是不可靠的，客户端无法保证最后发送的`ACK`报文一定被服务器收到，因此服务器处于`LAST_ACK`状态下的SOCKET可能会**因为超时未收到`ACK`报文**，而**重发`FIN`报文**，所以客户端这个`TIME_WAIT`状态的作用就是用来等待服务器可能重发丢失的`ACK`报文. 

