# Table的C实现

## 跟Table直接相关的几种数据结构

```C
/* lua中，一个table的元素可以是整数，浮点数，字符串，函数，C++对象等各种花样，就是靠这个实现的
   或者更大一统的说法，lua中变量不需要写明类型，也是靠这个实现的（我猜的）
*/
typedef union Value {
  GCObject *gc;    /* collectable objects 这个指针可以用来指向Lua种的GCObject，比如Table/TString等 */
  void *p;         /* light userdata */
  int b;           /* booleans */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;


// 数组中，每个元素的类型实际上就是这个TValue
struct TValue
{
    Value value;
    int tt;         -- 用来表示元素的类型（lua.h line 64 ~ 72）
}

// table哈希部分的key的定义
typedef union TKey {
  struct {
    TValuefields;
    // TValuefields equals to:
    // value value_;
    // int tt_;
    int next;  /* for chaining (offset for next node) */
  } nk;
  TValue tvk;
} 

// table哈希部分的每一个元素就长这样
typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node

// table本尊
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;  /* log2 of size of 'node' array 哈希部分的元素个数，真正的个数 = 2 ^ lsizenode */
  unsigned int sizearray;  /* size of 'array' array  数组部分的元素个数*/
  TValue *array;  /* array part */
  Node *node; /* 哈希部分，实际上也是连续内存，可以看作是一个数组 */
  Node *lastfree;  /* any free position is before this position，指向最新插入的node，如果这个node的数组索引是8，那么下一个能够插入Node的索引就是7 */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```

## 新建一个table

```C
Table *luaH_new (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_TTABLE, sizeof(Table));
  Table *t = gco2t(o);
  t->metatable = NULL;
  t->flags = cast_byte(~0);
  t->array = NULL;      /*新的table数组部分不会直接分配内存*/
  t->sizearray = 0;
  setnodevector(L, t, 0); /*新的table哈希部分也不直接分配内存，因为setnodevector第三个参数是0，node会直接指向一个全局存在的Node对象，叫dummynode，这样做可以避免野指针的问题（我猜的）*/
  return t;
}

/*dummynode 长这样*/
#define dummynode		(&dummynode_)

static const Node dummynode_ = {
  {NILCONSTANT},  /* value */
  {{NILCONSTANT, 0}}  /* key */
};

static void setnodevector (lua_State *L, Table *t, unsigned int size) {
  if (size == 0) {  /* no elements to hash part? */
    t->node = cast(Node *, dummynode);  /* use common 'dummynode'*/
    t->lsizenode = 0;
    t->lastfree = NULL;  /* signal that it is using dummy node */
  }
  else {
    int i;
    int lsize = luaO_ceillog2(size);
    if (lsize > MAXHBITS)
      luaG_runerror(L, "table overflow");
    size = twoto(lsize);/*这一步，保证哈希部分的元素个数是2的整数次幂，尽管传进来的size不是2的整数次幂，到了这一步，都会变成2的整数次幂*/
    t->node = luaM_newvector(L, size, Node); /*分配内存发生在这一步，调用的是luaM_realloct*/
    for (i = 0; i < (int)size; i++) {
      Node *n = gnode(t, i); /*gnode means getnode，获取第i个node*/
      gnext(n) = 0;/*gnext means getnext*/
      setnilvalue(wgkey(n));/*writable gkey, gkey means getkey, gkey 会把i_key变成const的，从而阻止写入, 把每个i_key的Value都set成nil（就是TValue的tt_ = 0）*/
      setnilvalue(gval(n));/*gval means getvalue, 这个没有区分写入跟不可写入，都是可写入的，同样把i_val设置成nil*/
    }
    t->lsizenode = cast_byte(lsize);
    t->lastfree = gnode(t, size);  /* all positions are free */
  }
}
```

## lua如何通过key取对应的value

通过key的TValuefields/TValue -> tt_，判断key的类型，调用对应的get函数

