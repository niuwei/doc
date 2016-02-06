# runtime

Objective-C语言是一门动态语言，类自身也被实例化为一个对象，属性，变量，方法，协议也是对象。

## 类型定义和数据结构

### 类定义：

```c
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;       // 指向metaClass(元类)
 
#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  // 父类
    const char *name                        OBJC2_UNAVAILABLE;  // 类名
    long version                            OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                               OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
    long instance_size                      OBJC2_UNAVAILABLE;  // 该类的实例变量大小
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  // 该类的成员变量链表
    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  // 方法缓存
    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  // 协议链表
#endif
} OBJC2_UNAVAILABLE;
``` 

### 对象定义

objc_object是表示一个类的实例的结构体，它的定义如下(objc/objc.h)：

```c
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY; // 指向实例对象所属的类
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

> NSObject类的alloc和allocWithZone:方法使用函数class\_createInstance来创建objc_object数据结构。

### 方法缓存定义

objc_cache定义： 

```c
struct objc_cache {
    unsigned int mask /* total = mask + 1 */                 OBJC2_UNAVAILABLE;
    unsigned int occupied                                    OBJC2_UNAVAILABLE;
    Method buckets[1]                                        OBJC2_UNAVAILABLE;
};

```
**mask**：一个整数，指定分配的缓存bucket的总数。在方法查找过程中，Objective-C runtime使用这个字段来确定开始线性查找数组的索引位置。指向方法selector的指针与该字段做一个AND位操作(index = (mask & selector))。这可以作为一个简单的hash散列算法。

**occupied**：一个整数，指定实际占用的缓存bucket的总数。

**buckets**：指向Method数据结构指针的数组。这个数组可能包含不超过mask+1个元素。需要注意的是，指针可能是NULL，表示这个缓存bucket没有被占用，另外被占用的bucket可能是不连续的。这个数组可能会随着时间而增长。

### 元类（Meta Class）

向一个对象发送消息时，runtime会通过对象的isa指针找到所属类，在类的方法列表中查找方法；而向一个类发送消息时，会通过类的isa指针找到它的meta-class，在其方法列表中查找类方法。

每个类都有自己的meta-class（所以称：Class Pair），meta-class也是一个类，也可以向它发送一个消息，meta-class的isa指向基类的meta-class，以此作为它们的所属类。而基类的meta-class的isa指针是指向它自己。

如下图所示：

![Meta Class](http://cn.cocos2d-x.org/uploads/20141031/1414721383966690.png)

## 类与对象操作函数  

### 类操作

```c
// 获取类的类名
const char *class_getName(Class cls);

// 获取类的父类，等于NSObject类的superclass方法
Class class_getSuperclass(Class cls);
 
// 判断给定的Class是否是一个元类
BOOL class_isMetaClass(Class cls);

// 获取实例大小
size_t class_getInstanceSize(Class cls);
```

### 成员变量及属性

> Objective-c中，类的成员变量和属性不是一回事。成员变量是那些带下划线在大括号内的，属性是不带下划线在大括号外用@property标记的。

#### 成员变量

在objc_class中，链表ivars保存成员变量、属性的信息：

```
// 获取类中指定名称实例成员变量的信息
Ivar class_getInstanceVariable(Class cls, const char *name);
 
// 获取类成员变量的信息(OC中没有类变量！)
Ivar class_getClassVariable(Class cls, const char *name);
 
// 添加成员变量
BOOL class_addIvar(Class cls, const char *name, size_t size, uint8_t alignment, const char *types);
 
// 获取整个成员变量列表
Ivar * class_copyIvarList(Class cls, unsigned int *outCount);
```
Objective-C不支持往已存在的类中添加实例变量，但可以通过运行时创建一个类，使用class\_addIvar函数了添加实例变量。需要注意的是，这个方法只能在objc\_allocateClassPair函数与objc_registerClassPair之间调用。另外，这个类也不能是元类。成员变量的按字节最小对齐量是1<

class\_copyIvarList函数，它返回一个指向成员变量信息的数组，数组中每个元素是指向该成员变量信息的objc\_ivar结构体的指针。这个数组不包含在父类中声明的变量。outCount指针返回数组的大小。必须使用free()来释放这个数组。

#### 属性

```c
// 获取指定的属性
objc_property_t class_getProperty(Class cls, const char *name);
 
// 获取属性列表
objc_property_t * class_copyPropertyList(Class cls, unsigned int *outCount);
 
// 为类添加属性
BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount);
 
// 替换类的属性
void class_replaceProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount);
```

#### 方法

```c
// 添加方法
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
 
// 获取实例方法
Method class_getInstanceMethod(Class cls, SEL name);
 
// 获取类方法
Method class_getClassMethod(Class cls, SEL name);
 
// 获取所有方法的数组
Method * class_copyMethodList(Class cls, unsigned int *outCount);
 
// 替代方法的实现
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types);
 
// 返回方法的具体实现
IMP class_getMethodImplementation(Class cls, SEL name);
IMP class_getMethodImplementation_stret(Class cls, SEL name);
 
// 类实例是否响应指定的selector
BOOL class_respondsToSelector(Class cls, SEL sel);
```

class\_addMethod的实现会覆盖父类的方法实现，但不会取代本类中已存在的实现，如果本类中包含一个同名的实现，则函数会返回NO。如果要修改已存在实现，可以使用method\_setImplementation。一个Objective-C方法是一个简单的C函数，它至少包含两个参数：self和_cmd。所以，我们的实现函数(IMP参数指向的函数)至少需要两个参数，如下所示：

```c
void myMethodIMP(id self, SEL _cmd)
{
    // implementation ....
}
```

可以为类动态添加方法，不管这个类是否已存在。

class_getInstanceMethod、class_getClassMethod函数，与class_copyMethodList不同的是，这两个函数都会去搜索父类的实现。

class_copyMethodList函数，返回包含所有实例方法的数组，如果需要获取类方法，则可以使用class_copyMethodList(object_getClass(cls), &count)(一个类的实例方法是定义在元类里面)。该列表不包含父类实现的方法。outCount参数返回方法的个数。在获取到列表后，我们需要使用free()方法来释放它。

class_replaceMethod函数，该函数的行为可以分为两种：如果类中不存在name指定的方法，则类似于class_addMethod函数一样会添加方法；如果类中已存在name指定的方法，则类似于method_setImplementation一样替代原方法的实现。

class_getMethodImplementation函数，该函数在向类实例发送消息时会被调用，并返回一个指向方法实现函数的指针。这个函数会比method_getImplementation(class_getInstanceMethod(cls, name))更快。返回的函数指针可能是一个指向runtime内部的函数，而不一定是方法的实际实现。例如，如果类实例无法响应selector，则返回的函数指针将是运行时消息转发机制的一部分。

class_respondsToSelector函数，我们通常使用NSObject类的respondsToSelector:或instancesRespondToSelector:方法来达到相同目的。

#### 版本

```c
// 获取版本号
int class_getVersion(Class cls );
 
