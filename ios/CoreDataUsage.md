# Core Data用法

[Core Data入门](http://blog.csdn.net/q199109106q/article/details/8563438/) 修改了部分代码

## 创建模型文件
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

## 了解NSManagedObject

1. 通过Core Data从数据库取出的对象，默认情况下都是NSManagedObject对象
![g10](http://img.my.csdn.net/uploads/201302/01/1359708756_9809.png)
![g11](http://img.my.csdn.net/uploads/201302/01/1359708756_9809.png)

2. NSManagedObject的工作模式有点类似于KVC操作：  
setValue:forKey:存储属性值(属性名为key)  
valueForKey:获取属性值(属性名为key)

## CoreData中的核心对象

![g12](http://img.my.csdn.net/uploads/201302/01/1359708878_8041.png)  
注：黑色表示类名，红色表示类里面的一个属性

### 开发步骤总结：
 
1. 初始化NSManagedObjectModel对象，加载模型文件，读取app中的所有实体信息
2. 初始化NSPersistentStoreCoordinator对象，添加持久化库(这里采取SQLite数据库)
3. 初始化NSManagedObjectContext对象，拿到这个上下文对象操作实体，进行CRUD操作

## 代码实现

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

## 打开CoreData的SQL语句输出开关

1. 打开Product，点击EditScheme...  

2. 点击Arguments，在ArgumentsPassed On Launch中添加2项  
1> -com.apple.CoreData.SQLDebug  
2> 1  
![g15](http://img.my.csdn.net/uploads/201302/01/1359711964_1550.png)

## 创建NSManagedObject的子类

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