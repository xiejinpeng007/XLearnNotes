## Rxjava2.x 原理解析

RxJava 相信各位已经使用了很久，但大部分人在刚学习 RxJava 感叹切换线程的方便，调用逻辑清晰的同时，并不知道其中的原理，主要是靠记住运行的顺序。
随着我们设计出的 RxJava流 越来越复杂，一些复杂的问题并不能靠着记住的运行顺序就能解决。  
下面，就通过最常用的操作符的源码来看看所谓的`流`是什么运行的。  


首先我们用`Single`举例，设计一个最基本的 RxJava 流，只有一个 `Observable(ColdObservable)` 和`Obsever`：

```
Disposable disposable = Single.just("wtf")
              			.subscribe(it -> Log.i("subscribe", it));
```

上游发送一个`"wtf"` ，下游接受时将其打印出来。上游发送端使用 `Single.just` 作为创建方法,
看一下 `just()` 方法里做了什么。  

```
    public static <T> Single<T> just(final T item) {
        ObjectHelper.requireNonNull(item, "value is null");
        return RxJavaPlugins.onAssembly(new SingleJust<T>(item));
    }
    
    public static <T> Single<T> onAssembly(@NonNull Single<T> source) {
    Function<? super Single, ? extends Single> f = onSingleAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```


其中 `ObjectHelper.requireNonNull` 只是空检查。  
`RxJavaPlugins.onAssembly` 方法，这个方法其实就是通过一个全局的变量 `onSingleAssembly` 来对方法进行 Hook ，这一系列`xxxAssembly`全局变量默认为空，所以实际上当我们没有设置的时候其实 `just` 方法是直接返回了一个 新实例化的`SingleJust`对象。


再看看`SingleJust`内部：

```
public final class SingleJust<T> extends Single<T> {

    final T value;
    public SingleJust(T value) {
        this.value = value;
    }

    @Override
    protected void subscribeActual(SingleObserver<? super T> observer) {
        observer.onSubscribe(Disposables.disposed());
        observer.onSuccess(value);
    }

}

```

实例化的时候只是将值保存了下来，没有其它操作。  
下一步调用`subscribe()`来启动这个流`(ColdObservable)`，然后看看`subscribe`中做了什么：

```
    public final void subscribe(SingleObserver<? super T> subscriber) {
        ObjectHelper.requireNonNull(subscriber, "subscriber is null");
        subscriber = RxJavaPlugins.onSubscribe(this, subscriber);
        ObjectHelper.requireNonNull(subscriber, "subscriber returned by the RxJavaPlugins hook is null");

        try {
 	         //核心逻辑
            subscribeActual(subscriber);
        } catch (NullPointerException ex) {
            throw ex;
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            NullPointerException npe = new NullPointerException("subscribeActual failed");
            npe.initCause(ex);
            throw npe;
        }
    }
```

同样 `RxJavaPlugins.onSubscribe `  默认没有作用,实际的核心逻辑是调用了`subscribeActual(SingleObserver) `。  
对于我们上面设计的流，则是调用了 SingleJust 中的 `subscribeActual(SingleObserver) `

回顾上面 `SingleJust` 中 `subscribeActual(SingleObserver)` 的实现：

```
        observer.onSubscribe(Disposables.disposed());
        observer.onSuccess(value);
```

得到两个信息

* 首先调用下游观察者 `SingleObserver` 的 `OnSubscribe` 方法并传递用于取消操作的 `Disposable` 
* 调用`OnSuccess` 方法并传递之前保存下来的 `value` 

### Map 操作符

现在我们加入一个常用且重要的`Map`操作到流中

```
Disposable disposable = Single.just("wtf")
				 .map(it-> 0)
                 .subscribe(it -> Log.i("subscribe", String.of(it)));
```

上面这个流包括了三种典型的操作 创建`Creation` 操作符`Transformation`和 订阅`Subscribe`。  

依然先检查`map()` 方法，可以看到其中实例化了一个`SingleMap` 

```
    public final <R> Single<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new SingleMap<T, R>(this, mapper));
    }
```

再看看 `SingleMap`

```
public final class SingleMap<T, R> extends Single<R> {
    final SingleSource<? extends T> source;
    final Function<? super T, ? extends R> mapper;

    public SingleMap(SingleSource<? extends T> source, Function<? super T, ? extends R> mapper) {
        this.source = source;
        this.mapper = mapper;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super R> t) {
        source.subscribe(new MapSingleObserver<T, R>(t, mapper));
    }

    static final class MapSingleObserver<T, R> implements SingleObserver<T> {

        final SingleObserver<? super R> t;
        final Function<? super T, ? extends R> mapper;

        MapSingleObserver(SingleObserver<? super R> t, Function<? super T, ? extends R> mapper) {
            this.t = t;
            this.mapper = mapper;
        }

        @Override
        public void onSubscribe(Disposable d) {
            t.onSubscribe(d);
        }

        @Override
        public void onSuccess(T value) {
            R v;
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(value), "The mapper function returned a null value.");
            } catch (Throwable e) {
                Exceptions.throwIfFatal(e);
                onError(e);
                return;
            }

            t.onSuccess(v);
        }

        @Override
        public void onError(Throwable e) {
            t.onError(e);
        }
    }
}
```

