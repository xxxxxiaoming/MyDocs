# Lua元表（metatable）的C实现

## 存储与“免检”优化

```C
/* Lua支持的所有元key */
typedef enum {
  TM_INDEX,
  TM_NEWINDEX,
  TM_GC,
  TM_MODE,
  TM_LEN,
  TM_EQ,  /* last tag method with fast access */
  TM_ADD,
  TM_SUB,
  TM_MUL,
  TM_MOD,
  TM_POW,
  TM_DIV,
  TM_IDIV,
  TM_BAND,
  TM_BOR,
  TM_BXOR,
  TM_SHL,
  TM_SHR,
  TM_UNM,
  TM_BNOT,
  TM_LT,
  TM_LE,
  TM_CONCAT,
  TM_CALL,
  TM_N		/* number of elements in the enum */
} TMS;

// table本尊
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present 用来缓存这个table没有定义哪些元key(TMS)，注意是没有 */
  lu_byte lsizenode;  /* log2 of size of 'node' array 哈希部分的元素个数，真正的个数 = 2 ^ lsizenode */
  unsigned int sizearray;  /* size of 'array' array  数组部分的元素个数*/
  TValue *array;  /* array part */
  Node *node; /* 哈希部分，实际上也是连续内存，可以看作是一个数组 */
  Node *lastfree;  /* any free position is before this position，指向最新插入的node，如果这个node的数组索引是8，那么下一个能够插入Node的索引就是7 */
  struct Table *metatable; /* Metatable 本尊 */
  GCObject *gclist;
} Table;
```

Lua有一个快速机制来判断一个table中有没有某个元key(根据TMS的注释，这个机制只用来判断TM_INDEX到TM_EQ这几个元key是否存在)：

```C
#define fasttm(l,et,e)  gfasttm(G(l), et, e)
#define gfasttm(g,et,e) ((et) == NULL ? NULL : \
  ((et)->flags & (1u<<(e))) ? NULL : luaT_gettm(et, e, (g)->tmname[e]))
  /*
  ** (et)->flags & (1u<<(e))，这里的就是对应TMS中的元素
  ** 去除flags的第e位，1 << 1 -> 10, 跟flags按位与，就是取出flags的第二位
  ** 如果 (et)->flags & (1u<<(e)) 为0，那就是没有这个元key, 不为0，就是有这个元key，通过luaT_gettm取出这个元key
  */


const TValue *luaT_gettm (Table *events, TMS event, TString *ename) {
  const TValue *tm = luaH_getshortstr(events, ename);
  lua_assert(event <= TM_EQ);
  if (ttisnil(tm)) {  /* no tag method? */
    /*再次检查，如果没有这个元key，更新flags*/
    events->flags |= cast_byte(1u<<event);  /* cache this fact */
    return NULL;
  }
  else return tm;
}
```

这个机制可以避免不管元key是否存在，都直接去metatable里面去索引查找，提高性能。

## 事件名称的注册机制（字符串是怎么变成底层的枚举的？/ 枚举值跟字符串的映射是在哪里建立的）

fasttm 宏里看到了 (g)->tmname[e] 这个东西。写 Lua 代码时明明用的是 "__index" 这样的字符串，底层是怎么把它和 TM_INDEX 这个数字对应起来的呢？看看 global_state 以及 luaT_init这两个函数就知道了.

```C
typedef struct global_State {
  lua_Alloc frealloc;  /* function to reallocate memory */
  void *ud;         /* auxiliary data to 'frealloc' */
  l_mem totalbytes;  /* number of bytes currently allocated - GCdebt */
  l_mem GCdebt;  /* bytes allocated not yet compensated by the collector */
  lu_mem GCmemtrav;  /* memory traversed by the GC */
  lu_mem GCestimate;  /* an estimate of the non-garbage memory in use */
  stringtable strt;  /* hash table for strings */
  TValue l_registry;
  unsigned int seed;  /* randomized seed for hashes */
  lu_byte currentwhite;
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
  TString *tmname[TM_N];  /* array with tag-method names Lua定义的所有元key，枚举值跟名称字符串的映射，放在global state中，保证lua虚拟机运行期间，都不会被GC */
  struct Table *mt[LUA_NUMTAGS];  /* metatables for basic types */
  TString *strcache[STRCACHE_N][STRCACHE_M];  /* cache for strings in API 字符串缓存池 */
} global_State;

void luaT_init (lua_State *L) {
  static const char *const luaT_eventname[] = {  /* ORDER TM */
    "__index", "__newindex",
    "__gc", "__mode", "__len", "__eq",
    "__add", "__sub", "__mul", "__mod", "__pow",
    "__div", "__idiv",
    "__band", "__bor", "__bxor", "__shl", "__shr",
    "__unm", "__bnot", "__lt", "__le",
    "__concat", "__call"
  };
  int i;
  /*
  ** 在初始化函数中，建立起tmname，就是Lua定义的所有元key，枚举值跟名称字符串的映射
  */
  for (i=0; i<TM_N; i++) {
    /* luaS_new会优先从字符串缓存池中取地址，如果不存在需要的字符串，就新建一个放入缓存池中 */
    /* 这个机制为 eqshrstr 中，比较字符串是否相等提供了一个快速方法，不需要真的逐字符对比，只要比较两个字符串的地址是否一致即可 */
    G(L)->tmname[i] = luaS_new(L, luaT_eventname[i]);
    luaC_fix(L, obj2gco(G(L)->tmname[i]));  /* never collect these names */
  }
}
```

