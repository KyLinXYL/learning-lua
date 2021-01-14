[toc]

## Luna变长整数压缩编码

### 无符号整数压缩

将整数7位一组从低位开始填充到字节数组中，每个字节的第八位表示编码是否结束，第八位为1则下个字节仍属于该整数，否则该整数已经编码完毕。

这种编码的好处是，对于小整数非常节省空间，可以少编码非常多的前导零。而且对于整数数组可以连续编码到同一个buffer中，因为我们可以使用第八位来判断当前编码的结束位置。

```cpp
size_t encode_u64(unsigned char* buffer, size_t buffer_size, uint64_t value) {
    auto pos = buffer, end = buffer + buffer_size;
    do {
        if (pos >= end)
            return 0;
        auto code = (unsigned char)(value &amp; 0x7F);
        value >>= 7;
        *pos++ = code | (value > 0 ? 0x80 : 0);
    } while (value > 0);
    return (size_t)(pos - buffer);
}
```

解码代码按照设计思想反向操作即可

```cpp
size_t decode_u64(uint64_t* value, const unsigned char* data, size_t data_len) {
    auto pos = data, end = data + data_len;
    uint64_t code = 0, number = 0;
    int bits = 0;
    // 在编码时,把数据按照7bit一组一组的编码,最多10个组,也就是10个字节
    // 第1组无需移位,第2组右移7位,第3组......,第10组(其实只有1位有效)右移了63位;
    // 所以,在解码的时候,最多左移63位就结束了:)
    while (true) {
        if (pos >= end || bits > 63)
            return 0;
        code = *pos &amp; 0x7F;
        number |= (code << bits);
        if ((*pos++ &amp; 0x80) == 0)
            break;
        bits += 7;
    }
    *value = number;
    return (size_t)(pos - data);
}
```

### 有符号整数压缩

有符号整数比较特殊的地方是，负数没有前导零，高位填充的全部都是1，这样就没法发挥这个压缩算法能压缩较小（绝对值较小）整数的优势了。

这里的处理方式是，如果该整数是负数，则计算其相反数，然后左移一位把符号位存储在最低位，如果是正数就直接左移一位（此时最低位为0可以和负数区分开来），这样就可以使用原有的无符号压缩方法压缩了。解压的时候先看最低位确认符号，然后把剩余位右移一位回来，最后进行相反数计算处理。

这样设计之后，必须严格区分有符号数据和无符号数据的buffer，将一个存储了无符号数据的buffer传递到有符号的函数中来，解压出来的会是错误数据。

```cpp
size_t encode_s64(unsigned char* buffer, size_t buffer_size, int64_t value) {
    uint64_t uvalue = (uint64_t)value;
    if (value < 0) {
        --uvalue;
        uvalue = ~uvalue;
        uvalue <<= 1;
        uvalue |= 0x1;
    } else {
        uvalue <<= 1;
    }
    return encode_u64(buffer, buffer_size, uvalue);
}

size_t decode_s64(int64_t* value, const unsigned char* data, size_t data_len) {
    uint64_t uvalue = 0;
    size_t count = decode_u64(&amp;uvalue, data, data_len);
    if (count == 0)
        return 0;

    if (uvalue &amp; 0x1) {
        uvalue >>= 1;
        if (uvalue == 0) {
            uvalue = 0x1ull << 63;
        }
        uvalue = ~uvalue;
        uvalue++;
    } else {
        uvalue >>= 1;
    }

    *value = (int64_t)uvalue;
    return count;
}
```


## Luna数据序列化与反序列化

### 序列化Lua栈上的数据

函数功能：将Lua栈上first到last位置的数据序列化到buffer中

首先初始化buffer以及一系列成员变量，后面递归序列化的时候，就依靠这些成员变量来控制buffer

然后按顺序调用save_value函数序列化各个数据

最后如果数据长度超过阈值，就使用lz4压缩一遍