// 设置版本号
void class_setVersion(Class cls, int version);
```

#### 其他

runtime还提供了两个函数来供CoreFoundation的tool-free bridging使用，即：  

```c
Class objc_getFutureClass(const char *name);
void objc_setFutureClass(Class cls, const char *name);
```
通常我们不直接使用这两个函数。

#### 动态创建

**动态创建类**

```c
// 创建一个新类和元类
Class objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes);
 
// 销毁一个类及其相关联的类
void objc_disposeClassPair(Class cls);
 
// 在应用中注册由objc_allocateClassPair创建的类
void objc_registerClassPair(Class cls);
```

objc_allocateClassPair函数：如果我们要创建一个根类，则superclass指定为nil。extraBytes通常指定为0，该参数是分配给类和元类对象尾部的索引ivars的字节数。

为了创建一个新类，我们需要调用objc_allocateClassPair。然后使用诸如class_addMethod，class_addIvar等函数来为新创建的类添加方法、实例变量和属性等。完成这些后，我们需要调用objc_registerClassPair函数来注册类，之后这个新类就可以在程序中使用了。

实例方法和实例变量应该添加到类自身上，而类方法应该添加到类的元类上。

objc_disposeClassPair函数用于销毁一个类，不过需要注意的是，如果程序运行中还存在类或其子类的实例，则不能调用针对类调用该方法。

**动态创建对象**

```c
// 创建类实例
id class_createInstance(Class cls, size_t extraBytes);
 
// 在指定位置创建类实例
id objc_constructInstance(Class cls, void *bytes);
 
// 销毁类实例，但不会释放并移除任何与其相关的引用
void * objc_destructInstance(id obj);
```

class_createInstance函数：创建实例时，会在默认的内存区域为类分配内存。extraBytes参数表示分配的额外字节数。这些额外的字节可用于存储在类定义中所定义的实例变量之外的实例变量。该函数在ARC环境下无法使用。

调用class_createInstance的效果与+alloc方法类似。不过在使用class_createInstance时，我们需要确切的知道我们要用它来做什么。在下面的例子中，我们用NSString来测试一下该函数的实际效果：使用class_createInstance函数获取的是NSString实例，而不是类簇中的默认占位符类__NSCFConstantString。

#### 对象操作

**操作对象**

```c
// 返回指定对象的一份拷贝
id object_copy(id obj, size_t size);
 
// 释放指定对象占用的内存
id object_dispose(id obj);
```

**操作对象的变量**

```c
// 修改类实例的实例变量的值
Ivar object_setInstanceVariable(id obj, const char *name, void *value);
 
// 获取对象实例变量的值
Ivar object_getInstanceVariable(id obj, const char *name, void **outValue);
 
// 返回指向给定对象分配的任何额外字节的指针
void * object_getIndexedIvars(id obj);
 
// 返回对象中实例变量的值
id object_getIvar(id obj, Ivar ivar);
 
// 设置对象中实例变量的值
void object_setIvar(id obj, Ivar ivar, id value);
```

如果实例变量的Ivar已经知道，那么调用object\_getIvar会比object\_getInstanceVariable函数快，相同情况下，object\_setIvar也比object_setInstanceVariable快。

**操作对象的类**

```c
// 返回给定对象的类名
const char * object_getClassName(id obj);
 
// 返回对象的类
Class object_getClass(id obj);
 
// 设置对象的类
Class object_setClass(id obj, Class cls);
```

**获取类定义**

```c
// 获取已注册的类定义的列表
int objc_getClassList(Class *buffer, int bufferCount);
 
// 创建并返回一个指向所有已注册类的指针列表
Class * objc_copyClassList(unsigned int *outCount);
 
// 返回指定类的类定义
Class objc_lookUpClass(const char *name);
Class objc_getClass(const char *name);
Class objc_getRequiredClass(const char *name);
 
// 返回指定类的元类
Class objc_getMetaClass(const char *name);
```

objc_getClassList函数：获取已注册的类定义的列表。我们不能假设从该函数中获取的类对象是继承自NSObject体系的，所以在这些类上调用方法是，都应该先检测一下这个方法是否在这个类中实现。

如果类在运行时未注册，则objc_lookUpClass会返回nil，而objc_getClass会调用类处理回调，并再次确认类是否注册，如果确认未注册，再返回nil。而objc_getRequiredClass函数的操作与objc_getClass相同，只不过如果没有找到类，则会杀死进程。

objc_getMetaClass函数：如果指定的类没有注册，则该函数会调用类处理回调，并再次确认类是否注册，如果确认未注册，再返回nil。不过，每个类定义都必须有一个有效的元类定义，所以这个函数总是会返回一个元类定义，不管它是否有效。

#### 实例

```c
//-----------------------------------------------------------
// MyClass.h

@interface MyClass : NSObject <NSCopying>

@property (nonatomic, strong) NSArray *array;
@property (nonatomic, copy) NSString *string;

- (void)method1;
- (void)method2;
+ (void)classMethod1;

@end

//-----------------------------------------------------------
// MyClass.m
#import "MyClass.h"

@interface MyClass () {
    
    NSInteger _instance1;
    NSString *_instance2;
}

@property (nonatomic, assign) NSUInteger integer;

- (void)method3WithArg1:(NSInteger)arg1 arg2:(NSString *)arg2;

@end

@implementation MyClass

+ (void)classMethod1 {
    
}

- (void)method1 {
    NSLog(@"call method method1");
}

- (void)method2 {
    
}

- (void)method3WithArg1:(NSInteger)arg1 arg2:(NSString *)arg2 {
    
    NSLog(@"arg1 : %ld, arg2 : %@", arg1, arg2);
}

@end

//-----------------------------------------------------------
// main.h
#import <objc/runtime.h>
#import "MyClass.h"

void imp_submethod1(id self, SEL _cmd) {
    NSLog(@"call imp_submethod1.");
}

