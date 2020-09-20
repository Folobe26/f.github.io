---
title: 常见Foundation需要判空的方法
date: 2020-09-20 20:52:09
tags:
---

【Feb】常用Foundation中需要对参数判空/判值的方法

Foundation中的类是日常开发中最常使用的，由于OC没有可选类型，很多时候要手动添加空判断，否则可能会因为一些外部原因导致大量的线上crash。

Collection              NS_ASSUME_NONNULL_BEGIN/END

- NSArray

**NSArray的整个头文件是被NS_ASSUME_NONNULL_BEGIN/END包住的**，所以理论上没有特别标明nullable的都需要传入前判空，检查下来，只有个别方法的context/option之类的作用的参数表明了nullable。下面列举几个高频使用的：

- [- initWithObjects:](http://apple-reference-documentation//hcAwvVoPlz)
- [- indexOfObject:](http://apple-reference-documentation//hctSRELWv5)
- [- objectsAtIndexes:](http://apple-reference-documentation//hcvvy16NAh)
- [- arrayByAddingObject:](http://apple-reference-documentation//hcc0rXmtUu)
- [- arrayByAddingObjectsFromArray:](http://apple-reference-documentation//hcYSNYYseL)

- NSMutableArray

**NSMutableArray也是被NS_ASSUME_NONNULL_BEGIN/END包住的。**

- [- addObject:](http://apple-reference-documentation//hcqhh_qX0z)
- [- addObjectsFromArray:](http://apple-reference-documentation//hc1vi7SemO)
- [- insertObject:atIndex:](http://apple-reference-documentation//hcObePqE4s)
- [- insertObjects:atIndexes:](http://apple-reference-documentation//hc7qpyAnwh)
- [- removeObject:](http://apple-reference-documentation//hcwYFGOkyI)
- [- removeObject:inRange:](http://apple-reference-documentation//hc8eQ4jmC6)
- [- removeObjectsAtIndexes:](http://apple-reference-documentation//hc96sYh-Ha)
- [- replaceObjectAtIndex:withObject:](http://apple-reference-documentation//hcP98I2W3-)

- NSDictionary 

NSDictionary的崩溃大部分是对语法糖/下标语法理解不够导致的。

这里还是讨论方法和KVC为主

KVC的方法定义：

- - (**nullable** **id**)valueForKey:(NSString *)key;

- - valueForKey在NSKeyValueCoding协议中有详细的流程描述

- - (**void**)setValue:(**nullable** **id**)value forUndefinedKey:(NSString *)key;

仍然是Key一定不能为空。

NSDictionary方法：

- - (**nullable** ObjectType)objectForKey:

NSMutableDictionary方法：

- - (**void**)setObject:(ObjectType)anObject forKey:(KeyType <NSCopying>)aKey

**这里是不能为空的**

所以在track的时候，我们要尽量使用下标语法而不是setObjectForKey或者@{}的语法糖。使用KVC也行，但是比较奇怪。

- NSSet 

- NSIndexPath

- NSCache

- NSCache是类似NSDictionary的集合类，是线程安全的。

- \- setObject:forKey: // 0 cost
- \- setObject:forKey:cost:

跟Dictionary是一样的，setObject不能为空，想移除的话必须使用remove方法

Strings and text         NS_ASSUME_NONNULL_BEGIN/END

- NSString 

- 主要是range和index问题

Numbers, Data, and Basic Values

- NSValue 

- [NSValue](https://nshipster.cn/nsvalue/) 是用于承载单一的 C 或 Objective-C 数据值的容器。它能承载标量和值类型，也能用于指针和对象 ID。

- NSURL

- 比较常见的出现crash的类，大部分是NSString转NSURL出现的问题，五花八门，日常要多注意要么用safe的构造器，要么自行判空上报。

- 同NSArray一样，**整个NSURL.h都处于NS_ASSUME_NONNULL_BEGIN/END的包裹下**，对待URL要像Array一样小心。

- Methods cluster which use filePath string to initialize a filePath URL

- - [- initFileURLWithPath:isDirectory:relativeToURL:](http://apple-reference-documentation//hcDF1bF8dC)
  - ...

- Methods cluster which use string to initialize a URL

- - [- initWithString:](http://apple-reference-documentation//hcR_ndXcp-)
  - [+ URLWithString:](http://apple-reference-documentation//hcC8WWVdqy)
  - Relative param version...

- NSData

- NSRange

Dates and Times

- NSDate

- NSDateFormatter

- NSDateComponent

Archive and serialization

- NSCoding

- NSJSONSerialization

Syntax Sugar

- NSDictionary

@{}是[NSDictionary dictionaryWithObjects:forKeys:]的语法糖，两个参数均不允许nil。


- NSArray

@[]是[NSArray arrayWithObjects:...]的语法糖，参考NSArray，不允许nil。



- NSString

@""是NSString的语法糖，因为@"xxx"其实在compile time就已经被替换为了constant value，但仍然不是objective-c对象，运行时从constant value转换为objc的NSString对象。

> Objective-C provides a shorthand notation for creating NSString objects from constant values.

- NSNumber

@#是NSNumber的语法糖，使用多种构造器构造NSNumber，支持使用后缀进一步指定类型：@42u。

- 动态评估

@()：基于表达式的值返回合适的对象常量（const char *返回NSString，int返回NSNumber等等），这也是数字常量和枚举值的指定方式。

下标语法

首先下标语法的本质是填充了这两个方法，并不需要遵守协议，由Clang来保证。

```
- (id)objectForKeyedSubscript:(id <NSCopying>)key;//自定义下标key的类型，如果只是希望使用NSUinteger，则可以把id <NSCopying>换成NSInteger
- (void)setObject:(id)obj forKeyedSubscript:(id <NSCopying>)key;
```

[下表语法真正强大的是可以用来定义DSL。](https://nshipster.cn/object-subscripting/)但是应该谨慎的判断DSL这种特立独行的东西是否行得通。

NSArray

只接受数字下标，只存在越界问题，不存在防空问题。

NSDictionary

来看下NSDictionary对上述两个函数的声明

```
- (**nullable** ObjectType)objectForKeyedSubscript:(KeyType)key;
- (**void**)setObject:(**nullable** ObjectType)obj forKeyedSubscript:(KeyType <NSCopying>)key;
```

所以对于NSDictionary我们**使用下标语法是可以set nil进去的，****但是Key永远不能为空****。需要对key防空。**

需要注意的是，**下标语法跟KVC是两码事。**

使用自定义类的下标语法的时候一定要在interface文件中声明做好是否可以为空的声明。