## 核心拦截器 Get 与 Set（__index 与 __newindex C实现）

当试图获取一个 Table 里不存在的 key 时，会掉进这个函数：

```C
/* val = t[key] 指向这句lua代码时 */
/* slot是用来存放t[key]的 */
void luaV_finishget (lua_State *L, const TValue *t, TValue *key, StkId val,
                      const TValue *slot) {
  int loop;  /* counter to avoid infinite loops */
  const TValue *tm;  /* metamethod */
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    if (slot == NULL) {  /* 't' is not a table? */
      lua_assert(!ttistable(t));
      tm = luaT_gettmbyobj(L, t, TM_INDEX);
      if (ttisnil(tm))
        luaG_typeerror(L, t, "index");  /* no metamethod */
      /* else will try the metamethod */
    }
    else {  /* 't' is a table */
      lua_assert(ttisnil(slot));
      /*尝试获取__index*/
      tm = fasttm(L, hvalue(t)->metatable, TM_INDEX);  /* table's metamethod */
      if (tm == NULL) {  /* no metamethod? */
        /* 没定义__index，val置为nil */
        setnilvalue(val);  /* result is nil */
        return;
      }
      /* else will try the metamethod */
    }
    if (ttisfunction(tm)) {  /* is metamethod a function? */
      /* __index 是函数的情况，直接指向这个函数 */
      luaT_callTM(L, tm, t, key, val, 1);  /* call it */
      return;
    }
    /* __index是table的情况，看看__index中有没有这个key（luaV_fastget），没有的话，用__index继续下一次循环，用MAXTAGLOOP 控制查找的最大次数，防止table嵌套太深的情况*/
    t = tm;  /* else try to access 'tm[key]' */
    if (luaV_fastget(L,t,key,slot,luaH_get)) {  /* fast track? */
      setobj2s(L, val, slot);  /* done */
      return;
    }
    /* else repeat (tail call 'luaV_finishget') */
  }
  luaG_runerror(L, "'__index' chain too long; possible loop");
}

```

当试图给 Table 设置一个全新的key，或者修改时触发了条件，会掉进这个函数：

```C
void luaV_finishset (lua_State *L, const TValue *t, TValue *key,
                     StkId val, const TValue *slot) {
  int loop;  /* counter to avoid infinite loops */
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    const TValue *tm;  /* '__newindex' metamethod */
    if (slot != NULL) {  /* is 't' a table? */
      Table *h = hvalue(t);  /* save 't' table */
      lua_assert(ttisnil(slot));  /* old value must be nil */
      tm = fasttm(L, h->metatable, TM_NEWINDEX);  /* get metamethod */
      if (tm == NULL) {  /* no metamethod? */
        if (slot == luaO_nilobject)  /* no previous entry? */
          slot = luaH_newkey(L, h, key);  /* create one */
        /* no metamethod and (now) there is an entry with given key */
        setobj2t(L, cast(TValue *, slot), val);  /* set its new value */
        invalidateTMcache(h);
        luaC_barrierback(L, h, val);
        return;
      }
      /* else will try the metamethod */
    }
    else {  /* not a table; check metamethod */
      if (ttisnil(tm = luaT_gettmbyobj(L, t, TM_NEWINDEX)))
        luaG_typeerror(L, t, "index");
    }
    /* try the metamethod */
    if (ttisfunction(tm)) {
      luaT_callTM(L, tm, t, key, val, 0);
      return;
    }
    t = tm;  /* else repeat assignment over 'tm' */
    if (luaV_fastset(L, t, key, slot, luaH_get, val))
      return;  /* done */
    /* else loop */
  }
  luaG_runerror(L, "'__newindex' chain too long; possible loop");
}
```

流程跟luaV_finishset大同小异，先看t有没有__newindex，没有的话，新的key直接放到t中，有的话，如果__newindex是一个函数，那就执行这个函数，如果__newindex是一个table（假设叫t1），查看t1有没有这个key，有的话直接修改t1的这个key的值，否则，就用t1开始下一次循环（判断t1有没有定义__newindex，没有的话，直接将这个key插入到t1，有的话，是函数还是table，重复t的步骤），这个函数同样用MAXTAGLOOP 来限制循环次数，防止table嵌套过深导致的问题。
