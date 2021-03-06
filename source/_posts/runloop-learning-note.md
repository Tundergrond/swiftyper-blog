---
title: RunLoop 学习笔记
date: 2017-01-04 21:18:46
tags: [RunLoop]
---

RunLoop 是 iOS 开发当中一个很基础又很重要的概念。由于它是一个很底层的概念，日常开发中很少直接接触到，再加上官方文档写得很难理解，导致很多开发者（包括我自己）都对这个概念一知半解。但是，既然它是很重要的概念，又是 iOS 开发的底层基础，我们就不得不去把它啃下来。

经过这几天的学习，感觉自己对 RunLoop 的概念理解得比较清晰了，因此写篇笔记来进行下总结。

<!-- more -->

## 为何要有 RunLoop

理解 RunLoop 的首要前提就是要明白它为何会存在，它存在的目的是什么。

相信大家都学过 C 语言，C 语言当中最简单的程序就是 Hello World 了。整个程序就只打印出一句 Hello World，然后立马结束运行。这也是大多数命令行程序的运行方法，一次执行一个任务，执行完后就退出。

但是一个 iOS 应用一旦启动起来后就会一直处于等待用户事件（类似点击或者触摸事件）的状态，除非用户手动关闭它，不然它是不会退出的。那它是怎么做到的呢，这就是 RunLoop 起到的作用了。

RunLoop 是一种事件驱动（Event Driven）模型，这种模型并非是 iOS 特有的。Android 中的 Looper 和 Windows SDK 开发中的消息循环机制都是属于这种模型。

作为一个开发过 Windows SDK 程序的程序员，对那些 WM_ 开头的各种消息都还记忆犹新。Windows 中的消息循环机制也是事件驱动的一种，只是那时需要接收并手动处理各种消息。而 iOS 封装得比较完全，我们不需要直接跟各种消息回调打交道，只需要处理事件就行了，这就是为什么我们平常很少接触到 RunLoop 的原因。

## RunLoop 何时启动

既然 RunLoop 可以让应用保持等待接受用户事件的状态，那么它是何时启动的呢。似乎我们在开发中并没有写过任何有关启动 RunLoop 的代码。

答案是，我们并不需要手动启动 RunLoop。

当我们使用 Xcode 创建一个新的 iOS 应用时，它会自动帮我们生成 main 方法（这个方法在 main.m 文件中），而在 main 方法里面会调用 `UIApplicationMain` 方法。`UIApplicationMain` 会将 AppDelegate 设置为应用程序的代码，同时会为主线程创建一个 RunLoop 并启动。

