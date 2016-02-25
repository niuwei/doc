# Core Data

Core Data 管理模型对象。Core Data是object-graph管理和持久化框架。

* 提供持久化存储，模型对象和数据存储相互转化。
* 提供基础，为跟踪模型对象的变化，自动支持undo和redo，保持对象之间的相互关系。
* 分离编辑组件和模型对象。未生效的编辑不影响模型对象。
* 内存中只加载模型对象的一部分，以节省内存消耗。
* 为数据存储版本迁移提供基础。

## Core Data模块

[Core Data stack url]:https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/single_persistent_stack.jpg

![Core Data stack][Core Data stack url]

从下到上依次为：

* 一个外部持久化存储（external persistent store），用来保存数据记录。
* 一个持久化对象存储（persistent object store），映射存储中的记录和程序中的对象。
* 一个持久化存储协调器（persistent store coordinator），聚合所有的存储。
* 一个托管对象模型（managed object model），描述存储的实体。
* 一个托管对象上下文（managed object context），提供托管对象的数据暂存器（scratch pad）。

一个stack只有一个coordinator。创建新的coordinator意味着创建新的stack。多个model可以汇总为一个，这个model有多个store，多个context，但coordinator只有一个。

托管对象上下文通常直接连接持久化存储协调器，但被其他上下文连接形成父子关系。

## 托管对象（Managed object）

托管对象，作为MVC中的model，代表存储中的一条记录（record）。他是 <a name="fenced-code-block">NSManagedObject</a> 或其子类实例，其在内存中对应实体（Entity，相当于表）的一条数据，数据成员为实体的Property。托管对象由context注册，一个托管对象代表store中的一条记录。

托管对象有实体描述（NSEntityDescription，相当于数据库中的一个表）对象的引用。一般不需要给每个实体定义一个NSManagedObject子类。当然，你也可以把特定实体定义为NSManagedObject子类，来实现一些附加行为：比如，计算派生的属性值，或实现验证逻辑。

![Managed object](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/mapping_moc_record.jpg)

### 用法

每一个 Managed Object 都有一个全局 ID（类型为：NSManagedObjectID）。Managed Object 会附加到一个 Managed Object Context，我们可以通过这个全局 ID 在 Managed Object Context 查询对应的 Managed Object。

NSManagedObject 常用方法：

Method        | Introduce
------------- | -------------
entity           | 获取其 Entity
objectID         | 获取其 Managed Object ID
valueForKey:     | 获取指定 Property 的值
setValue:forKey: | 设定指定 Property 的值

## 托管对象上下文（Managed object context）

托管对象上下文代表Core Data程序的一个单独对象空间，或者数据暂存器（scratch pad）。他是 <a name="fenced-code-block">NSManagedObjectContext</a> 的实例。它的主要管理一个托管对象的集合。这些托管对象表现为一个或多个存储的一致性视图。它管理托管对象生命期、监测数据变化、更新绑定数据到UI，和undo/redo。

托管对象上下文是Core Data stack的中心。它可以创建（create）和抓取（fetch）托管对象，管理undo、redo操作。给定了上下文，一个托管对象就代表存储的一条记录。

![Managed object context](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/moc_record.jpg)

context可以连接父级，父级可以是coordinator或者另一个context。当抓取（fetch）对象时，context会向父级发出抓取请求。你对托管对象的修改直到你保存context的时候才会提交给父级。

### 用法

当创建一个数据对象并插入 Managed Object Context 中，Context就开始跟踪这个数据对象的一切变动，并在合适的时候提供对 undo/redo 的支持，或将变化保存到数据文件中去。

通常我们将 controller 类（如：NSArrayController，NSTreeController）或其子类与 Managed Object Context 绑定，这样就方便我们动态地生成，获取数据对象等。

NSManagedObjectContext 常用方法：