类中信息稍微复杂一些：

1. 首先我们关注在`SingleMap`实例化的时候也是只做了保存数据的操作，而没有实际逻辑：将流的上游保存为 `source` 将数据转换的方法保存为 `mapper`
2. 第二步我们知道下游观察者 `SingleObserver` 会调用核心逻辑 `subscribeActual `方法来启动流
3. 在这里的`subscribeActual `方法中可以看到几个重要的信息
   * `MapSingleObserver`是一个观察者
   * `MapSingleObserver` 保存了下游的观察者 `SingleObserver` 以及 `mapper`
   * 上游 `source` 被 `MapSingleObserver` 订阅
   
由此可以看出在`SingleMap`被下游观察者订阅了之后，实例化了一个新的观察者`MapSingleObserver`并保存下游观察者`SingleObserver`的信息，再去订阅上游`SingleJust`。  
这种模式创建了一个装饰类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能的设计模式称为`装饰者模式`。

总结上面的执行顺序：

1. 在`Rx流`的最后一步调用 `subscribe`启动流`(ColdObservable)`
2. 首先执行`SingleMap`中的`subscribeActual`方法，其中包括生成新的`MapSingleObserver`并订阅 `SingleJust`
3. 执行`SingleJust`中的`subscribeActual`：调用下游`MapSingleObserver`的`onSubscribe` `onSuccess`方法
4. `MapSingleObserver `中的`onSubsribe``onSuccess`方法也很简单，分别调用下游 `Observer`的 `onSubsribe``onSuccess(异常时 onError)`方法

### observeOn 操作符

Rxjava首先被大家津津乐道之处是可以方便的切换线程，避免`Callback Hell`，现在来看看线程切换操作符。  
我们加入线程切换操作符 `observeOn`

```
Disposable disposable = Single.just("wtf")
				 .map(it-> 0)
				 .observeOn(Schedulers.io())
                 .subscribe(it -> Log.i("subscribe", String.of(it)));
```
同样的，在 `observeOn`方法中实例化了一个`SingleObserveOn`

```
    public final Single<T> observeOn(final Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new SingleObserveOn<T>(this, scheduler));
    }
```

继续看`SingleObserveOn`类中信息

```
public final class SingleObserveOn<T> extends Single<T> {

    final SingleSource<T> source;
    final Scheduler scheduler;

    public SingleObserveOn(SingleSource<T> source, Scheduler scheduler) {
        this.source = source;
        this.scheduler = scheduler;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super T> s) {
        source.subscribe(new ObserveOnSingleObserver<T>(s, scheduler));
    }

    static final class ObserveOnSingleObserver<T> extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {
        private static final long serialVersionUID = 3528003840217436037L;

        final SingleObserver<? super T> actual;
        final Scheduler scheduler;

        T value;
        Throwable error;

        ObserveOnSingleObserver(SingleObserver<? super T> actual, Scheduler scheduler) {
            this.actual = actual;
            this.scheduler = scheduler;
        }

        @Override
        public void onSubscribe(Disposable d) {
            if (DisposableHelper.setOnce(this, d)) {
                actual.onSubscribe(this);
            }
        }

        @Override
        public void onSuccess(T value) {
            this.value = value;
            Disposable d = scheduler.scheduleDirect(this);
            DisposableHelper.replace(this, d);
        }

        @Override
        public void onError(Throwable e) {
            this.error = e;
            Disposable d = scheduler.scheduleDirect(this);
            DisposableHelper.replace(this, d);
        }

        @Override
        public void run() {
            Throwable ex = error;
            if (ex != null) {
                actual.onError(ex);
            } else {
                actual.onSuccess(value);
            }
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
}


```

类似的

* 构造函数中保存了上游和线程切换的信息
* `subscribeActual` 实例化了一个新的观察者`ObserveOnSingleObserver`
 
不同的

* `ObserveOnSingleObserver` 还继承了`AtomicReference<Disposable>`、实现了`Disposable``Runnable`接口
* `onSuccess``onError`中都没有直接调用下游的`onSuccess``onError`方法，而是调用了`            Disposable d = scheduler.scheduleDirect(this);
`来执行`run`方法中的逻辑，而`run`方法中的逻辑则是调用下游的`onSuccess``onError`方法

查看`schedulerDirect`内部信息

```
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        final Worker w = createWorker();
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        DisposeTask task = new DisposeTask(decoratedRun, w);
        w.schedule(task, delay, unit);
        return task;
    }
```

创建了一个对应线程的`Worker`和一个可用于取消的`DisposeTask`并执行，对于`IoScheduler`则是创建了`EventLoopWorker`，再看看`EventLoopWorker`中的信息。

```
    @Override
    public Worker createWorker() {
        return new EventLoopWorker(pool.get());
    }
```

