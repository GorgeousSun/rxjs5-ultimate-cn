# Subject (主体)

Subject 有着双重特性，它同时拥有 [Observer](observer.md) 和 [Observable](observable-anatomy.md) 的行为。因此，以下是可能的：

发出值

```javascript
subject.next( 1 )
subject.next( 2 )
```

订阅值

```javascript
const subscription = subject.subscribe( (value) => console.log(value) )
```

总结以下，它可以进行以下操作：

```javascript
next([value])
error([error message])
complete()
subscribe()
unsubscribe()
```

## 作为代理

`Subject` 可以作为代理，也就是从另一个流接收值，而 `Subject` 的订阅者可以监听另外的这个流。

```javascript
let source$ = Rx.Observable.interval(500).take(3);
const proxySubject = new Rx.Subject();
let subscriber = source$.subscribe( proxySubject );

proxySubject.subscribe( (value) => console.log('proxy subscriber', value ) );

proxySubject.next( 3 );
```

所以本质上 `subject` 监听了 `source$`

但是它还可以增加自己的贡献

```javascript
proxySubject.next( 3 )  // 发出3，然后是0 1 2 ( 异步的 )
```

**陷阱** - 任何在订阅创建之前执行的 `next()` 就会丢失。下面会有其他类型的 Subject 可以解决这个问题。

### 业务场景

那么这有什么有趣的呢？当数据到达时，它可以监听一些数据源，同时还能够发出自己的数据，并且都能到达同一个订阅者。以总线方式在组件之间进行通信的能力是我能想到的最显而易见的用例。组件1可以通过 `next()` 来放置它的值，组件2可以订阅，反之亦然，组件2可以发出值，组件1可以订阅。

```javascript
sharedService.getDispatcher = function(){
   return subject;
}

sharedService.dispatch = function(value){
  subject.next(value)
}
```

## ReplaySubject

原型:

```javascript
new Rx.ReplaySubject([bufferSize], [windowSize], [scheduler])
```

示例:

```javascript
let replaySubject = new Rx.ReplaySubject( 2 );

replaySubject.next( 0 );
replaySubject.next( 1 );
replaySubject.next( 2 );

//  1, 2
let replaySubscription = replaySubject.subscribe((value) => {
    console.log('replay subscription', value);
});
```

哇，这发生了什么，第一个数字怎么了？因为 `.next()` 是在订阅创建之前执行的，按常理来说应该会丢失才对。但在这里使用的是 `ReplaySubject`，我们有机会把已经发出的值保存在缓存之中。在这个案例中，`ReplaySubject` 创建后，缓存已被决定为保存两个值。

我们来解释下它是如何工作的：

```javascript
replaySubject.next( 3 )
let secondSubscriber( (value) => console.log(value) ) // 2,3
```

**陷阱** - 当 `.next()` 操作发生时，缓存的大小以及创建订阅的时间都很重要。

在上面的示例中，已经演示了如何在构造函数中使用 `bufferSize` 参数来使用构造函数。然而还有一个 `windowSize` 参数可以用来指定值应该在缓存中保存多久。把它设置为 `null` 的话将永久保存在缓存中。

### 业务场景

`ReplaySubject` 的业务场景很容易就想到。你获取一些数据并想让应用记住最新获取的数据，同时获取的内容可能只在一段时间内是有效的，并且在保留足够的时间后会清除缓存。

## AsyncSubject

```javascript
let asyncSubject = new Rx.AsyncSubject();
asyncSubject.subscribe(
    (value) => console.log('async subject', value),
    (error) => console.error('async error', error),
    () => console.log('async completed')
);

asyncSubject.next( 1 );
asyncSubject.next( 2 );
```

看这个示例，我们期望发出的值是1，2，没错吧？错！不会发出任何值除非执行了 `complete()`

```javascript
asyncSubject.next( 3 )
asyncSubject.complete()

// 发出 3
```

`complete()` 需要执行，而不在乎它之前的操作是成功还是失败

```javascript
asyncSubject( 3 )
asyncSubject.error('err')
asyncSubject.complete()

// 'err' 作为最后的操作会被发出
```

### 业务场景

当你关心流结束前的最后状态是值还是错误时适合使用 `AsyncSubject`，注意不是通常所说的最后发出的状态，而是**关闭流前**的最后状态。这里的状态我指的是值或错误。

## BehaviourSubject

这个 Subject 发出最初的值，通常所发出的值和你可以检查最后发出的值。

方法:

```
next()
complete()
constructor([start value])
getValue()
```

```javascript
let behaviorSubject = new Rx.BehaviorSubject(42);

behaviorSubject.subscribe((value) => console.log('behaviour subject',value) );
console.log('Behaviour current value',behaviorSubject.getValue());
behaviorSubject.next(1);
console.log('Behaviour current value',behaviorSubject.getValue());
behaviorSubject.next(2);
console.log('Behaviour current value',behaviorSubject.getValue());
behaviorSubject.next(3);
console.log('Behaviour current value',behaviorSubject.getValue());

// 发出 42
// Behaviour current value 42
// 发出 1
// Behaviour current value 1
// 发出 2
// Behaviour current value 2
// 发出 3
// Behaviour current value 3
```

### 业务场景

这与 `ReplaySubject` 很像。但还是略微不同，我们可以利用默认值/起始值，如果在第一个值开始到达之前需要一段时间，我们可以显示初始值。我们可以检查最新的发出值，当然也可以监听已发出的一切。所以把 `ReplaySubject` 当作是 **长期记忆**而把 `BehaviourSubject` 当作是有默认行为的短期记忆。
