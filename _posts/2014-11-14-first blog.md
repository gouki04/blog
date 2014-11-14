---
layout: post
title: tolua下的new，new_local与takeownership，releaseownership
---

tolua下创建实例有2个方法：

* 使用new创建的实例，生命周期并不受tolua管理，相当于在c++调用new一样，需要用户自己释放内存
* 使用new_local创建的实例，生命周期被tolua管理，当实例在lua下没人引用它时，就会被lua回收掉。

那么问题来了，

如果使用new_local创建的实例，在c++层还被人使用（也就是还不应该析构时），在lua层却没人引用，就会导致实例被lua释放了，c++层的实例就变成野指针了。而如果使用new创建，但却没有合适的析构接口来释放实例，就会导致内存泄露。

这时就可以使用`tolua.takeownership`和`tolua.releaseownership`

* `tolua.takeownership(obj)`表示tolua会管理obj，也就是会被lua gc影响的
* `tolua.releaseownership(obj)`表示tolua不再管理obj，这时obj不受lua回收影响

一个经典的例子就是异步调用：

    local obj = BaseClass:new_local()
    obj:asyncDoSomething(function()
        end)

上述代码可能在`asyncDoSomething`的过程中，obj被回收了

可以改成这样：

    local obj = BaseClass:new_local()
    obj:asyncDoSomething(function()
            tolua.takeownership(obj)
        end)

    tolua.releaseownership(obj)

先把obj的所有权释放掉，等obj真正用完时再调用`takeownership`获取所有权即可。

或者更简单的：

    local obj = BaseClass:new()
    obj:asyncDoSomething(function()
            tolua.takeownership(obj)
        end)

结束