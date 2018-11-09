---
title: RAC Signal流程解析
date: 2017-10-09 15:03:33
tags:
---

# RAC Signal流程解析
> **ReactiveCocoa**是一个响应式的开源库，它把通知、代理、KVO等OC的技术整合在一起。这样写出的代码更简介，更优雅。尤其是如果你是用MVVM架构来开发的项目，有了它，你就能轻松并优雅的解决View和VeiwModel之间传值和更新的问题。

### 一. 信号的创建
创建一个信号最基本的方法是用`createSignal:`方法：
```objc
RACSignal *signal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
    [subscriber sendNext:@"hello RAC"];
    [subscriber sendCompleted];
    return nil;
}];
```
我们来看看这个方法内部的实现。
```objc
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	return [RACDynamicSignal createSignal:didSubscribe];
}
```
里面调用了`RACDynamicSignal`的`createSignal:`方法，进入里面，实现如下：
```objc
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
	signal->_didSubscribe = [didSubscribe copy];//这里会copy block
	return [signal setNameWithFormat:@"+createSignal:"];
}
```
这里需要注意的是返回的`RACDynamicSignal`会持有`didSubscribe` block,所以这里要注意在创建RACSignal的时候，是否会引`起循环引用`。
创建的过程比较简单。下面看看信号的订阅。

### 二. 信号的订阅
信号订阅会使用方法 `subscribeNext: error: completed:`，同时RAC也提供了这三个时间单独或者两两组合的方法，非常方便。使用如下:
```objc
[signal subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@", x);
} error:^(NSError * _Nullable error) {
    NSLog(@"error");
} completed:^{
    NSLog(@"completed");
}];
```
当对应的事件触发的时候，相应的block会调用。这个方法的实现如下：

```objc
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock error:(void (^)(NSError *error))errorBlock completed:(void (^)(void))completedBlock {
	NSCParameterAssert(nextBlock != NULL);
	NSCParameterAssert(errorBlock != NULL);
	NSCParameterAssert(completedBlock != NULL);
	
	RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:errorBlock completed:completedBlock];
	return [self subscribe:o];
}
```
这里面会生成`RACSubscriber`对象，`RACSubscriber`的创建方法内部实现是：
```objc
+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
	RACSubscriber *subscriber = [[self alloc] init];

	subscriber->_next = [next copy];
	subscriber->_error = [error copy];
	subscriber->_completed = [completed copy];

	return subscriber;
}
```
它也会把所有的block参数copy。同时这个类还实现了RACSubscriber协议,这个协议里面有四个方法
```objc
- (void)sendNext:(nullable id)value;	//next事件
- (void)sendError:(nullable NSError *)error; //错误事件
- (void)sendCompleted; //完成事件
- (void)didSubscribeWithDisposable:(RACCompoundDisposable *)disposable;
```
然后把`RACSubscriber`的对象作为参数，调用了`RACDynamicSignal`的 `subscribe:`方法，这个方法的实现：
```objc
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);

	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}
	
	return disposable;
}
```
这个方法使用`RACSignal`、`subscriber`、`RACCompoundDisposable`创建了一个`RACPassthroughSubscriber`实例，创建方法实现如下：
```objc
- (instancetype)initWithSubscriber:(id<RACSubscriber>)subscriber signal:(RACSignal *)signal disposable:(RACCompoundDisposable *)disposable {
	NSCParameterAssert(subscriber != nil);

	self = [super init];

	_innerSubscriber = subscriber;
	_signal = signal;
	_disposable = disposable;

	[self.innerSubscriber didSubscribeWithDisposable:self.disposable];//这里把disposable传递给subscriber，让整个清除处理操作连成一条线。
	return self;
}
```
这是一个简单的创建过程，唯一值得注意的是调用了`didSubscribeWithDisposable:`这个方法，在`RACSubscriber`里面这个方法的实现如下：
```objc
- (void)didSubscribeWithDisposable:(RACCompoundDisposable *)otherDisposable {
	if (otherDisposable.disposed) return;

	RACCompoundDisposable *selfDisposable = self.disposable;
	[selfDisposable addDisposable:otherDisposable];

	@unsafeify(otherDisposable);

	// If this subscription terminates, purge its disposable to avoid unbounded
	// memory growth.
	[otherDisposable addDisposable:[RACDisposable disposableWithBlock:^{
		@strongify(otherDisposable);
		[selfDisposable removeDisposable:otherDisposable];
	}]];
}
```
作用就是把整个清除处理操作连成一条线。

回到上一个方法里面，把创建好的`RACPassthroughSubscriber`实例作为参数调用`self.didSubscribe(subscriber)`（这里可以看出在订阅的block里面不会有循环引用问题），这个就是创建步骤里面`RACDynamicSignal`的`_didSubscribe`变量，也就是`createSignal:`方法的block。在block里面会调用`RACPassthroughSubscriber`的`sendNext:`和`sendCompleted`方法，它在`RACPassthroughSubscriber`的实现是：
```objc
```
```objc
- (void)sendNext:(id)value {
	if (self.disposable.disposed) return;

	if (RACSIGNAL_NEXT_ENABLED()) {
		RACSIGNAL_NEXT(cleanedSignalDescription(self.signal), cleanedDTraceString(self.innerSubscriber.description), cleanedDTraceString([value description]));
	}

	[self.innerSubscriber sendNext:value];
}

- (void)sendCompleted {
	if (self.disposable.disposed) return;

	if (RACSIGNAL_COMPLETED_ENABLED()) {
		RACSIGNAL_COMPLETED(cleanedSignalDescription(self.signal), cleanedDTraceString(self.innerSubscriber.description));
	}

	[self.innerSubscriber sendCompleted];
}
```
这里判断信号是否被`disposed`了，然后调用`innerSubscriber`（也就是RACSubscriber实例）对应的方法，在`RACSubscriber`里面。对应的方法实现是：
```objc
- (void)sendNext:(id)value {
	@synchronized (self) {
		void (^nextBlock)(id) = [self.next copy];
		if (nextBlock == nil) return;

		nextBlock(value);
	}
}

- (void)sendError:(NSError *)e {
	@synchronized (self) {
		void (^errorBlock)(NSError *) = [self.error copy];
		[self.disposable dispose];

		if (errorBlock == nil) return;
		errorBlock(e);
	}
}

- (void)sendCompleted {
	@synchronized (self) {
		void (^completedBlock)(void) = [self.completed copy];
		[self.disposable dispose];

		if (completedBlock == nil) return;
		completedBlock();
	}
}
```
这里使用了`@synchronized`以防止在多线程下面，调用dispose以致block next、error、completed被置于nil带来的问题。这里的block就是订阅时对应的block。

在执行完了创建`RACSignal`里面`sendNext:`和`sendCompleted`方法之后，会返回一个RACDisposable,用于订阅被清除时执行的操作，比如用来封装网络请求，在RACDisposable里面就可以执行cancel操作。该RACDisposable会被加入到上层的RACDisposable。

最后`- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber`会把`self.didSubscribe(subscriber)`返回的`RACDisposable`加入到创建的`RACCompoundDisposable`实例disposable中去，并返回出去，订阅信号的地方就可以拿到这个`RACDisposable`，在以后有需要的情况下就可以取消订阅了。

<div class="tip">
    <div>1. 在创建信号或者变换信号（调用所有操作方法）的block里面，如果持有了持有信号的对象，就会导致循环引用。</div>
    <div>2. 订阅的block里面不会引起循环引用。  </div>
</div>
