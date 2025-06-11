## PostgreSQL中List数据结构解析
PostgreSQL的List实际上是一个可扩展数组（而不是传统的链表），具有以下特点：
- 支持三种类型：
  - T_List: 指针列表（通常指向 Node）
  - T_IntList: 整数列表
  - T_OidList: Oid 列表
- 使用 ListCell 联合体来统一存储不同类型的数据。
- 支持动态扩容，自动管理内存。
- 空列表用 NIL 表示（即 NULL）。

优势：
- 节省内存，减少内存碎片，减少内存分配次数（否则每次增加元素都需要进行一次内存分配）
- 连续内存访问效率高，缓存友好
- 支持快速索引访问O(1)复杂度

List结构定义如下：
```c
typedef union ListCell
{
	void	   *ptr_value;      // 指针类型
	int			int_value;      // int类型
	Oid			oid_value;      // oid类型
} ListCell;    // 定义列表中的元素

typedef struct List
{
	NodeTag		type;			/* T_List, T_IntList, or T_OidList 列表类型 */
	int			length;			/* 列表中元素个数 */
	int			max_length;		/* elements[] 最大长度 */
	ListCell   *elements;		/* 指向实际存储元素的数组指针 */
	/* We may allocate some cells along with the List header: */
	ListCell	initial_elements[FLEXIBLE_ARRAY_MEMBER];
	/* If elements == initial_elements, it's not a separate allocation */
} List;

#define NIL						((List *) NULL)     // 空列表
```

List使用连续内存块存储元素（比传统链表更节省内存，提高缓存命中率，传统链表至少需要一个next指针），并在必要时自动扩容。扩容策略：按2的幂次增长。

### List关键实现
创建一个List，分配连续内存空间，并且在初始创建时，会额外分配一些空间（向上取整到2的幂）以减少后续可能发生的重新分配操作。
```c
static List *new_list(NodeTag type, int min_size)
{
	int max_size = pg_nextpower2_32(Max(8, min_size + LIST_HEADER_OVERHEAD));
	max_size -= LIST_HEADER_OVERHEAD;

	List *newlist = (List *) palloc(offsetof(List, initial_elements) +
							  max_size * sizeof(ListCell));    // 分配连续内存块
	newlist->type = type;
	newlist->length = min_size;
	newlist->max_length = max_size;
	newlist->elements = newlist->initial_elements;

	return newlist;
}
```

List扩容，因为这里的List并不是传统的链表，而是一个可扩展数组，所以List的扩容操作是动态的，并不是直接再增加一个节点然后更改next指针。需要重新分配一个更大的空间（当前容量的下一个2的幂次方）。），然后将原有数据复制过去。
```c
static void
enlarge_list(List *list, int min_size)
{
	/* clamp the minimum value to 16, a semi-arbitrary small power of 2 */
	int new_max_len = pg_nextpower2_32(Max(16, min_size));

	if (list->elements == list->initial_elements)  // 初始内存块的情况
	{
		/*
		 * Replace original in-line allocation with a separate palloc block.
		 * Ensure it is in the same memory context as the List header.  (The
		 * previous List implementation did not offer any guarantees about
		 * keeping all list cells in the same context, but it seems reasonable
		 * to create such a guarantee now.)
		 */
		list->elements = (ListCell *)
			MemoryContextAlloc(GetMemoryChunkContext(list),
							   new_max_len * sizeof(ListCell));  // 分配新的内存块
		memcpy(list->elements, list->initial_elements,
			   list->length * sizeof(ListCell));   // 复制原有数据

		/*
		 * We must not move the list header, so it's unsafe to try to reclaim
		 * the initial_elements[] space via repalloc.  In debugging builds,
		 * however, we can clear that space and/or mark it inaccessible.
		 * (wipe_mem includes VALGRIND_MAKE_MEM_NOACCESS.)
		 */
#ifdef CLOBBER_FREED_MEMORY
		wipe_mem(list->initial_elements,
				 list->max_length * sizeof(ListCell));
#else
		VALGRIND_MAKE_MEM_NOACCESS(list->initial_elements,
								   list->max_length * sizeof(ListCell));
#endif
	}
	else  // 非初始内存块的情况
	{
		/* 使用repalloc函数来扩展内存 */
		list->elements = (ListCell *) repalloc(list->elements,
											   new_max_len * sizeof(ListCell));
	}

	list->max_length = new_max_len;
}
```

