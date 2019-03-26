# dispatch_after?NSTimer

<a name="RunLoop"></a>
# RunLoop

ResidentThread

<a name="880b32a3"></a>
### 常驻线程

常驻线程的作用：我们知道，当子线程中的任务执行完毕之后就被销毁了，那么如果我们需要开启一个子线程，在程序运行过程中永远都存在，那么我们就会面临一个问题，如何让子线程永远活着，这时就要用到常驻线程：给子线程开启一个RunLoop<br />注意：子线程执行完操作之后就会立即释放，即使我们使用强引用引用子线程使子线程不被释放，也不能给子线程再次添加操作，或者再次开启。<br />子线程开启RunLoop的代码，先点击屏幕开启子线程并开启子线程RunLoop，然后点击button。----《AFN2.0x 》

<a name="1b89f230"></a>
#### AF2.x为什么需要常驻线程？

NSURLConnection<br />先来看看 NSURLConnection 发送请求时的线程情况，NSURLConnection 是被设计成异步发送的，调用了start方法后，NSURLConnection 会新建一些线程用底层的 CFSocket 去发送和接收请求，在发送和接收的一些事件发生后通知原来线程的Runloop去回调事件。

AFN2.0里面把每一个网络请求的发起和解析都放在了一个线程里执行。正常来说，一个线程执行完任务后就退出了。开启runloop是为了防止线程退出。一方面避免每次请求都要创建新的线程；另一方面，因为connection的请求是异步的，如果不开启runloop，线程执行完代码后不会等待网络请求完的回调就退出了，这会导致网络回调的代理方法不执行。

RunLoop相关知识参考文档[[https://www.yuque.com/litburke/cye5e3/gpiyam](https://www.yuque.com/litburke/cye5e3/gpiyam)];

所有针对AFN2.0下，若来一个请求开辟一条线程，设置runloop保活线程，等待结果回调。这种方式理论上是可行的，线程开销太大了，因此我们在实际开发中只开辟一条子线程（保活），设置runloop使线程常驻，所有的请求在这个线程上发起、同时也在这个线程上回调。

```objectivec
//networkRequestThread即常驻线程
[self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
```

```objectivec
- (void)operationDidStart {
    [self.lock lock];
    if (![self isCancelled]) {
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        for (NSString *runLoopMode in self.runLoopModes) {
            [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];
            [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
        }
        [self.outputStream open];
        [self.connection start];
    }
    [self.lock unlock];
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidStartNotification object:self];
    });
}
```

首先，每一个请求对应一个AFHTTPRequestOperation实例对象（以下简称operation），每一个operation在初始化完成后都会被添加到一个NSOperationQueue中。<br />由这个NSOperationQueue来控制并发，系统会根据当前可用的核心数以及负载情况动态地调整最大的并发 operation 数量，我们也可以通过setMaxConcurrentoperationCount:方法来设置最大并发数。注意：并发数并不等于所开辟的线程数。具体开辟几条线程由系统决定。也就是说此处执行operation是并发的、多线程的。

![](https://cdn.nlark.com/yuque/0/2019/png/161326/1553566601301-19313695-f932-4813-971c-d8c2b066646d.png#align=left&display=inline&height=546&originHeight=546&originWidth=573&size=0&status=done&width=573)


