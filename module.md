在node模块系统里，每个文件都被看作为一个独立模块

nodejs将模块封装在一个函数里：
```
(function(exports, require, module, __filename, __dirname) {
// Module code actually lives in here
});
```
这样就会产生如下特性：
* 模块中的顶层变量（通过var,const,let定义的变量）的作用域限制在模块内，而不是全局
* 提供module和exports供模块导出值
* 提供方便使用的`__filename`和`__dirname`变量，它们分别保存了模块名称和绝对目录地址
# require.main()
通过node命令直接运行模块时，`require.main === module`，这是由于module的`__filename`参数和`require.man.filename`属性比较之后的结果
# 依赖管理
为了让模块能在node repl中使用，可以将`/usr/lib/node_modules`设置为`$NODE_PATH`，由于require查找的是绝对路径，这样设置之后，模块就可以在任何地方引用了
# exports
exports是`module.exports`的简短引用