```cpp
void* lua_archiver::save(size_t* data_len, lua_State* L, int first, int last) {
    first = normal_index(L, first);
    last = normal_index(L, last);
    if (last < first || !alloc_buffer())
        return nullptr;

    *m_ar_buffer = 'x';
    m_begin = m_ar_buffer;
    m_end = m_ar_buffer + m_ar_buffer_size;
    m_pos = m_begin + 1;
    m_table_depth = 0;
    m_shared_string.clear();
    m_shared_strlen.clear();

    for (int i = first; i <= last; i++) {
        if (!save_value(L, i))
            return nullptr;
    }

    *data_len = (size_t)(m_pos - m_begin);
    if (*data_len >= m_lz_threshold) {
        *m_lz_buffer = 'z';
        int raw_len = ((int)*data_len) - 1;        
        int out_len = LZ4_compress_default((const char*)m_begin + 1, (char*)m_lz_buffer + 1, raw_len, (int)m_lz_buffer_size - 1);
        if (out_len > 0) {
            *data_len = 1 + out_len;
            return m_lz_buffer;
        }
    }
    return m_ar_buffer;
}
```

递归序列化的入口，根据lua类型，选择使用不同的序列化方法

后续序列化方法中，都是先存储一个字节的标志位，表示后续数据的类型，然后存储0或若干字节的压缩数据

```
bool lua_archiver::save_value(lua_State* L, int idx) {
    int type = lua_type(L, idx);
    switch (type) {
    case LUA_TNIL:
        return save_nil();

    case LUA_TNUMBER:
        return lua_isinteger(L, idx) ? save_integer(lua_tointeger(L, idx)) : save_number(lua_tonumber(L, idx));

    case LUA_TBOOLEAN:
        return save_bool(!!lua_toboolean(L, idx));

    case LUA_TSTRING:
        return save_string(L, idx);

    case LUA_TTABLE:
        return save_table(L, idx);

    default:
        break;
    }
    return false;
}
```

### 序列化实数

直接存储该实数的位表示即可

```
bool lua_archiver::save_number(double v) {
    if (m_end - m_pos < sizeof(unsigned char) + sizeof(double))
        return false;
    *m_pos++ = (unsigned char)ar_type::number;
    uint64_t ni64 = htonll(*(uint64_t*)&amp;v);
    memcpy(m_pos, &amp;ni64, sizeof(ni64));
    m_pos += sizeof(ni64);
    return true;
}
```

### 序列化整数

如果整数属于定义的小整数范围，则直接把这个整数和标志位存在一起

否则减去最大的小整数后，使用变长整数压缩存储

```
bool lua_archiver::save_integer(int64_t v) {
    if (v >= 0 &amp;&amp; v <= small_int_max) {
        if (m_end - m_pos < sizeof(unsigned char))
            return false;
        *m_pos++ = (unsigned char)(v + (int)ar_type::count);
        return true;
    }

    if (v > small_int_max) {
        v -= small_int_max;
    }

    if (m_end - m_pos < sizeof(unsigned char))
        return false;
    *m_pos++ = (unsigned char)ar_type::integer;
    size_t len = encode_s64(m_pos, (size_t)(m_end - m_pos), v);
    m_pos += len;
    return len > 0;
}
```

### 序列化布尔值

直接用标志位表示布尔的真假

```
bool lua_archiver::save_bool(bool v) {
    if (m_end - m_pos < sizeof(unsigned char))
        return false;
    *m_pos++ = (unsigned char)(v ? ar_type::bool_true : ar_type::bool_false);
    return true;
}
```

### 序列化nil

直接用标志位表示nil

```
bool lua_archiver::save_nil() {
    if (m_end - m_pos < sizeof(unsigned char))
        return false;
    *m_pos++ = (unsigned char)ar_type::nil;
    return true;
}
```

### 序列化string

因为Lua中的string是不可修改型的，lua_tolstring返回的是指向Lua环境中共享存储的字符串的指针，所以我们可以直接用这个指针地址作为字符串的唯一索引，地址一致的字符串一定一致

序列化时，取出字符串指针后，先在我们自己定义的共享字符串列表（
m_shared_string）中查找是否已经获取过一次这个指针，如果已经获取过，就直接存储这个指针的index即可，否则需要存储完整的字符串，并将这个指针加入共享字符串列表

反序列化时，读取到完整字符串应将其加入列表，读取到一个字符串指针index，则这个index一定小于当前列表长度（因为我们是按顺序序列化，按顺序反序列化的），我们一定能在列表中找到完整的字符串内容

