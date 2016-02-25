# Core Data用法

## 使用绑定

### 创建模型文件
  在Core Data，需要进行映射的对象称为实体(entity)，而且需要使用Core Data的模型文件来描述app中的所有实体和实体属性。这里以Person(人)和Card(身份证)2个实体为例子，先看看实体属性和实体之间的关联关系：  
![g0](http://img.my.csdn.net/uploads/201302/01/1359707024_5895.png)  
Person实体中有：name（姓名）、age（年龄）、card（身份证）三个属性  
Card实体中有：no（号码）、person（人）两个属性

1. 添加文件，选择模板  
![g1](http://img.my.csdn.net/uploads/201302/01/1359707426_5763.png)
![g2](http://img.my.csdn.net/uploads/201302/01/1359707501_8695.png)

2. 添加实体  
![g3](http://img.my.csdn.net/uploads/201302/01/1359707563_9302.png)

3. 添加Person的2个基本属性  
![g4](http://img.my.csdn.net/uploads/201302/01/1359707773_7614.png) 

4. 添加Card的1个基本属性  
![g5](http://img.my.csdn.net/uploads/201302/01/1359707796_4561.png)

5. 建立Card和Person的关联关系  
![g6](http://img.my.csdn.net/uploads/201302/01/1359708105_6064.png)
![g7](http://img.my.csdn.net/uploads/201302/01/1359708115_5772.png)  
右图中的![g8](http://img.my.csdn.net/uploads/201302/01/1359708186_1349.png)表示Card中有个Person类型的person属性，目的就是建立Card跟Person之间的一对一关联关系(建议补上这一项)，在Person中加上Inverse属性后，你会发现Card中Inverse属性也自动补上了  
![g9](http://img.my.csdn.net/uploads/201302/01/1359708436_2378.png)

### 了解NSManagedObject

1. 通过Core Data从数据库取出的对象，默认情况下都是NSManagedObject对象
![g10](http://img.my.csdn.net/uploads/201302/01/1359708756_9809.png)
![g11](http://img.my.csdn.net/uploads/201302/01/1359708756_9809.png)

2. NSManagedObject的工作模式有点类似于KVC操作：  
setValue:forKey:存储属性值(属性名为key)  
valueForKey:获取属性值(属性名为key)

### CoreData中的核心对象

![g12](http://img.my.csdn.net/uploads/201302/01/1359708878_8041.png)  
注：黑色表示类名，红色表示类里面的一个属性

#### 开发步骤总结：
 
1. 初始化NSManagedObjectModel对象，加载模型文件，读取app中的所有实体信息
2. 初始化NSPersistentStoreCoordinator对象，添加持久化库(这里采取SQLite数据库)
3. 初始化NSManagedObjectContext对象，拿到这个上下文对象操作实体，进行CRUD操作

### 代码实现

先添加CoreData.framework和导入主头文件\<CoreData/CoreData.h>  
![g13](http://img.my.csdn.net/uploads/201302/01/1359710937_3208.png)

```objc
NSError *error = nil;

@try {
    // 1.搭建上下文环境
    
    // 从应用程序包中加载模型文件
    NSManagedObjectModel *model = [NSManagedObjectModel mergedModelFromBundles:nil];
    // 传入模型对象，初始化NSPersistentStoreCoordinator
    NSPersistentStoreCoordinator *psc = [[NSPersistentStoreCoordinator alloc]
                                         initWithManagedObjectModel:model];
    // 构建SQLite数据库文件的路径
    NSString *docs = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
    NSURL *url = [NSURL fileURLWithPath:[docs stringByAppendingPathComponent:@"person.data"]];
    // 添加持久化存储库，这里使用SQLite作为存储库
    NSPersistentStore *store = [psc addPersistentStoreWithType:NSSQLiteStoreType
                                                 configuration:nil
                                                           URL:url
                                                       options:nil
                                                         error:&error];
    if (store == nil) { // 直接抛异常
        [NSException raise:@"添加数据库错误" format:@"%@", [error localizedDescription]];
    }
    // 初始化上下文，设置persistentStoreCoordinator属性
    NSManagedObjectContext *context = [[NSManagedObjectContext alloc]
                                       initWithConcurrencyType:NSMainQueueConcurrencyType];
    context.persistentStoreCoordinator = psc;
    
    // 2.添加数据到数据库
    
    // 传入上下文，创建一个Person实体对象
    NSManagedObject *person = [NSEntityDescription insertNewObjectForEntityForName:@"Person"
                                                            inManagedObjectContext:context];
    // 设置Person的简单属性
    [person setValue:@"MJ" forKey:@"name"];
    [person setValue:[NSNumber numberWithInt:27] forKey:@"age"];
    // 传入上下文，创建一个Card实体对象
    NSManagedObject *card = [NSEntityDescription insertNewObjectForEntityForName:@"Card"
                                                          inManagedObjectContext:context];
    [card setValue:@"4414241933432" forKey:@"no"];
    // 设置Person和Card之间的关联关系
    [person setValue:card forKey:@"card"];
    // 利用上下文对象，将数据同步到持久化存储库
    BOOL success = [context save:&error];
    if (!success) {
        [NSException raise:@"访问数据库错误" format:@"%@", [error localizedDescription]];
    }
    
    // 3.从数据库中查询数据
    
    // 初始化一个查询请求
    NSFetchRequest *request = [[NSFetchRequest alloc] init];
    // 设置要查询的实体
    request.entity = [NSEntityDescription entityForName:@"Person"
                                 inManagedObjectContext:context];
    // 设置排序（按照age降序）
    NSSortDescriptor *sort = [NSSortDescriptor sortDescriptorWithKey:@"age" ascending:NO];
    request.sortDescriptors = [NSArray arrayWithObject:sort];
    // 设置条件过滤(搜索name中包含字符串"Itcast-1"的记录，
    // 注意：设置条件过滤时，数据库SQL语句中的%要用*来代替，
    // 所以%Itcast-1%应该写成*Itcast-1*)
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name like %@", @"*Itcast-1*"];
    request.predicate = predicate;
    // 执行请求
    NSArray *objs = [context executeFetchRequest:request error:&error];
    if (error) {
        [NSException raise:@"查询错误" format:@"%@", [error localizedDescription]];
    }
    // 遍历数据
    for (NSManagedObject *obj in objs) {
        NSLog(@"name=%@", [obj valueForKey:@"name"]);
    }
    
    // 4.删除数据库中的数据
    
    NSManagedObject *deleteObject = nil;
    if ([objs count] != 0) {
        deleteObject = objs[0];
    }
    // 传入需要删除的实体对象
    [context deleteObject:deleteObject];
    // 将结果同步到数据库
    [context save:&error];
    if (error) {
        [NSException raise:@"删除错误" format:@"%@", [error localizedDescription]];
    }
}
@catch (NSException *exception) {
    NSLog(@"%@", exception);
}
@finally {
}
```

> 注：Core Data不会根据实体中的关联关系立即获取相应的关联对象，比如通过Core Data取出Person实体时，并不会立即查询相关联的Card实体；当应用真的需要使用Card时，才会再次查询数据库，加载Card实体的信息。这个就是Core Data的延迟加载机制

### 打开CoreData的SQL语句输出开关

1. 打开Product，点击EditScheme...  

2. 点击Arguments，在ArgumentsPassed On Launch中添加2项  
1> -com.apple.CoreData.SQLDebug  
2> 1  
![g15](http://img.my.csdn.net/uploads/201302/01/1359711964_1550.png)

### 创建NSManagedObject的子类

默认情况下，利用Core Data取出的实体都是NSManagedObject类型的，能够利用键-值对来存取数据。但是一般情况下，实体在存取数据的基础上，有时还需要添加一些业务方法来完成一些其他任务，那么就必须创建NSManagedObject的子类  
![g16](http://img.my.csdn.net/uploads/201302/01/1359712054_3978.png)

选择模型文件  
![g17](http://img.my.csdn.net/uploads/201302/01/1359712079_5045.png)

选择需要创建子类的实体  
![g18](http://img.my.csdn.net/uploads/201302/01/1359712094_5888.png)

创建完毕后，多了2个子类   
![g19](http://img.my.csdn.net/uploads/201302/01/1359712116_3772.png)

文件内容展示：

Person.h  

```objc
#import <Foundation/Foundation.h>  
#import <CoreData/CoreData.h>  
  
@class Card;  
  
@interface Person : NSManagedObject  
  
@property (nonatomic, retain) NSString * name;  
@property (nonatomic, retain) NSNumber * age;  
@property (nonatomic, retain) Card *card;  
  
@end  
```

Person.m

```objc
#import "Person.h"  
  
@implementation Person  
  
@dynamic name;  
@dynamic age;  
@dynamic card;  
  
@end
```

Card.h
   
```objc
#import <Foundation/Foundation.h>  
#import <CoreData/CoreData.h>  
  
@class Person;  
  
@interface Card : NSManagedObject  
  
@property (nonatomic, retain) NSString * no;  
@property (nonatomic, retain) Person *person;  
  
@end
```

Card.m

```objc
#import "Card.h"  
#import "Person.h"  
  
@implementation Card  
  
@dynamic no;  
@dynamic person;  
  
@end  
```

那么往数据库中添加数据的时候就应该写了：

```objc
Person *person = [NSEntityDescription insertNewObjectForEntityForName:@"Person" inManagedObjectContext:context];  
person.name = @"MJ";  
person.age = [NSNumber numberWithInt:27];  
  
Card *card = [NSEntityDescription insertNewObjectForEntityForName:@”Card" inManagedObjectContext:context];  
card.no = @”4414245465656";  
person.card = card;  
// 最后调用[context save&error];保存数据
```

## 手动创建实体

### 创建并设置模型类

```objc
- (NSManagedObjectModel *)managedObjectModel
{
    // 创建了一个全局模型
    static NSManagedObjectModel *moModel = nil;
    
    if (moModel != nil) {
        return moModel;
    }
    
    moModel = [[NSManagedObjectModel alloc] init];
    
    // 创建实体名为Run，对应的托管对象类名为Run
    NSEntityDescription *runEntity = [[NSEntityDescription alloc] init];
    [runEntity setName:@"Run"];
    [runEntity setManagedObjectClassName:@"Run"];
    // 加入实体到模型中
    [moModel setEntities:[NSArray arrayWithObject:runEntity]];
    
    // 给实体添加属性data，表示运行时间
    NSAttributeDescription *dateAttribute = [[NSAttributeDescription alloc] init];
    [dateAttribute setName:@"date"];
    [dateAttribute setAttributeType:NSDateAttributeType];
    [dateAttribute setOptional:NO];
    // 给实体添加属性processID，表示进程ID
    NSAttributeDescription *idAttribute = [[NSAttributeDescription alloc] init];
    [idAttribute setName:@"processID"];
    [idAttribute setAttributeType:NSInteger32AttributeType];
    [idAttribute setOptional:NO];
    [idAttribute setDefaultValue:[NSNumber numberWithInteger:-1]];
    
    // 给进程ID添加检验条件，必须大于0
    // 等价于：validationPredicate = [NSPredicate predicateWithFormat:@"SELF > 0"]
    NSExpression *lhs = [NSExpression expressionForEvaluatedObject];
    NSExpression *rhs = [NSExpression expressionForConstantValue:[NSNumber numberWithInteger:0]];
    
    NSPredicate *validationPredicate = [NSComparisonPredicate
                                        predicateWithLeftExpression:lhs
                                        rightExpression:rhs
                                        modifier:NSDirectPredicateModifier
                                        type:NSGreaterThanPredicateOperatorType
                                        options:0];
    
    NSString *validationWarning = @"Process ID < 1";
    [idAttribute setValidationPredicates:[NSArray arrayWithObject:validationPredicate]
                  withValidationWarnings:[NSArray arrayWithObject:validationWarning]];
    
    // 将两个属性设置给实体
    NSArray *properties = [NSArray arrayWithObjects: dateAttribute, idAttribute, nil];
    [runEntity setProperties:properties];
    
    // 给模型设置本地化描述词典
    NSMutableDictionary *localizationDictionary = [NSMutableDictionary dictionary];
    [localizationDictionary setObject:@"Date" forKey:@"Property/date/Entity/Run"];
    [localizationDictionary setObject:@"Process ID" forKey:@"Property/processID/Entity/Run"];
    [localizationDictionary setObject:@"Process ID must not be less than 1" forKey:@"ErrorString/Process ID < 1"];
    [moModel setLocalizationDictionary:localizationDictionary];
    
    return moModel;
}
```

本地化描述提供对 Entity，Property，Error信息等的便于理解的描述，其可用的键值对如下表：

Key           | Value
------------- | -------------
"Entity/NonLocalizedEntityName" | "LocalizedEntityName"
"Property/NonLocalizedPropertyName/Entity/EntityName" | "LocalizedPropertyName"
"Property/NonLocalizedPropertyName" | "LocalizedPropertyName"
"ErrorString/NonLocalizedErrorString" | "LocalizedErrorString"

### 创建并设置运行时类和对象

设置存储路径:

```objc
- (NSURL *)applicationLogDirectory {
    
    NSString *LOG_DIRECTORY = @"CoreDataTutorial";
    static NSURL *ald = nil;
    
    if (ald == nil)
    {
        NSFileManager *fileManager = [[NSFileManager alloc] init];
        NSError *error = nil;
        NSURL *libraryURL = [fileManager URLForDirectory:NSLibraryDirectory
                                                inDomain:NSUserDomainMask
                                       appropriateForURL:nil
                                                  create:YES
                                                   error:&error];
        if (libraryURL == nil) {
            NSLog(@"Could not access Library directory\n%@", [error localizedDescription]);
        }
        else
        {
            ald = [libraryURL URLByAppendingPathComponent:@"Logs"];
            ald = [ald URLByAppendingPathComponent:LOG_DIRECTORY];
            
            NSLog(@" >> log path %@", [ald path]);
            
            NSDictionary *properties =
            [ald resourceValuesForKeys:[NSArray arrayWithObject:NSURLIsDirectoryKey]
                                 error:&error];
            if (properties == nil)
            {
                if (![fileManager createDirectoryAtPath:[ald path]
                            withIntermediateDirectories:YES
                                             attributes:nil
                                                  error:&error])
                {
                    NSLog(@"Could not create directory %@\n%@",
                          [ald path], [error localizedDescription]);
                    ald = nil;
                }
            }
        }
    }
    
    return ald;
}
```

在上面的代码中，我们将持久化数据文件保存到路径：/Users/kesalin/Library/Logs/CoreDataTutorial 下。

创建运行时对象：ManagedObjectContext 和 PersistentStoreCoordinator

```objc
- (NSManagedObjectContext *)managedObjectContext {
    
    // 创建了一个全局 ManagedObjectContext 对象
    static NSManagedObjectContext *moContext = nil;
    if (moContext != nil) {
        return moContext;
    }
    
    moContext = [[NSManagedObjectContext alloc] init];
    
    // 传入模型对象，创建persistent store coordinator
    NSManagedObjectModel *moModel = [self managedObjectModel];
    NSPersistentStoreCoordinator *coordinator =
    [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:moModel];
    // 给context设置coordinator
    [moContext setPersistentStoreCoordinator: coordinator];
    
    // 创建persistent store，XML类型，放在给定路径+CoreDataTutorial.xml下
    NSString *STORE_TYPE = NSXMLStoreType;
    NSString *STORE_FILENAME = @"CoreDataTutorial.xml";
    
    NSError *error = nil;
    NSURL *url = [[self applicationLogDirectory] URLByAppendingPathComponent:STORE_FILENAME];
    
    NSPersistentStore *newStore = [coordinator addPersistentStoreWithType:STORE_TYPE
                                                            configuration:nil
                                                                      URL:url
                                                                  options:nil
                                                                    error:&error];
    
    if (newStore == nil) {
        NSLog(@"Store Configuration Failure\n%@",
              ([error localizedDescription] != nil) ?
              [error localizedDescription] : @"Unknown Error");
    }
    
    return moContext;
}
```

下面我们就来定义数据对象类。向工程添加 Core Data->NSManagedObject subclass 的类，名为 Run (模型中 Entity 定义的类名) 

Run.h

```objc
#import <CoreData/NSManagedObject.h>

@interface Run : NSManagedObject {
    NSInteger processID;
}

@property (retain) NSDate *date;
@property (retain) NSDate *primitiveDate;
@property NSInteger processID;

@end
```

Run.m

```objc
@implementation Run

@dynamic date;
@dynamic primitiveDate;

- (void) awakeFromInsert{
    [super awakeFromInsert];
    self.primitiveDate = [NSDate date];
}

- (NSInteger)processID {
    [self willAccessValueForKey:@"processID"];
    NSInteger pid = processID;
    [self didAccessValueForKey:@"processID"];
    return pid;
}

- (void)setProcessID:(NSInteger)newProcessID {
    [self willChangeValueForKey:@"processID"];
    processID = newProcessID;
    [self didChangeValueForKey:@"processID"];
}

// Implement a setNilValueForKey: method. 
// If the key is “processID” then set processID to 0.
- (void)setNilValueForKey:(NSString *)key {
    
    if ([key isEqualToString:@"processID"]) {
        self.processID = 0;
    }
    else {
        [super setNilValueForKey:key];
    }
}
@end
```

注意：

1）这个类中的 date 和 primitiveDate 的访问属性为 @dynamic，这表明在运行期会动态生成对应的 setter 和 getter；  
2）在这里我们演示了如何正确地手动实现 processID 的 setter 和 getter：为了让 ManagedObjecContext  能够检测 processID的变化，以及自动支持 undo/redo，我们需要在访问和更改数据对象时告之系统，will/didAccessValueForKey 以及 will/didChangeValueForKey 就是起这个作用的。  
3）当我们设置 nil 给数据对象 processID 时，我们可以在 setNilValueForKey 捕获这个情况，并将 processID  置 0；  
4）当数据对象被插入到 ManagedObjectContext 时，我们在 awakeFromInsert 将时间设置为当前时间。

### 创建或读取数据对象，设置其值，保存

```objc
NSError *error = nil;

// 获得全局的 Managed Object Model 对象
NSManagedObjectModel *moModel = [self managedObjectModel];
NSLog(@"The managed object model is defined as follows:\n%@", moModel);

if ([self applicationLogDirectory] == nil) {
    return;
}

// 获得全局的 Managed Object Context 对象
NSManagedObjectContext *moContext = [self managedObjectContext];

// 创建一个Run Entity，设置其 Property processID 为当前进程的 ID
NSEntityDescription *runEntity = [[moModel entitiesByName] objectForKey:@"Run"];
Run *run = [[Run alloc] initWithEntity:runEntity
        insertIntoManagedObjectContext:moContext];
NSProcessInfo *processInfo = [NSProcessInfo processInfo];
run.processID = [processInfo processIdentifier];

// 将该数据对象保存到持久化文件中
if (![moContext save: &error]) {
    NSLog(@"Error while saving\n%@",
          ([error localizedDescription] != nil) ?
          [error localizedDescription] : @"Unknown Error");
    return;
}

// 创建一个 FetchRequest 来查询持久化数据文件中保存的数据记录
NSFetchRequest *request = [[NSFetchRequest alloc] init];
[request setEntity:runEntity];
// 将结果按照日期升序排列
NSSortDescriptor *sortDescriptor = [[NSSortDescriptor alloc] initWithKey:@"date"
                                                               ascending:YES];
[request setSortDescriptors:[NSArray arrayWithObject:sortDescriptor]];

error = nil;
NSArray *array = [moContext executeFetchRequest:request error:&error];
if ((error != nil) || (array == nil))
{
    NSLog(@"Error while fetching\n%@",
          ([error localizedDescription] != nil) ?
          [error localizedDescription] : @"Unknown Error");
    return;
}

// 将查询结果打印输出
NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
[formatter setDateStyle:NSDateFormatterMediumStyle];
[formatter setTimeStyle:NSDateFormatterMediumStyle];

NSLog(@"%@ run history:", [processInfo processName]);

for (run in array)
{
    NSLog(@"On %@ as process ID %ld",
          [formatter stringForObjectValue:run.date],
          run.processID);
}
```

编译运行，我们可以得到如下显示：

```
2011-09-03 21:42:47.556 CoreDataTutorial[992:903] CoreDataTutorial run history:  
2011-09-03 21:42:47.557 CoreDataTutorial[992:903] On 2011-9-3 下午09:41:56 as process ID 940  
2011-09-03 21:42:47.557 CoreDataTutorial[992:903] On 2011-9-3 下午09:42:16 as process ID 955  
2011-09-03 21:42:47.558 CoreDataTutorial[992:903] On 2011-9-3 下午09:42:20 as process ID 965  
2011-09-03 21:42:47.558 CoreDataTutorial[992:903] On 2011-9-3 下午09:42:24 as process ID 978  
2011-09-03 21:42:47.559 CoreDataTutorial[992:903] On 2011-9-3 下午09:42:47 as process ID 992  
```
通过这个例子，我们可以更好理解 Core Data  的运作机制。在 Core Data 中我们最常用的就是 ManagedObjectContext，它几乎参与对数据对象的所有操作，包括对 undo/redo 的支持；而 Entity 对应的运行时类为 ManagedObject，我们可以理解为抽象数据结构 Entity 在内存中由 ManagedObject 来体现，而 Perproty 数据类型在内存中则由 ManagedObject 类的成员属性来体现。一般我们不需要与 PersistentStoreCoordinator 打交道，对数据文件的读写操作都由 ManagedObjectContext 为我们代劳了。

参考：

[Core Data入门](http://blog.csdn.net/q199109106q/article/details/8563438/) 修改了部分代码  
[[Cocoa]深入浅出 Cocoa 之 Core Data（2）- 手动编写代码](http://blog.csdn.net/kesalin/article/details/6746117) 