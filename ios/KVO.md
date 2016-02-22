# KVO (Key-Value Observing)

KVO提供了让对象监控另一个对象属性的机制，特别用于model和controller的交流。支持一对多。

## 设置属性观察的步骤：

1. 应用场景，一个对象的指定属性任何变化都被另一个对象观察。  
![scenario](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Art/kvo_objects.jpg)  
PersonObject想要得到BankObject的accountBalance属性变化信息。  

2. PersonObject发送 addObserver:forKeyPath:options:context: 消息，注册为BankObject的accountBalance属性的观察者。   
![registe](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Art/kvo_objects_connection.jpg)  
他们建立的关系存在于对象上，而不存在与类上。

3. 观察者对象实现observeValueForKeyPath:ofObject:change:context: 方法接受变化通知。  
![respond](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Art/kvo_objects_implementation.jpg)  

4. 当被观察的对象方式以KVC兼容方式改变时， observeValueForKeyPath:ofObject:change:context: 方法自动调用： 
![invoked](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Art/kvo_objects_notification.jpg)

5. 给被观察的对象发送removeObserver:forKeyPath: 消息，移除KVO。

### 注册观察者的注意

注册观察者，给被观察对象发送addObserver:forKeyPath:options:context: 消息。

其参数：change可以是NSKeyValueObservingOptionOld（观察者获得改变之前的旧值）或 NSKeyValueObservingOptionNew（观察者获得改变之后的新值）或者二者的合体（OR）。

参数：context可以让观察者提供上下文指针。这个指针可以是C指针或者对象的引用。可以作为观察的唯一标示。

```objc
- (void)registerAsObserver {
    /*
     Register 'inspector' to receive change notifications for the "openingBalance" property of
     the 'account' object and specify that both the old and new values of "openingBalance"
     should be provided in the observe… method.
     */
    [account addObserver:inspector
             forKeyPath:@"openingBalance"
                 options:(NSKeyValueObservingOptionNew |
                            NSKeyValueObservingOptionOld)
                    context:NULL];
}
```

> 注意：此方法不保存observing、被观察对象、context对象的强引用。

### 收到变化通知

当被观察的对象属性发生变化时，观察者收到observeValueForKeyPath:ofObject:change:context:消息。

观察者收到包含变化细节的字典，和注册时的上下文指针。

变化字典的NSKeyValueChangeKindKey条目提供变化发生条件的信息。如果被观察的属性改变，则其值为：NSKeyValueChangeSetting。

一对一情况下，条目：NSKeyValueChangeOldKey和NSKeyValueChangeNewKey分别保存变化之前和之后的值。

如果是一对多：

条目NSKeyValueChangeKindKey的值在inserted, removed, replaced对应值为 NSKeyValueChangeInsertion, NSKeyValueChangeRemoval, NSKeyValueChangeReplacement。

NSKeyValueChangeIndexesKey是一个NSIndexSet对象，返回变化的索引。如果注册时指定了 NSKeyValueObservingOptionNew 或 NSKeyValueObservingOptionOld，条目NSKeyValueChangeOldKey 和 NSKeyValueChangeNewKey 包含了变化前和变化后对象的数组。


```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
 
    if ([keyPath isEqual:@"openingBalance"]) {
        [openingBalanceInspectorField setObjectValue:
            [change objectForKey:NSKeyValueChangeNewKey]];
    }
    /*
     Be sure to call the superclass's implementation *if it implements it*.
     NSObject does not implement the method.
     */
    [super observeValueForKeyPath:keyPath
                         ofObject:object
                           change:change
                           context:context];
}
```

## KVO的特点

1. 观察者方案不需要自己实现，完全由框架实现，不需要额外的代码。
2. 不同于NSNotificationCenter，没有中央对象通知每个观察者对象，而是在变化时直接发给观察者对象。

## KVO Compliance

要成为被观察的对象，必须满足的规则。

被观察的类必须满足KVC Compliance，KVO支持与KVC相同的数据类型。 


### 自动变化通知

NSObject支持自动key-value变化通知。用访问者方法、KVC方法、集合代理对象都会触发自动通知。

```objc
/ Call the accessor method.
[account setName:@"Savings"];
 
// Use setValue:forKey:.
[account setValue:@"Savings" forKey:@"name"];
 
// Use a key path, where 'account' is a kvc-compliant property of 'document'.
[document setValue:@"Savings" forKeyPath:@"account.name"];
 
// Use mutableArrayValueForKey: to retrieve a relationship proxy object.
Transaction *newTransaction = <#Create a new transaction for the account#>;
NSMutableArray *transactions = [account mutableArrayValueForKey:@"transactions"];
[transactions addObject:newTransaction];
```

### 手动变化通知

手动变化通知提供了更细粒度的控制，可以最大限度避免不必要的通知，或者将多个通知归结为一个。

