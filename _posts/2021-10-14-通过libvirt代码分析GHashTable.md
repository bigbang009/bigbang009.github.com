---
layout:     post
title:      通过libvirt代码分析GHashTable
subtitle:   通过libvirt代码分析GHashTable
date:       2021-10-14
author:     lwk
catalog: true
tags:
    - linux
    - GHashTable
---

本文通过阅读libvirt源码作为入口，分析了GHashTable数据结构以及初始化和插入过程。

在阅读libvirt代码的define一个domain的时候发现有下面一段代码：

```
static virDomainObjPtr
virDomainObjListFindByUUIDLocked(virDomainObjListPtr doms,
                                 const unsigned char *uuid)
{
    char uuidstr[VIR_UUID_STRING_BUFLEN];
    virDomainObjPtr obj;

    virUUIDFormat(uuid, uuidstr);        
    obj = virHashLookup(doms->objs, uuidstr);
    if (obj) {                             
        virObjectRef(obj);
        virObjectLock(obj);                
    }
    return obj;
}
```
其中有一个virHashLookup()函数。

再看下virDomainObjListPtr的结构，如下所示：
```
typedef struct _virDomainObjList virDomainObjList;
typedef virDomainObjList *virDomainObjListPtr;

struct _virDomainObjList {
    virObjectRWLockable parent;

    /* uuid string -> virDomainObj  mapping
     * for O(1), lockless lookup-by-uuid */
    GHashTable *objs;

    /* name -> virDomainObj mapping for O(1),
     * lockless lookup-by-name */
    GHashTable *objsName;
};
```

其中有个GHashTable成员，并且成员名是objs，刚好是virHashLookup()函数的其中一个参数。通过查找资料&深入到代码的底层发现，GHashTable是个hashtable数据结构，并且该数据结构主要是glib库中实现的。那接下来看下virHashLookup()函数的定义：

```

void *
virHashLookup(GHashTable *table,
              const char *name)
{
    if (!table || !name)
        return NULL;

    return g_hash_table_lookup(table, name);
}

```

每个宿主机都在daemon启动初始化驱动的时候初始化全局的GHashTable变量。函数调用流程图如下所示：

