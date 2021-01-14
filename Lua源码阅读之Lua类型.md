[toc]

## Lua类型系统基础定义

下面代码节选自lobject.h，是Lua类型的基础定义

Lua中的值被表示为TValue类型的数据结构，其包含一个Value类型的字段持有实际的值（或指向值的指针），一个int字段记录这个值的具体数据类型

Value类型是一个union，用于持有所有的Lua值，其中GCObject是所有可以被垃圾回收的类型的公共头部，包括：Lua函数、字符串、userdata、表、协程

```
# define CommonHeader   GCObject *next; lu_byte tt; lu_byte marked

struct GCObject {
  CommonHeader;
};

typedef union Value {
  GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  int b;           /* booleans */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;

# define TValuefields   Value value_; int tt_

typedef struct lua_TValue {
  TValuefields;
} TValue;
```

在lua.h中定义了Lua所有基础类型的tag值（即上述tt_字段），同时在lobject.h中约定了tag值各个位的用途：
 - 0-3位是lua.h中定义的基础tag值
 - 4-5位是子类型tag值
 - 第6位标记该类型是否能被GC

下面第一个片段展示了lua.h中的相关定义，第二个片段是lobject.h中的相关定义。

```
/*
** basic types
*/
# define LUA_TNONE     (-1)

# define LUA_TNIL      0
# define LUA_TBOOLEAN      1
# define LUA_TLIGHTUSERDATA 2
# define LUA_TNUMBER       3
# define LUA_TSTRING       4
# define LUA_TTABLE    5
# define LUA_TFUNCTION     6
# define LUA_TUSERDATA     7
# define LUA_TTHREAD       8

# define LUA_NUMTAGS       9
```
```
/* Variant tags for functions */
# define LUA_TLCL   (LUA_TFUNCTION | (0 << 4))  /* Lua closure */
# define LUA_TLCF   (LUA_TFUNCTION | (1 << 4))  /* light C function */
# define LUA_TCCL   (LUA_TFUNCTION | (2 << 4))  /* C closure */

/* Variant tags for strings */
# define LUA_TSHRSTR    (LUA_TSTRING | (0 << 4))  /* short strings */
# define LUA_TLNGSTR    (LUA_TSTRING | (1 << 4))  /* long strings */


/* Variant tags for numbers */
# define LUA_TNUMFLT    (LUA_TNUMBER | (0 << 4))  /* float numbers */
# define LUA_TNUMINT    (LUA_TNUMBER | (1 << 4))  /* integer numbers */


/* Bit mark for collectable types */
# define BIT_ISCOLLECTABLE  (1 << 6)

/* mark a tag as collectable */
# define ctb(t)       ((t) | BIT_ISCOLLECTABLE)
```


## 表

### 基本定义

表相关定义在lobject.h中

```
typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;  /* log2 of size of 'node' array */
  unsigned int sizearray;  /* size of 'array' array */
  TValue *array;  /* array part */
  Node *node;
  Node *lastfree;  /* any free position is before this position */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```

Lua的表包含两种数据，一种是整数索引的数据，存放在array字段指向的数组中，另一种是hash方式存储的key-value数据，存放在node字段指向的数组中。

下面是表的hash节点的定义

```
typedef union TKey {
  struct {
    TValuefields;
    int next;  /* for chaining (offset for next node) */
  } nk;
  TValue tvk;
} TKey;


/* copy a value into a key without messing up field 'next' */
# define setnodekey(L,key,obj) \
   { TKey *k_=(key); const TValue *io_=(obj); \
     k_->nk.value_ = io_->value_; k_->nk.tt_ = io_->tt_; \
     (void)L; checkliveness(L,io_); }


typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;
```

可以看到，每个节点由一个Lua值和一个Key组成，其中Key也包含一个Lua值表示Key的值，以及一个next偏移量，用于记录冲突时去距离该节点多少偏移的下一个节点继续查找。

因此，整张哈希表就是一个巨大的数组，查找时首先计算key的hash值，索引到所谓main position上的node，对比后如果key值不相等，再去main position + next位置上的node继续比较。

### 冲突处理

插入时的冲突处理如下：
 - 首先14行计算出main position
 - 17-22行，如果该位置已经有别的元素占据了，我们需要从数组尾部获取一个空闲槽位或者没有空闲槽位则需要resize后再次插入
 - 24-25行，判断原本放在main position位置的元素，它的key计算hash得到的位置是不是也是这里，因为有可能是本来应该放在别的main position的元素由于冲突临时放在这里
 - 26-36行，处理冲突元素的main position不是本位置的状况，需要将它移动到空闲槽位，然后修改它的父节点的next值（它一定有父节点，因为它的main position不是这）
 - 37-44行，如果冲突元素本来也属于这，那就把新元素加到空闲槽位，然后链接空闲槽位到main position后面，原本在main position后面的元素就链接到空闲槽位后面

