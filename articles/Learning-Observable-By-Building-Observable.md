# [译] 通过构建 Observable 来学习 Observable

> 原文链接: [https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87](https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87)

通过社交媒体或在活动现场，我经常会被问到关于“热的” vs “冷的” observables，或者 observable 究竟是“多播”还是“单播”。对于 `Rx.Observable`，人们觉得它内部的工作原理完全是黑魔法，这令他们感到十分困惑。当被问及如何描述 observable 时，人们会说：“他们是流”或“他们类似于 promises ”。事实上，我在很多场合甚至在公开演讲中都谈论过这些内容。

与 promises 进行比较是必要的，但也是不幸的。鉴于 promieses 和 observables 都是异步基本类型 ( async primitives )，并且 promises 已被 JavaScript 社区广泛使用和熟悉，这通常是一个很好的起点。将 promise 的 `then` 与 observable 的 `subscribe` 进行比较，promise 是立即执行，而 observable 是惰性执行，是可取消、可复用的，等等。这是向初学者介绍 observables 的理想方式。

但这样有个问题: Observables 与 promises 的不同点要远多于它们之间的相同点。Promises 永远是多播的。Promise 的解析 ( resolution ) 和拒绝 ( rejection ) 永远是异步的。当人们处理 observables 时，仿佛就是在处理 promises，所以他们期望两者的行为也是相似的，但这不并总是对的。Observables 有时是多播的。Observables 通常是异步的。我也有些自责，因为我助长了这种误解的蔓延。

## Observables 只是一个函数，它接收 observer 并返回函数

如果你真的想要理解 observable，你可以自己写个简单的。这并没有听上去那么困难，真心的。将 observable 归结为最精简的部分，无外乎就是某种特定类型的函数，该函数有其针对性的用途。

### 模型:

  * 函数
  * 接收 observer: observer 是有 `next`、`error` 和 `complete` 方法的对象
  * 返回一个可取消的函数

### 目的:

将观察者 ( observer ) 与生产者 ( producer ) 连接，并返回一种手段来拆解与生产者之间的连接。观察者实际上是处理函数的注册表，处理函数可以随时间推移推送值。

### 基本实现:

```javascript
function myObservable(observer) {
    const datasource = new DataSource();
    datasource.ondata = (e) => observer.next(e);
    datasource.onerror = (err) => observer.error(err);
    datasource.oncomplete = () => observer.complete();
    return () => {
        datasource.destroy();
    };
}
``` 

