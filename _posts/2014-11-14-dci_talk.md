---
layout: default
title: 浅谈DCI
---

## 前言

DCI框架是由James O. Coplien和Trygve Reenskau在[《The DCI Architecture: A New Vision of Object-Oriented Programming》](http://www.artima.com/articles/dci_vision.html)提出的新架构方法。在[百度文库](http://wenku.baidu.com)有中文翻译版[《DCI架构_面向对象编程的新构想_上_》](http://wenku.baidu.com/link?url=LR5nhgBVxA_ELpf96zGkDUYgZpbz1_LNXvFgJk54YU_2WHsIFHR1bPDmZ6TvnFduR0FtZIeKljdqksooGoLGV6M0Up7RfBzPqPUPMmDS0N_)和[《DCI架构_面向对象编程的新构想_下_》](http://wenku.baidu.com/link?url=LR5nhgBVxA_ELpf96zGkDUYgZpbz1_LNXvFgJk54YU_2WHsIFHR1bPDmZ6TvnFdurOcNj7zIGoTQpMA1FN66srdyg_hjQuPrrqCUt20Co2O)

可以关注Google的经常讨论DCI的论坛[object-composition](https://groups.google.com/forum/#!forum/object-composition)

## DCI介绍

DCI分别代表Data，Context，Interaction
DCI框架指的是在一个特定的上下文（Context）中，让指定的数据（Data）扮演指定的角色（Role）来进行交互（Interaction）
Data代表的是纯数据，只包含自省的方法，不包含交互的方法。
Role代表的是在特定Context下的一个角色，包含了在这个Context下的交互方法
Context代表的是一个特定的上下文环境

DCI架构的特别之处在于引入了Role，传统的OO设计中，Class的定义包含了数据和方法，方法里也包含了交互方法，也就是一些需要多方交互的方法。如一个Character有攻击方法，攻击需要多方协调，把攻击这类交互方法放在Character中，导致Character内聚降低。DCI引入了Role，在上例中，在攻击这个Context下，有AttackRole和DefendRole 2个Role，Context负责把AttackRole注入到Character中，让Character在攻击Context下变成了一个AttackRole来进行交互。这样，Character只包含基本数据和自省函数，而特定的交互函数则放在了特定的Role中。这样有个很大的优点，由于Context的代码很集中，内聚非常高，可读性很强，而Role的注入也提高了可扩展性，也就是Data对修改封闭，而Role对扩展开放，符合开闭原则。

实际上，为了实现上述的解耦框架，并不一定需要DCI，这里要提出DCI的另一个特别之处，也就是心智模型（metal model），DCI关注的点与传统OO不同，DCI关注的是用户的心智模型与最终代码（也可以说程序猿）的心智模型的对应，这种对应减少了映射，加强了可读性，更加接近自然语言表达。所以在不同语言上实现DCI，并一定会让结构清晰，甚至会让代码看起来更难懂，但这很大程度是因为现在的语言缺少对交互的表达方法，而很多语言实现注入会比较难看，使得代码的整体架构和可读性降低。

我自己编写了一个例子来理解DCI的

```lua
    local function character(o)
        if o == nil then
            return { atk = 10, def = 5, hp = 100, inject = function(self, role) setmetatable(self, role) end }
        else
            o.inject = function(self, role) setmetatable(self, role) end
            return o
        end
    end

    AttackRole = {
        
    }

    DefendRole = {
        isHit = function(self, attackRole)
            if attackRole.agi >= self.agi then
                return true
            else
                return false
            end
        end
    }

    local function inject(data, role)
        setmetatable(data, role)
    end

    AttackContext = {
        execute = function(a, b)
            a:inject(AttackRole)
            b:inject(DefendRole)

            if b:isHit(a) then

            end
        end
    }

    local John = character { atk = 20, def = 15, hp = 100, agi = 10, }
    local Mike = character { atk = 10, def = 35, hp = 150, agi = 17, }

    AttackContext.execute(John, Mike)
```

结束