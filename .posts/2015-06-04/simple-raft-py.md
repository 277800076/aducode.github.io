﻿<!--{layout:default title:自己动手实现RAFT算法}-->
前段时间学习了一下分布式系统的[raft算法][1]，相比较Paxos协议，理解起来确实容易多了，于是就产生了自己动手实现一套基于raft一致性协议的分布式缓存的想法，经过大约两个月的空闲时间，终于完成了一个可以运行的python版本[aducode/simple-raft-py][2]
 
### 模块划分
项目主要包括如下几个模块：
**1. server**：

* main loop所在，在main loop中处理IO事件和超时事件（参考redis的实现）
<pre class="language-python line-numbers"><code>
def server_forever(self):
        """
        启动服务
        :return:
        """
        # init the server runtime env
        self.initialise()
        # Main Loop
        while self.inputs:
            # handle io
            self.handle_io_event()
            # handle timer
            self.handle_timeout_event()
            if not self.is_running():
                # stoppde or stopping
                break
        # release the resources
        self.realease()
        print 'Server stopped!'
</code></pre>
*  网络模型采用IO多路复用模型，使用python的select模块，水平触发level-triggered(也叫条件触发)
*  将socket连接封装成channel，channel中进行输入输出数据格式化处理，同时多个channel组成链式结构，按顺序格式化数据
<pre class="language-python line-numbers"><code>
class Channel(object):
    def __init__(self, server, client, _next):
        self.server = server
        self.client = client
        self.next = _next

    def input(self, data, recv):
        """
        :param data 数据
        :param recv 是否从socket接受来的消息
                    当为False时，说明数据是发送到其他server的，不是server接受来的
        """
        pass

    def output(self):
        pass

    def close(self):
        """
        说明socket被关闭，传递到handler
        """
        return self.next.close()
</code></pre>
*  handler进行真正的逻辑处理，位于channel链的最末端
<pre class="language-python line-numbers"><code>
class Handler(object):
    def handle(self, server, client, request):
        pass

    def close(self, server, client):
        """
        handler处理close
        :return:
        """
        pass
</code></pre>
*  server提供两个最基本的channel：1 IO2Channel 作为channel链的head，负责调用socket.recv接收数据和socket.send发送数据；2 Channel2Handler 作为channel链的tail与handler连接，负责调用handler.handle方法并将返回结果返回到前一个channel
<pre class="language-python line-numbers"><code>
class IO2Channel(Channel):
    """
    直接与IO关联的Channel
    """

    def __init__(self, server, client, _next):
        super(IO2Channel, self).__init__(server, client, _next)

    def input(self, data=None, recv=True):
        client_close = False
        try:
            if recv:
                request = self.client.recv(1024)
            else:
                request = data
        except Exception, e:
            client_close = True
        else:
            if request:
                self.next.input(request, recv)
            else:
                client_close = True
        if client_close:
            self.server.close(self.client)

    def output(self):
        response, end = self.next.output()
        if response:
            try:
                self.client.send(response)
            except (IOError, socket.error):
                self.server.close(self.client)
        if (not response or end) and self.client in self.server.outputs:
            self.server.outputs.remove(self.client)


class Channel2Handler(Channel):
    """
    与handler关联起来
    """

    def __init__(self, server, client, _next):
        super(Channel2Handler, self).__init__(server, client, _next)
        self.queue = Queue.Queue()

    def input(self, data, recv):
        response = None
        if recv:
            if isinstance(self.next, Handler):
                response = self.next.handle(self.server, self.client, data)
            elif isinstance(self.next, types.FunctionType):
                response = self.next(self.server, self.client, data)
        else:
            # 说明直接发出去
            response = data
        if response:
            self.queue.put(response)
            if self.client not in self.server.outputs:
                self.server.outputs.append(self.client)

    def output(self):
        try:
            return self.queue.get_nowait(), self.queue.empty()
        except Queue.Empty:
            return None, True

    def close(self):
        return self.next.close(self.server, self.client)
</code></pre>


**2. protocol**

