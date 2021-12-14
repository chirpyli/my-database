### 关系代数

关系型数据库的基础是关系代数，这里总结一些关系代数的内容。为什么说在深入学习数据库之前要先学一下关系代数呢？学习关系代数是理解逻辑优化的基础。数据库由查询优化器，事务，存储引擎等组成，查询优化器是非常重要的，而查询优化一般先进行逻辑优化后，再进行物理优化。而逻辑优化的主要思想就是关系代数等价变换，通过等价变换推导出代价最小的形式。所以呢，我们这里学习一下关系代数。因为涉及到的数学公式很多，Markdown并不方便书写，因此这部分仅做一个简单梳理，详细内容还请参考《数据库系统概念》中第六章形式化关系查询语言中关系代数相关内容，或者《数据库查询优化器的艺术》中第二章逻辑查询优化章节的内容。

#### 关系代数与SQL
关系代数基本的运算符有：选择（Selection）、投影（Projection）、笛卡儿积（CartesianProduct）、连接（Join）、除（Division）、关系并（Union）、关系差（Difference）。还有一些重命名等运算符，这里不再列出。这里重点讲一下关系代数与SQL的转换，我们先举个简单查询的例子。最基本的SQL查询结构通常由SELECT、FROM、WHERE构成，其中包含了关系代数中的投影、选择和连接。
```sql
--    投影          连接        选择
SELECT projection_column FROM join_table WHERE selection_qual;
```
其中连接可以由一个基本表构成，也可以是多个基本表的连接结果，选择操作是一组针对连接操作产生的结果的表达式，对连接结果进行过滤（逻辑优化中有可能进行谓词下推优化），过滤后的元组会组成新的中间关系，最后由投影操作输出。我们看一下数据库中是如何执行的：
```sql
postgres=# explain select * from t1 where t1.a > 1;
                    QUERY PLAN                    
--------------------------------------------------
 Seq Scan on t1  (cost=0.00..1.01 rows=1 width=8)
   Filter: (a > 1)
(2 rows)
```
对应的可以参考PostgreSQL中执行器的如下代码：
```c++
/* ----------------------------------------------------------------
 *		ExecScan
 *		Scans the relation using the 'access method' indicated and
 *		returns the next qualifying tuple.
 *		The access method returns the next tuple and ExecScan() is
 *		responsible for checking the tuple returned against the qual-clause.
 * ----------------------------------------------------------------*/
TupleTableSlot *
ExecScan(ScanState *node,
		 ExecScanAccessMtd accessMtd,	/* function returning a tuple */
		 ExecScanRecheckMtd recheckMtd) {
	ExprContext *econtext;
	ExprState  *qual;
	ProjectionInfo *projInfo;

    // 省略部分代码, 只保留关键代码......
	/* * get a tuple from the access method.  Loop until we obtain a tuple that
	 * passes the qualification. */
	for (;;) {
		TupleTableSlot *slot;
		bool exec_qual_res;

		slot = ExecScanFetch(node, accessMtd, recheckMtd);  // 获取一条元组
		if (TupIsNull(slot)){
			if (projInfo)
				return ExecClearTuple(projInfo->pi_state.resultslot);
			else
				return slot;
		}

        // place the current tuple into the expr context
		econtext->ecxt_scantuple = slot;
		// check that the current tuple satisfies the qual-clause
		if (qual == NULL || ExecQual(qual, econtext)) { //判断是否满足条件，如果不满足条件不进行投影
			/* * Found a satisfactory scan tuple.*/
			if (projInfo) {
				/* * Form a projection tuple, store it in the result tuple slot and return it. */
				return ExecProject(projInfo);
			} else {
				/* * Here, we aren't projecting, so just return scan tuple.*/
				return slot;
			}
		} else
			InstrCountFiltered1(node, 1);

		/* * Tuple fails qual, so free per-tuple memory and try again. */
		ResetExprContext(econtext);
	}
}
```