int main(int argc, char * argv[]) {
    @autoreleasepool {

        MyClass *myClass = [[MyClass alloc] init];
        Class cls = myClass.class;
        
        // 获得类名
        NSLog(@"class name: %s", class_getName(cls));
        
        // 获得父类类名
        NSLog(@"super class name: %s", class_getName(class_getSuperclass(cls)));
        
        // 是否是元类
        NSLog(@"MyClass is %@ a meta-class", (class_isMetaClass(cls) ? @"" : @"not"));
        
        Class meta_class = objc_getMetaClass(class_getName(cls));
        NSLog(@"%s's meta-class is %s", class_getName(cls), class_getName(meta_class));
        
        // 变量实例大小
        NSLog(@"instance size: %zu", class_getInstanceSize(cls));
        NSLog(@"==========================================================");
        
        // 成员变量
        // 打印MyClass的所有成员变量
        unsigned int outCount = 0;
        Ivar *ivars = class_copyIvarList(cls, &outCount);
        for (int i = 0; i < outCount; i++) {
            Ivar ivar = ivars[i];
            NSLog(@"instance variable's name: %s at index: %d", ivar_getName(ivar), i);
        }
        free(ivars);
        
        // 获得MyClass的成员变量_string
        Ivar string = class_getInstanceVariable(cls, "_string");
        if (string != NULL) {
            NSLog(@"instace variable %s", ivar_getName(string));
        }
        NSLog(@"==========================================================");
        
        // 属性操作
        // 打印MyClass的所有属性
        objc_property_t * properties = class_copyPropertyList(cls, &outCount);
        for (int i = 0; i < outCount; i++) {
            objc_property_t property = properties[i];
            NSLog(@"property's name: %s", property_getName(property));
        }
        free(properties);
        
        // 获得MyClass的属性array
        objc_property_t array = class_getProperty(cls, "array");
        if (array != NULL) {
            NSLog(@"property %s", property_getName(array));
        }
        NSLog(@"==========================================================");
        
        // 方法操作
        // 打印MyClass的所有方法
        Method *methods = class_copyMethodList(cls, &outCount);
        for (int i = 0; i < outCount; i++) {
            Method method = methods[i];
            NSLog(@"method's signature: %s", method_getName(method));
        }
        free(methods);
        
        // 获得MyClass的实例方法method1
        Method method1 = class_getInstanceMethod(cls, @selector(method1));
        if (method1 != NULL) {
            NSLog(@"method %s", method_getName(method1));
        }
        
        // 获得MyClass的类方法classMethod1
        Method classMethod = class_getClassMethod(cls, @selector(classMethod1));
        if (classMethod != NULL) {
            NSLog(@"class method : %s", method_getName(classMethod));
        }
        
        // 判断方法是否属于类
        NSLog(@"MyClass is%@ responsd to selector: method3WithArg1:arg2:", class_respondsToSelector(cls, @selector(method3WithArg1:arg2:)) ? @"" : @" not");
        
        // 获得方法的实现，并执行
        IMP imp = class_getMethodImplementation(cls, @selector(method1));
        imp();
        NSLog(@"==========================================================");
        
        // 协议
        Protocol * __unsafe_unretained * protocols = class_copyProtocolList(cls, &outCount);
        Protocol * protocol;
        for (int i = 0; i < outCount; i++) {
            protocol = protocols[i];
            NSLog(@"protocol name: %s", protocol_getName(protocol));
        }
        
        // 判断类是否遵循某协议
        NSLog(@"MyClass is%@ responsed to protocol %s", class_conformsToProtocol(cls, protocol) ? @"" : @" not", protocol_getName(protocol));
        NSLog(@"==========================================================");
        
        // 以MyClass为基类创建一个新类
        Class newSubClass = objc_allocateClassPair(MyClass.class, "MySubClass", 0);
        // 为新类添加方法submethod1，并添加实现
        class_addMethod(newSubClass, @selector(submethod1), (IMP)imp_submethod1, "v@:");
        // 在新类中替换街垒方法method1的实现
        class_replaceMethod(newSubClass, @selector(method1), (IMP)imp_submethod1, "v@:");
        // 为新类添加NSString*类型成员变量i
        class_addIvar(newSubClass, "_ivar1", sizeof(NSString *), log(sizeof(NSString *)), "i");
        
        // 为新类添加NString*类型属性property2
        objc_property_attribute_t type = {"T", "@\"NSString\""};
        objc_property_attribute_t ownership = { "C", "" };
        objc_property_attribute_t backingivar = { "V", "_ivar1"};
        objc_property_attribute_t attrs[] = {type, ownership, backingivar};
        
        class_addProperty(newSubClass, "property2", attrs, 3);
        // 注册新类，完成创建
        objc_registerClassPair(newSubClass);
        
        id instance = [[newSubClass alloc] init];
        [instance performSelector:@selector(submethod1)];
        [instance performSelector:@selector(method1)];
        NSLog(@"==========================================================");
        
        // 给已存在的类添加方法
        class_addMethod(cls, @selector(submethod1), (IMP)imp_submethod1, "v@:");
        [myClass submethod1];
        NSLog(@"==========================================================");
        
        // 销毁类之后，再调用类的方法或者创建新的实例都会崩溃，注释掉了
        objc_disposeClassPair(newSubClass);
//        [instance performSelector:@selector(submethod1)];
//        id instance2 = [[newSubClass alloc] init];
        NSLog(@"==========================================================");
        
        // 使用class_createInstance创建对象，需要关闭ARC
        id theObject = class_createInstance(NSString.class, sizeof(unsigned));
        id str1 = [theObject init];
        NSLog(@"%@", [str1 class]);
        
        id str2 = [[NSString alloc] initWithString:@"test"];
        NSLog(@"%@", [str2 class]);
        NSLog(@"==========================================================");
        
        int numClasses;
        Class * classes = NULL;
        
        // 会返回3000多个类，就不打印结果了
        numClasses = objc_getClassList(NULL, 0);
        if (numClasses > 0) {
            classes = malloc(sizeof(Class) * numClasses);
            numClasses = objc_getClassList(classes, numClasses);
            
            NSLog(@"number of classes: %d", numClasses);
            
            for (int i = 0; i < numClasses; i++) {
                
                Class cls = classes[i];
                NSLog(@"class name: %s", class_getName(cls));
            }
            
            free(classes);
        }
 
        return 0;
    }
}
```
输出：

```
class name: MyClass
==========================================================
super class name: NSObject
==========================================================
MyClass is not a meta-class
==========================================================
MyClass's meta-class is MyClass
==========================================================
instance size: 48
==========================================================
instance variable's name: _instance1 at index: 0
instance variable's name: _instance2 at index: 1
instance variable's name: _array at index: 2
instance variable's name: _string at index: 3
instance variable's name: _integer at index: 4
instace variable _string
==========================================================
property's name: array
property's name: string
property's name: integer
property array
==========================================================
method's signature: method1
method's signature: method2
method's signature: method3WithArg1:arg2:
method's signature: integer
method's signature: setInteger:
method's signature: setArray:
method's signature: .cxx_destruct
method's signature: string
method's signature: setString:
method's signature: array
method method1
class method : classMethod1
MyClass is responsd to selector: method3WithArg1:arg2:
call method method1
==========================================================
protocol name: NSCopying
MyClass is responsed to protocol NSCopying
==========================================================
call imp_submethod1.
call imp_submethod1.
==========================================================
NSString
__NSCFConstantString

