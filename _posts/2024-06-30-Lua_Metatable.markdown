---
layout:         post
title:          "Lua metatable & metamethod"
author:         "Bilthas"
header-img:     "img/post-bg-2015.jpg"
catalog:        true
tags:
    - Lua
---

表是Lua中最重要的数据结构，而metatable和metamethod又是表中的关键，lua可以通过元表来定义一些表的独特操作，像比较基础的定义表的算数运算和C++中的重定义运算符比较像。正因为元表的存在，才让lua有能力实现一些更加复杂的数据结构和模拟面向对象。本篇记录一些相关知识，对于基础部分简单过一下，主要记录后面一些思考。

# metatable & metamethod

元表让我们能够为某些类型的值添加一些未曾有过的操作，比如定义一个a + b，a和b都是表。一般情况下肯定是加不了的。而此时lua会去寻找a or b中有没有元表，如果发现了存在元表并且元表中存在key：\_\_add，那么会使用\_\_add对应的value来完成a + b的操作。

获取某个变量的元表：getmetatable(a)  
为某个变量设置元表：setmetatable(a, _mt) _mt就是元表

假设前面的a和b代表的是集合，比如：

```lua
local Set = {}
local mt = {}
function Set.new (l)
    local set = {}
    setmetatable(set, mt) -- 将mt设置为元表
    for _, v in ipairs(l) do set[v] = true end
    return set
end

function Set.union (a, b)
    local res = Set.new{}
    for k in pairs(a) do res[k] = true end
    for k in pairs(b) do res[k] = true end
    return res
end

return Set
```

那么为了实现相加的操作，可以为元表mt添加metamethod：

```lua
mt.__add = Set.union
```

此时两个集合相加的操作，就被设置成了并操作。

其他类型的metamethod用起来都和\_\_add没有本质的区别，这里仅记录一些都有哪些：

算数运算：乘法（mul），减法（sub）、浮点除法（div）、向下取整除法（idiv）、取反（unm）、取模（mod）和求幂（pow。所有的按位操作也有对应的元方法：按位与（band）、按位或（bor）、按位异或（bxor）、按位非（bnot）、左移（shl）和右移（shr）。还可以为连接运算符定义行为，使用字段 concat。（注意每个操作定义时都是需要\_\_的）  
关系运算：等于（eq），小于（lt），小于等于（le）。其他的比较都是根据这三个进行转换的。  
Library-Defined：\_\_tostring，\_\_metatable。

对于\_\_metatable，它可以保护我们已经定义好的类型，设置了\_\_metatable之后，再使用getmetatable会去获取\_\_metatable的value，并且此时不再允许setmetatable了。  
比如对之前的set类的元表添加一行：  
```lua
mt.__metatable = "can not set metatable"
```
之后再另一个文件中，按如下方式写：
```lua
local class  = require("set")
local s1 = class.new{10, 20, 30, 50}
print(getmetatable(s1))

local m2 = {}
setmetatable(s1, m2)
```
这时输出不会是表的地址，而是"can not set metatable"，并且后面的setmetatable会报错：  
*cannot change protected metatable*

# Table-Access Metamethods

除了上面的一些运算操作，对于元表最关键的元方法是\_\_index和\_\_newindex，他们分别定义了表中不存在key的时候的访问和设置操作。通过他们可以为表定义一些默认的行为。比如：

```lua
function setDefault(t, value)
    local mt = {
        __index = function ()
            return value
        end
    }
    setmetatable(t, mt)
end

local tab = { x = 1, y = 2}
print(tab.z) --> nil
setDefault(tab, 0)
print(tab.z) --> 0
```
上述代码中，tab初始时没设置z，所以自然返回nil，而在设置了元表之后，即便仍然没有z，但是元表中存在\_\_index，会按照index的metamethod执行，返回设置好的默认值0。

这里有一个设计上的问题，如果有多个表都调用setDefault设置元表，同时因为return value是通过外面的参数传进来的，每一次调用也会多一个闭包，在实际编程中可能经常会不注意造成性能浪费，尤其是在模拟面向对象的时候，元表往往就是类或者父类，如果不注意开销会很大。解决方法是要把可能会多次用于设置的元表提出去。比如：

