---
title: Decimaljs在class-transformer中异常的解决方案
toc: true
categories:
  - 文章分类
tags:
  - 文章标签
date: 2019-12-17 03:07:09
---

## 背景

在使用[`class-transformer`](https://github.com/typestack/class-transformer)进行对象转换并且遇到类型为Decimal的数据时工作不正常

## 异常复现

我们有以下类型，是目标对象：
```typescript
export class OrderItemTransfer {
    ///忽略其他无关内容

    @Expose()
    price: Decimal;

}
```
以及，源对象定义：
```typescript
@Entity()
export class OrderItem {
    ///忽略其他无关内容

    @Column({
        type: "decimal",
        precision: 5, scale: 2, default: 0,
        transformer: new DecimalTransformer()
    })
    price: Decimal;

}
```
我们的目标是将 `OrderItem` 类型的对象转换到 `OrderItemTransfer` 类型。于是有以下代码：
```typescript
let transfer = plainToClass(OrderItemTransfer, entity);
//发生异常: Error
//Error: [DecimalError] Invalid argument: undefined
```

## 异常分析

通过追踪调用栈发现如下信息:

![stack](stack.png)

找到调用`Decimal`构造函数的具体代码

![caller](caller.png)

可以看到，这里直接调用了`Decimal`构造函数，然而Decimal的构造函数并不支持参数是`null`/`undefined`从而引发了异常。

`plainToClass`这个方法的设计是针对`plain object`到`class`的转换，通过下图可见，他的类型判断相对简单

![type-check](type-check.png)

在判断对象类型是`object`之后并没有进一步判断构造函数相关信息（针对`plain object`的设计并不需要判断，因为这种情况并不存在自定义的构造函数）。但是这种特性恰巧妨碍了我们使用类似于`Decimal`这种没有默认构造函数的类型。

## 解决方案

其实2017年已经有人提过[这个问题](https://github.com/typestack/class-transformer/issues/92)，但是这种情况确实是此方法的设计情况之外，而且官方也并没有给出解决方案。仅有某位用户提供的一个临时性的解决方案:

```javascript
else if (value[valueKey] instanceof Function) {
    subValue = value[valueKey](); --> Comment out this line
}
```

这个解决方案真的能解决问题么？答案是：可以，但不优雅。

与其提交pr给`class-transformer`不如从另外一个角度来解决，那就是从`Decimal.js`下手。由于问题发生在Decimal类型没有默认的构造函数，那么我们为何不拓展以下这个类型呢？

```typescript
class DecimalPatch {
    constructor(v) {
        if (!v) v = 0;
        let d = new Decimal(v) as any;
        delete d.constructor;
        Object.assign(this, d);
    }
}
```

看到这里，你可能要问几个问题：

为什么不用`extends`呢？答案是`class-transformer`拿到的类型仍是Decimal的构造函数

为什么要`delete d.constructor`呢？答案是防止`plainToClass`再次调用`constructor`

(这个方法很迷，它会调用源对象的所有方法求值赋值给目标对象，看似是为了调用`getter`作用的函数，但是并没有判断能力，就连构造函数也不放过QAQ。。。)

现在，你可以这样使用它

```typescript
@Entity()
export class OrderItem {
    ///忽略其他无关内容

    @Column({
        type: "decimal",
        precision: 5, scale: 2, default: 0,
        transformer: new DecimalTransformer() //还记得这里么，
        //解释一下，DecimalTransformer是用于将数据库查询出的数据转换为自定义数据
    })
    price: Decimal;

}
```

它长这个样子

```typescript
export class DecimalTransformer implements ValueTransformer {
    to(data: Decimal): string {
        return data.toString();
    }
    from(data: string): Decimal {
        return new Decimal(data); //这里实力化了对象
    }
}
```
那么我们可以修改上述代码：
```typescript
from(data: string): DecimalPatch {
    return new DecimalPatch(data);
}
```
这样，`entity`中的数据实际类型是`DecimalPatch`，并且工作一切正常