```

## 类型编码(Type Encoding)

作为对Runtime的补充，编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的selector关联在一起。当给定一个类型时，@encode返回这个类型的字符串编码。这些类型可以是诸如int、指针这样的基本类型，也可以是结构体、类等类型。事实上，任何可以作为sizeof()操作参数的类型都可以用于@encode()。

[Type Encoding]:https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1

在Objective-C Runtime Programming Guide中的[Type Encoding][Type Encoding]一节中，列出了Objective-C中所有的类型编码。需要注意的是这些类型很多是与我们用于存档和分发的编码类型是相同的。但有一些不能在存档时使用。

注：Objective-C不支持long double类型。@encode(long double)返回d，与double是一样的。

另外，还有些编码类型，@encode虽然不会直接返回它们，但它们可以作为协议中声明的方法的类型限定符。可以参考Type Encoding。

对于属性而言，还会有一些特殊的类型编码，以表明属性是只读、拷贝、retain等等，详情可以参考Property Type String。

## 成员变量和属性

### 定义

**Ivar**

表示实例变量的类型:

```c
typedef struct objc_ivar *Ivar; 

struct objc_ivar {
    char *ivar_name          OBJC2_UNAVAILABLE; // 变量名
    char *ivar_type          OBJC2_UNAVAILABLE; // 变量类型
    int ivar_offset          OBJC2_UNAVAILABLE; // 基地址偏移字节
#ifdef __LP64__
    int space                OBJC2_UNAVAILABLE;
#endif
}    
```
**objc\_property_t**

表示Objective-C声明的属性的类型:

```c
typedef struct objc_property *objc_property_t;
```

**objc\_property\_attribute_t**

定义了属性的特性

```c
typedef struct {
    const char *name;           // 特性名
    const char *value;          // 特性值
} objc_property_attribute_t;
```

### 操作方法

**成员变量**

```c
// 获取成员变量名
const char * ivar_getName(Ivar v); 
 
// 获取成员变量类型编码 
const char * ivar_getTypeEncoding(Ivar v); 
 
// 获取成员变量的偏移量 
ptrdiff_t ivar_getOffset(Ivar v);
```
ivar\_getOffset函数，对于类型id或其它对象类型的实例变量，可以调用object\_getIvar和object_setIvar来直接访问成员变量，而不使用偏移量。

**属性**

```c
// 获取属性名
const char * property_getName(objc_property_t property); 
 
// 获取属性特性描述字符串 
const char * property_getAttributes(objc_property_t property); 
 
// 获取属性中指定的特性,使用完后需要调用free()释放
char * property_copyAttributeValue(objc_property_t property, const char *attributeName); 
 
// 获取属性的特性列表,使用完后需要调用free()释放
objc_property_attribute_t * property_copyAttributeList(objc_property_t property, unsigned int *outCount); 
```

## 关联对象(Associated Object)

### 目的

利用分类（Catagory）可以给已存在的类添加方法，但无法直接添加实例变量。利用关联可以给已存在的类添加成员变量。

我们可以把关联对象想象成一个Objective-C对象(如字典)，这个对象通过给定的key连接到类的一个实例上。不过由于使用的是C接口，所以key是一个void指针(const void *)。我们还需要指定一个内存管理策略，以告诉Runtime如何管理这个对象的内存。在宿主对象释放后对关联对象是否释放的处理。由以下值指定：

```
OBJC_ASSOCIATION_ASSIGN 
OBJC_ASSOCIATION_RETAIN_NONATOMIC 
OBJC_ASSOCIATION_COPY_NONATOMIC 
OBJC_ASSOCIATION_RETAIN 
OBJC_ASSOCIATION_COPY 
```

### 关联的操作函数（引入 objc/runtime.h）：

```c
// 设置关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
 
// 获取关联对象 
id objc_getAssociatedObject(id object, const void *key);
 
// 移除关联对象 
void objc_removeAssociatedObjects(id object);
```

**objc_setAssociatedObject**：需要四个参数：源对象，关键字，关联的对象和一个关联策略。  
关键字：是一个void类型的指针。每一个关联的关键字必须是唯一的。通常都是会采用静态变量（的地址）来作为关键字（以确保每个关键字是唯一的）。  

将一个对象连接到其它对象所需要做的就是下面代码：

```c
static char myKey; 
objc_setAssociatedObject(self, &myKey, anObject, OBJC_ASSOCIATION_RETAIN);

id anObject = objc_getAssociatedObject(self, &myKey);
```
可以使用objc\_removeAssociatedObjects函数来移除一个关联对象，或者使用objc_setAssociatedObject函数将key指定的关联对象设置为nil。

### 演示

假定我们想要动态地将一个Tap手势操作连接到任何UIView中，并且根据需要指定点击后的实际操作。这时候我们就可以将一个手势对象及操作的block对象关联到我们的UIView对象中。这项任务分两部分。首先，如果需要，我们要创建一个手势识别对象并将它及block做为关联对象。如下代码所示：

```c
- (void)setTapActionWithBlock:(void (^)(void))block 
{ 
    UITapGestureRecognizer *gesture = objc_getAssociatedObject(self, &kDTActionHandlerTapGestureKey); 
 
    if (!gesture) 
    { 
        gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(__handleActionForTapGesture:)]; 
        [self addGestureRecognizer:gesture]; 
        objc_setAssociatedObject(self, &kDTActionHandlerTapGestureKey, gesture, OBJC_ASSOCIATION_RETAIN); 
    } 
 
    objc_setAssociatedObject(self, &kDTActionHandlerTapBlockKey, block, OBJC_ASSOCIATION_COPY); 
} 
```

这段代码检测了手势识别的关联对象。如果没有，则创建并建立关联关系。同时，将传入的块对象连接到指定的key上。注意block对象的关联内存管理策略。

手势识别对象需要一个target和action，所以接下来我们定义处理方法：

```c
- (void)__handleActionForTapGesture:(UITapGestureRecognizer *)gesture 
{ 
    if (gesture.state == UIGestureRecognizerStateRecognized) 
    { 
        void(^action)(void) = objc_getAssociatedObject(self, &kDTActionHandlerTapBlockKey);
 
        if (action) 
        { 
            action(); 
        } 
    } 
} 
```
我们需要检测手势识别对象的状态，因为我们只需要在点击手势被识别出来时才执行操作。

## 方法和消息处理

### 数据类型

**SEL**

选择器，是表示一个方法的selector的指针，其定义如下：

```c
typedef struct objc_selector *SEL;
```
objc_selector结构体的详细定义没有在头文件中找到。方法的selector用于表示运行时方法的名字。Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)，这个标识就是SEL。如下代码所示：

```c
SEL sel1 = @selector(method1);
NSLog(@"sel : %p", sel1);
// 输出为：2014-10-30 18:40:07.518 RuntimeTest[52734:466626] sel : 0x100002d72
```

上面的代码中，@select()参数只与方法名相关，由此SEL也只与方法名称相关。即使不同类的方法，只要方法名相同，那么方法的SEL就是一样的。不同类的实例对象执行相同的selector时，会在各自的方法列表中去根据selector去寻找自己对应的IMP。

本质上，SEL只是一个指向方法的指针（准确的说，只是一个根据方法名hash化了的KEY值，能唯一代表一个方法），它的存在只是为了加快方法的查询速度。

获取SEL的方法：  

1. sel_registerName函数
2. Objective-C编译器提供的@selector()
3. NSSelectorFromString()方法

**IMP**

实际上是一个函数指针。其定义如下：

```c
id (*IMP)(id, SEL, ...)
```
这个函数使用当前CPU架构实现的标准的C调用约定。第一个参数是指向self的指针(如果是实例方法，则是类实例的内存地址；如果是类方法，则是指向元类的指针)，第二个参数是方法选择器(selector)，接下来是方法的实际参数列表。

前面介绍过的SEL就是为了查找方法的最终实现IMP的。由于每个方法对应唯一的SEL，因此我们可以通过SEL方便快速准确地获得它所对应的IMP，我们就可以像调用普通的C语言函数一样来使用这个函数指针了。

**Method**

用于表示类定义中的方法，则定义如下：

```c
typedef struct objc_method *Method;
 
