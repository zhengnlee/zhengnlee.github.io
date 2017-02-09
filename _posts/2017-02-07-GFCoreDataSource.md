---
title:  "GFCoreDataSource"
date:   2017-02-06 19:00:14 +0800
tags:
  - Core Data
  - realm
---

阅读此文时，假设你已经对Core Data 有了一定的认识，熟知Core Data 框架下的几个类及基本的使用方法，对多线程操作也有一定的了解。怎么？你是刚刚接触Core Data的？也不要急着走，来都来了，随便看看吧，也没有多少文字。
{% include base_path %}
{% include toc %}

## 概念

关于Core Data 的介绍、各种使用说明cookbook 非常多，在简书站内搜一下，就能发现好几篇非常详尽的、图文并茂的文章，所以这一块儿我就略过了。

## 坑

使用过Core Data 的兄弟们，或多或少会说，Core Data 不同于其它系统框架：难入门、难提高、蹦溃多，等等。网上大量的手册、帮助，但都是基本概念，只教你迈出第一步，但你想欢快地🏃起来，往往还需要自己琢磨，你看，我就琢磨出一条土路，用着还挺顺手，现将其封装，分享出来给各位，请不吝指教。

## 安装

```
pod 'GFCoreDataSource'
```
or source: [github](https://github.com/guofengld/GFCoreDataSource) （内附demo）

源码目录结构：

```
GFCoreDataSource
  |- GFCoreDataSource.h
  |- GFDataSource.h
  |- GFDataSource.m
  |- GFObjectOperation.h
  |- GFObjectOperation.m
```

## 如何使用？

```GFCoreDataSource``` 从名字也能看出来，这就是个数据源管理器，不错，这个Data source 封装了Core Data 对数据库的查询、写入操作，查询操作在**主线程**，写入操作在**子线程**，合并在**主线程**，如果你的项目需要长时间处理大量的数据（超过万条），那可能需要在子线程合并，我帮不了你，或者，建议你采用性能表现更佳的 [realm](https://github.com/realm/realm-cocoa)。

### 定义自己的Data Source

先声明一个基于`GFDataSource`的data source

```
@interface MyDataSource : GFDataSource

@end
```

实例化

```
@implementation MyDataSource

+ (instancetype)sharedClient {
    static dispatch_once_t onceToken;
    static MyDataSource *client = nil;
    dispatch_once(&onceToken, ^{
        client = [[MyDataSource alloc] init];
    });
    
    return client;
}

- (instancetype)init {
    if (self = [super initWithManagedObjectContext:managedObjectContext
                                       coordinator:persistentStoreCoordinator
                                             class:[MyObjectOperation class]]) {
    }
    
    return self;
}
@end
```
此Data source一个实例可以实现对同一个SQLite数据库的多种查询与修改监听，所以，一般只需要创建一个实例，建议用`+ (instancetype)sharedClient;`方法来创建，其初始化时，需要传入的三个参数：

1. managedObjectContext：NSManagedObjectContext 的实例，由主线程创建，查询操作使用（因此，查询操作只能在主线程中使用）
2. persistentStoreCoordinator：NSPersistentStoreCoordinator 的实例，子线程创建NSManagedObjectContext 实例时需要用到
3. [MyObjectOperation class]：此处注册了原子操作时对应的类，此类中实现了具体的增、删、改操作，其声明如下：

```
@interface MyObjectOperation : GFObjectOperation

@end

```
你需要实现相应的方法：

```
@implementation MyObjectOperation

- (void)onAddObject:(id)object {
}

- (void)onEditObject:(NSManagedObject *)object edit:(NSDictionary *)edit {
}

- (void)onDeleteObject:(NSManagedObject *)object {
}

@end
```

### 注册查询、监听

```
self.dataKey = @"viewListing";
self.dataSource = [MyDataSource sharedClient];

[self.dataSource registerDelegate:self
                           entity:[MyEntity entityName]
                        predicate:predicate
                  sortDescriptors:@[dateDescriptor]
               sectionNameKeyPath:nil
                              key:self.dataKey];

```
当前的`self`需要实现`GFDataSourceDelegate` 协议，当数据库中满足查询条件的数据有变动时，`self` 会收到协议方法调用，你如果是在UITableViewController 中注册了监听，则需要添加如下代码：

```

#pragma mark - GFDataSourceDelegate

- (void)dataSource:(id<GFDataSource>)dataSource willChangeContentForKey:(NSString *)key {
    [self.tableView beginUpdates];
}

- (void)dataSource:(id<GFDataSource>)dataSource didChangeContentForKey:(NSString *)key {
    [self.tableView endUpdates];
}

- (void)dataSource:(id<GFDataSource>)dataSource
  didChangeSection:(id<NSFetchedResultsSectionInfo>)sectionInfo
           atIndex:(NSUInteger)sectionIndex
     forChangeType:(NSFetchedResultsChangeType)type
            forKey:(nullable NSString *)key {
    switch (type) {
        case NSFetchedResultsChangeDelete:
            [self.tableView deleteSections:[NSIndexSet indexSetWithIndex:sectionIndex] withRowAnimation:UITableViewRowAnimationNone];
            break;
        case NSFetchedResultsChangeInsert:
            [self.tableView insertSections:[NSIndexSet indexSetWithIndex:sectionIndex] withRowAnimation:UITableViewRowAnimationNone];
            break;
            
        default:
            break;
    }
}

- (void)dataSource:(id<GFDataSource>)dataSource
   didChangeObject:(id)anObject
       atIndexPath:(NSIndexPath *)indexPath
     forChangeType:(NSFetchedResultsChangeType)type
      newIndexPath:(NSIndexPath *)newIndexPath
            forKey:(nullable NSString *)key {
    switch (type) {
        case NSFetchedResultsChangeUpdate:
            [self configureCell:[self.tableView cellForRowAtIndexPath:indexPath] atIndexPath:newIndexPath];
            break;
        case NSFetchedResultsChangeMove:
            [self.tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationNone];
            [self.tableView insertRowsAtIndexPaths:@[newIndexPath] withRowAnimation:UITableViewRowAnimationNone];
            break;
        case NSFetchedResultsChangeInsert:
            [self.tableView insertRowsAtIndexPaths:@[newIndexPath] withRowAnimation:UITableViewRowAnimationNone];
            break;
        case NSFetchedResultsChangeDelete:
            [self.tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationNone];
            break;
    }
}

```
本Data source 支持多个查询条件，只需要用不同的data key来区分，比如在示例demo中有如下查询:

```
self.dataSource = [DataSource sharedClient];
self.boxDataKey = @"boxData";
self.itemDataKey = @"itemData";

NSSortDescriptor *dateDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"date" ascending:NO];
[self.dataSource registerDelegate:self
                           entity:[BoxEntity entityName]
                        predicate:nil
                  sortDescriptors:@[dateDescriptor]
               sectionNameKeyPath:nil
                              key:self.boxDataKey];

[self.dataSource registerDelegate:self
                           entity:[ItemEntity entityName]
                        predicate:nil
                  sortDescriptors:@[dateDescriptor]
               sectionNameKeyPath:nil
                              key:self.itemDataKey];
```

### 数据增、删、改

因数据的修改与界面无关，也与线程无关，所以任何地方如果需要增、删、改数据，直接用`[MyDataSource sharedClient]` 来操作便可，比如：
```
[[MyDataSource sharedClient] addObject:data];
```
修改和删除都类似，可以看demo中具体实现。

## 最后

本地数据库存储数据，已经是app 不可或缺的一部分，我们只需要考虑选用哪一种方案，Core Data 虽然槽点太多，但它是Apple 自家产品，有其天然优势，还能与iCloud 无缝对接，如果你的项目没有处理大量数据的需求，还是建议用Core Data，当然，新起之秀  [realm](https://github.com/realm/realm-cocoa) 也是个不错的选项。

祝好！