> 关于更详细的 iOS 应用启动流程，可以[参考这篇文章](https://oleb.net/blog/2012/02/app-launch-sequence-ios-revisited/)。

因此任何一个 iOS 应用一旦启动了，就至少存在一个 RunLoop，并且这个 RunLoop 是属于主线程的。之所以说属于主线程，是因为 RunLoop 与线程的关系是一一对应的，关于这点，在后面会讲到。

## RunLoop 对象

其实事件驱动模型的本质就是一个死循环，然后在循环中不断等待消息的到来，然后对其进行处理。因此，这种模型的关键点就在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免占用过多的资源，并且在有消息到来时能够立即进行响应。

而上面提到的这些关键点就是 RunLoop 的工作。RunLoop 实际上就是一个对象，它为我们提供了一个进入事件循环的入口函数，线程执行这个函数后就会处于消息循环中，直到循环结束（比如接收到退出消息）。

在 iOS 中提供了两个 RunLoop 对象：NSRunLoop 与 CFRunLoopRef。

其中，NSRunLoop 是基于 CFRunLoopRef 的简单封装。NSRunLoop 的 API 是非线程安全的，而 CFRunLoopRef 的 API 是线程安全的。更重要的一点是 CFRunLoopRef 的代码是[开源的](http://opensource.apple.com/source/CF/CF-855.17/CFRunLoop.c)。所谓源码之前无秘密，能看到源码意味着我们能彻底搞懂它是如何工作的。因此，如果需要对 RunLoop 进行更加精细的操作，建议使用 CFRunLoopRef 而非 NSRunLoop。

## RunLoop 与线程的关系

查看下 NSRunLoop 与 CFRunLoopRef 的文档，你会发现苹果并没有提供直接创建 RunLoop 的方法。它只提供了两个自动获取的函数：`CFRunLoopGetMain()` 和 `CFRunLoopGetCurrent()`。

事实上，RunLoop 在其内部维护了一个全局的字典，这个字典以线程为键，以 RunLoop 为值，表明了线程与 RunLoop 是一一对应的。当我们用这两个方法获取 RunLoop 的时候，如果该线程的 RunLoop 不存在，则它会帮我们自动创建一个并添加到字典中。

所以线程在刚创建时是没有 RunLoop 的，并且如果我们不主动获取，它就一直不会有。除了主线程，其它的线程我们都只能在其内部获取 RunLoop。

## RunLoop 中的事件

从刚刚开始我们就一直在说 RunLoop 可以让 iOS 应用启动后不退出，而处于接受事件的状态。那么这里所谓的事件到底是什么呢。

从上层来讲，事件就是所谓的按钮点击，屏幕点击，以及手势识别等。

但是，从 RunLoop 的角度来讲，它只接受两种事件，一种就是 *Input Source*，这种 Source 事件是异步的，它通常是来自于其它线程或者其它程序的消息（按钮与屏幕点击就属于这一种事件）。另外一种是 *Timer Source*，这种 Source 我们就很熟悉了，就是我们平常开发中用到的计时器，这种 Source 事件是同步的。

上述两种事件在 RunLoop 对象中对应的类为 `CFRunLoopSourceRef` 和 `CFRunLoopTimerRef`。

RunLoop 除了可以处理输入事件外，还能在它的行为发生变化时产生通知。我们可以通过 `CFRunLoopObserverRef` 来为 RunLoop 添加观察者并注册回调，只要 RunLoop 的状态发生变化，观察者就能接受到这个变化并利用这些信息在线程中做进一步的处理。

RunLoop 中可以观察的点有如下几个：

```objective-c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};
```

上面的 Source/Timer/Observer 被统称为 *mode item*，这就涉及到了 RunLoop 中的另一个概念 *RunLoop Mode*，它的类是 `CFRunLoopModeRef`。一个 RunLoop Mode 就是一系列 *Input Source*、*Timer Source* 以及 *Observer* 的集合。

每次我们要启动一个 RunLoop 的时候，都需要指定一个 Mode。当我们使用某个 Mode 启动 RunLoop 后，在本次启动的 RunLoop 中只会监听与处理与该 Mode 相关联的事件，类似地，只有以该 Mode 相关联的观察者会接到通知。这样做的目的主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

RunLoop 每次启动时只能指定一个 Mode，如果需要切换 Mode，需要退出 RunLoop 再重新指定一个 Mode 进入。苹果公开提供的 Mode 有两个: kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。同时，这里还有一个特殊的概念叫 *CommonModes*：一个 Mode 可以将自己标记为 *Common* 属性（通过将其 ModeName 添加到 RunLoop 的 *commonModes* 中）。每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 *Common* 票房的所有 Mode 里。

CommonModes 最典型的一个应用就是在让我们可以在拖动 UIScrollView 的同时使定时器不要失效，具体的内容可以参考[这篇文章](http://blog.ibireme.com/2015/05/18/runloop/)中的 *RunLoop 的 Mode* 小节。

## RunLoop 中事件顺序

RunLoop 一旦启动后，它就会开始处理各种等待事件，并向其观察者发送通知。而 RunLoop 做这些事是有顺序的，具体的顺序如下：

1. 通知观察者：即将进入 RunLoop
2. 通知观察者：即将处理 Timer
3. 通知观察者：即将处理所有非基于 Port 的 Input Source
4. 处理所有非基于 Port 的 Input Source
5. 如果有基于 Port 的 Input Source 可供处理，跳到步骤 9
6. 通知观察者：线程即将休眠
7. 线程休眠，等待如下事件将唤醒：
	* 基于 Port 的事件到来
	* 计时器启动
	* RunLoop 超时
	* RunLoop 被手动唤醒
8. 通知观察者：线程被唤醒
9. 处理唤醒时收到的事件，并跳回 2
10. 通知观察者：即将退出 RunLoop

针对这个顺序，也可以在直接在源码中看到。

实际上，RunLoop 内部就是一个 do-while 循环。当我们调用 CFRunLoopRun() 时，线程就会一直停留在这个循环里；直到超时或者被手动停止，该函数才会返回。

## 什么时候使用 RunLoop

正如前面所讲的，一个 iOS 应用在启动时就会自动为主线程创建一个 RunLoop，因此我们大多数情况下都是不需要手动去创建 RunLoop。

根据官方文档，我们只有在下面提到的这些情况下才会去为子线程启动一个 RunLoop：

* 使用 port 或者自定义输入源与其它线程进行交流时
* 要在线程上使用计时器时
* 要使用 `performSelector...` 类的方法时
* 使用线程执行周期性的任务时

当我们为子线程启动 RunLoop 时，需要规划好在什么情况下退出这个线程，而非让它永远地运行下去。手动停止线程并做好清理工作永远比直接强制关闭线程来得好。

## 小结

作为开发者，虽然平常跟 RunLoop 直接进行接触的机会不是很多，但是理解它对我们了解 iOS 应用的运行机制也会有很大的帮助。

当然，本篇小结只总结了一些比较基础的知识，旨在对 RunLoop 机制进行一下梳理，让自己对其有更清晰的认识。关于更加详细及深入的内容，可以参考下面的参考资料。

## 参考资料

* [深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
* [Friday Q&A 2010-01-01: NSRunLoop Internals](https://www.mikeash.com/pyblog/friday-qa-2010-01-01-nsrunloop-internals.html)
* [Run Loops 官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)