struct objc_method {
    SEL method_name                 OBJC2_UNAVAILABLE;  // 方法名
    char *method_types              OBJC2_UNAVAILABLE;
    IMP method_imp                  OBJC2_UNAVAILABLE;  // 方法实现
}
```

**objc_method_description**

定义了一个Objective-C方法，其定义如下：

```c
struct objc_method_description{ 
	SEL name; 
	char *types; 
};
```

### 操作

**方法操作**

```c
// 调用指定方法的实现
id method_invoke(id receiver, Method m, ...);
 
// 调用返回一个数据结构的方法的实现
void method_invoke_stret(id receiver, Method m, ...);
 
// 获取方法名
SEL method_getName(Method m);
 
// 返回方法的实现
IMP method_getImplementation(Method m);
 
// 获取描述方法参数和返回值类型的字符串
const char * method_getTypeEncoding(Method m);
 
// 获取方法的返回值类型的字符串
char * method_copyReturnType(Method m);
 
// 获取方法的指定位置参数的类型字符串
char * method_copyArgumentType(Method m, unsigned int index);
 
// 通过引用返回方法的返回值类型字符串
void method_getReturnType(Method m, char *dst, size_t dst_len);
 
// 返回方法的参数的个数
unsigned int method_getNumberOfArguments(Method m);
 
// 通过引用返回方法指定位置参数的类型字符串
void method_getArgumentType(Method m, unsigned int index, char *dst, size_t dst_len);
 
// 返回指定方法的方法描述结构体
struct objc_method_description * method_getDescription(Method m);
 
// 设置方法的实现
IMP method_setImplementation(Method m, IMP imp);
 
// 交换两个方法的实现
void method_exchangeImplementations(Method m1, Method m2);
```

**选择器操作**

```c
// 返回给定选择器指定的方法的名称
const char * sel_getName(SEL sel);
 
// 在Objective-C Runtime系统中注册一个方法，将方法名映射到一个选择器，并返回这个选择器
SEL sel_registerName(const char *str);
 
// 在Objective-C Runtime系统中注册一个方法
SEL sel_getUid(const char *str);
 
// 比较两个选择器
BOOL sel_isEqual(SEL lhs, SEL rhs);
```

### 方法调用流程

在Objective-C中，消息直到运行时才绑定到方法实现上。编译器会将消息表达式[receiver message]转化为一个消息函数的调用，即objc_msgSend。这个函数将消息接收者和方法名作为其基础参数，如以下所示：

```c
objc_msgSend(receiver, selector, arg1, arg2, ...)
```
这个函数完成了动态绑定的所有事情：

1. 首先它找到selector对应的方法实现。因为同一个方法可能在不同的类中有不同的实现，所以我们需要依赖于接收者的类来找到的确切的实现。
2. 它调用方法实现，并将接收者对象及方法的所有参数传给它。
3. 最后，它将实现返回的值作为它自己的返回值。

下图演示了这样一个消息的基本框架：

![Message Send](http://cn.cocos2d-x.org/uploads/20141106/1415239732731658.gif)

当消息发送给一个对象时，objc_msgSend通过对象的isa指针获取到类的结构体，然后在方法分发表里面查找方法的selector。如果没有找到selector，则通过类结构体中的指向父类的指针找到其父类，并在父类的分发表里面查找方法的selector。依此，会一直沿着类的继承体系到达NSObject类。一旦定位到selector，函数会就获取到了实现的入口点，并传入相应的参数来执行方法的具体实现，并缓存方法地址。如果最后没有定位到selector，则会走消息转发流程。

**获取方法地址**

NSObject类提供了methodForSelector:方法，让我们可以获取到方法的指针，然后通过这个指针来调用实现代码。我们需要将methodForSelector:返回的指针转换为合适的函数类型，函数参数和返回值都需要匹配上。

我们通过以下代码来看看methodForSelector:的使用：

```c
void (*setter)(id, SEL, BOOL);
 
setter = (void (*)(id, SEL, BOOL))[target methodForSelector:@selector(setFilled:)];
for (int i = 0 ; i < 1000 ; i++)
    setter(targetList[i], @selector(setFilled:), YES);
```
这里需要注意的就是函数指针的前两个参数必须是id和SEL。

当然这种方式只适合于在类似于for循环这种情况下频繁调用同一方法，以提高性能的情况。另外，methodForSelector:是由Cocoa运行时提供的；它不是Objective-C语言的特性。

### 消息转发

默认情况下，如果是以[object message]的方式调用方法，如果object无法响应message消息时，编译器会报错。但如果是以perform…的形式来调用，则需要等到运行时才能确定object是否能接收message消息。如果不能，则程序崩溃。

通常，当我们不能确定一个对象是否能接收某个消息时，会先调用respondsToSelector:来判断一下。如下代码所示：

```c
if ([self respondsToSelector:@selector(method)]) {
    [self performSelector:@selector(method)];
}
```
当一个对象无法接收某一消息时，就会启动所谓”消息转发(message forwarding)“机制，通过这一机制，我们可以告诉对象如何处理未知的消息。默认情况下，对象接收到未知的消息，会导致程序崩溃，通过控制台，我们可以看到以下异常信息：

```c
-[SUTRuntimeMethod method]: unrecognized selector sent to instance 0x100111940
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[SUTRuntimeMethod method]: unrecognized selector sent to instance 0x100111940'
```
这段异常信息实际上是由NSObject的”doesNotRecognizeSelector”方法抛出的。不过，我们可以采取一些措施，让我们的程序执行特定的逻辑，而避免程序的崩溃。

消息转发机制基本上分为三个步骤：

1. 动态方法解析
2. 备用接收者
3. 完整转发

**动态方法解析**

对象在接收到未知的消息时，首先会调用所属类的类方法+resolveInstanceMethod:(实例方法)或者+resolveClassMethod:(类方法)。在这个方法中，我们有机会为该未知消息新增一个”处理方法”。不过使用该方法的前提是我们已经实现了该”处理方法”，只需要在运行时通过class_addMethod函数动态添加到类里面就可以了。如下代码所示：

```c
void functionForMethod1(id self, SEL _cmd) {
   NSLog(@"%@, %p", self, _cmd);
}
 
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    // 只有调用未知方法才会调用
    // 也可以通过[self method];
    NSString *selectorString = NSStringFromSelector(sel);
 
    if ([selectorString isEqualToString:@"method1"]) {
        class_addMethod(self.class, @selector(method1), (IMP)functionForMethod1, "@:");
    }
 
    return [super resolveInstanceMethod:sel];
}
```
不过这种方案更多的是为了实现@dynamic属性。

**备用方法**

如果在上一步无法处理消息，则Runtime会继续调以下方法：

```c
- (id)forwardingTargetForSelector:(SEL)aSelector
```
如果一个对象实现了这个方法，并返回一个非nil的结果，则这个对象会作为消息的新接收者，且消息会被分发到这个对象。当然这个对象不能是self自身，否则就是出现无限循环。当然，如果我们没有指定相应的对象来处理aSelector，则应该调用父类的实现来返回结果。

使用这个方法通常是在对象内部，可能还有一系列其它对象能处理该消息，我们便可借这些对象来处理消息并返回，这样在对象外部看来，还是由该对象亲自处理了这一消息。如下代码所示：

```c
@interface SUTRuntimeMethodHelper : NSObject
- (void)method2;
@end
 
