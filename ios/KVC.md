# KVC

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

## valueForKey的搜索规则

当valueForKey:寻找items时：

```objc
@interface DataModelObj : NSObject
@end

DataModelObj *model = [[DataModelObj alloc] init];
NSArray *items = [model valueForKey:@"items"];
```

1、 如果有名为items，isItems，_items，_isItems的实例变量，KVC会直接访问这些实例变量。下面这些实例变量都能直接访问到：

```obcj
@interface DataModelObj : NSObject {
    NSArray *items;
// 或者命名为：isItems, _items, _isItems
}
// 或者：
// @property (nonatomic, strong) NSArray *items;
// 或者命名为：isItems, _items, _isItems
@end
```

2、如果有名为items，getItems，isItems的方法，会返回找到的第一个方法的值：

```objc
@implementation DataModelObj
// 最上面的方法第一个找到，valueForKey:@"items"返回1
- (NSUInteger)items { return 1; }
- (NSUInteger)getItems { return 2; }
- (NSUInteger)isItems { return 3; }
@end
```
setValue:forKey:会查找setItems方法。

3、如果有方法countOfItems，和objectInItemsAtIndex或itemsAtIndexes二者之一，KVC会创建一个代理数组：

```objc
@implementation DataModelObj
- (NSUInteger)countOfItems { return 10; }
// 下面两个方法二选一即亦可：
- (id)objectInItemsAtIndex:(NSUInteger)index {
    return [NSNumber numberWithInt:index*2];
}
- (NSArray *)itemsAtIndexes:(NSIndexSet *)indexes {
    return [[NSArray alloc] initWithObjects:[NSNumber numberWithInt:1], nil];
}
```
此时valueForKey的返回值是一个包含10个元素（根据countOfItems:方法）的数组，数组的元素是根据objectInItemsAtIndex:和itemsAtIndexes:方法设置的规则返回的。比如依据objectInItemsAtIndex:规则，返回的数组是NSNumber类型的，值为：2、4、6、8...直到20。

4、如果有方法countOfItems，enumeratorOfItems和mumberOfItems三个方法的组合时，KVC返回一个代理集合：

这里就不举代码例子了，因为没有尝试成功过。具体可以看文档：

[Key-Value Coding Accessor Methods](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/AccessorConventions.html#//apple_ref/doc/uid/20002174-BAJEAIEE)

## 键值验证

需要指出的是， KVC是不会自动调用键值验证方法的，就是说我们需要手动验证。但是有些技术，比如CoreData会自动调用。

参考：

[Key-Value Coding Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/KeyValueCoding.html#//apple_ref/doc/uid/10000107i)