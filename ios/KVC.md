# KVC（Key-Value Coding）

## 键值编码

键值编码允许开发者通过名字访问对象属性。这是cocos的运行时特性。

比如：

```objc
@interface KVCObject : NSObject
@property (nonatomic, copy) NSString *property;
@end

@try {
    DataModelObj *model = [[DataModelObj alloc] init];
    NSArray *items = [model valueForKey:@"items"];
    NSUInteger count = [items count];
}
@catch (NSException *exception) { NSLog(@"%@", exception); }
@finally {}
// 等同于：obj._property;
// 也可以用：NSString *prop = [obj valueForKeyPath:@"property"];

[obj setValue:@"some prop" forKey:@"property"]; // 使用KVC
```
方法valueForKey:返回对象指定key的值，如果没有这个key，抛出异常。

可以通过名字（字符串）访问对象属性，这带来了很大的灵活性，比如可以通过**脚本**获得对象的属性值。就是KVC最常用的序列化和反序列化对象。

> 注意：valueForKey和valueForKeyPath的返回值都是id，对于NSNumber *number来说，[number valueForKey:@"intValue"]相当于[NSNumber numberWithInt:[number intValue]]，最后返回的是NSNumber对象。

KVC方法有key和keyPath两个版本，比如valueForKey:和valueForKeyPath:。区别在于，后者可以包含嵌套关系，用句点分开。后者更灵活，前者稍快些。 

## KVC获取对象属性值

* **valueForKey:** 返回对象指定key的值，如果没有这个key，对象会给自己发送 valueForUndefinedKey:消息。默认对这个消息的处理是抛出异常。
* **valueForKeyPath:** 返回对象指定key path的值。对没有值的情况处理同上。如果keyPath的第一个key返回的是个集合，那么会遍历这个集合，最终将所有结果放在一个集合中返回。
* **dictionaryWithValuesForKeys:** 提供一个key的数组，返回值是NSDictionary中保存着对应的key和value值。

处理对象中没有的key值，可以处理异常，也可以在对象中重载对valueForUndefinedKey:消息的处理：

```objc
- (id)valueForUndefinedKey:(NSString *)key {
    NSLog(@"%@", key);
    return [super valueForUndefinedKey:key]; // 抛出异常
}
```

## KVC设置对象属性

* **setValue:forKey:** 给对象名为key的属性赋值。会自动解包代表标量、Struct的NSValue赋值给对象。如果对象不存在名为key的属性值，会给自身发送setValue:forUndefinedKey:消息，对这个消息的默认处理是抛出异常。
* **setValue:forKeyPath:** 给对象指定keyPath的属性赋值。效果同上。
* **setValuesForKeysWithDictionary:** 批量赋值，循环调用setValue:forKey:。

如果给一个非空对象属性设置nil，对象会给自己发送setNilValueForKey:消息，默认处理是抛出异常。可以通过重载来处理:

```objc
- (void)setNilValueForKey:(NSString *)theKey {
    if ([theKey isEqualToString:@"hidden"]) {
        [self setValue:@YES forKey:@"hidden"];
    }
    else {
        [super setNilValueForKey:theKey];
    }
}
```

在被设置对象中重载对setValue:forUndefinedKey:消息的处理：

```objc
- (void)setValue:(id)value forUndefinedKey:(NSString *)key{
    NSLog(@"%@", key);
}
```

## 集合访问器方法（collection accessor methods）

直接读写对象中集合对象的内容，而不需要先拿出操作对象中的集合对象，在操作这个集合。

好处：提高性能；模糊与NSArray或NSSet的差别；兼容KVO。

### 有序集合访问器

#### 只读索引集合访问器

* -countOf\<Key>: 必须。类似NSArray的count方法。
* -objectIn\<Key>AtIndex: 或 -\<key>AtIndexes: 二选一。对应NSArray的objectAtIndex: and objectsAtIndexes:方法。
* -get\<Key>:range: 可选。可提高性能。对应NSArray的getObjects:range:方法。

