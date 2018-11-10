# XLua的obj和userdata的引用分析

为了防止c#和lua两端的内存泄漏，有必要了解xlua是怎样处理2端的引用关系的，尤其是在扩展xlua时，处理不得当很容易造成引用丢失或者内存泄漏。

一个c#的obj是不能直接传递到lua，需要一个中间层，这个中间层就是userdata。xlua会为每个传递到lua的obj生成唯一的一个userdata，并将2者绑定起来（具体绑定方式后面分析）。

这里需要处理2个基本问题：

1. 两端查询
    - 可以通过obj查找到其对应的ud（一般是在将obj压到lua时，要先查询其有没有对应的ud，保证同一个obj对应同一个ud）
    - 可以通过ud查找到其对应的obj（在wrap文件中大量使用，lua下通过ud调用其c#层的函数，那c#层就需要通过这个ud找回其obj，完成函数调用）
2. GC管理
    - obj在c#层被GC了，其对应的ud也应该赋nil。
    - ud在lua层被GC了，清理其对应的obj的引用。

在xlua中，基本是通过2个容器来管理obj和ud，在ObjectTranslator中：

```csharp
// ObjectTranslator.cs
public partial class ObjectTranslator
{
    internal readonly ObjectPool objects = new ObjectPool();
  
    internal readonly Dictionary<object, int> reverseMap = new Dictionary<object, int>(new ReferenceEqualsComparer());

    // ...
}
```

其中objects是ud到obj，reverseMap是obj到ud。（PS：ObjectPool采用了一种很有意思的数据结构来保存ud和obj，后续会在其他文章中分析，这里可以认为其就是一个`Dictionary<int, object>`）。

如果一个obj第一次传递到lua，会调用addObject函数来设置这2个容器，参考下面的代码：

```csharp
// ObjectTranslator.cs
int addObject(object obj, bool is_valuetype, bool is_enum)
{
    int index = objects.Add(obj);
    if (is_enum)
    {
        enumMap[obj] = index;
    }
    else if (!is_valuetype)
    {
        reverseMap[obj] = index;
    }

    return index;
}
```

其中objects.Add返回的index，可以先理解为objects可以通过这个index查询到这个obj。

此时，已经可以解决第1个查询问题，c#层可以通过reverseMap查询obj对应的ud了。可以参考xlua的Push函数，这个函数负责把一个obj传递给lua。

```csharp
// ObjectTranslator.cs
public void Push(RealStatePtr L, object o)
{
    if (o == null)
    {
        LuaAPI.lua_pushnil(L);
        return;
    }

    int index = -1;
    Type type = o.GetType();
    bool is_enum = type.IsEnum;
    bool is_valuetype = type.IsValueType;
    bool needcache = !is_valuetype || is_enum;
    if (needcache && (is_enum ? enumMap.TryGetValue(o, out index) : reverseMap.TryGetValue(o, out index)))
    {
        if (LuaAPI.xlua_tryget_cachedud(L, index, cacheRef) == 1)
        {
            return;
        }
    }

    bool is_first;
    int type_id = getTypeId(L, type, out is_first);

    //如果一个type的定义含本身静态readonly实例时，getTypeId会push一个实例，这时候应该用这个实例
    if (is_first && needcache && (is_enum ? enumMap.TryGetValue(o, out index) : reverseMap.TryGetValue(o, out index)))
    {
        if (LuaAPI.xlua_tryget_cachedud(L, index, cacheRef) == 1)
        {
            return;
        }
    }

    index = addObject(o, is_valuetype, is_enum);
    LuaAPI.xlua_pushcsobj(L, index, type_id, needcache, cacheRef);
}
```

可以看到，函数内部会先通过reverseMap查找其ud，如果有的话，就通过`LuaAPI.xlua_tryget_cachedud`将其ud传给lua。如果没有，则调用`addObject`和`LuaAPI.xlua_pushcsobj`来生成一个新的ud。

同时也可以解决第2个查询问题了，通过objects可以查询ud对应的obj，随便查看一个wrap文件，有如下类似代码：

```csharp
// wrap文件，此时ud在栈的位置是1
object __cl_gen_to_be_invoked = translator.FastGetCSObj(L, 1);



// ObjectTranslator文件
internal object FastGetCSObj(RealStatePtr L,int index)
{
    return getCsObj(L, index, LuaAPI.xlua_tocsobj_fast(L,index));
}


private object getCsObj(RealStatePtr L, int index, int udata)
{
    object obj;
    if (udata == -1)
    {
        if (LuaAPI.lua_type(L, index) != LuaTypes.LUA_TUSERDATA) return null;

        Type type = GetTypeOf(L, index);
        if (type == typeof(decimal))
        {
            decimal v;
            Get(L, index, out v);
            return v;
        }
        GetCSObject get;
        if (type != null && custom_get_funcs.TryGetValue(type, out get))
        {
            return get(L, index);
        }
        else
        {
            return null;
        }
    }
    else if (objects.TryGetValue(udata, out obj)) // 这里通过udata查找obj并返回
    {
#if !UNITY_5 && !XLUA_GENERAL
        if (obj != null && obj is UnityEngine.Object && ((obj as UnityEngine.Object) == null))
        {
            throw new UnityEngine.MissingReferenceException("The object of type '"+ obj.GetType().Name +"' has been destroyed but you are still trying to access it.");
        }
#endif
        return obj;
    }
    return null;
}
```

