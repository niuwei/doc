# UIAlertView的Delegate用法

曾经遇到一个需求，给用户操作添加Alert的警示框。要添加的地方有多处，有些是从同一个基类里继承的。于是就想到吧显示UIAlertView的代码放在基类中。

ViewControllerSuper.h:

```obcj
@interface ViewControllerSuper : UIViewController <UIAlertViewDelegate>
```

ViewControllerSuper.m:

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self showAlert];
}

- (void)showAlert {
    UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Title"
                                                        message:@"XXX message"
                                                       delegate:self
                                              cancelButtonTitle:@"NO"
                                              otherButtonTitles:@"YES", nil];
    [alertView show];
}

- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    NSLog(@"alert calback on super class.");
}
```
心想这样就可以，其子类在load的时候都会显示这个Alert，然后根据用户的选择YES或NO处理。

然而事情并不是这样，子类中也有Alert调用，早已成为其Alert的Delegete：

ViewController.h:

```objc
@interface ViewController : ViewControllerSuper <UIAlertViewDelegate>
```
ViewController.m:

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    [self showAlert];
}

- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    NSLog(@"alert calback on child class.");
}
```
在基类中展示的Alert，回调由于继承的关系，调用了子类的alertView:clickedButtonAtIndex!

这下麻烦了，难道必须要在每个子类上添加Alert的回调吗，这可是个浩瀚的工程。

既然UIAlertView只能通过设置Delegate指定接受结果的对象，那就给他单独一个没有继承关系的delegate来处理结果，不过最终结果还得super类来处理，兜了一圈子又回来了。

ViewControllerSuper.m:

```objc
#import "ViewControllerSuper.h"

@protocol ShowXXXAlertDelegate <NSObject>
- (void)onShowXXXAlert:(NSInteger)buttonIndex;
@end

@interface ShowXXXAlert : NSObject<UIAlertViewDelegate>
- (void)showXXXAlert;
@property (nullable, nonatomic, weak) id<ShowXXXAlertDelegate> delegate;
@end

@implementation ShowXXXAlert
- (void)showXXXAlert {
    UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Title"
                                                        message:@"XXX message"
                                                       delegate:self
                                              cancelButtonTitle:@"NO"
                                              otherButtonTitles:@"YES", nil];
    [alertView show];
}
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    if (_delegate != nil) { // deleage有nullable，可以不加这个判断
        [_delegate onShowXXXAlert:buttonIndex];
    }
}
@end

@interface ViewControllerSuper () <ShowXXXAlertDelegate> {
    ShowXXXAlert *_showXXXAlert;
}
@end

@implementation ViewControllerSuper

- (void)viewDidLoad {
    [super viewDidLoad];
    [self showAlert2];
}

- (void)showAlert2 {
    if (_showXXXAlert == nil) {
        _showXXXAlert = [[ShowXXXAlert alloc] init];
        _showXXXAlert.delegate = self;
    }
    [_showXXXAlert showXXXAlert];
}

- (void)onShowXXXAlert:(NSInteger)buttonIndex {
    NSLog(@"alert calback on super class.");
}
@end
```
用Delegae的方式接收Alert的结果，由于继承关系产生了这些问题。原本觉得UIAlertView就像VC中MessageBox那样只是轻量级的Domodel窗口，直接返回结果，或者使用block处理结果就够了。看来其机制还是不一样的。

通过这件事情也感受出，同样是callback，delegate和block还是有这些不同的。block不但直观看到回调代码，而且还可以拒绝出现这种继承关系导致的不明确。

> ios8中已经将UIAlertView和UIActionSheet废弃了，取而代之的是UIAlertController，这货不需要设置delegate，通过UIAlertAction的最后一个参数handler添加block来处理callback，比之前的强多了。

## 正确的方法

对于这类采用代理对象来处理回调的情况，为处理继承的因素，一般的方法是：

1、在创建的时候添加tag标示：

```objc
- (void)showAlert {
    UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"Title"
                                                        message:@"XXX message"
                                                       delegate:self
                                              cancelButtonTitle:@"NO"
                                              otherButtonTitles:@"YES", nil];
    [alertView setTag:100];
    [alertView show];
}
```

2、在处理的时候，本类只负责处理本类拥有tag的回调，否则就交予父类处理：

```objc
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    if (alertView.tag == 100) {
        NSLog(@"alert calback on super class.");
    } else {
        [super alertView:alertView clickedButtonAtIndex];
    }
}
```

同KVO的处理observeValueForKeyPath:ofObject:change:context:消息一样，这样回调方法固定的方式收到的消息，都应该这样处理。上面的处理方式仅作权宜之用，如果子类太多，对代码也不熟悉的情况下。