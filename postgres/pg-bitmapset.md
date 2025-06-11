## PostgreSQL数据结构 —— 位图集合 Bitmapset


位图集合bitmapset可以表示任意非负整数集合，其主要设计用途是处理最大值较小的集合（比如最多数百），保证内存和性能效率。

在稀疏大数值场景时，应避免使用位图集合，以为采用位图集合存储稀疏大数值时，会占用大量内存，并不是其最优场景。

为什么只能表示非负整数集合呢？因为位图中的元素数值是通过位索引表示的，即元素在位图中的第几位表示值为多少，所以只能表示非负整数集合，无法表示负数。

### 位图集合的原理

位图集合的实现原理就是用一个位来表示一个整数元素，位索引为元素值，位值为1表示元素存在，位值为0表示元素不存在。那总共有多少位呢？通过一个`uint64`数组来表示，共计能表示`nwords*64`个元素。

```c++
#define BITS_PER_BITMAPWORD 64
typedef uint64 bitmapword;		/* must be an unsigned type */
typedef int64 signedbitmapword; /* must be the matching signed type */

typedef struct Bitmapset
{
	int			nwords;			/* 位图字数组的长度 */
	bitmapword	words[FLEXIBLE_ARRAY_MEMBER];	/* 实际的位图表示 */
} Bitmapset;
```

### 位索引计算

`Bitmapset`使用两个宏定义进行位索引计算，也就是计算一个元素在`Bitmapset`的位置：
- `WORDNUM(x)`：计算整数`x`所在的字索引
- `BITNUM(x)`：计算整数`x`在字内的位索引

```c
#define WORDNUM(x)	((x) / BITS_PER_BITMAPWORD)   
#define BITNUM(x)	((x) % BITS_PER_BITMAPWORD)
```

那么位图所占用的内存大小是多少呢？
```c
#define BITMAPSET_SIZE(nwords)	\
	(offsetof(Bitmapset, words) + (nwords) * sizeof(bitmapword))
```

如果原先分配的位图大小不够怎么办呢？会自动进行扩展。



###  位图的使用
Bitmapset提供了如下接口：
```c
// 复制位图
extern Bitmapset *bms_copy(const Bitmapset *a); 
 // 比较位图是否相等
extern bool bms_equal(const Bitmapset *a, const Bitmapset *b); 
// 比较位图，相等为0，否则最高位不同将决定结果（该位一个位1，一个为0）
extern int	bms_compare(const Bitmapset *a, const Bitmapset *b); 
// 创建一个包含单个成员x的位图集合
extern Bitmapset *bms_make_singleton(int x);
extern void bms_free(Bitmapset *a);   // 释放位图集合的内存
// 集合并集
extern Bitmapset *bms_union(const Bitmapset *a, const Bitmapset *b);
// 集合交集
extern Bitmapset *bms_intersect(const Bitmapset *a, const Bitmapset *b);
// 集合差集
extern Bitmapset *bms_difference(const Bitmapset *a, const Bitmapset *b);
// 判断a集合是否是b集合的子集
extern bool bms_is_subset(const Bitmapset *a, const Bitmapset *b);
// 比较俩个位图集合的包含关系，
extern BMS_Comparison bms_subset_compare(const Bitmapset *a, const Bitmapset *b);
// 判断集合a中是否包含成员x
extern bool bms_is_member(int x, const Bitmapset *a);
// 在位图中查找成员x在位图集合的位置（属于位图集合中的第几个元素）
extern int	bms_member_index(Bitmapset *a, int x);
// 判断两个位图集合（Bitmapset）是否存在重叠（即是否有非空交集）
extern bool bms_overlap(const Bitmapset *a, const Bitmapset *b);
// 检测整数列表b是否与位图集合a有重叠
extern bool bms_overlap_list(const Bitmapset *a, const struct List *b);
// 判断a - b 是否为空，即a中是否有不在b中的元素
extern bool bms_nonempty_difference(const Bitmapset *a, const Bitmapset *b);
// 若集合仅含一个元素，返回该元素
extern int	bms_singleton_member(const Bitmapset *a);
// 安全版单元素检查，通过指针返回元素值，并返回是否成功
extern bool bms_get_singleton_member(const Bitmapset *a, int *member);
// 计算集合中元素的数量
extern int	bms_num_members(const Bitmapset *a);
// 返回集合的成员状态（空集、单元素、多元素），用于快速判断
extern BMS_Membership bms_membership(const Bitmapset *a);
// 判断集合是否为空
extern bool bms_is_empty(const Bitmapset *a);
//  添加元素x到集合a中
extern Bitmapset *bms_add_member(Bitmapset *a, int x);
// 删除集合a中的元素x
extern Bitmapset *bms_del_member(Bitmapset *a, int x);
// 将集合b中的元素添加到集合a中
extern Bitmapset *bms_add_members(Bitmapset *a, const Bitmapset *b);
// 向集合中添加一个整数区间 [lower, upper] 内的所有元素
extern Bitmapset *bms_add_range(Bitmapset *a, int lower, int upper);
// 计算a和b的交集，并在a的基础上修改，返回结果
extern Bitmapset *bms_int_members(Bitmapset *a, const Bitmapset *b);
// 计算a和b的差集，并在a的基础上修改，返回结果
extern Bitmapset *bms_del_members(Bitmapset *a, const Bitmapset *b);
// 集合a和集合b的并集，在a或b中选择大的集合进行修改，返回结果
extern Bitmapset *bms_join(Bitmapset *a, Bitmapset *b);

/* 获取并移除第一个成员 */
extern int	bms_first_member(Bitmapset *a);
// 获取下一个大于指定值的成员
extern int	bms_next_member(const Bitmapset *a, int prevbit);
// 获取下一个小于指定值的成员
extern int	bms_prev_member(const Bitmapset *a, int prevbit);

/* 位集合生成哈希值 */
extern uint32 bms_hash_value(const Bitmapset *a);
// 通用的哈希适配函数，将 Bitmapset 转换为哈希键
extern uint32 bitmap_hash(const void *key, Size keysize);
// 比较两个 Bitmapset 是否相等
extern int	bitmap_match(const void *key1, const void *key2, Size keysize);
```