```C
const TValue *luaH_get (Table *t, const TValue *key) {
  switch (ttype(key)) {
    case LUA_TSHRSTR: return luaH_getshortstr(t, tsvalue(key));
    case LUA_TNUMINT: return luaH_getint(t, ivalue(key));
    case LUA_TNIL: return luaO_nilobject;
    case LUA_TNUMFLT: {
      lua_Integer k;
      if (luaV_tointeger(key, &k, 0)) /* index is int? */
        return luaH_getint(t, k);  /* use specialized version */
      /* else... */
    }  /* FALLTHROUGH */
    default:
      return getgeneric(t, key);
  }
}
```

以luaH_getint为例：

```C
const TValue *luaH_getint (Table *t, lua_Integer key) {
  /* (1 <= key && key <= t->sizearray) */
  /* key小于等于数组部分的长度，直接在数组部分查找对应的值*/
  if (l_castS2U(key) - 1 < t->sizearray)
    return &t->array[key - 1];
  /*否则，在哈希部分查找对应值*/
  else {
    /*通过哈希，获取一个node（查找的起点）*/
    Node *n = hashint(t, key);
    for (;;) {  /* check whether 'key' is somewhere in the chain */
      /*node的key跟要查找的key一致，找到！返回node的value*/
      if (ttisinteger(gkey(n)) && ivalue(gkey(n)) == key)
        return gval(n);  /* that's it */
    /*否则，获取下一个node，继续判断*/
      else {
        int nx = gnext(n);
        if (nx == 0) break;
        n += nx;
      }
    }
    return luaO_nilobject;
  }
}
```

对于短字符串，有一个专用的函数

```C
/*
** search function for short strings
*/
const TValue *luaH_getshortstr (Table *t, TString *key) {
  Node *n = hashstr(t, key);
  lua_assert(key->tt == LUA_TSHRSTR);
  for (;;) {  /* check whether 'key' is somewhere in the chain */
    const TValue *k = gkey(n);
    if (ttisshrstring(k) && eqshrstr(tsvalue(k), key))
      return gval(n);  /* that's it */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return luaO_nilobject;  /* not found */
      n += nx;
    }
  }
}

```

此外，还有一个通用的函数，不管key是什么类型，都能够使用：

```C
/*
** "Generic" get version. (Not that generic: not valid for integers,
** which may be in array part, nor for floats with integral values.)
*/
static const TValue *getgeneric (Table *t, const TValue *key) {
  Node *n = mainposition(t, key);
  for (;;) {  /* check whether 'key' is somewhere in the chain */
    if (luaV_rawequalobj(gkey(n), key))
      return gval(n);  /* that's it */
    else {
      int nx = gnext(n);
      if (nx == 0)
        return luaO_nilobject;  /* not found */
      n += nx;
    }
  }
}
```

## 哈希部分如何设置一个key（重点：如何解决hash冲突）

```C
TValue *luaH_set (lua_State *L, Table *t, const TValue *key) {
  const TValue *p = luaH_get(t, key);
  if (p != luaO_nilobject)
    return cast(TValue *, p);
  else return luaH_newkey(L, t, key);
}
```

- 首先，查找这个key是不是已经存在的（就是那种要修改一个已有的key的value的情况）
- 如果存在，那就返回这个key的Node
- 如果不存在，那就要搞一个新的节点来存放这个key了
  
如何确定这个key要放在哪个节点是在luaH_newKey这个函数中完成的。涉及到哈希冲突如何处理，先看一下源码

