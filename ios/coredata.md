# Core Data

Core Data 管理模型对象。Core Data是object-graph管理和持久化框架。

* 提供持久化存储，模型对象和数据存储相互转化。
* 提供基础，为跟踪模型对象的变化，自动支持undo和redo，保持对象之间的相互关系。
* 分离编辑组件和模型对象。未生效的编辑不影响模型对象。
* 内存中只加载模型对象的一部分，以节省内存消耗。
* 为数据存储版本迁移提供基础。

## Core Data模块

* 一个外部持久化存储（external persistent store），用来保存数据记录。
* 一个持久化对象存储（persistent object store），映射存储中的记录和程序中的对象。
* 一个持久化存储协调器（persistent store coordinator），聚合所有的存储。
* 一个托管对象模型（managed object model），描述存储的实体。
* 一个托管对象上下文（managed object context），提供托管对象的数据暂存器（scratch pad）。

![Core Data stack](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/single_persistent_stack.jpg)

A stack is effectively defined by a persistent store coordinator—there is one and only one per stack. Creating a new persistent store coordinator implies creating a new stack. By implication, there is therefore only one model, although it may be aggregated from multiple models. There may be multiple stores—and hence object stores—and multiple managed object contexts.

A managed object context is usually connected directly to a persistent store coordinator, but may be connected to another context in a parent-child relationship.

## 托管对象（Managed object）

托管对象，是持久化存储的一条记录的对象模型（MVC模型）。他是NSManagedObject或其子类实例。托管对象注册为托管对象上下文。一个给定的上下文中，只有一个持久化存储对象实例，映射持久化存储的一条记录。

![Managed object](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/mapping_moc_record.jpg)

托管对象保存实体描述对象（entity description object）的引用，后者告诉我们实体代表的意义。因此，NSManagedObject可以代表任何实体，你不需要给每个实体一个独一无二的子类。你可以定义子类实现自定义行为，例如，计算派生的属性值，或实现验证逻辑。

## 托管对象上下文（Managed object context）

托管对象上下文代表Core Data程序的一个单独对象空间，或者数据暂存器。托管对象上下文是NSManagedObjectContext的实例。它的首要职责是管理一个托管对象的集合。这些托管对象代表一个或多个持久化存储的内部一致视图。上下文是一个强大的的对象，在托管对象的声明周期内位于中心角色。有生命周期管理职责，以验证、逆转管理处理，和undo/redo。

托管对象上下文，是Core Data stack的中心。可以用来创建和获取托管对象，管理undo、redo操作。在它的管理下，至多由一个托管对象就可以代表持久化存储的一条记录。

![Managed object context](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/moc_record.jpg)

上下文可以连接父级对象存储，通常是一个持久化存储协调器，也可以是另一个托管对象上下文。当你请求对象，上下文向父级对象存储发出请求。你对托管对象的修改直到你保存上下文的时候才会提交给父级存储。

## 持久化存储协调器（Persistent store coordinator）

持久化存储协调器，关联持久化对象存储和托管对象模型，给托管对象上下文提供了的外观（facade）模式，使一组持久化存储当做一个单独的存储。它是NSPersistentStoreCoordinator的实例。它有托管对象模型的引用。当你请求数据，Core Data检索所有store，除非你指定了store。

![Persistent store coordinator](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/advanced_persistent_stack.jpg)

## 抓取请求（Fetch request）

抓取请求告诉代理对象上下文，我们想要抓取的实体，指定了比如约束了对象属性，或返回对象的顺序。抓取请求是NSFetchRequest的实例。返回指定的实体表现为NSEntityDescription的实例，约束条件表现为NSPredicate对象，排序为NSSortDescriptor的实例。这些类似于数据库SELECT语法的：table name, WHERE clause, and ORDER BY clauses。

你可以通过给托管对象上下文发消息来执行抓取请求。上下文返回一个包含结果的数组。

![Fetch request](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/fetch_flow_of_data.jpg)

## 持久化存储（Persistent store）

持久化存储是一个存储托管对象的仓库。你可以想想它为数据库文件，里面保存着托管对象最后保存的记录。Core Data提供了3种原生文件类型：二进制、XML和SQLite。你可以实现自己的存储类型。Core Data也提供了进程生存期内的内存存储。

存储的实体被托管对象模型在创建时定义。如果你改变了程序的模型，你就不能打开任何之前模型创建的存储。你必须首先迁移存储的内容到新的格式中。

## 托管对象模型（Managed object model）





参考：

[Core Data Core Competencies](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/coreDataOverview.html#//apple_ref/doc/uid/TP40010398-CH28-SW1)  
[[Cocoa]深入浅出 Cocoa 之 Core Data（1）- 框架详解](http://blog.csdn.net/kesalin/article/details/6739319)  
[Core Data入门](http://blog.csdn.net/q199109106q/article/details/8563438/)