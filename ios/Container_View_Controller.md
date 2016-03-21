# Container View Controller

原文：[Implementing a Container View Controller](https://developer.apple.com/library/prerelease/ios/featuredarticles/ViewControllerPGforiPhoneOS/ImplementingaContainerViewController.html#//apple_ref/doc/uid/TP40007457-CH11-SW1)

容器视图控制器（Container view controllers，简称：容器VC）提供了一种方式，来组合多个视图控制器（view controller，简称：VC）到一个界面。容器VC常用于界面导航，还可以用现有内容创建新界面。在UIKit中，提供诸如UINavgationController、UITabBarController、UIPageViewController和UISplitViewController等。

## 设计一个自定义的容器类视图控制器

容器VC就像其他内容VC一样，管理一个根View和若干内容View。不同的是，容器VC从其他VC中获取内容。获取的内容仅限于其他VC的Views，将这些Views嵌入自身View层次。容器VC设置嵌入views的size和位置，但是这些view还是由其原始的controller来管理。

当设计的容器VC时，总是要理解容器和被包容的视图控制器之间的关系。这可以帮助你清楚被管理的控制器的内容如果现实到屏幕上，和本质上是怎样管理这些内容的。在设计流程中，需要考虑下面这些问题：

1. 容器VC和子VC扮演什么角色？
2. 有多少子VC可以同时呈现？
3. 如果有的话，子VC之间是什么关系？
4. 子VC如何加入和移除。
5. 子VC的位置和Size可以改变吗？在什么条件下发生？
6. 容器VC是否提供具体的组件或者类似于导航的东西来管理这些子视图么？
7. 子视图控制器和容器视图控制器的通讯方式是什么？以及容器视图控制器是否需要传送一些不是UIViewController提供的标准事件的特殊事件给子视图控制器？
8. 是否有不同的方式来呈现容器类视图控制器，如果有，怎么呈现？

当你思考完这些问题并且制定出相应的规则之后，实现一个容器类视图控制器将是一件非常简单的事情。UIKit中唯一需要的条件是构建容器视图控制器和你要管理的视图控制器之间的父子关系。这种父子关系能够保证子视图控制器能够接收到系统发送的消息。（子视图控制器的生命周期函数的正常调用等）除此之外，每一种容器类视图控制器不同的地方就是在布局和管理期间对它包含的views的做的操作。你可以增加一些自定义的Views到一些定制的组件和导航的视图层级中去。

### 例子：导航控制器

UINavigationController对象通过内容的层级相关数据来导航。导航控制器（navigation controller）一次展现一个View，上部的导航栏显示当前在子视图控制器当中的哪一个，左边后退按钮是返回上一个层级，向下导航交由数据层级的子VC决定，会影响table和button的行为。VC之间导航由导航控制器和其子类共同管理。当用户和子VC的按钮和表的一行进行交互，子VC请求导航控制器push一个新vc到容器view上，子VC更新内容配置，但导航控制器处理转场动画。导航控制器也管理导航bar，上面显示着返回按钮，以清除最上面的VC以返回之前的VC。

图5-1显示了一个导航控制器和它上面视图的结构，大部分区域是被子视图控制器给填充的，只有一小部分区域是导航栏。

Figure 5-1 Structure of a navigation interface

![navigation interface](https://developer.apple.com/library/prerelease/ios/featuredarticles/ViewControllerPGforiPhoneOS/Art/VCPG_structure-of-navigation-interface_5-1_2x.png)

无论简洁还是通常环境下，一个导航控制器一次只显示一个子vc。导航控制器重置子视图的布局以适应可用空间。

### 例子：分栏控制器

一个UISplitViewController对象可以分为主副区域来显示两个子视图控制器。在这种安排下，一个视图控制器的内容（主）决定另外一个副区域显示哪个子视图控制器。两个视图控制器的显示是可以配置的，同时也取决于两个子视图控制器的的当前环境。在宽松的水平环境下，分栏控制器可以显示两个子视图控制器，在紧凑的模式下，分栏控制器只能在同一时间显示一个试图控制器。

图5-2展示了一个分栏控制器在宽松的水平环境下的结构，分栏控制器仅仅包含自己默认的容器view，在这个例子中，两个子视图控制器显示在两边，子视图控制器的大小是可配置的，从而可以显示主控制器。

Figure 5-2 A split view interface

![split view interface](https://developer.apple.com/library/prerelease/ios/featuredarticles/ViewControllerPGforiPhoneOS/Art/VCPG-split-view-inerface_5-2_2x.png)

## 在interface builder中配置一个容器类视图控制器

在设计时创建父子容器关系，需要在storyboard场景里加入容器view对象。容器view对象是一个占位对象，用来显示子VC的内容。使用视图的大小和位置确定子视图控制器的根视图在容器类视图控制器中的位置和大小。

Figure 5-3 Adding a container view in Interface Builder

![in Interface Builder](https://developer.apple.com/library/prerelease/ios/featuredarticles/ViewControllerPGforiPhoneOS/Art/container_view_embed_2x.png)

当加载有一个或多个内容View的容器VC时，Interface Builder同时加载子View关联的VC。子视图控制器在此期间必须完成初始化并且父子关系也会被创建。

如果你不用Interface Builder去建立父子关系，必须通过代码形式创建，情见后面的“增加一个子视图控制器到你的容器里”

## 实现一个自定义的容器类试图控制器

要实现容器VC，你要在它和它的子VC间建立父子关系。在你尝试管理这些子VC的View之前。这是让UIKit知道你的容器VC在管理的子VC的位置和尺寸。你可以通过Interface Builder或编码创建这些关系。用编码，把添加移除子VC明确作为你容器VC设置的一部分。

### 增加一个子视图控制器到你的容器里

用代码添加子视图控制器，创建父子关系通过下面步骤：

1. 调用容器视图控制器 addChildViewController: 的方法，这个方法用来告诉UIKit你的容器类视图控制器即将要管理一个子视图控制器。
2. 增加子视图控制器的view到容器视图控制的view（或者Container View）的视图层级结构中，千万别忘记设置大小的位置。
3. 为管理子视图控制的view的大小和位置添加约束条件。
4. 调用子视图控制的 didMoveToParentViewController: 方法。

表5-1 展示了一个容器类视图控制器怎样嵌入子视图控制器，当创建完父子关系之后，在将子视图加入到容器视图中，设置了子视图的frame size，这非常重要，保证正确的将子视图显示在容器中。在增加完之后，容器类视图控制器调用didMoveToParentViewController:方法，这是因为要给子视图控制器能够响应变化的机会。（调用相应的生命周期函数布局根视图）

Listing 5-1Adding a child view controller to a container

```objc
- (void) displayContentController: (UIViewController*) content {
   [self addChildViewController:content];
   content.view.frame = [self frameForContentController];
   [self.view addSubview:self.currentClientView];
   [content didMoveToParentViewController:self];
}
```

在上一个例子中，你会发现你仅仅调用子视图控制器的didMoveToParentViewController:，这是因为容器类视图控制器调用addChildViewController:会引发自动调用子视图控制器的willMoveToParentViewController:，你必须自己调用容器类视图控制器的didMoveToParentViewController:方法，原因是你必须在完成将子视图控制器的view嵌入到容器类视图控制器的视图层级之后才能调用该方法。

当你使用AutoLayout的时候，请在将配置约束条件的工作放在子视图控制的view已经嵌入到容器类视图控制器的层级结构中。你的约束条件必须仅仅能影响子视图控制器的根视图的位置和大小，不要修改根视图的内容或者是其他子视图控制器视图结构中的视图。

### 移除一个子视图控制器

为了移除一个子视图控制器，移除父子关系必须遵循以下步骤：

1. 调用子视图的 willMoveToParentViewController: 方法 参数为nil。
2. 移除任何你对子视图控制的view的约束条件。
3. 将子视图控制的view从容器类视图控制器的层级结构中移除。
4. 调用子视图控制器的 removeFromParentViewController 方法，从而最终结束父子关系。

永久性的断绝子视图控制器与容器视图控制器的父子关系，移除一个子视图控制器必须保证你将不再需要引用这个子视图控制器。举个例子，一个导航控制器在push进来一个新的子视图控制器到导航栈中并不需要移除当前的子视图控制器，仅仅在该子视图控制器从导航栈中弹出才进行移除子视图控制器。

表5-2 展示了一个子视图控制器如何从容器中移除。调用willMoveToParentViewController:的方法，参数为nil，给子视图控制器一个应对变化的一个机会。removeFromParentViewController也会调用子视图控制器的didMoveToParentViewController: 的方法，传递nil的参数。最终解除与容器的父子关系：

Listing 5-2Removing a child view controller from a container

```objc
- (void) hideContentController: (UIViewController*) content {
   [content willMoveToParentViewController:nil];
   [content.view removeFromSuperview];
   [content removeFromParentViewController];
}
``

### 子视图控制器之间的转场

当你想要使用包含移入移出动画来切换子视图控制器的时候，在这些动画之前，你需要保证两个子视图控制器都是容器视图控制器的内容的一部分，并且你要使当前子视图控制器知道它即将要开始动画。在动画期间，要使得新的子视图控制器进入到预定位置，并且要移出旧的子视图控制器的view。在完成动画的时候，需要彻底完成移出子视图控制器的工作。

表5-3 展示了一个使用转场动画完成一个子视图控制器切换到另外一个子视图控制器的例子。在这个例子中，新的视图控制器开启动画移动到当前正在进行移出屏幕的视图控制器的矩形位置上。当动画完成，回调的block会从容器视图控制器移出子视图控制器，在这个例子中，transitionFromViewController:toViewController:duration:options:animations:completion: 这个方法自动的更新容器视图控制器中的view层级结构，所以你不需要增加或者删除views。

Listing 5-3Transitioning between two child view controllers

```objc
- (void)cycleFromViewController: (UIViewController*) oldVC
               toViewController: (UIViewController*) newVC {
   // Prepare the two view controllers for the change.
   [oldVC willMoveToParentViewController:nil];
   [self addChildViewController:newVC];
 
   // Get the start frame of the new view controller and the end frame
   // for the old view controller. Both rectangles are offscreen.
   newVC.view.frame = [self newViewStartFrame];
   CGRect endFrame = [self oldViewEndFrame];
 
   // Queue up the transition animation.
   [self transitionFromViewController: oldVC toViewController: newVC
        duration: 0.25 options:0
        animations:^{
            // Animate the views to their final positions.
            newVC.view.frame = oldVC.view.frame;
            oldVC.view.frame = endFrame;
        }
        completion:^(BOOL finished) {
           // Remove the old view controller and send the final
           // notification to the new view controller.
           [oldVC removeFromParentViewController];
           [newVC didMoveToParentViewController:self];
        }];
}
```

### 管理子视图控制器的外观更新

在将子视图控制器增加到容器视图控制器之后，容器类视图控制器自动的将外观相关的消息传送给子视图。这是正常的情况，因为保证了所有的事件都被发送。然而，有些时候，默认的行为在一条指令中发送一些事件，然而并没有使容器察觉。例如，如果多个子视图控制器同时更新它们view的状态，你可能会想合并这些变化，因此外观的回调全部发生在同一时间。

为了接管处理这些外观变化的回调，需要在你的容器类视图控制器中重写shouldAutomaticallyForwardAppearanceMethods 的方法，返回值为NO，在表5-4中的例子，返回NO使得UIKit知道你的容器类视图控制器会通知它的子视图控制器在其界面中的变化。

Listing 5-4Disabling automatic appearance forwarding

```objc
- (BOOL) shouldAutomaticallyForwardAppearanceMethods {
    return NO;
}
```

当显示的转场发生的时候，调用子视图的beginAppearanceTransition:animated:与endAppearanceTransition 来显现。例如，你的容器有一个单一的子视图控制器，你可以在不同的生命周期函数中调用，确保子视图控制器获取到这些消息。如表5-5。

Listing 5-5Forwarding appearance messages when the container appears or disappears

```objc
-(void) viewWillAppear:(BOOL)animated {
    [self.child beginAppearanceTransition: YES animated: animated];
}
 
-(void) viewDidAppear:(BOOL)animated {
    [self.child endAppearanceTransition];
}
 
-(void) viewWillDisappear:(BOOL)animated {
    [self.child beginAppearanceTransition: NO animated: animated];
}
 
-(void) viewDidDisappear:(BOOL)animated {
    [self.child endAppearanceTransition];
}
```

## 创建一个容器类视图控制器的建议

设计，开发和测试一个新的容器类视图控制器是需要花费时间的，即使独立的行为是简单易懂的，但将视图控制器作为整体仍然是复杂的，当实现一个容器类控制器的类的时候，请考虑以下贴士:

* 仅仅有获取子视图控制器根视图的权限。容器视图控制器仅仅只需要有每一个子视图控制器根视图的访问权限，比如返回子视图的view的属性。从不应该获取其他views的权限。
* 子视图控制器应该最小限度的了解它的容器（也就是说子视图控制器不需要做很多的配置才能够被添加到容器中，子视图控制器与容器控制器应该是松耦合的）。子视图控制器应该把精力放在自己内容的管理上. 如果一个容器类视图控制器想要影响子视图控制器上的内容，不要直接操作，使用代理设计模式去管理这些影响。
* Design your container using regular views first. Using regular views (instead of the views from child view controllers) gives you an opportunity to test layout constraints and animated transitions in a simplified environment. When the regular views work as expected, swap them out for the views of your child view controllers.

## 委托控制子视图控制器

一个容器试图控制器可以代理自身的appearance和一个或者多个子视图控制器，可以采用如下方式代理：

* 让子视图控制器决定状态栏风格。将子视图控制器作为修改状态栏的代理, 重写 childViewControllerForStatusBarStyle 和 childViewControllerForStatusBarHidden 在容器视图控制器中的两个方法
* 让子视图控制器提供期许的尺寸大小，容器类控制器有灵活的适应性的布局，能够使用子视图控制器的preferredContentSize 来帮助定位其尺寸

翻译参考：  
[Implementing a Container View Controller 翻译+自我实践](http://www.jianshu.com/p/68a589e244e4)