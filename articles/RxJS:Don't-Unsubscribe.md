# [译] RxJS: 别取消订阅

> 原文链接: [https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87](https://medium.com/@benlesh/rxjs-dont-unsubscribe-6753ed4fda87)

好吧...，只是不要过多地使用取消订阅操作。

> 译者注: Ben 也当了回标题党 (￣▽￣)"

我经常受邀去帮助他人调试他们的 RxJS 代码的问题，或者弄清楚如何构建一个大量使用 RxJS 的异步操作的应用。当这么做的时候，我通常都会看到同样的问题一遍又一遍反复地出现，人们维护大量的 subscription 对象及其处理方法。开发者会一成不变地使用一个 Observable 发起3个 HTTP 请求，然后保留3个 subscription 对象，当某个事件发生时，会调用事件所对应的 subscription 对象。

我能理解这是如何发生的。人们习惯性地使用 `addEventListener` N 次，然后还需要做一些清理工作，他们不得不调用 `removeEventListener` N 次。对 subscription 对象也进行同样的处理感觉是自然而然的，在很大程度上来说这是没错，但是还有更好的方式。**保留过多的 subscription 对象是一个信号，你命令式地管理了你的 subscriptions ，并且没有利用 Rx 的强大之处。**

## 命令式的 subscription 管理看起来应该是怎样的

以这个虚构的组件为例 (我是故意使这个组件即非 React，也非 Angular，而是更通用一些):

```javascript
class MyGenericComponent extends SomeFrameworkComponent {
 updateData(data) {
  // 在此执行一些框架指定的操作来更新你的组件
 }

 onMount() {
  this.dataSub = this.getData()
   .subscribe(data => this.updateData(data));

  const cancelBtn = this.element.querySelector(‘.cancel-button’);
  const rangeSelector = this.element.querySelector(‘.rangeSelector’);

  this.cancelSub = Observable.fromEvent(cancelBtn, ‘click’)
   .subscribe(() => {
    this.dataSub.unsubscribe();
   });

  this.rangeSub = Observable.fromEvent(rangeSelector, ‘change’)
   .map(e => e.target.value)
   .subscribe((value) => {
    if (+value > 500) {
      this.dataSub.unsubscribe();
    }
   });
 }

 onUnmount() {
  this.dataSub.unsubscribe();
  this.cancelSub.unsubscribe();
  this.rangeSub.unsubscribe();
 }
}
```

在上面的示例中，你可以看到在 `onUnmount()` 方法中我手动地调用3个 subscription 对象的 `unsubscribe` 方法。当某人点击了取消按钮时我调用了一次 `this.dataSub.unsubscribe()`，然后当用户设置的范围选择大于500时，我又调用了一次 `this.dataSub.unsubscribe()` 。500是我想停止流的一个阈值。(我不知道为什么这样做，这是个奇怪的组件)

这段代码的丑陋之处就在于在这个相当简单的示例中，我在多个地方命令式地管理了取消订阅。

使用这种方式的唯一真正的优势就是性能。因为你使用了更少的抽象来完成工作，它执行起来可能会更快一些。这在大多数网络应用中不太可能有明显的效果，所以我不认为这是值得担心的。

作为选择，你可以总是将多个 subscriptions 组合成单个 subscription，通过创建一个父 subscription 并将其他所有的 subscriptions 作为子 subscription 添加进来。但是在一天结束的时候，你仍然在做着同样的事情，你可能已经错过了这种更好的方式。

## 使用 takeUntil 来构成你的 subscription 管理

现在我们来做同样的基础示例，只是我们使用了 RxJS 的 `takeUntil` 操作符:

```javascript
class MyGenericComponent extends SomeFrameworkComponent {
 updateData(data) {
  // 在此执行一些框架指定的操作来更新你的组件
 }

 onMount() {
   const data$ = this.getData();
   const cancelBtn = this.element.querySelector(‘.cancel-button’);
   const rangeSelector = this.element.querySelector(‘.rangeSelector’);
   const cancel$ = Observable.fromEvent(cancelBtn, 'click');
   const range$ = Observable.fromEvent(rangeSelector, 'change').map(e => e.target.value);
   
   const stop$ = Observable.merge(cancel$, range$.filter(x => x > 500))
   this.subscription = data$.takeUntil(stop$).subscribe(data => this.updateData(data));
 }

 onUnmount() {
  this.subscription.unsubscribe();
 }
}
```

首先你可能注意到的就是代码更少了。这只是其中一个好处。另外你可能注意到就是在这我有了一个组合而成的 `stop$` 事件流，它用来停止数据流。
这意味着当我决定我想要添加另外一个停止流的条件时，比如说定时器，我可以简单地合并一个新的 Observable 到 `stop$` 中。另外一件很明显的事情就是我命令式地管理的 subscription 对象只有一个。没有办法解决这个问题，因为这是函数式编程和面向对象的世界相会的地方。毕竟 JavaScript 是一种命令式语言，我们必须在某些情况下与命令式之外的编程世界相遇。

另外一个优势是这种方式实际上完成了 observable 。这意味着会有完成事件，任何你想要杀掉你的 observalbe 的时候都可以来处理它。
如果你只是调用返回的 subscription 对象上的 `unsubscribe` 方法，那么则没有办法通知你取消订阅的发生。然而如果你使用 `takeUntil` (或下面所列出的其他操作符)，会通过你的完成处理方法通知你 observable 已经停止。

我要指出的最后一个优势就是实际上你通过在一处调用 `subscribe` 来“连通一切”，这是有优势的，因为有了规矩，在你的代码中找到开始你的 subscriptions 的位置将变得容易得多。记住，observables 不会做任何事直到你订阅了它们，所以 subscription 所在之处是重要的代码片段。

在 RxJS 语义方面存在一个缺点，但相对于其他优点，这几乎不值一提。语义上的不足之处在于，完成 observable 是一个信号，即生产者想要告诉消费者已经完成，而取消订阅是消费者告诉生产者它不再关心数据了。

这种方式与只是命令式地调用 `unsubscribe` 之间还略微有点性能上的区别。然而，在大多数应用中，这种性能打击几乎是不明显的。

## 其他操作符

还有一些其他操作符也可以以一种更 “Rx” 的方式来停止流。我建议至少检验以下操作符:

  * take(n): 在停止 observable 前发出 N 个值。
  * takeWhile(predicate): 通过断言函数来测试发出的值，如果一旦函数返回 `false`，则完成 observable 。
  * first(): 发出首个值和完成通知。
  * first(predicate): 根据断言函数检查每个值，如果函数返回 `true`，则发出值和完成通知。

## 总结: 使用 takeUntil、takeWhile，及其他

你应该尽可能的使用像 `takeUntil` 这样的操作符来管理你的 RxJS subscriptions 。作为经验法则，如果你在一个组件中管理发现2个或2个以上的 subscriptions，你应该问自己是否能将它们组合的更好。

  * 可组合性更强
  * 当你停止流时会触发完成事件
  * 通常代码更少
  * 需要管理的更好
  * 实际 subscription 的点更少 (因为调用 `subscribe` 少了)