@implementation SUTRuntimeMethodHelper
- (void)method2 {
    NSLog(@"%@, %p", self, _cmd);
}
@end
 
#pragma mark -
 
@interface SUTRuntimeMethod () {
    SUTRuntimeMethodHelper *_helper;
}
 
@end
 
@implementation SUTRuntimeMethod
 
+ (instancetype)object {
    return [[self alloc] init];
}
 
- (instancetype)init {
    self = [super init];
    if (self != nil) {
        _helper = [[SUTRuntimeMethodHelper alloc] init];
    }
 
    return self;
}
 
- (void)test {
    [self performSelector:@selector(method2)];
}
 
- (id)forwardingTargetForSelector:(SEL)aSelector {
 
    NSLog(@"forwardingTargetForSelector");
 
    NSString *selectorString = NSStringFromSelector(aSelector);
 
    // 将消息转发给_helper来处理
    if ([selectorString isEqualToString:@"method2"]) {
        return _helper;
    }
 
    return [super forwardingTargetForSelector:aSelector];
}
 
@end
```
这一步合适于我们只想将消息转发到另一个能处理该消息的对象上。但这一步无法对消息进行处理，如操作消息的参数和返回值。

**完整消息转发**

如果在上一步还不能处理未知消息，则唯一能做的就是启用完整的消息转发机制了。此时会调用以下方法：

```c
- (void)forwardInvocation:(NSInvocation *)anInvocation
```
运行时系统会在这一步给消息接收者最后一次机会将消息转发给其它对象。对象会创建一个表示消息的NSInvocation对象，把与尚未处理的消息有关的全部细节都封装在anInvocation中，包括selector，目标(target)和参数。我们可以在forwardInvocation方法中选择将消息转发给其它对象。

forwardInvocation:方法的实现有两个任务：

1. 定位可以响应封装在anInvocation中的消息的对象。这个对象不需要能处理所有未知消息。
2. 使用anInvocation作为参数，将消息发送到选中的对象。anInvocation将会保留调用结果，运行时系统会提取这一结果并将其发送到消息的原始发送者。

不过，在这个方法中我们可以实现一些更复杂的功能，我们可以对消息的内容进行修改，比如追回一个参数等，然后再去触发消息。另外，若发现某个消息不应由本类处理，则应调用父类的同名方法，以便继承体系中的每个类都有机会处理此调用请求。

还有一个很重要的问题，我们必须重写以下方法：

```c
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
```
消息转发机制使用从这个方法中获取的信息来创建NSInvocation对象。因此我们必须重写这个方法，为给定的selector提供一个合适的方法签名。

完整的示例如下所示：

```c
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
 
    if (!signature) {
        if ([SUTRuntimeMethodHelper instancesRespondToSelector:aSelector]) {
            signature = [SUTRuntimeMethodHelper instanceMethodSignatureForSelector:aSelector];
        }
    }
 
    return signature;
}
 
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if ([SUTRuntimeMethodHelper instancesRespondToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:_helper];
    }
}
```
NSObject的forwardInvocation:方法实现只是简单调用了doesNotRecognizeSelector:方法，它不会转发任何消息。这样，如果不在以上所述的三个步骤中处理未知消息，则会引发一个异常。

从某种意义上来讲，forwardInvocation:就像一个未知消息的分发中心，将这些未知的消息转发给其它对象。或者也可以像一个运输站一样将所有未知消息都发送给同一个接收对象。这取决于具体的实现。

**消息转发与多重继承**

回过头来看第二和第三步，通过这两个方法我们可以允许一个对象与其它对象建立关系，以处理某些未知消息，而表面上看仍然是该对象在处理消息。通过这种关系，我们可以模拟“多重继承”的某些特性，让对象可以“继承”其它对象的特性来处理一些事情。不过，这两者间有一个重要的区别：多重继承将不同的功能集成到一个对象中，它会让对象变得过大，涉及的东西过多；而消息转发将功能分解到独立的小的对象中，并通过某种方式将这些对象连接起来，并做相应的消息转发。

不过消息转发虽然类似于继承，但NSObject的一些方法还是能区分两者。如respondsToSelector:和isKindOfClass:只能用于继承体系，而不能用于转发链。便如果我们想让这种消息转发看起来像是继承，则可以重写这些方法，如以下代码所示：

```c
- (BOOL)respondsToSelector:(SEL)aSelector   {
       if ( [super respondsToSelector:aSelector] )
                return YES;     
       else {
                 /* Here, test whether the aSelector message can
                  *            
                  * be forwarded to another object and whether that  
                  *            
                  * object can respond to it. Return YES if it can.  
                  */      
       }
       return NO;  
}
```

![message diapatch](http://images.cnitblog.com/i/31852/201404/231837047638961.png)

## Method Swizzling

Method Swizzling是改变一个selector的实际实现的技术。通过这一技术，我们可以在运行时通过修改类的分发表中selector对应的函数，来修改方法的实现。

例如，我们想跟踪在程序中每一个view controller展示给用户的次数：当然，我们可以在每个view controller的viewDidAppear中添加跟踪代码；但是这太过麻烦，需要在每个view controller中写重复的代码。创建一个子类可能是一种实现方式，但需要同时创建UIViewController, UITableViewController, UINavigationController及其它UIKit中view controller的子类，这同样会产生许多重复的代码。

这种情况下，我们就可以使用Method Swizzling，如在代码所示：

```objc
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];         
        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // 交换viewWillAppear与xxx_viewWillAppear的实现
        BOOL didAddMethod =
                class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
            swizzledSelector,
            method_getImplementation(originalMethod),
            method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```
在这里，我们通过method swizzling修改了UIViewController的@selector(viewWillAppear:)对应的函数指针，使其实现指向了我们自定义的xxx_viewWillAppear的实现。这样，当UIViewController及其子类的对象调用viewWillAppear时，都会打印一条日志信息。

上面的例子很好地展示了使用method swizzling来一个类中注入一些我们新的操作。当然，还有许多场景可以使用method swizzling，在此不多举例。在此我们说说使用method swizzling需要注意的一些问题：

### Swizzling应该总是在+load中执行

在Objective-C中，运行时会自动调用每个类的两个方法。+load会在类初始加载时调用，+initialize会在第一次调用类的类方法或实例方法之前被调用。这两个方法是可选的，且只有在实现了它们时才会被调用。由于method swizzling会影响到类的全局状态，因此要尽量避免在并发处理中出现竞争的情况。+load能保证在类的初始化过程中被加载，并保证这种改变应用级别的行为的一致性。相比之下，+initialize在其执行时不提供这种保证—事实上，如果在应用中没为给这个类发送消息，则它可能永远不会被调用。

### Swizzling应该总是在dispatch_once中执行

与上面相同，因为swizzling会改变全局状态，所以我们需要在运行时采取一些预防措施。原子性就是这样一种措施，它确保代码只被执行一次，不管有多少个线程。GCD的dispatch_once可以确保这种行为，我们应该将其作为method swizzling的最佳实践。

### 选择器、方法与实现

在Objective-C中，选择器(selector)、方法(method)和实现(implementation)是运行时中一个特殊点，虽然在一般情况下，这些术语更多的是用在消息发送的过程描述中。

以下是Objective-C Runtime Reference中的对这几个术语一些描述：

**Selector**(typedef struct objc\_selector * SEL)：用于在运行时中表示一个方法的名称。一个方法选择器是一个C字符串，它是在Objective-C运行时被注册的。选择器由编译器生成，并且在类被加载时由运行时自动做映射操作。

**Method**(typedef struct objc_method * Method)：在类定义中表示方法的类型

**Implementation**(typedef id (*IMP)(id, SEL, …))：这是一个指针类型，指向方法实现函数的开始位置。这个函数使用为当前CPU架构实现的标准C调用规范。每一个参数是指向对象自身的指针(self)，第二个参数是方法选择器。然后是方法的实际参数。

理解这几个术语之间的关系最好的方式是：一个类维护一个运行时可接收的消息分发表；分发表中的每个入口是一个方法(Method)，其中key是一个特定名称，即选择器(SEL)，其对应一个实现(IMP)，即指向底层C函数的指针。

为了swizzle一个方法，我们可以在分发表中将一个方法的现有的选择器映射到不同的实现，而将该选择器对应的原始实现关联到一个新的选择器中。

### 调用_cmd

我们回过头来看看前面新的方法的实现代码：

```objc
- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", NSStringFromClass([self class]));
}
```
咋看上去是会导致无限循环的。但令人惊奇的是，并没有出现这种情况。在swizzling的过程中，方法中的[self xxx_viewWillAppear:animated]已经被重新指定到UIViewController类的-viewWillAppear:中。在这种情况下，不会产生无限循环。不过如果我们调用的是[self viewWillAppear:animated]，则会产生无限循环，因为这个方法的实现在运行时已经被重新指定为xxx_viewWillAppear:了。

### 注意事项

Swizzling通常被称作是一种黑魔法，容易产生不可预知的行为和无法预见的后果。虽然它不是最安全的，但如果遵从以下几点预防措施的话，还是比较安全的：

总是调用方法的原始实现(除非有更好的理由不这么做)：API提供了一个输入与输出约定，但其内部实现是一个黑盒。Swizzle一个方法而不调用原始实现可能会打破私有状态底层操作，从而影响到程序的其它部分。

避免冲突：给自定义的分类方法加前缀，从而使其与所依赖的代码库不会存在命名冲突。

明白是怎么回事：简单地拷贝粘贴swizzle代码而不理解它是如何工作的，不仅危险，而且会浪费学习Objective-C运行时的机会。阅读Objective-C Runtime Reference和查看<objc/runtime.h>头文件以了解事件是如何发生的。

小心操作：无论我们对Foundation, UIKit或其它内建框架执行Swizzle操作抱有多大信心，需要知道在下一版本中许多事可能会不一样。

## 协议与分类

Objective-C中的分类允许我们通过给一个类添加方法来扩充它（但是通过category不能添加新的实例变量），并且我们不需要访问类中的代码就可以做到。

Objective-C中的协议是普遍存在的接口定义方式，即在一个类中通过@protocol定义接口，在另外类中实现接口，这种接口定义方式也成为“delegation”模式，@protocol声明了可以呗其他任何方法类实现的方法，协议仅仅是定义一个接口，而由其他的类去负责实现。

### 基础数据类型

**Category**

Category是表示一个指向分类的结构体的指针，其定义如下：

```c
typedef struct objc_category *Category;

