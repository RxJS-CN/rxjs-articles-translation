# [译] RxJS 调度器入门

> 原文链接: [https://staltz.com/primer-on-rxjs-schedulers.html](https://staltz.com/primer-on-rxjs-schedulers.html)

RxJS 中的调度器 ( Schedulers ) 是用来控制事件发出的顺序和速度的(发送给观察者的)。它还可以控制订阅 ( Subscriptions ) 的顺序。为了不搞得太理论化，先考虑下这个示例:

```javascript
const a$ = Rx.Observable.of(1, 2);
const b$ = Rx.Observable.of(10);

const c$ = Rx.Observable.combineLatest(a$, b$, (a, b) => a + b);

c$.subscribe(c => console.log(c));
```

你觉得控制台输出的结果是什么呢？大多数人会猜是:

```
11
12
```

因为首先 `a$` 中`1`会和 `b$` 中的`10`配对，然后 `a$` 中的`2`和 `b$` 中的`10`配对。

事实上，出现在控制台中的是:

```
12
```

`1 + 10` 的组合并没有发生。原因是 Observables `a$` 和 `b$` 都是“同步的”，它们会尽可能快地执行。那么事件发出的顺序到底是怎样的呢？答案是不确定的，它可能是以下任意一种:

  * 1, 2, 10
  * 1, 10, 2
  * 10, 1, 2

在这种顺序不确定的情况下，我们应该描述出事件的发出顺序是怎样的。这就是调度器所做的事。默认情况下，RxJS 使用所谓的**递归调度器**。下面是它的工作原理:

  1. c$ 被订阅
  1. combineLatest 的第一个输入流 a$ 被订阅
  1. a$ 发出值 1
  1. combineLatest 将 1 作为 a$ 的最新值进行保存
  1. a$ 发出值 2
  1. combineLatest 将 2 作为 a$ 的最新值进行保存
  1. combineLatest 的第二个输入流 b$ 被订阅
  1. b$ 发出值 10
  1. combineLatest 将 10 作为 b$ 的最新值进行保存
  1. combineLatest 现在同时拥有了 a$ 和 b$ 的值，因此它发出值 2 + 10

发出的顺序为 `1, 2, 10` 。最有意思的是在 `b$` 被订阅前， 将 `a$` 的所有事件都尽快地发出了。RxJS 使用这种调度器作为默认调度器出于两点原因:

  * 使用此策略性能的整体表现更好
  * 在调试工具中更易于进行堆栈跟踪

然而，可以通过使用不同的调度器来自定义事件发出的顺序及速度。我们在 `a$` 上使用 `asap` 调度器来让其“慢下来”:

```javascript
// const a$ = Rx.Observable.of(1, 2);
const a$ = Rx.Observable.from([1, 2], Rx.Scheduler.asap); // 新代码
const b$ = Rx.Observable.of(10);

const c$ = Rx.Observable.combineLatest(a$, b$, (a, b) => a + b);

c$.subscribe(c => console.log(c))
```

`from` 操作符的第二个参数是调度器，用来自定义事件的发出。`asap` 调度器使用 [setImmediate](https://github.com/YuzuJS/setImmediate) 来安排任务尽快运行，但不是同步的。代码改变后，控制台会输出:

```
11
12
```

因为内部运行顺序如下:

  1. c$ 被订阅
  1. combineLatest 的第一个输入流 a$ 被订阅
  1. **combineLatest 的第二个输入流 b$ 被订阅**
  1. b$ 发出值 10
  1. combineLatest 将 10 作为 b$ 的最新值进行保存
  1. a$ 发出值 1
  1. combineLatest 将 1 作为 a$ 的最新值进行保存
  1. combineLatest 现在同时拥有了 a$ 和 b$ 的值，因此它发出值 1 + 10
  1. a$ 发出值 2
  1. combineLatest 将 2 作为 a$ 的最新值进行保存
  1. combineLatest 发出值 2 + 10

发出的顺序为 `10, 1, 2` 。为了得到另外一种发出顺序，可以为 `b$` 也自定义调度器:

```javascript
const a$ = Rx.Observable.from([1, 2], Rx.Scheduler.asap);
// const b$ = Rx.Observable.of(10);
const b$ = Rx.Observable.from([10], Rx.Scheduler.asap); // 新代码

const c$ = Rx.Observable.combineLatest(a$, b$, (a, b) => a + b);

c$.subscribe(c => console.log(c));
```

现在发出的顺序为 `1, 10, 2`，因为运行顺序如下:

  1. c$ 被订阅
  1. combineLatest 的第一个输入流 a$ 被订阅
  1. combineLatest 的第二个输入流 b$ 被订阅
  1. a$ 发出值 1
  1. combineLatest 将 1 作为 a$ 的最新值进行保存
  1. b$ 发出值 10
  1. combineLatest 将 10 作为 b$ 的最新值进行保存
  1. combineLatest 现在同时拥有了 a$ 和 b$ 的值，因此它发出值 1 + 10
  1. a$ 发出值 2
  1. combineLatest 将 2 作为 a$ 的最新值进行保存
  1. combineLatest 发出值 2 + 10

调度器还可以让事件的发出变得更快，同时保持发出的顺序不变。例如，RxJS 的 `TestScheduler` 可以使 `Observable.interval(1000).take(10)` 被订阅时进行同步执行，而不需要花费10秒钟来完成:

```javascript
Rx.Observable.interval(1000, new Rx.TestScheduler()).take(10)
```

`TestScheduler` 是在 RxJS 内部使用的 ([参见 filter 的测试用例](https://github.com/ReactiveX/rxjs/blob/a44d8e5d610fa2c419d0515c456bc924aa1fa095/spec/operators/filter-spec.ts#L37-L44))，它使得成百上千个时间相关的测试代码飞快地运行，但有一些像 [Rx Sandbox](https://github.com/kwonoj/rx-sandbox/) 这样的工具和积极的讨论来丰富此调度器的使用场景，使得在 RxJS 内部之外的地方也可以使用。

如果你也喜欢本文，可以考虑将此[推文](https://twitter.com/intent/tweet?original_referer=https%3A%2F%2Fstaltz.com%2Fprimer-on-rxjs-schedulers.html&text=Primer%20on%20RxJS%20Schedulers&tw_p=tweetbutton&url=https%3A%2F%2Fstaltz.com%2Fprimer-on-rxjs-schedulers.html&via=andrestaltz)分享给你的关注者。