```
bool lua_archiver::save_string(lua_State* L, int idx) {
    size_t len = 0, encode_len = 0;
    const char* str = lua_tolstring(L, idx, &amp;len);
    int shared = find_shared_str(str);
    if (shared >= 0) {
        if (m_end - m_pos < sizeof(unsigned char))
            return false;
        *m_pos++ = (unsigned char)ar_type::string_idx;
        encode_len = encode_u64(m_pos, (size_t)(m_end - m_pos), shared);
        m_pos += encode_len;
        return encode_len > 0;
    }

    if (m_end - m_pos < sizeof(unsigned char))
        return false;
    *m_pos++ = (unsigned char)ar_type::string;

    encode_len = encode_u64(m_pos, (size_t)(m_end - m_pos), len);
    if (encode_len == 0)
        return false;
    m_pos += encode_len;

    if (m_end - m_pos < (int)len)
        return false;
    memcpy(m_pos, str, len);
    m_pos += len;

    if (m_shared_string.size() < max_share_string) {
        m_shared_string.push_back(str);
    }

    return true;
}

int lua_archiver::find_shared_str(const char* str) {
    auto it = std::find(m_shared_string.begin(), m_shared_string.end(), str);
    if (it != m_shared_string.end())
        return (int)(it - m_shared_string.begin());
    return -1;
}
```

### 序列化表

首先嵌套表的深度不能超过一个设定值，因为序列化表时需要提取表的数据到Lua栈上，嵌套过深会导致栈溢出

序列化后的buffer结构如首行注释所示，前后是表开始和表结束标志位。然后是一个直接存储的lhsize数值表示实际的元素数量比narr大约多多少，如果不多或者少于则lhsize是0，如果多于则大约多2^lhsize，这个值用于反序列化时提示Lua构造表时预分配多少空间。然后是变长整数存储的narr值，这个值和Lua中使用#操作符得到的值一致。然后就是一系列key-value

序列化表中元素时，使用lua_next函数将表中元素一个一个取出放到栈上，然后继续调用save_value递归序列化

lua_next函数要求栈顶是前一个元素的key（希望取出第一个元素时，栈顶需是nil），然后它弹出栈顶元素，并依次压入下一个元素的key和value。因此，我们先序列化-2位置的key，再序列化-1位置的value，然后pop栈顶的value，下个循环的时候栈顶就是上一个key了，可以顺利取出下一个元素。

```
// table: table_head + lhsize + narr + (k,v)... + table_tail
bool lua_archiver::save_table(lua_State* L, int idx) {
    if (++m_table_depth > max_table_depth)
        return false;

    if (m_end - m_pos < (ptrdiff_t)sizeof(unsigned char) * 2)
        return false;

    idx = normal_index(L, idx);
    *m_pos++ = (unsigned char)ar_type::table_head;
    unsigned char* lhsize = m_pos++;
    uint64_t narr = (uint64_t)luaL_len(L, idx);
    size_t encode_len = encode_u64(m_pos, (size_t)(m_end - m_pos), narr);
    if (encode_len == 0)
        return false;
    m_pos += encode_len;

    if (!lua_checkstack(L, 1))
        return false;

    int size = 0;
    lua_pushnil(L);
    while (lua_next(L, idx)) {
        if (!save_value(L, -2) || !save_value(L, -1))
            return false;
        ++size;
        lua_pop(L, 1);
    }

    // 考虑数组出现空洞的情况,narr未必是准确的,可能出现narr>size的情况
    // 由于这里的hsize只是作为load时预留之用,所以这种情况下记为0
    *lhsize = (unsigned char)fast_log2((unsigned)(size > narr ? size - narr : 0));

    --m_table_depth;

    if (m_end - m_pos < (ptrdiff_t)sizeof(unsigned char))
        return false;
    *m_pos++ = (unsigned char)ar_type::table_tail;
    return true;
}
```

### 反序列化数据到Lua栈上

序列化时，如果数据过长，会触发lz4压缩，所以首先检测是否需要解压，然后持续调用load_value将数据反序列化到Lua栈上，如果中途出错，则丢弃所有数据，恢复栈顶到原来的位置。