```
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
  mp = mainposition(t, key);
  if (!ttisnil(gval(mp)) || isdummy(t)) {  /* main position is taken? */
    Node *othern;
    Node *f = getfreepos(t);  /* get a free place */
    if (f == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      return luaH_set(L, t, key);  /* insert key into grown table */
    }
    lua_assert(!isdummy(t));
    othern = mainposition(t, gkey(mp));
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */
      while (othern + gnext(othern) != mp)  /* find previous */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* correct 'next' */
        gnext(mp) = 0;  /* now 'mp' is free */
      }
      setnilvalue(gval(mp));
    }
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }
  setnodekey(L, &mp->i_key, key);
  luaC_barrierback(L, t, key);
  lua_assert(ttisnil(gval(mp)));
  return gval(mp);
}
```

### 序列与Hash的平衡关系

在Lua中，并不是所有的整数Key都会存放在array数组中，因为允许空洞的存在，如果所有整数Key都在array中，那么array可能会存在大量的nil浪费空间。Lua在每次对表resize时，会通过检测每一段空间中实际存在的数据数量，决定分配多长的array数组，多出来的数据就存储到hash形式的node数组中。参考下面的rehash函数：

```
/*
** nums[i] = number of keys 'k' where 2^(i - 1) < k <= 2^i
*/
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
  na = numusearray(t, nums);  /* count keys in array part */
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* count keys in hash part */
  /* count extra key */
  na += countint(ek, nums);
  totaluse++;
  /* compute new size for array part */
  asize = computesizes(nums, &na);
  /* resize the table to new computed sizes */
  luaH_resize(L, t, asize, totaluse - na);
}
```

变量na存放了以整数索引的数据的总数，变量totaluse是整个表中数据的总数，nums[i]表示以整数索引的数据中，索引值属于范围 ( 2^(i - 1) , 2^i ]的数据数量，通过numusearray和numusehash可以分别计算当前array和node数组中这些数据的情况，最后通过computesizes函数计算array和node的长度的最优分配方式

```
/*
** Compute the optimal size for the array part of table 't'. 'nums' is a
** "count array" where 'nums[i]' is the number of integers in the table
** between 2^(i - 1) + 1 and 2^i. 'pna' enters with the total number of
** integer keys in the table and leaves with the number of keys that
** will go to the array part; return the optimal size.
*/
static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
  int i;
  unsigned int twotoi;  /* 2^i (candidate for optimal size) */
  unsigned int a = 0;  /* number of elements smaller than 2^i */
  unsigned int na = 0;  /* number of elements to go to array part */
  unsigned int optimal = 0;  /* optimal size for array part */
  /* loop while keys can fill more than half of total size */
  for (i = 0, twotoi = 1; *pna > twotoi / 2; i++, twotoi *= 2) {
    if (nums[i] > 0) {
      a += nums[i];
      if (a > twotoi/2) {  /* more than half elements present? */
        optimal = twotoi;  /* optimal size (till now) */
        na = a;  /* all elements up to 'optimal' will go to array part */
      }
    }
  }
  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
  *pna = na;
  return optimal;
}
```

computesizes函数的思想是，array数组的nil率不超过50%，同时array也要尽可能长能容纳更多的序列数据。因为我们提供了每个2的幂的区间内有效数据的数量，只要我们array数组的长度也是2的幂，就可以找到一个最长的，同时符合nil率不超过50%的长度

### 长度操作符

在Lua官方文档中有提到，Lua的长度操作符\#不应该作用于包含空洞的表，对包含空洞的表使用该操作符返回的值没有任何规定。我们通过ltable.c中求长度的操作实现，可以看到这个操作符的不稳定性：

```
/*
** Try to find a boundary in table 't'. A 'boundary' is an integer index
** such that t[i] is non-nil and t[i+1] is nil (and 0 if t[1] is nil).
*/
int luaH_getn (Table *t) {
  unsigned int j = t->sizearray;
  if (j > 0 && ttisnil(&t->array[j - 1])) {
    /* there is a boundary in the array part: (binary) search for it */
    unsigned int i = 0;
    while (j - i > 1) {
      unsigned int m = (i+j)/2;
      if (ttisnil(&t->array[m - 1])) j = m;
      else i = m;
    }
    return i;
  }
  /* else must find a boundary in hash part */
  else if (isdummy(t))  /* hash part is empty? */
    return j;  /* that is easy... */
  else return unbound_search(t, j);
}
```