要实施手动通知，需要重载NSObject的automaticallyNotifiesObserversForKey:方法。在类中可能需要自动和手动通知同时存在，需要手动通知的属性，上面的重载函数返回NO。

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)theKey {
 
    BOOL automatic = NO;
    if ([theKey isEqualToString:@"openingBalance"]) {
        automatic = NO;
    }
    else {
        automatic = [super automaticallyNotifiesObserversForKey:theKey];
    }
    return automatic;
}
```

为实现手动通知，需要在值改变前调用willChangeValueForKey:，在值改变后didChangeValueForKey: 。

```objc
- (void)setOpeningBalance:(double)theBalance {
    if (theBalance != _openingBalance) { // 确保值确实变化了
        [self willChangeValueForKey:@"openingBalance"];
        _openingBalance = theBalance;
        [self didChangeValueForKey:@"openingBalance"];
    }
}
```

如果单个操作导致多个值发生变化：

```objc
- (void)setOpeningBalance:(double)theBalance {
    [self willChangeValueForKey:@"openingBalance"];
    [self willChangeValueForKey:@"itemChanged"];
    _openingBalance = theBalance;
    _itemChanged = _itemChanged+1;
    [self didChangeValueForKey:@"itemChanged"];
    [self didChangeValueForKey:@"openingBalance"];
}
```

对于一对多关系，不但要指定变化对象属性的key，还要指定类型（NSKeyValueChange类型）和索引。

```objc
- (void)removeTransactionsAtIndexes:(NSIndexSet *)indexes {
    [self willChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
 
    // Remove the transaction objects at the specified indexes.
 
    [self didChange:NSKeyValueChangeRemoval
        valuesAtIndexes:indexes forKey:@"transactions"];
}
```

## Registering Dependent Keys

### 一对一关联

很多情况下，一个属性值依赖于其他对象的多个属性值。

方案一：在观察对象类里重载keyPathsForValuesAffectingValueForKey:方法，

```objc
// fullName 依赖属性firstName和lastName
- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@",firstName, lastName];
}