```objc
- (NSUInteger)countOfEmployees {
    return [self.employees count];
}

- (id)objectInEmployeesAtIndex:(NSUInteger)index {
    return [employees objectAtIndex:index];
}
 
- (NSArray *)employeesAtIndexes:(NSIndexSet *)indexes {
    return [self.employees objectsAtIndexes:indexes];
}

- (void)getEmployees:(Employee * __unsafe_unretained *)buffer range:(NSRange)inRange {
    // Return the objects in the specified range in the provided buffer.
    // For example, if the employees were stored in an underlying NSArray
    [self.employees getObjects:buffer range:inRange];
}
```

比如当我们调用 [object valueForKey:@"employees”] 的时候，它会返回一个由上面方法来代理所有调用方法的 NSArray 对象。这个数组支持所有正常的对 NSArray 的调用。换句话说，调用者并不知道返回的是一个真正的 NSArray， 还是一个代理的数组。

#### 可变索引集合访问器

* -insertObject:in\<Key>AtIndex: 或 -insert\<Key>:atIndexes: 二选一。对应NSMutableArray的insertObject:atIndex: 和 insertObjects:atIndexes:。
* -removeObjectFrom\<Key>AtIndex: or -remove\<Key>AtIndexes:. 二选一。对应NSMutableArray的 removeObjectAtIndex: 和 removeObjectsAtIndexes:。
* -replaceObjectIn\<Key>AtIndex:withObject: 或 -replace\<Key>AtIndexes:with\<Key>: 可选。可提高性能。

```objc
- (void)insertObject:(Employee *)employee inEmployeesAtIndex:(NSUInteger)index {
    [self.employees insertObject:employee atIndex:index];
    return;
}
 
- (void)insertEmployees:(NSArray *)employeeArray atIndexes:(NSIndexSet *)indexes {
    [self.employees insertObjects:employeeArray atIndexes:indexes];
    return;
}

- (void)removeObjectFromEmployeesAtIndex:(NSUInteger)index {
    [self.employees removeObjectAtIndex:index];
}
 
- (void)removeEmployeesAtIndexes:(NSIndexSet *)indexes {
    [self.employees removeObjectsAtIndexes:indexes];
}

- (void)replaceObjectInEmployeesAtIndex:(NSUInteger)index
                             withObject:(id)anObject {
 
    [self.employees replaceObjectAtIndex:index withObject:anObject];
}
 
- (void)replaceEmployeesAtIndexes:(NSIndexSet *)indexes
                    withEmployees:(NSArray *)employeeArray {
 
    [self.employees replaceObjectsAtIndexes:indexes withObjects:employeeArray];
}
```

### 无序集合

#### 只读无序集合访问器

* -countOf\<Key> 必须。对应NSSet的count方法。
* -enumeratorOf\<Key> 必须. 对应NSSet的objectEnumerator方法。
* -memberOf\<Key>: 必须。对应NSSet的member:方法。

```objc
- (NSUInteger)countOfTransactions {
    return [self.transactions count];
}
 
- (NSEnumerator *)enumeratorOfTransactions {
    return [self.transactions objectEnumerator];
}
 
- (Transaction *)memberOfTransactions:(Transaction *)anObject {
    return [self.transactions member:anObject];
}
```

#### 可变无序集合访问器

* -add\<Key>Object: 或 -add\<Key>: 二选一。对应NSMutableSet的addObject:。
* -remove\<Key>Object: 或 -remove\<Key>: 。对应NSMutableSet的removeObject:。
* -intersect\<Key>: 可选的。可提高性能。对应NSSet的intersectSet:。

```objc
- (void)addTransactionsObject:(Transaction *)anObject {
    [self.transactions addObject:anObject];
}
 
- (void)addTransactions:(NSSet *)manyObjects {
    [self.transactions unionSet:manyObjects];
}

- (void)removeTransactionsObject:(Transaction *)anObject {
    [self.transactions removeObject:anObject];
}
 
- (void)removeTransactions:(NSSet *)manyObjects {
    [self.transactions minusSet:manyObjects];
}

- (void)intersectTransactions:(NSSet *)otherObjects {
    return [self.transactions intersectSet:otherObjects];
}
```

## 键值验证

KVC提供了验证Key对应的Value是否可用的方法：

```objc
- (BOOL)validateValue:(inout id *)ioValue forKey:(NSString *)inKey error:(out NSError **)outError;  
```
该方法默认的实现是调用validate\<Key>:error:。比如对属性name：

```objc
-(BOOL)validateName:(id *)ioValue error:(NSError * __autoreleasing *)outError {
    // Implementation specific code.
    return ...;
}
```

需要指出的是，KVC是不会自动调用键值验证方法的，就是说我们需要手动验证。但是有些技术，比如CoreData会自动调用。

可以在设置Value之前调用validateValue:forKey:error:验证对应key的value是否可用，在对象里重载validate\<Key>:error:做具体判断。具体判断的方式参考：[Key-Value Validation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/Validation.html#//apple_ref/doc/uid/20002173-CJBDBHCB)

## 数值和结构体支持

KVC可以自动的将数值或结构体型的数据打包或解包成NSNumber或NSValue对象。

* valueForKey:对于指定key的属性或访问器方法的返回值，不是对象的，就创建NSNumber或NSValue将值封装后返回。
* setValue:forKey:对于指定key的属性或访问器方法的参数，不是对象，就将参数用-<type>Value方法展开后赋值。
* 当setValue:forKey:如果给一个非空对象属性设置nil，对象会给自己发送setNilValueForKey:消息，默认处理是抛出异常。

自动打包和解包结构体不仅限于NSPoint, NSRange, NSRect, 和 NSSize。任何结构体都是可以转化成NSValue对象的，比如:

```objc
typedef struct {
    float x, y, z;
} ThreeFloats;
 
@interface MyClass
- (void)setThreeFloats:(ThreeFloats)threeFloats;
- (ThreeFloats)threeFloats;
@end

MyClass *mc = [[MyClass alloc] init];
NSValue *value = [mc valueForKey:@"threeFloats"]; // 调用threeFloats，结果用NSValue封装
NSValue *newValue = [NSValue value:&mc withObjCType:@encode(struct ThreeFloats);
[mc setValue:newValue forKey:@"threeFloats"]; // 调用setThreeFloats:，对NSValue对象参数发送getValue:消息解包
```

> 注意：该机制不考虑引用计数和垃圾回收。

## valueForKey的搜索规则

KVC再某种程度上提供了访问器的替代方案。KVC优先使用访问器方法设置或者返回对象属性，其次才直接访问对象属性。以下是KVC如何决定访问一个值：

### setValue:forKey:的默认搜索规则

1. 搜索访问器方法：set\<Key>:
2. 没有访问器方法，且对象方法accessInstanceVariablesDirectly返回YES，按顺序搜索：_\<key>, _is\<Key>, \<key>, is\<Key>
3. 都没有，就调用对象的setValue:forUndefinedKey:

### valueForKey:的默认搜索规则

1. 按顺序搜索：get\<Key>, \<key>, is\<Key>
2. 没有上面的访问器方法，搜索：countOf\<Key> 和 objectIn\<Key>AtIndex: 或 \<key>AtIndexes: ，后者二选一。创建NSArray集合代理对象，所有发送给代理集合对象的消息，都会通过上面这三个方法，发送给valueForValue:的原始对象。
3. 上面的都没有，搜索：countOf\<Key>, enumeratorOf\<Key>, memberOf\<Key>: ，创建NSSet集合代理对象。
4. 否则，对象的方法 accessInstanceVariablesDirectly 返回 YES，按顺序搜索： _\<key>, _is\<Key>, \<key>, is\<Key>
5. 都没有，调用对象的valueForUndefinedKey:

### mutableArrayValueForKey:访问有序集合的搜索规则

1. 搜索：insertObject:in\<Key>AtIndex: 和 removeObjectFrom\<Key>AtIndex: 或insert\<Key>:atIndexes: 和 remove\<Key>AtIndexes:，创建NSMutableArray集合代理对象。
2. 搜索：set\<Key>:，有性能问题，推荐上面的方法。
3. 如果方法accessInstanceVariablesDirectly返回YES，按顺序搜索： _\<key> 或 \<key>,
4. 都没有，调用setValue:forUndefinedKey

> mutableOrderedSetValueForKey: 和 mutableSetValueForKey:的搜索规则与Array的类似，就不一一列举了。



参考：  
[Key-Value Coding Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/KeyValueCoding.html#//apple_ref/doc/uid/10000107i)  
[KVC/KVO原理详解及编程指南](http://blog.csdn.net/wzzvictory/article/details/9674431#)