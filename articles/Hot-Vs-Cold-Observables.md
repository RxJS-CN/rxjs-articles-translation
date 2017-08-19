# [译] 热的 Vs 冷的 Observables

> 原文链接: [https://medium.com/@benlesh/hot-vs-cold-observables-f8094ed53339](https://medium.com/@benlesh/hot-vs-cold-observables-f8094ed53339)

**TL;DR: 当不想一遍又一遍地创建生产者( producer )时，你需要热的 Observable 。**

## 冷的是指 Observable 创建了生产者

```javascript
// 冷的
var cold = new Observable((observer) => {
  var producer = new Producer();
  // observer 会监听 producer
});
```

## 热的是指 Observable 复用生产者

```javascript
// 热的
var producer = new Producer();
var hot = new Observable((observer) => {
  // observer 会监听 producer
});
```

## 深入了解发生了什么...

我最新的文章[通过构建 Observable 来学习 Observable](https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87) 主要是为了说明 Observable 只是函数。这篇文章的目标是为了揭开 Observable 自身的神秘面纱，但它并没有真正深入到 Observable 让初学者最容易困惑的问题: “热”与“冷”的概念。

### Observables 只是函数而已！

**Observables 是将观察者和生产者联系起来的函数。** 仅此而已。它们并不一定要建立生产者，它们只需建立观察者来监听生产者，并且通常会返回一个拆卸机制来删除该监听器。

### 什么是“生产者”?

生产者是 Observable 值的来源。它可以是 Web Socket、DOM 事件、迭代器或在数组中循环的某种东西。基本上，这是你用来获取值的任何东西，并将它们传递给 `observe.next（value）` 。

## 冷的 Observables: 在**内部**创建生产者

如果底层的生产者是在订阅期间**创建并激活的**，那么 Observable 就是“冷的”。这意味着，如果 Observables 是函数，而生产者是通过**调用该函数**创建并激活的。

  1. 创建生产者
  2. 激活生产者
  3. 开始监听生产者
  4. 单播

下面的示例 Observable 是“冷的”，因为它在订阅函数(在订阅该 Observable 时调用)中创建并监听了 WebSocket :

```javascript
const source = new Observable((observer) => {
  const socket = new WebSocket('ws://someurl');
  socket.addEventListener('message', (e) => observer.next(e));
  return () => socket.close();
});
```

所以任何 `source` 的订阅都会得到自己的 WebSocket 实例，当取消订阅时，它会关闭 socket 。这意味着 `source` 是真正的单播，因为生产者只会发送给一个观察者。[这是用来阐述概念的基础 JSBin 示例](http://jsbin.com/wabuguy/1/edit?js,output)。

## 热的 Observables: 在**外部**创建生产者

如果底层的生产者是在 订阅¹ 外部创建或激活的，那么 Observable 就是“热的”。

  1. 共享生产者的引用
  2. 开始监听生产者
  3. 多播(通常情况下²)

如果我们沿用上面的示例并将 WebSocket 的创建移至 Observable 的外部，那么 Observable 就会变成“热的”:

```javascript
const socket = new WebSocket('ws://someurl');
const source = new Observable((observer) => {
  socket.addEventListener('message', (e) => observer.next(e));
});
```

现在任何 `source` 的订阅都会共享同一个 WebSocket 实例。它实际上会多播给所有订阅者。但还有个小问题: 我们不再使用 Observable 来运行拆卸 socket 的逻辑。这意味着像错误和完成这样的通知不再会为我们来关闭 socket ，取消订阅也一样。所以我们真正想要的其实是使“冷的” Observable 变成“热的”。[这是用来展示基础概念的 JSBin 示例](http://jsbin.com/godawic/edit?js,output)。

### 为什么要变成“热的” Observable ？

从上面展示冷的 Observable 的第一个示例中，你可以发现所有冷的 Observables 可能都会些问题。就拿一件事来说，如果你不止一次订阅了 Observable ，而这个 Observable 本身创建一些稀缺的资源，比如 WebSocket 连接，你不想一遍又一遍地创建这个 WebSocket 连接。实际上真的很容易创建了一个 Observable 的多个订阅而却没有意识到。假如说你想要在 WebSocket 订阅外部过滤所有的“奇数”和“偶数”值。在此场景下最终你会创建两个订阅:

```javascript
source.filter(x => x % 2 === 0)
  .subscribe(x => console.log('even', x));
source.filter(x => x % 2 === 1)
  .subscribe(x => console.log('odd', x));
```

## Rx Subjects

在将“冷的” Observable 变成“热的”之前，我们需要介绍一个新的类型: Rx Subject 。它有如下特性:

  1. 它是 Observable 。它的结构类似 Observable 并拥有 Observable 的所有操作符。
  2. 它是 Observer 。它作为 Observer 的鸭子类型。当作为 Observable 被订阅时，将作为 Observer 发出 `next` 的任何值。
  3. 它是多播的。所有通过 `subscribe()` 传递给它的 Observers 都会被添加到内部的观察者列表。
  4. 当它完成时，就是完成了。Subjects 在取消订阅、完成或发生错误后无法被复用。
  5. 它通过自身传递值。需要重申下第2点。如果 `next` 值给它，值会从它 observable 那面出来。

Rx 中的 Subject 之所以叫做 “Subject” 是因为上面的第3点。在 GoF (译注: 大名鼎鼎的《设计模式》一书) 的观察者模式中，“Subjects” 通常是有 `addObserver` 的类。在这里，我们的 `addObserver` 方法就是 `subscribe` 。[这是用来展示 Rx Subject 的基础行为的 JSBin 示例](http://jsbin.com/muziva/1/edit?js,output)。

## 将冷的 Observable 变成热的

了解了上面的 Rx Subject 后，我们可以使用一些功能性的程序将任何“冷的” Observable 变成“热的”:

```javascript
function makeHot(cold) {
  const subject = new Subject();
  cold.subscribe(subject);
  return new Observable((observer) => subject.subscribe(observer));
}
```

我们的新方法 `makeHot` 接收任何冷的 Observable 并通过创建由所得到的 Observable 共享的 Subject 将其变成热的。[这是用来演示 JSBin 示例](http://jsbin.com/ketodu/1/edit?js,output)。

还有一点问题，就是没有追踪源的订阅，所以当想要拆卸时该如何做呢？我们可以添加一些引用计数来解决这个问题:

```javascript
function makeHotRefCounted(cold) {
  const subject = new Subject();
  const mainSub = cold.subscribe(subject);
  let refs = 0;
  return new Observable((observer) => {
    refs++;
    let sub = subject.subscribe(observer);
    return () => {
      refs--;
      if (refs === 0) mainSub.unsubscribe();
      sub.unsubscribe();
    };
  });
}
```

现在我们有了一个热的 Observable ，当它的所有订阅结束时，我们用来引用计数的 `refs` 会变成0，我们将取消冷的源 Observable 的订阅。[这是用来演示的 JSBin 示例](http://jsbin.com/lubata/1/edit?js,output)。

## 在 RxJS 中, 使用 `publish()` 或 `share()`

你可能不应该使用上面提及的任何 `makeHot` 函数，而是应该使用像 `publish()` 和 `share()` 这样的操作符。将冷的 Observable 变成热的有很多种方式和手段，在 Rx 中有高效和简洁的方式来完成此任务。关于 Rx 中可以做此事的各种操作符可以写上一整篇文章，但这不是文本的目标。本文的目标是巩固概念，什么是“热的”和“冷的” Observable 以及它们的真正意义。

在 RxJS 5 中，操作符 `share()` 会产生一个热的，引用计数的 Observable ，它可以在失败时重试，或在成功时重复执行。因为 Subjects 一旦发生错误、完成或取消订阅，便无法复用，所以 `share()` 操作符会重复利用已完结的 Subjects，以使结果 Observable 启用重新订阅。

[这是 JSBin 示例，演示了在 RxJS 5 中使用 `share()` 将源 Observable 变热，以及它可以重试](http://jsbin.com/mexuma/1/edit?js,output)。

## “暖的” Observable

鉴于上述一切，人们能够看到 Observable 是怎样的，它只是一个函数，实际上可以同时是“热的”和“冷的”。或许它观察了两个生产者？一个是它创建的而另一个是它复用的？这可能不是个好主意，但有极少数情况下可能是必要的。例如，多路复用的 WebSocket 必须共享 socket ，但同时发送自己的订阅并过滤出数据流。

## “热的”和“冷的”都关乎于生产者

如果在 Observable 中复用了生产者的共享引用，它就是“热的”，如果在 Observable 中创建了新的生产者，它就是“冷的”。如果两者都做了…。那它到底是什么？我猜是“暖的”。

#### 注释

¹ (注意: 生产者在订阅内部“激活”，直到未来某个时间点才“创建”出来，这种做法是有些奇怪，但使用代理的话，这也是可能的。) 通常“热的” Observables 的生产者是在订阅外部创建和激活的。

² 热的 Observables 通常是多播的，但是它们可能正在监听一个只支持一个监听器的生产者。在这一点上将其称之为“多播”有点勉强。