```
    static final class EventLoopWorker extends Scheduler.Worker {
        private final CompositeDisposable tasks;
        private final CachedWorkerPool pool;
        private final ThreadWorker threadWorker;

        final AtomicBoolean once = new AtomicBoolean();

        EventLoopWorker(CachedWorkerPool pool) {
            this.pool = pool;
            this.tasks = new CompositeDisposable();
            this.threadWorker = pool.get();
        }

        @Override
        public void dispose() {
            if (once.compareAndSet(false, true)) {
                tasks.dispose();

                // releasing the pool should be the last action
                pool.release(threadWorker);
            }
        }

        @Override
        public boolean isDisposed() {
            return once.get();
        }

        @NonNull
        @Override
        public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
            if (tasks.isDisposed()) {
                // don't schedule, we are unsubscribed
                return EmptyDisposable.INSTANCE;
            }

            return threadWorker.scheduleActual(action, delayTime, unit, tasks);
        }
    }
```

`EventLoopWorker`中则是维护了一套包含相应的`线程池`、可取消的`CompositeDisposable`、以及用于运行`Runable`的`ThreadWorker`。总的来说就是一套可以在相应线程运行且可取消的类和逻辑。

* 上面则解释了为什么`observeOn`可以切换下游的线程(`onSuccess``onError`)
* 同样解释了为什么不会改变`onSubsribe`的调用线程，因为可以看到`onSubscribe`方法中直接调用了下游的`onSucsribe`，并没有受到线程切换的影响。


### SubscribeOn

现在设计两个`Rx流`

```
Disposable disposable = Single.just("wtf")
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 0)
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 1)
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 2)
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 3)
                 .subscribe(it -> Log.i("subscribe", 4);
```

```
Disposable disposable2 = Single.just("wtf")
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 0)
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 1)
				 .subscribeOn(Schedulers.io())
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 2)
				 .doOnSubsribe(it-> Log.i("doOnSubsribe", 3)
                 .subscribe(it -> Log.i("subscribe", 4);
```

经过了优质的飞狼新人培训课程你可能已经知道并记住了两个流的打印的顺序分别是 `01234``23014`，但是为什么`doOnSubsribe`方法和`RxJava1`中调用顺序完全不一样，为什么通过`subscribeOn`切换线程会影响执行顺序？

先找到 `SingleSubscribeOn` 类

```
public final class SingleSubscribeOn<T> extends Single<T> {
    final SingleSource<? extends T> source;
    final Scheduler scheduler;
    
    public SingleSubscribeOn(SingleSource<? extends T> source, Scheduler scheduler) {
        this.source = source;
        this.scheduler = scheduler;
    }

    @Override
    protected void subscribeActual(final SingleObserver<? super T> s) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s, source);
        //直接调用下游 onSubscribe
        s.onSubscribe(parent);
        //再执行订阅上游的方法
        Disposable f = scheduler.scheduleDirect(parent);
        parent.task.replace(f);
    }

    static final class SubscribeOnObserver<T>
    extends AtomicReference<Disposable>
    implements SingleObserver<T>, Disposable, Runnable {

        private static final long serialVersionUID = 7000911171163930287L;
        final SingleObserver<? super T> actual;
        final SequentialDisposable task;
        final SingleSource<? extends T> source;
        
        SubscribeOnObserver(SingleObserver<? super T> actual, SingleSource<? extends T> source) {
            this.actual = actual;
            this.source = source;
            this.task = new SequentialDisposable();
        }

        @Override
        public void onSubscribe(Disposable d) {
        	  //没有继续调用下游的 onSubscribe 方法
            DisposableHelper.setOnce(this, d);
        }

        @Override
        public void onSuccess(T value) {
            actual.onSuccess(value);
        }

        @Override
        public void onError(Throwable e) {
            actual.onError(e);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
            task.dispose();
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        @Override
        public void run() {
            source.subscribe(this);
        }
    }

}
```

同样的直接看`subscribeActual`方法及`onSubscribe`方法，和之前的操作符的逻辑区别很大：

* `SubscribeOnObserver`同样还继承了`AtomicReference<Disposable>`，实现了`Disposable``Runnable`接口
* 并没有直接调用`subscribe`订阅上游，而是执行了其它操作符在 `onSubscribe`中订阅下游的操作
* 然后再结合`        Disposable f = scheduler.scheduleDirect(parent);
`和`run`方法可以知道在新的线程中执行了订阅上游的操作 `source.subscribe(this);`
* `onSubsribe`中并没有再继续调用下游的 `onSubsribe`

综合起来可以知道，本来应该在整个流从下至上订阅完成后按照从上至下的顺序执行 `onSubscribe`的流，在使用`subsribeOn`操作符的后，在订阅的时(执行`subscribeActual`)，就开始执行下游的`onSubscribe`且在当前线程！然后才在指定的`io`线程执行之下而上的操作，这也是为什么`subsribeOn`影响的是上游的线程。

其它操作符可以按照同样的方式去分析。