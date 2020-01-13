# ListenableFuture

Concurrency is a _hard_ problem, but it is significantly simplified by working
with powerful and simple abstractions. To simplify matters, Guava extends the
`Future` interface of the JDK with [`ListenableFuture`].

**We strongly advise that you always use `ListenableFuture` instead of `Future`
in all of your code**, because:

*   Most `Futures` methods require it.
*   It's easier than changing to `ListenableFuture` later.
*   Providers of utility methods won't need to provide `Future` and
    `ListenableFuture` variants of their methods.

## Interface

A traditional `Future` represents the result of an asynchronous computation: a
computation that may or may not have finished producing a result yet. A `Future`
can be a handle to an in-progress computation, a promise from a service to
supply us with a result.

A `ListenableFuture` allows you to register callbacks to be executed once the
computation is complete, or if the computation is already complete, immediately.
This simple addition makes it possible to efficiently support many operations
that the basic `Future` interface cannot support.

The basic operation added by `ListenableFuture` is [`addListener(Runnable,
Executor)`], which specifies that when the computation represented by this
`Future` is done, the specified `Runnable` will be run on the specified
`Executor`.

## Adding Callbacks

Most users will prefer to use [`Futures.addCallback(ListenableFuture<V>,
FutureCallback<V>, Executor)`], or the [version][no-executor-add-callback] which
defaults to using `MoreExecutors.directExecutor()`, for use when the callback is
fast and lightweight. A [`FutureCallback<V>`] implements two methods:

*   [`onSuccess(V)`], the action to perform if the future succeeds, based on its
    result
*   [`onFailure(Throwable)`], the action to perform if the future fails, based
    on the failure

## Creation

Corresponding to the JDK [`ExecutorService.submit(Callable)`] approach to
initiating an asynchronous computation, Guava provides the
[`ListeningExecutorService`] interface, which returns a `ListenableFuture`
wherever `ExecutorService` would return a normal `Future`. To convert an
`ExecutorService` to a `ListeningExecutorService`, just use
[`MoreExecutors.listeningDecorator(ExecutorService)`].

```java
ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));
ListenableFuture<Explosion> explosion = service.submit(
    new Callable<Explosion>() {
      public Explosion call() {
        return pushBigRedButton();
      }
    });
Futures.addCallback(
    explosion,
    new FutureCallback<Explosion>() {
      // we want this handler to run immediately after we push the big red button!
      public void onSuccess(Explosion explosion) {
        walkAwayFrom(explosion);
      }
      public void onFailure(Throwable thrown) {
        battleArchNemesis(); // escaped the explosion!
      }
    },
    service);
```

Alternatively, if you're converting from an API based on [`FutureTask`], Guava
offers [`ListenableFutureTask.create(Callable<V>)`] and
[`ListenableFutureTask.create(Runnable, V)`]. Unlike the JDK,
`ListenableFutureTask` is not meant to be extended directly.

If you prefer an abstraction in which you set the value of the future rather
than implementing a method to compute the value, consider extending
[`AbstractFuture<V>`] or using [`SettableFuture`] directly.

If you must convert a `Future` provided by another API to an `ListenableFuture`,
you may have no choice but to use the heavyweight
[`JdkFutureAdapters.listenInPoolThread(Future)`] to convert a `Future` to a
`ListenableFuture`. Whenever possible, it is preferred to modify the original
code to return a `ListenableFuture`.

## Application

The most important reason to use `ListenableFuture` is that it becomes possible
to have complex chains of asynchronous operations.

```java
ListenableFuture<RowKey> rowKeyFuture = indexService.lookUp(query);
AsyncFunction<RowKey, QueryResult> queryFunction =
  new AsyncFunction<RowKey, QueryResult>() {
    public ListenableFuture<QueryResult> apply(RowKey rowKey) {
      return dataService.read(rowKey);
    }
  };
ListenableFuture<QueryResult> queryFuture =
    Futures.transformAsync(rowKeyFuture, queryFunction, queryExecutor);
```

Many other operations can be supported efficiently with a `ListenableFuture`
that cannot be supported with a `Future` alone. Different operations may be
executed by different executors, and a single `ListenableFuture` can have
multiple actions waiting upon it.

When several operations should begin as soon as another operation starts --
"fan-out" -- `ListenableFuture` just works: it triggers all of the requested
callbacks. With slightly more work, we can "fan-in," or trigger a
`ListenableFuture` to get computed as soon as several other futures have _all_
finished: see [the implementation of `Futures.allAsList`] for an example.

| Method | Description | See also |
|:-------|:------------|:---------|
| [`transformAsync(ListenableFuture<A>, AsyncFunction<A, B>, Executor)`]`*` | Returns a new `ListenableFuture` whose result is the product of applying the given `AsyncFunction` to the result of the given `ListenableFuture`.  | [`transformAsync(ListenableFuture<A>, AsyncFunction<A, B>)`] |
| [`transform(ListenableFuture<A>, Function<A, B>, Executor)`] | Returns a new `ListenableFuture` whose result is the product of applying the given `Function` to the result of the given `ListenableFuture`. | [`transform(ListenableFuture<A>, Function<A, B>)`] |
| [`allAsList(Iterable<ListenableFuture<V>>)`] | Returns a `ListenableFuture` whose value is a list containing the values of each of the input futures, in order.  If any of the input futures fails or is cancelled, this future fails or is cancelled. | [`allAsList(ListenableFuture<V>...)`] |
| [`successfulAsList(Iterable<ListenableFuture<V>>)`] | Returns a `ListenableFuture` whose value is a list containing the values of each of the successful input futures, in order.  The values corresponding to failed or cancelled futures are replaced with `null`. | [`successfulAsList(ListenableFuture<V>...)`] |