struct objc_category {

    char *category_name                          OBJC2_UNAVAILABLE; // 分类名

    char *class_name                             OBJC2_UNAVAILABLE; // 分类所属的类名

    struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE; // 实例方法列表

    struct objc_method_list *class_methods       OBJC2_UNAVAILABLE; // 类方法列表

    struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}
```  
这个结构体主要包含了分类定义的实例方法与类方法，其中instance\_methods列表是objc\_class中方法列表的一个子集，而class_methods列表是元类方法列表的一个子集。

**Protocol**

Protocol的定义如下：

```c
typedef struct objc_object Protocol;
```
我们可以看到，Protocol其中实就是一个对象结构体。

### 操作函数

Runtime并没有在<objc/runtime.h>头文件中提供针对分类的操作函数。因为这些分类中的信息都包含在objc_class中，我们可以通过针对objc_class的操作函数来获取分类的信息。如下例所示：

```objc
@interface RuntimeCategoryClass : NSObject
- (void)method1;
@end

@interface RuntimeCategoryClass (Category)
- (void)method2;
@end

@implementation RuntimeCategoryClass
- (void)method1 {
}
@end

@implementation RuntimeCategoryClass (Category)
- (void)method2 {
}
@end

#pragma mark -

NSLog(@"测试objc_class中的方法列表是否包含分类中的方法");

unsigned int outCount = 0;
Method *methodList = class_copyMethodList(RuntimeCategoryClass.class, &outCount);

