---
layout: default
title: 解决cannot resume running coroutine问题
---

## 问题

最近在搞lua下的coroutine，打算做一个方便编写异步流程的库，一切都正常，不管是等待时间，等待网络都ok，但偏偏在等待控件点击时出现了“cannot resume running coroutine”错误。

伪代码如下：

```lua
    function WaitForMessageBox(msg)
        MessageBox:Show(msg, function() coroutine.resume(coObj) end)
        coroutine.yield()
    end
```

## 排查

反复验证代码，协程确实已经挂起了，但奇怪的是在回调时协程的状态却真的是运行中，导致无法resume。因为在其他情况下都正常，所以分析在等待时间时和等待点击时的区别。发现2者记录回调函数的方式不一样，等待时间时使用的`toluafix_ref_function`而等待点击时使用的是`luaL_ref`。

* `toluafix_ref_function`
   这方法是cocos2dx里提供的，基本原理是在registry表建立一个`toluafix_refid_function_mapping`表，来保存所有的注册函数，并把其对应的下标返回作为方法在c层的id。

   伪代码：

```lua
   refid = refid + 1 -- func_id一直递增，保证不重复
   registry.toluafix_refid_function_mapping[refid] = func
   return refid
```

* `luaL_ref`
   这个实现是别人从其他库里抄过来的，所以一直没研究。经过研究后，发现原理也非常简单。
   先把func放在栈顶，然后使用`luaL_ref`将其放入registry表，并将返回的id记录下来作为方法在c层的id。

   伪代码：

```c
    int refid = luaL_ref(L, LUA_REGISTRYINDEX);
    return refid
```

通过注册时的区别没法发现问题。在网上搜索到有人出现了一样的问题[点这里](http://lua.2524044.n2.nabble.com/Coroutine-trouble-td7640996.html)，是因为记录回调函数时把当时的lua_State保存在一起了，而由于是在协程内记录函数的，所以**记录的`lua_State`是协程的环境**！而在回调时直接用了保存的`lua_State`，从而直接到了协程的环境下去执行了，而这时协程就是运行状态，所以无法resume！

检查本地的代码，发现和上文里是一模一样的情况，也是因为直接使用了当时保存的`lua_State`，导致回调时使用了协程的环境，导致无法resume。

## 解决

1. 直接把调用时的`lua_State`改为使用主`lua_State`，这样回调时就不在协程环境下，可以正常resume了。

2. 如果无法直接改动c层的代码，可以在想办法在lua下绕过这个问题：
把记录回调函数放在协程外

```lua
    function WaitForMessageBox(msg)
        -- 因为cocos2dx的Scheduler调用回调时使用的是主`lua_State`，所以回调时已经不在协程内
        Scheduler:WaitOneFrame(function()
                MessageBox:Show(msg, function() coroutine.resume(coObj) end)
            end)
        
        coroutine.yield()
    end
```

结束