```lua
local key = {}
local mt_g = {
    __index = function(t)
        return t[key]
    end
}
function setDefault_(t, value)
    t[key]= value
    setmetatable(t, mt_g)
end

setDefault_(tab, 0)

local tab2 = { x = 3 }
setDefault_(tab2, 1)

print(getmetatable(tab))    --> table:000001C5A425F650
print(getmetatable(tab2))   --> table:000001C5A425F650
```

如上述代码，此时的\_\_index设置的函数需要传递参数t，也就是包含这个元表的变量本身。在setDefault_中设置了t\[key\]用来保存默认值数据，所以按照这样再访问元表时，每次访问的都会是一个元表，默认值本身存到了每个变量自身当中。  
这里有一个小点：使用{}作为key，可以保证键值唯一，防止命名冲突，因为lua表以表为键的时候存的是地址，每次新建一个空表存的地址不一样。
```lua
local t = {}
local key1 = {}
local key2 = {}

t[key1] = "value1"
t[key2] = "value2"

print(t[key1]) -- 输出 "value1"
print(t[key2]) -- 输出 "value2"
```

## 控制表的访问

如果我们想对某个表的访问进行控制，比如说进行追踪记录或者限制读写等，可以使用代理表的方法。以追踪表的访问功能为例，如果只是加一个元表，我们虽然可以再元表中的访问方法中设置一些记录，但是如果本身表中就存在某些key，lua是不会执行元表中的index和newindex的，这里就需要使用另外一个表来代理。

比如Programming in lua中的例子：
```lua
function track (t)
    local proxy = {} -- proxy table for 't'
    -- create metatable for the proxy
    local mt = {
        __index = function (_, k)
            print("*access to element " .. tostring(k))
            return t[k] -- access the original table
        end,

        __newindex = function (_, k, v)
            print("*update of element " .. tostring(k) ..
            " to " .. tostring(v))
            t[k] = v -- update original table
        end,

        __pairs = function ()
            return function (_, k) -- iteration function
                local nextkey, nextvalue = next(t, k)
                if nextkey ~= nil then -- avoid last value
                    print("*traversing element " .. tostring(nextkey))
                end
                return nextkey, nextvalue
            end
        end,

        __len = function () return #t end
    }
    setmetatable(proxy, mt)
    return proxy
end

t = {x = 1} -- an arbitrary table
t = track(t)
```

可以看到在track函数中，返回的是空表proxy，其作为代理表，而在访问表的时候因为是空表，所有的操作均会走metamethod，而在metamethod之中会去方位原有表中有没有：t\[k\]，只是按这样写又构成了闭包，我们仍按照前面提到的方法进行改进，作为练习，可以改成如下：

```lua
local key = {}
local mt_ = {
    __index = function (t, k)
        print("*access to element " .. tostring(k))
        return t[key][k] -- access the original table
    end,

    __newindex = function (t, k, v)
        print("*update of element " .. tostring(k) ..
        " to " .. tostring(v))
        t[key][k] = v -- update original table
    end,

    __len = function (t) return #t[key] end
}

function track (t)
    local proxy = {} -- proxy table for 't'
    proxy[key] = t
    setmetatable(proxy, mt_)
    return proxy
end

local trackTab = {10, 20}
trackTab = track(trackTab)
trackTab[1] = "hello"
print(trackTab[1])
print(getmetatable(trackTab))

local t2 = {}
t2 = track(t2)
t2[1] = "world"
print(t2[1])
print(getmetatable(t2))

--[[
Output:
    *update of element 1 to hello
    *access to element 1
    hello
    table:000001C5A425EE10
    *update of element 1 to world
    *access to element 1
    world
    table:000001C5A425EE10
]]
```

类似的方法，我们可以将表设置为只读的，在newindex中输出error，而不进行赋值即可。

More

Programming in lua 4th