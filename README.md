# RxWeaver

关于这个repo的起源，请参考这篇文章：

[不要打破链式调用！一个极低成本的RxJava全局Error处理方案](https://juejin.im/post/5be7bb9f6fb9a049f069c706)

## 通知

* 关于移除了 `Gradle` 依赖的方式(2019/3/18)

最新的代码中，我移除了 `Gradle` 依赖的方式，根本原因是，我认为这个 `repo` 被称为 **业务逻辑的展示** 更为确切——每个项目都有属于自己的特定业务，这些业务逻辑千变万化却又不离其宗。

因此我把最基本的逻辑抽了出来，放在了`rxweaver`这个`Module`里，然后又把复杂的业务逻辑放在了`sample`中进行展示。我希望您能够从我的代码中得到灵感，即使它的代码可能对您来说非常抽象。

同样也非常真诚的欢迎您宝贵的建议和讨论～

## 简介

关于`RxJava`的全局的error处理，通用的方案是通过 **继承**，将处理的逻辑加入到放入`Observer`导出类的`onError()`中，这种方式相对 **简单** 且 **易于上手**，但限制性在于：

* 1.开发者无法在`subscribe()`中使用lambda，灵活性降低， 这也是为什么强调尽量考虑 **组合** 的设计方式而不直接采用 **继承** 的原因。
* 2.如果开发者想要移除（或者重构）相关代码，就必须改动既有的业务逻辑，比如每一个网络请求的`subscribe()`中的代码，这种逻辑改动是产品级的。
* 3.`onError()`中对error进行处理，这意味着打破了`Observable`的链式调用。

RxWeaver是轻量且灵活的RxJava2 **全局Error处理中间件** ，类似 **AOP** 的思想，在既有的代码上  **插入** 或 **删除**  一行代码，即可实现全局Error处理的需求——而不是破坏`RxJava`所表达的 **响应式编程** 和 **函数式编程** 的思想。

## 特性

* 1.**轻量**：整个工具库只有4个类共不到200行代码，jar包体积仅3kb;
* 2.**无额外依赖**：所有代码的实现都依赖于`RxJava`本身的原生操作符;
* 3.**高扩展性**：开发者可以通过接口实现任意复杂的需求实现，详情请参考下文;
* 4.**灵活**：不破坏既有的业务代码，而是在原本的流中 **插入** 或 **删除** 一行代码——它是 **可插拔的**。

## Usage

### 1.Fork/Clone项目,并将`rxweaver`模块的代码复制进自己的项目中，并进行对应的修改：

> $ git clone https://github.com/qingmei2/RxWeaver.git

~~或者直接添加依赖（不建议）~~：

```groovy
implementation 'com.github.qingmei2.rxweaver:rxweaver:0.3.0'
```

### 2.配置GlobalErrorTransformer

[GlobalErrorTransformer](https://github.com/qingmei2/RxWeaver/blob/kotlin/rxweaver/src/main/java/com/github/qingmei2/core/GlobalErrorTransformer.kt) 是一个是`Transformer<T, R>`的实现类，负责把全局的error处理逻辑，分发给不同的 **响应式类型**(Observable、Flowable、Single、Maybe、Completable)：

```Kotlin
class GlobalErrorTransformer<T> constructor(
        private val globalOnNextInterceptor: (T) -> Observable<T> = { Observable.just(it) },
        private val globalOnErrorResume: (Throwable) -> Observable<T> = { Observable.error(it) },
        private val retryConfigProvider: (Throwable) -> RetryConfig = { RetryConfig() },
        private val globalDoOnErrorConsumer: (Throwable) -> Unit = { }
) : ObservableTransformer<T, T>, FlowableTransformer<T, T>, SingleTransformer<T, T>,  MaybeTransformer<T, T>, CompletableTransformer {
      // ...
}
```

配置一个函数，保证能够返回`GlobalErrorTransformer`的实例：

```kotlin
fun <T> handleGlobalError(activity: FragmentActivity): GlobalErrorTransformer<T>{
  return .....
}
```

[点击这里](https://github.com/qingmei2/RxWeaver/blob/kotlin/sample/src/main/java/com/github/qingmei2/RxUtils.kt)查看sample中的配置方式示例。

### 3.对需要进行全局error处理的RxJava流中添加这行代码：

```kotlin
private fun requestHttp(observable: Observable<UserInfo>) {
    observable
            .compose(handleGlobalError<UserInfo>(this))  // 将上面的接口配置给Observable
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe {
                  // ....
            }
}
```

## 功能

下面通过几个案例展示RxWeaver的功能支持，你也可以运行sample并运行查看效果。

### 1.全局异常配置，弹出toast：

<div align:left;display:inline;> <img width="200" height="360" src="https://github.com/qingmei2/RxWeaver/blob/kotlin/screenshots/1.gif"/> </div>


### 2.全局异常配置，弹出Dialog并是否重试：

<div align:left;display:inline;> <img width="200" height="360" src="https://github.com/qingmei2/RxWeaver/blob/kotlin/screenshots/2.gif"/> </div>


### 3.全局异常配置，token异常，重新登录后返回界面重试：

<div align:left;display:inline;> <img width="200" height="360" src="https://github.com/qingmei2/RxWeaver/blob/kotlin/screenshots/3.gif"/> </div>

### 4.更多

当然也能够实现更多私有化定制的需求....

## 原理&如何配置GlobalErrorTransformer

**RxWeaver** 是一个轻量且灵活的RxJava2 **全局Error处理组件** ，这意味着，它并不是一个设计非常复杂的框架。**Weaver** 翻译过来叫做 **编织鸟**， 可以让开发者对error处理逻辑 **组织**，以达到实现全局Error的把控。

它的核心原理是依赖 `compose()` 操作符——这是RxJava给我们提供的可以面向 **响应式数据类型** (Observable/Flowable/Single等等)进行 **AOP** 的接口, 可以对响应式数据类型 **加工** 、**修饰** ，甚至 **替换**。

它的原理也是非常 **简单** 的，只要熟悉了 `onErrorResumeNext` 、 `retryWhen` 、 `doOnError` 这几个关键的操作符，你就可以马上上手对应的配置；它也是非常 **轻量** 的，轻到甚至可以直接把源代码复制粘贴到自己的项目中，通过jcenter依赖，它的体积也只有3kb。

### 1. globalOnNextInterceptor: (T) -> Observable<T>

**将正常数据转换为一个异常** 是很常见的需求，有时流中的数据可能会是一个错误的状态（比如，token失效）。

`globalOnNextInterceptor` 函数内部直接使用 `flatMap()` 操作符将数据进行了转换，因此你可以将一个错误的状态转换为 `Observable.error()` 向下传递：

```kotlin
globalOnNextInterceptor = {
    when (it.statusCode) {
        STATUS_UNAUTHORIZED -> {        // token 失效，将流转换为error，交由下游处理
            Observable.error(TokenExpiredException())
        }
        else -> Observable.just(it)     // 其他情况，数据向下游正常传递
    }
}
```

### 2. globalOnErrorResume: (Throwable) -> Observable<T>

和 `globalOnNextInterceptor` 函数很相似，更常见的情况是通过解析不同的 `Throwable` ，然后根据实际业务做出对应的处理：

```kotlin
globalOnErrorResume = { error ->
    when (error) {
        is ConnectException -> {        // 连接错误，转换为特殊的异常（标记作用），交给下游
            Observable.error<T>(ConnectFailedAlertDialogException())
        }
        else -> Observable.error<T>(error)  // 其他情况，异常向下游正常传递
    }
}
```

`globalOnErrorResume` 函数内部是通过 `onErrorResumeNext()` 操作符实现的。

### 3.retryConfigProvider: (Throwable) -> RetryConfig

当需要做出是否要重试的决定时，需要根据异常的类型进行判断，并做出对应的行为：

```kotlin
retryConfigProvider = { error ->
    when (error) {
        is ConnectFailedAlertDialogException -> RetryConfig {
            // .....
        }
        is TokenExpiredException -> RetryConfig(delay = 3000) {
            // .....
        }
        else -> RetryConfig() // 其它异常都不重试
    }
}
```

`retryConfigProvider` 函数内部是通过 `retryWhen()` 操作符实现的。


### 4.globalDoOnErrorConsumer: (Throwable) -> Unit

`globalDoOnErrorConsumer` 函数并不会拦截或消费上游发射的异常，它的内部实际上就是通过 `doOnError()` 操作符的调用，做一些 **顺势而为** 的处理,比如toast，或者其它。

```kotlin
globalDoOnErrorConsumer = { error ->
    when (error) {
        is JSONException -> {
            Toast.makeText(activity, "全局异常捕获-Json解析异常！", Toast.LENGTH_SHORT).show()
        }
        else -> {

        }
    }
}
```

## License

    The RxWeaver: MIT License

    Copyright (c) 2018 qingmei2

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