可以看到，这个操作符的实质是寻找一个“边界”，在这个边界i上，t[i]不是nil但t[i+1]是nil。这个算法的复杂之处在于，表中的序列部分t并不是一个完整的数组，它的一部分存在array数组中，另一部分存在hash表示的node数组中。

因此，算法先判断array数组的最后一个元素是不是nil，因为我们假定不包含空洞，所以这个边界的左边一定全部不是nil，右边则全部是nil。如果最后一个元素是nil，则边界一定在array数组中，否则使用unbound_search函数去node数组中查找。

既然假定了边界左边全部不是nil，右边全部是nil，那么就可以使用二分查找来找到这个边界，如上述片段的8-16行所示。

unbound_search函数的代码片段如下所示，它首先遍历从j开始的每个2的幂次是不是nil来确定一个搜索的上界，同时搜索的下界就是上一个遍历到的幂（因为上一个幂已知一定不是nil）。另外注意到j是array数组的长度，这个长度根据分配算法来看也一定是2的幂次。

最后找到的搜索区间就是[i,j]，j是第一个数据为nil的2的幂次索引值，i是幂比j小1的2的幂次，在这个区间上进行二分查找就可以找到边界了。

```
static int unbound_search (Table *t, unsigned int j) {
  unsigned int i = j;  /* i is zero or a present index */
  j++;
  /* find 'i' and 'j' such that i is present and j is not */
  while (!ttisnil(luaH_getint(t, j))) {
    i = j;
    if (j > cast(unsigned int, MAX_INT)/2) {  /* overflow? */
      /* table was built with bad purposes: resort to linear search */
      i = 1;
      while (!ttisnil(luaH_getint(t, i))) i++;
      return i - 1;
    }
    j *= 2;
  }
  /* now do a binary search between them */
  while (j - i > 1) {
    unsigned int m = (i+j)/2;
    if (ttisnil(luaH_getint(t, m))) j = m;
    else i = m;
  }
  return i;
}
```

理解了上述长度运算符的实现，就可以知道，对一个有空洞的表进行长度操作，会在很多地方得到预期外的结果，比如array数组的最后一个值恰好是空洞，那算法就会错误地只搜索array数组。或是在二分查找的时候，错误地在某个空洞上停止搜索。

因此，千万不要对有空洞的表使用长度运算符。


## 字符串

### 基本定义

`lobject.h`
```
typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */
    struct TString *hnext;  /* linked list for hash table */
  } u;
} TString;

typedef union UTString {
  L_Umaxalign dummy;  /* ensures maximum alignment for strings */
  TString tsv;
} UTString;
```

Lua字符串的结构如上所示，UTString是字符串的头部，字符串的内容放置在头部的后面。UTString里包含一个用于字节对齐的字段dummy和字符串的主要结构TString。

TString是一个可以GC的对象，包含通用头部CommonHeader。

Lua字符串对长短字符串有不同的存储策略，当前版本（5.3.4）超过40字节的字符串为长字符串。

extra字段在短字符串里用于标记该字符串是哪个Lua关键字，在长字符串里用于标记该字符串的hash是否已经计算过

shrlen用于存储短字符串的长度

hash用于存储字符串的hash，长字符串的hash可能还没计算

u是一个union，长字符串使用lnglen字段表示字符串长度，短字符串用hnext来记录其在全局字符串表中的下一个字符串

### 全局字符串表与长短字符串

`lstate.h`

```
typedef struct stringtable {
  TString **hash;
  int nuse;  /* number of elements */
  int size;
} stringtable;

typedef struct global_State {
 /* fold */
  stringtable strt;  /* hash table for strings */
 /* fold */
} global_State;
```

在Lua的全局环境中有一个字符串hash表，用于记录当前已创建的所有短字符串，结构如stringtable所示。hash字段是一个字符串指针数组，它的长度记录在size中，整个hash表中的元素个数记录在nuse中，hash表采用开链形式构造。

