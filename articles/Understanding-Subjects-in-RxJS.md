# [译] 理解 RxJS 中 Subjects

> 原文链接: [https://netbasal.com/understanding-subjects-in-rxjs-55102a190f3](https://netbasal.com/understanding-subjects-in-rxjs-55102a190f3)

在开始阅读前，本文需要一些 Rx 中的基础知识。

比如说你有两个 Observables 订阅:

```javascript
const interval$ = Rx.Observable.interval(1000);

interval$.subscribe(console.log);

setTimeout(() => {
  interval$.subscribe(console.log);
}, 2000);
```

每次使用新的观察者 ( observer ) 调用 `subscribe` 方法时，你都在创建新的执行。
你可以把它想象成一个执行两次的普通函数。例如:

```javascript
function interval() {
  setInterval(() => console.log('..'), 1000);
}

interval();

setTimeout(() => {
  interval();
}, 2000);
```

我们创建了定时器，它们是彼此独立的。但如果你需要两个观察者获得同样的事件呢？

这时候你就需要使用 Rx 中 的 Subject 。

## 什么是 Subjects ？

Subject 既是 Observable ，又是 Observer 。

  1. Observer - 拥有 next、error 和 complete 方法。
  2. Observable - 拥有 Observable 的所有操作符，并且你可以订阅它。

Subject 可以在源 Observable 和多个观察者之间充当桥梁或代理，使得多个观察者可以**共享**同一个 Observable 执行。

我们在第一个示例中来看下如何**共享**同一个执行:

```javascript
const interval$ = Rx.Observable.interval(1000);
const subject = new Rx.Subject();
interval$.subscribe(subject);
```

首先，我们创建了一个新的 Subject 。现在请记住， Subject 还是一个观察者，那么观察者可以做什么呢？它们可以使用 next()、error() 和 completed() 方法来监听 Observables 。如果你将 subject 打印到控制台，你可以看到 subject 拥有如下这些方法。

![Subject Methods](../assets/Understanding-Subjects-in-RxJS-Subject-Method.png)

所以在我们这个案例中，Subject 正在观察 interval observable 。简单点说，当你有新值的时候要让我知道。

现在让我们继续下一部分。

```javascript
subject.subscribe(val => console.log(`First observer ${val}`));

setTimeout(() => {
  subject.subscribe(val => console.log(`Second observer ${val}`))
}, 2000);
```

Subject 同时还是 Observable ，我们可以使用 Observables 做什么呢？我们可以订阅它们。

在这个案例中，我们订阅了 Subject 。但 Subject 会给我们什么值呢？

如果你还记得 Subject 正在观察 interval Observables ，所以每次 interval 发送值给 Subject ，Subject 都会将值发送给它的所有观察者。

所以 Subject 充当代理和桥梁的作用，正因为如此，才只有一个执行。

## **_哦，我从 interval Observable 那得到一个新值，然后我将这个值传递给我的所有观察者 (监听者)_**

别忘了每个 Subject 同时还是一个观察者，所以我们可以使用观察者的方法: next()、error()、complete() 。我们来看个示例:

```javascript
const input = document.querySelector('input[type=text]');
const p = document.querySelector('p');

input.addEventListener('input', event => {
  subject.next(event.target.value);
});

subject.subscribe(val => {
  p.textContent = val;
});
```

我们可以订阅 Subject ，并且我们还可以手动触发 next() 方法。当你调用 next() 方法时，每个订阅者都会收到值。(你还可以触发 error() 和 complete())

我们已经学习了 Rx 中最简单的 Subject 。还有更多类型的 Subjects，它们可以解决更复杂的情况。它们是 BehaviorSubject、AsyncSubject、ReplaySubject 。

最常用的是 BehaviorSubject ，你可以在我的最新[文章](https://netbasal.com/angular-2-persist-your-login-status-with-behaviorsubject-45da9ec43243)中更多的了解它。

就这些了!
