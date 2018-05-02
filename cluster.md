# Cluster
单个nodejs实例运行在单个线程上，多核系统里，有时需要创建node进程集群处理负载任务。cluster模块允许轻松创建共享服务器端口的子进程
```
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  console.log(`Master ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`worker ${worker.process.pid} died`);
  });
} else {
  // Workers can share any TCP connection
  // In this case it is an HTTP server
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');
  }).listen(8000);

  console.log(`Worker ${process.pid} started`);
}
```
请注意,在Windows中,还不能在工作进程中设置命名的管道(Pipe)服务器。
# 工作方式
工作进程通过child_process.fork()创建，从而通过IPC和父进程进行通信，从而使各进程交替处理连接服务。

cluster模块支持两种分配连接方式

第一种（除了windows平台）循环式，主进程监听一个端口，接收新连接后再将连接循环分发给工作进程，在分发中使用了一些内置技巧防止工作进程任务过载。

第二种方式主进程创建一个监听socket，把它发送给感兴趣的进程，之后进程直接处理新的连接。

理论上第二种方法应该是效率最佳的，但在实际情况下，由于操作系统调度机制的难以捉摸，会使分发变得不稳定。我们遇到过这种情况：8个进程中的2个，分担了70%的负载。

由于server.listen()将大部分工作分配给主进程，普通的node进程和cluster作业进程产生区别的三种情况如下：
* `server.listen({fd: 7})`由于消息传递到主进程，文件描述符7会在主进程中被监听，将文件句柄（handle）传递给工作进程，而不是文件描述符“7”本身。
* `server.listen(handle)`进程明确监听具柄会导致进程直接使用该具柄，而不是和主进程通信。
* `server.listen(0)`通常这会导致服务器监听随机的端口，但在cluster模式中，所有工作进程每次调用listen(0)时会收到相同的“随机”端口。实质上，这种端口只在第一次分配时随机，之后就变得可预料。如果要使用独立端口的话，应该根据工作进程的ID来生成端口号。

注意：Node.js不支持路由逻辑。因此在设计应用时，不应该过分依赖内存数据对象（如sessions和login等）。