Method        | Introduce
------------- | -------------
save:         | 将数据对象保存到数据文件
objectWithID: | 查询指定 Managed Object ID 的数据对象
deleteObject: | 将一个数据对象标记为删除，但是要等到 Context 提交更改时才真正删除数据对象
undo          | 回滚最后一步操作，这是都 undo/redo 的支持
lock          | 加锁，常用于多线程以及创建事务。同类接口还有：-unlock and -tryLock
rollback      | 还原数据文件内容
reset         | 清除缓存的 Managed Objects。只应当在添加或删除 Persistent Stores 时使用
undoManager   | 返回当前 Context 所使用的 NSUndoManager
assignObject:toPersistantStore: |	由于 Context 可以管理从不同数据文件而来的数据对象，这个接口的作用就是指定数据对象的存储数据文件（通过指定 PersistantStore 实现）
executeFetchRequest:error:      |	执行 Fetch Request 并返回所有匹配的数据对象

## 持久化存储协调器（Persistent store coordinator）

持久化存储协调器，关联存储和托管对象模型，用外观（facade）模式，把一组存储当做一个提供给context。它是 <a name="fenced-code-block">NSPersistentStoreCoordinator</a> 的实例。它有托管对象模型的引用，以描述实体和它管理的存储。

复杂的程序中有多个store，每个store包含不同实体（entities），coordinator的作用是提供给contexts这些store的facade。当你请求数据，Core Data检索所有store，除非你指定了store。

![Persistent store coordinator](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/advanced_persistent_stack.jpg)

### 使用

使用 Core Data document 类型的应用程序，通常会从磁盘上的数据文中中读取或存储数据，这些底层的读写就由 Persistent Store Coordinator 来处理。一般我们无需与它直接打交道来读写文件，Managed Object Context 在背后已经为我们调用 Persistent Store Coordinator 做了这部分工作。

NSPersistentStoreCoordinator 常用方法：

Method        | Introduce
------------- | -------------
addPersistentStoreForURL: configuration:URL:options:error: | 装载数据存储，对应的卸载数据存储的接口为 removePersistentStore:error:
migratePersistentStore: toURL:options:withType:error: | 迁移数据存储，效果与 "save as"相似，但是操作成功后，迁移前的数据存储不可再使用
managedObjectIDForURIRepresentation: | 返回给定 URL所指示的数据存储的 object id，如果找不到匹配的数据存储则返回 nil
persistentStoreForURL: |	返回指定路径的 Persistent Store
URLForPersistentStore: |	返回指定 Persistent Store 的存储路径

## 抓取请求（Fetch request）

抓取请求告诉context，要抓取的托管对象的实体(Entity)，且选择性的指定了约束了对象属性，返回对象的顺序。请求是<a name="fenced-code-block"> NSFetchRequest </a>的实例。指定实体为<a name="fenced-code-block"> NSEntityDescription </a>的实例，约束条件为<a name="fenced-code-block"> NSPredicate </a>对象，排序条件为<a name="fenced-code-block"> NSSortDescriptor </a>的实例。以上类似于数据库SELECT语法的：table name, WHERE clause, and ORDER BY clauses。

你可以通过给context发送executeFetchRequest:error:消息来执行抓取请求。context返回一个包含结果的托管对象数组。

![Fetch request](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/Art/fetch_flow_of_data.jpg)

### 用法

Fetch Requests 相当于一个查询语句，你必须指定要查询的 Entity。我们通过 Fetch Requests 向 Managed Object Context 查询符合条件的数据对象，以 NSArray 形式返回查询结果，如果我们没有设置任何查询条件，则返回该 Entity 的所有数据对象。我们可以使用谓词来设置查询条件，通常会将常用的 Fetch Requests 保存到 dictionary 以重复利用。

示例：

