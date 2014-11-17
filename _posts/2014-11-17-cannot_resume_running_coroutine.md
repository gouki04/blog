---
layout: default
title: 解决cannot resume running coroutine问题
---

## 问题

最近在搞lua下的coroutine，打算做一个方便编写异步流程的库，一切都正常，不管是等待时间，等待网络都ok，但偏偏在等待控件点击时出现了cannot resume running coroutine错误。

伪代码如下：

```lua
    function WaitForMessageBox(msg)
        MessageBox:Show(msg, function() coroutine.resume(coObj) end)
        coroutine.yield()
    end
```

## 排查

反复验证代码，协程确实已经挂起了，但奇怪的是在回调时协程的状态却真的是运行中，导致无法resume。因为在其他情况下都正常，所以分析在等待时间时和等待点击时的区别。发现2者记录回调函数的方式不一样，等待时间时使用的toluafix_ref_function而等待点击时使用的是luaL_ref。

* toluafix_ref_function
   这方法是cocos2dx里提供的，基本原理是在registry表建立一个toluafix_refid_function_mapping表，来保存所有的注册函数，并把其对应的下标返回作为方法在c层的id。

   伪代码：

```lua
   refid = refid + 1 -- func_id一直递增，保证不重复
   registry.toluafix_refid_function_mapping[refid] = func
   return refid
```

* luaL_ref
   这个实现是别人从其他库里抄过来的，所以一直没研究。经过研究后，发现原理也非常简单。
   先把func放在栈顶，然后使用luaL_ref将其放入registry表，并将返回的id记录下来作为方法在c层的id。

通过注册时的区别没法发现问题。然后检查调用的代码。发现2者也是不同的实现。等待时间的情况下是通过CCLuaStack::executeFunction来调用的，而等待点击时是直接调用的lua_pcall。再仔细分析代码，发现2者有个重大的区别。CCLuaStack::executeFunction调用lua_pcall时使用的是全局的lua_State，而等待点击时回调用的是注册时保存的lua_State！

由于在等待点击的情况下是在协程里注册的回调，所以保存下来的是协程的lua_State，所以真正回调时，函数又回到协程的环境下，而这时候协程就是在运行中！所以无法自己resume自己，导致报错。而其他情况，由于使用的全局的lua_State，而这个lua_State是主线程（我也不确定到底叫什么线程好），所以没有这个问题！

## 解决



结束
