Promise/A+规范翻译
================

###1. 术语
* promise是拥有`then`方法，其行为符合此规范的对象或者函数。  
* thenable是定义`then`方法的对象或者函数。  
* value是任何合法的javascript值(包括undefined, 一个thenable, 一个promise)  
* exception是一个值，通过`throw`语句抛出。  
* reason是一个值，表明promise被reject的原因。

###2. 要求

**2.1 promise状态**

一个promise必须是下面三种状态之一：`pending`, `fulfilled`, `rejected`

* 当一个promise是`pending`状态：

> 可以转变到fulfilled状态或者rejected状态

* 当一个promise是`fulfilled`状态：

> 1\. 不可以转变到其他任何状态
> 2\. 必须有一个不能改变的value

* 当一个promise是`rejected`状态：

> 1\. 不可以转变到其他任何状态
> 2\. 必须有一个不可改变的reason

在这里，"不能改变"意味着不可改变身份(即 ===)，但并不意味着深不变性。

<!-- more -->

**2.2 then方法**

一个promise必须提供一个then方法来访问当前或者最终的value或者reason.

一个promise的`then`方法，接受两个参数：

``` javascript
promise.then(onFulfilled, onRejected)
```

* `onFulfilled`和`onRejected`都是可选参数：

> 如果onFulfilled或者onRejected不是函数，将会被忽略

* 如果`onFulfilled`是一个函数：

> 1\. 当promise是fulfilled状态的时候被调用，promise的值将作为其第一个参数。
> 2\. 只能执行一次

* 如果`onRejected`是一个函数：

> 当promise是rejected状态的时候被调用，promise的值将作为其第一个参数
> 该函数只能执行一次

* `onFulfilled`或者`onRejected`只有当执行上下文栈里只包含平台代码[3.1](#note)的时候才会被调用执行。

* `onFulfilled`或者`onRejected`只有是函数的时候才会被调用(即没有`this`值).[3.2](#note)

* `then`在同一个promise里可以被调用多次。

> 当promise是fulfilled或者rejected状态的时候，onFulfilled和onRejected回调函数的调用顺序将会按照在then里定义的顺序进行调用。

* `then`必须返回一个promise[3.3](#note)

``` javascript
promise2 = promise1.then(onFulfilled, onRejcted);
```

* 如果`onFulfilled`或者`onRejected`返回一个值`x`，那么将会执行Promise处理程序`[[Resolve]](promise2, x)`。
* 如果`onFulfilled`或者`onRejected`抛出一个异常`e`，那么`promise2`将会因为异常`e`被rejected。
* 如果`onFulfilled`不是一个function，而且`promise1`是fulfilled，那么`promise2`也必须以相同的值被fulfilled。
* 如果`onRejected`不是一个function，而且`promise1`是rejected，那么`promise2`也必须以相同的原因被rejected

**2.3 Promise处理程序**

Promise处理程序是一个输入promise和值的抽象操作，我们把它表示成：`[[Resolve]](promise, x)`。如果`x`是thenable，并假设`x`的行为至少在某种情况下是一个promise，那么它将会试图根据`x`的状态做出promise。否则`x`的值满足promise。

对于thenables的处理，允许promise实现互操作，只要它们公开一个与Promise/A+兼容的方法。它也允许Promise/A+的实现，可以不符合标准，但是合理的实现。

要运行`[[Resolve]](promise, x)`，请执行下列步骤：

0. 如果`promise`和`x`指向同一个对象，那么将会抛出异常`TypeError`，从而拒绝promise
1. 如果`x`是promise，根据其状态[3.4](#note):
    * 如果`x`是pending状态，`x`必须保持该状态，直到`x`是fulfilled或者rejected。
    * 如果`x`是fulfilled状态，将会以相同的值去fulfill promise。
    * 如果`x`是rejected状态，将会以相同的原因去rejected promise。
    
2. 另外，如果`x`是对象或者函数：
    * 让`then`赋给`x.then`。[3.5](#note)
    * 如果检索`x.then`属性的时候抛出了异常`e`，那么将会因为异常`e`reject promise。
    * 如果`then`是一个function，把`x`当做`this`去调用它，第一个参数为`resolvePromise`，第二个参数`rejectPromise`，这时：
        * 如果当`resolvePromise`作为一个值`y`被调用的时候，将会执行`[[Resolve]](promise, y)`
        * 如果当`rejectPromise`作为一个原因`r`被调用的时候，将会因为`r`reject promise。
        * 如果`resolvePromise`和`resolvePromise`都被调用，或者一个参数被调用多次，那么只有第一次调用生效，其他次调用将会被忽略。
    * 如果调用`then`抛出了一个异常`e`：
        * 如果`resolvePromise`或者`resolvePromise`被调用过了，该异常将会被忽略。
        * 否则将会因为`e`reject promise
    * 如果`then`不是一个function，将会通过`x`去fulfill promise
    * 如果`x`既不是对象也不是函数，也会通过`x`去fulfill promise
    
如果promise通过一个圆形的thenable链被resolve，那么`[[Resolve]](promise, x)`的递归性质将最终导致`[[Resolve]](promise, x)`被再次调用，根据上述算法将导致无限递归。我们推荐去实现,但不是必需的,检测这样的递归和通过一个异常`TypeError`来reject promise。

###3. Notes<a name="note"></a>
**3.1** 这里的“平台代码”是指引擎，环境，和promise执行代码。在实践中，需要确保onFulfilled和onRejected在一个新的堆栈中，并在事件循环`then`后异步执行。这可以实现任何一个“宏任务”机制，例如：setTimeout或setImmediate，或“微任务”的机制，如MutationObserver或process.nextTick。由于promise的实现被认为是平台的代码，它本身可能包含在该处理程序的调用的任务调度队列或“蹦床”。

**3.2** 也就是说，在严格模式下，`this`将是不确定的 ;在非严格模式下，`this`将是全局对象。

**3.3** 实现可能允许`promise2 === promise1`，提供了符合所有要求的实现。每个实现应说明是否`promise2 === promise1`会在某种条件下可以产生。

**3.4** 一般来说，如果它来自当前的实现，`x`将是一个真正的promise。并允许根据符合promise的状态，使用特定的实现手段。

**3.5** 程序第一次执行的时候会保存`x.then`的引用，然后并会测试该引用，并调用该引用，避免了多次访问`x.then`属性。并能确保一致性。

**3.6** 实现不应设置没有任何深度限制的thenable链，否则将会导致无限递归。只有正常的周期才会导致一个`TypeError`异常，如果遇到一个无限递归thenables链，递归永远是正确的行为.





