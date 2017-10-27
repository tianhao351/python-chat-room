## 聊天室程序需求

我们要实现的是简单的聊天室的例子，就是允许多个人同时一起聊天，每个人发送的消息所有人都能接收到，类似于 QQ 群的功能，而不是点对点的 QQ 好友之间的聊天。

我们要实现的有两部分：

- **Chat Server**：聊天服务器，负责与用户建立 Socket 连接，并将某个用户发送的消息广播到所有在线的用户。
- **Telnet Client**：用户聊天客户端，可以输入聊天的内容并发送，同时可以显示其他用户的消息记录。

同样，我们的消息通信采用 TCP 连接保证可靠性。在分别对服务端和客户端进行程序设计之前，首先要学习一下 Python 中实现异步 I/O 的一个函数 —— `select`。

## Python 异步 I/O

Python 在 `select` 模块中提供了异步 I/O（Asynchronous I/O），这与 Linux 下的 select 机制相似，但进行一些简化。我首先介绍一下 `select`，然后告诉你在 Python 中如何使用它。

前面[文章](http://blog.codingcorner.cn/python/2014/12/11/python_socket_programming)使用多线程来并行处理多路 socket I/O，这里介绍的`select` 方法允许你响应不同 socket 的多个事件以及其它不同事件。例如你可以让 `select` 在某个 socket 有数据到达时，或者当某个 socket 可以写数据时，又或者是当某个 socket 发生错误时通知你，好处是你可以同时响应很多 socket 的多个事件。

Linux 下 C 语言的 `select` 使用到位图来表示我们要关注哪些文件描述符的事件，Python 中使用 list 来表示我们监控的文件描述符，当有事件到达时，返回的也是文件描述符的 list，表示这些文件有事件到达。下面的简单程序是表示等待从标准输入中获得输入：

```python
rlist, wlist, elist = select.select( [sys.stdin], [], [] )

print sys.stdin.read()
```

`select` 方法的三个参数都是 list 类型，分别代表读事件、写事件、错误事件，同样方法返回值也是三个 list，包含的是哪些事件（读、写、异常）满足了。上面的例子，由于参数只有一个事件 `sys.stdin`，表示只关心标准输入事件，因此当 `select` 返回时 rlist 只会是 `[sys.stdin]`，表示可以从 stdin 中读入数据了，我们使用 `read` 方法来读入数据。

当然 `select` 对于 socket 描述符也是有效的，下面的一个例子是创建了两个 socket 客户端连接到远程服务器，`select` 用来监控哪个 socket 有数据到达：

```python
import socket
import select

sock1 = socket.socket( socket.AF_INET, socket.SOCK_STREAM )
sock2 = socket.socket( socket.AF_INET, socket.SOCK_STREAM )

sock1.connect( ('192.168.1.1', 25) )
sock2.connect( ('192.168.1.1', 25) )

while 1:

    # Await a read event
    rlist, wlist, elist = select.select( [sock1, sock2], [], [], 5 )

    # Test for timeout
    if [rlist, wlist, elist] == [ [], [], [] ]:
        print "Five seconds elapsed.\n"

    else:
        # Loop through each socket in rlist, read and print the available data
        for sock in rlist:
            print sock.recv( 100 )
```

好了，有了上面的基础，我们就可以来设计聊天室的服务器和客户端了。

## 聊天室服务器

聊天室服务器主要完成下面两件事：

- 接收多个客户端的连接
- 从每个客户端读入消息病广播到其它连接的客户端

我们定义一个 list 型变量 `CONNECTION_LIST` 表示监听多个 socket 事件的可读事件，那么利用上面介绍的我们的服务器使用 `select` 来处理多路复用 I/O 的代码如下：

```python
# Get the list sockets which are ready to be read through select
read_sockets,write_sockets,error_sockets = select.select(CONNECTION_LIST,[],[])
```

当 `select` 返回时，说明在 `read_sockets` 上有可读的数据，这里又分为两种情况：

1. 如果是主 socket（即服务器开始创建的 socket，一直处于监听状态）有数据可读，表示有新的连接请求可以接收，此时需要调用 `accept` 函数来接收新的客户端连接，并将其连接信息广播到其它客户端。
2. 如果是其它 sockets（即与客户端已经建立连接的 sockets）有数据可读，那么表示客户端发送消息到服务器端，使用 `recv` 函数读消息，并将消息转发到其它所有连接的客户端。

上面两种情况到涉及到广播消息的过程，广播也就是将从某个 socket 获得的消息通过 `CONNECTION_LIST` 的每个 socket （除了自身和主 socket）一个个发送出去：

```python
def broadcast_data (sock, message):
    #Do not send the message to master socket and the client who has send us the message
    for socket in CONNECTION_LIST:
        if socket != server_socket and socket != sock :
            try :
                socket.send(message)
            except :
                # broken socket connection may be, chat client pressed ctrl+c for example
                socket.close()
                CONNECTION_LIST.remove(socket)
```

如果发送失败，我们假设某个客户端已经断开了连接，关闭该 socket 病将其从连接列表中删除。

完整的聊天室服务器源代码如下：

```python
# Tcp Chat server
 
import socket, select
 
#Function to broadcast chat messages to all connected clients
def broadcast_data (sock, message):
    #Do not send the message to master socket and the client who has send us the message
    for socket in CONNECTION_LIST:
        if socket != server_socket and socket != sock :
            try :
                socket.send(message)
            except :
                # broken socket connection may be, chat client pressed ctrl+c for example
                socket.close()
                CONNECTION_LIST.remove(socket)
 
if __name__ == "__main__":
     
    # List to keep track of socket descriptors
    CONNECTION_LIST = []
    RECV_BUFFER = 4096 # Advisable to keep it as an exponent of 2
    PORT = 5000
     
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # this has no effect, why ?
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind(("0.0.0.0", PORT))
    server_socket.listen(10)
 
    # Add server socket to the list of readable connections
    CONNECTION_LIST.append(server_socket)
 
    print "Chat server started on port " + str(PORT)
 
    while 1:
        # Get the list sockets which are ready to be read through select
        read_sockets,write_sockets,error_sockets = select.select(CONNECTION_LIST,[],[])
 
        for sock in read_sockets:
            #New connection
            if sock == server_socket:
                # Handle the case in which there is a new connection recieved through server_socket
                sockfd, addr = server_socket.accept()
                CONNECTION_LIST.append(sockfd)
                print "Client (%s, %s) connected" % addr
                 
                broadcast_data(sockfd, "[%s:%s] entered room\n" % addr)
             
            #Some incoming message from a client
            else:
                # Data recieved from client, process it
                try:
                    #In Windows, sometimes when a TCP program closes abruptly,
                    # a "Connection reset by peer" exception will be thrown
                    data = sock.recv(RECV_BUFFER)
                    if data:
                        broadcast_data(sock, "\r" + '<' + str(sock.getpeername()) + '> ' + data)                
                 
                except:
                    broadcast_data(sock, "Client (%s, %s) is offline" % addr)
                    print "Client (%s, %s) is offline" % addr
                    sock.close()
                    CONNECTION_LIST.remove(sock)
                    continue
     
    server_socket.close()
```

在控制台下运行该程序：

```python
$ python chat_server.py 
Chat server started on port 5000
```

## 聊天室客户端

我们写一个客户端程序可以连接到上面的服务器，完成发送消息和接收消息的过程。主要做下面两件事：

- 监听服务器是否有消息发送过来
- 检查用户的输入，如果用户输入某条消息，需要发送到服务器

这里有两个 I/O 事件需要监听：连接到服务器的 socket 和标准输入，同样我们可以使用 `select` 来完成：

```python
rlist = [sys.stdin, s]
         
# Get the list sockets which are readable
read_list, write_list, error_list = select.select(rlist , [], [])
```

那逻辑就很简单了，如果是 `sys.stdin` 有数据可读，表示用户从控制台输入数据并按下回车，那么就从标准输入读数据，并发送到服务器；如果是与服务器连接的 socket 有数据可读，表示服务器发送消息给该客户端，那么就从 socket 接收数据。加上一些提示信息及异常处理的完整客户端代码如下：

```python
# telnet program example
import socket, select, string, sys
 
def prompt() :
    sys.stdout.write('<You> ')
    sys.stdout.flush()
 
#main function
if __name__ == "__main__":
     
    if(len(sys.argv) < 3) :
        print 'Usage : python telnet.py hostname port'
        sys.exit()
     
    host = sys.argv[1]
    port = int(sys.argv[2])
     
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(2)
     
    # connect to remote host
    try :
        s.connect((host, port))
    except :
        print 'Unable to connect'
        sys.exit()
     
    print 'Connected to remote host. Start sending messages'
    prompt()
     
    while 1:
        rlist = [sys.stdin, s]
         
        # Get the list sockets which are readable
        read_list, write_list, error_list = select.select(rlist , [], [])
         
        for sock in read_list:
            #incoming message from remote server
            if sock == s:
                data = sock.recv(4096)
                if not data :
                    print '\nDisconnected from chat server'
                    sys.exit()
                else :
                    #print data
                    sys.stdout.write(data)
                    prompt()
             
            #user entered a message
            else :
                msg = sys.stdin.readline()
                s.send(msg)
                prompt()
```

可以在多个终端下运行该代码：

```shell
$ python telnet.py localhost 5000
Connected to remote host. Start sending messages
<You> hello
<You> I am fine
<('127.0.0.1', 38378)> ok good
<You>
```

在另一个终端显示的信息：

```shell
<You> [127.0.0.1:39339] entered room
<('127.0.0.1', 39339)> hello
<('127.0.0.1', 39339)> I am fine
<You> ok good
```

## 总结

上面的代码注意两点：

1. 聊天室客户端代码不能在 windows 下运行，因为代码使用 `select` 同时监听 socket 和输入流，在 Windows 下 `select` 函数是由 WinSock 库提供，不能处理不是由 WinSock 定义的文件描述符。
2. 客户端代码还有个缺陷是，当某个客户端在输入消息但还未发送出去时，服务器也发送消息过来，这样会冲刷掉客户端正在输入的消息。这目前来看没办法解决的，唯一的解决方法是使用像 ncurses 终端库使用户输入和输出独立开，或者写一个 GUI 的程序。

那么本文通过一个聊天室的范例进一步学习了 Python 下 Socket 编程。