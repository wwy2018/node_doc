# Process
`process`全局对象提供当前nodejs进程的信息并进行管控。它是一个EventEmitter实例。
> 事件：beforeExit

当nodejs的事件循环已清空并且没有新的工作安排时，会触发beforeExit事件，通常，没有工作安排时nodejs进程就会退出，但是beforeexit事件的监听能够调用异步操作，进而让进程持续。<br>
监听回调会被传递的唯一参数process.exitCode的值触发<br>
使用process.exit()或者未捕获异常引起的显示终止将不会触发beforeexit事件<br>
beforeexit不能作为exit事件的替换，除非是为了安排额外的工作
> 事件：disconnect

如果进程是通过IPC通道创建的，IPC通道关闭时就会触发disconnect事件
> 事件：exit

两种情况会触发exit事件：
* 明确调用process.exit()方法
* 事件循环没有额外的工作安排

这时没有方法能阻止退出，node进程也会随之终止<br>
监听回调的code参数可以通过process.exitCode指定也可以通过参数形式传递
```
process.on('exit', (code) => {
  console.log(`About to exit with code: ${code}`);
});
```
'exit' 事件监听器的回调函数，只允许包含同步操作。所有监听器的回调函数被调用后，任何在事件循环数组中排队的工作都会被强制丢弃，然后 Nodje.js 进程会立即结束。 例如在下例中，定时器中的操作永远不会被执行（因为不是同步操作）
```
process.on('exit', (code) => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```
> 事件：message

如果进程是通过IPC通道创建的，每次父进程通过childprocess.send()发送的消息被子进程接收到的时候都会触发message事件<br>
监听回调会被一下参数触发：
* `message`(object): 一个被解析之后的json对象或者一个原始值
* `sendHandle:  <net.Server> | <net.Socket> a net.Server or net.Socket对象或undefined`

信息经历序列化和解析，结果信息可能和初始传递的信息不同
> 事件：rejectionHandled

如果一个promise绑定了错误处理函数（promise.catch())，而且被rejected，就会触发rejectionHandled事件<br>
监听接收被拒绝的promise的引用为唯一参数<br>
```
const unhandledRejections = new Map();
process.on('unhandledRejection', (reason, p) => {
  unhandledRejections.set(p, reason);
});
process.on('rejectionHandled', (p) => {
  unhandledRejections.delete(p);
});
```
> 事件：uncaughtException

未捕获的js异常会触发此事件并导致进程终止
```
process.on('uncaughtException', (err) => {
  fs.writeSync(1, `Caught exception: ${err}\n`);
});

setTimeout(() => {
  console.log('This will still run.');
}, 500);

// Intentionally cause an exception, but don't catch it.
nonexistentFunc();
console.log('This will not run.');
```
注意：正确使用 'uncaughtException' 事件的方式，是用它在进程结束前执行一些已分配资源（比如文件描述符，句柄等等）的同步清理操作。 触发 'uncaughtException' 事件后，用它来尝试恢复应用正常运行的操作是不安全的。
> 事件：unhandledRejection

用于跟踪没有绑定错误处理器的promise，promise被rejected时触发，参数有：
* `reason <Error> | <any>` 此对象包含了 promise 被 rejected 的相关信息（通常是一个 Error 对象）。
* p 被 rejected 的 promise 对象。

```
process.on('unhandledRejection', (reason, p) => {
  console.log('Unhandled Rejection at:', p, 'reason:', reason);
  // application specific logging, throwing an error, or other logic here
});

somePromise.then((res) => {
  return reportToUser(JSON.pasre(res)); // note the typo (`pasre`)
}); // no `.catch` or `.then`
```
如下代码也会触发'unhandledRejection'事件
```
function SomeResource() {
  // Initially set the loaded status to a rejected promise
  this.loaded = Promise.reject(new Error('Resource not yet loaded!'));
}

const resource = new SomeResource();
// no .catch or .then on resource.loaded for at least a turn
```
> 事件：warning