![image](https://user-images.githubusercontent.com/36918717/177042844-f0134231-6198-4de3-883b-4b09370b2bc5.png)

接下来重点看g_hash_table_new_full()函数

函数位置位于glib的glib/ghash.c文件中，注意是glib，不是glibc

```
GHashTable *
g_hash_table_new_full (GHashFunc      hash_func,
                       GEqualFunc     key_equal_func,
                       GDestroyNotify key_destroy_func,
                       GDestroyNotify value_destroy_func)
{
  GHashTable *hash_table;

  hash_table = g_slice_new (GHashTable);
  g_atomic_ref_count_init (&hash_table->ref_count);
  hash_table->nnodes             = 0;
  hash_table->noccupied          = 0;
  hash_table->hash_func          = hash_func ? hash_func : g_direct_hash;
  hash_table->key_equal_func     = key_equal_func;
#ifndef G_DISABLE_ASSERT
  hash_table->version            = 0;
#endif
  hash_table->key_destroy_func   = key_destroy_func;
  hash_table->value_destroy_func = value_destroy_func;

  g_hash_table_setup_storage (hash_table);

  return hash_table;
}
```
g_hash_table_new_full()函数就是创建并初始化一个hashtable出来，其中hash_func是一个函数，它为key创建一个hash值；key_equal_func用于比较两个key是否相等；key_destroy_func当你从hash表里删除、销毁一个条目时，glib库会自动调用它释放key所占用的内存空间，这对于key是动态分配内存的hash表来说非常有用；value_destroy_func的作用是释放value占用的内存空间；g_hash_table_setup_storage()函数主要是初始化hashtable大小、mod和mask等，具体代码如下：


```
static const gint prime_mod [] =
{
  1,          /* For 1 << 0 */
  2,
  3,
  7,
  13,
  31,
  61,
  127,
……
}
static void
g_hash_table_setup_storage (GHashTable *hash_table)
{
  gboolean small = FALSE;

  /* We want to use small arrays only if:
   *   - we are running on a system where that makes sense (64 bit); and
   *   - we are not running under valgrind.
   */

#ifdef USE_SMALL_ARRAYS
  small = TRUE;

# ifdef ENABLE_VALGRIND
  if (RUNNING_ON_VALGRIND)
    small = FALSE;
# endif
#endif

  g_hash_table_set_shift (hash_table, HASH_TABLE_MIN_SHIFT);// HASH_TABLE_MIN_SHIFT=3

  hash_table->have_big_keys = !small;
  hash_table->have_big_values = !small;

  hash_table->keys   = g_hash_table_realloc_key_or_value_array (NULL, hash_table->size, hash_table->have_big_keys);
  hash_table->values = hash_table->keys;
  hash_table->hashes = g_new0 (guint, hash_table->size);
}

static void 
g_hash_table_set_shift (GHashTable *hash_table, gint shift)
{
  hash_table->size = 1 << shift;//shift=3,hash_table->size=8
  hash_table->mod  = prime_mod [shift];//prime_mode[shift]=7

  /* hash_table->size is always a power of two, so we can calculate the mask
   * by simply subtracting 1 from it. The leading assertion ensures that
   * we're really dealing with a power of two. */

  g_assert ((hash_table->size & (hash_table->size - 1)) == 0);
  hash_table->mask = hash_table->size - 1; //mask=7
}
```

以上就完成了GHashTable的创建和初始化工作。接下来就是使用，包括查找、插入、删除和更新。我们先看下g_hash_table_lookup()
```
gpointer
g_hash_table_lookup (GHashTable    *hash_table,
                     gconstpointer  key) 
{
  guint node_index;
  guint node_hash;

  g_return_val_if_fail (hash_table != NULL, NULL);

  node_index = g_hash_table_lookup_node (hash_table, key, &node_hash);

  return HASH_IS_REAL (hash_table->hashes[node_index])
    ? g_hash_table_fetch_key_or_value (hash_table->values, node_index, hash_table->have_big_values)
    : NULL;
}

static inline guint
g_hash_table_lookup_node (GHashTable    *hash_table,
                          gconstpointer  key,
                          guint         *hash_return)
{
  guint node_index;
  guint node_hash;
  guint hash_value;
  guint first_tombstone = 0;
  gboolean have_tombstone = FALSE;
  guint step = 0;

  hash_value = hash_table->hash_func (key);//根据hash_func()计算key对应的hash值
  if (G_UNLIKELY (!HASH_IS_REAL (hash_value)))
    hash_value = 2;

  *hash_return = hash_value;

  node_index = g_hash_table_hash_to_index (hash_table, hash_value);// g_hash_table_hash_to_index()函数实现(hash_value * 11) %hash_table->mod，从初始化可以知道hash_table->mod=7，比如10%7 =3，则这个下标为3
  node_hash = hash_table->hashes[node_index];// 取出下标为3的hashs数组中的值，判断他是否被占用了

  while (!HASH_IS_UNUSED (node_hash))//node_hash不为0进入循环体，不为0表示已经被占用；为0表示未被占用，直接返回node_index即可。
    {
      /* We first check if our full hash values
       * are equal so we can avoid calling the full-blown
       * key equality function in most cases.
       */
      if (node_hash == hash_value)// //当hash值相等时，判断key是否相等，如果相等则返回下标值。
        {
          gpointer node_key = g_hash_table_fetch_key_or_value (hash_table->keys, node_index, hash_table->have_big_keys);

          if (hash_table->key_equal_func)
            {
              if (hash_table->key_equal_func (node_key, key))//判断key是否相等,相等则返回node_index，这里主要看hash_table->key_equal_func是否存在，存在则使用hashtable的key比较函数，在libvirt代码里，这里使用的key比较函数是g_str_equal，即比较两字符串是否相等。
                return node_index;
            }
          else if (node_key == key)
            {
              return node_index;
            }
        }
      else if (HASH_IS_TOMBSTONE (node_hash) && !have_tombstone) //当hash值不相等是，判断是否为tombstone
        {
          first_tombstone = node_index;
          have_tombstone = TRUE;
        }

      step++;
      node_index += step;
      node_index &= hash_table->mask;
      node_hash = hash_table->hashes[node_index];
//重点说下step++至node_hash = hash_table->hashes[node_index];这几行代码，这几行代码其实就是随机选择node_index从[0, hash_table->mask]满足条件的位置，当然node_index不是从0开始的，而是采用了开放定址法（线性探测）算法进行计算，因此顺序可能是2,4,6,1,2,0,……,这样的顺序进行遍历。
    }

  if (have_tombstone)
    return first_tombstone;

  return node_index;
}
static inline guint
g_hash_table_hash_to_index (GHashTable *hash_table, guint hash)
{
  /* Multiply the hash by a small prime before applying the modulo. This
   * prevents the table from becoming densely packed, even with a poor hash
   * function. A densely packed table would have poor performance on
   * workloads with many failed lookups or a high degree of churn. */
  return (hash * 11) % hash_table->mod;
}
```
g_hash_table_lookup_node()函数主要是接受一个key变量，他会返回这个key相对应的下标，利用这个下标node_index，可以方便的得到这个key对映的value和hash。

接下来我们看下GHashTable的插入过程，同样从libvirt代码作为入口开始。Libvirt源码文件src/util/virhash.c使用virHashAddEntry()函数进行GHashTable的插入操作。先看下libvirt调用GHashTable进行插入操作的流程：
![image](https://user-images.githubusercontent.com/36918717/177042901-4c6ff38e-7f9b-4c88-8403-c3df889ce2be.png)

```
int
virHashAddEntry(GHashTable *table, const char *name, void *userdata)
{
    if (!table || !name)
        return -1; 

    if (g_hash_table_contains(table, name)) {
        virReportError(VIR_ERR_INTERNAL_ERROR,
                       _("Duplicate hash table key '%s'"), name);//查找hastable是否已经存在name
        return -1; 
    }   

g_hash_table_insert(table, g_strdup(name), userdata);//执行插入操作，这里的key就是UUID；userdata就是vm对象
    return 0;
}
```
因此g_hash_table_insert()函数的key，value分别就是UUID和vm对象
```
gboolean
g_hash_table_insert (GHashTable *hash_table,
                     gpointer    key, 
                     gpointer    value)
{
  return g_hash_table_insert_internal (hash_table, key, value, FALSE);
}


g_hash_table_insert_internal (GHashTable *hash_table,
                              gpointer    key,
                              gpointer    value,
                              gboolean    keep_new_key)
{
  guint key_hash;
  guint node_index;

  g_return_val_if_fail (hash_table != NULL, FALSE);

  node_index = g_hash_table_lookup_node (hash_table, key, &key_hash);//查找key对应的index，具体逻辑详见上面的g_hash_table_lookup_node()函数

  return g_hash_table_insert_node (hash_table, node_index, key_hash, key, value, keep_new_key, FALSE);
}


// keep_new_key=false,reusing_key=false
static gboolean
g_hash_table_insert_node (GHashTable *hash_table,
                          guint       node_index,
                          guint       key_hash,
                          gpointer    new_key,
                          gpointer    new_value,
                          gboolean    keep_new_key,
                          gboolean    reusing_key)
{
  gboolean already_exists;
  guint old_hash;
  gpointer key_to_free = NULL;
  gpointer key_to_keep = NULL;
  gpointer value_to_free = NULL;

  old_hash = hash_table->hashes[node_index];//根据node_index获取hash值
  already_exists = HASH_IS_REAL (old_hash);//判断key是否存在hash表中

  /* Proceed in three steps.  First, deal with the key because it is the
   * most complicated.  Then consider if we need to split the table in
   * two (because writing the value will result in the set invariant
   * becoming broken).  Then deal with the value.
   *
   * There are three cases for the key:
   *
   *  - entry already exists in table, reusing key:
   *    free the just-passed-in new_key and use the existing value
   *
   *  - entry already exists in table, not reusing key:
   *    free the entry in the table, use the new key
   *
   *  - entry not already in table:
   *    use the new key, free nothing
   *
   * We update the hash at the same time...
   */
  if (already_exists)
    {
      /* Note: we must record the old value before writing the new key
       * because we might change the value in the event that the two
       * arrays are shared.
       */
      value_to_free = g_hash_table_fetch_key_or_value (hash_table->values, node_index, hash_table->have_big_values);//获取node_index所对应的value

      if (keep_new_key)//是否使用新key
        {
          key_to_free = g_hash_table_fetch_key_or_value (hash_table->keys, node_index, hash_table->have_big_keys);
          key_to_keep = new_key;
        }
      else
        {
          key_to_free = new_key;//不适用新key
          key_to_keep = g_hash_table_fetch_key_or_value (hash_table->keys, node_index, hash_table->have_big_keys);
        }
    }
  else //key不在hash表中
    {
      hash_table->hashes[node_index] = key_hash;
      key_to_keep = new_key;
    }

  /* Resize key/value arrays and split table as necessary */
  g_hash_table_ensure_keyval_fits (hash_table, key_to_keep, new_value);
  g_hash_table_assign_key_or_value (hash_table->keys, node_index, hash_table->have_big_keys, key_to_keep);

  /* Step 3: Actually do the write */
  g_hash_table_assign_key_or_value (hash_table->values, node_index, hash_table->have_big_values, new_value);

  /* Now, the bookkeeping... */
  if (!already_exists)
    {
      hash_table->nnodes++;

      if (HASH_IS_UNUSED (old_hash))
        {
          /* We replaced an empty node, and not a tombstone */
          hash_table->noccupied++;
          g_hash_table_maybe_resize (hash_table);
        }

#ifndef G_DISABLE_ASSERT    
      hash_table->version++;
#endif
    }

  if (already_exists)
    {
      if (hash_table->key_destroy_func && !reusing_key)
        (* hash_table->key_destroy_func) (key_to_free);//释放key
      if (hash_table->value_destroy_func)
        (* hash_table->value_destroy_func) (value_to_free);//释放value
    }

  return !already_exists;
}
```
以上就是对GHashTable的简要分析过程，主要是在阅读libvirt源码时看到有使用该数据结构，因此产生了兴趣，毕竟学习源码除了学习核心思想之外，另外就是学习一些关键的数据结构和算法了。另外，对于glib的hashtable阐述比较详细的文章参考http://blog.chinaunix.net/uid-24774106-id-3605760.html。




