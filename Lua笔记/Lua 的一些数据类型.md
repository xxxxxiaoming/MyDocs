# Lua的一些数据类型

## global_State

```C
/*
** 'global state', shared by all threads of this state
*/
typedef struct global_State {
  lua_Alloc frealloc;  /* function to reallocate memory */
  void *ud;         /* auxiliary data to 'frealloc' */
  l_mem totalbytes;  /* number of bytes currently allocated - GCdebt */
  l_mem GCdebt;  /* bytes allocated not yet compensated by the collector */
  lu_mem GCmemtrav;  /* memory traversed by the GC */
  lu_mem GCestimate;  /* an estimate of the non-garbage memory in use */
  stringtable strt;  /* hash table for strings */
  TValue l_registry; /* Lua的注册表，实际上是利用TValue这个Union的GCObject*这个指针指向一个Table，想要访问这个Table可以用 gco2t(l_registery.gc)这种方式 */
  unsigned int seed;  /* randomized seed for hashes */
  lu_byte currentwhite; /* 当前白，GC对象的marked为当前白时，在当前所处的GC周期中，不会被清理掉 */
  lu_byte gcstate;  /* state of garbage collector */
  lu_byte gckind;  /* kind of GC running */
  lu_byte gcrunning;  /* true if GC is running */
  GCObject *allgc;  /* list of all collectable objects */
  GCObject **sweepgc;  /* current position of sweep in list */
  GCObject *finobj;  /* list of collectable objects with finalizers */
  GCObject *gray;  /* list of gray objects */
  GCObject *grayagain;  /* list of objects to be traversed atomically */
  GCObject *weak;  /* list of tables with weak values */
  GCObject *ephemeron;  /* list of ephemeron tables (weak keys) */
  GCObject *allweak;  /* list of all-weak tables */
  GCObject *tobefnz;  /* list of userdata to be GC */
  GCObject *fixedgc;  /* list of objects not to be collected */
  struct lua_State *twups;  /* list of threads with open upvalues */
  unsigned int gcfinnum;  /* number of finalizers to call in each GC step */
  int gcpause;  /* size of pause between successive GCs */
  int gcstepmul;  /* GC 'granularity' */
  lua_CFunction panic;  /* to be called in unprotected errors */
  struct lua_State *mainthread;
  const lua_Number *version;  /* pointer to version number */
  TString *memerrmsg;  /* memory-error message */
  TString *tmname[TM_N];  /* array with tag-method names */
  struct Table *mt[LUA_NUMTAGS];  /* metatables for basic types */
  TString *strcache[STRCACHE_N][STRCACHE_M];  /* cache for strings in API */
} global_State;
```

关于global_state中的二级指针

在 Lua 的增量清扫（Incremental Sweep）中，为了能够随时中断并恢复，g->sweepgc 始终指向 “下一个待处理对象的指针地址”。

在 allgc 这个单向链表中，如果你想指向某个对象 B，你有两种选择：

指向 B 的首地址（GCObject *）。

指向 “前一个对象 A 的 next 成员的地址”（GCObject **）。

Lua 选择了第 2 种。这样做的好处是：当清扫过程中需要删除对象 B 时，直接通过 *p = B->next 就能把 B 从链表中抠掉，而不需要再次遍历链表去找 B 的前驱节点。

## l_registry

l_registry的底细已经加上注释了，这里再来看一下l_registry的初始化，已经l_registry这个注册表里有哪些东西。从初始化l_registry的函数可以看出一些细节：

```C
/*
** Create registry table and its predefined values
*/
static void init_registry (lua_State *L, global_State *g) {
  TValue temp;
  /* create registry */
  Table *registry = luaH_new(L);
  sethvalue(L, &g->l_registry, registry); /* 利用TValue这个Union的GCObject*这个指针指向一个registry这个Table */
  luaH_resize(L, registry, LUA_RIDX_LAST, 0); /* 对registry这个Table进行扩容，array part容量扩展为2（LUA_RIDX_LAST这个宏定义就是2），hash part容量依旧是0 */
  /* registry[LUA_RIDX_MAINTHREAD] = L */
  setthvalue(L, &temp, L);  /* temp = L */
  luaH_setint(L, registry, LUA_RIDX_MAINTHREAD, &temp); /* 往registry这个Table插入global_State的mainthread，key为1 */
  /* registry[LUA_RIDX_GLOBALS] = table of globals */
  sethvalue(L, &temp, luaH_new(L));  /* temp = new table (global table) 再往registry插入一个空Table，key为2，这个空Table我猜就是用来存放Lua中定义的所有全局的东西 */
  luaH_setint(L, registry, LUA_RIDX_GLOBALS, &temp);
}
```

```C
/*
** 'per thread' state
*/
struct lua_State {
  CommonHeader;
  unsigned short nci;  /* number of items in 'ci' list */
  lu_byte status;
  StkId top;  /* first free slot in the stack top是下一个可以写入TValue的地方，不是最后一次写入TValue的地方！！！ */
  global_State *l_G;
  CallInfo *ci;  /* call info for current function */
  const Instruction *oldpc;  /* last pc traced */
  StkId stack_last;  /* last free slot in the stack */
  StkId stack;  /* stack base */
  UpVal *openupval;  /* list of open upvalues in this stack */
  GCObject *gclist;
  struct lua_State *twups;  /* list of threads with open upvalues */
  struct lua_longjmp *errorJmp;  /* current error recover point */
  CallInfo base_ci;  /* CallInfo for first level (C calling Lua) */
  volatile lua_Hook hook;
  ptrdiff_t errfunc;  /* current error handling function (stack index) */
  int stacksize;
  int basehookcount;
  int hookcount;
  unsigned short nny;  /* number of non-yieldable calls in stack */
  unsigned short nCcalls;  /* number of nested C calls */
  l_signalT hookmask;
  lu_byte allowhook;
};
```

stack 指向的是一块连续内存，**首先需要明确，这块内存中，只存放一种数据类型的数据，就是TValue！** 这块连续内存以及stack的初始化过程的函数调用顺序是这样的：

`lua_newstate` -> `luaD_rawrunprotected` -> `f_luaopen` -> `stack_init` -> `luaM_newvector`

这块内存的初始化大小是 `BASIC_STACK_SIZE` 这个宏定义的，`stack` 指针指向内存的起始位置，`stack_last` 指向最后一个可用的位置，`stacksize`是这块内存总共能够放多少个TValue对象，`stack_last` 并不简单等于 `stack + stack_size`，而是` stack + stack_size - EXTRA_STACK`，就是这块内存其实有一部分用作其他用途了。