`lstring.h`
```
TString *luaS_newlstr (lua_State *L, const char *str, size_t l) {
  if (l <= LUAI_MAXSHORTLEN)  /* short string? */
    return internshrstr(L, str, l);
  else {
    TString *ts;
    if (l >= (MAX_SIZE - sizeof(TString))/sizeof(char))
      luaM_toobig(L);
    ts = luaS_createlngstrobj(L, l);
    memcpy(getstr(ts), str, l * sizeof(char));
    return ts;
  }
}

/*
** checks whether short string exists and reuses it or creates a new one
*/
static TString *internshrstr (lua_State *L, const char *str, size_t l) {
  TString *ts;
  global_State *g = G(L);
  unsigned int h = luaS_hash(str, l, g->seed);
  TString **list = &g->strt.hash[lmod(h, g->strt.size)];
  lua_assert(str != NULL);  /* otherwise 'memcmp'/'memcpy' are undefined */
  for (ts = *list; ts != NULL; ts = ts->u.hnext) {
    if (l == ts->shrlen &&
        (memcmp(str, getstr(ts), l * sizeof(char)) == 0)) {
      /* found! */
      if (isdead(g, ts))  /* dead (but not collected yet)? */
        changewhite(ts);  /* resurrect it */
      return ts;
    }
  }
  if (g->strt.nuse >= g->strt.size && g->strt.size <= MAX_INT/2) {
    luaS_resize(L, g->strt.size * 2);
    list = &g->strt.hash[lmod(h, g->strt.size)];  /* recompute with new size */
  }
  ts = createstrobj(L, l, LUA_TSHRSTR, h);
  memcpy(getstr(ts), str, l * sizeof(char));
  ts->shrlen = cast_byte(l);
  ts->u.hnext = *list;
  *list = ts;
  g->strt.nuse++;
  return ts;
}
```
luaS_newlstr函数创建字符串时，按照字符串长度分短字符串和长字符串分别处理。

短字符串使用internshrstr函数在全局字符串表中查找，如果找不到则创建一个新的短字符串加入该表并返回。

全局字符串表的查询和插入就是普通的开链hash表处理方式，按hash值找到对应桶，然后遍历链表一一对比字符串是否相等即可。查询的时候注意把即将被GC回收的字符串标记回使用状态。插入的时候如果hash表中元素数量大于等于桶的数量，则需要翻倍扩表，然后构造一个Lua字符串对象，将字符串内容copy到对象头部后面，最后将这个新建的字符串插入对应桶的链表头部即可。

而长字符串则不在全局字符串表中管理，每次都是新建一个字符串，所以长字符串在Lua中的地址不唯一。

### 长短字符串的表索引区别

`ltable.h`
```
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

在表查询的总入口处我们可以发现，短字符串使用的方法是luaH_getshortstr，而长字符串使用的是getgeneric，两个函数在对hash表的使用上流程差不多，都是先根据hash找到main position，然后遍历链表找到相等的字符串。区别在于长短字符串的hash计算方法和相等判定方法不一样。

```
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

下面展示了长短字符串的hash和equal的区别。

首先短字符串由于创建时需要在全局字符串表中保存，所以它的hash一定是计算过的，直接取hash字段即可。但长字符串只有用到hash的时候才会去计算，所以需要先判断extra字段，然后再计算，才能确保hash字段的值有效。

短字符串由于创建时使用全局字符串表保存，确保了全局唯一性，所以相等的判断只需要判断地址一致即可。而长字符串相同时地址不保证唯一，需要使用内存比对的方式对比两个字符串的值。

`短字符串的hash和equal计算方法`
```
#define hashstr(t,str)     hashpow2(t, (str)->hash)

#define eqshrstr(a,b)   check_exp((a)->tt == LUA_TSHRSTR, (a) == (b))
```
`长字符串的hash和equal计算方法`
```
unsigned int luaS_hashlongstr (TString *ts) {
  lua_assert(ts->tt == LUA_TLNGSTR);
  if (ts->extra == 0) {  /* no hash? */
    ts->hash = luaS_hash(getstr(ts), ts->u.lnglen, ts->hash);
    ts->extra = 1;  /* now it has its hash */
  }
  return ts->hash;
}

int luaS_eqlngstr (TString *a, TString *b) {
  size_t len = a->u.lnglen;
  lua_assert(a->tt == LUA_TLNGSTR && b->tt == LUA_TLNGSTR);
  return (a == b) ||  /* same instance or... */
    ((len == b->u.lnglen) &&  /* equal length and ... */
     (memcmp(getstr(a), getstr(b), len) == 0));  /* equal contents */
}
```

由此可见，Lua的长短字符串处理造成了一些差异，由于假设长字符串不太容易重复，且不用作索引，所以创建时没有使用全局字符串表来保证唯一性，这样提高了创建字符串的效率。而短字符串常常重复且经常用作索引，在hash计算和equal计算上具有极高的性能，提高了表的效率。

我们在使用的时候也需要注意到这种区别，比如表的索引不要使用长字符串，实在需要使用长字符串时，Lua设定的40字节长度其实很巧妙，恰好是SHA1以16进制格式表示的长度（以二进制表示的话20字节就足够了），我们可以对原字符串求SHA1之后，用SHA1值来当做表的Key。
