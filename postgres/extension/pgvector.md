
pgvector是一款PostgreSQL扩展，用于在PostgreSQL中存储和检索向量数据。支持
- exact and approximate nearest neighbor search 
- single-precision, half-precision, binary, and sparse vectors
- L2 distance, inner product（内积）, cosine distance, L1 distance, Hamming distance, and Jaccard distance

### 什么是向量
我们高中数学中其实就讲到了向量的概念，比如二维平面向量可用`a=(x,y)`表示，三维空间向量可用`a=(x,y,z)`表示，而四维向量可用`a=(x,y,z,w)`表示，进而n维向量可用`a=(x1,x2,...,xn)`来表示。那么在计算机中，怎么表示向量呢？计算机中，向量可以用一个浮点数组来表示，比如三维向量可表示为[x,y,z]，四维向量可表示为[x,y,z,w]，n维向量可表示为[x1,x2,...,xn]。

我们看一下pgvector的源码，pgvector是一个扩展，它定义了一个名为`Vector`的结构体，这个结构体用来表示一个向量。
```c++
typedef struct Vector
{
	int32		vl_len_;		/* varlena header (do not touch directly!) */
	int16		dim;			/* 向量维度 */
	int16		unused;			/* reserved for future use, always zero */
	float		x[FLEXIBLE_ARRAY_MEMBER];  // 浮点数组，用来表示向量
}			Vector;
```
初始化一个向量：
```c++
Vector *InitVector(int dim)
{
	int			size = VECTOR_SIZE(dim);
	Vector	   *result = (Vector *) palloc0(size);
	SET_VARSIZE(result, size);
	result->dim = dim;

	return result;
}
```

在pgvector中，最高支持16000维的向量，这是由`VECTOR_MAX_DIM`宏定义的。
```c++
#define VECTOR_MAX_DIM 16000
```

### 安装与使用
下面我们通过示例来说明如何使用pgvector。向量数据库最重要的是解决向量存储、索引和查询等操作。
```sql
-- 安装pgvector
create extension vector; 
-- 创建一个包含3维向量数据列的表， 向量列被声明为vector(3)
CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(3));
postgres=# \d items
                               Table "public.items"
  Column   |   Type    | Collation | Nullable |              Default              
-----------+-----------+-----------+----------+-----------------------------------
 id        | bigint    |           | not null | nextval('items_id_seq'::regclass)
 embedding | vector(3) |           |          | 
Indexes:
    "items_pkey" PRIMARY KEY, btree (id)

-- 插入一些向量数据
postgres=# INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');
INSERT 0 2
-- 查询向量数据
postgres=# select * from items;
 id | embedding 
----+-----------
  1 | [1,2,3]
  2 | [4,5,6]
(2 rows)
-- 获取最接近向量[3,1,2]的向量
postgres=# SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 1;
 id | embedding 
----+-----------
  1 | [1,2,3]
(1 row)

```
支持的距离计算有：
- `<->` - L2 distance
- `<#>` - (negative) inner product
- `<=>` - cosine distance
- `<+>` - L1 distance
- `<~>` - Hamming distance (binary vectors)
- `<%>` - Jaccard distance (binary vectors)

支持的索引类型有：
- [HNSW](https://github.com/pgvector/pgvector?tab=readme-ov-file#hnsw)
- [IVFFlat](https://github.com/pgvector/pgvector?tab=readme-ov-file#ivfflat)

### 代码初步分析
代码结构日下：
```sh
── sql
│   ├── vector--0.1.0--0.1.1.sql
│   ├── vector--0.1.1--0.1.3.sql
│   ├── vector--0.1.3--0.1.4.sql
│   ├── vector--0.1.4--0.1.5.sql
│   ├── vector--0.1.5--0.1.6.sql
│   ├── vector--0.1.6--0.1.7.sql
│   ├── vector--0.1.7--0.1.8.sql
│   ├── vector--0.1.8--0.2.0.sql
│   ├── vector--0.2.0--0.2.1.sql
│   ├── vector--0.2.1--0.2.2.sql
│   ├── vector--0.2.2--0.2.3.sql
│   ├── vector--0.2.3--0.2.4.sql
│   ├── vector--0.2.4--0.2.5.sql
│   ├── vector--0.2.5--0.2.6.sql
│   ├── vector--0.2.6--0.2.7.sql
│   ├── vector--0.2.7--0.3.0.sql
│   ├── vector--0.3.0--0.3.1.sql
│   ├── vector--0.3.1--0.3.2.sql
│   ├── vector--0.3.2--0.4.0.sql
│   ├── vector--0.4.0--0.4.1.sql
│   ├── vector--0.4.1--0.4.2.sql
│   ├── vector--0.4.2--0.4.3.sql
│   ├── vector--0.4.3--0.4.4.sql
│   ├── vector--0.4.4--0.5.0.sql
│   ├── vector--0.5.0--0.5.1.sql
│   ├── vector--0.5.1--0.6.0.sql
│   ├── vector--0.6.0--0.6.1.sql
│   ├── vector--0.6.1--0.6.2.sql
│   ├── vector--0.6.2--0.7.0.sql
│   ├── vector--0.7.0--0.7.1.sql
│   ├── vector--0.7.1--0.7.2.sql
│   ├── vector--0.7.2--0.7.3.sql
│   ├── vector--0.7.3--0.7.4.sql
│   ├── vector--0.7.4--0.8.0.sql
│   └── vector.sql
├── src
│   ├── bitutils.c
│   ├── bitutils.h
│   ├── bitvec.c
│   ├── bitvec.h
│   ├── halfutils.c
│   ├── halfutils.h
│   ├── halfvec.c
│   ├── halfvec.h
│   ├── hnswbuild.c
│   ├── hnsw.c
│   ├── hnsw.h
│   ├── hnswinsert.c
│   ├── hnswscan.c
│   ├── hnswutils.c
│   ├── hnswvacuum.c
│   ├── ivfbuild.c
│   ├── ivfflat.c
│   ├── ivfflat.h
│   ├── ivfinsert.c
│   ├── ivfkmeans.c
│   ├── ivfscan.c
│   ├── ivfutils.c
│   ├── ivfvacuum.c
│   ├── sparsevec.c
│   ├── sparsevec.h
│   ├── vector.c
│   └── vector.h

```

安装pgvector扩展时，会调用`_PG_init`函数，这个函数会调用其他函数来进行相关初始化。
```c++
void _PG_init(void)
{
	BitvecInit();   // 初始化位向量（bit vector）功能，通常用于二进制特征向量的高效存储和计算。
	HalfvecInit();  // 初始化半精度浮点向量（16-bit float）支持，适用于内存敏感场景下的向量运算。
	HnswInit();     // 初始化HNSW索引
	IvfflatInit();  // 初始化IVFFlat索引
}
```

> 索引非常重要