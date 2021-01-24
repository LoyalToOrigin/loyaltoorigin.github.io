---
title: Objective-C 快速消息转发机制在项目中的使用
date: 2020-01-24
categories: iOS
---


OC 消息转发机制, 想必大家并不陌生, 作为 iOS 开发, 作为面试必问, 即使不是深入了解, 也肯定有所耳闻。网上介绍消息转发机制的文章一大堆, 但说到具体应用场景的寥寥无几, 有的说到消息转发的应用, ***- (void)forwardInvocation:(NSInvocation *)anInvocation*** 方法中调用另一个对象执行对应方法, 快速转发能实现的场景却用标准消息转发来实现, 未免有点大材小用。


# 简述 OC 消息转发机制

当 OC 对象调用未实现的方法时,  通过消息转发机制可以使对象执行用户预先定义的处理过程, 会按下图流程依次调用方法。

![forward_process.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/forward_process.png)

从上图可以看出, 当对象无法响应方法时, 会依次调用这4个方法。消息转发有快速转发和标准（完整）转发两类, 快速转发即实现 ***- (id)forwardingTargetForSelector:(SEL)aSelector*** 方法,  快速消息转发仅需要返回一个可执行的对应方法的对象, 走完下面两个方法即处理对应方法, 本文主要介绍的就是快速转发在项目的应用。

```
//1.当一个对象调用未实现的方法时,会调用该方法,通过该方法决定是否动态添加方法
+ (BOOL)resolveInstanceMethod:(SEL)sel {
//     NSLog(@"%s", __func__);
//    if (sel == @selector(run)) {

//        class_addMethod(self, sel, (IMP)funcRun, "v@:@@");
//        return YES;
//        //YES表示执行该方法 , 不进行下一步
//        //NO表示不执行该方法

// ❤返回值没什么意义, 只能说是一种规范, 源码中并没有拿返回值做相关操作, 如果动态添加了方法,
// 无论 YES or NO, 都不会到下一步, 因为动态方法解析后会再去找这个方法, 找到直接跳转执行函数, msg_send方法结束, 
// 只有当 不动态添加方法,  才会进下一步
//    }

    //返回NO进入到下一步
    return NO;
    return [super resolveInstanceMethod:sel];
}


//2.这个方法用于指定备用对象相应这个selector,不能指定为self.
//  如果返回某个对象,会调用对象的对应方法[dog run],如果该对象没实现该方法,会报错;返回nil则进入下一步

- (id)forwardingTargetForSelector:(SEL)aSelector {
	return [Dog new];
}
```


# 快速消息转发在项目中的使用

### 自定义容器 view  转发给自己的 subview

在日常开发中, 我们往往会遇到一个场景, 一个同样的自定义 view 用于 cell 中和不用于 cell 中, 这时候我们会自定义一个 view 暴露如下图所示接口, 供外界调用。

![view_interface.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/view_interface.png)

当我们使用这个 view 时候, 直接使用就可以了。但当需要该 view 嵌到 cell 中使用时, 还需要创建一个 cell 容器类, 将这个 view 添加进去, 而这个 cell 类其实仅仅作为添加 view 的容器, 本身并不具备什么逻辑, 当外界调用 cell 时, 它应该和 view 保持一样的接口如下图所示:

![cell_interface.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/cell_interface.png)

传统的写法, 在 ***getter*** 、***setter*** 方法实现中, 调用添加在自己中的 view 的对应方法即可, 如图所示:

![cell_common_implementation.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/cell_common_implementation.png)

但这里如果使用快速消息转发,  会是代码更加简洁, 我们仅需要在 ***- (id)forwardingTargetForSelector:(SEL)aSelector*** 方法中 return 加在自己中的自定义 view 对象即可, 当然这里要配合 ***@dynamic*** 关键字使用。因为 ***@property*** 关键字会生成对应的 ***getter*** 、***setter*** 方法, 这样的话 cell 相当于实现了 ***getter*** 、***setter*** 方法, 自然不会走到 ***- (id)forwardingTargetForSelector:(SEL)aSelector*** 中。

而 ***@dynamic*** 关键字意味着编译器不会帮助我们自动生成 ***getter*** 、***setter*** 方法。这样一来, 相当于我们只申明了这些方法而并没有实现这些方法, 当调用时便会遵循消息转发机制的步骤, 从而调用 ***- (id)forwardingTargetForSelector:(SEL)aSelector***。

![cell_forward_implementation.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/cell_forward_implementation.png)


可以对比一下, 代码简洁了很多, 如果自定义 view 对外暴露的属性及方法更多, 这么写省去的代码就越多。后续如果需要增加对外暴露接口, 只需要在 .h 文件中添加对应接口, 如果是属性, .m 文件 ***@dynamic*** 关键字也要添加对应属性。


### 数据模型数据结构不一致

曾经在项目中做直播间道具相关业务, 第一次使用了快速消息转发。由于第二期添加了背包道具, 就是下图中的 ***SPTBagPropInfo*** 类, 相比于原来的道具, 背包道具是通过做任务等赠送的, 所以多了个数量字段, 作为我们前端来说, 理想的数据模型即和原道具数据结构一致, 只是增加一个数量字段即可, 但是后端同学给到的数据结构却是在外层加了数量这个字段, 对于我们来说, 相当于一个新对象, 这样对我们来说用起来是不方便的。

![model_interface.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/model_interface.png)

所以在 .m 文件中, 直接转发给 self.detail, 即转发给原来的道具对象, 仅需自己实现原来道具没有的方法即可。

![model_implementation.png](https://blog-1258097834.cos.ap-shanghai.myqcloud.com/Objective-C%20%E5%BF%AB%E9%80%9F%E6%B6%88%E6%81%AF%E8%BD%AC%E5%8F%91%E6%9C%BA%E5%88%B6%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/model_implementation.png)

可以看出, 这里实现接口对外统一的作用只要还是靠协议, 快速转发和上一个案例其实本质是一致的, 直接转发给了自己内部的一个对象, 使 .m 文件中省去了一些冗余代码。

# 总结

通过项目中的一些实际场景, 使用到消息转发机制, 能够更加深刻了解相关知识点。本文主要是自己的一点思考和尝试, 如果有说的不对的地方, 望不吝赐教。