* 主要负责分布式节点间通信的message定义与解析
*  消息主要分为ClientMessage & NodeMessage
*  其中ClientMessage为客户端发来的操作，如set get del commit rollback
*  NodeMessage为分布式节点之间传递的消息，如选举消息、心跳消息等


**3.db**

* 主要负责数据存储
* 定义一个db engine接口，用来处理client操作，将来只要修改配置文件，就可以选择不同的存储引擎了（参考mysql）
*  目前只有一个简单的db engine实现，所有数据保存在python dict中

**4.node & state**

* node定义了一个逻辑上的节点，以(host, port)作为node的唯一标识
* node中实现了MessageChannel用来序列化/反序列化，将str格式的message转换成ClientMessage/NodeMessage
*  node中实现了NodeHandler，用来路由不同类型的Message
* state用来定义节点状态，主要状态有Leader、Follower、Candidate
<pre class="language-python line-numbers"><code>
class State(object):
    """
    节点状态类
    不同状态有不同的表现
    """

    def __init__(self, node):
        self.node = node

    def handle(self, client, message):
        """
        处理其他node的消息
        """
        assert isinstance(message, Message)
        if isinstance(message, ClientMessage):
            if message.op == 'get':
                return self.on_get(client, message)
            else:
                return self.on_update(client, message)
        elif isinstance(message, ClientCloseMessage):
            return self.on_client_close(client, message)
        elif isinstance(message, HeartbeatRequestMessage):
            return self.on_heartbeat_request(client, message)
        elif isinstance(message, HeartbeatResponseMessage):
            return self.on_heartbeat_response(client, message)
        elif isinstance(message, ElectRequestMessage):
            return self.on_elect_request(client, message)
        elif isinstance(message, ElectResponseMessage):
            return self.on_elect_response(client, message)
        elif isinstance(message, SyncRequestMessage):
            return self.on_sync_request(client, message)
        elif isinstance(message, SyncResponseMessage):
            return self.on_sync_response(client, message)
        else:
            pass

    def on_get(self, client, message):
        """
        处理客户端get请求
        """
        return self.node.config.db.handle(client, message.op, message.key, message.value, message.auto_commit)

    def on_update(self, client, message):
        """
        处理客户端除了get外的请求
        """
        return '@%s:%d@redirect' % self.node.leader if self.node.leader \
            else 'No Leader Elected, please wait until we have a leader...'

    def on_client_close(self, client, message):
        """
        处理客户端关闭
        """
        return self.node.config.db.release(message.client)

    def on_heartbeat_request(self, client, message):
        """
        接收到heartbeat请求的处理方法
        """
        pass

    def on_heartbeat_response(self, client, message):
        """
        接收到heartbeata响应的处理方法
        """
        pass

    def on_elect_request(self, client, message):
        """
        接收到选举请求的处理方法
        """
        pass

    def on_elect_response(self, client, message):
        """
        接收到选举响应的处理方法
        """
        pass

    def on_sync_request(self, client, message):
        """
        接收到同步请求
        """
        pass

    def on_sync_response(self, client, message):
        """
        接收同步响应
        """
        pass

    def __str__(self):
        return self.__class__.__name__.strip().upper()
</code></pre>

### 目前实现的功能
* 多节点选举Leader
* Leader节点下线重选
* Leader发送心跳，根据响应维护follower存活列表
* Node之间数据同步
* 简单数据事物commit rollback

### 不足和想法
* 只是用来学习raft的一个小项目，性能肯定不足，扛不住高并发的请求
* 节点之间的Request & Response 基本上都是异步处理的，异步编程的代码风格对人不是很友好，好多on_xxxx方法，许多代码逻辑是割裂开的（考虑用python中的yield实现协程把代码风格编程同步风格）
*  异步的逻辑导致调试起来比较痛苦
* 本来以为很简单的功能，没想到用了两个月，才实现了最最最基础的逻辑，从中我除了学到了raft一致性协议的原理以外，还对IO多路复用更加的熟练了。

----------------------------------------
[1]: http://thesecretlivesofdata.com/raft/
[2]: https://github.com/aducode/simple-raft-py
