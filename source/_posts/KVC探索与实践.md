---
title: KVC探索与实践
date: 2017-05-23 17:41:56
---
# KVC探索与实践
> KVC通过实现非正式协议`NSKeyValueCoding`，来实现间接访问和设置对象属性的一种机制。

使用KVC你可以做到以下一些操作：

##KVC基础（Key-Value Coding Fundamental）
###访问对象属性（Accessing Object Properties）
* 获取值可以用以下方法:

    ```
    - (id)valueForKey:(NSString *)key;
    - (id)valueForKeyPath:(NSString *)keyPath;
    - (NSDictionary<NSString *, id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
    ```
* 设置值用以下方法:

 ```
- (void)setValue:(nullable id)value forKey:(NSString *)key;
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;
- (void)setValuesForKeysWithDictionary:(NSDictionary<NSString *, id> *)keyedValues;
```
**注意：**
 1. 如果对象没有找到key，会抛出`NSUndefinedKeyException`,可以重写子类的`setValue:forUndefinedKey:`做一些自定义的逻辑判断。
 2. 如果设置值是nil，会抛出 `NSInvalidArgumentException`，`可以重写setNilValueForKey:`做一些自定义的逻辑判断。
###访问集合属性
返回的值是集合，key对应的属性可以使任何值。涉及的方法如下：

**NSMutableArray**

```
mutableArrayValueForKey: 
mutableArrayValueForKeyPath:
```
**NSMutableSet**


```
mutableSetValueForKey:
mutableSetValueForKeyPath:
```

**NSMutableOrderSet**

```
mutableOrderedSetValueForKey:
mutableOrderedSetValueForKeyPath:
```

###使用集合操作符
使用集合操作符，可以对集合属性进行一下计算操作，比如计算平均值，总数等。使用的时候有两种方式

```
// Employee.h
@property (nonatomic, assign) NSInteger age;

// Department.h
@property (nonatomic, strong) NSArray<Employee *> *allEmployees;

// client.m
Department *depart = [Department new];
 
// 第一种方式
NSNumber *number = [depart.allEmployees valueForKeyPath:@"@sum.age"];

// 第二种方式
NSNumber *number = [depart valueForKeyPath:@"allEmployees.@sum.age"];
```

在KVC中使用操作符的格式如下(@count除外)：

![](media/15217728079050/15217758003899.jpg)

* 左路径 (left key path):指向是集合类型的属性
* 操作符 (colletion operator): 以'@'开始
* 右路径 (right key path):


**操作符的种类：**

1. **聚合操作符（Aggregation Operators）：**返回单个值，比如`@count`、`@avg`、`@max`、`@min`、`@sum`
2. **数组操作符（Array Operators）：**返回值是数组，比如`@distinctUnionOfObjects`、`@unionOfObjects`，两者区别是@distinctUnionOfObjects没有重复值
3. **嵌入操作符（Nesting Operators）：**返回值是NSArray或者NSSet,比如`@distinctUnionOfArrays`、`@unionOfArrays`、`@distinctUnionOfSets`。实例：

    ```
    NSArray *arrayOne = @[.....];
    NSArray *arrayOther = @[.....];
    NSArray *concactArray = @[arrayOne, arrayOther];
    
    NSArray *collectedDistinctPayees = [concactArray valueForKeyPath:@"@distinctUnionOfArrays.payee"];
```

###非对象的数据
基本的数据类型如`int`、`float`等，需要转换成`NSNumber`类型，`struct`需要转换成`NSValue`类型（针对Objective-C）

###键值验证（Validating Properties）
KVC提供了可以检测设置的值的类型是否符合属性类型的要求的方法：

```
- (BOOL)validateValue:(inout id _Nullable * _Nonnull)ioValue forKey:(NSString *)inKey error:(out NSError **)outError;
- (BOOL)validateValue:(inout id _Nullable * _Nonnull)ioValue forKeyPath:(NSString *)inKeyPath error:(out NSError **)outError;
```
子类可以根据自己的校验逻辑重写这个方法。

###KVC设值和取值的流程

<br>
#####通过Getter获取值
当调用方法`valueForKey:`时，按照以下流程处理：

