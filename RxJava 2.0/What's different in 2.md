本人读完了 `RxJava 2.x` ：[What's different in 2.0](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0) 后，将值得重点注意的变化进行了翻译和归纳，如果错误请批评指正。

## Maven仓库依赖地址和包名的变更 ：   
* Maven 仓库地址在 `io.reactivex.rxjava2:rxjava:2.x.y` 下  
* 类在包名为 `io.reactivex` 下  
* JavaDoc 在 <http://reactivex.io/RxJava/2.x/javadoc/>

## Nulls
Rxjava 2.x 不再接受 null，下面的情况会立即抛出 `NullPointerException`
  
```java
Observable.just(null);

Single.just(null);

Observable.fromCallable(() -> null)
    .subscribe(System.out::println, Throwable::printStackTrace);

Observable.just(1).map(v -> null)
    .subscribe(System.out::println, Throwable::printStackTrace);
```
    
也就是说 `Observable<Null>` 也不被允许，作为代替，我们可以定义一个`Observable<Object>`传入一个不相关的值，如下：

```java
enum Irrelevant { INSTANCE; }

Observable<Object> source = Observable.create((ObservableEmitter<Object> emitter) -> {
   System.out.println("Side-effect 1");
   emitter.onNext(Irrelevant.INSTANCE);

   System.out.println("Side-effect 2");
   emitter.onNext(Irrelevant.INSTANCE);

   System.out.println("Side-effect 3");
   emitter.onNext(Irrelevant.INSTANCE);
});

source.subscribe(e -> { /* Ignored. */ }, Throwable::printStackTrace);
```

## Observable and Flowable

Rxjava 2.x 一个大的改进就是解决了 1.x 中不支持 `backpressure` 问题  
解决方案是将过去的 `Observable` 重新设计为：  

* 不支持 backpressure 的 `io.reactivex.Observable`  :
  现在 `Observable`会将没有消费的数据保存在内存中直到`OutOfMemoryError`而不会抛出`MissBackpressureException`
* 支持 backpressure 的 `io.reactivex.Flowable`  
  在 `Flowable.create()`创建时指定背压策略 : 如`BackpressureStrategy.DROP`

#### 两种类型的使用场景

#### Observable:
* 流需要处理的元素不超过1K 或者不会产生 OOME
* 响应 Mouse Touch 相关GUI操作的事件

#### Flowable:
* 流的超过10K个元素的流
* 一系列可能阻塞消费的操作（解析文件、网络请求、数据库操作...）

## Single Completable Maybe

2.x 整个架构都按照 Reactive-Streams 规范设计, 所以现在的基本消费者类型改为了接口

#### Single:  只关心调用成功后对数据的处理
` onSubscribe (onSuccess | onError)? `
#### Completable：只关心是否调用成功
` onSubscribe (onComplete | onError)?.`
#### Maybe：可能出现无数据或只有一个数据的情况，所以onSuccess()和onComplete()只会调用其中一个
` onSubscribe (onSuccess | onError | onComplete)?`  

## Base reactive interfaces
与 Reactive-Streams 中 `Flowable` extends `Publisher<T>` 风格一样，其他基本响应类也有类似的基础接口  

```java
interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}

interface SingleSource<T> {
    void subscribe(SingleObserver<? super T> observer);
}

interface CompletableSource {
    void subscribe(CompletableObserver observer);
}

interface MaybeSource<T> {
    void subscribe(MaybeObserver<? super T> observer);
}
```
所以现在的操作符也接受 `Publisher` 和 `XSource`:

```java
Flowable<R> flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper);

Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper);
```

## Subjects and Processors
同样是为了解决 `backpressure` 问题把 `Subjects` 分为了 `Subjects` 和 `Processors`

`io.reactivex.subjects.AsyncSubject`  
`io.reactivex.subjects.BehaviorSubject`  
`io.reactivex.subjects.PublishSubject`  
`io.reactivex.subjects.ReplaySubject`   `io.reactivex.subjects.UnicastSubject`  
不支持 backpressure , 属于 Observable 系列

`io.reactivex.processors.AsyncProcessor`
`io.reactivex.processors.BehaviorProcessor` `io.reactivex.processors.PublishProcessor` `io.reactivex.processors.ReplayProcessor` `io.reactivex.processors.UnicastProcessor`  
支持 backpressure ，属于 Flowable 系列