```objc
NSManagedObjectContext * context  = [[NSApp delegate] managedObjectContext];  
NSManagedObjectModel   * model    = [[NSApp delegate] managedObjectModel];  
NSDictionary           * entities = [model entitiesByName];  
NSEntityDescription    * entity   = [entities valueForKey:@"Post"];  
  
NSPredicate * predicate;  
predicate = [NSPredicate predicateWithFormat:@"creationDate > %@", date];  
                           
NSSortDescriptor * sort = [[NSortDescriptor alloc] initWithKey:@"title"];  
NSArray * sortDescriptors = [NSArray arrayWithObject: sort];  
  
NSFetchRequest * fetch = [[NSFetchRequest alloc] init];  
[fetch setEntity: entity];  
[fetch setPredicate: predicate];  
[fetch setSortDescriptors: sortDescriptors];  
  
NSArray * results = [context executeFetchRequest:fetch error:nil];  
[sort release];  
[fetch release];
```
在上面代码中，我们查询在指定日期之后创建的 post，并将查询结果按照 title 排序返回。

NSFetchRequest 常用方法：

Method        | Introduce
------------- | -------------
setEntity:         | 设置你要查询的数据对象的类型（Entity）
setPredicate:      | 设置查询条件
setFetchLimit:     | 设置最大查询对象数目
setSortDescriptors:| 设置查询结果的排序方法
setAffectedStores: | 设置可以在哪些数据存储中查询

## 持久化存储（Persistent store）

持久化存储是一个存储托管对象的仓库。你可以想想它为数据库文件，里面保存着托管对象最后保存的记录。Core Data提供了3种原生文件类型：二进制、XML和SQLite。你可以实现自己的存储类型。Core Data也提供了进程生存期内的内存存储。

存储的实体被托管对象模型在创建时定义。如果你改变了程序的模型，你就不能打开任何之前模型创建的存储。你必须首先迁移存储的内容到新的格式中。

## 托管对象模型（Managed object model）

托管对象模型，是描述你程序中一系列托管对象组织的模型。它把存储中的记录映射为托管对象。这个模型包含实体描述对象（entity description objects，<a name="fenced-code-block"> NSEntityDescription</a> 实例），实体可以看数据库的一个表，表的条目包含：实体的名称、程序中代表实体的类名、实体的特征（properties）。

![Managed object model][Core Data stack url]

### 用法

模型有点像数据库的表结构，里面包含 Entry， 实体又包含三种 Property：Attribute（属性），RelationShip（关系）， Fetched Property（读取属性）。Model class 的名字多以 "Description" 结尾。我们可以看出：模型就是描述数据类型以及其关系的。

主要的 Model class 有：   
1. 数据模型（Managed Object Model）：NSManagedObjectModel  
2. 实体（Entity）：NSEntityDescription - 抽象数据类型，相当于数据库中的表  
3. 特性（Property）：NSPropertyDescription -	相当于数据库表中的一列   

* Attribute：NSAttributeDescription	基本数值型属性（如Int16, BOOL, Date等类型的属性）   
* Relationship：NSRelationshipDescription - 属性之间的关系  
* Fetched Property：NSFetchedPropertyDescription - 查询属性（相当于数据库中的查询语句）


## 持久化对象存储（Persistent object store）

持久化对象存储是你app里的对象和持久化存储中记录的映射。那些不同的持久化对象存储类代表不同Core Data支持的不同文件类型。你也可以实现自己想要支持的文件类型。

你不能直接创建持久化对象存储。当你发送addPersistentStoreWithType:configuration:URL:options:error:消息给coordinator时，Core Data为你创建适当类型的存储。

## 映射模型（Mapping model）

Core Data映射模型描述了一种迁移数据必须的转换，从源托管对象模型描述到目的模型描述。当你做个新版本的代理对象模型，你需要从旧图表迁移持久化数据到新图表。对于简单的模型修改，Core Data能够推断需要的映射。对于复杂的修改，你需要提供一个映射模型描述如果执行迁移。

## Persistent Document

NSPersistentDocument 是 NSDocument 的子类。 multi-document Core Data 应用程序使用它来简化对 Core Data 的操作。通常使用 NSPersistentDocument 的默认实现就足够了，它从 Info.plist 中读取 Document types 信息来决定数据的存储格式（xml,sqlite, binary）。

NSPersistentDocument 常用方法：