for (int i = 0; i < outCount; i++) {
    Method method = methodList[i];
    const char *name = sel_getName(method_getName(method));
    NSLog(@"RuntimeCategoryClass's method: %s", name);
    if (strcmp(name, sel_getName(@selector(method2)))) {
        NSLog(@"分类方法method2在objc_class的方法列表中");
    }
}
```
其输出是：

```
2014-11-08 10:36:39.213 [561:151847] 测试objc_class中的方法列表是否包含分类中的方法
2014-11-08 10:36:39.215 [561:151847] RuntimeCategoryClass's method: method2
2014-11-08 10:36:39.215 [561:151847] RuntimeCategoryClass's method: method1
2014-11-08 10:36:39.215 [561:151847] 分类方法method2在objc_class的方法列表中
```

而对于Protocol，runtime提供了一系列函数来对其进行操作，这些函数包括：

```c
// 返回指定的协议，未实现的协议会返回nil
Protocol * objc_getProtocol(const char *name);

// 获取运行时所知道的所有协议的数组，需要使用free来释放
Protocol ** objc_copyProtocolList(unsigned int *outCount);

// 创建新的协议实例，如果同名的协议已经存在，则返回nil
Protocol * objc_allocateProtocol(const char *name);

// 在运行时中注册新创建的协议。
// 协议注册后便可以使用，但不能再做修改，即注册完后不能再向协议添加方法或协议
void objc_registerProtocol(Protocol *proto);

// 为协议添加方法
void protocol_addMethodDescription(Protocol *proto, SEL name, const char *types, BOOL isRequiredMethod, BOOL isInstanceMethod);

// 添加一个已注册的协议到协议中
void protocol_addProtocol(Protocol *proto, Protocol *addition);

// 为协议添加属性
void protocol_addProperty(Protocol *proto, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount, BOOL isRequiredProperty, BOOL isInstanceProperty);

// 返回协议名
const char * protocol_getName(Protocol *p);

// 测试两个协议是否相等
BOOL protocol_isEqual(Protocol *proto, Protocol *other);

// 获取协议中指定条件的方法的方法描述数组
struct objc_method_description * protocol_copyMethodDescriptionList (Protocol *p, BOOL isRequiredMethod, BOOL isInstanceMethod, unsigned int *outCount);

// 获取协议中指定方法的方法描述
struct objc_method_description protocol_getMethodDescription(Protocol *p, SEL aSel, BOOL isRequiredMethod, BOOL isInstanceMethod);

// 获取协议中的属性列表
objc_property_t * protocol_copyPropertyList(Protocol *proto, unsigned int *outCount);

// 获取协议的指定属性
objc_property_t protocol_getProperty(Protocol *proto, const char *name, BOOL isRequiredProperty, BOOL isInstanceProperty);

// 获取协议采用的协议
Protocol ** protocol_copyProtocolList(Protocol *proto, unsigned int *outCount);

// 查看协议是否采用了另一个协议
BOOL protocol_conformsToProtocol(Protocol *proto, Protocol *other);
```

## 库相关操作

库相关的操作主要是用于获取由系统提供的库相关的信息，主要包含以下函数：

```c
// 获取所有加载的Objective-C框架和动态库的名称
const char ** objc_copyImageNames(unsigned int *outCount);

// 获取指定类所在动态库
const char * class_getImageName(Class cls);

// 获取指定库或框架中所有类的类名
const char ** objc_copyClassNamesForImage(const char *image, unsigned int *outCount);
```
通过这几个函数，我们可以了解到某个类所有的库，以及某个库中包含哪些类。如下代码所示：

```objc
NSLog(@"获取指定类所在动态库");

NSLog(@"UIView's Framework: %s", class_getImageName(NSClassFromString(@"UIView")));

NSLog(@"获取指定库或框架中所有类的类名");
const char ** classes = objc_copyClassNamesForImage(class_getImageName(NSClassFromString(@"UIView")), &outCount);
for (int i = 0; i < outCount; i++) {
    NSLog(@"class name: %s", classes[i]);
}
```
其输出结果如下：

```c
2014-11-08 12:57:32.689 [747:184013] 获取指定类所在动态库
2014-11-08 12:57:32.690 [747:184013] UIView's Framework: /System/Library/Frameworks/UIKit.framework/UIKit
2014-11-08 12:57:32.690 [747:184013] 获取指定库或框架中所有类的类名
2014-11-08 12:57:32.691 [747:184013] class name: UIKeyboardPredictiveSettings
2014-11-08 12:57:32.691 [747:184013] class name: _UIPickerViewTopFrame
2014-11-08 12:57:32.691 [747:184013] class name: _UIOnePartImageView
2014-11-08 12:57:32.692 [747:184013] class name: _UIPickerViewSelectionBar
2014-11-08 12:57:32.692 [747:184013] class name: _UIPickerWheelView
2014-11-08 12:57:32.692 [747:184013] class name: _UIPickerViewTestParameters
......
```

## 块操作

我们都知道block给我们带到极大的方便，苹果也不断提供一些使用block的新的API。同时，苹果在runtime中也提供了一些函数来支持针对block的操作，这些函数包括：

```objc
// 创建一个指针函数的指针，该函数调用时会调用特定的block
IMP imp_implementationWithBlock(id block);

// 返回与IMP(使用imp_implementationWithBlock创建的)相关的block
id imp_getBlock(IMP anImp);

// 解除block与IMP(使用imp_implementationWithBlock创建的)的关联关系，并释放block的拷贝
BOOL imp_removeBlock(IMP anImp);
```
imp\_implementationWithBlock函数：参数block的签名必须是method_return_type ^(id self, method_args …)形式的。该方法能让我们使用block作为IMP。如下代码所示：

```objc
@interface MyRuntimeBlock : NSObject
@end

@implementation MyRuntimeBlock
@end

// 测试代码
IMP imp = imp_implementationWithBlock(^(id obj, NSString *str) {
    NSLog(@"%@", str);
});

class_addMethod(MyRuntimeBlock.class, @selector(testBlock:), imp, "v@:@");

MyRuntimeBlock *runtime = [[MyRuntimeBlock alloc] init];
[runtime performSelector:@selector(testBlock:) withObject:@"hello world!"];
```
输出结果是

```
2014-11-09 14:03:19.779 [1172:395446] hello world!
```

## 弱引用操作

```c
// 加载弱引用指针引用的对象并返回
id objc_loadWeak(id *location);

// 存储__weak变量的新值
id objc_storeWeak(id *location, id obj);
```
● objc_loadWeak函数：该函数加载一个弱指针引用的对象，并在对其做retain和autoreleasing操作后返回它。这样，对象就可以在调用者使用它时保持足够长的生命周期。该函数典型的用法是在任何有使用__weak变量的表达式中使用。

● objc_storeWeak函数：该函数的典型用法是用于__weak变量做为赋值对象时。

可以参考《Objective-C高级编程：iOS与OS X多线程和内存管理》中对__weak实现的介绍。