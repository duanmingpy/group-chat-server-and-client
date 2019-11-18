# 群聊客户端-服务器实现  
使用python来编写群聊服务器和客户端。 use python to code a group-chat-server and a group-chat-client.

## 使用python来实现服务端 
功能：  function:
  1. 能够同时连接n个客户端；  a group-chat-server could link many clients;
  2. 能够实现即时通讯；  could make it that many clients communicate at same time;
  3. 客户端必须定时给服务端发送心跳包；  a client must send heat-beat each period;
  4. 如果超时没有发送心跳包将会被移除；   if client heat-beat is timeout, it will be out;
  5. 客户端可以主动退出，不影响其他客户端。  a client can quit anytime, don't have influence for other clients.
  
```python   
import socketserver
import datetime
import threading


class MyHandle(socketserver.BaseRequestHandler):  # 每一个线程就一个实例
    def self_server_init(self):
        if not hasattr(self.server, "clients"):
            setattr(self.server, "clients", {})  # 如果没有这个属性就给他增加一个
        if not hasattr(self.server, "_lock_clients"):
            setattr(self.server, "lock_clients", threading.Lock())
        self.server.__dict__["hb_interval"] = 10
        self.server.clients[self.key] = datetime.datetime.now().timestamp()  # 记录时间

    def setup(self):
        # self init
        self.event = threading.Event()
        self.key = self.request, self.event
        # self.server init
        self.self_server_init()

    def handle(self):
        no_hb = set()
        while not self.event.is_set():
            data = self.request.recv(1024)
            print(data.decode())
            if data == b"^hb^":   # 收到了心跳包
                with self.server.lock_clients:
                    self.server.clients[self.key] = datetime.datetime.now().timestamp()
            if data == b"" or data.strip() == b"quit":
                with self.server.lock_clients:
                    self.server.clients.pop(self.key, None)  # 如果退出就弹出了
                break  # 因为一个线程是一个连接

            # 如果没有退出我们就刷新时间，也有可能是新的连接
            self.server.clients[self.key] = datetime.datetime.now().timestamp()

            # 发送消息的时候检查是否超时了
            with self.server.lock_clients:
                for key, t in self.server.clients.items():
                    if datetime.datetime.now().timestamp() - t > self.server.hb_interval:  # 如果超时了记录一下这个key
                        no_hb.add(key)  # s和e都可哈希
                        continue
                    key[0].send("from {}: {}".format(self.client_address, data.decode()).encode())
                for key in no_hb:
                    self.server.clients.pop(key)  # 移除没有心跳的客户端
                no_hb.clear()

    def finish(self):
        super().finish()
        self.event.set()


server = socketserver.ThreadingTCPServer(("127.0.0.1", 9999), MyHandle)
server.serve_forever()   # 启动
```   

