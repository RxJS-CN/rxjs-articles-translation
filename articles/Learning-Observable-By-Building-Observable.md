# Learning Observable By Building Observable

> 原文链接: [https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87](https://medium.com/@benlesh/learning-observable-by-building-observable-d5da57405d87)

Frequently via social media, or in person at events, I’m asked questions about “hot” vs “cold” observables, or whether or not an observable is “multicast” or “unicast”. People seem to be really mystified by what they feel to be the dark inner-workings of `Rx.Observable`. When asked to describe an observable, people say things like, “They’re streams” or, “They’re like promises”. In fact, I’ve said these things on many occasions and even during public talks.

The comparison to promises is necessary, but unfortunate. Given that both promises and observables are async primitives, and promises are already widely used and familiar to the JavaScript community, it’s generally a great starting point. Comparing promise’s `then` to observable’s `subscribe`, showing differences in eager vs lazy execution, showing cancellation and reuse of observables, etc. It’s a handy way to introduce beginners to observables.

There’s one problem: Observables are more different from promises than they are similar. Promises are always multicast. Promise resolution and rejection is always async. When people approach observables as though they’re similar to a promises, they expect these things, but they’re not always true. Observables are sometimes multicast. Observables are usually async. I blame myself a little, I’ve helped perpetuate this misunderstanding.

## Observable is just a function that takes an observer and returns a function

If you really want to understand observable, you could simply write one. It’s not as hard as it sounds, honestly. An observable, boiled down to it’s smallest parts, is no more than a specific type of function with a specific purpose.

### Shape:

  * A function
  * That accepts an observer: An object with `next`, `error` and `complete` methods on it.
  * And returns a cancellation function

### Purpose:

To connect the observer to something that produces values (a producer), and return a means to tear down that connection to the producer. The observer is really a registry of handlers that can be pushed values over time.

### Basic Implementation:

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

[(You can try it out on JSBin here)](http://jsbin.com/yazedu/1/edit?js,console,output)

As you can see, there’s not a lot to it. It’s a fairly simple contract.

## Safe Observers: Make Observers Great Again

When we talk about RxJS or Reactive programming, generally observables get top billing. But the observer implementation is actually the workhorse of this type of reactive programming. Observables are inert. They’re just functions. They sit there until you `subscribe` to them, they set up your observer, and they’re done, back to being boring old functions waiting to be called. The observers on the other hand, stay active and listen for events from your producers.

You can subscribe to the observable now with any Plain-Old JavaScript Object (POJO) that has `next`, `error` and `complete` methods on it, but the POJO observer that you’ve used to subscribe to the observable is really just the beginning. In RxJS 5, we need to provide some guarantees for you. Below are just a few of the more important guarantees:

### Observer Guarantees

  1. If you pass an Observer doesn’t have all of the methods implemented, that’s okay.
  2. You don’t want to call `next` after a `complete` or an `error`
  3. You don’t want anything called if you’ve unsubscribed.
  4. Calls to `complete` and `error` need to call unsubscription logic.
  5. If your `next`, `complete` or `error` handler throws an exception, you want to call your unsubscription logic so you don’t leak resources.
  6. `next`, `error` and `complete` are all actually optional. You don’t have to handle every value, or errors or completions. You might just want to handle one or two of those things.

In order to accomplish this, we need to wrap the anonymous observer you provide in a “SafeObserver” that enforces the above guarantees. Because of guarantee #2 above, we need to track whether or not `complete` or `error` have been called. Because of #3, we need to let our SafeObserver know when the consumer has signaled it wants to unsubscribe. Finally, because of #4, our SafeObserver is actually going to need to know about the unsubscription logic so it can call it when `complete` or `error` is called.

So if we wanted to do this with our ad-hoc function implementation of an observable above, it’s going to get gross… Here is just a snippet [from a JSBin you can play with to show you just how gross](http://jsbin.com/kezejiy/2/edit?js,console,output). I didn’t want the (very primitive) SafeObserver implementation (in the JSBin) in this example, because it would eat the entire article, but here’s just our observable using the SafeObserver:

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

## Designing Observable: Ergonomic Observer Safety

Having observables as a class/object enables us to easily apply a SafeObserver to passed anonymous observers (and handler functions if you like the `subscribe(fn, fn, fn)` signature in RxJS) and provide better ergonomics for the developer-user. By handling the creation of a SafeObserver inside Observable’s `subscribe` implementation, Observables can again be defined in the simplest possible way:

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

You’ll notice that the above snippet looks almost identical to our first example. It’s easier to read and easier to understand. I’ve augmented our [JSBin example to show the minimal Observable implementation](http://jsbin.com/depeka/edit?js,console).

## Operators: Also Just Functions

An “operator” in RxJS is little more than a function that takes a source observable, and returns a new observable that will subscribe to that source observable when you subscribe to it. We can implement a basic, standalone operator like this [(again in JSBin)](http://jsbin.com/xavaga/2/edit?js,console,output):

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

The most important thing to notice about what this operator is doing: When you subscribe to its returned observable, it’s creating a `mapObserver` to do the work and chaining `observer` and `mapObserver` together. Building operator chains is really just creating a template for wiring observers together on subscription.

## Designing Observable: Making Operator Chains Pretty

If we were to have all of our operators implemented as standalone functions like the example above, chaining our operators gets a bit ugly:

```javascript
map(map(myObservable, (x) => x + 1), (x) => x + 2);
```

Now imagine the above, nested five or six operators deep with more complicated operators that have more arguments. Basically unreadable.

You could go with a simple `pipe` implementation [(as suggested by Texas Toland)](https://twitter.com/AppShipIt/status/701806357012471809) that reduces over an array of these operators to produce your final observable, but that’s going to mean writing more complicated operators that return functions [(JSBin example of that here)](http://jsbin.com/vipuqiq/6/edit?js,console,output). It’s also not going to look perfect either:

```javascript
pipe(myObservable, map(x => x + 1), map(x => x + 2));
```

Ideally, we’d be able to chain things in a more natural way like so:

```javascript
myObservable.map(x => x + 1).map(x => x + 2);
```

Fortunately, we already have an Observable class onto which we can put our operators to support this sort of chaining behavior. It doesn’t add any complexity to the operator implementation, but it does come at the cost of “junking up the prototype” I suppose, once you add enough operators, of which there are many — perhaps too many. Here is what our map operator looks like when added to our Observable implementation’s prototype [(with JSBin)](http://jsbin.com/quqibe/edit?js,console,output):

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

Now we have that nicer syntax we were going for. There are other benefits to this approach as well that are a little more advanced. For example, we can subclass Observable for specific types of observables (observables wrapping a Promise or a set of static values for example) and make optimizations for our operators by overriding them for that subclass.

## TLDR: Observables are a function that take an observer and return a function

Keep in mind, after reading everything above, that all of this was designed around a simple function. **Observables are a function that take an observer and return a function**. Nothing more, nothing less. If you write a function that takes an observer and returns a function, is it async or sync? Neither. It’s a function. The behavior of any function all depends on how it’s implemented. So when dealing with an Observable, treat it like a function reference you’re passing around, not some magic, stateful alien type. When you’re building your operator chains, what you’re really doing is composing a function that sets up a chain of observers that are linked together and pass values through to your observer.

> NOTICE: The example Observable implementations still return functions above, where RxJS and the es-observable spec return Subscription objects. Subscription objects are a much better design, but I could write a whole article about that. I kept it to unsubscription functions just to keep the examples simple.