```C
/*
** inserts a new key into a hash table; first, check whether key's main
** position is free. If not, check whether colliding node is in its main
** position or not: if it is not, move colliding node to an empty place and
** put new key in its main position; otherwise (colliding node is in its main
** position), new key goes to an empty position.
** 这里，为了方便，假设这个key叫做king
*/
TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp;
  TValue aux;
  if (ttisnil(key)) luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {
    lua_Integer k;
    if (luaV_tointeger(key, &k, 0)) {  /* does index fit in an integer? */
      setivalue(&aux, k);
      key = &aux;  /* insert it as an integer */
    }
    else if (luai_numisnan(fltvalue(key)))
      luaG_runerror(L, "table index is NaN");
  }
  mp = mainposition(t, key); /*通过hash，计算出king要放在哪个节点上，这个节点就叫做这个king的main position*/
  
  /*king的main position节点不为空，就是说已经被其他key占用了，假设占用这个node的key叫做queen*/
  if (!ttisnil(gval(mp)) || isdummy(t)) {  /* main position is taken? */
    Node *othern;
    Node *f = getfreepos(t);  /* get a free place 获取一个没被占用的节点*/
    
    /*所有节点都被占用了，需要扩容了（rehash做的事情）*/
    if (f == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      return luaH_set(L, t, key);  /* insert key into grown table */
    }
    lua_assert(!isdummy(t));/*扩容失败？*/
    
    /*计算一下queen的main position是哪个节点，othern直线queen的main position节点*/
    othern = mainposition(t, gkey(mp));

    /*othern != mp，说明queen现在所在的节点并不是queen的main position节点*/
    /*这种情况，Lua会将queen移动到f这个没被占用的节点，这就需要修改node之间的连接关系*/
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */

      /*while 循环找到，哪个节点目前连接到queen节点上*/
      while (othern + gnext(othern) != mp)  /* find previous */
        othern += gnext(othern);

      /*找到之后，讲这个节点连接到找到的空闲节点f，也就是queen的新家上*/
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */
      
      /*将queen搬动到新的空闲节点f上*/
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      
      /*让f这个节点连接到原来queen所在节点的下一个节点上，queen的搬家工作完成*/
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* correct 'next' */
        gnext(mp) = 0;  /* now 'mp' is free */
      }
      setnilvalue(gval(mp));
    }

    /*other == mp，说明queen目前就在她的main position，king只能放到空闲节点f上*/
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */

      /*同样需要修改连接关系，让queen指向king，king指向原先queen指向的那个节点*/
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }

  /*搬家 + 连接关系修改完毕，也得到能够放置king的node，初始化这个node的内容，工作完成*/
  setnodekey(L, &mp->i_key, key);
  luaC_barrierback(L, t, key);
  lua_assert(ttisnil(gval(mp)));
  return gval(mp);
}
```

整个过程，可以简单总结为函数开头的注释：

- 找到新key的main position
- 如果main position被占用，看占用这个main postion的节点的main position是不是也是同一个节点
  - 如果是，那只能给新的节点找一个空闲的节点放置（没用空闲节点就需要rehash扩容）
  - 如果不是，那就把占用这个main position的节点移动到一个空闲节点上
  
对，就这样，函数里比较难懂的部分我觉得就是修改连接关系。

## table的扩容

对table进行扩容的函数是：

```C
/*
** nums[i] = number of keys 'k' where 2^(i - 1) < k <= 2^i
** 比如，nums[7] = 77，表示array part + hash part 总共有77个value不为nil的，且key是正整数，key的值范围在[2^6, 2^7)的key
*/
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse; /*array part + hash part 总共有多少个value不为nil的key，也就是总共有多少个元素*/
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
  na = numusearray(t, nums);  /* count keys in array part */
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
  /* count extra key，别忘记统计新插入的key */
  na += countint(ek, nums);
  totaluse++;
  /* compute new size for array part */
  /* computesizes之前，na是现在这个table(未扩容之前)的array part总共有多少个key，也就是多少个元素 */
  asize = computesizes(nums, &na);
  /* computesizes之后，na表示扩容之后的table的array part总共有多少个key，也就是扩容后的table的array part有多少个元素 */
  /* asize是扩容后的table中，array part的长度，也就是扩容后的table的array part最多可以容纳多少个元素 */
  /* resize the table to new computed sizes */
  luaH_resize(L, t, asize, totaluse - na);
}
```

接下来，看一下如何计算扩容后的table的array part的容量以及array part需要放多少个元素，也就是computesize返回的asize以及na是如何计算的

