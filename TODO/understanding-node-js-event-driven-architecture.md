> * 原文地址：[Understanding Node.js Event-Driven Architecture](https://medium.freecodecamp.com/understanding-node-js-event-driven-architecture-223292fcbc2d)
> * 原文作者：[Samer Buna](https://medium.freecodecamp.com/@samerbuna)
> * 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
> * 译者：[薛定谔的猫]((https://github.com/Aladdin-ADD)
> * 校对者：

# Understanding Node.js Event-Driven Architecture #
# 理解 Node.js 事件驱动

![](https://cdn-images-1.medium.com/max/2000/1*Nozl2qd0SV8Uya2CEkF_mg.jpeg)

Most of Node’s objects — like HTTP requests, responses, and streams — implement the `EventEmitter` module so they can provide a way to emit and listen to events.
大多数 Nodejs 对象 -- 比如说 HTTP requests、responses、stream -- 都实现了 `EventEmitter` 模块，因此它们能够提供发出（emit）和 监听（listen）事件。
![](https://cdn-images-1.medium.com/max/800/1*74K5OhiYt7WTR0WuVGeNLQ.png)

The simplest form of the event-driven nature is the callback style of some of the popular Node.js functions — for example, `fs.readFile`. In this analogy, the event will be fired once (when Node is ready to call the callback) and the callback acts as the event handler.
事件驱动的最简单的形式就是回调函数。正如 `fs.readFile` 那样，当 Node 要调用回调函数时，会触发一个事件，回调函数就作为一个事件处理器。
Let’s explore this basic form first.
下面我们先来看一下这种简单的情况。
#### Call me when you’re ready, Node! ####
#### 回调函数 ####

The original way Node handled asynchronous events was with callback. This was a long time ago, before JavaScript had native promises support and the async/await feature.
回调函数是最原始的异步事件处理机制。在 JavaScript 原生支持 promise 和 async/await 之前就存在了很久。
Callbacks are basically just functions that you pass to other functions. This is possible in JavaScript because functions are first class objects.
回调函数就是将函数作为参数传递给其它函数。这样做是可行的，因为在 JavaScript 中函数是一等公民。
It’s important to understand that callbacks do not indicate an asynchronous call in the code. A function can call the callback both synchronously and asynchronously.
需要理解的很重要的一点：回调函数并不一定是异步的。可以以同步/异步的方式调用回调函数。
For example, here’s a host function `fileSize` that accepts a callback function `cb` and can invoke that callback function both synchronously and asynchronously based on a condition:
举例来说，下面的代码中 `fileSize` 函数可以接受一个回调函数 `cb` 作为参数，根据条件可以执行同步/异步方式调用。
```
function fileSize (fileName, cb) {
  if (typeof fileName !== 'string') {
    return cb(new TypeError('argument should be string')); // Sync
  }
  
  fs.stat(fileName, (err, stats) => {
    if (err) { return cb(err); } // Async
    
    cb(null, stats.size); // Async
  });
}
```

Note that this is a bad practice that leads to unexpected errors. Design host functions to consume callback either always synchronously or always asynchronously.
请注意：这并不是一个好的实践，它会导致预期外的错误。推荐设计 host functions 总是使用同步，或者异步的方式。

Let’s explore a simple example of a typical asynchronous Node function that’s written with a callback style:
我们来以回调方式写一个典型的异步函数的例子吧！
```
const readFileAsArray = function(file, cb) {
  fs.readFile(file, function(err, data) {
    if (err) {
      return cb(err);
    }

    const lines = data.toString().trim().split('\n');
    cb(null, lines);
  });
};
```

`readFileAsArray` takes a file path and a callback function. It reads the file content, splits it into an array of lines, and calls the callback function with that array.
`readFileAsArray` 接受一个文件路径和回调作为参数。它读取文件内容，分隔为行的数组，对这个数组调用回调函数。
Here’s an example use for it. Assuming that we have the file `numbers.txt` in the same directory with content like this:
下面是调用它的例子。假设我们在同目录下有一个名为 `numbers.txt` 的文件，内容如下：
```
10
11
12
13
14
15
```

If we have a task to count the odd numbers in that file, we can use `readFileAsArray` to simplify the code:
如果我们要统计这个文件中奇数的个数，可以使用 `readFileAsArray` 来简化代码：

```
readFileAsArray('./numbers.txt', (err, lines) => {
  if (err) throw err;

  const numbers = lines.map(Number);
  const oddNumbers = numbers.filter(n => n%2 === 1);
  console.log('Odd numbers count:', oddNumbers.length);
});
```

The code reads the numbers content into an array of strings, parses them as numbers, and counts the odd ones.
这段代码将数字内容读取到一个字符串数组中，转化为数字，然后统计奇数的个数。

Node’s callback style is used purely here. The callback has an error-first argument `err` that’s nullable and we pass the callback as the last argument for the host function. You should always do that in your functions because users will probably assume that. Make the host function receive the callback as its last argument and make the callback expect an error object as its first argument.
这里仅仅使用了Node的回调方式。回调函数的第一个参数`err`表示错误（可以为null），第二个参数为回调函数。你应当遵守这一约定，因为使用者很可能有这样的假定。确保host function最后一个参数为回调函数，该函数的第一个参数为错误对象。
#### The modern JavaScript alternative to Callbacks ####
#### 现代JavaScript方式 ####
In modern JavaScript, we have promise objects. Promises can be an alternative to callbacks for asynchronous APIs. Instead of passing a callback as an argument and handling the error in the same place, a promise object allows us to handle success and error cases separately and it also allows us to chain multiple asynchronous calls instead of nesting them.
在现代JavaScript程序中，我们可以使用promise对象。Promise可以取代回调来处理异步API。相比回调函数将错误对象作为第一个参数，promise可以让我们分别处理成功和失败的情况，并且允许我们链式处理异步调用，而不是像回调一样嵌套。
If the `readFileAsArray` function supports promises, we can use it as follows:
如果`readFileAsArray`支持promise，我们就可以这样调用：
```
readFileAsArray('./numbers.txt')
  .then(lines => {
    const numbers = lines.map(Number);
    const oddNumbers = numbers.filter(n => n%2 === 1);
    console.log('Odd numbers count:', oddNumbers.length);
  })
  .catch(console.error);
```

Instead of passing in a callback function, we called a `.then` function on the return value of the host function. This `.then` function usually gives us access to the same lines array that we get in the callback version, and we can do our processing on it as before. To handle errors, we add a `.catch` call on the result and that gives us access to an error when it happens.
不再需要传递一个回调函数，我们对host function 的返回值调用了一个`.then`函数。这个函数的参数就是上面回调方式中的行数组，剩下的操作和之前一样。要处理错误情况，我们可以调用`.catch`，当有错误发生时，它可以接收到错误对象。

Making the host function support a promise interface is easier in modern JavaScript thanks to the new Promise object. Here’s the `readFileAsArray` function modified to support a promise interface in addition to the callback interface it already supports:
在现代JavaScript中有了promise对象，我们可以很容易的让它支持promise接口。
```
const readFileAsArray = function(file, cb = () => {}) {
  return new Promise((resolve, reject) => {
    fs.readFile(file, function(err, data) {
      if (err) {
        reject(err);
        return cb(err);
      }
      
      const lines = data.toString().trim().split('\n');
      resolve(lines);
      cb(null, lines);
    });
  });
};
```

So we make the function return a Promise object, which wraps the `fs.readFile` async call. The promise object exposes two arguments, a `resolve` function and a `reject` function.
所以我们确保函数返回一个包裹了`fs.readFile`调用的promise对象，它暴露了2个参数：`reslove` 和 `reject` 函数。
Whenever we want to invoke the callback with an error we use the promise `reject` function as well, and whenever we want to invoke the callback with data we use the promise `resolve` function as well.
无论何时，想调用错误情况的回调，调用promise对象的 `reject` 函数；当调用执行成功的回调，使用promise的`reslove`函数。
The only other thing we needed to do in this case is to have a default value for this callback argument in case the code is being used with the promise interface. We can use a simple, default empty function in the argument for that case: `() => {}`.
我们需要做的另外一件事情是：给传入的回调函数参数一个默认值。在这个例子中，我们简单的给一个默认的空函数：`() => {}`。
#### Consuming promises with async/await ####
#### async/await ####
Adding a promise interface makes your code a lot easier to work with when there is a need to loop over an async function. With callbacks, things become messy.

Promises improve that a little bit, and function generators improve on that a little bit more. This said, a more recent alternative to working with async code is to use the `async` function, which allows us to treat async code as if it was synchronous, making it a lot more readable overall.

Here’s how we can consume the `readFileAsArray` function with async/await:

```
async function countOdd () {
  try {
    const lines = await readFileAsArray('./numbers');
    const numbers = lines.map(Number);
    const oddCount = numbers.filter(n => n%2 === 1).length;
    console.log('Odd numbers count:', oddCount);
  } catch(err) {
    console.error(err);
  }
}

countOdd();
```

We first create an async function, which is just a normal function with the word `async` before it. Inside the async function, we call the `readFileAsArray` function as if it returns the lines variable, and to make that work, we use the keyword `await`. After that, we continue the code as if the `readFileAsArray` call was synchronous.

To get things to run, we execute the async function. This is very simple and more readable. To work with errors, we need to wrap the async call in a `try`/`catch` statement.

With this async/await feature, we did not have to use any special API (like .then and .catch). We just labeled functions differently and used pure JavaScript for the code.

We can use the async/await feature with any function that supports a promise interface. However, we can’t use it with callback-style async functions (like setTimeout for example).

### The EventEmitter Module ###

The EventEmitter is a module that facilitates communication between objects in Node. EventEmitter is at the core of Node asynchronous event-driven architecture. Many of Node’s built-in modules inherit from EventEmitter.

The concept is simple: emitter objects emit named events that cause previously registered listeners to be called. So, an emitter object basically has two main features:

- Emitting name events.
- Registering and unregistering listener functions.

To work with the EventEmitter, we just create a class that extends EventEmitter.

```
class MyEmitter extends EventEmitter {

}
```

Emitter objects are what we instantiate from the EventEmitter-based classes:

```
const myEmitter = new MyEmitter();
```

At any point in the lifecycle of those emitter objects, we can use the emit function to emit any named event we want.

```
myEmitter.emit('something-happened');
```

Emitting an event is the signal that some condition has occurred. This condition is usually about a state change in the emitting object.

We can add listener functions using the `on` method, and those listener functions will be executed every time the emitter object emits their associated name event.

#### Events !== Asynchrony ####

Let’s take a look at an example:

```
const EventEmitter = require('events');

class WithLog extends EventEmitter {
  execute(taskFunc) {
    console.log('Before executing');
    this.emit('begin');
    taskFunc();
    this.emit('end');
    console.log('After executing');
  }
}

const withLog = new WithLog();

withLog.on('begin', () => console.log('About to execute'));
withLog.on('end', () => console.log('Done with execute'));

withLog.execute(() => console.log('*** Executing task ***'));
```

Class `WithLog` is an event emitter. It defines one instance function `execute`. This `execute` function receives one argument, a task function, and wraps its execution with log statements. It fires events before and after the execution.

To see the sequence of what will happen here, we register listeners on both named events and finally execute a sample task to trigger things.

Here’s the output of that:

```
Before executing
About to execute
*** Executing task ***
Done with execute
After executing
```

What I want you to notice about the output above is that it all happens synchronously. There is nothing asynchronous about this code.

- We get the “Before executing” line first.
- The `begin` named event then causes the “About to execute” line.
- The actual execution line then outputs the “*** Executing task ***” line.
- The `end` named event then causes the “Done with execute” line
- We get the “After executing” line last.

Just like plain-old callbacks, do not assume that events mean synchronous or asynchronous code.

This is important, because if we pass an asynchronous `taskFunc` to `execute`, the events emitted will no longer be accurate.

We can simulate the case with a `setImmediate` call:

```
// ...

withLog.execute(() => {
  setImmediate(() => {
    console.log('*** Executing task ***')
  });
});
```

Now the output would be:

```
Before executing
About to execute
Done with execute
After executing
*** Executing task ***
```

This is wrong. The lines after the async call, which were caused the “Done with execute” and “After executing” calls, are not accurate any more.

To emit an event after an asynchronous function is done, we’ll need to combine callbacks (or promises) with this event-based communication. The example below demonstrates that.

One benefit of using events instead of regular callbacks is that we can react to the same signal multiple times by defining multiple listeners. To accomplish the same with callbacks, we have to write more logic inside the single available callback. Events are a great way for applications to allow multiple external plugins to build functionality on top of the application’s core. You can think of them as hook points to allow for customizing the story around a state change.

#### Asynchronous Events ####

Let’s convert the synchronous sample example into something asynchronous and a little bit more useful.

```
const fs = require('fs');
const EventEmitter = require('events');

class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    this.emit('begin');
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err);
      }

      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    });
  }
}

const withTime = new WithTime();

withTime.on('begin', () => console.log('About to execute'));
withTime.on('end', () => console.log('Done with execute'));

withTime.execute(fs.readFile, __filename);
```

The `WithTime` class executes an `asyncFunc` and reports the time that’s taken by that `asyncFunc` using `console.time` and `console.timeEnd` calls. It emits the right sequence of events before and after the execution. And also emits error/data events to work with the usual signals of asynchronous calls.

We test a `withTime` emitter by passing it an `fs.readFile` call, which is an asynchronous function. Instead of handling file data with a callback, we can now listen to the data event.

When we execute this code , we get the right sequence of events, as expected, and we get a reported time for the execution, which is helpful:

```
About to execute
execute: 4.507ms
Done with execute
```

Note how we needed to combine a callback with an event emitter to accomplish that. If the `asynFunc` supported promises as well, we could use the async/await feature to do the same:

```
class WithTime extends EventEmitter {
  async execute(asyncFunc, ...args) {
    this.emit('begin');
    try {
      console.time('execute');
      const data = await asyncFunc(...args);
      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    } catch(err) {
      this.emit('error', err);
    }
  }
}
```

I don’t know about you, but this is much more readable to me than the callback-based code or any .then/.catch lines. The async/await feature brings us as close as possible to the JavaScript language itself, which I think is a big win.

#### Events Arguments and Errors ####

In the previous example, there were two events that were emitted with extra arguments.

The error event is emitted with an error object.

```
this.emit('error', err);
```

The data event is emitted with a data object.

```
this.emit('data', data);
```

We can use as many arguments as we need after the named event, and all these arguments will be available inside the listener functions we register for these named events.

For example, to work with the data event, the listener function that we register will get access to the data argument that was passed to the emitted event and that data object is exactly what the `asyncFunc` exposes.

```
withTime.on('data', (data) => {
  // do something with data
});
```

The `error` event is usually a special one. In our callback-based example, if we don’t handle the error event with a listener, the node process will actually exit.

To demonstrate that, make another call to the execute method with a bad argument:

```
class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err); // Not Handled
      }

      console.timeEnd('execute');
    });
  }
}

const withTime = new WithTime();

withTime.execute(fs.readFile, ''); // BAD CALL
withTime.execute(fs.readFile, __filename);
```

The first execute call above will trigger an error. The node process is going to crash and exit:

```
events.js:163
      throw er; // Unhandled 'error' event
      ^
Error: ENOENT: no such file or directory, open ''

```

The second execute call will be affected by this crash and will potentially not get executed at all.

If we register a listener for the special `error` event, the behavior of the node process will change. For example:

```
withTime.on('error', (err) => {
  // do something with err, for example log it somewhere
  console.log(err)
});
```

If we do the above, the error from the first execute call will be reported but the node process will not crash and exit. The other execute call will finish normally:

```
{ Error: ENOENT: no such file or directory, open '' errno: -2, code: 'ENOENT', syscall: 'open', path: '' }
execute: 4.276ms
```

Note that Node currently behaves differently with promise-based functions and just outputs a warning, but that will eventually change:

```
UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 1): Error: ENOENT: no such file or directory, open ''

DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

The other way to handle exceptions from emitted errors is to register a listener for the global `uncaughtException` process event. However, catching errors globally with that event is a bad idea.

The standard advice about `uncaughtException` is to avoid using it, but if you must do (say to report what happened or do cleanups), you should just let the process exit anyway:

```
process.on('uncaughtException', (err) => {
  // something went unhandled.
  // Do any cleanup and exit anyway!

  console.error(err); // don't do just that.

  // FORCE exit the process too.
  process.exit(1);
});
```

However, imagine that multiple error events happen at the exact same time. This means the `uncaughtException` listener above will be triggered multiple times, which might be a problem for some cleanup code. An example of this is when multiple calls are made to a database shutdown action.

The `EventEmitter` module exposes a `once` method. This method signals to invoke the listener just once, not every time it happens. So, this is a practical use case to use with the uncaughtException because with the first uncaught exception we’ll start doing the cleanup and we know that we’re going to exit the process anyway.

#### Order of Listeners ####

If we register multiple listeners for the same event, the invocation of those listeners will be in order. The first listener that we register is the first listener that gets invoked.

```
// प्रथम
withTime.on('data', (data) => {
  console.log(`Length: ${data.length}`);
});

// दूसरा
withTime.on('data', (data) => {
  console.log(`Characters: ${data.toString().length}`);
});

withTime.execute(fs.readFile, __filename);
```

The above code will cause the “Length” line to be logged before the “Characters” line, because that’s the order in which we defined those listeners.

If you need to define a new listener, but have that listener invoked first, you can use the `prependListener` method:

```
// प्रथम
withTime.on('data', (data) => {
  console.log(`Length: ${data.length}`);
});

// दूसरा
withTime.prependListener('data', (data) => {
  console.log(`Characters: ${data.toString().length}`);
});

withTime.execute(fs.readFile, __filename);
```

The above will cause the “Characters” line to be logged first.

And finally, if you need to remove a listener, you can use the `removeListener` method.

That’s all I have for this topic. Thanks for reading! Until next time!


*If you found this article helpful, please click the💚 below. Follow me for more articles on Node and JavaScript.*

If you have any questions about this article or any other article I wrote, find me on [this slack account](https://slack.jscomplete.com/) (you can invite yourself) and ask in the #questions room.

I create online courses for [Pluralsight](https://www.pluralsight.com/search?q=samer+buna&amp;categories=course) and [Lynda](https://www.lynda.com/Samer-Buna/7060467-1.html) . My most recent courses are [Getting Started with React.js](https://www.pluralsight.com/courses/react-js-getting-started) , [Advanced Node.js](https://www.pluralsight.com/courses/nodejs-advanced) , and [Learning Full-stack JavaScript](https://www.lynda.com/Express-js-tutorials/Learning-Full-Stack-JavaScript-Development-MongoDB-Node-React/533304-2.html) .

---

> [掘金翻译计划](https://github.com/xitu/gold-miner) 是一个翻译优质互联网技术文章的社区，文章来源为 [掘金](https://juejin.im) 上的英文分享文章。内容覆盖 [Android](https://github.com/xitu/gold-miner#android)、[iOS](https://github.com/xitu/gold-miner#ios)、[React](https://github.com/xitu/gold-miner#react)、[前端](https://github.com/xitu/gold-miner#前端)、[后端](https://github.com/xitu/gold-miner#后端)、[产品](https://github.com/xitu/gold-miner#产品)、[设计](https://github.com/xitu/gold-miner#设计) 等领域，想要查看更多优质译文请持续关注 [掘金翻译计划](https://github.com/xitu/gold-miner)。