### 部分关键代码
我们看一下`bms_make_singleton`的实现，标识创建一个包含单个成员x的位图集合，核心操作就是创建一个足够大的位图，并设置x位置为1。
```c
Bitmapset *bms_make_singleton(int x)
{
	int wordnum = WORDNUM(x);    // 获取x所在的word编号
	int bitnum = BITNUM(x);      // 获取x所在的bit编号
	Bitmapset  *result = (Bitmapset *) palloc0(BITMAPSET_SIZE(wordnum + 1));   // 申请足够大的内存
	result->nwords = wordnum + 1; 
	result->words[wordnum] = ((bitmapword) 1 << bitnum); // 将对应位置为1
	return result;
}
```

添加一个元素到位图集合中，添加元素很简单，找到元素在位图集合中的位置，将位置对应的bit置1，其中还存在一个问题，就是原有的位图空间大小是否足够，如果不够，需要扩展位图大小。
```c
Bitmapset *bms_add_member(Bitmapset *a, int x)
{
	if (a == NULL)
		return bms_make_singleton(x);  // 如果集合为空，则创建一个包含单个元素的集合
	int wordnum = WORDNUM(x);  // 计算位图索引位置
	int bitnum = BITNUM(x);

	/* 判断是否需要扩展位图大小 */
	if (wordnum >= a->nwords)
	{
		int			oldnwords = a->nwords;
		int			i;
        //  扩展位图大小
		a = (Bitmapset *) repalloc(a, BITMAPSET_SIZE(wordnum + 1));
		a->nwords = wordnum + 1;
		/* 将新扩展的空间进行置0 */
		for (i = oldnwords; i < a->nwords; i++)
			a->words[i] = 0;
	}

	a->words[wordnum] |= ((bitmapword) 1 << bitnum); // 将对应位置为1
	return a;
}
```

我们再看一下位图集合并集是如何实现的
1. 识别较长和较短的输入集合
2. 复制较长的集合作为结果
3. 将较短集合的位与结果进行或运算

```c
Bitmapset *bms_union(const Bitmapset *a, const Bitmapset *b)
{
	Bitmapset  *result;
	const Bitmapset *other;
	int			otherlen;
	int			i;

	if (a == NULL)   // 如果a为空，则返回b的副本
		return bms_copy(b);
	if (b == NULL)   // 如果b为空，则返回a的副本
		return bms_copy(a);

    // 识别较长和较短的输入集合，复制较长的集合作为结果
	if (a->nwords <= b->nwords)
	{
		result = bms_copy(b);
		other = a;
	}
	else
	{
		result = bms_copy(a);
		other = b;
	}
	/* 将较短集合的位与结果进行或运算 */
	otherlen = other->nwords;
	for (i = 0; i < otherlen; i++)
		result->words[i] |= other->words[i];  // 位或运算
	return result;
}
```

返回位图的成员个数，实质上就是计算所有位为1的个数，怎么统计才更高效呢？
```c
int bms_num_members(const Bitmapset *a)
{
	int			result = 0;

	if (a == NULL)
		return 0;
	int nwords = a->nwords;
	for (int wordnum = 0; wordnum < nwords; wordnum++)
	{
		bitmapword	w = a->words[wordnum];

		/* 只统计非零位，值为0无需统计 */
		if (w != 0)
			result += bmw_popcount(w);
	}
	return result;
}
```
这里有个细节，我们可以看一下PostgreSQL中是如何高效计算一个64位整数二进制表示汇总1的位数的实现的。是直接利用硬件提供的指令计算的。
```c
// 输入一个64位无符号数，返回其中二进制位为1的个数
static int pg_popcount64_asm(uint64 word)
{
	uint64		res;

__asm__ __volatile__(" popcntq %1,%0\n":"=q"(res):"rm"(word):"cc");
	return (int) res;
}
```
`__asm__`表示为汇编语言，`__volatile__`告诉编译器不要优化这段代码，调用CPU的`popcntq`指令对输入的`word`统计其二进制位为1的个数。`"=q"(res)`表示输出操作数（`=`表示这是一个输出操作，`q`表示一个常规的寄存器就可以），`"rm"(word)`表示输入操作数（`rm`表示CPU选择寄存器或者内存都可以Rigister or Memory）。`%1,%0`表示占位符。


其他运算实现原理类似，这里不再进行赘述。

### 总结

位图集合是一种常用的数据结构，在某些特殊场景下非常高效，其设计思想和BloomFilter、桶排序等类似，在PostgreSQL中也被大量的使用。