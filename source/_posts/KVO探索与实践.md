---
title: KVO探索与实践
date: 2017-04-12 17:41:56
---
# KVO探索与实践

###	基础

```objc
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context
```
这里有两个需要掌握的知识点：
#### `1. options`
* NSKeyValueObservingOptionNew：最新的`value`
* NSKeyValueObservingOptionOld：之前的`value`
* NSKeyValueObservingOptionInitial: 在注册成观察者之前就会向观察者发送一条同时
* NSKeyValueObservingOptionPrior: 在调用
`- (void)willChangeValueForKey:(NSString *)key`之前被调用

#### `2. context`
这个参数可以给观察者发送一些数据，多用于区分`keyPath`，当然是用响应通知的KeyPath也能区分。

```objc
void *ChangeNameContenxt = &ChangeNameContenxt;

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    //way one
    if ([keyPath isEqualToString:@"name"]) {
        
    }
    //or
    //way two
    if (context == ChangeNameContenxt) {
        
    }
}
```

<br>
### 实践探索

#### To-One

**1. 自动发送（Automatic Change Notification）**

这里就是一般使用的KVO的场景，省略。
<br>
**2. 手动发送（ Manual Notification）**
首先要重写方法：

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
```
根据key判断哪些属性需要手动触发KVO,它还可以用另外一个简单的方法代替：

```objc
+ (BOOL)automaticallyNotifiesObserversOf*Key*
```
此处的key就是需要手动触发KVO的属性。另外，还需要在setter里面手动触发消息，在赋值之前调用：

```objc
- (void)willChangeValueForKey:(NSString *)key;
```

在赋值完成后调用：

```objc
- (void)didChangeValueForKey:(NSString *)key;
```
<br>
#### To-Many
了解这个知识点之前，需要知道`NSKeyValueChange`:

* `NSKeyValueChangeSetting`: 赋值通知（默认）
* `NSKeyValueChangeInsertion`: 插入通知
* `NSKeyValueChangeRemoval`: 删除通知
* `NSKeyValueChangeReplacement`: 替换通知

当需要观察的属性是一个`Collection`的时候，单纯的观察属性是不在`Collection`进行`添加`、`删除`、`替换`操作是能获取到通知。所以需手动触发KVO.
首先要重写方法：

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key
或者
+ (BOOL)automaticallyNotifiesObserversOf*Key*
```
然后要在改变之前调用：

```objc
- (void)willChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
```
改变之后调用

```objc
- (void)didChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key;
```
这两个接口需要传入一个`NSIndexSet`对象，来确定进行`添加`、`删除`、`替换`操作的位置。
<br>
#### 相互影响的属性
有些情况下，除了改变被观察的属性发生变化要发通知之外，修改某一属性的值也可能影响到被观察的属性，也要触发KVO通知。
对于这种情况，需要重写：

```objc
+ (NSSet<NSString *> *)keyPathsForValuesAffectingValueForKey:(NSString *)key
 或者
 + (NSSet<NSString *> *)keyPathsForValuesAffecting*Key*
```
把相关的属性通过`NSSet`返回。除此之外，还需要重写被观察属性的`getter`方法，表明被观察属性和影响属性的关系**（如果不写：那么接收KVO的方法返回的值就不会和影响属性有关系）**。
例如，姓名由姓和名组成，那么改变姓或者名，那么这个姓名就会发生变化。

```objc
// .h
@property (nonatomic, strong) NSString *fullName;
@property (nonatomic, strong) NSString *firstName;
@property (nonatomic, strong) NSString *lastName;

// .m
+ (NSSet<NSString *> *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"firstName", @"lastName", nil];
}

- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@-%@", _firstName, _lastName];
}
```

在观察的接受方法中：

```objc
- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change context:(nullable void *)context
```
就会接收到`fullName`改变的通知

<br>
#### 深层嵌套属性
> 假象一种场景：一个部门有一些工作者，当他们的工资发生变化是，统计部门的总工资也要变化。解决这一问题有两种方法；

***解决方法一：*** 先分析，这里有两个模型 `Department`、`Employee`

```objc
// Department.h
@property (nonatomic, strong) NSArray <Employee *> *employees;
@property (nonatomic, assign) NSInteger totalSalary;
```


```
// Employee.h
@property (nonatomic, assign) NSInteger salary;
```

在初始化`Department`时，把`Department`注册成为`Employee`的观察者。
例如：

```objc
// Department.m (init)
 - (instancetype)init {
    self = [super init];
    if (self) {
        //array
        NSMutableArray *array = [NSMutableArray array];
        for (int i = 0 ; i < 10; i++) {
            Employee *employe = [Employee new];
            employe.salary = i;
            [employe addObserver:self forKeyPath:@"salary" options:NSKeyValueObservingOptionNew context:nil];
            [array addObject:employe];
        }
        _employees = [array copy];
    }
    return self;
}
```

然后在接收KVO通知方法里面：

```obj-c
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    [self setTotalSalary:[[self valueForKeyPath:@"employees.@sum.salary"] integerValue]];
}
```
这样观察`Department`的`totalSalary`属性就可以观察到变化了。

***解决方法二：***使用`CoreData`来实现。
<br>
### 原理分析
正如苹果官方文档里面的描述：

>Automatic key-value observing is implemented using a technique called isa-swizzling.
>
>The isa pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.
>
>When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.
>
>You should never rely on the isa pointer to determine class membership. Instead, you should use the class method to determine the class of an object instance.

当某一属性被观察时，会生成一个中间类**（NSKVONotifying_原类名）**，改中间类会做一下处理

* 重写`setter`，手动发送KVO通知。
* 重写`class`方法，隐藏真实类名，不过可以用`object_getClass()`获取。
* 添加方法`- (BOOL)_isKVOA`判断被动态生成KVO子类

<br>
### 注意
<div class="tip">
	<div>1. 在被观察对象释放的时候需要移除KVO</div>
	<div>2. 一个需要注意的地方是，KVO 行为是同步的，并且发生与所观察的值发生变化的同样的线程上。</div>
</div>

<br>

######  资料参考：
 * [Key-Value Observing Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)
 * [objc kvo简单探索](http://blog.sunnyxx.com/2014/03/09/objc_kvo_secret/)
	