Method        | Introduce
------------- | -------------
managedObjectContext | 返回文档的 Managed Object Context，在多文档应用程序中，每个文档都有自己的 Context。
managedObjectModel   | 返回文档的 Managed Object Model

## 实体（Entity）

Entity 相当于数据库中的一个表，它描述一种抽象数据类型，其对应的类为 NSManagedObject 或其子类。

NSEntityDescription 常用方法:

Method        | Introduce
------------- | -------------
+insertNewObjectForEntityForName: inManagedObjectContext: | 工厂方法，根据给定的 Entity 描述，生成相应的 NSManagedObject 对象，并插入 ManagedObjectContext 中。
-managedObjectClassName | 返回映射到 Entity 的 NSManagedObject 类名
-attributesByName | 以名字为 key， 返回 Entity 中对应的 Attributes
-relationshipsByName | 以名字为 key， 返回 Entity 中对应的 Relationships

Property - NSPropertyDescription  
Property 为 Entity 的特性，它相当于数据库表中的一列，或者 XML 文件中的 value-key 对中的 key。它可以描述实体数据(Attribute)，Entity之间的关系(RelationShip)，或查询属性(Fetched Property)。

Attribute - NSAttributeDescription  
Attribute 存储基本数据，如 NSString, NSNumber or NSDate 等。它可以有默认值，也可以使用正则表达式或其他条件对其值进行限定。一个属性可以是 optional 的。

Relationship - NSRelationshipDescription  
Relationship 描述 Entity，Property 之间的关系，可以是一对一，也可以是一对多的关系。

数据插入代码：

```objc
NSManagedObjectContext * context = [[NSApp delegate] managedObjectContext];  
NSManagedObject        * author  = nil;  
      
author = [NSEntityDescription insertNewObjectForEntityForName:@"Author"
                                       inManagedObjectContext:context];
[author setValue: @"nemo@pixar.com" forKey: @"email"];
[context save:&error];
```
在上面代码中，我们先取得 NSManagedObjectContext，然后调用 NSEntityDescription 的方法，以 Author 为实体模型，生成对应的 NSManagedObject 对象，插入 NSManagedObjectContext 中，然后给这个对象设置特性 email 的值。

## 代码实现

1. 应用程序先创建或读取模型文件（后缀为xcdatamodeld）生成 NSManagedObjectModel 对象。Document应用程序是一般是通过 NSDocument 或其子类 NSPersistentDocument）从模型文件（后缀为 xcdatamodeld）读取。  
2. 然后生成 NSManagedObjectContext 和 NSPersistentStoreCoordinator 对象，前者对用户透明地调用后者对数据文件进行读写。  
3. NSPersistentStoreCoordinator 负责从数据文件(xml, sqlite,二进制文件等)中读取数据生成 Managed Object，或保存 Managed Object 写入数据文件。  
4. NSManagedObjectContext 参与对数据进行各种操作的整个过程，它持有 Managed Object。我们通过它来监测 Managed Object。监测数据对象有两个作用：支持 undo/redo 以及数据绑定。这个类是最常被用到的。  
5. Array Controller, Object Controller, Tree Controller 这些控制器一般与 NSManagedObjectContext 关联，因此我们可以通过它们在 nib 中可视化地操作数据对象。


参考：

[Core Data Core Competencies](https://developer.apple.com/library/prerelease/mac/documentation/DataManagement/Devpedia-CoreData/coreDataOverview.html#//apple_ref/doc/uid/TP40010398-CH28-SW1)  
[[Cocoa]深入浅出 Cocoa 之 Core Data（1）- 框架详解](http://blog.csdn.net/kesalin/article/details/6739319)  
[[Cocoa]深入浅出 Cocoa 之 Core Data（2）- 手动编写代码](http://blog.csdn.net/kesalin/article/details/6746117)  
[[Cocoa]深入浅出 Cocoa 之 Core Data（3）- 使用绑定](http://blog.csdn.net/kesalin/article/details/6757279)  
[[Cocoa]深入浅出 Cocoa 之 Core Data（4）- 使用绑定](http://blog.csdn.net/kesalin/article/details/6757412)