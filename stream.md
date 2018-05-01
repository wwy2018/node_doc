# Stream
流是nodejs处理流数据的一个抽象接口，steam模块提供一套基础api，使容易构建实现流接口的对象。<br>
nodejs提供很多流对象，例如，http服务器的request，process.stdout

流可读，可写或可读写，所有流都是eventemitter的实例
```
const stream = require('stream');
```
# 流类型
四种基础流类型：
* Readable-流数据可读（例如fs.createReadStream())
* Writable-流数据可写（例如fs.createWriteStream())
* Duplex-可读写（例如net.Socket)
* Transform-可读写并可修改变换（例如zlib.createDeflate())

另外这个模块还有工具函数pipeline和finished
# 对象模式
所有nodejs创建的流对象只能操作字符串和Buffer(Uint8Array)对象，不过也可以以“对象模式”操作其他类型的js值（除了null）

流实例在创建时通过设置objectMode属性可以切换到对象模式。尝试将已创建的流对象切换到对象模式是不安全的。
# 缓冲