```
process.on('warning', (warning) => {
  console.warn(warning.name);    // Print the warning name
  console.warn(warning.message); // Print the warning message
  console.warn(warning.stack);   // Print the stack trace
});
```
> 自定义警告

查看 process.emitWarning()
> 信号事件

进程接收到信号时触发，处理器第一个参数是信号名称（大写）
```
// Begin reading from stdin so the process does not exit.
process.stdin.resume();

process.on('SIGINT', () => {
  console.log('Received SIGINT. Press Control-D to exit.');
});

// Using a single function to handle multiple signals
function handle(signal) {
  console.log(`Received ${signal}`);
}

process.on('SIGINT', handle);
process.on('SIGTERM', handle);
```
> process.abort()

立即结束进程并生成core文件
> process.arch

返回编译nodejs二进制的系统cpu架构名称：'arm', 'arm64', 'ia32', 'mips', 'mipsel', 'ppc', 'ppc64', 's390', 's390x', 'x32', and 'x64'
> process.argv

返回进程参数，第一个参数是当前目录
```
$ node process-args.js one two=three four
```
> process.argv0
> process.channel

如果Node.js进程是由IPC channel(请看 Child Process 文档) 方式创建的，process.channel属性保存IPC channel的引用。 如果IPC channel不存在，此属性值为undefined。
> process.chdir(directory)

process.chdir()方法变更Node.js进程的当前工作目录，如果变更目录失败会抛出异常(例如，如果指定的目录不存在)。
```
console.log(`Starting directory: ${process.cwd()}`);
try {
  process.chdir('/tmp');
  console.log(`New directory: ${process.cwd()}`);
} catch (err) {
  console.error(`chdir: ${err}`);
}
```
> process.config
> process.connected

如果Node.js进程是由IPC channel方式创建的， 只要IPC channel保持连接，process.connected属性就会返回true。<br> process.disconnect()被调用后，此属性会返回false。<br>
process.connected如果为false，则不能通过IPC channel使用process.send()发送信息。
> process.cpuUsage([previousValue])
* `previousValue <Object>` 上一次调用此方法的返回值
* 返回: <Object>
 - user <integer> 微秒
 - system <integer> 微秒

返回当前进程使用的用户和系统cpu时间，多核情况下可能比实际时间长
```
const startUsage = process.cpuUsage();
// { user: 38579, system: 6986 }

// spin the CPU for 500 milliseconds
const now = Date.now();
while (Date.now() - now < 500);

console.log(process.cpuUsage(startUsage));
// { user: 514883, system: 11226 }
```
> process.cwd()

当前工作目录
> process.debugPort

调试端口
```
process.debugPort = 5858;
```
> process.disconnect()

如果 Node.js 进程是从IPC频道派生出来的（具体看 Child Process 和 Cluster 的文档）, process.disconnect()函数会关闭到父进程的IPC频道，以允许子进程一旦没有其他链接来保持活跃就优雅地关闭。

调用process.disconnect()的效果和父进程调用ChildProcess.disconnect()的一样ChildProcess.disconnect().

如果 Node.js 进程不是从IPC频道派生出来的，那调用process.disconnect()函数的结果是undefined.
> process.dlopen(module, filename[, flags])

动态加载，优先使用require()
> process.emitWarning(warning[, options])
> process.emitWarning(warning[, type[, code]][, ctor])
> process.env
> process.execArgv
> process.execPath
> process.exit([code])
> process.getegid()
> process.geteuid()
> process.getgroups()
> process.getuid()
> process.hasUncaughtExceptionCaptureCallback()
> process.hrtime([time])
> process.initgroups(user, extraGroup)
> process.kill(pid[, signal])
> process.mainModule
> process.memoryUsage()
> process.nextTick(callback[, ...args])
> process.noDeprecation
> process.pid
> process.platform
> process.ppid
> process.release
> process.send(message[, sendHandle[, options]][, callback])
> process.setegid(id)
> process.seteuid(id)
> process.setgid(id)
> process.setgroups(groups)
> process.setuid(id)
> process.setUncaughtExceptionCaptureCallback(fn)
> process.stderr
> process.stdin
> process.stdout

