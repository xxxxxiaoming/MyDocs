# C++ 与 Lua 交互机制深度解析

Lua 与 C++ 的交互核心在于一个**虚拟栈（Virtual Stack）**。这种设计实现了宿主语言与脚本语言的解耦，并解决了两种语言之间动态类型与静态类型、自动内存管理与手动内存管理的冲突。

---

## 1. 交互基石：虚拟栈的底层实现

Lua 不直接暴露其内部对象的指针给 C++，而是通过栈索引来操作。

### 1.1 索引转地址 (`index2addr`)
在 `lapi.c` 中，几乎所有的 API 都会先调用 `index2addr` 将逻辑索引转换为物理内存地址。

```c
/* lapi.c: 将索引转换为 TValue 地址 */
static TValue *index2addr (lua_State *L, int idx) {
  CallInfo *ci = L->ci;
  if (idx > 0) {  /* 正数索引：从当前函数的栈底开始算 */
    TValue *o = ci->func + idx;
    api_check(L, idx <= ci->top - (ci->func + 1), "unacceptable index");
    if (o >= L->top) return NONVALIDVALUE;
    else return o;
  }
  else if (!ispseudo(idx)) {  /* 负数索引：从当前栈顶向后数 */
    api_check(L, idx != 0 && -idx <= L->top - (ci->func + 1), "invalid index");
    return L->top + idx;
  }
  else if (idx == LUA_REGISTRYINDEX) /* 伪索引：指向注册表 */
    return &G(L)->l_registry;
  else {  /* Upvalues 或者是其他伪索引 */
    /* ... 篇幅原因省略 Upvalue 处理逻辑 ... */
  }
}
```

### 1.2 栈的物理构成
- **`L->stack`**: 整个线程的物理栈空间。
- **`L->top`**: 指向栈中第一个空闲插槽的指针。
- **`L->ci->func`**: 指向当前正在运行的函数的插槽。当前函数的第一个参数位于 `func + 1`。

---

## 2. C++ 调用 Lua 脚本

这是 C++ 作为宿主，主动控制脚本的行为。

### 2.1 步骤与代码示例
1. **压入函数**：将要调用的 Lua 函数从全局表或注册表中取出。
2. **压入参数**：按照从左到右的顺序压入参数。
3. **执行调用**：使用 `lua_pcall` 发起保护模式调用。

```cpp
// C++ 代码示例
void CallLuaFunction(lua_State* L) {
    // 1. 获取全局函数 "add"
    lua_getglobal(L, "add"); 

    // 2. 压入两个参数
    lua_pushinteger(L, 10); // 第1个参数
    lua_pushinteger(L, 20); // 第2个参数

    // 3. 执行调用 (2个参数, 1个返回值)
    // lua_pcall 会在调用结束后自动将函数和参数弹出栈
    if (lua_pcall(L, 2, 1, 0) != LUA_OK) {
        const char* err = lua_tostring(L, -1);
        printf("Error: %s\n", err);
        lua_pop(L, 1); // 弹出错误信息
        return;
    }

    // 4. 从栈顶获取结果
    if (lua_isinteger(L, -1)) {
        int result = lua_tointeger(L, -1);
        printf("Result: %d\n", result);
    }
    lua_pop(L, 1); // 清理栈顶结果
}
```

### 2.2 源码层面的“返回处理” (`luaD_poscall`)
当 Lua 函数执行完，底层会调用 `luaD_poscall` 将结果移动到函数原本所在的位置。

```c
/* ldo.c: 处理函数返回 */
int luaD_poscall (lua_State *L, CallInfo *ci, StkId firstResult, int nres) {
  StkId res = ci->func;  /* res 指向函数原来在栈中的位置 */
  int wanted = ci->nresults; /* 期望得到的返回值个数 */
  L->ci = ci->previous;  /* 恢复调用者的执行环境 (CallInfo) */
  
  /* moveresults 将 firstResult 处的 nres 个结果移动到 res 处，并重置 L->top */
  return moveresults(L, firstResult, res, nres, wanted);
}
```

---

## 3. Lua 调用 C++ 函数

这是通过将 C++ 函数封装为 `lua_CFunction` 并注册到 Lua 环境中实现的。

### 3.1 C++ 函数规范
必须符合 `typedef int (*lua_CFunction) (lua_State *L);` 签名。

```cpp
// C++ 函数实现
int cpp_Multiply(lua_State* L) {
    // 1. 从栈中获取 Lua 传来的参数
    int a = luaL_checkinteger(L, 1); 
    int b = luaL_checkinteger(L, 2);

    // 2. 计算结果
    int res = a * b;

    // 3. 将结果压入栈中给 Lua
    lua_pushinteger(L, res);

    // 4. 返回结果个数
    return 1; 
}
```

### 3.2 注册与使用
```cpp
// 在初始化时注册
lua_register(L, "multiply", cpp_Multiply);

// --- Lua 脚本中即可像普通函数一样调用 ---
// local res = multiply(5, 6)
// print(res) -- 输出 30
```

---

## 4. 深度细节：Table 与全局状态操作

### 4.1 获取 Table 字段的底层流程 (`lua_getfield`)
当你调用 `lua_getfield(L, idx, "name")` 时：
1. **内部压栈**：Lua 内部会自动执行 `lua_pushstring(L, "name")`。
2. **表查找**：根据 `idx` 找到 Table，以栈顶的字符串作为 Key 进行查找。
3. **结果覆盖**：查找完成后，原本占位的 Key 字符串会被找到的 Value **覆盖**。

### 4.2 保护模式调用的意义
C++ 与 Lua 交互最忌讳的是 **Lua 内部 Error 导致整个 C++ 进程直接 `abort`**。
- **`lua_call`**: 非保护调用。如果脚本报错，Lua 会直接调用 `panic` 函数并退出程序。
- **`lua_pcall`**: 保护调用。它利用了 C 语言的 `setjmp/longjmp` 机制，在发生错误时能“跳回”到 `pcall` 处并返回错误代码，保证 C++ 环境的稳定。

---

## 5. 总结：交互流转图

1. **C++ -> Lua**:
   `获取函数 -> 压参 -> lua_pcall -> 获取结果 -> lua_pop`
2. **Lua -> C++**:
   `实现符合签名的C函数 -> lua_register -> Lua脚本调用 -> C函数内读取栈参数 -> C函数内压入返回结果`

**关键点提醒**：
- 始终关注 **栈平衡**：谁压入的，谁负责弹出（或者由 API 如 `pcall` 自动清理）。
  ```cpp
  // 错误：忘记弹出。多次执行会导致栈溢出
  lua_getglobal(L, "myVar"); 
  int v = lua_tointeger(L, -1);
  // 必须添加：lua_pop(L, 1); 
  ```
- 索引 **-1** 永远是你的“当前操作目标”。
- 复杂对象（Table/Userdata）传递的是**引用**，基本类型传递的是**拷贝**。
  ```cpp
  // 修改参数 table (引用) 会影响到 Lua 层原有的 table
  int cpp_ModifyTable(lua_State* L) {
      lua_pushstring(L, "new_key");
      lua_pushinteger(L, 999);
      lua_settable(L, 1); // 修改了 Lua 传进来的第一个参数 (table)
      return 0; // Lua 中的原有 table 现在有了 "new_key" = 999
  }
  
  // 修改参数 number (拷贝) 不会影响 Lua 层原有的变量
  int cpp_TryModifyNum(lua_State* L) {
      int v = lua_tointeger(L, 1);
      v = 1000; // 仅仅修改了 C++ 局部变量
      return 0; // Lua 层原变量数值保持不变
  }
  ```
