# 调用栈和错误堆栈

很多同学每天都会面对一大堆错误信息，但是繁杂的错误堆栈信息是否会对我们排查错误产生影响呢。今天我们就通过Node的原生文档和源码，加深对Node调用栈和错误捕获的理解。

## JS调用栈的工作原理

在理解js的异常抛出和捕获之前，首先要理解调用栈的工作原理。如果你已经对这些很熟悉了，那么可以跳过这一部分。

> 首先回忆一下函数调用的大概过程：在栈上创建。在执行函数时，函数内局部变量的存储单元都在栈上创建，函数执行结束时这些存储单元自动被释放。

如下图：

![avatar](http://owkj16paf.bkt.clouddn.com/markdown-03-%E8%B0%83%E7%94%A8%E6%A0%88.png)


简单的说：函数被调用时，就会被加入到调用栈顶部，执行结束之后，就会从调用栈顶部移除该函数，这种数据结构的关键在于后进先出，即大家所熟知的 LIFO。比如，当我们在函数 y 内部调用函数 x 的时候，调用栈从下往上的顺序就是 y -> x 。

```javascript
function c() {
    console.log('c');
}

function b() {
    console.log('b');
    c();
}

function a() {
    console.log('a');
    b();
}

a();
```

`执行顺序：   a -> b -> c`

`栈移除顺序： c -> b -> a`


为了更好的说明调用栈的工作机制，我们对上面的代码稍作改动，使用 console.trace 来把当前的调用栈输出到 console 中，你可以认为console.trace 打印出来的调用栈的每一行出现的原因是它下面的那行调用而引起的。

```javascript
function c() {
    console.log('c');
    console.trace();
}

function b() {
    console.log('b');
    c();
}

function a() {
    console.log('a');
    b();
}

a();
```

结果：

![avatar](http://owkj16paf.bkt.clouddn.com/markdown-03-%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C1.jpg)

显而易见当我们在 c 内部调用 console.trace 的时候，调用栈从下往上的结构是：`a -> b -> c`。我们再修改一下示例代码。

```javascript
function c() {
    console.log('c');
}

function b() {
    console.log('b');
    c();
    console.trace();
}

function a() {
    console.log('a');
    b();
}

a();
```

结果：

![avatar](http://owkj16paf.bkt.clouddn.com/markdown-03-%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C2.jpg)

我们可以看到，此时打印的调用栈从下往上是：a -> b，已经没有 c 了，因为 c 执行完之后就从调用栈移除了。

了解函数的调用栈对我们理解接下来的错误堆栈信息很帮助。


## Error对象及错误处理

首先引入Node文档中对Node.js错误的描述：

>Node.js 中运行的应用程序一般会遇到以下四类错误：
>
>* 标准的 JavaScript 错误：
>    * {EvalError} : 当调用 `eval()` 失败时抛出。
>    * {SyntaxError} : 当 JavaScript 语法错误时抛出。
>    * {RangeError} : 当值不在预期范围内时抛出。
>    * {ReferenceError} : 当使用未定义的变量时抛出。
>    * {TypeError} : 当传入错误类型的参数时抛出。
>    * {URIError} : 当全局的 URI 处理函数被误用时抛出。
>* 由底层操作系触发的系统错误，例如试图打开一个不存在的文件、试图通过一个已关闭的 socket 发送数据等。
>* 由应用程序代码触发的用户自定义的错误。
>* 断言错误是错误的一个特殊类别，每当 Node.js 检测到一个不应该发生的异常逻辑时触发。
>    这类错误通常由 `assert` 模块引起。
>
>所有由 Node.js 引起的 JavaScript 错误与系统错误都继承自或实例化自标准的JavaScript {Error} 类，且保证至少提供类中的属性。

我们可以看出所有由 `Node.js` 产生的错误，包括所有系统的和 JavaScript 的错误都实例化自或继承自 Error 类。`Error` 对象会捕捉一个“堆栈跟踪”，详细说明被实例化的 `Error` 对象在代码中的位置，并可能提供错误的文字描述。

常见的Error类型`(均是Error类的子类，继承至Error类)`：

* RangeError: Error 的一个子类，表明一个函数的一个给定的参数的值不在可接受的集合或范围内； 无论是一个数字范围还是给定函数参数的选项的集合。

* ReferenceError: Error 的一个子类，表明试图访问一个未定义的变量。 这些错误通常表明代码有拼写错误或程序已损坏。

* SyntaxError: Error 的一个子类，表明程序不是有效的 JavaScript 代码。 这些错误是代码执行的结果产生和传播的。 代码执行可能产生自 eval、Function、require 或 vm。 这些错误几乎都表明程序已损坏。

* TypeError: Error 的一个子类，表明提供的参数不是一个被允许的类型。 例如，将一个函数传给一个期望字符串的参数会被视为一个 TypeError。

* AssertionError: Error 的一个子类, 常用于断言模块。

我们看一段Node中关于Error类的源码：

```javascript
// node-master/lib/internal/errors.js

// 使用ES6中的新数据类型Symbol定义变量，确保变量唯一不重复
const kCode = Symbol('code');
// 定义枚举错误信息集合
const messages = new Map();

function makeNodeError(Base) {
    // 错误继承至传入的Base，Base为Error类或继承至Error类
    return class NodeError extends Base {
        constructor(key, ...args) {
            super(message(key, args));
            this[kCode] = key;
        }

        // 代理方法，获取捕获错误的name值
        get name() {
            return `${super.name} [${this[kCode]}]`;
        }

        // 代理犯法，获取捕获错误的code值
        get code() {
            return this[kCode];
        }
    };
}

// ...

// 使用Node内置util工具类输出标准类型的错误信息 util.format()
function message(key, args) {
    const msg = messages.get(key);
    internalAssert(msg, `An invalid error message key was used: ${key}.`);
    let fmt;
    if (typeof msg === 'function') {
        fmt = msg;
    } else {
        if (util === null) util = require('util');
        fmt = util.format;
        if (args === undefined || args.length === 0)
            return msg;
        args.unshift(msg);
    }
    return String(fmt.apply(null, args));
}

// 将枚举的错误信息写入集合中的方法
function E(sym, val) {
    messages.set(sym, typeof val === 'function' ? val : String(val));
}

// 枚举错误信息

//... 各种预定义枚举错误，写入到Map对象中
E('ERR_INDEX_OUT_OF_RANGE', 'Index out of range');
E('ERR_INVALID_ARG_TYPE', invalidArgType);
E('ERR_INVALID_ARRAY_LENGTH',
    (name, len, actual) => {
        internalAssert(typeof actual === 'number', 'actual must be a number');
        return `The array "${name}" (length ${actual}) must be of length ${len}.`;
    });
//... 各种预定义枚举错误

// 拼装错误信息
function invalidArgType(name, expected, actual) {
    internalAssert(name, 'name is required');

    // determiner: 'must be' or 'must not be'
    let determiner;
    if (expected.includes('not ')) {
        determiner = 'must not be';
        expected = expected.split('not ')[1];
    } else {
        determiner = 'must be';
    }

    let msg;
    if (Array.isArray(name)) {
        var names = name.map((val) => `"${val}"`).join(', ');
        msg = `The ${names} arguments ${determiner} ${oneOf(expected, 'type')}`;
    } else if (name.includes(' argument')) {
        // for the case like 'first argument'
        msg = `The ${name} ${determiner} ${oneOf(expected, 'type')}`;
    } else {
        const type = name.includes('.') ? 'property' : 'argument';
        msg = `The "${name}" ${type} ${determiner} ${oneOf(expected, 'type')}`;
    }

    // if actual value received, output it
    if (arguments.length >= 3) {
        msg += `. Received type ${actual !== null ? typeof actual : 'null'}`;
    }
    return msg;
}

module.exports = exports = {
    message,
    Error: makeNodeError(Error),
    TypeError: makeNodeError(TypeError),
    RangeError: makeNodeError(RangeError),
    // 断言类错误
    AssertionError,
    // 内部方法，暴露到外部只为测试
    E,
};
```

调用示例：

```javascript
// node-master/lib/internal/process.js

const errors = require('internal/errors');

// 抛出一个类型错误的异常
throw new errors.TypeError('ERR_INVALID_ARG_TYPE',
                                   'preValue.user', 'Number');
```

输出：

![avatar](http://owkj16paf.bkt.clouddn.com/markdown-03-%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C3.jpg)

当代码中我们想抛出一个自定义类型的错误时，我们通常会这样：

```javascript
throw new Error('错误的描述');
```

新建一个 `Error` 实例，并设置 `error.message` 属性以提供文本信息。 如果 `message` 传的是一个对象，则会调用 `message.toString()` 生成文本信息。 `error.stack` 属性表示被调用的 `new Error()` 在代码中的位置。 堆栈跟踪是基于 V8 的堆栈跟踪 API 的。 堆栈跟踪只会取（a）异步代码执行的开头或（b）`Error.stackTraceLimit` 属性给出的栈帧中的最小项。

Error 对象可以作为扩展和创建自定义错误类型的原型。Error 对象的 prototype 具有以下属性：

* constructor: 负责该实例的原型构造函数；
* message: 错误信息；
* name: 错误的名字；

上面都是标准属性，有些 JS 运行环境还提供了标准属性之外的属性，如 Node.js、Firefox、Chrome、IE、Opera 和 Safari 中会有 stack 属性，它包含了错误代码的调用栈，接下来我们简称错误堆栈。`错误堆栈包含了产生该错误时完整的调用栈信息。`

出错误时，你必须使用 throw 关键字。为了捕获抛出的错误，则必须使用 try catch 语句把可能出错的代码块包起来，catch 的时候可以接收一个参数，该参数就是被抛出的错误。与 Java 中类似，JS 中也可以在 try catch 语句之后有 finally，不论前面代码是否抛出错误 finally 里面的代码都会执行，这种语言的常见用途有：在 finally 中做些清理的工作。

三种常见try语句形式：

* try ... catch
* try ... finally
* try ... catch ... finally

`有一点需要的注意的是：throw可以抛出不是错误对象的任意值，但这对我们捕获错误分析错误堆栈信息很不友好，建议不要这样使用。`

`try ... catch` 只适用于捕获同步的错误。因为异步的错误在抛出错误的时候，捕获错误的catch函数已经执行完毕，所以无法捕获异步错误。对于所有的 `EventEmitter` 对象，如果没有提供一个 'error' 事件句柄，则错误会被抛出，并造成 Node.js 进程报告一个未处理的异常且随即崩溃，除非： 适当地使用 `domain` 模块或已经注册了一个 `process.on('uncaughtException')` 事件的句柄。

```javascript
const EventEmitter = require('events');
const ee = new EventEmitter();

setImmediate(() => {
  // 这会使进程崩溃，因为还未添加 'error' 事件句柄。
  ee.emit('error', new Error('这会崩溃'));
});
```

这种方式产生的错误无法使用 try / catch 截获，因为它们是在调用的代码已经退出后抛出的。

JavaScript 的 `try / catch` 机制无法用于捕获由异步 API 引起的错误。 尝试使用 throw 而不是一个 Node.js 风格的回调，是初学者常犯的错误：

```javascript
// 这无法使用：
const fs = require('fs');

try {
    fs.readFile('/some/file/that/does-not-exist', (err, data) => {
        // 假设的错误：在这里抛出
        if (err) {
        throw err;
        }
    });
} catch (err) {
    // 这不会捕获到抛出！
    console.error(err);
}
```

这无法使用，因为传给 `fs.readFile()` 的回调函数是被异步地调用。 当回调被调用时，周围的代码（包括 `try { } catch (err) { }` 区域）已经退出。 大多数情况下，在回调内抛出一个错误会使 Node.js 进程崩溃。 如果域已启用，或已在 `process.on('uncaughtException')` 注册了一个句柄，则这些错误可被捕获。


## 错误堆栈的裁剪

Node.js 才支持这个特性，通过 Error.captureStackTrace 来实现，Error.captureStackTrace 接收一个 object 作为第 1 个参数，以及可选的 function 作为第 2 个参数。其作用是捕获当前的调用栈并对其进行裁剪，捕获到的调用栈会记录在第 1 个参数的 stack 属性上，裁剪的参照点是第 2 个参数，也就是说，此函数之前的调用会被记录到调用栈上面，而之后的不会。

让我们用代码来说明，首先，把当前的调用栈捕获并放到 myObj 上：

```javascript
const myObj = {};

function c() {
}

function b() {
    // 把当前调用栈写到 myObj 上
    Error.captureStackTrace(myObj);
    c();
}

function a() {
    b();
}

// 调用函数 a
a();

// 打印 myObj.stack
console.log(myObj.stack);
```

输出：

![avatar](http://owkj16paf.bkt.clouddn.com/markdown-03-%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C4.jpg)

上面的调用栈中只有 `a -> b`，因为我们在 b 调用 c 之前就捕获了调用栈。现在对上面的代码稍作修改，然后看看会发生什么：

```javascript
const myObj = {};

function d() {
    // 我们把当前调用栈存储到 myObj 上，但是会去掉 b 和 b 之后的部分
    Error.captureStackTrace(myObj, b);
}

function c() {
    d();
}

function b() {
    c();
}

function a() {
    b();
}

// 执行代码
a();

// 打印 myObj.stack
console.log(myObj.stack);
```

输出：

![avatar](http://owkj16paf.bkt.clouddn.com/markdown-03-%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C5.jpg)

在这段代码里面，因为我们在调用 Error.captureStackTrace 的时候传入了 b，这样 b 之后的调用栈都会被隐藏。

现在你可能会问，知道这些到底有啥用？如果你想对用户隐藏跟他业务无关的错误堆栈（比如某个库的内部实现）就可以试用这个技巧。

用在向最终用户隐藏错误生成的具体细节时非常有用。例如：

```javascript
function MyError() {
  Error.captureStackTrace(this, MyError);
}

// 没传入 MyError 到 captureStackTrace，MyError 帧会显示在 .stack 属性。
// 通过传入构造函数，可以省略该帧，且保留其下面的所有帧。
new MyError().stack;
```

### 最后附上常见的系统错误，供大家参考（取自Node文档，v8.6.0）

以下列表是不完整的，但列举了编写 Node.js 程序时会遇到的一些常见的系统错误。 详细的列表可从 ERRNO 文档找到。

* EACCES (拒绝访问): 试图以被一个文件的访问权限禁止的方式访问一个文件。

* EADDRINUSE (地址已被使用): 试图绑定一个服务器（[net]、[http] 或 [https]）到本地地址，但因另一个本地系统的服务器已占用了该地址而导致失败。

* ECONNREFUSED (连接被拒绝): 无法连接，因为目标机器积极拒绝。 这通常是因为试图连接到外部主机上的废弃的服务。

* ECONNRESET (连接被重置): 一个连接被强行关闭。 这通常是因为连接到远程 socket 超时或重启。 常发生于 [http] 和 [net] 模块。

* EEXIST (文件已存在): 一个操作的目标文件已存在，而要求目标不存在。

* EISDIR (是一个目录): 一个操作要求一个文件，但给定的路径是一个目录。

* EMFILE (系统打开了太多文件): 已达到系统[文件描述符]允许的最大数量，且描述符的请求不能被满足直到至少关闭其中一个。 当一次并行打开多个文件时会发生这个错误，尤其是在进程的文件描述限制数量较低的操作系统（如 macOS）。 要解决这个限制，可在运行 Node.js 进程的同一 shell 中运行 ulimit -n 2048。

* ENOENT (无此文件或目录): 通常是由 [fs] 操作引起的，表明指定的路径不存在，即给定的路径找不到文件或目录。

* ENOTDIR (不是一个目录): 给定的路径虽然存在，但不是一个目录。 通常是由 [fs.readdir] 引起的。

* ENOTEMPTY (目录非空): 一个操作的目标是一个非空的目录，而要求的是一个空目录。 通常是由 [fs.unlink] 引起的。

* EPERM (操作不被允许): 试图执行一个需要更高权限的操作。

* EPIPE (管道损坏): 写入一个管道、socket 或 FIFO 时没有进程读取数据。 常见于 [net] 和 [http] 层，表明远端要写入的流已被关闭。

* ETIMEDOUT (操作超时): 一个连接或发送的请求失败，因为连接方在一段时间后没有做出合适的响应。 常见于 [http] 或 [net]。 往往标志着 socket.end() 没有被正确地调用。

参考链接： http://nodejs.cn/api/errors.html#errors_common_system_errors
