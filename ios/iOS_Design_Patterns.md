# iOS设计模式

## 单例

### 设计单例需要注意的问题

1. 共享实例访问入口的唯一性，方便性。
2. 唯一性，阻止创建多个实例。
3. 实例的析构问题。
4. 多线程安全问题。

### 不使用全局变量和类静态方法

* 不使用全局变量，因为会引出未封装变量的维护问题，不符合面向对象特性。
* 不使用类的静态方法，因为需要子类化的时候，需要修改原类名为自类名。

> 在使用类的任意位置硬编码类名会降低将来在使用一个不同类的灵活性.

Objective-C：

```objc
+ (MySingletonClass *)sharedInstance {
  static MySingletonClass *instance;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{ instance = [[self alloc] init]; });
  return instance;
}
```
局部静态变量保存共享实例指针，确保访问入口必须为sharedInstance。dispatch_once确保线程安全只调用一次。

Swift：

```swift
class TheOneAndOnlyKraken {
    static let sharedInstance = TheOneAndOnlyKraken()
    private init() {} //This prevents others from using the default '()' initializer for this class.
}
```
全局变量（还有结构体和枚举体的静态成员）的Lazy初始化方法会在其被访问的时候调用一次，局部静态变量确保访问入口唯一和线程安全。init设置为私有的确保创建入口唯一。