`*` An [`AsyncFunction<A, B>`] provides one method, `ListenableFuture<B> apply(A
input)`. It can be used to asynchronously transform a value.

```java
List<ListenableFuture<QueryResult>> queries;
// The queries go to all different data centers, but we want to wait until they're all done or failed.

ListenableFuture<List<QueryResult>> successfulQueries = Futures.successfulAsList(queries);

Futures.addCallback(successfulQueries, callbackOnSuccessfulQueries);
```

## Avoid nested Futures

In cases where code calls a generic interface and returns a Future, it's
possible to end up with nested `Future`s. For example:

```java
executorService.submit(new Callable<ListenableFuture<Foo>() {
  @Override
  public ListenableFuture<Foo> call() {
    return otherExecutorService.submit(otherCallable);
  }
});
```

would return a `ListenableFuture<ListenableFuture<Foo>>`. This code is
incorrect, because if a `cancel` on the outer future races with the completion
of the outer future, that cancellation will not be propagated to the inner
future. It's also a common error to check for failure of the other future using
`get()` or a listener, but unless special care is taken an exception thrown from
`otherCallable` would be suppressed. To avoid this, all of Guava's
future-handling methods (and some from the JDK) have *Async versions that safely
unwrap this nesting - [`transform(ListenableFuture<A>, Function<A, B>,
Executor)`] and [`transformAsync(ListenableFuture<A>, AsyncFunction<A, B>,
Executor)`], or [`ExecutorService.submit(Callable)`] and
[`submitAsync(AsyncCallable<A>, Executor)`], etc.

[`ListenableFuture`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFuture.html
[`addListener(Runnable, Executor)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFuture.html#addListener-java.lang.Runnable-java.util.concurrent.Executor-
[`Futures.addCallback(ListenableFuture<V>, FutureCallback<V>, Executor)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#addCallback-com.google.common.util.concurrent.ListenableFuture-com.google.common.util.concurrent.FutureCallback-java.util.concurrent.Executor-
[no-executor-add-callback]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#addCallback-com.google.common.util.concurrent.ListenableFuture-com.google.common.util.concurrent.FutureCallback-
[`FutureCallback<V>`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/FutureCallback.html
[`onSuccess(V)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/FutureCallback.html#onSuccess-V-
[`onFailure(Throwable)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/FutureCallback.html#onFailure-java.lang.Throwable-
[`ExecutorService.submit(Callable)`]: http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html#submit-java.util.concurrent.Callable-
[`ListeningExecutorService`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListeningExecutorService.html
[`MoreExecutors.listeningDecorator(ExecutorService)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/MoreExecutors.html#listeningDecorator-java.util.concurrent.ExecutorService-
[`FutureTask`]: http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/FutureTask.html
[`ListenableFutureTask.create(Callable<V>)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFutureTask.html#create-java.util.concurrent.Callable-
[`ListenableFutureTask.create(Runnable, V)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFutureTask.html#create-java.lang.Runnable-V-
[`AbstractFuture<V>`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/AbstractFuture.html
[`SettableFuture`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/SettableFuture.html
[`JdkFutureAdapters.listenInPoolThread(Future)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/JdkFutureAdapters.html
[`JdkFutureAdapters.listenInPoolThread(Future)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/JdkFutureAdapters.html
[the implementation of `Futures.allAsList`]: https://google.github.io/guava/releases/snapshot/api/docs/src-html/com/google/common/util/concurrent/Futures.html#line.1276
[`transformAsync(ListenableFuture<A>, AsyncFunction<A, B>, Executor)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#transformAsync-com.google.common.util.concurrent.ListenableFuture-com.google.common.util.concurrent.AsyncFunction-java.util.concurrent.Executor-
[`transformAsync(ListenableFuture<A>, AsyncFunction<A, B>)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#transformAsync-com.google.common.util.concurrent.ListenableFuture-com.google.common.util.concurrent.AsyncFunction-
[`transform(ListenableFuture<A>, Function<A, B>, Executor)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#transform-com.google.common.util.concurrent.ListenableFuture-com.google.common.base.Function-java.util.concurrent.Executor-
[`transform(ListenableFuture<A>, Function<A, B>)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#transform-com.google.common.util.concurrent.ListenableFuture-com.google.common.base.Function-
[`allAsList(Iterable<ListenableFuture<V>>)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#allAsList-java.lang.Iterable-
[`allAsList(ListenableFuture<V>...)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#allAsList-com.google.common.util.concurrent.ListenableFuture...-
[`successfulAsList(Iterable<ListenableFuture<V>>)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#successfulAsList-java.lang.Iterable-
[`successfulAsList(ListenableFuture<V>...)`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#successfulAsList-com.google.common.util.concurrent.ListenableFuture...-
[`AsyncFunction<A, B>`]: https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/AsyncFunction.html
[`submitAsync(AsyncCallable<A>, Executor)`]:
https://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#submitAsync-com.google.common.util.concurrent.AsyncCallable-java.util.concurrent.Executor-
