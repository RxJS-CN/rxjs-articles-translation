# [译] RxJS Observable 与 Promises 和 Async-Await 交互

> 原文链接: [https://medium.com/@benlesh/rxjs-observable-interop-with-promises-and-async-await-bebb05306875](https://medium.com/@benlesh/rxjs-observable-interop-with-promises-and-async-await-bebb05306875)

不时地会有人问我关于如何与 RxJS 配合使用 async 函数或 promises，还有更糟的，我被告之“事实”的真相是 async-await 和 Observables 并不能“在一起使用”。RxJS 从一开始就具备与 Promises 的高度互操作性。希望这篇文章能对此有所启发。

## 如果可以接收 Observable，就可以接收 Promise

例如，如果使用 switchMap，你可以返回 Promise 来替代，就像返回 Observable 那样。以下这些都是有效的:

```javascript
// Observable: 每1秒发出自增数值乘以100，共发出10次
const source$ = Observable.interval(1000)
  .take(10)
  .map(x => x * 100);
/**
 * 返回 promise，它等待 `ms` 毫秒并发出 "done" 
 */
function promiseDelay(ms) {
  return new Promise(resolve => {
    setTimeout(() => resolve('done'), ms);
  });
}

// 在 switchMap 中使用 promiseDelay
source$.switchMap(x => promiseDelay(x)) // 正常运行
  .subscribe(x => console.log(x)); 

source$.switchMap(promiseDelay) // 更简洁了
  .subscribe(x => console.log(x)); 

// 或者使用 takeUntil
source$.takeUntil(doAsyncThing('hi')) // 完全可以运行
  .subscribe(x => console.log(x))

// 或者类似这样的奇怪组合
Observable.of(promiseDelay(100), promiseDelay(10000)).mergeAll()
  .subscribe(x => console.log(x))
```

## 使用 defer 使得返回 Promise 的函数可以重试

如果你可以访问创建 promise 的函数，你可以使用 `Observable.defer()` 来包装它，以使 Observable 可以在报错时进行重试。

```javascript
function getErroringPromise() {
  console.log('getErroringPromise called');
  return Promise.reject(new Error('sad'));
}

Observable.defer(getErroringPromise)
  .retry(3)
  .subscribe(x => console.log);

// 输出 "getErroringPromise called" 4次 (开始1次 + 3次重试), 然后报错
```

## 使用 defer() 定义使用 async-await 的 Observable

事实证明， defer 是个非常强大的小工具。你可以使用它，基本上是直接使用 async 函数，它会创建一个发出返回值及完成的 Observable 。

```javascript
Observable.defer(async function() {
  const a = await promiseDelay(1000).then(() => 1);
  const b = a + await promiseDelay(1000).then(() => 2);
  return a + b + await promiseDelay(1000).then(() => 3);
})
.subscribe(x => console.log(x)) // 输出 7
```

## 使用 forEach 订阅 Observable 以在 async-await 中创建并发任务

这是 RxJS 中较少使用的功能，它来自 TC39 Observable 提议。订阅 Observable 可不止一种方式！ `subscribe` 是订阅 Observable 的传统方式，它返回用来取消数据流的 `Subscription` 对象。而 `forEach` 以一种不可取消的方式来订阅 Observable ，它接收一个函数用于每个值，并返回 Promise，该 Promise 体现了 Observable 的完成和错误路径。

```javascript
const click$ = Observable.fromEvent(button, 'click');
/**
 * 等待10次按钮点击，然后使用 fetch 将第10次点击的时间戳发送给端点
 */
async function doWork() {
  await click$.take(10)
    .forEach((_, i) => console.log(`click ${i + 1}`));
  return await fetch(
    'notify/tenclicks',
    { method: 'POST', body: Date.now() }
  );
}
```

## 使用 toPromise() 和 async/await 将 Observable 最后发出的值作为 Promise 发出

`toPromise` 函数实际上是有些巧妙的，因为它并不是真正的“操作符”，而是以一种 RxJS 特定的方式来订阅 Observable 并将其包装成一个 Promise 。一旦 Observable 完成，Promise 便会 resolve Observable 最后发出的值。这意味着如果 Observable 发出值 “hi” 然后等待10秒才完成，那么返回的 Promise 会等待10秒才 resolve “hi” 。如果 Observable 一直不完成，那么 Promise 便永远不会 resolve 。

**注意: 使用 toPromise() 是一种反模式，除非当你正在处理预期为 Promise 的 API， 比如 async-await**

```javascript
const source$ = Observable.interval(1000).take(3); // 0, 1, 2
// 等待3秒，然后输出 "2"
// 因为 Observable 需要3秒才能完成，而 interval 发出从0开始自增的数字
async function test() {
  console.log(await source$.toPromise());
}
```

## Observables 和 Promises 能很好地一起使用

不可否认地，如果你的目标是响应式编程，那么大多数时间里你可能想要使用 Observable ，但是 RxJS 尝试去尽可能地满足大众需求，毕竟当下 Promises 还是很受欢迎的。此外，在 async 函数中使用 RxJS Observables 和 forEach，为管理并发性和在 async-await 中“只能正常运行”的任务开启了大量有趣的可能性。

**想学习更多 RxJS 知识， 我可以亲自教学或选择在线学习，尽在[http://rxworkshop.com](http://rxworkshop.com)!**