[(你可以点击这里进行在线调试)](http://jsbin.com/yazedu/1/edit?js,console,output)

如你所见，并没有太多东西，只是一个相当简单的契约。

## 安全的观察者: 让观察者变得更好

当谈论到 RxJS 或响应式编程时，通常 observables 出现的频率是最高的。但实际上，观察者的实现才是这类响应式编程的中流砥柱。Observables 是惰性的。它们只是函数而已。它们什么也不做直到你 `subscribe` 它们，它们装配好了观察者，然后就完事了，与沉闷的老式函数并无差别，等待着被调用而已。另一方面，观察者保持活跃状态并监听来自生产者的事件。

你可以使用任何有 `next`、`erro` 和 `complete` 方法的简单 JavaScript 对象 (POJO) 来订阅 observable，但你所用来订阅 observable 的 POJO 观察者真的只是个开始。在 RxJS 5中，我们需要为你提供一些保障。下面罗列了一些重要的保障:

### 观察者保障

  1. 如果你传递的观察者完全没有以上所述的三个方法，也是可以的。
  2. 你不想在 `complete` 或 `error` 之后调用 `next` 。
  3. 如果取消订阅了，那么你不想任何方法被调用。
  4. 调用 `complete` 和 `error` 需要调用取消订阅逻辑。
  5. 如果 `next` 、`complete` 或 `error` 处理方法抛出异常，你想要调用取消订阅逻辑，以确保不会泄露资源。
  6. `next`、`error` 和 `complete` 实际上都是可选的。你无需处理每个值、错误或完成。你可能只是想要处理其中一二。

为了完成列表中的任务，我们需要将你提供的匿名观察者包装在 “SafeObserver” 中以实施上述保障。因为上面的#2，我们需要追踪 `complete` 或 `error` 是否被调用过。因为#3，我们需要使 SafeObserver 知道消费者何时要取消订阅。最后，因为#4，SafeObserver 实际上需要了解取消订阅逻辑，这样当 `complete` 或 `error` 被调用时才可以调用它。

如果我们想要用上面临时实现的 observable 函数来做这些的话，会变得有些粗糙... 这里有个 [JSBin 代码片段](http://jsbin.com/kezejiy/2/edit?js,console,output)，你可以看看并感受下有多粗糙。我并没有想要在这个示例中实现非常正宗的 SafeObserver，因为那么占用整篇文章篇幅，下面是我们的 observable，这次它使用了 SafeObserver:

```javascript
function myObservable(observer) {
   const safeObserver = new SafeObserver(observer);
   const datasource = new DataSource();
   datasource.ondata = (e) => safeObserver.next(e);
   datasource.onerror = (err) => safeObserver.error(err);
   datasource.oncomplete = () => safeObserver.complete();
 
   safeObserver.unsub = () => {
       datasource.destroy();
   };
 
   return safeObserver.unsubscribe.bind(safeObserver);
}
```

## 设计 Observable: 确保观察者安全

将 observables 作为类/对象使我们能够轻松地将 SafeObserver 应用于传入的匿名观察者(和处理函数，如果你喜欢 RxJS 中的 `subscribe(fn, fn, fn)` 签名的话) 并为开发人员提供更好的开发体验。通过在 Observable 的 `subscribe` 实现中处理 SafeObserver 的创建，Observables 可以再次以最简单的方式来定义:

```javascript
const myObservable = new Observable((observer) => {
    const datasource = new DataSource();
    datasource.ondata = (e) => observer.next(e);
    datasource.onerror = (err) => observer.error(err);
    datasource.oncomplete = () => observer.complete();
    return () => {
        datasource.destroy();
    };
});
```

你会注意到上面的代码片段与第一个示例看起来几乎一样。但它更容易阅读，也更容易理解。我扩展了 [JSBin 示例来展示 Observable 的最小化实现](http://jsbin.com/depeka/edit?js,console)。

## 操作符: 同样只是函数

RxJS 中的“操作符”只不过是接收源 observable 的函数，并返回一个新的 observable，当你订阅该 observable 时它会订阅源 observable 。我们可以实现一个基础、独立的操作符，如[这个在线 JSBin 示例所示](http://jsbin.com/xavaga/2/edit?js,console,output):

```javascript
function map(source, project) {
  return new Observable((observer) => {
    const mapObserver = {
      next: (x) => observer.next(project(x)),
      error: (err) => observer.error(err),
      complete: () => observer.complete()
    };
    return source.subscribe(mapObserver);
  });
}
```

最重要的是要注意此操作符在做什么: 当你订阅它返回的 observable 时，它会创建 `mapObserver` 来完成工作并将 `observer` 和 `mapObserver` 连接起来。
构建操作符链实际上只是创建一个将观察者与订阅 ( subscription ) 连接起来的模板。

## 设计 Observable: 更优雅的操作符链

如果我们所有的操作符都是像上面示例中那样用独立函数实现的话，将操作符链接起来会有些难看:

```javascript
map(map(myObservable, (x) => x + 1), (x) => x + 2);
```

可以想象一下上面的代码，嵌套了5个或6个更复杂的操作符将会产生更多的参数。导致代码完全不可读。

你可以使用简单的 `pipe` 实现 [(正如 Texas Toland 所建议的那样)](https://twitter.com/AppShipIt/status/701806357012471809)，它会对操作符数组进行累加以生成最终的 observable，但这意味着要编写更复杂的、返回函数的操作符，[(点击这里查看 JSBin 示例)](http://jsbin.com/vipuqiq/6/edit?js,console,output)。这同样没有使得一切变得完美:

```javascript
pipe(myObservable, map(x => x + 1), map(x => x + 2));
```

理想的是我们能够将操作符以一种更自然的方式链接起来，比如这样:

```javascript
myObservable.map(x => x + 1).map(x => x + 2);
```

幸运的是，我们的 Observable 类已经支持这种操作符的链式行为。它不会给操作符的实现代码任何额外的复杂度，但它的代价是破坏了我所提倡的“摒弃原型 ( prototype )”，一旦添加了足够你使用的操作符，原型中的方法或许就太多了。点击 [(这里的 JSBin 示例)](http://jsbin.com/quqibe/edit?js,console,output) 查看我们添加到 Observable 实现原型中的 map 操作符:

```javascript
Observable.prototype.map = function (project) {
    return new Observable((observer) => {
        const mapObserver = {
            next: (x) => observer.next(project(x)),
            error: (err) => observer.error(err),
            complete: () => observer.complete()
        };
        return this.subscribe(mapObserver);
    });
};
```

现在我们拥有了更好的语法。这种方法还有其他好处，也更高级。例如，我们可以将 Observable 子类化为特定类型的 observables (例如包装 Promise 或一组静态值的 observables)，并通过覆盖它们来对我们的运算符进行优化。

## TLDR: Observable 是函数，它接收 observer 并返回函数

记住，阅读以上所有内容后，所有的这一切都是围绕一个简单的函数设计的。**Observables 是函数，它接收 observer 并返回函数**。仅此而已。如果你编写了一个函数，它接收 observer 并返回函数，那它是异步的，还是同步的？都不是，它就是个函数。任何函数的行为都完全取决于它是如何实现的。所以，当处理 Observable 时，就像你所传递的函数引用那样对待它，而不是一些所谓的魔法，有状态的外星类型。当你构建操作符链式，你真正要做的是构成一个函数，该函数会设置链接在一起的观察者链，并将值传递给观察者。

> 注意: 示例中的 Observable 实现仍然返回的是函数，而 RxJS 和 es-observable 规范返回的是 Subscription 对象。Subscription 对象是一种更好的设计，但我又得写一整篇文章来讲它。所以我只保留了它的取消订阅功能，以保持本文中所有示例的简单性。