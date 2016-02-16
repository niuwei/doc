# iOS事件处理-响应链（The Responder Chain）

## 事件分类

事件类型有以下三种：

1. 触屏事件（Touch Event）
2. 运动事件（Motion Event）
3. 远端控制事件（Remote-Control Event）

当用户通过以上方式触发一个事件时，会将相应的事件对象添加到UIApplication的事件队列中。UIApplication会循环的从队列中拿出第一个事件来处理。首先将该事件分发给UIApplication 的主窗口对象(KeyWindow)，然后由主窗口决定如何将事件交给最合适的响应者(UIResponder)来处理取决于事件的类型。这里主要分两种情况：

1. 触摸事件：UIApplication通过一个触摸检测来决定最合适来处理该事件的响应者，一般情况下，这个响应者是UIView对象。
2. 加速计事件或远程遥控事件：UIApplication寻找UIWindow中的第一响应者。找到第一响应者(The First Responder)后，会将该事件对象派发给该响应者以便处理。

## 第一响应者(The First Responder)

第一响应者对象（The first responder）被设计用来第一个接收事件，一个对象要成为“第一响应者对象”需要做到下面两点:

1. 重载 canBecomeFirstResponder 方法并返回 YES。
2. 接收 becomeFirstResponder 消息。需要的话, 对象可以给自身发送这个消息。
3. canResignFirstResponder 第一响应者身份是否可以取消。resignFirstResponder取消第一响应者。isFirstResponder判断是否为第一相应者。

> 注意：指定第一响应者之前确保应用程序已经建立了它的object graph。例如，通常在重写viewdidappear方法中调用becomefirstresponder。如果你想在viewwillappear指定第一响应者，此时object graph尚未建立，所以becomefirstresponder方法返回NO。

[Shake-Motion url]:https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/motion_event_basics/motion_event_basics.html#//apple_ref/doc/uid/TP40009541-CH6-SW2

[Edit Menu url]:https://developer.apple.com/library/ios/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/AddingCustomEditMenuItems/AddingCustomEditMenuItems.html#//apple_ref/doc/uid/TP40009542-CH13

[Custom Views url]:https://developer.apple.com/library/ios/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/InputViews/InputViews.html#//apple_ref/doc/uid/TP40009542-CH12

Events 不是仅有的依赖响应者链的对象。响应者链有以下的用途：

* Touch events. 如果hit-test view 不能处理事件，事件将上交给上一个hit-test view.
* Motion events. 为处理晃动事件，第一响应者必须实现UIResponder类的motionBegan:withEvent:方法和 motionEnded:withEvent:方法。见[Detecting Shake-Motion Events with UIEvent][Shake-Motion url];
* Remote control events. 为处理远程控制事件，第一响应者必须实现UIResponder类的remoteControlReceivedWithEvent:方法。
* Action messages. 当用户操作控件，比如按钮或switch按钮，且动作方法的目标为nil，消息会通过响应者链从control view发出。
* Editing-menu messages. When a user taps the commands of the editing menu, iOS uses a responder chain to find an object that implements the necessary methods (such as cut:, copy:, and paste:). For more information, see [Displaying and Managing the Edit Menu][Edit Menu url] and the sample code project, CopyPasteTile.
* Text editing. When a user taps a text field or a text view, that view automatically becomes the first responder. By default, the virtual keyboard appears and the text field or text view becomes the focus of editing. You can display a custom input view instead of the keyboard if it’s appropriate for your app. You can also add a custom input view to any responder object. For more information, see [Custom Views for Data Input][Custom Views url].

UIKit自动设置用户点击的文本区域或者text view为第一响应者。程序必须通过becomeFirstResponder方法明确设置所有其他的第一响应者。


## 触摸事件和Hit-Testing

iOS用Hit-Testing来查找哪个view被点击了。Hit-Testing检查点击是否在一个view的bounds内，如果是，再以此检查次view的subview。包含点击的层及最低的的view成为hit-test view。传递点击事件给他处理。这个view就是最优先处理点击事件的view。

![Hit-Testing](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/hit_testing_2x.png)

1. 点击在view A内，检查其subview B和C。
2. 点击不在view B内，在view C内，检查subview D和E。
3. 点击不在view D内，在view E内。
4. view E成为最底层级的接受点击的view，所以它成为hit-test view。

> hitTest:withEvent:方法返回hit test view。他调用自身的pointInside:withEvent:方法

首先我们需要明确一个UIView对象能够接收触摸事件至少要保证以下三个条件：

1. userInteractionEnabled属性为YES，该属性表示允许控件同用户交互。
2. Hidden属性为NO。
3. opacity属性值0 ～0.01，不能透明过分了吧？