## TestSubject
1.x 的 `TestSubject` 已被丢弃，现在通过 `TestScheduler` , `PublishProcessor`/`PublishSubject` 和 `observeOn(TestScheduler)`代替

## SerializedSubject
`SerializedSubject`   
由 `Subject.toSerialized()` 和 `FlowableProcessor.toSerialized()` 代替

## Other classes
`rx.observables.GroupedObservable` 由`io.reactivex.observables.GroupedObservable<T>` 和`io.reactivex.flowables.GroupedFlowable<T>`代替

## Functional interfaces
functional 接口 默认定义了 `throws Exception`  
不需要再内部`try-catch`

## Actions Functions
符合 java8 命名规范

* `Func` -> `Function` , `Action0`/`Action1`/`Action2` -> `Action`/`Consumer`/`BiConsumer`  
* 删除了 `Action3-9`/`Func3-9`  
由 `Action<Object[]>`/`Function<Object[],R>`代替

## Subscriber Subscription

* Reactive-Streams 规范中已经定义了 `Subscriber` 接口  
所以以前的 `Subscriber` 类的职能现在由 `Subscriber` 接口的实现类代替:  
`DefaultSubscriber`, `ResourceSubscriber`,`DisposableSubscriber`（以及它们的`XObserver` 变体）  
因为以上继承了`Disposable` 所以也支持通过 `dispose()` 来断开对信号的监听

```
ResourceSubscriber<Integer> subscriber = new ResourceSubscriber<Integer>() {
    @Override
    public void onStart() {
        request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(Integer t) {
        System.out.println(t);
    }

    @Override
    public void onError(Throwable t) {
        t.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("Done");
    }
};

Flowable.range(1, 10).delay(1, TimeUnit.SECONDS).subscribe(subscriber);

subscriber.dispose();
```

* `CompositeSubscription` -> `CompositeDisposable `  
* `subscribe()` 不返回值  
* `subscribWith()` 返回 `CompositeDisposable`  
* `onCompleted()` -> `onComplete()`
* `request()` 决定 `subscriber`最大接受多少个事件

## Schedulers

* 默认不变的线程:`computation` `io` `newThread` `trampoline`
* 移除:`immediate` -> `tranmpoline`
* 移除: `test()` -> `new TestScheduler()`
* 启动 `Scheduler` 无需再 `createWorker`
* `new()` 现在接受 `TimeUnit` 参数

## Entering the reactive world
现在调用 `Observalble.create()` 更安全

## Leaving the reactive world

从响应式流中离开的方式：

```
List<Integer> list = Flowable.range(1, 100).toList().blockingGet(); // toList() returns Single

Integer i = Flowable.range(100, 100).blockingLast();
```

另一个关于 `rx.Subscriber` 和 `org.reactivestreams.Subscriber`的重大变化是
`Subscribers` 和 `Observers` 内部不在允许 throws 任何东西除了致命异常（参见 ` Exceptions.throwIfFatal()`）,所以下面的代码现在不合法：

```
Subscriber<Integer> subscriber = new Subscriber<Integer>() {
    @Override
    public void onSubscribe(Subscription s) {
        s.request(Long.MAX_VALUE);
    }

    public void onNext(Integer t) {
        if (t == 1) {
            throw new IllegalArgumentException();
        }
    }

    public void onError(Throwable e) {
        if (e instanceof IllegalArgumentException) {
            throw new UnsupportedOperationException();
        }
    }

    public void onComplete() {
        throw new NoSuchElementException();
    }
};

Flowable.just(1).subscribe(subscriber);
```
(Observer, SingleObserver, MaybeObserver and CompletableObserver 同理)

如果必须要这样 throws 可以选择使用 `safeSubscribe()`或 `subscribe(Consumer<T>, Consumer<Throwable>, Action)`的相关 Consumer 重载

## Operator differences
关于操作符的调整主要是为了适配以上的变化，调整命名/参数/返回值  
详见官方 [Wiki](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0#operator-differences) 表格

## 总结
总的来说，我认为RxJava 2.x 主要做了三点更新  

* 对 `backpressure` 问题进行了修正（产生了大量关联的修正）  
* 按照 `Reactive-Streams` 规范对整个架构进行了重新设计
* 一些的其它零散更新

所以说虽然基本思想没有变化但还是需要重新理解学习的