```
int lua_archiver::load(lua_State* L, const void* data, size_t data_len) {
    if (data_len == 0 || !alloc_buffer())
        return 0;

    m_pos = (unsigned char*)data;
    m_end = (unsigned char*)data + data_len;

    if (*m_pos == 'z') {
        m_pos++;
        int len = LZ4_decompress_safe((const char*)m_pos, (char*)m_lz_buffer, (int)data_len - 1, (int)m_lz_buffer_size);
        if (len <= 0)
            return 0;
        m_pos = m_lz_buffer;
        m_end = m_lz_buffer + len;
    } else {
        if (*m_pos != 'x')
            return 0;
        m_pos++;
    }

    m_shared_string.clear();
    m_shared_strlen.clear();
    m_arr_reserve = m_max_arr_reserve;
    m_hash_reserve = m_max_hash_reserve;

    int count = 0;
    int top = lua_gettop(L);
    while (m_pos < m_end) {
        if (!load_value(L, false)) {
            lua_settop(L, top);
            return 0;
        }
        count++;
    }
    return count;
}
```

### 反序列化递归入口

反序列化时只要按照第一个标志位字节，即可确定后续数据的类型，使用对应的反序列化方法

小整数、nil、实数、布尔的长度都是固定的，整数的长度根据变长整数的字节第八位可以判断，字符串的长度写在标志位后面，表的长度由表结束标志位确定。

因为小整数的值也存在标志位里面了，需要先判断标志位是否大于小整数标志位值，特殊处理小整数，然后才能进入switch

```
bool lua_archiver::load_value(lua_State* L, bool tab_key) {
    if (!lua_checkstack(L, 1))
        return false;

    if (m_end - m_pos < (ptrdiff_t)sizeof(unsigned char))
        return false;

    int code = *m_pos++;
    if (code >= (int)ar_type::count) {
        lua_pushinteger(L, code - (int)ar_type::count);
        return true;
    }

    size_t decode_len = 0;
    uint64_t str_len = 0, str_idx = 0;

    switch ((ar_type)code) {
    case ar_type::nil:
        if (tab_key)
            return false;
        lua_pushnil(L);
        break;

    case ar_type::number: {        
        if (m_end - m_pos < (ptrdiff_t)sizeof(int64_t))
            return false;
        uint64_t i64 = 0;       
        memcpy(&amp;i64, m_pos, sizeof(i64));
        m_pos += sizeof(i64);        
        i64 = ntohll(i64);
        double f64 = *(double*)&amp;i64;
        if (tab_key &amp;&amp; isnan(f64))
            return false;
        lua_pushnumber(L, (lua_Number)f64);
        break;
    }

    case ar_type::integer: {
        int64_t integer = 0;
        decode_len = decode_s64(&amp;integer, m_pos, (size_t)(m_end - m_pos));
        if (decode_len == 0)
            return false;
        m_pos += decode_len;
        if (integer >= 0) {
            integer += small_int_max;
        }
        lua_pushinteger(L, (lua_Integer)integer);
        break;
    }

    case ar_type::bool_true:
        lua_pushboolean(L, true);
        break;

    case ar_type::bool_false:
        lua_pushboolean(L, false);
        break;

    case ar_type::string:
        decode_len = decode_u64(&amp;str_len, m_pos, (size_t)(m_end - m_pos));
        if (decode_len == 0)
            return false;
        m_pos += decode_len;
        if (str_len > (uint64_t)(m_end - m_pos))
            return false;
        m_shared_string.push_back((char*)m_pos);
        m_shared_strlen.push_back((size_t)str_len);
        lua_pushlstring(L, (char*)m_pos, (size_t)str_len);
        m_pos += str_len;
        break;

    case ar_type::string_idx:
        decode_len = decode_u64(&amp;str_idx, m_pos, (size_t)(m_end - m_pos));
        if (decode_len == 0 || str_idx >= m_shared_string.size())
            return false;
        m_pos += decode_len;
        lua_pushlstring(L, m_shared_string[(int)str_idx], m_shared_strlen[(int)str_idx]);
        break;

    case ar_type::table_head:
        return load_table(L);

    default:
        return false;
    }

    return true;
}
```

### 反序列化表

反序列化表的主要部分是计算lua_createtable的第二第三个参数，这两个参数分别提示Lua应该预分配多少空间给表中的序列部分和其他部分。