+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
 
    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
 
    if ([key isEqualToString:@"fullName"]) {
        NSArray *affectingKeys = @[@"lastName", @"firstName"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}
```
方法返回一个set，里面包含所有key依赖的属性名。

方案二：创建一个方法形如keyPathsForValuesAffecting\<Key>，依旧返回key的所有依赖属性名Set：

```objc
+ (NSSet *)keyPathsForValuesAffectingFullName {
    return [NSSet setWithObjects:@"lastName", @"firstName", nil];
}
```
这样可以在没有办法重载方法的情况下，比如在分类（category）中实施。

### 一对多关联

当一个属性依赖于一系列对象的某个属性的时候。

keyPathsForValuesAffectingValueForKey:方法不支持一对多关系的key-paths，比如有个Department（部门）对象，包含多个Employee（雇员），后者有salary（工资）属性。可能需要Department对象有totalSalary属性，与每个员工的工资相关。

这里有连个解决方案：

方案一：用KVO注册他们的父级（就是Department）为每个Employee的观察者，添加和移除雇员的时候注意add和remove观察者对象，在接受变化的方法observeValueForKeyPath:ofObject:change:context:中更新totalSalary。

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
 
    if (context == totalSalaryContext) {
        [self updateTotalSalary];
    }
    else
    // deal with other observations and/or invoke super...
}
 
- (void)updateTotalSalary {
    [self setTotalSalary:[self valueForKeyPath:@"employees.@sum.salary"]];
}
 
- (void)setTotalSalary:(NSNumber *)newTotalSalary {
 
    if (totalSalary != newTotalSalary) {
        [self willChangeValueForKey:@"totalSalary"];
        _totalSalary = newTotalSalary;
        [self didChangeValueForKey:@"totalSalary"];
    }
}
 
- (NSNumber *)totalSalary {
    return _totalSalary;
}
```

方案二：如果使用Core Data，可以在app通知中心，注册父级为他管理的相关对象的观察者，父级应当以类似KVO的方式返回孩子对象通知的相应变化。

## 实现原理

KVO是由isa-swizzling技术实现的。

当某个类的对象第一次被观察时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的 setter 方法。

派生类在被重写的 setter 方法实现真正的通知机制，就如前面手动实现键值观察那样。这么做是基于设置属性会调用 setter 方法，而通过重写就获得了 KVO 需要的通知机制。当然前提是要通过遵循 KVO 的属性设置方式来变更属性值，如果仅是直接修改属性对应的成员变量，是无法实现 KVO 的。

同时派生类还重写了 class 方法以“欺骗”外部调用者它就是起初的那个类。然后系统将这个对象的 isa 指针指向这个新诞生的派生类，因此这个对象就成为该派生类的对象了，因而在该对象上对 setter 的调用就会调用重写的 setter，从而激活键值通知机制。此外，派生类还重写了 dealloc 方法来释放资源。

```objc
#import <objc/runtime.h>

@interface Foo : NSObject
@property int x;
@property int y;
@property int z;
@end

@implementation Foo
@end

static NSArray * ClassMethodNames(Class c)
{
    NSMutableArray * array = [NSMutableArray array];
    
    unsigned int methodCount = 0;
    Method * methodList = class_copyMethodList(c, &methodCount);
    unsigned int i;
    for(i = 0; i < methodCount; i++) {
        [array addObject: NSStringFromSelector(method_getName(methodList[i]))];
    }
    
    free(methodList);
    
    return array;
}

static void PrintDescription(NSString * name, NSObject *obj)
{
    Class cls = obj.class;
    Class isa = object_getClass(obj);
    NSString * str = [NSString stringWithFormat:
                      @"\n\t%@: %@\n\tNSObject class %s\n\tlibobjc class %s\n\timplements methods <%@>",
                      name,
                      obj,
                      class_getName(cls),
                      class_getName(isa),
                      [ClassMethodNames(isa) componentsJoinedByString:@", "]];
    NSLog(@"%@", str);
}

int main(int argc, char * argv[]) {
    @autoreleasepool {

        Foo * anything = [[Foo alloc] init];
        Foo * x = [[Foo alloc] init];
        Foo * y = [[Foo alloc] init];
        Foo * xy = [[Foo alloc] init];
        Foo * control = [[Foo alloc] init];
        
        [x addObserver:anything forKeyPath:@"x" options:0 context:NULL];
        [y addObserver:anything forKeyPath:@"y" options:0 context:NULL];
        
        [xy addObserver:anything forKeyPath:@"x" options:0 context:NULL];
        [xy addObserver:anything forKeyPath:@"y" options:0 context:NULL];
        
        PrintDescription(@"control", control);
        PrintDescription(@"x", x);
        PrintDescription(@"y", y);
        PrintDescription(@"xy", xy);
        
        NSLog(@"\n\tUsing NSObject methods, normal setX: is %p, overridden setX: is %p\n",
              [control methodForSelector:@selector(setX:)],
              [x methodForSelector:@selector(setX:)]);
        
        NSLog(@"\n\tUsing libobjc functions, normal setX: is %p, overridden setX: is %p\n",
              method_getImplementation(class_getInstanceMethod(object_getClass(control),
                                                               @selector(setX:))),
              method_getImplementation(class_getInstanceMethod(object_getClass(x),
                                                               @selector(setX:))));
        
        return 0;
    }
}

```

在这里，我创建了四个对象，x 对象的 x 属性被观察，y 对象的 y 属性被观察，xy 对象的 x 和 y 属性均被观察，参照对象 control 没有属性被观察。在代码的最后部分，分别通过两种方式(对象方法和 runtime 方法)打印出参数对象 control 和被观察对象 x 对象的 setX 方面的实现地址，来对比显示正常情况下 setter 实现以及派生类中重写的 setter 实现。 
 
输出:

```
2016-02-22 18:42:12.072 Callback[1308:963476] 
	control: <Foo: 0x7ff071c09170>
	NSObject class Foo
	libobjc class Foo
	implements methods <z, x, setX:, y, setY:, setZ:>
2016-02-22 18:42:12.074 Callback[1308:963476] 
	x: <Foo: 0x7ff071c09110>
	NSObject class Foo
	libobjc class NSKVONotifying_Foo
	implements methods <setY:, setX:, class, dealloc, _isKVOA>
2016-02-22 18:42:12.074 Callback[1308:963476] 
	y: <Foo: 0x7ff071c09130>
	NSObject class Foo
	libobjc class NSKVONotifying_Foo
	implements methods <setY:, setX:, class, dealloc, _isKVOA>
2016-02-22 18:42:12.074 Callback[1308:963476] 
	xy: <Foo: 0x7ff071c09150>
	NSObject class Foo
	libobjc class NSKVONotifying_Foo
	implements methods <setY:, setX:, class, dealloc, _isKVOA>
2016-02-22 18:42:12.074 Callback[1308:963476] 
	Using NSObject methods, normal setX: is 0x105540e10, overridden setX: is 0x105675b5f
2016-02-22 18:42:12.075 Callback[1308:963476] 
	Using libobjc functions, normal setX: is 0x105540e10, overridden setX: is 0x105675b5f
```
除了最后的setX方法和文章中所述不同（这里两种方法获得的地址是一样的），其他的基本是一致的。

参考：

[Introduction to Key-Value Observing Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177i)  
[Registering for Key-Value Observing](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOBasics.html#//apple_ref/doc/uid/20002252-BAJEAIEE)  
[Registering Dependent Keys](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVODependentKeys.html#//apple_ref/doc/uid/20002179-BAJEAIEE)  
[KVO Compliance](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOCompliance.html#//apple_ref/doc/uid/20002178-BAJEAIEE)  
[Key-Value Observing Implementation Details](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html#//apple_ref/doc/uid/20002307-BAJEAIEE)  
[[深入浅出Cocoa]详解键值观察（KVO）及其实现机理](http://blog.csdn.net/kesalin/article/details/8194240)