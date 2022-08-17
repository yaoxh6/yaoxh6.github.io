---
layout:     post
title:      "luna库使用"
subtitle:   ""
date:       2022-08-17 15:15:00
author:     "Yaoxh6"
catalog: true
header-style: text
tags:
  - lua
  - c++
---

仓库地址
-------
luna库主要作用是lua/c++绑定, 即方便导出c++类供lua使用

[luna库地址](https://github.com/trumanzhao/luna)

使用[demo地址](https://github.com/yaoxh6/lua-navmesh-demo)

luna库使用方法
-------------

### c++侧
class中使用`DECLARE_LUA_CLASS(navmesh);`

```c++
class navmesh {
public:
    navmesh() = default;
    ~navmesh();
    DECLARE_LUA_CLASS(navmesh);
    ...
};
```
方法导出
```c++
LUA_EXPORT_CLASS_BEGIN(navmesh)
LUA_EXPORT_METHOD(set_flags)
LUA_EXPORT_METHOD(set_flags_)
LUA_EXPORT_METHOD(set_extent)
LUA_EXPORT_METHOD(find_path)
LUA_EXPORT_CLASS_END()
```

lua接口

主要通过`lua_push_object(L, m_navmesh);`将数据返回给lua
```c++
static int lua_navmesh_create(lua_State* L)
{
    ...
    auto m_navmesh = new navmesh();
    ...
    lua_push_object(L, m_navmesh);
    return 1;
}

int luaopen_navmesh(lua_State* L)
{
    luaL_checkversion(L);
    luaL_Reg l[] = {
        { "create", lua_navmesh_create},
        { NULL, NULL },
    };
    luaL_newlib(L, l);
    return 1;
}
```
### lua侧
```lua
local navmesh = require "navmesh"
local function init_navmesh()
    local query_extent = {x=50, y=50, z=250}
    local include_flag = 32767
    local exclude_flag = 0
    local navmesh_instance = navmesh.create("../../app/demo/demo.navmesh", include_flag, exclude_flag, query_extent)
    assert(navmesh_instance)
    navmesh_instance.set_flags_(32767, 0);
end
init_navmesh()
```
### 宏展开
通过下面设置,可以得到`*.i`文件
![得到预编译后的文件](/img/in-post/post-luna/define.png)
```c++
const char* lua_get_meta_name() { return "_class_meta:""navmesh"; } lua_member_item* lua_get_meta_data();
```

```c++
lua_member_item* navmesh::lua_get_meta_data() { using class_type = navmesh; static lua_member_item s_member_list[] = {
{ "set_flags", 0, lua_export_helper::getter(&class_type::set_flags), lua_export_helper::setter(&class_type::set_flags)},
{ "set_flags_", 0, lua_export_helper::getter(&class_type::set_flags_), lua_export_helper::setter(&class_type::set_flags_)},
{ "set_extent", 0, lua_export_helper::getter(&class_type::set_extent), lua_export_helper::setter(&class_type::set_extent)},
{ "find_path", 0, lua_export_helper::getter(&class_type::find_path), lua_export_helper::setter(&class_type::find_path)},
{ nullptr, 0, luna_member_wrapper(), luna_member_wrapper()} }; return s_member_list; }
```
### 调用过程

1. `local navmesh = require "navmesh"` 通过`require "navmesh"`, 找到c++侧的`luaopen_navmesh`函数
2. lua侧的`create`函数在c++对应`lua_navmesh_create`, 所以`navmesh.create(...)`相当于直接调用c++`的lua_navmesh_create`
```c++
luaL_Reg l[] = {
    { "create", lua_navmesh_create},
    { NULL, NULL },
};
```
3. `lua_navmesh_create`主要是初始化一个`navmesh`实例, 并且通过`lua_push_object`的方式将封装好的数据放在栈上,`return 1`则会把这个数据返回给lua
4. `lua_push_object`过程有点复杂, 通过注释的方式将栈的情况写出来, 可以看出返回lua的是一个table, `{__pointer__:obj}`,obj则是c++类navmesh指针,使用的是`lua_pushlightuserdata`放到栈上
```c++
template <typename T>
void lua_push_object(lua_State* L, T obj) {
    if (obj == nullptr) {
        lua_pushnil(L);
        return;
    }

    lua_getfield(L, LUA_REGISTRYINDEX, "__objects__");
    if (lua_isnil(L, -1)) {
        lua_pop(L, 1);
        lua_newtable(L); // L

        lua_newtable(L); // L, L1
        lua_pushstring(L, "v"); // L, L1, "v"
        lua_setfield(L, -2, "__mode"); // L, L1 {"__mode__":v}
        lua_setmetatable(L, -2); // L

        lua_pushvalue(L, -1); // L, L
        lua_setfield(L, LUA_REGISTRYINDEX, "__objects__"); // L
    }

    char key[MAX_LUA_OBJECT_KEY];
    const char* pkey = lua_get_object_key(key, obj);
    if (pkey == nullptr) {
        lua_pop(L, 1);
        lua_pushnil(L);
        return;
    }

    // stack: __objects__
    if (lua_getfield(L, -1, pkey) != LUA_TTABLE) { // L, nullptr
        if (!_lua_set_fence(L, pkey)) {
            lua_remove(L, -2);
            return;
        }

        lua_pop(L, 1); // L

        lua_newtable(L);// L, L1
        lua_pushstring(L, "__pointer__"); // L, L1, "__pointer__"
        lua_pushlightuserdata(L, obj); // L, L1, "__pointer__", obj
        lua_rawset(L, -3); // L, L1 {"__pointer__":obj}

        // stack: __objects__, tab
        const char* meta_name = obj->lua_get_meta_name();
        luaL_getmetatable(L, meta_name); // L, L1, meta
        if (lua_isnil(L, -1)) {
            lua_remove(L, -1);
            lua_register_class(L, obj);
            luaL_getmetatable(L, meta_name);
        }
        lua_setmetatable(L, -2); // L, L1

        // stack: __objects__, tab
        lua_pushvalue(L, -1); // L, L1, L1
        lua_setfield(L, -3, pkey); // L {"pkey":L1}, L1
    }
    lua_remove(L, -2); // L1 {"__pointer__":obj}
}
```