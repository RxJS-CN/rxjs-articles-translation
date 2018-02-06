# [译] RxJS: 如何使用 refCount

> 原文链接: [https://blog.angularindepth.com/rxjs-how-to-use-refcount-73a0c6619a4e](https://blog.angularindepth.com/rxjs-how-to-use-refcount-73a0c6619a4e)

![Cover](../assets/How-To-Use-RefCount/header.jpeg)

照片取自 [Unsplash](https://unsplash.com/)，作者 [Mike Wilson](https://unsplash.com/photos/Tqcd9_j4Xjo) 。

在我的上篇文章 [ 理解 publish 和 share 操作符](./Understanding-The-Publish-And-Share-Operators.md) 中，只是简单介绍了 refCount 方法。在这篇文章中我们将深入介绍。

## refCount 的作用是什么？

简单回顾一下， RxJS 多播的基本心智模型包括: 一个源 observable，一个订阅源 observable 的 subject 和多个订阅 subject 的观察者。[`multicast`](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-multicast) 操作符封装了基于 subject 的基础结构并返回拥有 `connect` 和 `refCount` 方法的 `ConnectableObservable`。

顾名思义，`refCount` 返回的 observable 维护订阅者的引用计数。

当观察者订阅有引用计数的 observable 时，引用计数会增加，如果上一个引用计数为零的话，负责多播基础结构的 subject 会订阅源 observable 。而当观察者取消订阅时，引用计数则会减少，如果引用计数归零的话，subject 会取消源 observable 的订阅。

这种引用计数的行为有两种用途:

  * 当所有观察者都取消订阅后，自动取消 subject 对源 observable 的订阅
  * 当所有观察者都取消订阅后，自动取消 subject 对源 observable 的订阅，然后当再有观察者订阅该引用计数的 observable 时，subject 重新订阅源 observable 

我们来详细介绍每一种情况，然后建立一些使用 `refCount` 的通用指南。

## 使用 refCount 自动取消订阅

`publish` 操作符返回 `ConnectableObservable` 。调用 `ConnectableObservable` 的 `connect` 方法时，负责多播基础结构的 subject 会订阅源 observable 并返回 subscription (订阅)。subject 会保持对源 observable 的订阅直到调用 subscription 的 `unsubscribe` 方法。

我们来看下面的示例，观察者会接收一个值，然后(隐式地)取消对调用过 `publish` 的 observable 的订阅:

```ts
const source = instrument(Observable.interval(100));
const published = source.publish();
const a = published.take(1).subscribe(observer("a"));
const b = published.take(1).subscribe(observer("b"));
const subscription = published.connect();
```

本文中的示例都将使用下面的工具函数来让源 observable 具备日志功能，以及创建有名称的观察者:

```ts
function instrument<T>(source: Observable<T>) {
  return Observable.create((observer: Observer<T>) => {
    console.log("source: subscribing");
    const subscription = source
      .do(value => console.log(`source: ${value}`))
      .subscribe(observer);
    return () => {
      subscription.unsubscribe();
      console.log("source: unsubscribed");
    };
  }) as Observable<T>;
}

function observer<T>(name: string) {
  return {
    next: (value: T) => console.log(`observer ${name}: ${value}`),
    complete: () => console.log(`observer ${name}: complete`)
  };
}
```

示例的输出如下所示:

```
source: subscribing
source: 0
observer a: 0
observer a: complete
observer b: 0
observer b: complete
source: 1
source: 2
source: 3
...
```

两个观察者都只接收一个值然后完成，完成的同时取消对调用过 `publish` 的 observable 的订阅。但是，多播基础结构仍然保持着对源 observable 的订阅。

如果不想显示地执行取消订阅操作的话，可以使用 `refCount`:

```ts
const source = instrument(Observable.interval(100));
const counted = source.publish().refCount();
const a = counted.take(1).subscribe(observer("a"));
const b = counted.take(1).subscribe(observer("b"));
```

观察者订阅使用引用计数的 observable 的话，当引用计数归零时，负责多播的基础结构的 subject 会取消源 observable 的订阅，示例的输出如下所示:

```
source: subscribing
source: 0
observer a: 0
observer a: complete
observer b: 0
observer b: complete
source: unsubscribed
```

## 重新订阅已完成的 observables

当引用计数归零后，多播的基础结构除了取消源 observable 的订阅，当负责引用计数的 observable 再次发生订阅时，它还会重新订阅源 observable 。

我们使用下面的示例来看看当使用已完成的源 observable 时会发生什么:

```ts
const source = instrument(Observable.timer(100));
const counted = source.publish().refCount();
const a = counted.subscribe(observer("a"));
setTimeout(() => a.unsubscribe(), 110);
setTimeout(() => counted.subscribe(observer("b")), 120);
```

示例中使用 `timer` observable 作为源。它会等待指定的毫秒数后发出 `next` 和 `complete` 通知。还有两个观察者: `a` 在源 observable 完成后订阅，在源 observable 完成后取消订阅；`b` 在 `a` 取消订阅后订阅。

示例的输出如下:

```
source: subscribing
source: 0
observer a: 0
source: unsubscribed
observer a: complete
observer b: complete
```

当 `b` 订阅时，引用计数为零，所以多播的基础结构会期望 subject 重新订阅源 observable 。但是，由于 subject 已经收到了源 observable 的 `complete` 通知，并且 subject 是无法复用的，所以实际上并没有进行重新订阅，`b` 只能收到 `complete` 通知。

如果使用 `publishBehavior(-1)` 来代替 `publish()` 的话，输出类似，但会包含  `BehaviorSubject` 的初始值:

```
observer a: -1
source: subscribing
source: 0
observer a: 0
source: unsubscribed
observer a: complete
observer b: complete
```

同样的，`b` 还是只能收到 `complete` 通知。

如果使用 `publishReplay(1)` 来代替 `publish()` 的话，情况会有些变化，输出如下:

```
source: subscribing
source: 0
observer a: 0
source: unsubscribed
observer a: complete
observer b: 0
observer b: complete
```

同样的，这次也没有重新订阅源 observable，因为 subject 已经完成了。但是，已完成的 `ReplaySubject` 将通知重放给后来的订阅者，所以 `b` 能收到重放的 `next` 通知和 `complete` 通知。

如果使用 `publishLast()` 来代替 `publish()` 的话，情况又会有些不同，输出如下:

```
source: subscribing
source: 0
source: unsubscribed
observer a: 0
observer a: complete
observer b: 0
observer b: complete
```

同样的，依然没有重新订阅源 observable，因为 subject 已经完成了。但是，`AsyncSubject` 会将最后收到的 `next` 通知发给它的订阅者，所以 `a` 和 `b` 都收到的是 `next` 和 `complete` 通知。

综上所述，根据示例我们可以发现 `publish` 以及它的变种:

  * 当源 observable 完成时，负责多播基础结构的 subject 也会完成，而且这会阻止对源 observable 的重新订阅。
  * 当 `publish` 和 `publishBehavior` 与 `refCount` 一起使用时，后来的订阅者只会收到 `complete` 通知，这似乎并不是我们想要的效果。
  * 当 `publishReplay` 和 `publishLast` 与 `refCount` 一起使用时，后来的订阅者会收到预期的通知。

## 重新订阅未完成的 observables

我们已经看过了重新订阅已完成的源 observable 时会发生什么，现在我们再来看看重新订阅未完成的源 observable 是怎样一个情况。

这个示例中将使用 `interval` observable 来替代 `timer` observable，它会根据指定的时间间隔重复地发出包含自增数字的 `next` 通知:

```ts
const source = instrument(Observable.interval(100));
const counted = source.publish().refCount();
const a = counted.subscribe(observer("a"));
setTimeout(() => a.unsubscribe(), 110);
setTimeout(() => counted.subscribe(observer("b")), 120);
```

示例的输出如下所示:

```
source: subscribing
source: 0
observer a: 0
source: unsubscribed
source: subscribing
source: 0
observer b: 0
source: 1
observer b: 1
...
```

与使用已完成的源 observable 的示例不同的是，负责多播基础结构的 subject 能够被重新订阅，所以源 observable 可以产生新的订阅。`b` 所收到的 `next` 通知便是重新订阅的证据: 该通知包含数值0，因为重新订阅已经开启了全新的 `interval` 序列。

如果使用 `publishBehavior(-1)` 来代替 `publish()` 的话，情况会有所不同，输出如下所示:

```
observer a: -1
source: subscribing
source: 0
observer a: 0
source: unsubscribed
observer b: 0
source: subscribing
source: 0
observer b: 0
source: 1
observer b: 1
...
```

输出是类似的，可以清楚地看到重新订阅开启了全新的 `interval` 序列。但是，在收到 `interval` 的 `next` 通知前，`a` 还收到了包含 `BehaviorSubject` 初始值-1的 `next` 通知，`b` 会收到包含 `BehaviorSubject` 当前值0的 `next` 通知。

如果使用 `publishReplay(1)` 来代替 `publish()` 的话，情况又会有所不同，输出如下所示:

```
source: subscribing
source: 0
observer a: 0
source: unsubscribed
observer b: 0
source: subscribing
source: 0
observer b: 0
source: 1
observer b: 1
...
```

输出也是类似的，可以清楚地看到重新订阅开启了全新的 `interval` 序列。但是，`b` 在收到源 observable 的第一个 `next` 通知之前会收到重放的 `next` 通知。

综上所述，根据示例我们可以发现，当对未完成的源 observable 使用 `refCount` 时，`publish`、`publishBehavior` 和 `publishReplay` 的行为都如预期一般，没有让人出乎意料之处。

## shareReplay 的作用是什么？

在 RxJS [5.4.0](https://github.com/ReactiveX/rxjs/blob/master/CHANGELOG.md#540-2017-05-09) 版本中引入了 [shareReplay](https://github.com/ReactiveX/rxjs/blob/5.4.3/src/operator/shareReplay.ts#L7-L26) 操作符。它与 `publishReplay().refCount()` 十分相似，只是有一个细微的差别。

与 `share` 类似， `shareReplay` 传给 `multicast` 操作符的也是 subject 的工厂函数。这意味着当重新订阅源 observable 时，会使用工厂函数来创建出一个新的 subject 。但是，只有当前一个被订阅 subject 未完成的情况下，工厂函数才会返回新的 subject 。

`publishReplay` 传给 `multicast` 操作符的是 `ReplaySubject` 实例，而不是工厂函数，这是影响行为不同的原因。

对调用了 `publishReplay().refCount()` 的 observable 进行重新订阅，subject 会一直重放它的可重放通知。但是，对调用了 `shareReplay()` 的 observable 进行重新订阅，行为未必如前者一样，如果 subject 还未完成，会创建一个新的 subject 。所以区别在于，使用调用了 `shareReplay()` 的 observable 的话，当引用计数归零时，如果 subject 还未完成的话，可重放的通知会被冲洗掉。

## 不完全使用准则

根据我们看过的这些示例，可以归纳出如下使用准则:

  * `refCount` 可以与 `publish` 及其变种一起使用，从而自动地取消源 observable 的订阅。
  * 当使用 `refCount` 来自动取消已完成的源 observable 的订阅时，`publishReplay` 和 `publishLast` 的行为会如预期一样，但是，对于后来的订阅，`publish` 和 `publishBehavior` 的行为并没太大帮助，所以你应该只使用 `publish` 和 `publishBehavior` 来自动取消订阅。
  * 当使用 `refCount` 来自动取消未完成的源 observable 的订阅时，`publish`、`publishBehavior` 和 `publishRelay` 的行为都会如预期一样。
  * `shareReplay()` 的行为类似于 `publishReplay().refCount()`，在对两者进行选择时，应该根据在对源 observable 进行重新订阅时，你是否想要冲洗掉可重放的通知。

上面所描述的 `shareReplay` 的行为只适用于 RxJS 5.5 之前的版本。在 5.5.0 beta 中，`shareReplay` 做出了[变更](https://github.com/ReactiveX/rxjs/pull/2910): 当引用计数归零时，操作符不再取消源 observable 的订阅。

这项变化立即使得引用计数变得多余，因为只有当源 observable 完成或报错时，源 observable 的订阅才会取消订阅。这项变化也意味着只有在处理错误时，`shareReplay` 和 `publishReplay().refCount()` 才有所不同:

  * 如果源 observable 报错，`publishReplay().refCount()` 返回的 observable 的任何后来订阅者都将收到错误。
  * 但是，`shareReplay` 返回的 observable 的任何后来订阅者都将产生一个源 observable 的新订阅。
  