我们在序列化表时存储了两个数值，一个是narr，表示原来的表使用Lua的#运算符得到的长度，第二个是hsize（hsize=2^lhsize），表示实际元素个数比narr多大约多少个。

narr由于有空洞的关系，可能比实际的序列元素个数要多，所以要使用估计的上界来限制，同样hsize由于是2的次方表示，可能也过分高估，需要使用上界来限制。同时luna还允许限制所有的表的预分配空间之和，如果预分配空间已经超出限制则需要相应减少。

最后表的反序列化也是一个简单的递归过程即可完成，先把key和value都反序列化到栈上，然后用lua_settable插入表中

```
bool lua_archiver::load_table(lua_State* L) {
    if (m_end - m_pos < (ptrdiff_t)sizeof(unsigned char))
        return false;

    unsigned char lhsize = *m_pos++;
    uint64_t narr = 0;
    size_t decode_len = decode_u64(&amp;narr, m_pos, (size_t)(m_end - m_pos));
    if (decode_len == 0)
        return false;
    m_pos += decode_len;

    uint64_t rest_len = (uint64_t)(m_end - m_pos);
    if (rest_len < 1)
        return false;

    uint64_t max_count = (rest_len - 1) / 2;
    if (narr > max_count)
        narr = 0;

    uint64_t hsize = (lhsize > 0 &amp;&amp; lhsize < 31) ? (1ull << lhsize) : 0;
    if (hsize > max_count)
        hsize = max_count;

    int narr_i = (int)narr;
    if (m_max_arr_reserve >= 0) {
        if (narr_i > m_arr_reserve) {
            narr_i = m_arr_reserve;
        }
        m_arr_reserve -= narr_i;
    }

    int hsize_i = (int)hsize;
    if (m_max_hash_reserve >= 0) {
        if (hsize_i > m_hash_reserve) {
            hsize_i = m_hash_reserve;
        }
        m_hash_reserve -= hsize_i;
    }
    
    lua_createtable(L, narr_i, hsize_i);
    while (m_pos < m_end) {
        if (*m_pos == (unsigned char)ar_type::table_tail) {
            m_pos++;
            return true;
        }
        if (!load_value(L, true) || !load_value(L, false))
            return false;
        lua_settable(L, -3);
    }
    return false;
}
```


## Luna进行Lua/C++绑定

### 导出C++函数

原本Lua库自带的将C函数push到Lua栈上的API，仅支持push如lua_CFunction定义的函数，参数只允许有一个Lua环境指针，返回值表示返回到Lua中的返回值个数。

```cpp
typedef int (*lua_CFunction) (lua_State *L);
void lua_pushcfunction (lua_State *L, lua_CFunction f);
```

而具体的参数和返回值的传递需要通过Lua栈来操作，Lua调用C函数时，将参数按顺序压栈（栈底是第一个参数），因此C函数在取参数时，通过lua_gettop函数获得栈顶的索引，就是参数的数量（Lua栈底编号是1）。C函数返回时，将返回值按顺序压栈，C函数的返回值表示返回值的个数。

为了节约程序员的时间，少写很多将普通C函数包装成Lua规范的代码，luna提供了基于模板的包装器，能十分简单地将普通C函数包装成Lua规范的样子

```cpp
template<size_t... integers, typename return_type, typename... arg_types>
return_type call_helper(lua_State* L, return_type(*func)(arg_types...), luna_sequence<integers...>&amp;&amp;) {
    return (*func)(lua_to_native<arg_types>(L, integers + 1)...);
}

template <typename return_type, typename... arg_types>
lua_global_function lua_adapter(return_type(*func)(arg_types...)) {
    return [=](lua_State* L) {
        native_to_lua(L, call_helper(L, func, make_luna_sequence<sizeof...(arg_types)>()));
        return 1;
    };
}
```

包装器的核心在于lua_adapter函数，它接收一个任意类型的函数，在内部包装成一个符合Lua规范的匿名函数

call_helper的作用是按照原函数的参数顺序，依次调用lua_to_native从Lua栈上取出参数，然后调用原函数

最后再使用native_to_lua将函数返回值压入Lua栈中，返回1表示只有一个返回值