1. 按照`get<Key>`, `<key>`, `is<Key>`, `_<key>`的顺序寻找方法，如果寻找到了，跳转到步骤5，否则进行步骤2。
2. 如果对象实现了`countOf<Key>`，同时实现了 `objectIn<Key>AtIndex:`，`<key>AtIndexes:` 其中的一个。对于方法 `get<Key>:range:`的是实现是**可选**的。如果实现了，那么就会生成一个集合代理对象（其实就是一个`NSMutatableArray`的子类`NSKeyValueMutableArray`），这个`集合代理对象`和`NSArray`的操作没什么区别。并返回这个集合代理对象。否则，进行步骤3。
3. 如果对象同时实现了`countOf<Key>`, `enumeratorOf<Key>,` and `memberOf<Key>:`那么就会生成一个集合代理对象（其实就是一个`NSSet`的子类`NSKeyValueSet`），这个`集合代理对象`和`NSSet`的操作没什么区别。并返回这个集合代理对象。否则，进行步骤4。
4. 查看对象的类方法 `accessInstanceVariablesDirectly` 返回的值是`YES`,那么按照顺序寻找实例变量` _<key>`, `_is<Key>`, `<key>`, 或者`is<Key>`,然后直接获取值，进行步骤5。否则进行步骤6。
5. 如果是`对象类型`，直接返回，如果是`int`、`struct`等数据类型封装成`NSNumber`或者`NSValue`。
6. 触发`valueForUndefinedKey`:方法，抛出异常。

<br>
#####通过Setter设值
当调用方法`setValue:forKey:`时，按照以下流程处理：

1. 安装顺序寻找方法 `set<Key>:` ， `_set<Key>`，如果找到了把输入的值当做参数传入其中。
2. 查看对象的类方法 `accessInstanceVariablesDirectly` 返回的值是`YES`,那么按照顺序寻找实例变量` _<key>`, `_is<Key>`, `<key>`, 或者`is<Key>`,然后直接获赋值。
3. 触发`valueForUndefinedKey`:方法，抛出异常。

<br>
#####返回可变数组/Mutable Ordered Sets
当调用方法` mutableArrayValueForKey:`或者` mutableOrderedSetValueForKey: `时，按照以下流程处理：

1. 判断是否实现以下方法

    ```
    // 这两个方法和 NSMutableArray/NSMutableOrderedSet 的方法 insertObject:atIndex: 和 removeObjectAtIndex: 对应
    insertObject:in<Key>AtIndex:
    removeObjectFrom<Key>AtIndex:
    
    // 这两个方法和 NSMutableArray/NSMutableOrderedSet 的方法 insertObjects:atIndexes: 和 removeObjectsAtIndexes: 对应
    insert<Key>:atIndexes: 
    remove<Key>AtIndexes:
    ```
      中的一个**插入**和**移除**方法。如果实现了，那么就返回一个代理对象。否则进行步骤2。而对于方法
        
    ```
    replaceObjectIn<Key>AtIndex:withObject: 
    //或者
    replace<Key>AtIndexes:with<Key>:
    ```
    可以增强效果。
2. 寻找 `set<Key>:`方法。
3. 如果`array / mtuableOrderSet`的方法和 `set<Key>:`都没有寻找到。查看对象的类方法 `accessInstanceVariablesDirectly` 返回的值是`YES`,那么按照顺序寻找实例变量` _<key>`, `<key>`,如果找到了，会生成一个代理对象，这个对象会把NSMutableArray的方法发送给这个实例变量。
4.  触发`valueForUndefinedKey`:方法，抛出异常。


<br>
#####返回mutableSet
当调用方法`mutableSetValueForKey:`时，按照以下流程处理：

1. 判断是否实现以下方法

    ```
    // 这两个方法和  NSMutableSet 的方法  addObject: 和  removeObject: 对应
    add<Key>Object:
    remove<Key>Object:
     
    // 这两个方法和  NSMutableSet 的方法  unionSet: 和  minusSet: 对应
    add<Key>:
    remove<Key>: 
    ```
    中的一个**添加**和**移除**方法。如果实现了，那么就返回一个代理对象。否则进行步骤2。而对于方法
 
    ```
    intersect<Key>:
    set<Key>:
    ```
可以增强效果。

2. 如果响应 `mutableSetValueForKey:`的对象是一个`managed object`（像` CoreData的Managed Object`）,查询将停止。
3. 寻找 `set<Key>:`方法。
4. 如果`array / mtuableOrderSet`的方法和 `set<Key>:`都没有寻找到。查看对象的类方法 `accessInstanceVariablesDirectly` 返回的值是`YES`,那么按照顺序寻找实例变量` _<key>`, `<key>`,如果找到了，会生成一个代理对象，这个对象会把NSMutableArray的方法发送给这个实例变量。
5.  触发`valueForUndefinedKey`:方法，抛出异常。

<br>
参考资料：
[Key-Value Coding Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html#//apple_ref/doc/uid/20000955-CJBBBFFA)