其实可以发现，上面的代码在每次查询时，都需要通过一次转换，例如：`LuaAPI.xlua_tryget_cachedud`、`LuaAPI.xlua_tocsobj_fast`等调用。这是因为objects和reverseMap里保存的并不是真正的ud，而是ud的key，xlua可以通过这个key找到真正的ud。

为什么要这么麻烦，再加一个中间层呢？因为lua下变量都是不能直接在c#里保存的，一般会采用lua_ref来生成一个引用值，lua会通过这个引用值来找到对应的变量。不过xlua不是采用这种方法，我估计主要有几个方面的考虑：

- 将ud的引用保存在一个独立的表中，方便管理。
- 保存ud引用的表可以设置为弱表，减少引用管理。
- 采用自定义的key，方便objects的数据结构的增删改操作。

具体实现方法如下：

首先translator会构建一个lua的table，也就是上面经常出现的cacheRef，注意它是一张弱表（值是弱引用，可以查看其构建过程），这张表的键就是objects和reverseMap保存的整数，值就是ud。

这样c#层通过reverseMap查询到obj对应的key，`LuaAPI.xlua_tryget_cachedud`再通过这个key找到对应的ud，即完成查询过程。

但反过来，通过ud怎么找回其key呢？难道要再建一张table来保存ud到key的关系吗？xlua采用了一个比较好的解决方案，即把key直接保存在userdata中。因为在lua下，userdata其实就是一个内存块，参考xlua的代码：

```c
// xlua.c
static void cacheud(lua_State *L, int key, int cache_ref) {
    lua_rawgeti(L, LUA_REGISTRYINDEX, cache_ref);
    lua_pushvalue(L, -2);
    lua_rawseti(L, -2, key);
    lua_pop(L, 1);
}


LUA_API void xlua_pushcsobj(lua_State *L, int key, int meta_ref, int need_cache, int cache_ref) {
    int* pointer = (int*)lua_newuserdata(L, sizeof(int));
    *pointer = key;

    if (need_cache) cacheud(L, key, cache_ref);

    lua_rawgeti(L, LUA_REGISTRYINDEX, meta_ref);

    lua_setmetatable(L, -2);

    #if LUA_VERSION_NUM == 501
    lua_pushvalue(L, XLUA_NOPEER);
    lua_setfenv(L, -2);
    #endif
}
```

可以看到，lua生成的userdata的大小是一个int，其内容就是key的值。同时也可以通过cacheud函数看到，这个key会作为键值将ud记录在cacheRef中。

这样lua层通过ud的值获取到key（通过`xlua_tocsobj_fast`或者`xlua_tocsobj_safe`函数），通过key就可以查询objects里的obj了。

下面来分析下GC过程。

首先看下ud的引用。注意到cacheRef是没有造成引用的，因为它是一张弱表，同时c#层也只记录了ud的key，也没有直接引用ud，所以ud被没有任何的额外引用。也就是说，**一个ud传递给lua后，如果在lua下没有地方引用这个ud了，那么它就会被GC了。这一点非常重要，因为如果需要一个ud的生命周期要跟随obj的生命周期的话，obj就必须要自己对这个ud生成一个引用，否则这个ud就有可以因为在lua下没人引用它而导致被GC**。举个例子就是[Peer机制](2018-11-10-xlua_peer.md)，由于peer是绑在ud上的，如果ud被GC了，其peer也会丢失，这样下次obj被传递到lua时，就会生成一个新的ud，从而丢失了其peer内容。

ud的GC处理是通过`__gc`这个metamethod实现的（lua5.1也只支持ud的`__gc`，table都不支持），xlua会把所有ud的metatable都设置一个统一的`__gc`函数，即`StaticLuaCallbacks的GcMeta`，看下面的代码：