在列表尾部追加一个元素
```c
List *lappend(List *list, void *datum)
{
	if (list == NIL)    // 如果是空列表，则创建一个新列表
		list = new_list(T_List, 1);
	else
		new_tail_cell(list);   // 确保有足够的空间

	llast(list) = datum;    // 赋值给列表的尾部元素
	check_list_invariants(list);
	return list;
}
```

### 为什么这么设计？
为什么这么设计，是针对PostgreSQL对列表这一数据结构的使用场景进行优化设计的。在PostgreSQL中，对List的使用往往不会太大，不会有一个特别长的List，也较少使用在List中间插入或删除元素，更多的场景是追加和遍历List。在PostgreSQL的原有设计上，是采用的经典链表实现的，而这样在PostgreSQL的场景下并不是最优的。下面直接引用提交该patch的讨论，清晰的说明了为什么要这么设计的原因：

多年来，人们对我们的List数据结构实现一直存在不满。由于它单独为每个列表单元分配内存（palloc），导致消耗大量内存，并且由于这些单元不一定相邻，对缓存不够友好。此外，我们通常的使用模式是通过重复的lappend操作构建列表，而从不进行修改，因此允许单独插入/删除列表单元的灵活性通常是多余的。

每次讨论这个问题时，我都认为正确的解决方案是重构List API并采用新的底层实现，正如我们曾经做过的那样（参考提交d0b4399d81）。为了证明这个观点，我提供了一个概念验证补丁，将List改为可扩展数组实现，同时保留大部分现有API。

**性能变化**
- 优势：list_nth() 成为O(1)操作，lappend() 依然高效
- 劣势：lcons() 变为O(N)，中间位置的插入/删除也变为O(N)
- 实际影响：由于极少在长列表中执行中间操作，而lcons()使用频率较低，整体收益显著。例如，list_nth()的效率提升对性能至关重要。

**兼容性挑战**
- 指针失效：修改列表可能导致单元移动，破坏依赖邻近单元指针的代码（常见于foreach循环中操作列表）。
- 解决方案：用基于整数索引的循环替代foreach，使用新的list_delete_nth_cell函数。已通过调试模式（强制列表移动）验证，但仍可能存在未发现的边界情况。

**API调整**
- lnext()变更：需要同时传递List指针和当前ListCell指针，以检测列表边界。影响了约150处代码。
- foreach宏问题：现有实现中List参数在每次迭代时被重新求值，存在潜在风险。实验性方案（通过辅助函数捕获List指针）受限于编译器优化和变量修饰符（const/volatile），可能需要牺牲部分灵活性。

**未来优化方向**
- 内存分配优化：合并列表头和数组的单一palloc，但需处理现有API对列表头不可移动的隐式依赖。
- list_qsort改进：直接对数组原地排序，需调整比较函数API（减少间接层级）。
- 代码简化：消除现有代码中与List并行的冗余数组（如优化器的simple_rte_array）。

**发布计划**
- 该改动规模较大，可能存在潜在风险，不适合v12版本。
- 目标是在v13开发周期早期推进，预留充足时间测试和修复问题。

通过这一重构，我们期望在保持代码兼容性的前提下，显著提升列表操作的性能和内存效率，同时为未来优化奠定基础。


>参考文档：[POC: converting Lists into arrays](https://www.postgresql.org/message-id/11587.1550975080@sss.pgh.pa.us)