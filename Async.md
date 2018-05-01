# Async Hooks
> 稳定性：1 - 实验中
`async_hooks`模块提供一个api注册回调，跟踪nodejs应用中创建的异步资源生命周期，可通过如下方式调用：
```
const async_hooks = require('async_hooks');
```
## 术语
一个异步资源代表一个具有回调的对象，回调函数可以多次调用，例如net.createServer的`connection`事件，或者只调用一次，例如fs.open。一个资源可以在回调调用之前关闭，异步钩子不会准确区分这些不同情况，但是会代表资源的抽象概念。
## 公共接口
## 概览
如下是公共接口的简单概要
```
const async_hooks = require('async_hooks');

// 返回当前执行上下文的id
const eid = async_hooks.executionAsyncId();

// 返回前执行作用域将要调用的回调函数的触发手柄id
const tid = async_hooks.triggerAsyncId();

// 创建一个新的异步钩子实例，所有的回调都是可选项
const asyncHook =
    async_hooks.createHook({ init, before, after, destroy, promiseResolve });

// 使异步钩子的所有回调都能够被调用。运行构造函数之后必须指明，才能开始执行回调
asyncHook.enable();

// 禁止监听新的异步事件
asyncHook.disable();

// 下面是能够传入到createHook()的回调函数
// 对象构造时调用init函数，init回调运行时资源可能没有完成构建，因此asyncId指向的区域可能没有完全激活
function init(asyncId, type, triggerAsyncId, resource) { }

// before在资源回调函数调用前调用，它可以被处理器(e.g. TCPWrap)调用0-n次，但是只能被请求方(e.g. FSReqWrap)调用一次
function before(asyncId) { }

// 资源回调执行完成之后调用after
function after(asyncId) { }

// 当AsyncWrap实例销毁时调用destroy
function destroy(asyncId) { }

// promiseResolve只能被promise资源调用，当`resolve`函数传递到`Promise`构造函数时被触发
function promiseResolve(asyncId) { }
```
### async_hooks.createHook(callbacks)
* `callbacks <对象> 要注册的回调钩子`
- init <函数> The init callback.
- before <函数> The before callback.
- after <函数> The after callback.
- destroy <函数> The destroy callback.
* 返回: <AsyncHook> Instance used for disabling and enabling hooks
---
为每个异步操作的生命中不同事件注册对应的执行函数<br>
回调函数：init()/before()/after()/destroy()在资源异步事件不同阶段分别调用<br>
所有的回调都是可选的，例如，如果只跟踪资源的清理，仅需要传递destroy回调
```
const async_hooks = require('async_hooks');

const asyncHook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId, resource) { },
  destroy(asyncId) { }
});
```
注意回调可以通过原型链继承
```
class MyAsyncCallbacks {
  init(asyncId, type, triggerAsyncId, resource) { }
  destroy(asyncId) {}
}

class MyAddedCallbacks extends MyAsyncCallbacks {
  before(asyncId) { }
  after(asyncId) { }
}

const asyncHook = async_hooks.createHook(new MyAddedCallbacks());
```
### 错误处理
任何回调钩子异常，应用都会打印堆叠跟踪并退出。虽然退出路径会遵循未获取的异常，但是所有的uncaughtException监听都会被移除，因此强迫进程退出。除非进程开启了`--abort-on-uncaught-exception`，`exit`回调仍然会被调用。<br>
造成这种情况的原因是这些回调函数运行在对象潜在的多个点，例如类的构造和销毁，因此，立即结束进程能阻止意外麻烦。
### 打印异步钩子的回调函数
由于打印是异步操作，console.log()会引起异步勾子回调函数的调用，在异步钩子回调函数的内部使用console.log()及类似的异步操作会引起无限递归。简单的解决办法是使用同步打印操作，例如fs.writeSync(1, msg)
```
const fs = require('fs');
const util = require('util');

function debug(...args) {
  // use a function like this one when debugging inside an AsyncHooks callback
  fs.writeSync(1, `${util.format(...args)}\n`);
}
```
假如一个异步操作需要被记录，可以通过使用异步钩子本身提供的信息跟踪此异步操作是由什么引起的。如果是记录本身引起了异步钩子回调的调用则记录会跳过，避免可能的无限递归
### asyncHook.enable()
* `返回: <AsyncHook> 一个异步钩子的引用`
---
启用指定的异步钩子实例的回调，如果没有回调则启用无效<br>
默认回调钩子实例被禁止，使用如下方式可以让实例立即生效。
```
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook(callbacks).enable();
```
### asyncHook.disable()
* `返回: <AsyncHook> 一个异步钩子的引用`
---
从全局等待被执行的异步勾子回调池中禁用指定的异步勾子实例的回调，一旦钩子禁用将不会被调用直到被启用
## 钩子回调函数
异步事件的关键事件被划分到四个区域：实例开始，回调调用之前／之后，实例销毁时
### init(asyncId, type, triggerAsyncId, resource)
* `asyncId <number> 异步资源的独特id`
* `type <string> 异步资源类型`
* `triggerAsyncId <number> 当前创建的异步资源所依托的执行上下文的那个异步资源的独特id`
* `resource <Object> 异步操作所引用的资源，销毁时需要释放`
---
当一个类创建时可能触发异步事件时调用，不代表实例必须在销毁之前调用`before/after`，只是存在这种可能<br>
此行为可以通过开启一个资源并在执行前随即关闭来进行观察：
```
require('net').createServer().listen(function() { this.close(); });
// OR
clearTimeout(setTimeout(() => {}, 10));
```
每个新的资源都会在当前进程作用域中分配一个id
#### type
type是一个字符串，标记引起调用init的资源类型，通常是资源构造函数的名称
```
FSEVENTWRAP, FSREQWRAP, GETADDRINFOREQWRAP, GETNAMEINFOREQWRAP, HTTPPARSER,
JSSTREAM, PIPECONNECTWRAP, PIPEWRAP, PROCESSWRAP, QUERYWRAP, SHUTDOWNWRAP,
SIGNALWRAP, STATWATCHER, TCPCONNECTWRAP, TCPSERVER, TCPWRAP, TIMERWRAP, TTYWRAP,
UDPSENDWRAP, UDPWRAP, WRITEWRAP, ZLIB, SSLCONNECTION, PBKDF2REQUEST,
RANDOMBYTESREQUEST, TLSWRAP, Timeout, Immediate, TickObject
```
也有`PROMISE`资源类型，用于跟踪promise实例和其异步工程<br>
使用公共内嵌api时，可以自定义type<br>
注意：可能会有名称雷同，所以鼓励加前缀
#### triggerId
triggerAsyncId是引起（或触发）这个新资源初始化并使init被调用的那个资源的asyncId，和async_hooks.executionAsyncId()是不同的，后者只表现了资源是什么时间创建的，而前者能显示资源为什么创建：
```
async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = async_hooks.executionAsyncId();
    fs.writeSync(
      1, `${type}(${asyncId}): trigger: ${triggerAsyncId} execution: ${eid}\n`);
  }
}).enable();

require('net').createServer((conn) => {}).listen(8080);
```
当输入命令nc localhost 8080会输出：
```
TCPSERVERWRAP(2): trigger: 1 execution: 1
TCPWRAP(4): trigger: 2 execution: 0
```
TCPSERVERWRAP是接收连接的服务器<br>
TCPWRAP是从客户端生成的新的连接，新连接创建时TCPWRAP会立即创建，这发生在js栈之外（executionAsyncId为0表示从c++执行）。triggerAsyncId负责分配负责新资源存在的宿主资源
#### resource
`resource`代表实际初始化的异步资源，类型不同，包含的信息也不同。
#### Asynchronous context example
见官方文档


