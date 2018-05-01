# c++插件
nodejs插件是用c++编写的动态链接共享对象，可以通过require()函数加载到nodejs，像普通node模块一样使用。插件主要用于为js提供一个能在nodejs和c／c++库之间运行的接口。<br>
目前，插件的实现的过程相当复杂，涉及多个组件和api的知识：
* V8: 目前用于提供js实现的c++库，v8提供了对象创建，函数调用的机制等等，v8的api文档大多在`v8.h`文件中（`deps/v8/include/v8.h`）
* libuv: 实现nodejs事件循环，工作线程及所有异步执行平台的c语言库，它也作为跨平台抽象库，提供简单的类可移植操作系统入口，可以在所有主流操作系统上执行常用系统任务，比如操作文件，套接字，计时器及系统事件。同时还提供了一个抽象多线程系统，可被用于强化更复杂的需要超越标准事件循环的异步插件。建议插件开发者多思考如何通过在 libuv 的非阻塞系统操作、工作线程、或自定义的 libuv 线程中降低工作负载来避免在 I/O 或其他时间密集型任务中阻塞事件循环。
* 内部node库: node本身提供一些可供插件使用的c++api，其中最重要的就是`node::ObjectWrap`
* nodejs包括一些其他的静态链接库，比如OpenSSL，这些库在`deps/` 目录下，仅有libuv,openssl,v8和zlib符号是目的性再输出的，可能用于不同的插件扩展
## hello world插件
这个例子使用c++编写，相当于执行下面这段js代码：
```
module.exports.hello=()=>'world'
```
首先新建`hello.cc`文件
```
#include <node.h>
namespace demo{
  using v8::FunctionCallbackInfo;
  using v8::Isolate;
  using v8::Local;
  using v8::Object;
  using v8::Object;
  using v8::String;
  using v8::Value;
  void Method(const FunctionCallbackInfo<Value>&args){
    Isolate* isolate=args.GetIsolate()
    args.GetReturnValue().Set(String::NewFromUtf8(isolate, "world"))
  }
  void init(Local<Object> exports) {
    NODE_SET_METHOD(exports, "hello", Method)
  }
  NODE_MODULE(NODE_GYP_MODULE_NAME, init)
}
```
所有nodejs插件必须按照如下模式导出初始化函数
```
void Initialize(Local<Object> exports);
NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)
```
`NODE_MODULE`后面没有分号，因为它不是函数<br>
`module_name`必须匹配最终二进制文件名（不包括.node后缀）<br>
这个例子的初始化函数是init，插件模块名称是addon
## 构建
源代码写好后，必须编译成二进制文件addon.node，这需要在项目最上层创建一个文件名为binding.gyp的文件，使用类json格式描述构建配置，这个文件被node-gyp使用，node-gyp是专门用于编译nodejs插件的
```
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "hello.cc" ]
    }
  ]
}
```
node-gyp可以通过npm安装
```
npm install -g node-gyp
```
binding.gyp文件创建之后，使用node-gyp configure生成当前平台下合适的构建文件，在目录`build/`下，unix平台会生成makefile文件，windows平台会生成vcxproj文件<br>
之后 使用node-gyp build生成编译好的`addon.node`文件，这个文件会在`build/release/`目录下<br>
当使用npm install安装一个插件时，npm会使用自带的node-gyp版本执行以上一系列操作<br>
一旦构建完成，可以通过require()调用模块
```
// hello.js
const addon = require('./build/Release/addon');

console.log(addon.hello());
// Prints: 'world'
```
由于插件最终存放的目录依赖于编译方式，有时候可能会在`./build/debug/`目录下面，插件可以使用`binding`包加载编译后的模块<br>
注意虽然`binding`包的实现比定位插件模块更加复杂，使用try-catch非常必要：
```
try {
  return require('./build/Release/addon.node');
} catch (err) {
  return require('./build/Debug/addon.node');
}
```
## 链接到nodejs自身依赖
nodejs使用一些静态链接库，比如v8,libuv,openssl。所有的插件都要求链接到v8也可能链接导其他的依赖。一般像引用`#include<...>`那样简单，node-gyp会自动定位合适的头文件，如下两点需要注意：
* 使用node-gyp时，它会自动检测nodejs版本并下载整个源码包或只下载头文件。如果下载了完整的源代码，则插件对全套的 Node.js 依赖有完全的访问权限。 如果只下载了 Node.js 的文件头，则只有 Node.js 导出的符号可用。
* node-gyp可以使用`--nodedir`命令指向本地nodejs资源镜像，如果使用该选项，则插件有全套依赖的访问权限。
## 使用 require() 加载插件
编译后文件扩展名以`.node`为后缀，require() 函数用于查找具有 .node 文件扩展名的文件，并初始化为动态链接库。<br>
当调用 require() 时，.node 拓展名通常可被省略，Node.js 仍会找到并初始化该插件。 注意，Node.js 会优先尝试查找并加载同名的模块或 JavaScript 文件。 例如，如果与二进制的 addon.node 同一目录下有一个 addon.js 文件，则 require('addon') 会优先加载 addon.js 文件。
## Node.js 的原生抽象
本文档的每个例子都直接使用nodejs和v8api来实现插件。v8api每次版本迭代都会有很大改变，每次改变都可能影响插件的使用，nodejs的发布日程尽量减少这样的更新频率，但是目前nodejs对保证v8api稳定性的作用不大。<br>
[Node.js 的原生抽象（或称为 nan）](https://github.com/nodejs/nan)提供了一组工具，建议插件开发者使用这些工具来保持插件在过往与将来的 V8 和 Node.js 的版本之间的兼容性。 查看 [nan](https://github.com/nodejs/nan/tree/master/examples/) 示例了解它是如何被使用的。
## N-API
> 稳定性：1-试验型

n-api是用于编译原生插件的，它独立于js运行环境（例如：v8），并作为nodejs的部分进行维护，这个api在nodejs各版本间保持应用二进制接口（ABI)稳定，目的是为了让插件与底层js引擎变化相隔离，允许模块不用重新编译就能在nodejs后续版本里面使用。使用n-api创建插件的方式同上。<br>
在之前的hellow world例子中使用n-api，需要替换如下代码，其他保持不变
```
// hello.cc using N-API
#include <node_api.h>

namespace demo {

napi_value Method(napi_env env, napi_callback_info args) {
  napi_value greeting;
  napi_status status;

  status = napi_create_string_utf8(env, "hello", NAPI_AUTO_LENGTH, &greeting);
  if (status != napi_ok) return nullptr;
  return greeting;
}

napi_value init(napi_env env, napi_value exports) {
  napi_status status;
  napi_value fn;

  status = napi_create_function(env, nullptr, 0, Method, nullptr, &fn);
  if (status != napi_ok) return nullptr;

  status = napi_set_named_property(env, exports, "hello", fn);
  if (status != napi_ok) return nullptr;
  return exports;
}

NAPI_MODULE(NODE_GYP_MODULE_NAME, init)

}  // namespace demo
```
在`C/C++ Addons - N-API.`有具体函数和使用方法介绍
## 插件例子
以下插件例子使用v8api，帮助开发者入门，[v8参考](https://v8docs.nodesource.com/)以及[嵌入器指南](https://github.com/v8/v8/wiki/Embedder's%20Guide)<br>
每个示例都使用以下的 binding.gyp 文件：
```
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [ "addon.cc" ]
    }
  ]
}
```
如果有一个以上的 .cc 文件，则只需添加额外的文件名到 sources 数组。 例如：
```
"sources": ["addon.cc", "myexample.cc"]
```
当 binding.gyp 文件准备就绪，则插件示例可以使用 node-gyp 进行配置和构建：
```
$ node-gyp configure build
```
插件通常会开放一些对象和函数，供运行在 Node.js 中的 JavaScript 访问。 当从 JavaScript 调用函数时，输入参数和返回值必须与 C/C++ 代码相互映射。<br>
以下例子描述了如何读取从 JavaScript 传入的函数参数与如何返回结果：
```
// addon.cc
#include <node.h>

namespace demo {

using v8::Exception;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::String;
using v8::Value;

// This is the implementation of the "add" method
// Input arguments are passed using the
// const FunctionCallbackInfo<Value>& args struct
void Add(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  // Check the number of arguments passed.
  if (args.Length() < 2) {
    // Throw an Error that is passed back to JavaScript
    isolate->ThrowException(Exception::TypeError(
        String::NewFromUtf8(isolate, "Wrong number of arguments")));
    return;
  }

  // Check the argument types
  if (!args[0]->IsNumber() || !args[1]->IsNumber()) {
    isolate->ThrowException(Exception::TypeError(
        String::NewFromUtf8(isolate, "Wrong arguments")));
    return;
  }

  // Perform the operation
  double value = args[0]->NumberValue() + args[1]->NumberValue();
  Local<Number> num = Number::New(isolate, value);

  // Set the return value (using the passed in
  // FunctionCallbackInfo<Value>&)
  args.GetReturnValue().Set(num);
}

void Init(Local<Object> exports) {
  NODE_SET_METHOD(exports, "add", Add);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```
编译后，示例插件就可以在 Node.js 中引入和使用：
```
// test.js
const addon = require('./build/Release/addon');

console.log('This should be eight:', addon.add(3, 5));
```
## 回调
把 JavaScript 函数传入到一个 C++ 函数并在那里执行它们，这在插件里是常见的做法。 以下例子描述了如何调用这些回调：
```
// addon.cc
#include <node.h>

namespace demo {

using v8::Function;
using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Null;
using v8::Object;
using v8::String;
using v8::Value;

void RunCallback(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  Local<Function> cb = Local<Function>::Cast(args[0]);
  const unsigned argc = 1;
  Local<Value> argv[argc] = { String::NewFromUtf8(isolate, "hello world") };
  cb->Call(Null(isolate), argc, argv);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", RunCallback);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```
注意，该例子使用了一个带有两个参数的 Init()，它接收完整的 module 对象作为第二个参数。 这使得插件可以用一个单一的函数完全地重写 exports，而不是添加函数作为 exports 的属性。<br>
为了验证它，运行以下 JavaScript：
```
// test.js
const addon = require('./build/Release/addon');

addon((msg) => {
  console.log(msg);
// 打印: 'hello world'
});
```
## 对象工厂
插件可从 C++ 函数中创建并返回新的对象，如以下例子所示。 一个带有 msg 属性的对象被创建并返回，该属性会输出传入 createObject() 的字符串：
```
// addon.cc
#include <node.h>

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  Local<Object> obj = Object::New(isolate);
  obj->Set(String::NewFromUtf8(isolate, "msg"), args[0]->ToString());

  args.GetReturnValue().Set(obj);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", CreateObject);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```
在 JavaScript 中测试它：
```
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon('hello');
const obj2 = addon('world');
console.log(obj1.msg, obj2.msg);
// Prints: 'hello world'
```
## 函数工厂
另一种常见情况是创建 JavaScript 函数来包装 C++ 函数，并返回到 JavaScript
```
// addon.cc
#include <node.h>

namespace demo {

using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void MyFunction(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hello world"));
}

void CreateFunction(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, MyFunction);
  Local<Function> fn = tpl->GetFunction();

  // omit this to make it anonymous
  fn->SetName(String::NewFromUtf8(isolate, "theFunction"));

  args.GetReturnValue().Set(fn);
}

void Init(Local<Object> exports, Local<Object> module) {
  NODE_SET_METHOD(module, "exports", CreateFunction);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)

}  // namespace demo
```
测试它：
```
// test.js
const addon = require('./build/Release/addon');

const fn = addon();
console.log(fn());
// Prints: 'hello world'
```
## 包装 C++ 对象
也可以包装 C++ 对象/类使其可以使用 JavaScript 的 new 操作来创建新的实例：
```
// addon.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Local;
using v8::Object;

void InitAll(Local<Object> exports) {
  MyObject::Init(exports);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, InitAll)

}  // namespace demo
```
然后，在 myobject.h 中，包装类继承自 node::ObjectWrap：
```
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Local<v8::Object> exports);

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void PlusOne(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```
在 myobject.cc 中，实现要被开放的各种方法。 下面，通过把 plusOne() 添加到构造函数的原型来开放它：
```
// myobject.cc
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Local<Object> exports) {
  Isolate* isolate = exports->GetIsolate();

  // Prepare constructor template
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(isolate, "MyObject"));
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  // Prototype
  NODE_SET_PROTOTYPE_METHOD(tpl, "plusOne", PlusOne);

  constructor.Reset(isolate, tpl->GetFunction());
  exports->Set(String::NewFromUtf8(isolate, "MyObject"),
               tpl->GetFunction());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // Invoked as constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ? 0 : args[0]->NumberValue();
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invoked as plain function `MyObject(...)`, turn into construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Context> context = isolate->GetCurrentContext();
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> result =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(result);
  }
}

void MyObject::PlusOne(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj = ObjectWrap::Unwrap<MyObject>(args.Holder());
  obj->value_ += 1;

  args.GetReturnValue().Set(Number::New(isolate, obj->value_));
}

}  // namespace demo
```
要构建这个例子，myobject.cc 文件必须被添加到 binding.gyp：
```
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "addon.cc",
        "myobject.cc"
      ]
    }
  ]
}
```
测试：
```
// test.js
const addon = require('./build/Release/addon');

const obj = new addon.MyObject(10);
console.log(obj.plusOne());
// Prints: 11
console.log(obj.plusOne());
// Prints: 12
console.log(obj.plusOne());
// Prints: 13
```
## 包装对象的工厂
也可以使用一个工厂模式，避免显式地使用 JavaScript 的 new 操作来创建对象实例：
```
const obj = addon.createObject();
// instead of:
// const obj = new addon.Object();
```
首先，在 addon.cc 中实现 createObject() 方法：
```
// addon.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  MyObject::NewInstance(args);
}

void InitAll(Local<Object> exports, Local<Object> module) {
  MyObject::Init(exports->GetIsolate());

  NODE_SET_METHOD(module, "exports", CreateObject);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, InitAll)

}  // namespace demo
```
在 myobject.h 中，添加静态方法 NewInstance() 来处理实例化对象。 这个方法用来代替在 JavaScript 中使用 new：
```
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Isolate* isolate);
  static void NewInstance(const v8::FunctionCallbackInfo<v8::Value>& args);

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static void PlusOne(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```
myobject.cc 中的实现类似与之前的例子：
```
// myobject.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Isolate* isolate) {
  // Prepare constructor template
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(isolate, "MyObject"));
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  // Prototype
  NODE_SET_PROTOTYPE_METHOD(tpl, "plusOne", PlusOne);

  constructor.Reset(isolate, tpl->GetFunction());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // Invoked as constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ? 0 : args[0]->NumberValue();
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invoked as plain function `MyObject(...)`, turn into construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Context> context = isolate->GetCurrentContext();
    Local<Object> instance =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(instance);
  }
}

void MyObject::NewInstance(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  const unsigned argc = 1;
  Local<Value> argv[argc] = { args[0] };
  Local<Function> cons = Local<Function>::New(isolate, constructor);
  Local<Context> context = isolate->GetCurrentContext();
  Local<Object> instance =
      cons->NewInstance(context, argc, argv).ToLocalChecked();

  args.GetReturnValue().Set(instance);
}

void MyObject::PlusOne(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj = ObjectWrap::Unwrap<MyObject>(args.Holder());
  obj->value_ += 1;

  args.GetReturnValue().Set(Number::New(isolate, obj->value_));
}

}  // namespace demo
```
要构建这个例子，myobject.cc 文件必须被添加到 binding.gyp：
```
{
  "targets": [
    {
      "target_name": "addon",
      "sources": [
        "addon.cc",
        "myobject.cc"
      ]
    }
  ]
}
```
测试：
```
// test.js
const createObject = require('./build/Release/addon');

const obj = createObject(10);
console.log(obj.plusOne());
// Prints: 11
console.log(obj.plusOne());
// Prints: 12
console.log(obj.plusOne());
// Prints: 13

const obj2 = createObject(20);
console.log(obj2.plusOne());
// Prints: 21
console.log(obj2.plusOne());
// Prints: 22
console.log(obj2.plusOne());
// Prints: 23
```
## 传递包装的对象
除了包装和返回 C++ 对象，也可以通过使用 Node.js 的辅助函数 node::ObjectWrap::Unwrap 进行去包装来传递包装的对象。 以下例子展示了一个 add() 函数，它可以把两个 MyObject 对象作为输入参数：
```
// addon.cc
#include <node.h>
#include <node_object_wrap.h>
#include "myobject.h"

namespace demo {

using v8::FunctionCallbackInfo;
using v8::Isolate;
using v8::Local;
using v8::Number;
using v8::Object;
using v8::String;
using v8::Value;

void CreateObject(const FunctionCallbackInfo<Value>& args) {
  MyObject::NewInstance(args);
}

void Add(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  MyObject* obj1 = node::ObjectWrap::Unwrap<MyObject>(
      args[0]->ToObject());
  MyObject* obj2 = node::ObjectWrap::Unwrap<MyObject>(
      args[1]->ToObject());

  double sum = obj1->value() + obj2->value();
  args.GetReturnValue().Set(Number::New(isolate, sum));
}

void InitAll(Local<Object> exports) {
  MyObject::Init(exports->GetIsolate());

  NODE_SET_METHOD(exports, "createObject", CreateObject);
  NODE_SET_METHOD(exports, "add", Add);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, InitAll)

}  // namespace demo
```
在 myobject.h 中，新增了一个新的公共方法用于在去包装对象后访问私有值。
```
// myobject.h
#ifndef MYOBJECT_H
#define MYOBJECT_H

#include <node.h>
#include <node_object_wrap.h>

namespace demo {

class MyObject : public node::ObjectWrap {
 public:
  static void Init(v8::Isolate* isolate);
  static void NewInstance(const v8::FunctionCallbackInfo<v8::Value>& args);
  inline double value() const { return value_; }

 private:
  explicit MyObject(double value = 0);
  ~MyObject();

  static void New(const v8::FunctionCallbackInfo<v8::Value>& args);
  static v8::Persistent<v8::Function> constructor;
  double value_;
};

}  // namespace demo

#endif
```
myobject.cc 中的实现类似之前的例子
```
// myobject.cc
#include <node.h>
#include "myobject.h"

namespace demo {

using v8::Context;
using v8::Function;
using v8::FunctionCallbackInfo;
using v8::FunctionTemplate;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::Persistent;
using v8::String;
using v8::Value;

Persistent<Function> MyObject::constructor;

MyObject::MyObject(double value) : value_(value) {
}

MyObject::~MyObject() {
}

void MyObject::Init(Isolate* isolate) {
  // Prepare constructor template
  Local<FunctionTemplate> tpl = FunctionTemplate::New(isolate, New);
  tpl->SetClassName(String::NewFromUtf8(isolate, "MyObject"));
  tpl->InstanceTemplate()->SetInternalFieldCount(1);

  constructor.Reset(isolate, tpl->GetFunction());
}

void MyObject::New(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  if (args.IsConstructCall()) {
    // Invoked as constructor: `new MyObject(...)`
    double value = args[0]->IsUndefined() ? 0 : args[0]->NumberValue();
    MyObject* obj = new MyObject(value);
    obj->Wrap(args.This());
    args.GetReturnValue().Set(args.This());
  } else {
    // Invoked as plain function `MyObject(...)`, turn into construct call.
    const int argc = 1;
    Local<Value> argv[argc] = { args[0] };
    Local<Context> context = isolate->GetCurrentContext();
    Local<Function> cons = Local<Function>::New(isolate, constructor);
    Local<Object> instance =
        cons->NewInstance(context, argc, argv).ToLocalChecked();
    args.GetReturnValue().Set(instance);
  }
}

void MyObject::NewInstance(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();

  const unsigned argc = 1;
  Local<Value> argv[argc] = { args[0] };
  Local<Function> cons = Local<Function>::New(isolate, constructor);
  Local<Context> context = isolate->GetCurrentContext();
  Local<Object> instance =
      cons->NewInstance(context, argc, argv).ToLocalChecked();

  args.GetReturnValue().Set(instance);
}

}  // namespace demo
```
测试：
```
// test.js
const addon = require('./build/Release/addon');

const obj1 = addon.createObject(10);
const obj2 = addon.createObject(20);
const result = addon.add(obj1, obj2);

console.log(result);
// Prints: 30
```
## AtExit钩子
一个atexit钩子在nodejs事件循环结束时触发，但在 JavaScript 虚拟机被终止与 Node.js 关闭前被调用，使用`node::AtExit`API注册
### void AtExit(callback, args)
* `callback <void (*)(void*)>` - 一个退出时调用的函数的指针。
* `args <void*>` - 一个退出时传递给回调的指针。
回调按照后进先出的顺序运行。<br>
以下 addon.cc 实现了 AtExit：
```
// addon.cc
#include <assert.h>
#include <stdlib.h>
#include <node.h>

namespace demo {

using node::AtExit;
using v8::HandleScope;
using v8::Isolate;
using v8::Local;
using v8::Object;

static char cookie[] = "yum yum";
static int at_exit_cb1_called = 0;
static int at_exit_cb2_called = 0;

static void at_exit_cb1(void* arg) {
  Isolate* isolate = static_cast<Isolate*>(arg);
  HandleScope scope(isolate);
  Local<Object> obj = Object::New(isolate);
  assert(!obj.IsEmpty());  // assert VM is still alive
  assert(obj->IsObject());
  at_exit_cb1_called++;
}

static void at_exit_cb2(void* arg) {
  assert(arg == static_cast<void*>(cookie));
  at_exit_cb2_called++;
}

static void sanity_check(void*) {
  assert(at_exit_cb1_called == 1);
  assert(at_exit_cb2_called == 2);
}

void init(Local<Object> exports) {
  AtExit(at_exit_cb2, cookie);
  AtExit(at_exit_cb2, cookie);
  AtExit(at_exit_cb1, exports->GetIsolate());
  AtExit(sanity_check);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, init)

}  // namespace demo
```
测试：
```
// test.js
require('./build/Release/addon');
```
