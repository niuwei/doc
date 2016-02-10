# ios callback - delegate, notification, block, kvo

iOS回调机制 - 代理，通知，block，key value observe

![image1](http://upload-images.jianshu.io/upload_images/552593-5aacbc7a4471c7b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## delegate(代理)

特点：1对1，对象之间没有联系。

```objc
@protocol CallBackDelegate;

@interface ViewController : UIViewController
@property (weak, nonatomic) id<CallBackDelegate> delegate;
@end

@protocol CallBackDelegate <NSObject>
- (void)showArrayWithDelegate:(NSArray *)array;
@end
```

![delegate graph](http://img.blog.csdn.net/20130921004015906?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29jb2FyYW5uaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## notification(通知)

特点：1对多，对象之间没有联系。

```objc
// 发起通知
- (IBAction)callBack
{
    NSDictionary *dict = @{@"array": @[@"Chelsea", @"MUFC", @"Real Madrid"]};
    NSArray *array = dict[@"array"];
    [[NSNotificationCenter defaultCenter] postNotificationName:@"OutputArrayNotification" object:array];
}

// 注册通知
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(outputWithNote:) name:@"OutputArrayNotification" object:nil];
}

// 移除通知
- (void)viewDidDisappear:(BOOL)animated
{
    [super viewDidDisappear:animated];
    [[NSNotificationCenter defaultCenter] removeObserver:self name:@"OutputArrayNotification" object:nil];
}

// 响应通知
- (void)outputWithNote:(NSNotification *)aNotification
{
    NSArray *receiveArray = [aNotification object];
    _outputLabel.text = receiveArray[0];
}
```

![notification graph](http://img.blog.csdn.net/20130921005718187?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29jb2FyYW5uaWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## black

特点：回调，且回调时可以传入参数。封装代码使回调代码位置清晰易读。

```objc
typedef void (^Arr_Block)(NSArray *array);

- (void)showArrayUsingBlock:(Arr_Block)block
{
    NSDictionary *dict = @{@"array": @[@"Chelsea", @"MUFC", @"Real Madrid"]};
    NSArray *array = dict[@"array"];
    block(array);
}

- (IBAction)blockCallBack
{
    [self showArrayUsingBlock:^(NSArray *array) {
        _outputLabel.text = array[1];
    }];
}
```

## kvo(key value observe)

特点：kvo能观察到一个对象的某个属性的变化。当这个对象的属性发生变化的时候会自动调用这个方法。

1 注册

```objc
-(void)addObserver:(NSObject *)anObserver forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(void*)context
```

keyPath就是要观察的属性值，options给你观察键值变化的选择，而context方便传输你需要的数据（注意这是一个void型）。

options是监听的选项，也就是说明监听的字典包含什么值。有两个常用的选项：

NSKeyValueObservingOptionsNew 指返回的字典包含新值。
NSKeyValueObservingOptionsOld 值返回的字典包含旧值。

2 实现变化方法：

```objc
-(void) observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
change:(NSDictionary *)change context:(void*)context
```

change里存储了一些变化的数据，比如变化前的数据，变化后的数据；如果注册时context不为空，这里context就能接收到。

1.person类

```objc
@implementation Person 
@synthesize name,age;//属性name 将被监视 
- (void)changeName 
{ 
    name = @"changeName directly"; 
} 
@end
```

2.PersonMonitor类  监视了name属性

```objc
@implementation PersonMonitor 
 
- (void)observeValueForKeyPath:(NSString *)keyPath 
                      ofObject:(id)object 
                        change:(NSDictionary *)change 
                       context:(void *)context 
{ 
    if ([keyPath isEqual:@"name"]) 
    { 
        NSLog(@"change happen, old:%@   new:%@",[change objectForKey:NSKeyValueChangeOldKey],[change objectForKey:NSKeyValueChangeNewKey]); 
    } 
} 
@end 
```
3.测试代码

```objc
    //初始化被监视对象 
    Person *p =[[Person alloc] init]; 
    //监视对象 
    PersonMonitor *pm= [[PersonMonitor alloc] init]; 
    [p addObserver:pm forKeyPath:@"name" options:(NSKeyValueObservingOptionNew |NSKeyValueObservingOptionOld) context:nil]; 
   
    //测试前的数据 
    NSLog(@"p.name is %@",p.name); 
    
    //通过setvalue 的方法，PersonMonitor的监视将被调用 
    [p setValue:@"name kvc" forKey:@"name"]; 
  
    //查看设置后的值 
    NSLog(@"p name get by kvc is %@",[p valueForKey:@"name"]); 
 
    //效果和通过setValue 是一致的     
    p.name=@"name change by .name="; 
 
    //通过person自己的函数来更改name  
    [p changeName];  
```
输出

``` 
2011-07-03 16:35:57.406 Cocoa[13970:903] p.name is name 
2011-07-03 16:35:57.418 Cocoa[13970:903] change happen, old:name   new:name kvc 
2011-07-03 16:35:57.420 Cocoa[13970:903] p name get by kvc is name kvc 
2011-07-03 16:35:57.421 Cocoa[13970:903] change happen, old:name kvc   new:name change by .name= 
```

