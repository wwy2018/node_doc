# Buffer
在TypedArray之前，js中没有读取和操作二进制流的机制，Buffer的引入是为了处理类似tcp流和文件操作上下文里操作二进制数据流<br>
typedarray引入之后，buffer类实现并优化了Uint8Array<br>
buffer类实例类似整数数组，但是大小固定不可变，并且在v8堆外分配物理内存<br>
buffer是全局变量
```
// 创建一个以0填充长度为10的buffer
const buf1 = Buffer.alloc(10);

// 创建一个以0x1填充长度为10的buffer
const buf2 = Buffer.alloc(10, 1);

// 创建一个未初始化的长度为10的buffer，通过此方法创建比buffer.alloc()更快，但是创建的实例可能包含旧的数据，需要使用fill()或write()重写
const buf3 = Buffer.allocUnsafe(10);

// 创建buffer [0x1, 0x2, 0x3]
const buf4 = Buffer.from([1, 2, 3]);

// 创建buffer，包含uft8编码的字节[0x74, 0xc3, 0xa9, 0x73, 0x74]
const buf5 = Buffer.from('tést');

// 创建buffer，包含latin1编码的字节[0x74, 0xe9, 0x73, 0x74]
const buf6 = Buffer.from('tést', 'latin1');
```
### Buffer.from(), Buffer.alloc(), and Buffer.allocUnsafe()
nodejs6.0.0之前，buffer实例通过buffer构造函数创建，根据参数不同分配不同的buffer
* 第一个参数传递一个数字，例如new Buffer(10),分配固定长度的buffer，nodejs8.0.0之前，这样的buffer实例不会初始化内存分配，并且能包含敏感信息，这样的实例后续必须使用buf.fill(0)或者写入到整个buffer进行初始化，这样做是为了提高性能，实践经验表明明确区分快速未初始化和慢速但安全的buffer很有必要，从8.0.0开始，Buffer(num)和new Buffer(num)会返回内存初始化的buffer
* 传递字符串，数组或buffer作为第一个参数，会复制对象已有的数据到buffer中
* 传递ArrayBuffer或者SharedArrayBuffer将返回与给定的数组buffer共享内存的buffer实例
---
因为通过new Buffer()创建会因为参数不同而出现多种结果，所以这个方法被弃用了，替代方法有：Buffer.from(), Buffer.alloc(), Buffer.allocUnsafe()<br>
之前使用new Buffer()开发的代码也需要使用新的api替代<br>
* Buffer.from(array) 返回包含给定数组拷贝的buffer
* Buffer.from(arrayBuffer, byteOffset, length)返回一个新的buffer，与给定的arraybuffer共享内存
* Buffer.from(buffer)返回包含给定buffer拷贝的新buffer
* Buffer.from(string, encoding)返回包含给定字符串拷贝的新的buffer
* Buffer.alloc(size, fill, encoding返回固定大小已经初始化的buffer，此方法比Buffer.allocUnsafe(size)慢，但是能保证新的实例不包含旧的敏感数据
* Buffer.allocUnsafe(size)和Buffer.allocUnsafeSlow(size)分别返回固定大小未初始化的buffer，因为没有初始化，分配的内存扇区可能包含敏感旧数据
### --zero-fill-buffers命令
这个命令下所有新的buffer实例自动以0填充，但是对性能有消极影响，只在需要强制保证新创建的实例不能够包含旧的敏感信息时使用
```
$ node --zero-fill-buffers
> Buffer.allocUnsafe(5);
<Buffer 00 00 00 00 00>
```
### 是什么造成Buffer.allocUnsafe()和Buffer.allocUnsafeSlow不安全？
当使用Buffer.allocUnsafe()和Buffer.allocUnsafeSlow()时，分配的内存没有初始化（没有用0填充），这样设计使内存分配非常快速，但是分配的内存区域可能包含旧的敏感数据，通过Buffer.allocUnsafe()创建的buffer如果内存没有重写，读取时可能导致旧数据泄漏<br>
### buffer和字符编码
当存储字符串数据或者导出一个buffer实例时，需要指定字符编码
```
const buf = Buffer.from('hello world', 'ascii');

console.log(buf.toString('hex'));
// Prints: 68656c6c6f20776f726c64
console.log(buf.toString('base64'));
// Prints: aGVsbG8gd29ybGQ=

console.log(Buffer.from('fhqwhgads', 'ascii'));
// Prints: <Buffer 66 68 71 77 68 67 61 64 73>
console.log(Buffer.from('fhqwhgads', 'utf16le'));
// Prints: <Buffer 66 00 68 00 71 00 77 00 68 00 67 00 61 00 64 00 73 00>
```
nodejs目前支持的字符编码有：
* `ascii` - 仅限7位ASCII数据，速度快，自动去掉高位
* `utf8` - 多字节编码的Unicode字符，web页面和文档格式广泛使用
* `utf16le` - 2或4字节，小字节序编码的unicode字符
* `ucs2` - `utf16le`别名
* `base64` - base64编码
* `latin1` - 一种把 Buffer 编码成一字节编码的字符串的方式
* `binary` - `latin1`别名
* `hex` - 将每个字节编码成两个十六进制字符
---
注意：现代浏览器遵循 WHATWG 编码标准 将 'latin1' 和 ISO-8859-1 别名为 win-1252。 这意味着当进行例如 http.get() 这样的操作时，如果返回的字符编码是 WHATWG 规范列表中的，则有可能服务器真的返回 win-1252 编码的数据，此时使用 'latin1' 字符编码可能会错误地解码数据。
### Buffer 与 TypedArray
buffer实例也是uint8array实例，然而和typedarray稍微不同。例如，arraybuffer#slice()会创建切片副本，buffer#slice()在现有的buffer上不经拷贝直接创建，更高效。<br>
通过buffer创建typedarray时需要注意：
1. buffer对象的内存被拷贝到创建的typedarray，不是共享
2. buffer对象的内存解析为一个包含明确元素的数组，而不是目标类型的比特数组，也就是说，new Uint32Array(Buffer.from([1, 2, 3, 4])) 会创建一个包含 [1, 2, 3, 4] 四个元素的 Uint32Array，而不是一个只包含一个元素 [0x1020304] 或 [0x4030201] 的 Uint32Array
---
可以通过typedarray对象的buffer属性创建一个与typedarray共享内存的buffer
```
const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// Copies the contents of `arr`
const buf1 = Buffer.from(arr);
// Shares memory with `arr`
const buf2 = Buffer.from(arr.buffer);

console.log(buf1);
// Prints: <Buffer 88 a0>
console.log(buf2);
// Prints: <Buffer 88 13 a0 0f>

arr[1] = 6000;

console.log(buf1);
// Prints: <Buffer 88 a0>
console.log(buf2);
// Prints: <Buffer 88 13 70 17>
```
注意当使用typedarray的buffer属性创建buffer时，可以通过byteoffset和length两个参数只使用目标typedarray的部分进行创建
```
const arr = new Uint16Array(20);
const buf = Buffer.from(arr.buffer, 0, 16);

console.log(buf.length);
// Prints: 16
```
buffer.from()和typedarray.from()有着不同的签名与实现。 具体而言，TypedArray 的变种接受第二个参数，在类型数组的每个元素上调用一次映射函数：
* TypedArray.from(source[, mapFn[, thisArg]])
---
Buffer.from() 方法不支持使用映射函数
### buffer和迭代器
可以使用for..of对buffer实例进行迭代
```
const buf = Buffer.from([1, 2, 3]);

// Prints:
//   1
//   2
//   3
for (const b of buf) {
  console.log(b);
}
```
另外，buf.values(), buf.keys(), buf.entries()可以生成迭代器
### buffer类
buffer类是一个全局变量类型，用来直接处理二进制数据的。 它能够使用多种方式构建。
### new Buffer(array) 已废弃
### new Buffer(arrayBuffer[, byteOffset [, length]]) 已废弃
### new Buffer(buffer) 已废弃
### new Buffer(size) 已废弃
### new Buffer(string[, encoding]) 已废弃
### 类方法：buffer.alloc(size,fill,encoding)
* size 整数类型 目标buffer长度
* fill 字符串／buffer／整数 默认0
* encoding 字符串 默认utf8
---
分配一个大小为size字节的新的buffer，默认填充值为0
```
const buf = Buffer.alloc(5);

console.log(buf);
// Prints: <Buffer 00 00 00 00 00>
```
如果size超过buffer.constants.max_length或者小于0，会抛出rangeerror，如果size为0会创建一个大小为0的buffer<br>
如果指定了fill参数，初始化时会调用buf.fill(fill)
```
const buf = Buffer.alloc(5, 'a');

console.log(buf);
// Prints: <Buffer 61 61 61 61 61>
```
如果fill和encoding都指定了，初始化时会调用buf.fill(fill, encoding)
```
const buf = Buffer.alloc(11, 'aGVsbG8gd29ybGQ=', 'base64');

console.log(buf);
// Prints: <Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64>
```
调用 Buffer.alloc() 会明显地比另一个方法 Buffer.allocUnsafe() 慢，但是能确保新建的 Buffer 实例的内容不会包含敏感数据。<br>

如果 size 不是一个数值，则抛出 TypeError 错误
### 类方法：buffer.allocunsafe(size)
* size 整形 目标长度
---
分配一个大小为 size 字节的新建的 Buffer 。 如果 size 大于 buffer.constants.MAX_LENGTH 或小于 0，则抛出 RangeError 错误。 如果 size 为 0，则创建一个长度为 0 的 Buffer<br>
以这种方式创建的 Buffer 实例的底层内存是未初始化的。 新创建的 Buffer 的内容是未知的，且可能包含敏感数据。 可以使用 buf.fill(0) 初始化 Buffer 实例为0。
```
const buf = Buffer.allocUnsafe(10);

console.log(buf);
// Prints: (contents may vary): <Buffer a0 8b 28 3f 01 00 00 00 50 32>

buf.fill(0);

console.log(buf);
// Prints: <Buffer 00 00 00 00 00 00 00 00 00 00>
```
如果size不是数字会抛出typeerror<br>
注意buffer模块会预分配一个内部的buffer实例，大小为buffer.pollsize，作为快速创建buffer实例的池子，当size小于或等于Buffer.poolSize>>1（buffer.poolsize除二之后取整）时，buffer.allocunsafe()将从快速分配池创建<br>
对这个预分配的内部内存池的使用，是调用 Buffer.alloc(size, fill) 和 Buffer.allocUnsafe(size).fill(fill) 的关键区别。 具体地说，Buffer.alloc(size, fill) 永远不会使用这个内部的 Buffer 池，但如果 size 小于或等于 Buffer.poolSize 的一半， Buffer.allocUnsafe(size).fill(fill) 会使用这个内部的 Buffer 池。 当应用程序需要 Buffer.allocUnsafe() 提供额外的性能时，这个细微的区别是非常重要的。
### 类方法：Buffer.allocUnsafeSlow(size)
* size 整形 目标长度
---
分配一个大小为 size 字节的新建的 Buffer 。 如果 size 大于 buffer.constants.MAX_LENGTH 或小于 0，则抛出 RangeError 错误。 如果 size 为 0，则创建一个长度为 0 的 Buffer<br>
以这种方式创建的 Buffer 实例的底层内存是未初始化的。 新创建的 Buffer 的内容是未知的，且可能包含敏感数据。 可以使用 buf.fill(0) 初始化 Buffer 实例为0。<br>
当使用buffer.allocunsafe()时，如果分配的内存小于4kb，将直接从单个预分配的buffer内存中切割出来，以避免创建过多独立分配的buffer实例造成垃圾回收超负荷，同时因为无需跟踪和清理多个持久化对象从而提高性能和内存使用<br>
当然，在开发者可能需要在不确定的时间段从内存池保留一小块内存的情况下，使用 Buffer.allocUnsafeSlow() 创建一个非池的 Buffer 实例然后拷贝出相关的位元是合适的做法
```
// Need to keep around a few small chunks of memory
const store = [];

socket.on('readable', () => {
  const data = socket.read();

  // Allocate for retained data
  const sb = Buffer.allocUnsafeSlow(10);

  // Copy the data into the new allocation
  data.copy(sb, 0, 0, 10);

  store.push(sb);
});
```
Buffer.allocUnsafeSlow() 应当仅仅作为开发者已经在他们的应用程序中观察到过度的内存保留之后的终极手段使用。
### 类方法: Buffer.byteLength(string[, encoding])
返回一个字符串的实际字节长度。 这与 String.prototype.length 不同，因为那返回字符串的字符数。<br>

注意 对于 'base64' 和 'hex'， 该函数假定有效的输入。 对于包含 non-Base64/Hex-encoded 数据的字符串 (e.g. 空格)， 返回值可能大于 从字符串中创建的 Buffer 的长度。
```
const str = '\u00bd + \u00bc = \u00be';

console.log(`${str}: ${str.length} characters, ` +
            `${Buffer.byteLength(str, 'utf8')} bytes`);
// Prints: ½ + ¼ = ¾: 9 characters, 12 bytes
```
当`string`是一个 `Buffer/DataView/TypedArray/ArrayBuffer/SharedArrayBuffer` 时，返回实际的字节长度。
### 类方法：Buffer.compare(buf1, buf2)
比较 buf1 和 buf2 ，通常用于 Buffer 实例数组的排序。 相当于调用 buf1.compare(buf2) 。
```
onst buf1 = Buffer.from('1234');
const buf2 = Buffer.from('0123');
const arr = [buf1, buf2];

console.log(arr.sort(Buffer.compare));
// Prints: [ <Buffer 30 31 32 33>, <Buffer 31 32 33 34> ]
// (This result is equal to: [buf2, buf1])
```
### 类方法：Buffer.concat(list[, totalLength])
### 类方法：Buffer.from(array)
### 类方法：Buffer.from(arrayBuffer[, byteOffset[, length]])
### 类方法：Buffer.from(buffer)
### 类方法：Buffer.from(string[, encoding])
### 类方法：Buffer.from(object[, offsetOrEncoding[, length]])
* `object <Object> 一个支持Symbol.toPrimitive 或者 valueOf()的对象`
* `offsetOrEncoding <number> | <string> 字节偏移量或者编码类型，依赖返回值是根据object.valueOf() 还是 object[Symbol.toPrimitive]()`
* `length <number> 长度值, 依赖返回值是根据object.valueOf() 还是 object[Symbol.toPrimitive]()`
---
对于一个自身valueof()方法返回的值不严格等于这个对象时，返回Buffer.from(object.valueOf(), offsetOrEncoding, length).
```
const buf = Buffer.from(new String('this is a test'));
// Prints: <Buffer 74 68 69 73 20 69 73 20 61 20 74 65 73 74>
```
对于支持Symbol.toPrimitive的对象返回Buffer.from(object[Symbol.toPrimitive](), offsetOrEncoding, length)
```
class Foo {
  [Symbol.toPrimitive]() {
    return 'this is a test';
  }
}

const buf = Buffer.from(new Foo(), 'utf8');
// Prints: <Buffer 74 68 69 73 20 69 73 20 61 20 74 65 73 74>
```
### 类方法：Buffer.isBuffer(obj)
### 类方法：Buffer.isEncoding(encoding)
### 类方法：Buffer.poolSize
### buf[index]
通过索引获取或设置index位置的单个字节，合法的值范围从0x00到0xff（16进制）或者0到255（10进制）<br>
这个方法继承自uint8array,所以超出范围的操作和uint8array的处理相同--返回udefined值，不能设置值
```
// Copy an ASCII string into a `Buffer` one byte at a time.

const str = 'Node.js';
const buf = Buffer.allocUnsafe(str.length);

for (let i = 0; i < str.length; i++) {
  buf[i] = str.charCodeAt(i);
}

console.log(buf.toString('ascii'));
// Prints: Node.js
```
### buf.buffer
代表buffer所依赖创建的底层arraybuffer
```
const arrayBuffer = new ArrayBuffer(16);
const buffer = Buffer.from(arrayBuffer);

console.log(buffer.buffer === arrayBuffer);
// Prints: true
```
### buf.compare(target[, targetStart[, targetEnd[, sourceStart[, sourceEnd]]]])
### buf.copy(target[, targetStart[, sourceStart[, sourceEnd]]])
### buf.entries()
### buf.equals(otherBuffer)
### buf.fill(value[, offset[, end]][, encoding])
### buf.includes(value[, byteOffset][, encoding])
### buf.indexOf(value[, byteOffset][, encoding])
### buf.equals(otherBuffer)
### buf.fill(value[, offset[, end]][, encoding])
### buf.includes(value[, byteOffset][, encoding])
### buf.indexOf(value[, byteOffset][, encoding])
### buf.keys()
### buf.lastIndexOf(value[, byteOffset][, encoding])
### buf.length
### buf.readDoubleBE(offset[, noAssert])
### buf.readDoubleLE(offset[, noAssert])
### buf.readInt8(offset[, noAssert])
### buf.readInt16BE(offset[, noAssert])
### buf.readInt16LE(offset[, noAssert])
### buf.readIntBE(offset, byteLength[, noAssert])
### buf.readIntLE(offset, byteLength[, noAssert])
### buf.readUInt8(offset[, noAssert])
### buf.readUInt16BE(offset[, noAssert])
### buf.readUInt16LE(offset[, noAssert])
### buf.readUInt32BE(offset[, noAssert])
### buf.readUInt32LE(offset[, noAssert])
### buf.readUIntBE(offset, byteLength[, noAssert])
### buf.readUIntLE(offset, byteLength[, noAssert])
### buf.slice([start[, end]])
### buf.swap16()
### buf.swap32()
### buf.swap64()
### buf.toJSON()
### buf.toString([encoding[, start[, end]]])
### buf.values()
### buf.write(string[, offset[, length]][, encoding])
### buf.writeDoubleBE(value, offset[, noAssert])
### buf.writeDoubleLE(value, offset[, noAssert])
### buf.writeFloatBE(value, offset[, noAssert])
### buf.writeFloatLE(value, offset[, noAssert])
### buf.writeInt8(value, offset[, noAssert])
### buf.writeInt16BE(value, offset[, noAssert])
### buf.writeInt16LE(value, offset[, noAssert])
### buf.writeInt32BE(value, offset[, noAssert])
### buf.writeInt32LE(value, offset[, noAssert])
### buf.writeIntBE(value, offset, byteLength[, noAssert])
### buf.writeIntLE(value, offset, byteLength[, noAssert])
### buf.writeUInt8(value, offset[, noAssert])
### buf.writeUInt16BE(value, offset[, noAssert])
### buf.writeUInt16LE(value, offset[, noAssert])
### buf.writeUInt32BE(value, offset[, noAssert])
### buf.writeUInt32LE(value, offset[, noAssert])
### buf.writeUIntBE(value, offset, byteLength[, noAssert])
### buf.writeUIntLE(value, offset, byteLength[, noAssert])
### buffer.INSPECT_MAX_BYTES
### buffer.kMaxLength
### buffer.transcode(source, fromEnc, toEnc)
### Buffer Constants
### buffer.constants.MAX_LENGTH
### buffer.constants.MAX_STRING_LENGTH