找到响应者后，响应者可以重写以下方法来对触摸事件做响应：

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesBegan:touches withEvent:event];//让下一个响应者可以有机会继续处理
}
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesBegan:touches withEvent:event];
}
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesBegan:touches withEvent:event];
}
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesBegan:touches withEvent:event];
}
```

## 响应者链（Responder Chain）

响应者链由一系列的响应者对象（Responder Object）组成。它始于第一响应者，终止于application object。如果前面的响应者无法处理事件，就交由响应者链后面的响应者处理。

**UIResponder**是所有响应对象的基类，在UIResponder类中定义了处理上述各种事件的接口。UIApplication、 UIViewController、UIView都继承自UIResponder，所以它们的实例都是可以构成响应者链的响应者对象。

> 注意：Core Animation layers不是响应者对象。

下图展示了响应者链的基本构成：

![Figure 2](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/iOS_responder_chain_2x.png)

![graph 1](http://img.blog.csdn.net/20130707181937406?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3p6dmljdG9yeV90anNk/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上图中可以看到，响应者链有以下特点：

1. 响应者链通常是由视图（UIView）构成的；
2. 一个视图的下一个响应者是它视图控制器（UIViewController）（如果有的话），然后再转给它的父视图（Super View）；
3. 视图控制器（如果有的话）的下一个响应者为其管理的视图的父视图；
4. 单例的窗口（UIWindow）的内容视图将指向窗口本身作为它的下一个响应者
5. 单例的应用（UIApplication）是一个响应者链的终点，它的下一个响应者指向nil，以结束整个循环。

> Important: If you implement a custom view to handle remote control events, action messages, shake-motion events with UIKit, or editing-menu messages, don’t forward the event or message to nextResponder directly to send it up the responder chain. Instead, invoke the superclass implementation of the current event handling method and let UIKit handle the traversal of the responder chain for you.

## 事件分发（Event Delivery）

第一响应者（First responder）指的是当前接受触摸的响应者对象（通常是一个UIView对象），即表示当前该对象正在与用户交互，它是响应者链的开端。整个响应者链和事件分发的使命都是找出第一响应者。

UIWindow对象以消息的形式将事件发送给第一响应者，使其有机会首先处理事件。如果第一响应者没有进行处理，系统就将事件（通过消息）传递给响应者链中的下一个响应者，看看它是否可以进行处理。

iOS系统检测到手指触摸(Touch)操作时会将其打包成一个UIEvent对象，并放入当前活动Application的事件队列，单例的UIApplication会从事件队列中取出触摸事件并传递给单例的UIWindow来处理，UIWindow对象首先会使用hitTest:withEvent:方法寻找此次Touch操作初始点所在的视图(View)，即需要将触摸事件传递给其处理的视图，这个过程称之为hit-test view。

UIWindow实例对象会首先在它的内容视图上调用**hitTest:withEvent:**，此方法会在其视图层级结构中的每个视图上调用**pointInside:withEvent:**（该方法用来判断点击事件发生的位置是否处于当前视图范围内，以确定用户是不是点击了当前视图），如果pointInside:withEvent:返回YES，则继续逐级调用，直到找到touch操作发生的位置，这个视图也就是要找的hit-test view。

hitTest:withEvent:方法的处理流程如下:

首先调用当前视图的pointInside:withEvent:方法判断触摸点是否在当前视图内；

若返回NO,则hitTest:withEvent:返回nil;

若返回YES,则向当前视图的所有子视图(subviews)发送hitTest:withEvent:消息，所有子视图的遍历顺序是从最顶层视图一直到到最底层视图，即从subviews数组的末尾向前遍历，直到有子视图返回非空对象或者全部子视图遍历完毕；

若第一次有子视图返回非空对象，则hitTest:withEvent:方法返回此对
象，处理结束；

如所有子视图都返回非，则hitTest:withEvent:方法返回自身(self)。

加入用户点击了View E，下面结合图二介绍hit-test view的流程：

1. A是UIWindow的根视图，因此，UIWindwo对象会首相对A进行hit-test；
2. 显然用户点击的范围是在A的范围内，因此，pointInside:withEvent:返回了YES，这时会继续检查A的子视图；
3. 这时候会有两个分支，B和C：点击的范围不再B内，因此B分支的pointInside:withEvent:返回NO，对应的hitTest:withEvent:返回nil；点击的范围在C内，即C的pointInside:withEvent:返回YES；
4. 这时候有D和E两个分支：
点击的范围不再D内，因此D的pointInside:withEvent:返回NO，对应的hitTest:withEvent:返回nil；
点击的范围在E内，即E的pointInside:withEvent:返回YES，由于E没有子视图（也可以理解成对E的子视图进行hit-test时返回了nil），因此，E的hitTest:withEvent:会将E返回，再往回回溯，就是C的hitTest:withEvent:返回E--->>A的hitTest:withEvent:返回E。

至此，本次点击事件的第一响应者就通过响应者链的事件分发逻辑成功的找到了。

不难看出，这个处理流程有点类似二分搜索的思想，这样能以最快的速度，最精确地定位出能响应触摸事件的UIView。

## 说明

1、如果最终hit-test没有找到第一响应者，或者第一响应者没有处理该事件，则该事件会沿着响应者链向上回溯，如果UIWindow实例和UIApplication实例都不能处理该事件，则该事件会被丢弃；

2、hitTest:withEvent:方法将会忽略隐藏(hidden=YES)的视图，禁止用户操作(userInteractionEnabled=YES)的视图，以及alpha级别小于0.01(alpha<0.01)的视图。如果一个子视图的区域超过父视图的bound区域(父视图的clipsToBounds 属性为NO，这样超过父视图bound区域的子视图内容也会显示)，那么正常情况下对子视图在父视图之外区域的触摸操作不会被识别,因为父视图的pointInside:withEvent:方法会返回NO,这样就不会继续向下遍历子视图了。当然，也可以重写pointInside:withEvent:方法来处理这种情况。

3、我们可以重写hitTest:withEvent:来达到某些特定的目的，下面的链接就是一个有趣的应用举例，当然实际应用中很少用到这些。


[Event Delivery: The Responder Chain](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/event_delivery_responder_chain/event_delivery_responder_chain.html#//apple_ref/doc/uid/TP40009541-CH4-SW1)

[http://www.knowsky.com/884401.html](http://www.knowsky.com/884401.html)