```csharp
// StaticLuaCallbacks.cs
[MonoPInvokeCallback(typeof(LuaCSFunction))]
public static int LuaGC(RealStatePtr L)
{
    try
    {
        int udata = LuaAPI.xlua_tocsobj_safe(L, 1);
        if (udata != -1)
        {
            ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
            translator.collectObject(udata);
        }
        return 0;
    }
    catch (Exception e)
    {
        return LuaAPI.luaL_error(L, "c# exception in LuaGC:" + e);
    }
}

// ObjectTranslator.cs
internal void collectObject(int obj_index_to_collect)
{
    object o;

    if (objects.TryGetValue(obj_index_to_collect, out o))
    {
        objects.Remove(obj_index_to_collect);

        if (o != null)
        {
            int obj_index;
            //lua gc是先把weak table移除后再调用__gc，这期间同一个对象可能再次push到lua，关联到新的index
            bool is_enum = o.GetType().IsEnum;
            if ((is_enum ? enumMap.TryGetValue(o, out obj_index) : reverseMap.TryGetValue(o, out obj_index))
                && obj_index == obj_index_to_collect)
            {
                if (is_enum)
                {
                    enumMap.Remove(o);
                }
                else
                {
                    reverseMap.Remove(o);
                }
            }
        }
    }
}
```

可以看到，在ud被gc时，首先获取其key（也就是LuaGC函数的udata变量），然后collectObject函数，根据key来清理objects和reverseMap容器，完成清理过程。（**注意cacheRef不需要清理，因为它是弱表！**）

最后来看obj的引用。首先注意到objects和reveresMap都会对obj产生引用，也就是说这个obj就算c#层其他地方都不引用它了，它也不会被GC，直到lua层也没人引用obj对应的ud了，ud被GC导致objects和reverseMap被清理，obj才能被GC。所以，**只要lua层还有地方引用这个obj，那么这个obj就不会被GC，这是很重要的一点，为了防止内存泄漏，lua层的引用必须管理好，否则会导致c#层的GC**。

不过有一种情况，就算lua层还存在引用，obj仍然会被GC，这种情况就是这个obj是UnityEngine.Object派生的！由于可以通过`GameObject.Destroy`等函数显式删除一个unity obj，此时c#层所有引用这个unity obj的变量的值都会变成null（我也不知道unity怎么做到的。。）。为了处理这种情况，xlua采用了一个GC轮询，当发现一个unity obj被析构后，会对其ud做一些清理。看下面的代码：

```csharp
// LuaEnv.cs
#if !XLUA_GENERAL
int last_check_point = 0;

int max_check_per_tick = 20;

static bool ObjectValidCheck(object obj)
{
    return (!(obj is UnityEngine.Object)) ||  ((obj as UnityEngine.Object) != null);
}

Func<object, bool> object_valid_checker = new Func<object, bool>(ObjectValidCheck);
#endif

public void Tick()
{
#if THREAD_SAFT || HOTFIX_ENABLE
    lock (luaEnvLock)
    {
#endif
        var _L = L;
        lock (refQueue)
        {
            while (refQueue.Count > 0)
            {
                GCAction gca = refQueue.Dequeue();
                translator.ReleaseLuaBase(_L, gca.Reference, gca.IsDelegate);
            }
        }
#if !XLUA_GENERAL
        last_check_point = translator.objects.Check(last_check_point, max_check_per_tick, object_valid_checker, translator.reverseMap);
#endif
#if THREAD_SAFT || HOTFIX_ENABLE
    }
#endif
}

//兼容API
public void GC()
{
    Tick();
}
```

可以看到，在Tick函数中，会把`ObjectValidCheck`函数传给objects，Check函数内部会遍历指定数量的obj，通过`ObjectValidCheck`判断其是否已经失效了，如果失效的话，对应的obj会被赋null。这样外部通过ud查询到的obj就会是null。

要注意到，ud自身不是nil，只是其对应的obj是null，所以在lua下通过判断变量是否为nil是不能发现unity obj被删除了，所以xlua提供了一个简单函数来判断一个unity obj对应的ud是否失效了。

```lua
--unity 对象判断为空, 如果你有些对象是在c#删掉了，lua 不知道
--判断这种对象为空时可以用下面这个函数。
function IsNil(uobj)
    return uobj == nil or uobj:Equals(nil)
end
```

最后的最后，分析一个容易出现的循环引用问题：

上面提到，如果obj需要保证ud的生命周期，会存储一个ud的引用，那这个引用应该什么时候释放呢？xlua里的LuaBase类也负责了lua引用的管理，它是采用Dispose和析构函数的方式来释放lua引用的。那我们也可以为obj实现Dispose和析构函数来释放引用吗？答案是不行的，这样会造成循环引用。由于obj自身被objects和reverseMap引用着，这个引用必须等到ud被GC才会释放，而ud却又被obj引用了，从而产生循环引用，所以obj根本不可能触发到析构函数，也无法释放ud引用，最后造成内存泄漏。

怎么解决呢？可以通过提供一个统一的Destroy函数来做释放。例如轩辕的BaseObject就有一个Destroy函数，所有BaseObject不用时都需要调用这个函数来做清理工作，调用这个函数后，就可以认为其已经不能再使用了，这样就可以在这个Destroy函数里清理ud的引用。