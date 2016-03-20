# Container View Controller

原文：[Implementing a Container View Controller](https://developer.apple.com/library/prerelease/ios/featuredarticles/ViewControllerPGforiPhoneOS/ImplementingaContainerViewController.html#//apple_ref/doc/uid/TP40007457-CH11-SW1)

容器视图控制器（Container view controllers，简称：容器VC）提供了一种方式，来组合多个视图控制器（view controller，简称：VC）到一个界面。容器VC常用于界面导航，还可以用现有内容创建新界面。在UIKit中，提供诸如UINavgationController、UITabBarController、UIPageViewController和UISplitViewController等。

## 设计一个自定义的容器类视图控制器

In almost every way, a container view controller is like any other content view controller in that it manages a root view and some content. The difference is that a container view controller gets part of its content from other view controllers. The content it gets is limited to the other view controllers’ views, which it embeds inside its own view hierarchy. The container view controller sets the size and position of any embedded views, but the original view controllers still manage the content inside those views.

容器VC就像其他内容VC一样，管理一个根View和若干内容View。不同的是，容器VC从其他VC中获取内容。获取的内容仅限于其他VC的Views，将这些Views嵌入自身View层次。容器VC设置嵌入views的size和位置，但是这些view还是由其原始的controller来管理。

When designing your own container view controllers, always understand the relationships between the container and contained view controllers. The relationships of the view controllers can help inform how their content should appear onscreen and how your container manages them internally. During the design process, ask yourself the following questions:

当你设计的容器VC，必须理解包容和被包容的VC之间的关系。这VC之间的关系

## Implementing a Custom Container View Controller

### Adding a Child View Controller to Your Content