```C
static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
  int i;
  unsigned int twotoi;  /* 2^i (candidate for optimal size) */
  unsigned int a = 0;  /* number of elements smaller than 2^i */
  unsigned int na = 0;  /* number of elements to go to array part，新的array part的元素个数 */
  unsigned int optimal = 0;  /* optimal size for array part， 新的array part的容量*/
  /* loop while keys can fill more than half of total size */

  for (i = 0, twotoi = 1; *pna > twotoi / 2; i++, twotoi *= 2) {
    if (nums[i] > 0) {
      a += nums[i]; /* 未扩容前的table中，key值小于 2^i 的key有多少个 */
      if (a > twotoi/2) {  /* more than half elements present? */
        /* 新的array part如果容量为 2^i，按照当前table的统计，能够填满这个array part超过一半，那么新的array part的容量先变为 2^i */
        /* 这种做法保证了新的array part至少有一半容量是被占用的，提高了array part的空间利用率 */
        /*
        ** 举一个极端一些的例子，假如原来的table的array part只有key为1，2, 3跟key为17的值
        ** 当 i = 2 时，2^i / 2 = 2, nums[2] = 3, 满足 3 > 2^1 / 2，所以，到这个循环，optimal会变成4，na会变成3
        ** 继续循环 i = 2，3，..., 16, 虽然还有一个key为17，但是无法在满足 a > towtoi / 2的条件
        ** 所以，最终optimal = 4， na会变成3
        */
        optimal = twotoi;  /* optimal size (till now) */
        na = a;  /* all elements up to 'optimal' will go to array part，新的array part有a个元素 */
      }
    }
  }
  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
  *pna = na;
  return optimal;
}
```

上面这段代码，举得例子你会问，拿这个key为17的值怎么办，没办法放到新的array part里面了，是的，继续看这个函数，就能知道这个key为17的值会变到哪里去

```C
void luaH_resize (lua_State *L, Table *t, unsigned int nasize,
                                          unsigned int nhsize) {
  unsigned int i;
  int j;
  unsigned int oldasize = t->sizearray;
  int oldhsize = allocsizenode(t);
  Node *nold = t->node;  /* save old hash ... */
  if (nasize > oldasize)  /* array part must grow? */
    /* 新的array part需要更大的容量的情况 */
    setarrayvector(L, t, nasize);
  /* create new hash part with appropriate size */
  setnodevector(L, t, nhsize);
  if (nasize < oldasize) {  /* array part must shrink? */
    /* 新的array part的容量比原来小的情况 */
    t->sizearray = nasize;
    /* re-insert elements from vanishing slice */
    /* 这一步就回答了上面key为17的值去哪里的问题，落入到哈希部分了 */
    for (i=nasize; i<oldasize; i++) {
      if (!ttisnil(&t->array[i]))
        /* 
        ** luaH_setint 会调用 luaH_getint，这个函数会判断key是不是小于array part的容量，不是的话，会在哈希里面去取对应key的节点（见“lua如何通过key取对应的value”）
        ** 注意上面已经通过setnodevector分配好新的hash部分了，而且 t->sizearray = nasize也设置好新的array size了，所以setinit肯定是set到哈希部分去的了
        */
        luaH_setint(L, t, i + 1, &t->array[i]);
    }
    /* shrink array */
    luaM_reallocvector(L, t->array, oldasize, nasize, TValue);
  }
  /* re-insert elements from hash part */
  for (j = oldhsize - 1; j >= 0; j--) {
    Node *old = nold + j;
    if (!ttisnil(gval(old))) {
      /* doesn't need barrier/invalidate cache, as entry was
         already present in the table */
      setobjt2t(L, luaH_set(L, t, gkey(old)), gval(old));
    }
  }
  if (oldhsize > 0)  /* not the dummy node? */
    luaM_freearray(L, nold, cast(size_t, oldhsize)); /* free old hash */
}
```

总结一下，rehash的过程就是先统计array part + hash part中所有正整数key落在（0，1], (1, 2], (2,4],....每个区间的个数，然后根据这个统计重新计算出新的array part的容量和元素个数，以及新的hash part的容量，新的array part中，需要有超过一半的空间被占用是计算容量的核心原则，这个有可能会导致原先array part的一些元素落入到扩容后新table的hash part中。就这样
