---
layout: post
title:  "Nanite: Build Nanite Pages"
date:   2023-05-16 01:02:03 +800
category: Unreal Engine
---

- [1. 找对向共享边](#1-找对向共享边)
	- [1.1. 对向共享边](#11-对向共享边)
	- [1.2. 构建 EdgeHash](#12-构建-edgehash)
		- [1.2.1. Cycle3](#121-cycle3)
		- [1.2.2. FEdgeHash::Add\_Concurrent](#122-fedgehashadd_concurrent)
		- [1.2.3. FHashTable](#123-fhashtable)
			- [1.2.3.1 桶数组 `Hash`](#1231-桶数组-hash)
			- [1.2.3.2. 链表数组 `NextIndex`](#1232-链表数组-nextindex)
	- [1.3. 根据 EdgeHash 统计对向共享边](#13-根据-edgehash-统计对向共享边)
- [2. 根据对向共享边信息构建三角形 Mesh 的连通关系](#2-根据对向共享边信息构建三角形-mesh-的连通关系)
	- [2.1. `FDisjointSet::Init`](#21-fdisjointsetinit)
	- [2.2. `FDisjointSet::Union`](#22-fdisjointsetunion)
	- [2.3. `FDisjointSet::UnionSequential`](#23-fdisjointsetunionsequential)
	- [2.4. `FDisjointSet::Find`](#24-fdisjointsetfind)
- [3. 划分 Clusters](#3-划分-clusters)
	- [3.1. 创建图划分器](#31-创建图划分器)
	- [3.2. 构建空间局部邻接信息 `FGraphPartitioner::BuildLocalityLinks`](#32-构建空间局部邻接信息-fgraphpartitionerbuildlocalitylinks)
		- [3.2.1. 计算 Morton 编码](#321-计算-morton-编码)
		- [3.2.2. 通过 Morton 编码排序所有的三角形 Mesh](#322-通过-morton-编码排序所有的三角形-mesh)
		- [3.2.3. 构建 IslandRuns](#323-构建-islandruns)
		- [3.2.4. 对于所属连通块区间元素数量小于 128 的三角形 Mesh ，构建它们的空间局部邻接信息](#324-对于所属连通块区间元素数量小于-128-的三角形-mesh-构建它们的空间局部邻接信息)
	- [3.3. 创建图对象（FGraphData）](#33-创建图对象fgraphdata)
		- [3.3.1. 一个例子](#331-一个例子)
	- [3.4. 使用 METIS 对构建好的图进行递归二分划分子 Partitions](#34-使用-metis-对构建好的图进行递归二分划分子-partitions)
		- [3.4.1. METIS](#341-metis)
		- [3.4.2. 核心函数 `FGraphPartitioner::BisectGraph`](#342-核心函数-fgraphpartitionerbisectgraph)
	- [3.5. 根据划分的结果 `Partitioner.Ranges` 创建叶子 Clusters](#35-根据划分的结果-partitionerranges-创建叶子-clusters)
		- [3.5.1 收集 Cluster 的 ExternalEdges](#351-收集-cluster-的-externaledges)
		- [3.5.2. 计算 Cluster 的空间包围盒与几何尺度度量](#352-计算-cluster-的空间包围盒与几何尺度度量)
			- [3.5.2.1. `Cluster::Bounds`](#3521-clusterbounds)
			- [3.5.2.2. `Cluster::SphereBounds`](#3522-clusterspherebounds)
			- [3.5.2.3. `Cluster::SurfaceArea`](#3523-clustersurfacearea)
			- [3.5.2.4. `Cluster::EdgeLength`](#3524-clusteredgelength)
- [4. 构建 Clusters LOD 层级结构](#4-构建-clusters-lod-层级结构)
	- [4.1. `FClusterDAG::ReduceGroup`](#41-fclusterdagreducegroup)
		- [4.1.1. 简化 `FCluster::Simplify`](#411-简化-fclustersimplify)
			- [4.1.1.1 统计 UV 面积 + 标记 UV 镜像](#4111-统计-uv-面积--标记-uv-镜像)
			- [4.1.1.2. 模型尺度归一化](#4112-模型尺度归一化)
			- [4.1.1.3. 计算属性权重 AttributeWeights](#4113-计算属性权重-attributeweights)
			- [4.1.1.4. 构建 `FMeshSimplifier`](#4114-构建-fmeshsimplifier)
			- [4.1.1.5. 锁定 `ExternalEdges`](#4115-锁定-externaledges)
			- [4.1.1.6. Cluster 简化 `FMeshSimplifier::Simplify`](#4116-cluster-简化-fmeshsimplifiersimplify)
				- [4.1.1.6.1. `FMeshSimplifier::FixUpTri`](#41161-fmeshsimplifierfixuptri)
				- [4.1.1.6.2. `FMeshSimplifier::CalcEdgeQuadric`](#41162-fmeshsimplifiercalcedgequadric)
				- [4.1.1.6.3. `FMeshSimplifier::EvaluateMerge`](#41163-fmeshsimplifierevaluatemerge)
			- [4.1.1.7. 材质面积丢失补偿 `FMeshSimplifier::PreserveSurfaceArea`](#4117-材质面积丢失补偿-fmeshsimplifierpreservesurfacearea)
			- [4.1.1.8. 重建简化后的 Cluster 网格数据](#4118-重建简化后的-cluster-网格数据)
		- [4.1.2. 切分父 Cluster `SplitCluster`](#412-切分父-cluster-splitcluster)
- [5. 基于 `Settings.KeepPercentTriangles` 或 `Settings.TrimRelativeError` 做裁剪](#5-基于-settingskeeppercenttriangles-或-settingstrimrelativeerror-做裁剪)
- [6. 编码数据并写入 Page](#6-编码数据并写入-page)
	- [6.1. 顶点数据清洗（ Sanitize ）](#61-顶点数据清洗-sanitize-)
	- [6.2. 构建材质 Range ，并重排序三角形 Mesh](#62-构建材质-range-并重排序三角形-mesh)
		- [6.2.1. 为什么要把三角形 Mesh 聚类为大段连续的 Range](#621-为什么要把三角形-mesh-聚类为大段连续的-range)
	- [6.3. 对 Cluster 添加硬约束](#63-对-cluster-添加硬约束)
		- [6.3.1. 为什么要这个 trailing window 约束](#631-为什么要这个-trailing-window-约束)
		- [6.3.2. 添加约束后顶点数 \> 256 的额外处理](#632-添加约束后顶点数--256-的额外处理)
	- [6.4. 对每个 MaterialRange, 把它内部的三角形 Mesh 再划分成多个 Batch](#64-对每个-materialrange-把它内部的三角形-mesh-再划分成多个-batch)
	- [6.5. 顶点位置量化: 决定每个 Cluster 的 Position 怎么变成整数并写入 bitstream](#65-顶点位置量化-决定每个-cluster-的-position-怎么变成整数并写入-bitstream)
		- [6.5.1. 选择合适量化精度](#651-选择合适量化精度)
		- [6.5.2. float -\> int 量化](#652-float---int-量化)
		- [6.5.3. 转换为 Cluster 局部坐标](#653-转换为-cluster-局部坐标)
		- [6.5.4. 为 X/Y/Z 每个轴计算最小 bit 数](#654-为-xyz-每个轴计算最小-bit-数)
	- [6.6. 计算每个 Cluster 的编码信息](#66-计算每个-cluster-的编码信息)
		- [6.6.1. UV 压缩](#661-uv-压缩)
	- [6.7. 把 Cluster 按 Group 顺序打包进 GPU Page, 同时生成 GroupPart 和实例数据](#67-把-cluster-按-group-顺序打包进-gpu-page-同时生成-grouppart-和实例数据)
	- [6.8. 构建层次结构 BVH](#68-构建层次结构-bvh)
	- [6.9. 写入 Page 数据](#69-写入-page-数据)

## 1. 找对向共享边

### 1.1. 对向共享边

对于每一个三角形 Mesh，对向共享边指的是两条边满足：**顶点相同、方向相反**。例如：三角形 A 的边 v0->v1，三角形 B 的边 v1->v0，这两条边顶点相同、方向相反，也就是说它们互相是对方的对向共享边。

### 1.2. 构建 EdgeHash

核心逻辑：

```cpp
FEdgeHash EdgeHash( Indexes.Num() );

auto GetPosition = [ &Verts, &Indexes ]( uint32 EdgeIndex )
{
	return Verts.Position[ Indexes[ EdgeIndex ] ];
};

// 遍历所有的有向边，根据有向边两个端点的 Position, 计算出这个有向边的哈希, 最后存入 EdgeHash 中
ParallelFor( TEXT("Nanite.ClusterTriangles.PF"), Indexes.Num(), 4096,
	[&]( int32 EdgeIndex )
	{
		EdgeHash.Add_Concurrent( EdgeIndex, GetPosition );
	} );
```

通过 Mesh 数据中的 IndexBuffer 构建 EdgeHash（边查找表），其中：

- IndexBuffer 中每 3 个 Index 表示一个三角形 Mesh
- EdgeIndex 与 Index 一一对应，每一个 EdgeIndex 都被当作一个**有向边的起点**

举个例子：假设有 Mesh 数据 Verts = [v0, v1, v2, v3, v4, v5, ...]，Indexes = [i0, i1, i2, ...]，那么对于三角形 Mesh (i0, i1, i2)：

- 有向边 EdgeIndex 0 表示的是边 i0->i1
- 有向边 EdgeIndex 1 表示的是边 i1->i2
- 有向边 EdgeIndex 2 表示的是边 i2->i0

通过构建好的 EdgeHash，可以快速查找每条三角形 Mesh 边的对向共享边，从而构建三角形 Mesh 之间的连通关系。

下面是构建 EdgeHash 时的**一些核心的类和函数**：

#### 1.2.1. Cycle3

```cpp
FORCEINLINE uint32 Cycle3( uint32 Value )
{
	uint32 ValueMod3 = Value % 3;
	uint32 Value1Mod3 = ( 1 << ValueMod3 ) & 3;
	return Value - ValueMod3 + Value1Mod3;
}
```

`Cycle3` 函数在三角形 3 个顶点内循环+1，对于任意传入的 EdgeIndex（有向边 EdgeIndex 的起点），此函数返回下一个 EdgeIndex（有向边 EdgeIndex 的终点）

#### 1.2.2. FEdgeHash::Add_Concurrent

```cpp
template< typename FGetPosition >
void Add_Concurrent( int32 EdgeIndex, FGetPosition&& GetPosition )
{
	// 有向边 EdgeIndex 的起点 Position
	const FVector3f& Position0 = GetPosition( EdgeIndex );
	// 有向边 EdgeIndex 的终点 Position
	const FVector3f& Position1 = GetPosition( Cycle3( EdgeIndex ) );
	
	// 根据有向边 EdgeIndex 的两个端点的 Position, 计算这条有向边的哈希
	uint32 Hash0 = HashPosition( Position0 );
	uint32 Hash1 = HashPosition( Position1 );
	uint32 Hash = Murmur32( { Hash0, Hash1 } );

	// 添加到 HashTable 中, 其中 key 是有向边 EdgeIndex 的哈希, value 是 EdgeIndex
	HashTable.Add_Concurrent( Hash, EdgeIndex );
}
```

**注意**：因为 `Murmur32( { Hash0, Hash1 } )` 和 `Murmur32( { Hash1, Hash0 } )` 计算出来的哈希是不同的，也就是说如果有两条边，他们公用相同的顶点，但是方向却不同（一条是 v0->v1，另一条是 v1->v0），那么这两条边代表了**不同的两边有向边**！

#### 1.2.3. FHashTable

再来看看类 `FHashTable` 的数据结构，它的核心是两个数组：桶数组 `Hash` 和链表数组 `NextIndex`

```cpp
uint32 HashSize;	// = 1 << FMath::FloorLog2(Indexes.Num()) 桶数量（必须是 2 的幂）
uint32 HashMask;	// = HashSize - 1
uint32 IndexSize;   // 可存储的索引容量

uint32* Hash;	   // 桶数组
uint32* NextIndex;  // 链表数组
```

##### 1.2.3.1 桶数组 `Hash`

桶数组 `Hash` 是一个长度为 `HashSize` 的数组，每个元素表示一个**哈希桶**，`Hash[Bucket]` 存储的是该桶中的**链表的头节点 Index**，若 `Hash[Bucket] = ～0u (0xffffffff)` ，则表示该桶为空。**实际冲突解决采用单向链表结构**，链表通过 `NextIndex[Index]` 进行串联。

桶的计算方式： `Bucket = Key & HashMask` ，其中：

- `Key` 是有向边的哈希
- `HashSize` 必须是 2 的幂
- `HashMask = HashSize - 1`

由于 HashSize 为 2 的幂，因此 `Key & HashMask` 等价于 `Key % HashSize` ，保证 `Bucket` 不会越界，并且使用位运算可以避免除法开销，从而提升性能。

##### 1.2.3.2. 链表数组 `NextIndex`

链表数组 `NextIndex` 是一个长度为 `IndexSize` 的数组，`NextIndex[i]` 表示在同一桶内，索引 `i` 的下一个节点的 `Index` ，若 `NextIndex[i] = ~0u (0xffffffff)` ，则表示 `i` 为该桶链表的尾节点。

`NextIndex` 与 `Hash` 共同构成单向链表结构：

```css
Hash[Bucket] -> Index -> NextIndex[Index] -> ...
```

### 1.3. 根据 EdgeHash 统计对向共享边

```cpp
template< typename FGetPosition, typename FuncType >
void ForAllMatching( int32 EdgeIndex, bool bAdd, FGetPosition&& GetPosition, FuncType&& Function )
{
	const FVector3f& Position0 = GetPosition( EdgeIndex );
	const FVector3f& Position1 = GetPosition( Cycle3( EdgeIndex ) );

	uint32 Hash0 = HashPosition( Position0 );
	uint32 Hash1 = HashPosition( Position1 );

	// 注意: 这里是用反方向计算哈希
	// 在 FEdgeHash::Add_Concurrent 方法中是 uint32 Hash = Murmur32( { Hash0, Hash1 } );
	// 而这里查找对向共享边时, 是 uint32 Hash = Murmur32( { Hash1, Hash0 } );
	uint32 Hash = Murmur32( { Hash1, Hash0 } );

	for( uint32 OtherEdgeIndex = HashTable.First( Hash ); HashTable.IsValid( OtherEdgeIndex ); OtherEdgeIndex = HashTable.Next( OtherEdgeIndex ) )
	{
		if( Position0 == GetPosition( Cycle3( OtherEdgeIndex ) ) &&
			Position1 == GetPosition( OtherEdgeIndex ) )
		{
			// Found matching edge.
			Function( EdgeIndex, OtherEdgeIndex );
		}
	}

	if( bAdd )
		HashTable.Add( Murmur32( { Hash0, Hash1 } ), EdgeIndex );
}

FAdjacency Adjacency( Indexes.Num() );

// 遍历所有的有向边
ParallelFor( TEXT("Nanite.ClusterTriangles.PF"), Indexes.Num(), 1024,
	[&]( int32 EdgeIndex )
	{
		int32 AdjIndex = -1;
		int32 AdjCount = 0;
		// 找每条有向边的对向共享边
		EdgeHash.ForAllMatching( EdgeIndex, false, GetPosition,
			[&]( int32 EdgeIndex, int32 OtherEdgeIndex )
			{
				AdjIndex = OtherEdgeIndex;
				AdjCount++;
			} );

		if( AdjCount > 1 )  // 对向共享边数量超过 1 个, 直接赋值为 -2
			AdjIndex = -2;

		Adjacency.Direct[ EdgeIndex ] = AdjIndex;
	} );
```

通过 `Indexes` 遍历所有的有向边，在 EdgeHash 中查找每条有向边的对向共享边，在 `FAdjacency::Direct` 数组中统计每条有向边的对向共享边数量，最终记录 **3 种情况**：

- `FAdjacency::Direct[EdgeIndex] = -1` ：表示有向边 EdgeIndex **没有邻接**的对向共享边（边界边）
- `FAdjacency::Direct[EdgeIndex] = 1` ：表示有向边 EdgeIndex **有唯一**的对向共享边（大部分情况，正常流形）
- `FAdjacency::Direct[EdgeIndex] = -2` ：表示有向边 EdgeIndex **有超过 1 个**的对向共享边（特殊情况，非流形）

对于有超过 1 个对向共享边的特殊有向边：

- 先找到其所有的对向共享边
- 因为 EdgeHash 是并行构建的，因此还要对找到的对向共享边按照 EdgeIndex （也就是顶点索引 Index）进行排序**保证 Link 结果的确定性：同样的 Mesh 输入，永远得到同样的 Link 顺序**
- 将一对 N 的结果存入到 `FAdjacency::Extended` 中

```cpp
// 额外处理有超过 1 个对向共享边的特殊有向边
if( Adjacency.Direct[ EdgeIndex ] == -2 )
{
	// EdgeHash is built in parallel, so we need to sort before use to ensure determinism.
	// This path is only executed in the rare event that an edge is shared by more than two triangles,
	// so performance impact should be negligible in practice.
	TArray< TPair< int32, int32 >, TInlineAllocator< 16 > > Edges;
	EdgeHash.ForAllMatching( EdgeIndex, false, GetPosition,
		[&]( int32 EdgeIndex0, int32 EdgeIndex1 )
		{
			Edges.Emplace( EdgeIndex0, EdgeIndex1 );
		} );
	Edges.Sort();   // 按照 EdgeIndex 排序, 保证确定性
	
	for( const TPair< int32, int32 >& Edge : Edges )
	{
		Adjacency.Link( Edge.Key, Edge.Value );
	}
}
```

## 2. 根据对向共享边信息构建三角形 Mesh 的连通关系

核心逻辑：

```cpp
FDisjointSet DisjointSet( NumTriangles );

Adjacency.ForAll( EdgeIndex,
	[&]( int32 EdgeIndex0, int32 EdgeIndex1 )
	{
		if( EdgeIndex0 > EdgeIndex1 )
			DisjointSet.UnionSequential( EdgeIndex0 / 3, EdgeIndex1 / 3 );
	} );
```

类 `FDisjointSet` 是一个并查集（Union-Find），用于把三角形 Mesh 按照拓扑连通性分组成若干**连通块（Island）**，**每个连通块代表一个拓扑连通的三角形 Mesh 的集合**，在 Nanite 中如果两个三角形 Mesh 通过任一条边（对向共享边）连通，那么它们就属于同一个连通块。

下面是类 `FDisjointSet` 的一些核心函数：

### 2.1. `FDisjointSet::Init`

```cpp
FORCEINLINE void FDisjointSet::Init( uint32 Size )
{
	Parents.SetNumUninitialized( Size, EAllowShrinking::No);
	for( uint32 i = 0; i < Size; i++ )
	{
		Parents[i] = i;
	}
}
```

初始化 `Parents` 数组，表示 `Size` 个元素每个自成一个连通块（也就是说，每个元素的父节点就是它自己）， `Parents[x] = x` 表示 `x` 是它自己的父节点，即它是根节点（root）

### 2.2. `FDisjointSet::Union`

```cpp
// Union with splicing
inline void FDisjointSet::Union( uint32 x, uint32 y )
{
	// 获取 x 和 y 的父节点 px 和 py
	uint32 px = Parents[x];
	uint32 py = Parents[y];

	while( px != py )		// 如果 px 和 py 不相等, 说明它们不是同一个连通块, 继续合并
	{
		// 合并规则:
		// 如果 px < py, 说明 x 所在的树较小, 将 x 的父节点设置为 py, 也就是把 x 合并到 y 所在的树上
		// 如果 px >= py, 则反之, 将 y 合并到 x 所在的树上
		if( px < py )
		{
			Parents[x] = py;	// 把 x 的父节点直接设置为 py
			if( x == px )		// x 本身就是 root, 说明现在 union 完成
			{
				return;
			}
			
			x = px;				// 将 x 设置为旧的 x 的父节点, 继续往上
			px = Parents[x];	// 更新 x 的父节点
		}
		else
		{
			Parents[y] = px;	// 把 y 的父节点直接设置为 px
			if( y == py )		// y 本身就是 root, 说明现在 union 完成
			{
				return;
			}
			y = py;				// 将 y 设置为旧的 y 的父节点, 继续往上
			py = Parents[y];	// 更新 y 的父节点
		}
	}
}
```

**Union** 操作的目标是将两个元素 x 和 y 所在的连通块合并成一个连通块，并且使用按大小合并的策略，始终将较小的树合并到较大的树上。下面是一个 Union 的例子：

```css
假设:
x -> a -> b -> c
y -> d
x < y

Union后:
x -> d
a -> d
b -> d
c -> d
y -> d
```

### 2.3. `FDisjointSet::UnionSequential`

**特化版本的** `FDisjointSet::Union` ，使用前提：

- `x` 一定是根节点（root）
- `y <= x`

```cpp
// Optimized version of Union when iterating for( x : 0 to N ) unioning x with lower indexes.
// Neither x nor y can have already been unioned with an index > x.
inline void FDisjointSet::UnionSequential( uint32 x, uint32 y )
{
	// 两个必备的前提!!!
	checkSlow( x >= y );
	checkSlow( x == Parents[x] );

	uint32 px = x;			// x 的父节点, 也就是 x 自己
	uint32 py = Parents[y];	// y 的父节点
	while( px != py )
	{
		Parents[y] = px;
		if( y == py )
		{
			return;
		}
		y = py;
		py = Parents[y];
	}
}
```

### 2.4. `FDisjointSet::Find`

```cpp
// Find with path compression
inline uint32 FDisjointSet::Find( uint32 i )
{
	// 找到 i 的根节点
	uint32 Start = i;
	uint32 Root = Parents[i];
	while( Root != i )
	{
		i = Root;
		Root = Parents[i];
	}

	// 路径压缩:
	// 把从 i 到 i 的根节点之间的所有节点的父节点都设置为 i 的根节点
	// 这样后续再 Find 此路径上的任意节点, 都只需要一次内存访问即可
	i = Start;
	uint32 Parent = Parents[i];
	while( Parent != Root )
	{
		Parents[i] = Root;
		i = Parent;
		Parent = Parents[i];
	}

	return Root;
}
```

查找节点 i 的根节点（也就是查找 i 属于哪个连通块），并且**压缩路径：把从 i 到 i 的根节点之间的所有节点的父节点都设置为 i 的根节点**，这样后续再 Find 此路径上的任意节点，都只需要一次内存访问即可。

## 3. 划分 Clusters

把整个 Mesh 的 NumTriangles 个三角形 Mesh 划分成若干个 Clusters ， Cluster 需要满足：

- 大小接近，每个 Cluster 大约包含 128 个三角形 Mesh
- 优先保证拓扑连续（连通的三角形 Mesh 在同一个 Cluster），其次保证空间局部性（空间上接近的三角形 Mesh 在同一个 Cluster）

### 3.1. 创建图划分器

```cpp
FGraphPartitioner::FGraphPartitioner( uint32 InNumElements, int32 InMinPartitionSize, int32 InMaxPartitionSize )
	: NumElements( InNumElements )
	, MinPartitionSize( InMinPartitionSize )
	, MaxPartitionSize( InMaxPartitionSize )
{
	Indexes.AddUninitialized( NumElements );
	for( uint32 i = 0; i < NumElements; i++ )
	{
		Indexes[i] = i;
	}
}

FGraphPartitioner Partitioner( NumTriangles, FCluster::ClusterSize - 4, FCluster::ClusterSize );
```

`FGraphPartitioner::NumElements` 是节点数量，每个三角形 Mesh 是一个节点

### 3.2. 构建空间局部邻接信息 `FGraphPartitioner::BuildLocalityLinks`

#### 3.2.1. 计算 Morton 编码

```cpp
ParallelFor( TEXT("BuildLocalityLinks.PF"), NumElements, 4096,
	[&]( uint32 Index )
	{
		// 获取三角形 Mesh 的中心点坐标
		FVector3f Center = GetCenter( Index );
		// 归一化
		FVector3f CenterLocal = ( Center - Bounds.Min ) / FVector3f( Bounds.Max - Bounds.Min ).GetMax();

		// 计算 Morton 编码
		uint32 Morton;
		Morton  = FMath::MortonCode3( uint32( CenterLocal.X * 1023 ) );
		Morton |= FMath::MortonCode3( uint32( CenterLocal.Y * 1023 ) ) << 1;
		Morton |= FMath::MortonCode3( uint32( CenterLocal.Z * 1023 ) ) << 2;
		SortKeys[ Index ] = Morton;
	} );
```

根据三角形 Mesh 中心点的 3D 坐标，计算其对应的 Morton 编码，将 3D 坐标转换为一维可排序整数。 **Morton 编码**，又叫做 **Z-order Curve** 或 **Z 值编码**，它具有一个重要的性质， **3D 空间上接近的点，其 Morton 编码也接近**，也就是说：**空间上的邻居 $\approx$ 排序后数组中的邻居**。

#### 3.2.2. 通过 Morton 编码排序所有的三角形 Mesh

```cpp
RadixSort32( SortedTo.GetData(), Indexes.GetData(), NumElements,
	[&]( uint32 Index )
	{
		return SortKeys[ Index ];
	} );

SortKeys.Empty();

Swap( Indexes, SortedTo );  // 这里 Swap 之后, Indexes 是排序后的数组
for( uint32 i = 0; i < NumElements; i++ )
{
	SortedTo[ Indexes[i] ] = i;
}
```

`SortedTo` 是排序后的数组， `Indexes` 是原始未排序的数组，将它们 `Swap` 之后再重排索引，最终建立 `SortedTo[排序后的三角形 Mesh Index] = 原始的三角形 Mesh Index` 的映射数组

#### 3.2.3. 构建 IslandRuns

```cpp
TArray< FRange > IslandRuns;
IslandRuns.AddUninitialized( NumElements );

// Run length acceleration
// Range of identical IslandID denoting that elements are connected.
// Used for jumping past connected elements to the next nearby disjoint element.
{
	uint32 RunIslandID = 0;
	uint32 RunFirstElement = 0;

	// 遍历所有排好序的三角形 Mesh
	for( uint32 i = 0; i < NumElements; i++ )
	{
		// 此三角形 Mesh 属于哪一个连通块
		uint32 IslandID = DisjointSet.Find( Indexes[i] );

		// 当前三角形 Mesh 属于一个新的连通块
		if( RunIslandID != IslandID )
		{
			// 上一个连通块区间的起点 RunFirstElement, 终点 i - 1
			// 为上一个连通块区间中的每个元素设置终点 End = i - 1
			for( uint32 j = RunFirstElement; j < i; j++ )
			{
				IslandRuns[j].End = i - 1;
			}
		
			// 现在新的连通块区间从 i 开始
			RunIslandID = IslandID;
			RunFirstElement = i;
		}

		// 更新当前元素的连通块区间起点 Begin
		IslandRuns[i].Begin = RunFirstElement;
	}
	// 单独处理最后一个连通块区间的终点 End
	for( uint32 j = RunFirstElement; j < NumElements; j++ )
	{
		IslandRuns[j].End = NumElements - 1;
	}
}
```

在一个**已经按照 Morton 编码排好序**的三角形 Mesh 数组中, 通过遍历每个元素并使用并查集（ `DisjointSet` ）查找其所属的连通块（ `IslandID` ），识别并标记出每一段连续属于同一个连通块的区间。每个连通块区间被记录为一个 `FRange` 对象，包含该区间的**三角形网格数组中的索引起始位置（Begin）和结束位置（End）**

#### 3.2.4. 对于所属连通块区间元素数量小于 128 的三角形 Mesh ，构建它们的空间局部邻接信息

```cpp
for( uint32 i = 0; i < NumElements; i++ )
{
	uint32 Index = Indexes[i];

	uint32 RunLength = IslandRuns[i].End - IslandRuns[i].Begin + 1;
	// 对于所在连通块区间元素数量小于 128 的三角形 Mesh, 构建它们的空间局部邻接信息
	if( RunLength < 128 )
	{
		uint32 IslandID = DisjointSet[ Index ];
		int32 GroupID = bElementGroups ? GroupIndexes[ Index ] : 0;

		FVector3f Center = GetCenter( Index );

		// 保留空间上距离最近的 5 个三角形 Mesh 邻居
		const uint32 MaxLinksPerElement = 5;

		uint32 ClosestIndex[MaxLinksPerElement];
		float  ClosestDist2[MaxLinksPerElement];
		for (int32 k = 0; k < MaxLinksPerElement; k++)
		{
			ClosestIndex[k] = ~0u;
			ClosestDist2[k] = MAX_flt;
		}

		// 在排好序的数组前后 2 个方向查找
		for( int Direction = 0; Direction < 2; Direction++ )
		{
			uint32 Limit = Direction ? NumElements - 1 : 0;
			uint32 Step  = Direction ? 1 : -1;

			uint32 Adj = i;
			// 每个方向最多找 16 次, 一共最多找 32 个三角形 Mesh 邻居
			for( int32 Iterations = 0; Iterations < 16; Iterations++ )
			{
				if( Adj == Limit )
					break;
				Adj += Step;

				uint32 AdjIndex = Indexes[ Adj ];
				uint32 AdjIslandID = DisjointSet[ AdjIndex ];
				int32 AdjGroupID = bElementGroups ? GroupIndexes[AdjIndex] : 0;
				
				// 跳过同一个连通块或不同 Group 的邻居
				// 同一个连通块 -> 不需要连
				// 不同 Group -> 禁止连
				if( IslandID == AdjIslandID || ( GroupID != AdjGroupID ) )
				{
					// Skip past this run
					if( Direction )
						Adj = IslandRuns[ Adj ].End;
					else
						Adj = IslandRuns[ Adj ].Begin;
				}
				else
				{
					// Add to sorted list
					// 计算空间上的距离并使用插入排序更新
					float AdjDist2 = ( Center - GetCenter( AdjIndex ) ).SizeSquared();
					for( int k = 0; k < MaxLinksPerElement; k++ )
					{
						if( AdjDist2 < ClosestDist2[k] )
						{
							Swap( AdjIndex, ClosestIndex[k] );
							Swap( AdjDist2, ClosestDist2[k] );
						}
					}
				}
			}
		}

		// 建立双向 LocalityLinks
		for( int k = 0; k < MaxLinksPerElement; k++ )
		{
			if( ClosestIndex[k] != ~0u )
			{
				// Add both directions
				LocalityLinks.AddUnique( Index, ClosestIndex[k] );
				LocalityLinks.AddUnique( ClosestIndex[k], Index );
			}
		}
	}
}
```

### 3.3. 创建图对象（FGraphData）

```cpp
// 初始化图对象, 每个三角形 Mesh 是一个节点
auto* RESTRICT Graph = Partitioner.NewGraph( NumTriangles * 3 );

for( uint32 i = 0; i < NumTriangles; i++ )
{
	// 第 i 个节点的邻接节点列表, 从当前 Graph->Adjacency.Num() 开始
	Graph->AdjacencyOffset[i] = Graph->Adjacency.Num();

	// 按 Morton 编码排序后的三角形 Mesh 数组顺序构建
	uint32 TriIndex = Partitioner.Indexes[i];

	// 遍历三角形 Mesh 的三条有向边, 找每条有向边的对向共享边
	for( int k = 0; k < 3; k++ )
	{
		Adjacency.ForAll( 3 * TriIndex + k,
			[ &Partitioner, Graph ]( int32 EdgeIndex, int32 AdjIndex )
			{
				// 如果能找到对向共享边, 说明有其它的三角形 Mesh 与当前三角形 Mesh 共享同一条边, 对这两个有共享边的节点设置一个大权重
				Partitioner.AddAdjacency( Graph, AdjIndex / 3, 4 * 65 );
			} );
	}

	// 根据三角形 Mesh 的空间局部邻接信息设置节点之间的极小权重
	Partitioner.AddLocalityLinks( Graph, TriIndex, 1 );
}

// 结束时补最后一个 Offset 作为数组终点
Graph->AdjacencyOffset[ NumTriangles ] = Graph->Adjacency.Num();
```

`FGraphData` 对象包含：

- `Num`：节点数量，每个三角形 Mesh 是一个节点
- `Adjacency`： `Num` 个节点的**所有**邻接节点的索引
- `AdjacencyCost`： `Num` 个节点与邻接节点的邻接权重，数组中的每个元素与 `Adjacency` 数组中的每个元素一一对应

在设置节点之间邻接权重时：

- 对于有共享边的节点，设置一个大的邻接权重（260），大权重意味着：**如果两个三角形 Mesh 有共享边，则非常强烈的希望它们被分到同一个 Cluster**
- 而对于没有共享边的节点，则根据空间局部邻接信息，设置空间上比较接近的节点一个极小的邻接权重（1），也就是说：**鼓励空间上邻近的三角形 Mesh 被分到同一个 Cluster ，但它的优先级远远低于真正的拓扑邻接（互相之间有对向共享边，大权重260）**

#### 3.3.1. 一个例子

```css
T0 ---1--- T2
|		 / |
|	   /   |
260  260   260
|   /	   |
| /		 |
T1 ---1--- T3
```

如上图所示，一个 Mesh 有 4 个三角形 Mesh： T0 、 T1 、 T2 和 T3 ，它们之间关系如下：

- 拓扑邻接： T0 -- T1 、 T1 -- T2 、 T2 -- T3
- 空间上邻近： T0 邻近 T2 、 T1 邻近 T3
- 4 个三角形 Mesh 的中心点坐标按照 Morton 编码排序后的顺序是： T2 -> T0 -> T1 -> T3

那么：

- `FGraphPartitioner::BuildLocalityLinks` 函数中有 `Swap( Indexes, SortedTo )` 操作，所以此时 `Partitioner.Indexes` 是排好序后的三角形 Mesh 数组，也就是 `Partitioner.Indexes = [2, 0, 1, 3]`
- 节点构建顺序 TriIndex ： T2 -> T0 -> T1 -> T3
- 重映射后的 `Partitioner.SortedTo` 数组：
  - `Partitioner.SortedTo[2] = 0`
  - `Partitioner.SortedTo[0] = 1`
  - `Partitioner.SortedTo[1] = 2`
  - `Partitioner.SortedTo[3] = 3`
- 优先根据拓扑邻接（对向共享边）设置大权重，再根据空间局部邻接信息设置极小权重

最终构建的图对象如下：

```css
Graph->Num = 4; // 4 个节点
Graph->Adjacency =
[
	2, 3, 1,	// 节点 T2: 与 T1 和 T3 是拓扑邻接, 与 T0 是空间局部邻接
	2, 0,	   // 节点 T0: 与 T1 是拓扑邻接, 与 T2 是空间局部邻接
	1, 0, 3,	// 节点 T1: 与 T0 和 T2 是拓扑邻接, 与 T3 是空间局部邻接
	0, 2		// 节点 T3: 与 T2 是拓扑邻接, 与 T1 是空间局部邻接
];
Graph->AdjacencyCost =
[
	260, 260, 1,
	260, 1,
	260, 260, 1,
	260, 1
];
Graph->AdjacencyOffset =
[
	0,	  // 节点 T2 的邻接节点数据在 Graph->Adjacency 数组中的起始 Offset
	3,	  // 节点 T0 的邻接节点数据在 Graph->Adjacency 数组中的起始 Offset
	5,	  // 节点 T1 的邻接节点数据在 Graph->Adjacency 数组中的起始 Offset
	8,	  // 节点 T3 的邻接节点数据在 Graph->Adjacency 数组中的起始 Offset
	10
];
```

### 3.4. 使用 METIS 对构建好的图进行递归二分划分子 Partitions

#### 3.4.1. METIS

下面有一个使用 METIS 库的简单使用说明：

[使用 METIS 软件包进行图划分](https://www.jianshu.com/p/55e6b6897057)

关于 METIS 的控制参数 `Options[ METIS_OPTION_UFACTOR ]` ：用于控制划分后子 Partition 之间允许的不平衡程度（以千分之一为单位），简单来说：就是划分后每个子 Partition 最多可以比“理想大小”大多少。假设一个图一共有 N 个节点，要将其划分为 `k` 个子 Partition，那么理想情况下，每个子 Partition 应该包含 `N/k` 个节点，如果 `Options[ METIS_OPTION_UFACTOR ] = 20`，那么每个子 Partition 最多比理想大小多 2%（20/1000=0.02）的节点数。

#### 3.4.2. 核心函数 `FGraphPartitioner::BisectGraph`

```cpp
void FGraphPartitioner::BisectGraph( FGraphData* Graph, FGraphData* ChildGraphs[2] )
{
	ChildGraphs[0] = nullptr;
	ChildGraphs[1] = nullptr;

	auto AddPartition = [ this ]( int32 Offset, int32 Num )
	{
		FRange& Range = Ranges[ NumPartitions++ ];
		Range.Begin	= Offset;
		Range.End	= Offset + Num;
	};

	// 快速终止条件: 只要当前节点数量够小, 不再继续划分, 当前图直接成为最终的子 Partition
	if( Graph->Num <= MaxPartitionSize )
	{
		AddPartition( Graph->Offset, Graph->Num );
		return;
	}

	const int32 TargetPartitionSize = ( MinPartitionSize + MaxPartitionSize ) / 2;
	const int32 TargetNumPartitions = FMath::Max( 2, FMath::DivideAndRoundNearest( Graph->Num, TargetPartitionSize ) );

	check( Graph->AdjacencyOffset.Num() == Graph->Num + 1 );

	idx_t NumConstraints = 1;
	idx_t NumParts = 2;	 // 每次将图划分为 2 个子 Partition
	idx_t EdgesCut = 0;

	// 每个子 Partition 的权重, 使划分出的每个子 Partition 的 Cluster 数量尽量均衡
	real_t PartitionWeights[] = {
		float( TargetNumPartitions / 2 ) / TargetNumPartitions,
		1.0f - float( TargetNumPartitions / 2 ) / TargetNumPartitions
	};

	idx_t Options[ METIS_NOPTIONS ];
	METIS_SetDefaultOptions( Options );

	// Allow looser tolerance when at the higher levels. Strict balance isn't that important until it gets closer to partition sized.
	// 在较高层级时（大图，节点比较多）可以允许子 Partition 之间更不均衡
	bool bLoose = TargetNumPartitions >= 128 || MaxPartitionSize / MinPartitionSize > 1;
	bool bSlow = Graph->Num < 4096;
	
	Options[ METIS_OPTION_UFACTOR ] = bLoose ? 200 : 1;
	//Options[ METIS_OPTION_NCUTS ] = Graph->Num < 1024 ? 8 : ( Graph->Num < 4096 ? 4 : 1 );
	//Options[ METIS_OPTION_NCUTS ] = bSlow ? 4 : 1;
	//Options[ METIS_OPTION_NITER ] = bSlow ? 20 : 10;
	//Options[ METIS_OPTION_IPTYPE ] = METIS_IPTYPE_RANDOM;
	//Options[ METIS_OPTION_MINCONN ] = 1;

	// 使用 METIS 进行划分, 输出结构 PartitionIDs, 其中:
	// PartitionIDs[Graph->Offset + index] = 0 -> 划分到左子 Partition
	// PartitionIDs[Graph->Offset + index] = 1 -> 划分到右子 Partition
	int r = METIS_PartGraphRecursive(
		&Graph->Num,
		&NumConstraints,					// number of balancing constraints
		Graph->AdjacencyOffset.GetData(),
		Graph->Adjacency.GetData(),
		NULL,								// Vert weights
		NULL,								// Vert sizes for computing the total communication volume
		Graph->AdjacencyCost.GetData(),		// Edge weights
		&NumParts,
		PartitionWeights,					// Target partition weight
		NULL,								// Allowed load imbalance tolerance
		Options,
		&EdgesCut,
		PartitionIDs.GetData() + Graph->Offset
	);

	if (r == METIS_ERROR_MEMORY)
	{
		UE_LOG(LogStaticMesh, Error, TEXT("Call to METIS_PartGraphRecursive() failed - error code: %d, Graph->Num: %d"), r, Graph->Num);
		// We can't get the precise allocation size, but Metis logs an error that contains the actual error.
		FPlatformMemory::OnOutOfMemory(0, 0);
	}

	checkf(r == METIS_OK, TEXT("Call to METIS_PartGraphRecursive() failed - error code: %d, Graph->Num: %d"), r, Graph->Num);

	{
		// In place divide the array
		// Both sides remain sorted but back is reversed.
		// 按划分结果 PartitionIDs 对数组 Indexes 进行原地重排, 前半部分全是 PartitionID = 0, 后半部分全是 PartitionID = 1
		// 双指针法, Front 从前往后, Back 从后往前
		int32 Front = Graph->Offset;
		int32 Back =  Graph->Offset + Graph->Num - 1;
		while( Front <= Back )
		{
			// 跳过已经正确在前半部分的元素, 直到遇到一个应该在后半部分的元素
			while( Front <= Back && PartitionIDs[ Front ] == 0 )
			{
				SwappedWith[ Front ] = Front;
				Front++;						// 继续后移指针
			}
			// 跳过已经正确在右侧的元素, 直到遇到一个应该在左侧的元素
			while( Front <= Back && PartitionIDs[ Back ] == 1 )
			{
				SwappedWith[ Back ] = Back;
				Back--;						 // 继续前移指针
			}

			if( Front < Back )
			{
				Swap( Indexes[ Front ], Indexes[ Back ] );  // 交换这 2 个错位的元素

				SwappedWith[ Front ] = Back;
				SwappedWith[ Back ] = Front;
				Front++;
				Back--;
			}
		}

		// 计算 2 个子 Partition 的节点数量
		int32 Split = Front;

		int32 Num[2];
		Num[0] = Split - Graph->Offset;
		Num[1] = Graph->Offset + Graph->Num - Split;
				
		check( Num[0] > 0 );
		check( Num[1] > 0 );

		// 如果 2 个子 Partition 的节点数量都足够小, 在当前层停止继续二分划分, 分别记录最终的 2 个子 Partition
		if( Num[0] <= MaxPartitionSize &&
			Num[1] <= MaxPartitionSize )
		{
			AddPartition( Graph->Offset,	Num[0] );
			AddPartition( Split,			Num[1] );
		}
		else	// 否则进入递归二分划分准备阶段
		{
			// 为 2 个子 Partition 创建 FGraphData
			for( int32 i = 0; i < 2; i++ )
			{
				ChildGraphs[i] = new FGraphData;
				ChildGraphs[i]->Adjacency.Reserve( Graph->Adjacency.Num() >> 1 );
				ChildGraphs[i]->AdjacencyCost.Reserve( Graph->Adjacency.Num() >> 1 );
				ChildGraphs[i]->AdjacencyOffset.Reserve( Num[i] + 1 );
				ChildGraphs[i]->Num = Num[i];
			}

			ChildGraphs[0]->Offset = Graph->Offset;
			ChildGraphs[1]->Offset = Split;

			// 遍历所有的父图节点, 根据父图节点数据构建子 Partition 的 FGraphData 数据
			for( int32 i = 0; i < Graph->Num; i++ )
			{
				// 判断父图节点属于哪一个子 Partition, 并获取其 FGraphData 数据
				FGraphData* ChildGraph = ChildGraphs[ i >= ChildGraphs[0]->Num ];

				ChildGraph->AdjacencyOffset.Add( ChildGraph->Adjacency.Num() );
				
				// 父图节点中的索引 OrgIndex
				int32 OrgIndex = SwappedWith[ Graph->Offset + i ] - Graph->Offset;
				// 遍历父图中索引为 OrgIndex 的节点的所有邻接节点
				for( idx_t AdjIndex = Graph->AdjacencyOffset[ OrgIndex ]; AdjIndex < Graph->AdjacencyOffset[ OrgIndex + 1 ]; AdjIndex++ )
				{
					// 索引为 OrgIndex 的节点在父图中的邻接节点索引
					idx_t Adj	 = Graph->Adjacency[ AdjIndex ];
					// 索引为 OrgIndex 的节点与此邻接节点的权重
					idx_t AdjCost = Graph->AdjacencyCost[ AdjIndex ];

					// Remap to child
					// 将父图中的节点索引映射到子 Partition 中的新索引
					Adj = SwappedWith[ Graph->Offset + Adj ] - ChildGraph->Offset;

					// Edge connects to node in this graph
					// 保证在当前子 Partition 中
					if( 0 <= Adj && Adj < ChildGraph->Num )
					{
						// 添加子 Partition 的 FGraphData 数据
						ChildGraph->Adjacency.Add( Adj );
						ChildGraph->AdjacencyCost.Add( AdjCost );
					}
				}
			}
			ChildGraphs[0]->AdjacencyOffset.Add( ChildGraphs[0]->Adjacency.Num() );
			ChildGraphs[1]->AdjacencyOffset.Add( ChildGraphs[1]->Adjacency.Num() );
		}
	}
}
```

### 3.5. 根据划分的结果 `Partitioner.Ranges` 创建叶子 Clusters

```cpp
const uint32 BaseCluster = Clusters.Num();
Clusters.AddDefaulted( Partitioner.Ranges.Num() );

{
	TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::BuildClusters);
	ParallelFor( TEXT("Nanite.BuildClusters.PF"), Partitioner.Ranges.Num(), 1024,
		[&]( int32 Index )
		{
			auto& Range = Partitioner.Ranges[ Index ];

			Clusters[ BaseCluster + Index ] = FCluster(
				Verts,
				Indexes,
				MaterialIndexes,
				VertexFormat,
				Range.Begin, Range.End,
				Partitioner.Indexes, Partitioner.SortedTo, Adjacency );

			// Negative notes it's a leaf
			// 叶子 Clusters 的 EdgeLength 为负值
			Clusters[ BaseCluster + Index ].EdgeLength *= -1.0f;
		});
}
```

下面是 Cluster 构造函数中的一些核心逻辑的分析：

#### 3.5.1 收集 Cluster 的 ExternalEdges

```cpp
ExternalEdges.Reserve( 3 * NumTris );
NumExternalEdges = 0;

// 当前三角形 Mesh 的第 k 条有向边
int32 EdgeIndex = TriIndex * 3 + k;
int32 AdjCount = 0;

// 查找这条有向边的对向共享边
Adjacency.ForAll( EdgeIndex,
	[ &AdjCount, Begin, End, &SortedTo ]( int32 EdgeIndex, int32 AdjIndex )
	{
		// 当前三角形 Mesh 的拓扑邻接三角形 Mesh
		uint32 AdjTri = SortedTo[ AdjIndex / 3 ];

		// 如果邻接三角形 Mesh 不属于当前 Cluster, 则累计 AdjCount
		if( AdjTri < Begin || AdjTri >= End )
			AdjCount++;
	} );

// 为当前三角形 Mesh 的第 k 条有向边存储一个整数
ExternalEdges.Add( (int8)AdjCount );
// 统计当前 Clutser 有多少条边界边
NumExternalEdges += AdjCount != 0 ? 1 : 0;
```

`ExternalEdges` 中存储了 Cluster 中每个三角形 Mesh 的每条有向边的**对向共享边（属于其它 Cluster）**数量 `AdjCount` ：

- `AdjCount == 0` ：说明此条有向边完全在当前 Cluster 内部，没有与其它 Cluster 相邻
- `AdjCount > 0` ：说明此有向边是当前 Clutser 的边界边，它与其它 Cluster 相邻

#### 3.5.2. 计算 Cluster 的空间包围盒与几何尺度度量

```cpp
void FCluster::Bound()
{
	Bounds = FBounds3f();
	SurfaceArea = 0.0f;
	
	TArray< FVector3f, TInlineAllocator<128> > Positions;
	Positions.SetNum( NumVerts, EAllowShrinking::No );

	// 收集所有顶点 Position, 计算 Cluster 的 AABB
	for( uint32 i = 0; i < NumVerts; i++ )
	{
		Positions[i] = GetPosition(i);
		Bounds += Positions[i];
	}

	// 计算 Cluster 的包围球
	SphereBounds = FSphere3f( Positions.GetData(), Positions.Num() );
	// 用于 LOD 层级判断的包围球
	LODBounds = SphereBounds;
	
	float MaxEdgeLength2 = 0.0f;	// Cluster 内所有三角形 Mesh 中最长边的平方
	for( int i = 0; i < Indexes.Num(); i += 3 )
	{
		// 三角形 Mesh 的三个顶点
		FVector3f v[3];
		v[0] = GetPosition( Indexes[ i + 0 ] );
		v[1] = GetPosition( Indexes[ i + 1 ] );
		v[2] = GetPosition( Indexes[ i + 2 ] );

		// 三角形 Mesh 的三条有向边
		FVector3f Edge01 = v[1] - v[0];
		FVector3f Edge12 = v[2] - v[1];
		FVector3f Edge20 = v[0] - v[2];

		// 更新最长边的平方
		MaxEdgeLength2 = FMath::Max( MaxEdgeLength2, Edge01.SizeSquared() );
		MaxEdgeLength2 = FMath::Max( MaxEdgeLength2, Edge12.SizeSquared() );
		MaxEdgeLength2 = FMath::Max( MaxEdgeLength2, Edge20.SizeSquared() );

		// 计算三角形 Mesh 面积: ^ 是叉乘, |a x b| = 平行四边形面积, 所以三角形 Mesh 的面积 = 0.5 * 平行四边形面积
		float TriArea = 0.5f * ( Edge01 ^ Edge20 ).Size();
		SurfaceArea += TriArea;
	}
	EdgeLength = FMath::Sqrt( MaxEdgeLength2 );
}
```

##### 3.5.2.1. `Cluster::Bounds`

能完全包住 Cluster 的 AABB ，核心用途：

- **视锥裁剪（Frustum Culling）** ：只用做 6 个平面测试，速度极快
- **层级结构构建（Hierarchy Node Bounds）** ：父节点的 AABB = 所有子节点的 AABB 合并，如果父节点不可见，那么所有子节点都不用访问
- **遮挡剔除（Hi-Z Occlusion）** ：把 Cluster 的 AABB 投影成屏幕矩形，如果被深度图完全遮挡，则将其剔除
- **Streaming 范围判断** ：如果 Cluster 的 AABB 距离摄像机很远，那么此 Cluster 的数据低优先级加载或根本不加载

##### 3.5.2.2. `Cluster::SphereBounds`

通过 **Ritter 包围球算法（Ritter's Bounding Sphere Algorithm）** 计算 Cluster 的包围球，核心用途：

- **屏幕误差（Screen Error）** ：在 Nanite 中，屏幕误差的计算公式近似： `ScreenError` $\approx$ `(GeometricError / Distance) * ProjectionScale` ，这里的 `Distance` 是 `distance(CameraPos, Sphere.Center) - Sphere.Radius` 。当 Cluster 包围球在远处，此时屏幕误差小，可以使用更高 LOD （低细节）的 Cluster；当 Cluster 的包围球在近处，此时屏幕误差大，必须使用更低 LOD （高细节）的 Cluster

Ritter 包围球算法：

```cpp
// [ Ritter 1990, "An Efficient Bounding Sphere" ]
template<typename T>
static FORCEINLINE void ConstructFromPoints(TSphere<T>& ThisSphere, const TVector<T>* Points, int32 Count)
{
	check(Count > 0);

	// Min/max points of AABB
	// 1. 找 AABB 的三个轴的极点值
	int32 MinIndex[3] = { 0, 0, 0 };	// { X 轴最小, Y 轴最小, Z 轴最小 }
	int32 MaxIndex[3] = { 0, 0, 0 };	// { X 轴最大, Y 轴最大, Z 轴最大 }

	for (int i = 0; i < Count; i++)
	{
		for (int k = 0; k < 3; k++)
		{
			MinIndex[k] = Points[i][k] < Points[MinIndex[k]][k] ? i : MinIndex[k];
			MaxIndex[k] = Points[i][k] > Points[MaxIndex[k]][k] ? i : MaxIndex[k];
		}
	}

	// 2. 找跨度最大的轴
	T LargestDistSqr = 0.0f;
	int32 LargestAxis = 0;
	for (int k = 0; k < 3; k++)
	{
		TVector<T> PointMin = Points[MinIndex[k]];
		TVector<T> PointMax = Points[MaxIndex[k]];

		T DistSqr = (PointMax - PointMin).SizeSquared();
		if (DistSqr > LargestDistSqr)
		{
			LargestDistSqr = DistSqr;
			LargestAxis = k;
		}
	}

	// 3. 用最远的两点初始化球, 此时球正好包住这两个极端点, 但可能还包不住其它点
	TVector<T> PointMin = Points[MinIndex[LargestAxis]];
	TVector<T> PointMax = Points[MaxIndex[LargestAxis]];

	ThisSphere.Center = 0.5f * (PointMin + PointMax);
	ThisSphere.W = 0.5f * FMath::Sqrt(LargestDistSqr);
	T WSqr = ThisSphere.W * ThisSphere.W;

	// Adjust to fit all points
	// 扩张球以包住所有点
	for (int i = 0; i < Count; i++)
	{
		T DistSqr = (Points[i] - ThisSphere.Center).SizeSquared();

		if (DistSqr > WSqr)
		{
			// Ritter 算法核心: 扩大并移动球心以包住该点
			T Dist = FMath::Sqrt(DistSqr);
			T t = 0.5f + 0.5f * (ThisSphere.W / Dist);

			ThisSphere.Center = FMath::LerpStable(Points[i], ThisSphere.Center, t);
			ThisSphere.W = 0.5f * (ThisSphere.W + Dist);
		}
	}
}
```

##### 3.5.2.3. `Cluster::SurfaceArea`

Cluster 总表面积，核心用途：

- **误差归一化** ： Nanite 在构建 Hierarchy 时会计算 `RelativeError = Error / SurfaceArea` ，这意味着：表面积大的 Cluster ，允许更大绝对误差；而表面积小的 Cluster ，要求更小的绝对误差，这样大块 Mesh 不会因为体积过大而被过度细分

##### 3.5.2.4. `Cluster::EdgeLength`

Cluster 中最长边的长度，核心用途：

- **几何误差上限** ：如果一个 Cluster 有很长的边，表示此 Cluster 存在大尺度结构，不能过度简化

## 4. 构建 Clusters LOD 层级结构

```cpp
void FClusterDAG::ReduceMesh( uint32 ClusterRangeStart, uint32 ClusterRangeNum, uint32 MeshIndex )
{
	TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::DAG.ReduceMesh);

	if( ClusterRangeNum == 0 )
	{
		return;
	}

	TUniquePtr<FRayTracingScene> RayTracingScene;

#if NANITE_VOXEL_DATA
	if( Settings.bPreserveArea )
	{
		RayTracingScene = MakeUnique< FRayTracingScene >( Clusters, ClusterRangeStart, ClusterRangeNum );
	}
#endif
	
	// 当前层的 Clusters 在 FClusterDAG::Clusters 数组中的起始位置, 初始指向叶子 Clusters 的起始位置(所有的叶子 Clusters 首先被添加进 FClusterDAG::Clusters 数组，所以也就是 0)，之后每次生成父 Clusters 后都会把 LevelOffset 设置到父 Clusters 起始处
	uint32 LevelOffset	= ClusterRangeStart;
	
	// 最终有效 Clusters 数量, 因为是并行构建, 所以这里用原子操作
	TAtomic< uint32 > NumClusters( Clusters.Num() );

	bool bFirstLevel = true;

	UE::Tasks::FCancellationToken* CancellationToken = UE::Tasks::FCancellationTokenScope::GetCurrentCancellationToken();
	while( true )
	{
		if (CancellationToken && CancellationToken->IsCanceled())
		{
			return;
		}

		// 当前层的 Clusters
		TArrayView< FCluster > LevelClusters( &Clusters[ LevelOffset ], bFirstLevel ? ClusterRangeNum : (Clusters.Num() - LevelOffset) );
		bFirstLevel = false;

		// 累加当前层所有的 Clusters 的边界边数量 NumExternalEdges , Bounds 和 LOD 误差
		uint32 NumExternalEdges = 0;

		float MinError = +MAX_flt;
		float MaxError = -MAX_flt;
		float AvgError = 0.0f;

		for( FCluster& Cluster : LevelClusters )
		{
			NumExternalEdges	+= Cluster.NumExternalEdges;
			TotalBounds			+= Cluster.Bounds;

			MinError = FMath::Min( MinError, Cluster.LODError );
			MaxError = FMath::Max( MaxError, Cluster.LODError );
			AvgError += Cluster.LODError;
		}
		AvgError /= (float)LevelClusters.Num();

		UE_LOG( LogStaticMesh, Verbose, TEXT("Num clusters %i. Error %.4f, %.4f, %.4f"), LevelClusters.Num(), MinError, AvgError, MaxError );

		// 默认每个 Group 的最大 Cluster 数量 MaxGroupSize = 128
		uint32 MaxClusterSize = FCluster::ClusterSize;

		// 1. 当前层只有 1 个 Cluster, 如果此 Cluster 的三角形 Mesh 数量大于 0, 结束当前层的构建, 否则根据 Cluster 中三角形 Mesh 的材质数量决定是更新每个 Group 的最大 Cluster 数量 MaxGroupSize 还是结束当前层的构建
		if( LevelClusters.Num() < 2 )
		{
			if( LevelClusters[0].NumTris )
			{
				break;
			}
			else if( LevelClusters[0].MaterialIndexes.Num() > 64 )
			{
				MaxClusterSize = 64;
			}
			else if( LevelClusters[0].MaterialIndexes.Num() > 32 )
			{
				MaxClusterSize = 32;
			}
			else
			{
				break;
			}
		}
		
		// 2. 如果当前层 Cluster 数量 <= MaxGroupSize , 则当前层整体 ReduceGroup
		if( LevelClusters.Num() <= MaxGroupSize )
		{
			TArray< uint32, TInlineAllocator< MaxGroupSize > > Children;

			// 计算每个 Group 的所有 Cluster 合并为一个大的 Cluster 之后, 可以切分的最大父 Cluster 数量
			// 父 Cluster 数量取决于合并后的大 Cluster 中三角形 Mesh 的材质数量
			uint32 NumGroupElements = 0;
			for( FCluster& Cluster : LevelClusters )
			{
				NumGroupElements  += Cluster.MaterialIndexes.Num();
				Children.Add( LevelOffset++ );
			}
			uint32 MaxParents = FMath::DivideAndRoundUp( NumGroupElements, MaxClusterSize * 2 );

			LevelOffset = Clusters.Num();
			Clusters.AddDefaulted( MaxParents );
			Groups.AddDefaulted( 1 );

			ReduceGroup( RayTracingScene.Get(), NumClusters, Children, MaxClusterSize, MaxParents, Groups.Num() - 1, MeshIndex );

			check( LevelOffset < NumClusters );

			// Correct num to atomic count
			Clusters.SetNum( NumClusters, EAllowShrinking::No );

			continue;
		}

		// 3. 若当前层 Cluster 数量很多 ( > MaxGroupSize )

		// 首先根据每个 Cluster 的边界边来找 Clusters 之间的共享边
		struct FExternalEdge
		{
			uint32	ClusterIndex;
			int32	EdgeIndex;
		};
		TArray< FExternalEdge >	ExternalEdges;
		FHashTable				ExternalEdgeHash;
		TAtomic< uint32 >		ExternalEdgeOffset(0);

		// We have a total count of NumExternalEdges so we can allocate a hash table without growing.
		ExternalEdges.AddUninitialized( NumExternalEdges );
		ExternalEdgeHash.Clear( 1 << FMath::FloorLog2( NumExternalEdges ), NumExternalEdges );

		// Add edges to hash table
		// 构建每个 Cluster 的边界边的 ExternalEdgeHash, 用于快速匹配与其它 Cluster 的共享边
		ParallelFor( TEXT("Nanite.BuildDAG.PF"), LevelClusters.Num(), 32,
			[&]( uint32 ClusterIndex )
			{
				FCluster& Cluster = LevelClusters[ ClusterIndex ];

				if (CancellationToken && CancellationToken->IsCanceled())
				{
					return;
				}

				for( int32 EdgeIndex = 0; EdgeIndex < Cluster.ExternalEdges.Num(); EdgeIndex++ )
				{
					if( Cluster.ExternalEdges[ EdgeIndex ] )
					{
						uint32 VertIndex0 = Cluster.Indexes[ EdgeIndex ];
						uint32 VertIndex1 = Cluster.Indexes[ Cycle3( EdgeIndex ) ];
	
						const FVector3f& Position0 = Cluster.GetPosition( VertIndex0 );
						const FVector3f& Position1 = Cluster.GetPosition( VertIndex1 );

						uint32 Hash0 = HashPosition( Position0 );
						uint32 Hash1 = HashPosition( Position1 );
						uint32 Hash = Murmur32( { Hash0, Hash1 } );

						uint32 ExternalEdgeIndex = ExternalEdgeOffset++;
						ExternalEdges[ ExternalEdgeIndex ] = { ClusterIndex, EdgeIndex };
						ExternalEdgeHash.Add_Concurrent( Hash, ExternalEdgeIndex );
					}
				}
			});

		if (CancellationToken && CancellationToken->IsCanceled())
		{
			return;
		}

		check( ExternalEdgeOffset == ExternalEdges.Num() );

		TAtomic< uint32 > NumAdjacency(0);

		// Find matching edge in other clusters
		// 查找 Clusters 之间的共享边, 并构建 Clusters 之间的邻接关系
		ParallelFor( TEXT("Nanite.BuildDAG.PF"), LevelClusters.Num(), 32,
			[&]( uint32 ClusterIndex )
			{
				FCluster& Cluster = LevelClusters[ ClusterIndex ];

				if (CancellationToken && CancellationToken->IsCanceled())
				{
					return;
				}

				for( int32 EdgeIndex = 0; EdgeIndex < Cluster.ExternalEdges.Num(); EdgeIndex++ )
				{
					if( Cluster.ExternalEdges[ EdgeIndex ] )
					{
						uint32 VertIndex0 = Cluster.Indexes[ EdgeIndex ];
						uint32 VertIndex1 = Cluster.Indexes[ Cycle3( EdgeIndex ) ];
	
						const FVector3f& Position0 = Cluster.GetPosition( VertIndex0 );
						const FVector3f& Position1 = Cluster.GetPosition( VertIndex1 );

						uint32 Hash0 = HashPosition( Position0 );
						uint32 Hash1 = HashPosition( Position1 );
						uint32 Hash = Murmur32( { Hash1, Hash0 } );

						for( uint32 ExternalEdgeIndex = ExternalEdgeHash.First( Hash ); ExternalEdgeHash.IsValid( ExternalEdgeIndex ); ExternalEdgeIndex = ExternalEdgeHash.Next( ExternalEdgeIndex ) )
						{
							FExternalEdge ExternalEdge = ExternalEdges[ ExternalEdgeIndex ];

							FCluster& OtherCluster = LevelClusters[ ExternalEdge.ClusterIndex ];

							if( OtherCluster.ExternalEdges[ ExternalEdge.EdgeIndex ] )
							{
								uint32 OtherVertIndex0 = OtherCluster.Indexes[ ExternalEdge.EdgeIndex ];
								uint32 OtherVertIndex1 = OtherCluster.Indexes[ Cycle3( ExternalEdge.EdgeIndex ) ];
			
								if( Position0 == OtherCluster.GetPosition( OtherVertIndex1 ) &&
									Position1 == OtherCluster.GetPosition( OtherVertIndex0 ) )
								{
									if( ClusterIndex != ExternalEdge.ClusterIndex )
									{
										// Increase it's count
										Cluster.AdjacentClusters.FindOrAdd( ExternalEdge.ClusterIndex, 0 )++;

										// Can't break or a triple edge might be non-deterministically connected.
										// Need to find all matching, not just first.
									}
								}
							}
						}
					}
				}
				NumAdjacency += Cluster.AdjacentClusters.Num();

				// Force deterministic order of adjacency.
				// 排序, 保证确定性: 相同的 Mesh 输入, 得到相同的结果
				Cluster.AdjacentClusters.KeySort(
					[ &LevelClusters ]( uint32 A, uint32 B )
					{
						return LevelClusters[A].GUID < LevelClusters[B].GUID;
					} );
			});

		if (CancellationToken && CancellationToken->IsCanceled())
		{
			return;
		}

		// 基于 Clusters 之间的邻接关系构建并查集, 将 Clusters 划分为若干个连通块
		FDisjointSet DisjointSet( LevelClusters.Num() );

		for( uint32 ClusterIndex = 0; ClusterIndex < (uint32)LevelClusters.Num(); ClusterIndex++ )
		{
			for( auto& Pair : LevelClusters[ ClusterIndex ].AdjacentClusters )
			{
				uint32 OtherClusterIndex = Pair.Key;

				uint32 Count = LevelClusters[ OtherClusterIndex ].AdjacentClusters.FindChecked( ClusterIndex );
				check( Count == Pair.Value );

				if( ClusterIndex > OtherClusterIndex )
				{
					DisjointSet.UnionSequential( ClusterIndex, OtherClusterIndex );
				}
			}
		}

		// 创建图划分器
		FGraphPartitioner Partitioner( LevelClusters.Num(), MinGroupSize, MaxGroupSize );

		// 根据 Cluster 包围盒中心点构建 Clusters 之间在空间上的邻近信息
		auto GetCenter = [&]( uint32 Index )
		{
			FBounds3f& Bounds = LevelClusters[ Index ].Bounds;
			return 0.5f * ( Bounds.Min + Bounds.Max );
		};
		Partitioner.BuildLocalityLinks( DisjointSet, TotalBounds, TArrayView< const int32 >(), GetCenter );

		if (CancellationToken && CancellationToken->IsCanceled())
		{
			return;
		}

		// 创建图对象, 每个节点表示一个 Cluster
		auto* RESTRICT Graph = Partitioner.NewGraph( NumAdjacency );

		for( int32 i = 0; i < LevelClusters.Num(); i++ )
		{
			Graph->AdjacencyOffset[i] = Graph->Adjacency.Num();

			uint32 ClusterIndex = Partitioner.Indexes[i];

			for( auto& Pair : LevelClusters[ ClusterIndex ].AdjacentClusters )
			{
				uint32 OtherClusterIndex = Pair.Key;
				uint32 NumSharedEdges = Pair.Value;

				const auto& Cluster0 = Clusters[ LevelOffset + ClusterIndex ];
				const auto& Cluster1 = Clusters[ LevelOffset + OtherClusterIndex ];

				// TODO: 本来在同一个 Group 的权重反而更低???
				bool bSiblings = Cluster0.GroupIndex != MAX_uint32 && Cluster0.GroupIndex == Cluster1.GroupIndex;

				// 拓扑上邻接的 Cluster 之间设置一个大权重
				Partitioner.AddAdjacency( Graph, OtherClusterIndex, NumSharedEdges * ( bSiblings ? 1 : 16 ) + 4 );
			}

			// 空间上邻近的 Cluster 之间设置一个极小权重
			Partitioner.AddLocalityLinks( Graph, ClusterIndex, 1 );
		}
		Graph->AdjacencyOffset[ Graph->Num ] = Graph->Adjacency.Num();

		LOG_CRC( Graph->Adjacency );
		LOG_CRC( Graph->AdjacencyCost );
		LOG_CRC( Graph->AdjacencyOffset );
		
		bool bSingleThreaded = LevelClusters.Num() <= 32;

		// 递归二分划分子 Partition
		Partitioner.PartitionStrict( Graph, !bSingleThreaded );

		LOG_CRC( Partitioner.Ranges );

		// 累加所有子 Partition 要生成的父 Cluster 数量, 以及要新增的 Group 数量
		uint32 MaxParents = 0;
		for( auto& Range : Partitioner.Ranges )
		{
			uint32 NumGroupElements = 0;
			for( uint32 i = Range.Begin; i < Range.End; i++ )
			{
				// Global indexing is needed in Reduce()
				Partitioner.Indexes[i] += LevelOffset;
				NumGroupElements += Clusters[ Partitioner.Indexes[i] ].MaterialIndexes.Num();
			}
			MaxParents += FMath::DivideAndRoundUp( NumGroupElements, MaxClusterSize * 2 );
		}

		LevelOffset = Clusters.Num();

		Clusters.AddDefaulted( MaxParents );
		Groups.AddDefaulted( Partitioner.Ranges.Num() );

		// 对每个子 Partition 做 ReduceGroup
		ParallelFor( TEXT("Nanite.BuildDAG.PF"), Partitioner.Ranges.Num(), 1,
			[&]( int32 PartitionIndex )
			{
				if (CancellationToken && CancellationToken->IsCanceled())
				{
					return;
				}

				auto& Range = Partitioner.Ranges[ PartitionIndex ];

				TArrayView< uint32 > Children( &Partitioner.Indexes[ Range.Begin ], Range.End - Range.Begin );

				uint32 NumGroupElements = 0;
				for( uint32 i = Range.Begin; i < Range.End; i++ )
				{
					NumGroupElements += Clusters[ Partitioner.Indexes[i] ].MaterialIndexes.Num();
				}
				uint32 MaxParents = FMath::DivideAndRoundUp( NumGroupElements, MaxClusterSize * 2 );
				uint32 ClusterGroupIndex = PartitionIndex + Groups.Num() - Partitioner.Ranges.Num();

				ReduceGroup( RayTracingScene.Get(), NumClusters, Children, MaxClusterSize, MaxParents, ClusterGroupIndex, MeshIndex );
			} );

		if (CancellationToken && CancellationToken->IsCanceled())
		{
			return;
		}

		check( LevelOffset < NumClusters );

		// Correct num to atomic count
		Clusters.SetNum( NumClusters, EAllowShrinking::No );

		// Force a deterministic order of the generated parent clusters
		{
			// TODO: Optimize me.
			// Just sorting the array directly seems like the safest option at this stage (right before UE5 final build).
			// On AOD_Shield this seems to be on the order of 0.01s in practice.
			// As the Clusters array is already conservatively allocated, it seems storing the parent clusters in their designated
			// conservative ranges and then doing a compaction pass at the end would be a more efficient solution that doesn't involve sorting.
			
			//uint32 StartTime = FPlatformTime::Cycles();
			TArrayView< FCluster > Parents( &Clusters[ LevelOffset ], Clusters.Num() - LevelOffset );
			Parents.Sort(
				[&]( const FCluster& A, const FCluster& B )
				{
					return A.GUID < B.GUID;
				} );
			//UE_LOG(LogStaticMesh, Log, TEXT("SortTime Adjacency [%.2fs]"), FPlatformTime::ToMilliseconds(FPlatformTime::Cycles() - StartTime) / 1000.0f);
		}
		
	}

#if RAY_TRACE_VOXELS
	for( FCluster& Cluster : Clusters )
	{
		Cluster.ExtraVoxels.Empty();	// VOXELTODO: Free this earlier
	}
#endif
	
	// Max out root node
	// 最后构建一个 RootGroup: 此时 LevelOffset 指向此次构建的最高层 (只剩 1 个 Cluster) 的起始位置, 也就是 RootClusterIndex
	uint32 RootIndex = LevelOffset;
	FClusterGroup RootClusterGroup;
	RootClusterGroup.Children.Add( RootIndex );
	RootClusterGroup.Bounds				= Clusters[ RootIndex ].SphereBounds;
	RootClusterGroup.LODBounds			= FSphere3f( 0 );
	RootClusterGroup.MaxParentLODError	= 1e10f;
	RootClusterGroup.MinLODError		= -1.0f;
	RootClusterGroup.MipLevel			= Clusters[ RootIndex ].MipLevel + 1;
	RootClusterGroup.MeshIndex			= MeshIndex;
	RootClusterGroup.AssemblyPartIndex	= MAX_uint32;
	RootClusterGroup.bTrimmed			= false;
	Clusters[ RootIndex ].GroupIndex = Groups.Num();
	Groups.Add( RootClusterGroup );
}
```

**总结**：

从**叶子 Cluster 层开始**，循环把当前层的 Cluster 通过 METIS 库按**拓扑邻接关系（大权重）**和**空间邻近关系（小权重）**分 Group ，**每个 Group 做 `ReduceGroup` 生成若干父 Cluster （生成的父 Cluster 的最大数量 MaxParents 取决于每个 Group 中三角形 Mesh 的材质数量），父 Cluster 再成为下一层 Cluster 输入，直到只剩一个 Cluster ，这个 Cluster 叫做根 Cluster ，根 Cluster 单独属于根 Group**

### 4.1. `FClusterDAG::ReduceGroup`

```cpp
void FClusterDAG::ReduceGroup( FRayTracingScene* RayTracingScene, TAtomic< uint32 >& NumClusters, TArrayView< uint32 > Children, uint32 MaxClusterSize, uint32 NumParents, int32 GroupIndex, uint32 MeshIndex )
{
	check( GroupIndex >= 0 );

	bool bAnyTriangles = false;
	bool bAllTriangles = true;

	TArray< FSphere3f, TInlineAllocator< MaxGroupSize > > Children_LODBounds;
	TArray< FSphere3f, TInlineAllocator< MaxGroupSize > > Children_SphereBounds;

	float ChildMinLODError = MAX_flt;
	float ChildMaxLODError = 0.0f;
	for( uint32 Child : Children )
	{
		FCluster& Cluster = Clusters[ Child ];

		bAnyTriangles = bAnyTriangles || Cluster.NumTris > 0;
		bAllTriangles = bAllTriangles && Cluster.NumTris > 0;

		// 是否是叶子 Cluster
		bool bLeaf = Cluster.EdgeLength < 0.0f;
		float LODError = Cluster.LODError;

		// Force monotonic nesting.
		Children_LODBounds.Add( Cluster.LODBounds );
		Children_SphereBounds.Add( Cluster.SphereBounds );

		// 强制 LOD 单调性: 父层 Cluster 的 LODError >= 子层 Cluster 的 LODError, 防止出现父层 Cluster 比子层 Cluster 更精细 (这是错误的)
		ChildMinLODError = FMath::Min( ChildMinLODError, bLeaf ? -1.0f : LODError );
		ChildMaxLODError = FMath::Max( ChildMaxLODError, LODError );

		// 建立 Group 与子 Cluster 的关系
		Cluster.GroupIndex = GroupIndex;
		Groups[ GroupIndex ].Children.Add( Child );
		check( Groups[ GroupIndex ].Children.Num() <= NANITE_MAX_CLUSTERS_PER_GROUP_TARGET );
	}
	
	// 计算 Group 得包围体
	FSphere3f ParentLODBounds( Children_LODBounds.GetData(), Children_LODBounds.Num() );
	FSphere3f ParentBounds( Children_SphereBounds.GetData(), Children_SphereBounds.Num() );

	uint32 ParentStart = 0;
	uint32 ParentEnd = 0;

	FCluster Merged;
	float SimplifyError = MAX_flt;

	bool bVoxels = false;

	uint32 TargetClusterSize = MaxClusterSize - 2;
	if( bAllTriangles )
	{
		// 初始简化目标三角形 Mesh 数量
		uint32 TargetNumTris = NumParents * TargetClusterSize;

		{
			// 将子 Cluster 合并为一个大的 Cluster
			Merged = FCluster( *this, Children );
			// 简化到目标三角形 Mesh 数量
			SimplifyError = Merged.Simplify( *this, TargetNumTris );
		}
	}

	if( !bVoxels )
	{
		check( bAllTriangles );

		// 尝试多次简化并切分父 Cluster
		while(1)
		{
			// 将简化后的大 Cluster 切分为若干个父 Cluster
			bool bSplitSuccess = SplitCluster< FGraphPartitioner >( Merged, Clusters, NumClusters, MaxClusterSize, NumParents, ParentStart, ParentEnd,
				[ &Merged ]( FGraphPartitioner& Partitioner, FAdjacency& Adjacency )
				{
					Merged.Split( Partitioner, Adjacency );
				} );

			// 如果切分父 Cluster 成功, 结束
			if( bSplitSuccess )
				break;

			// 如果切分父 Cluster 失败, 则尝试降低目标三角形 Mesh 数量, 再进行尝试
			TargetClusterSize -= 2;
			if( TargetClusterSize <= MaxClusterSize / 2 )
				break;

			uint32 TargetNumTris = NumParents * TargetClusterSize;

			// Start over from scratch. Continuing from simplified cluster screws up ExternalEdges and LODError.
			// 回退到原始未简化版本, 重新开始简化再切分
			Merged = FCluster( *this, Children );
			SimplifyError = Merged.Simplify( *this, TargetNumTris );
		}
	}

	float ParentMaxLODError = FMath::Max( ChildMaxLODError, SimplifyError );

	// Force parents to have same LOD data. They are all dependent.
	// 统一父 Cluster 的 LOD 信息, 它们都是相互依赖的
	for( uint32 Parent = ParentStart; Parent < ParentEnd; Parent++ )
	{
		Clusters[ Parent ].LODBounds			= ParentLODBounds;
		Clusters[ Parent ].LODError				= ParentMaxLODError;
		Clusters[ Parent ].GeneratingGroupIndex = GroupIndex;
	}

	// 填充 Group 结构
	Groups[ GroupIndex ].Bounds				= ParentBounds;
	Groups[ GroupIndex ].LODBounds			= ParentLODBounds;
	Groups[ GroupIndex ].MinLODError		= ChildMinLODError;
	Groups[ GroupIndex ].MaxParentLODError	= ParentMaxLODError;
	Groups[ GroupIndex ].MipLevel			= Clusters[ Children[0] ].MipLevel;
	Groups[ GroupIndex ].MeshIndex			= MeshIndex;
	Groups[ GroupIndex ].AssemblyPartIndex	= MAX_uint32;
	Groups[ GroupIndex ].bTrimmed			= false;
}
```

**总结**：

把若干**子 Cluster （Children）合并、简化、再切分成父 Cluster （Parents）**，并建立**Group 与子 Cluster 的层级关系**

#### 4.1.1. 简化 `FCluster::Simplify`

**总结**：

把 Cluster 的顶点数据喂给 `FMeshSimplifier` （基于 QEM （Quadric Error Metrics） + 属性 Quadric）做边折叠

##### 4.1.1.1 统计 UV 面积 + 标记 UV 镜像

```cpp
float UVArea[ MAX_STATIC_TEXCOORDS ] = { 0.0f };
if( VertexFormat.NumTexCoords > 0 )
{
	// 遍历所有三角形 Mesh, 统计每个 UV 通道的 "UV 面积" , 同时向 MaterialIndexes[ TriIndex ] 中写入每个 UV 通道的 UV 镜像标记
	for( uint32 TriIndex = 0; TriIndex < NumTris; TriIndex++ )
	{
		uint32 Index0 = Indexes[ TriIndex * 3 + 0 ];
		uint32 Index1 = Indexes[ TriIndex * 3 + 1 ];
		uint32 Index2 = Indexes[ TriIndex * 3 + 2 ];

		FVector2f* UV0 = GetUVs( Index0 );
		FVector2f* UV1 = GetUVs( Index1 );
		FVector2f* UV2 = GetUVs( Index2 );

		for( uint32 UVIndex = 0; UVIndex < VertexFormat.NumTexCoords; UVIndex++ )
		{
			FVector2f EdgeUV1 = UV1[ UVIndex ] - UV0[ UVIndex ];
			FVector2f EdgeUV2 = UV2[ UVIndex ] - UV0[ UVIndex ];
			float SignedArea = 0.5f * ( EdgeUV1 ^ EdgeUV2 );
			UVArea[ UVIndex ] += FMath::Abs( SignedArea );

			// Force an attribute discontinuity for UV mirroring edges.
			// Quadric could account for this but requires much larger UV weights which raises error on meshes which have no visible issues otherwise.
			// 将每个 UV 通道的 UV 镜像(SignedArea 的符号) 编码到 MaterialIndexes[ TriIndex ] 的高位中, 最大支持 MAX_STATIC_TEXCOORDS(8) 个 UV 通道
			// 也就是说: MaterialIndexes[ TriIndex ] 的高 8 位中的每一位记录了每个 UV 通道的 UV 镜像标记, MaterialIndexes[ TriIndex ] 的低 24 位记录的还是材质 ID
			MaterialIndexes[ TriIndex ] |= ( SignedArea >= 0.0f ? 1 : 0 ) << ( UVIndex + 24 );
		}
	}
}
```

- UV 面积用来做 UV 权重的归一化

	后续需要给简化器（ `FMeshSimplifier` ）设置 UV 属性权重，如果 UV 属性权重不通过 UV 面积做归一化，会导致：

	- UV 很密集（ UV 三角形很小）时， UV 误差巨大，简化器不敢动
	- UV 很稀疏（ UV 三角形很大）时， UV 误差很小，简化器会把 UV 破坏的很厉害

- 把 UV 镜像编码到 `MaterialIndexes[ TriIndex ]` 的高位

	后续在函数 `FMeshSimplifier::CalcEdgeQuadric` 中判断是否存在对向共享边时，会**要求 MaterialIndex 相同**，这意味着：

	- 同样几何边，如果两侧三角形 Mesh 的 UV 镜像符号不同，它们的 MaterialIndex （因为高位的 UV 镜像标记不同）就不同，简化器会认为这条边没有对向共享边（或属性不连续），从而添加**边界约束（ Edge Constraint ）**，避免把 UV 镜像缝合处 Collapse 从而造成严重贴图反转的问题

	需要注意的是，在函数最后还会把这些高位上的 UV 镜像标记清空： `MaterialIndexes[ TriIndex ] &= 0xffffff`

##### 4.1.1.2. 模型尺度归一化

```cpp
// 估算平均三角形 Mesh 面积
float TriangleSize = FMath::Sqrt( SurfaceArea / (float)NumTris );
	
FFloat32 CurrentSize( FMath::Max( TriangleSize, THRESH_POINTS_ARE_SAME ) );
FFloat32 DesiredSize( 0.25f );
FFloat32 FloatScale( 1.0f );

// Lossless scaling by only changing the float exponent.
// 只通过指数差来决定缩放倍数
int32 Exponent = FMath::Clamp( (int)DesiredSize.Components.Exponent - (int)CurrentSize.Components.Exponent, -126, 127 );
FloatScale.Components.Exponent = Exponent + 127;	//ExpBias
// Scale ~= DesiredSize / CurrentSize
float PositionScale = FloatScale.FloatValue;

for( uint32 i = 0; i < NumVerts; i++ )
{
	GetPosition(i) *= PositionScale;
}
TargetError *= PositionScale;
```

对于按照 IEEE 754 标准存储和计算的单精度浮点数（32-bit float）来说（忽略特殊值：非规格化数（Exponent = 0 ，非常接近 0 的数）、正负无穷大（Exponent = 255 且 Mantissa = 0）和 NaN （Exponent = 255 且 Mantissa $\not ={0}$）），数值大概是： `value = (-1)^Sign * (1 + Mantissa/(2^23)) * 2^(Exponent - bias)` ，其中 `bias = 127` 。

如果想用缩放因子 `S` 把 `CurrentSize` 缩放到接近 `DesiredSize` ，最理想的是： `S = DesiredSize / CurrentSize` ，但这里**只用指数来近似**： `(int)DesiredSize.Components.Exponent - (int)CurrentSize.Components.Exponent` 相当于在 `log2` 空间做差，得到的是一个整数 `k` ，使缩放因子为 `S = 2^k` ，最终的 `FloatScale = 1.0 * 2^k` 。使用这种**无损浮点缩放**的方法：

- 不会引入额外的误差，并且没有精度损失，对后续几何处理来说更稳定
- 它只用 `Exponent` 来匹配量级，因此只是把 `CurrentSize` 缩放到与 `DesiredSize` **同一个量级**（例如 `CurrentSize = 0.3f; DesiredSize = 0.25f` ，最终的 `FloatScale` 还是 `1.0f` ），所以这种无损浮点缩放更像是**把模型尺度归一化到一个合理的区间**

##### 4.1.1.3. 计算属性权重 AttributeWeights

```css
Attribute Weights:

Normal.xyz	 -> 1.0						  // 法线: 非常重要
Tangent.xyz	-> 0.0625					   // 切线: 较重要
Tangent.w	  -> 0.5						  // 副切线方向: 重要
Color.xyzw	 -> 0.0625					   // 顶点颜色: 较重要
UVs			-> 1/(128 * TriangleUVSize)	 // UV: 按 UV 尺度归一化
BoneIndex	 -> 0.0						  // Skinning 相关顶点属性权重为 0
BoneWeight	-> 0.0						  // Skinning 相关顶点属性权重为 0
```

为什么将 Skinning 相关的顶点属性（ BoneIndex / BoneWeight ）权重设置为 0 ？

- 评估将两个顶点合并成一个新的顶点时的**代价（ Error ）**时，不应该考虑 Skinning 相关的顶点属性
- 在对每个 Cluster 做**边折叠（ Edge Collapse ）**时，新生成的简化顶点的属性通常都是通过线性插值得到的， UV 、颜色这类属性是可以安全插值的，但 Skinning 属性却不可以，原因是：

	- BoneIndex 是离散数据，骨骼索引不能插值，否则会变成非法骨骼（例如： `BoneIndex = lerp(12, 3, 0.5)` ）
	- 对 BoneWeight 线性插值会破坏归一化

##### 4.1.1.4. 构建 `FMeshSimplifier`

```cpp
FMeshSimplifier::FMeshSimplifier( float* InVerts, uint32 InNumVerts, uint32* InIndexes, uint32 InNumIndexes, int32* InMaterialIndexes, uint32 InNumAttributes )
	: NumVerts( InNumVerts )
	, NumIndexes( InNumIndexes )
	, NumAttributes( InNumAttributes )
	, NumTris( NumIndexes / 3 )
	, RemainingNumVerts( NumVerts )
	, RemainingNumTris( NumTris )
	, Verts( InVerts )
	, Indexes( InIndexes )
	, MaterialIndexes( InMaterialIndexes )
	, VertHash( 1 << FMath::Min( 16u, FMath::FloorLog2( NumVerts ) ) )
	, CornerHash( 1 << FMath::Min( 16u, FMath::FloorLog2( NumIndexes ) ) )
	, TriRemoved( false, NumTris )
{
	// 构建 VertHash, 按 Position 哈希
	for( uint32 VertIndex = 0; VertIndex < NumVerts; VertIndex++ )
	{
		VertHash.Add( HashPosition( GetPosition( VertIndex ) ), VertIndex );
	}

	VertRefCount.AddZeroed( NumVerts );
	CornerFlags.AddZeroed( NumIndexes );
	
	EdgeQuadrics.AddUninitialized( NumIndexes );
	EdgeQuadricsValid.Init( false, NumIndexes );

	// Guess number of edges based on Euler's formula.
	uint32 NumEdges = FMath::Min3( NumIndexes, 3 * NumVerts - 6, NumTris + NumVerts );
	Pairs.Reserve( NumEdges );
	PairHash0.Clear( 1 << FMath::Min( 16u, FMath::FloorLog2( NumEdges ) ), NumEdges );
	PairHash1.Clear( 1 << FMath::Min( 16u, FMath::FloorLog2( NumEdges ) ), NumEdges );

	// 遍历所有 Corner (每个 Corner 就是 IndexBuffer 中的一个 Index) 构建 Pair
	for( uint32 Corner = 0; Corner < NumIndexes; Corner++ )
	{
		uint32 VertIndex = Indexes[ Corner ];
		
		// 统计每个顶点被多少个 Corner 引用
		VertRefCount[ VertIndex ]++;

		// 构建 CornerHash, 按 Position 哈希
		const FVector3f& Position = GetPosition( VertIndex );
		CornerHash.Add( HashPosition( Position ), Corner );

		uint32 OtherCorner = Cycle3( Corner );

		// 生成 Edge Pair
		FPair Pair;
		Pair.Position0 = Position;
		Pair.Position1 = GetPosition( Indexes[ Cycle3( Corner ) ] );

		// 去重, 保证每条拓扑边只生成一次, 后续 Collapse 不重复计算
		if( AddUniquePair( Pair, Pairs.Num() ) )
		{
			Pairs.Add( Pair );
		}
	}
}

FMeshSimplifier Simplifier( Verts.GetData(), NumVerts, Indexes.GetData(), Indexes.Num(), MaterialIndexes.GetData(), NumAttributes );
```

**总结**：

把原始三角形 Mesh 转换成适合快速查找、合并和误差计算的内部数据结构：

- `VertHash` ：遍历每个顶点，按顶点 Position 哈希
- `CornerHash` ：遍历每个 **Corner （也就是 IndexBuffer 中的每个 Index ）**，按顶点 Position 哈希
- `EdgePair` ：遍历每个 Corner ，按结构体 `FPair` 哈希，结构体 `FPair` 中的变量要满足 `Hash(FPair::Position0) < Hash(FPair::Position1)` （如果不满足就 `Swap(FPair::Position0, FPair::Position1)` ），并通过 `PairHash0` 和 `PairHash1` 去重

##### 4.1.1.5. 锁定 `ExternalEdges`

```cpp
// 锁定 Cluster 的 ExternalEdges, 边界边的顶点禁止 Collapse
TMap< TTuple< FVector3f, FVector3f >, int8 > LockedEdges;

for( int32 EdgeIndex = 0; EdgeIndex < ExternalEdges.Num(); EdgeIndex++ )
{
	if( ExternalEdges[ EdgeIndex ] )
	{
		uint32 VertIndex0 = Indexes[ EdgeIndex ];
		uint32 VertIndex1 = Indexes[ Cycle3( EdgeIndex ) ];

		const FVector3f& Position0 = GetPosition( VertIndex0 );
		const FVector3f& Position1 = GetPosition( VertIndex1 );

		// 所有引用边界边顶点的 Corner 打上被锁标记 LockedVertMask
		Simplifier.LockPosition( Position0 );
		Simplifier.LockPosition( Position1 );

		// 保存被锁的边界边, 简化后使用此数据重建 Cluster 的 ExternalEdges
		LockedEdges.Add( MakeTuple( Position0, Position1 ), ExternalEdges[ EdgeIndex ] );
	}
}
```

**总结**：

所有位于 Cluster 外轮廓的边（边界边）被标记为不可 Collapse （ `LockedVertMask` ），以保证简化不会改变 Cluster 的边界拓扑与几何，从而避免与相邻 Cluster 之间产生裂缝

##### 4.1.1.6. Cluster 简化 `FMeshSimplifier::Simplify`

通过**边折叠（ Edge Collapse ）**，把当前 Cluster 简化到满足目标（顶点数、三角形 Mesh 数和误差范围内）

```cpp
float FMeshSimplifier::Simplify(
	uint32 TargetNumVerts, uint32 TargetNumTris, float TargetError,
	uint32 LimitNumVerts, uint32 LimitNumTris, float LimitError )
{
	check( TargetNumVerts < NumVerts || TargetNumTris < NumTris || TargetError > 0.0f );
	check( TargetNumVerts >= LimitNumVerts );
	check( TargetNumTris >= LimitNumTris );
	check( TargetError <= LimitError );

	for( uint32 i = 0; i < NumAttributes; i++ )
	{
		if( AttributeWeights[i] == 0.0f )
		{
			bZeroWeights = true;
			break;
		}
	}

	const SIZE_T QuadricSize = sizeof( FQuadricAttr ) + NumAttributes * 4 * sizeof( QScalar );

	TriQuadrics.AddUninitialized( NumTris * QuadricSize );

	// 简化前修正三角形 Mesh 数据:
	// 1. 确保三角形 Mesh 顶点索引合法
	// 2. 删除退化的 Triangle, 删除重复的顶点, 删除重复 Triangle
	// 3. 计算 TriQuadric
	for( uint32 TriIndex = 0; TriIndex < NumTris; TriIndex++ )
	{
		FixUpTri( TriIndex );
	}

	// 遍历 Index 计算 EdgeQuadric
	for( uint32 i = 0; i < NumIndexes; i++ )
	{
		CalcEdgeQuadric(i);
	}

	// Initialize heap
	// 初始化最小堆
	PairHeap.Resize( Pairs.Num(), Pairs.Num() );
	
	// 评估 Collapse 每个 Pair 的代价, 并将其 Add 进最小堆
	for( uint32 PairIndex = 0, Num = Pairs.Num(); PairIndex < Num; PairIndex++ )
	{
		FPair& Pair = Pairs[ PairIndex ];

		float MergeError = EvaluateMerge( Pair.Position0, Pair.Position1, false );
		PairHeap.Add( MergeError, PairIndex );
	}

	float MaxError = 0.0f;

	// 主循环: 不断取最小代价的 Pair 并进行 Collapse
	while( PairHeap.Num() > 0 )
	{
		uint32 PrevNumVerts = RemainingNumVerts;
		uint32 PrevNumTris  = RemainingNumTris;

		// 堆顶是当前最小代价的 Pair
		// 如果连最小代价都超过 LimitError, 停止 Collapse
		if( PairHeap.GetKey( PairHeap.Top() ) > LimitError )
			break;

		{
			// 弹出堆顶并执行 Collapse
			uint32 PairIndex = PairHeap.Top();
			PairHeap.Pop();

			FPair& Pair = Pairs[ PairIndex ];
		
			PairHash0.Remove( HashPosition( Pair.Position0 ), PairIndex );
			PairHash1.Remove( HashPosition( Pair.Position1 ), PairIndex );

			// 执行 Collapse
			float MergeError = EvaluateMerge( Pair.Position0, Pair.Position1, true );
			MaxError = FMath::Max( MaxError, MergeError );
		}

		// 达到目标, 停止 Collapse
		if( RemainingNumVerts	<= TargetNumVerts &&
			RemainingNumTris	<= TargetNumTris &&
			MaxError			>= TargetError )
		{
			break;
		}

		// 达到 Limit 限制, 停止 Collapse
		if( RemainingNumVerts	<= LimitNumVerts ||
			RemainingNumTris	<= LimitNumTris ||
			MaxError			>= LimitError )
		{
			break;
		}

		// 重新评估受 Collapse 影响的 Pair 的代价并 Add 进最小堆
		for( uint32 PairIndex : ReevaluatePairs )
		{
			FPair& Pair = Pairs[ PairIndex ];

			float MergeError = EvaluateMerge( Pair.Position0, Pair.Position1, false );
			PairHeap.Add( MergeError, PairIndex );
		}
		ReevaluatePairs.Reset();
	}

	// If couldn't hit targets through regular edge collapses, resort to randomly removing triangles.
	// 如果通过普通的 Collapse 仍然无法达到目标，那就考虑随机移除三角形 Mesh
	{
		uint32 TriIndex = 0;
		while(1)
		{
			if( RemainingNumVerts	<= TargetNumVerts &&
				RemainingNumTris	<= TargetNumTris &&
				MaxError			>= TargetError )
			{
				break;
			}
	
			if( RemainingNumVerts	<= LimitNumVerts ||
				RemainingNumTris	<= LimitNumTris ||
				MaxError			>= LimitError )
			{
				break;
			}

			while( TriRemoved[ TriIndex ] )
				TriIndex++;
	
			// 随机移除三角形 Mesh
			RemoveTri( TriIndex );
		}
	}
	
	return MaxError;
}
```

###### 4.1.1.6.1. `FMeshSimplifier::FixUpTri`

```cpp
void FMeshSimplifier::FixUpTri( uint32 TriIndex )
{
	check( !TriRemoved[ TriIndex ] );

	const FVector3f& p0 = GetPosition( Indexes[ TriIndex * 3 + 0 ] );
	const FVector3f& p1 = GetPosition( Indexes[ TriIndex * 3 + 1 ] );
	const FVector3f& p2 = GetPosition( Indexes[ TriIndex * 3 + 2 ] );

	bool bRemoveTri = CornerFlags[ TriIndex * 3 ] & RemoveTriMask;

	if( !bRemoveTri )
	{
		// Remove degenerates
		// 删除退化的三角形 Mesh
		bRemoveTri =
			p0 == p1 ||
			p1 == p2 ||
			p2 == p0;
	}

	if( !bRemoveTri )
	{
		// 删除重复顶点
		for( uint32 k = 0; k < 3; k++ )
		{
			RemoveDuplicateVerts( TriIndex * 3 + k );
		}

		// 删除重复三角形 Mesh
		bRemoveTri = IsDuplicateTri( TriIndex );
	}

	if( bRemoveTri )
		RemoveTri( TriIndex );
	else
		CalcTriQuadric( TriIndex );	// 计算 TriQuadric
}
```

**总结**：

删除退化的三角形 Mesh 、删除重复顶点、删除重复三角形 Mesh ，并计算三角形 Mesh 的 Quadric

###### 4.1.1.6.2. `FMeshSimplifier::CalcEdgeQuadric`

```cpp
void FMeshSimplifier::CalcEdgeQuadric( uint32 EdgeIndex )
{
	uint32 TriIndex = EdgeIndex / 3;

	// 被删除的三角形 Mesh 不再贡献任何约束
	if( TriRemoved[ TriIndex ] )
	{
		EdgeQuadricsValid[ EdgeIndex ] = false;
		return;
	}

	int32 MaterialIndex = MaterialIndexes[ TriIndex ];

	uint32 VertIndex0 = Indexes[ EdgeIndex ];
	uint32 VertIndex1 = Indexes[ Cycle3( EdgeIndex ) ];

	const FVector3f& Position0 = GetPosition( VertIndex0 );
	const FVector3f& Position1 = GetPosition( VertIndex1 );
	
	// Find edge with opposite direction that shares these 2 verts.
	// If none then we need to add an edge constraint.
	/*
		  /\
		 /  \
		o-<<-o
		o->>-o
		 \  /
		  \/
	*/
	// 先尝试找 拓扑连续+属性连续 的对向共享边
	uint32 Hash = HashPosition( Position1 );
	uint32 Corner;
	for( Corner = CornerHash.First( Hash ); CornerHash.IsValid( Corner ); Corner = CornerHash.Next( Corner ) )
	{
		uint32 OtherVertIndex0 = Indexes[ Corner ];
		uint32 OtherVertIndex1 = Indexes[ Cycle3( Corner ) ];
		
		// 不仅仅是拓扑连续 (共享顶点), 还需要保证属性连续 (同材质 ID)
		// TODO-注意0: 使用顶点索引 Index 判断严格拓扑共享, 避免把 "仅位置相同" 误判为共享
		// TODO-注意1: 这里的同材质 ID 还意味着 UV 镜像标记位 (高 8 位) 也相同
		if( VertIndex0 == OtherVertIndex1 &&
			VertIndex1 == OtherVertIndex0 &&
			MaterialIndex == MaterialIndexes[ Corner / 3 ] )
		{
			// Found matching edge.
			// No constraints needed so remove any that exist.
			// 如果满足就表示这条边是两个三角形 Mesh 共享的内部边, 在 拓扑+属性 上是连续的, 这种边不应该被额外约束, 否则会过度阻止正常 Collapse
			EdgeQuadricsValid[ EdgeIndex ] = false;
			return;
		}
	}

	// Don't double count attribute discontinuities.
	// 再尝试找仅 拓扑连续 的对向共享边
	float Weight = EdgeWeight;
	for( Corner = CornerHash.First( Hash ); CornerHash.IsValid( Corner ); Corner = CornerHash.Next( Corner ) )
	{
		uint32 OtherVertIndex0 = Indexes[ Corner ];
		uint32 OtherVertIndex1 = Indexes[ Cycle3( Corner ) ];
			
		if( Position0 == GetPosition( OtherVertIndex1 ) &&
			Position1 == GetPosition( OtherVertIndex0 ) )
		{
			// Found matching edge.
			// 此情况会在两个方向的 EdgeIndex 上各出现一次 (v0->v1, v1->v0), 为了避免同一条边被计算两次太强, 这里把约束权重减半
			Weight *= 0.5f;
			break;
		}
	}

	// Didn't find matching edge. Add edge constraint.
	// 仅找到 拓扑连续 的对向共享边 (此时权重减半) 或没有找到任何连续的对象共享边, 正常计算约束
	EdgeQuadrics[ EdgeIndex ] = FEdgeQuadric( GetPosition( VertIndex0 ), GetPosition( VertIndex1 ), Weight );
	EdgeQuadricsValid[ EdgeIndex ] = true;
}
```

**总结**：

判断每条有向边是否处在**边界（没有对向共享边）**或**属性不连续（仅拓扑连续，但属性上有断裂（ MaterialIndex 不相等））**处，如果是，就为这条有向边计算一个**约束 EdgeQuadric （ Collapse 惩罚项）**

###### 4.1.1.6.3. `FMeshSimplifier::EvaluateMerge`

```cpp
uint32 FMeshSimplifier::CornerIndexMoved( uint32 TriIndex ) const
{
	// 根据 CornerFlags 判断被移动的是哪个 Corner, 返回值 IndexMoved 有 3 种情况:
	// 0/1/2: 只有一个 Corner 会被移动
	// 3: 没有 Corner 会被移动
	// 4: 有两个或三个 Corner 会被移动
	uint32 IndexMoved = 3;
	for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
	{
		uint32 Corner = TriIndex * 3 + CornerIndex;

		if( CornerFlags[ Corner ] & MergeMask )
		{
			if( IndexMoved == 3 )
				IndexMoved = CornerIndex;
			else
				IndexMoved = 4;
		}
	}
	return IndexMoved;
}

bool FMeshSimplifier::TriWillInvert( uint32 TriIndex, const FVector3f& NewPosition ) const
{
	uint32 IndexMoved = CornerIndexMoved( TriIndex );

	// 只在有一个 Corner 被移动时做翻转检测
	// 当有两个或三个 Corner 被移动时, 跳过翻转检测, 此时三角形 Mesh 大概率会退化并在 FixUpTri 函数中被删除
	if( IndexMoved < 3 )
	{
		uint32 Corner = TriIndex * 3 + IndexMoved;

		// 取被移动的那个 Corner 的位置 p0, 以及其所属三角形 Mesh 的另外两个 Corner p1 和 p2
		const FVector3f& p0 = GetPosition( Indexes[ Corner ] );
		const FVector3f& p1 = GetPosition( Indexes[ Cycle3( Corner ) ] );
		const FVector3f& p2 = GetPosition( Indexes[ Cycle3( Corner, 2 ) ] );

		const FVector3f d21 = p2 - p1;
		const FVector3f d01 = p0 - p1;
		const FVector3f dp1 = NewPosition - p1;

		// 旧三角形 Mesh 的法线
		FVector3f n0 = d01 ^ d21;
		// 新三角形 Mesh 的法线
		FVector3f n1 = dp1 ^ d21;

		// 比较新旧法线点积结果, >= 0 同向, < 0 反向(被翻转)
		return (n0 | n1) < 0.0f;
	}

	return false;
}

float FMeshSimplifier::EvaluateMerge( const FVector3f& Position0, const FVector3f& Position1, bool bMoveVerts )
{
	// 相同位置的顶点 Collapse -> 零代价
	if( Position0 == Position1 )
		return 0.0f;

	WedgeDisjointSet.Reset();

	// Find unique adjacent triangles
	// 所有受影响的三角形 Mesh
	TArray< uint32, TInlineAllocator<16> > AdjTris;

	struct FWedgeVert
	{
		uint32 VertIndex;
		uint32 AdjTriIndex;
	};
	TArray< FWedgeVert, TInlineAllocator<16> > WedgeVerts[2];

	int32 VertDegree = 0;

	auto GatherAdjTris = [ this, &AdjTris, &WedgeVerts, &VertDegree ]( const FVector3f& Position, uint32 Index, uint32& FlagsUnion )
	{
		// 遍历所有引用该顶点的 Corner
		ForAllCorners( Position,
			[ this, &AdjTris, &WedgeVerts, &VertDegree, Index, &FlagsUnion ]( uint32 Corner )
			{
				VertDegree++;	// Corner 数量累计
				
				{
					uint8& RESTRICT CornerFlag = CornerFlags[ Corner ];
					FlagsUnion |= CornerFlag;		// 汇总所有 Corner 的 ECornerFlags 标记
					CornerFlag |= 1 << Index;		// 标记 Corner 属于 Position0 还是 Position1
				}
				
				uint32 TriIndex = Corner / 3;	// 该 Corner 属于哪一个三角形 Mesh
				uint32 AdjTriIndex;
				bool bNewTri = true;
				
				uint8& RESTRICT FirstCornerFlag = CornerFlags[ TriIndex * 3 ];
				// 第一次遇到这个三角形 Mesh, 加入 AdjTris 数组
				if( ( FirstCornerFlag & AdjTriMask ) == 0 )
				{
					FirstCornerFlag |= AdjTriMask;
					AdjTriIndex = AdjTris.Add( TriIndex );
					WedgeDisjointSet.AddDefaulted();
				}
				else
				{
					// Should only happen 2 times per collapse on average
					// 查找已存在于 AdjTris 数组中的三角形 Mesh
					AdjTriIndex = AdjTris.Find( TriIndex );
					bNewTri = false;
				}
		
				// 当前 Corner 的顶点索引 VertIndex
				uint32 VertIndex = Indexes[ Corner ];

				// WedgeVerts[0] 数组: 所有引用 Position0 的 Corner 信息
				// WedgeVerts[1] 数组: 所有引用 Position1 的 Corner 信息
				// WedgeVerts[0] 数组和 WedgeVerts[1] 数组中的每个元素: 记录每个 Corner 的顶点索引 VertIndex, 以及这个 Corner 属于 AdjTris 数组中的哪个三角形 Mesh
				uint32 OtherAdjTriIndex = ~0u;
				for( FWedgeVert& WedgeVert : WedgeVerts[ Index ] )
				{
					if( VertIndex == WedgeVert.VertIndex )
					{
						OtherAdjTriIndex = WedgeVert.AdjTriIndex;	// 有其它的三角形 Mesh 也在使用此顶点索引
						break;
					}
				}
				if( OtherAdjTriIndex == ~0u )
				{
					WedgeVerts[ Index ].Add( { VertIndex, AdjTriIndex } );
				}
				else
				{
					// 将所有受影响的三角形 Mesh 按照拓扑连通性 (是否有公用顶点) 分组成若干 Wedge, 每个 Wedge 代表一个拓扑连通的三角形 Mesh 的集合
					if( bNewTri )
						WedgeDisjointSet.UnionSequential( AdjTriIndex, OtherAdjTriIndex );
					else
						WedgeDisjointSet.Union( AdjTriIndex, OtherAdjTriIndex );
				}
			} );
	};

	uint32 FlagsUnion0 = 0;
	uint32 FlagsUnion1 = 0;

	// 收集此次 Collapse 会影响的所有三角形 Mesh
	GatherAdjTris( Position0, 0, FlagsUnion0 );
	GatherAdjTris( Position1, 1, FlagsUnion1 );

	// Position0 和 Position1 没有被任何 Corner 引用, Collapse 不会影响任何三角形 Mesh -> 零代价
	if( VertDegree == 0 )
	{
		return 0.0f;
	}

	// This would mean this collapse will remove all remaining triangles.
	// Collapse 会删光所有三角形 Mesh (每个三角形 Mesh 都同时连着 Position0 和 Position1, Collapse 后整个 Cluster 消失) -> 零代价
	if( VertDegree == RemainingNumTris * 2 )
	{
		// Clean up corner flags
		check( AdjTris.Num() > 0 );
		for( uint32 TriIndex : AdjTris )
		{
			// 清掉 Corner 上的 ECornerFlags 标记
			for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
			{
				CornerFlags[ TriIndex * 3 + CornerIndex ] &= ~( MergeMask | AdjTriMask );
			}
		}
		return 0.0f;
	}

	bool bLocked0 = FlagsUnion0 & LockedVertMask;
	bool bLocked1 = FlagsUnion1 & LockedVertMask;

	// 惩罚系统: 只在评估时 (bMoveVerts = false) 加成到最终代价 (Error) 上
	float Penalty = 0.0f;

	// VertDegree 过高 -> 惩罚, 避免产生超高 Valence 的点 (容易造成细长三角形 Mesh)
	if( VertDegree > DegreeLimit )
		Penalty += DegreePenalty * ( VertDegree - DegreeLimit );
	
	// 构建 WedgeQuadric
	TArray< uint32, TInlineAllocator<8> >	WedgeIDs;
	TArray< uint8, TInlineAllocator<1024> >	WedgeQuadrics;
	
	const SIZE_T QuadricSize = sizeof( FQuadricAttr ) + NumAttributes * 4 * sizeof( QScalar );

	auto GetWedgeQuadric =
		[ &WedgeQuadrics, QuadricSize ]( int32 WedgeIndex ) -> FQuadricAttr&
		{
			return *reinterpret_cast< FQuadricAttr* >( &WedgeQuadrics[ WedgeIndex * QuadricSize ] );
		};

	// 处于同一个 Wedge 的 TriQuadric 累加
	for( uint32 AdjTriIndex = 0, Num = AdjTris.Num(); AdjTriIndex < Num; AdjTriIndex++ )
	{
		uint32 TriIndex = AdjTris[ AdjTriIndex ];

		FQuadricAttr& RESTRICT TriQuadric = GetTriQuadric( TriIndex );

		uint32 WedgeID = WedgeDisjointSet.Find( AdjTriIndex );
		int32 WedgeIndex = WedgeIDs.Find( WedgeID );
		if( WedgeIndex != INDEX_NONE )
		{
			FQuadricAttr& RESTRICT WedgeQuadric = GetWedgeQuadric( WedgeIndex );

#if SIMP_REBASE
			// 做 Rebase, 把 Quadric 的原点移动到 Position0, 提高数值稳定性, 避免 float 精度爆炸
			uint32 VertIndex0 = Indexes[ TriIndex * 3 ];
			WedgeQuadric.Add( TriQuadric, GetPosition( VertIndex0 ) - Position0, GetAttributes( VertIndex0 ), AttributeWeights, NumAttributes );
#else
			WedgeQuadric.Add( TriQuadric, NumAttributes );
#endif
		}
		else
		{
			WedgeIndex = WedgeIDs.Add( WedgeID );
			WedgeQuadrics.AddUninitialized( QuadricSize );

			FQuadricAttr& RESTRICT WedgeQuadric = GetWedgeQuadric( WedgeIndex );

			FMemory::Memcpy( &WedgeQuadric, &TriQuadric, QuadricSize );
#if SIMP_REBASE
			uint32 VertIndex0 = Indexes[ TriIndex * 3 ];
			WedgeQuadric.Rebase( GetPosition( VertIndex0 ) - Position0, GetAttributes( VertIndex0 ), AttributeWeights, NumAttributes );
#endif
		}
	}

	FQuadricAttrOptimizer QuadricOptimizer;
	for( int32 WedgeIndex = 0, Num = WedgeIDs.Num(); WedgeIndex < Num; WedgeIndex++ )
	{
		QuadricOptimizer.AddQuadric( GetWedgeQuadric( WedgeIndex ), NumAttributes );
	}

	FVector3f	BoundsMin = {  MAX_flt,  MAX_flt,  MAX_flt };
	FVector3f	BoundsMax = { -MAX_flt, -MAX_flt, -MAX_flt };

	FQuadric EdgeQuadric;
	EdgeQuadric.Zero();

	// 遍历 AdjTris 数组中的所有三角形 Mesh 的所有 Corner, 构建邻域 BoundingBox 以及累加 EdgeQuadric
	for( uint32 TriIndex : AdjTris )
	{
		for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
		{
			uint32 Corner = TriIndex * 3 + CornerIndex;

			const FVector3f& Position = GetPosition( Indexes[ Corner ] );

			BoundsMin = FVector3f::Min( BoundsMin, Position );
			BoundsMax = FVector3f::Max( BoundsMax, Position );
			
			// 如果这个 Corner 对应的有向边有被约束, 则累加 EdgeQuadric
			if( EdgeQuadricsValid[ Corner ] )
			{
				// Only if edge is part of this pair
				uint32 EdgeFlags;
				EdgeFlags  = CornerFlags[ Corner ];
				EdgeFlags |= CornerFlags[ TriIndex * 3 + ( ( 1 << CornerIndex ) & 3 ) ];

				// 只对当前 Collapse 的边有效, 这里的 CornerFlags 是在 GatherAdjTris 函数中标记的 (标记 Corner 属于 Position0 还是 Position1)
				if( EdgeFlags & MergeMask )
				{
#if SIMP_REBASE
					EdgeQuadric.Add( EdgeQuadrics[ Corner ], GetPosition( Indexes[ Corner ] ) - Position0 );
#else
					uint32 VertIndex0 = Indexes[ Corner ];
					uint32 VertIndex1 = Indexes[ Cycle3( Corner ) ];
					//EdgeQuadric += FQuadric( GetPosition( VertIndex0 ), GetPosition( VertIndex1 ), GetNormal( TriIndex ), EdgeWeight );
					EdgeQuadric.Add( EdgeQuadrics[ Corner ], GetPosition( Indexes[ Corner ] ) - QuadricOrigin );
#endif
				}
			}
		}
	}

	QuadricOptimizer.AddQuadric( EdgeQuadric );
	
	// 用于 NewPosition 几何合法性 (不远离邻域 BoundingBox + 不翻转) 检查
	auto IsValidPosition =
		[ this, &AdjTris, &BoundsMin, &BoundsMax ]( const FVector3f& Position ) -> bool
		{
			// Limit position to be near the neighborhood bounds
			// 邻域 BoundingBox 约束: 用来限制 NewPosition 在靠近邻域 BoundingBox 的范围内, 防止简化器求出一个在数值上最小但跑到很远处的 "怪点"
			if( ComputeSquaredDistanceFromBoxToPoint( BoundsMin, BoundsMax, Position ) > ( BoundsMax - BoundsMin ).SizeSquared() * 4.0f )
				return false;

			// 翻转检测: Collapse 后不会导致三角形 Mesh 被翻转
			for( uint32 TriIndex : AdjTris )
			{
				if( TriWillInvert( TriIndex, Position ) )
					return false;
			}

			return true;
		};

	// 求 NewPosition
	FVector3f NewPosition;
	{
		// Position0 和 Position1 都被禁止移动 -> 惩罚, 但仍允许被 Collapse, 只是代价超级高 (1e8f)
		if( bLocked0 && bLocked1 )
			Penalty += LockPenalty;

		// find position
		// Position0 被禁止移动但 Position1 没有, 强制 NewPosition = Position0, 如果 NewPosition 不合法 -> 惩罚, 但仍允许被 Collapse, 只是代价很高 (100)
		if( bLocked0 && !bLocked1 )
		{
			NewPosition = Position0;

			if( !IsValidPosition( NewPosition ) )
				Penalty += InversionPenalty;
		}
		// Position1 被禁止移动但 Position0 没有, 强制 NewPosition = Position1, 如果 NewPosition 不合法 -> 惩罚, 但仍允许被 Collapse, 只是代价很高 (100)
		else if( bLocked1 && !bLocked0 )
		{
			NewPosition = Position1;

			if( !IsValidPosition( NewPosition ) )
				Penalty += InversionPenalty;
		}
		else	// Position0 和 Position1 都没有被禁止移动, 则按照优先级分别尝试 4 中方案:
		{
			// 1. 完整 3D Quadric 最有解
			bool bIsValid = QuadricOptimizer.OptimizeVolume( NewPosition );
#if SIMP_REBASE
			NewPosition += Position0;
#endif
			if( bIsValid )
				bIsValid = IsValidPosition( NewPosition );

			if( !bIsValid )
			{
				// 2. 降级优化, 退化解
				bIsValid = QuadricOptimizer.Optimize( NewPosition );
#if SIMP_REBASE
				NewPosition += Position0;
#endif
				if( bIsValid )
					bIsValid = IsValidPosition( NewPosition );
			}
			
			if( !bIsValid )
			{
				// Try a point on the edge.
				// 3. 限制在边上 (Position0 - Position1 边) 找最优解
#if SIMP_REBASE
				bIsValid = QuadricOptimizer.OptimizeLinear( FVector3f::ZeroVector, Position1 - Position0, NewPosition );
				NewPosition += Position0;
#else
				bIsValid = QuadricOptimizer.OptimizeLinear( Position0, Position1, NewPosition );
#endif
				if( bIsValid )
					bIsValid = IsValidPosition( NewPosition );
			}

			if( !bIsValid )
			{
				// Couldn't find optimal so choose middle
				// 4. 保底解: 取中点
				NewPosition = ( Position0 + Position1 ) * 0.5f;

				if( !IsValidPosition( NewPosition ) )
					Penalty += InversionPenalty;
			}
		}
	}

	int32 NumWedges = WedgeIDs.Num();
	WedgeAttributes.Reset();
	WedgeAttributes.AddUninitialized( NumWedges * NumAttributes );

	// Collapse 后的新顶点
#if SIMP_REBASE
	FVector3f NewPositionRebase = NewPosition - Position0;
#else
	FVector3f& NewPositionRebase = NewPosition;
#endif

	float Error = 0.0f;

	// 计算在新顶点上的 EdgeQuadric
	float EdgeError = EdgeQuadric.Evaluate( NewPositionRebase );
	float SurfaceArea = 0.0f;

	// 特殊情况: 如果 Position0 和 Position1 中有一个被禁止移动, 或者存在有属性权重为 0 的情况 (例如 Skinning 属性权重就为 0) 时, 直接拷贝旧属性到新属性
	if( bLocked0 != bLocked1 || bZeroWeights )
	{
		// 新顶点分别与 Position0 和 Position1 的距离平方
		const float DistSqr0 = FVector3f::DistSquared( NewPosition, Position0 );
		const float DistSqr1 = FVector3f::DistSquared( NewPosition, Position1 );
	
		// Start with farthest so that the second pass with closest will overwrite any verts from the same wedge.
		// Can't do only the closest since there may be wedges only present in the farthest.
		// 拷贝属性时, 根据距离, 先从距离更远的一端拷贝, 再从距离更近的一端拷贝, 这样保证远近两端都有的顶点属性最终保留的是近端的属性, 而只有远端有的顶点属性也不会遗漏掉
		uint32 Farthest = DistSqr0 > DistSqr1 ? 0 : 1;

		for( uint32 j = 0; j < 2; j++ )
		{
			// 遍历所有引用 Position0 和 Position1 的 Corner
			for( FWedgeVert& WedgeVert : WedgeVerts[ ( Farthest + j ) & 1 ] )
			{
				// 获取 WedgeIndex
				int32 WedgeIndex = WedgeIDs.Find( WedgeDisjointSet[ WedgeVert.AdjTriIndex ] );

				// 拷贝旧属性到新属性
				float* RESTRICT NewAttributes = &WedgeAttributes[ WedgeIndex * NumAttributes ];
				float* RESTRICT OldAttributes = GetAttributes( WedgeVert.VertIndex );

				for( uint32 i = 0; i < NumAttributes; i++ )
					NewAttributes[i] = OldAttributes[i];
			}
		}
	}

	for( int32 WedgeIndex = 0; WedgeIndex < NumWedges; WedgeIndex++ )
	{
		float* RESTRICT NewAttributes = &WedgeAttributes[ WedgeIndex * NumAttributes ];

		FQuadricAttr& RESTRICT WedgeQuadric = GetWedgeQuadric( WedgeIndex );
		if( WedgeQuadric.a > 1e-8 )		// Wedge 面积有效
		{
			float WedgeError;
			// 特殊情况: 有一端被禁止移动, 在前面已经计算过新属性 (拷贝旧属性) , 所以这里只根据新属性计算代价
			if( bLocked0 != bLocked1 )
			{
				WedgeError = WedgeQuadric.Evaluate( NewPositionRebase, NewAttributes, AttributeWeights, NumAttributes );
			}
			else
			{
				// calculate vert attributes from the new position
				// 正常情况: 根据 NewPosition 计算新属性, 并计算代价
				WedgeError = WedgeQuadric.CalcAttributesAndEvaluate( NewPositionRebase, NewAttributes, AttributeWeights, NumAttributes );
				
				// Correct after eval. Normal length is unimportant for error but can bias the calculation.
				// 纠正新属性: 归一化法线, 归一化切线, Clamp 顶点颜色, 避免影响后续光照计算
				if( CorrectAttributes != nullptr )
					CorrectAttributes( NewAttributes );
			}

			if( bLimitErrorToSurfaceArea )
				WedgeError = FMath::Min( WedgeError, WedgeQuadric.a );

			Error += WedgeError;
		}
		else
		{
			// Wedge 面积太小则认为无效, 新属性置 0, 不需要计算代价
			for( uint32 i = 0; i < NumAttributes; i++ )
			{
				NewAttributes[i] = 0.0f;
			}
		}
		
		SurfaceArea += WedgeQuadric.a;
	}

	Error += EdgeError;
	
	// Position0 和 Position1 在一个相对独立的表面, 分两种情形:
	// 1. AdjTris.Num() == 1 -> Position0 和 Position1 是一个独立三角形 Mesh 的一条有向边的两个端点
	// 2. ( AdjTris.Num() == 2 && VertDegree == 4 ) -> Position0 和 Position1 是两个独立三角形 Mesh 的公共边的两个端点
	bool bIsDisjoint = AdjTris.Num() == 1 || ( AdjTris.Num() == 2 && VertDegree == 4 );

	if( bLimitErrorToSurfaceArea )
	{
		// Limit error to be no greater than the size of the triangles it could affect.
		// 把代价 clamp 到所影响的 Wedge 的表面积大小
		Error = FMath::Min( Error, SurfaceArea );
		
		// Collapsing will completely remove this surface area. The position merged to is irrelevant.
		// 如果 Position0 和 Position1 在一个相对独立的表面, 那么 Collapse 会完全删除这个表面, 此时 NewPosition 在哪里并不重要, 直接用面积来表示代价更合理
		if( bIsDisjoint )
			Error = SurfaceArea;
	}

	// Check to set error based on edge length
	// 是否把 NewPosition 生成的新的边的长度也考虑进代价的计算
	if( MaxEdgeLengthFactor > 0.0f )
	{
		for( uint32 TriIndex : AdjTris )
		{
			uint32 IndexMoved = CornerIndexMoved( TriIndex );

			if( IndexMoved < 3 )
			{
				uint32 Corner = TriIndex * 3 + IndexMoved;

				const FVector3f& p1 = GetPosition( Indexes[ Cycle3( Corner ) ] );
				const FVector3f& p2 = GetPosition( Indexes[ Cycle3( Corner, 2 ) ] );

				Error = FMath::Max( Error, ( NewPosition - p1 ).SizeSquared() / ( MaxEdgeLengthFactor * MaxEdgeLengthFactor ) );
				Error = FMath::Max( Error, ( NewPosition - p2 ).SizeSquared() / ( MaxEdgeLengthFactor * MaxEdgeLengthFactor ) );
			}
		}
	}

	// 真正执行 Collapse (bMoveVerts = true)
	if( bMoveVerts )
	{
		// 把 VertHash, CornerHash, PairHash 中涉及 Position0 和 Position1 的元素删除, 并把受影响的 VertexIndex/Corner/PairIndex 添加进 MovedVerts/MovedCorners/MovePairs 中
		BeginMovePosition( Position0 );
		BeginMovePosition( Position1 );

		// 更新 AdjTris 中所有 Corner 的信息
		for( uint32 AdjTriIndex = 0, Num = AdjTris.Num(); AdjTriIndex < Num; AdjTriIndex++ )
		{
			int32 WedgeIndex = WedgeIDs.Find( WedgeDisjointSet[ AdjTriIndex ] );

			for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
			{
				uint32 Corner = AdjTris[ AdjTriIndex ] * 3 + CornerIndex;
				uint32 VertIndex = Indexes[ Corner ];

				FVector3f& OldPosition = GetPosition( VertIndex );
				if( OldPosition == Position0 ||
					OldPosition == Position1 )
				{
					// 如果 Corner 中记录的顶点位置等于 Position0 或 Position1, 将 Corner 中记录的顶点位置改为 NewPosition
					OldPosition = NewPosition;

					// Only use attributes if we calculated them.
					// 如果 Corner 所在的三角形 Mesh 所属的 Wedge 的面积有效, 顶点属性更新为新属性
					if( GetWedgeQuadric( WedgeIndex ).a > 1e-8 )
					{
						float* RESTRICT NewAttributes = &WedgeAttributes[ WedgeIndex * NumAttributes ];
						float* RESTRICT OldAttributes = GetAttributes( VertIndex );

						for( uint32 i = 0; i < NumAttributes; i++ )
						{
							OldAttributes[i] = NewAttributes[i];
						}
					}
					
					// If either position was locked then lock the new verts.
					// 如果 Position0 或 Position1 中有一个被禁止移动, 则 Corner 也禁止移动
					if( bLocked0 || bLocked1 )
						CornerFlags[ Corner ] |= LockedVertMask;
				}
			}
		}

		// 更新 MovedPairs 中 Pair 的 Position0 和 Position1
		for( uint32 PairIndex : MovedPairs )
		{
			FPair& Pair = Pairs[ PairIndex ];

			checkSlow( Pair.Position0 != Position0 || Pair.Position1 != Position1 );

			if( Pair.Position0 == Position0 ||
				Pair.Position0 == Position1 )
			{
				Pair.Position0 = NewPosition;
			}

			if( Pair.Position1 == Position0 ||
				Pair.Position1 == Position1 )
			{
				Pair.Position1 = NewPosition;
			}
		}

		// EndMovePositions 函数中:
		// 1. 把 MovedVerts 和 MovedCorners 重新插入 VertHash 和 CornerHash 中
		// 2. 根据 MovedPairs 对最小堆 PairHeap 做去重和无效剔除
		EndMovePositions();

		// 从受影响的 AdjTris 数组中收集所有顶点索引, 去重后添加进 AdjVerts 数组中
		TArray< uint32, TInlineAllocator<16> > AdjVerts;
		for( uint32 TriIndex : AdjTris )
		{
			for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
			{
				AdjVerts.AddUnique( Indexes[ TriIndex * 3 + CornerIndex ] );
			}
		}

		// Reevaluate all pairs touching an adjacent tri.
		// Duplicate pairs have already been removed.
		// AdjVerts 数组中收集的顶点相关的 FPair 如果还在 PairHeap 中, 则从 PairHeap 中将其移除并添加进 ReevaluatePairs 中
		// 在上层 FMeshSimplifier::Simplify 函数中会对 ReevaluatePairs 中的 Pair 进行重新评估代价 EvaluateMerge(..., false) 并添加进 PairHeap 中
		for( uint32 VertIndex : AdjVerts )
		{
			const FVector3f& Position = GetPosition( VertIndex );

			ForAllPairs( Position,
				[ this ]( uint32 PairIndex )
				{
					// IsPresent used to mark Pairs we have already added to the list.
					if( PairHeap.IsPresent( PairIndex ) )
					{
						PairHeap.Remove( PairIndex );
						ReevaluatePairs.Add( PairIndex );
					}
				} );
		}

		// 以材质 ID 为 Key 统计 Collapse 后减少的几何表面积, 减少的三角形 Mesh 数量和减少的相对独立的表面数量
		for( uint32 TriIndex : AdjTris )
		{
			// 三角形 Mesh 的材质 ID
			int32 MaterialIndex = MaterialIndexes[ TriIndex ] & 0xffffff;
			if( !PerMaterialDeltas.IsValidIndex( MaterialIndex ) )
				PerMaterialDeltas.SetNumZeroed( MaterialIndex + 1 );

			// 以材质 ID 为 Key 统计
			auto& Delta = PerMaterialDeltas[ MaterialIndex ];

			// 减少三角形 Mesh 面积
			Delta.SurfaceArea -= GetTriQuadric( TriIndex ).a;
			// 减少三角形 Mesh 数量
			Delta.NumTris--;
			// 如果 Position0 和 Position1 处于相对独立的表面, 则 Delta.NumDisjoint -= 1
			Delta.NumDisjoint -= bIsDisjoint ? 1 : 0;

			// 删除退化三角形 Mesh, 删除重复三角形 Mesh, 重新计算 TriQuadric
			FixUpTri( TriIndex );
			
			// 如果三角形 Mesh 在 Collapse 后仍在存在, 则添加回三角形 Mesh 面积和数量
			if( !TriRemoved[ TriIndex ] )
			{
				Delta.SurfaceArea += GetTriQuadric( TriIndex ).a;
				Delta.NumTris++;
			}
		}
	}
	else
	{
		// 如果只是评估, 则代价额外加上惩罚
		Error += Penalty;
	}

	// 统一清理 Corner 上的 ECornerFlags 标记
	for( uint32 TriIndex : AdjTris )
	{
		for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
		{
			uint32 Corner = TriIndex * 3 + CornerIndex;

			// Must be separated from FixUpTri loop because relies on correct indexing
			// 如果是真正执行 Collapse, 还需要在清 ECornerFlags 标记前更新 EdgeQuadric, 这是因为 Collapse 后边界状态可能改变, 所以必须重新计算 EdgeQuadric
			if( bMoveVerts )
				CalcEdgeQuadric( Corner );

			// Clear flags
			CornerFlags[ Corner ] &= ~( MergeMask | AdjTriMask );
		}
	}

	return Error;
}
```

**总结**：

**评估**或**执行（ `bMoveVerts = true` ）**把两个顶点（ `Position0` 和 `Position1` ）合并成一个顶点的**代价（Error）**

##### 4.1.1.7. 材质面积丢失补偿 `FMeshSimplifier::PreserveSurfaceArea`

```cpp
void FMeshSimplifier::PreserveSurfaceArea()
{
	// 选出表面积损失最大的材质 ID
	int32 DilateMaterialIndex = INDEX_NONE;
	float DilateSurfaceArea = 0.0f;
	for( int32 MaterialIndex = 0; MaterialIndex < PerMaterialDeltas.Num(); MaterialIndex++ )
	{
		if( PerMaterialDeltas[ MaterialIndex ].SurfaceArea < DilateSurfaceArea )
		{
			DilateMaterialIndex = MaterialIndex;
			DilateSurfaceArea = PerMaterialDeltas[ MaterialIndex ].SurfaceArea;
		}
	}

	if( DilateMaterialIndex == INDEX_NONE )
		return;

	TArray< FVector4f > EdgeNormals;
	EdgeNormals.AddZeroed( NumVerts );

	float Perimeter = 0.0f;		// 表面积损失最大的材质 ID 对应几何区域的边界边长度总和
	float TotalArea = 0.0f;		// 完整 Mesh 的面积
	float ThisArea = 0.0f;		// 表面积损失最大的材质 ID 对应几何区域的面积

	uint32 NumEdges = 0;
	uint32 NumFaces = 0;

	// 遍历所有三角形 Mesh
	for( uint32 TriIndex = 0; TriIndex < NumTris; TriIndex++ )
	{
		// 跳过被移除的三角形 Mesh
		if( TriRemoved[ TriIndex ] )
			continue;

		// 累加三角形 Mesh 的面积
		float SurfaceArea = GetTriQuadric( TriIndex ).a;
		TotalArea += SurfaceArea;

		// 跳过材质 ID 不是表面积损失最大的材质 ID 的三角形 Mesh
		if( ( MaterialIndexes[ TriIndex ] & 0xffffff ) != DilateMaterialIndex )
			continue;

		ThisArea += SurfaceArea;
		NumFaces++;

		// 遍历三角形 Mesh 的三条有向边
		for( uint32 CornerIndex = 0; CornerIndex < 3; CornerIndex++ )
		{
			uint32 EdgeIndex = TriIndex * 3 + CornerIndex;
			
			if( EdgeQuadricsValid[ EdgeIndex ] )
			{
				uint32 VertIndex0 = Indexes[ EdgeIndex ];
				uint32 VertIndex1 = Indexes[ Cycle3( EdgeIndex ) ];

				const FVector3f& Position0 = GetPosition( VertIndex0 );
				const FVector3f& Position1 = GetPosition( VertIndex1 );

				// Find edge with opposite direction that shares these 2 verts.
				/*
					  /\
					 /  \
					o-<<-o
					o->>-o
					 \  /
					  \/
				*/
				// 找对向共享边
				uint32 Hash = HashPosition( Position1 );
				uint32 Corner;
				for( Corner = CornerHash.First( Hash ); CornerHash.IsValid( Corner ); Corner = CornerHash.Next( Corner ) )
				{
					uint32 OtherVertIndex0 = Indexes[ Corner ];
					uint32 OtherVertIndex1 = Indexes[ Cycle3( Corner ) ];
			
					if( Position0 == GetPosition( OtherVertIndex1 ) &&
						Position1 == GetPosition( OtherVertIndex0 ) )
					{
						// Found matching edge.
						break;
					}
				}

				// 如果没有找到对向共享边, 则说明此有向边是边界边, 外扩就是应该沿着边界边的方向
				if( !CornerHash.IsValid( Corner ) )
				{
					NumEdges++;

					// 三角形 Mesh 的三个顶点
					const FVector3f& p0 = GetPosition( Indexes[ TriIndex * 3 + 0 ] );
					const FVector3f& p1 = GetPosition( Indexes[ TriIndex * 3 + 1 ] );
					const FVector3f& p2 = GetPosition( Indexes[ TriIndex * 3 + 2 ] );

					// 将三角形 Mesh 的表面法线投影到边界边的方向上
					FVector3f Edge = Position1 - Position0;
					FVector3f FaceNormal = ( p2 - p0 ) ^ ( p1 - p0 );
					FVector3f EdgeNormal = FaceNormal ^ Edge;
					EdgeNormal.Normalize();

					float EdgeLength = Edge.Length();
					Perimeter  += EdgeLength;
					EdgeNormal *= EdgeLength;

					auto AddEdgeNormal = [&]( uint32 VertIndex )
					{
						EdgeNormals[ VertIndex ] += FVector4f( EdgeNormal, EdgeLength );
					};

					// 忽略被禁止移动的顶点
					bool bLocked0 = CornerFlags[ EdgeIndex ]			& LockedVertMask;
					bool bLocked1 = CornerFlags[ Cycle3( EdgeIndex ) ]	& LockedVertMask;

					// 累加所有引用 Position0 和 Position1 的 VertexIndex 的 EdgeNormal
					if( !bLocked0 ) ForAllVerts( Position0, AddEdgeNormal );
					if( !bLocked1 ) ForAllVerts( Position1, AddEdgeNormal );
				}
			}
		}
	}

	// If the scale is too big don't do it. Losing some area is better than suddenly getting a few gigantic leaves.
	// 如果需要补回的面积超过表面积损失最大的材质 ID 对应几何区域的面积的 4 倍, 就不做外扩, 这种情况下失去一些面积比外扩导致的爆炸式扩张要好
	if( -DilateSurfaceArea > 4.0f * ThisArea )
		return;

	// Does not accurately factor in corners.
	float DilateDistance = -DilateSurfaceArea / Perimeter;

	FRandomStream Random( NumEdges );

	for( uint32 VertIndex = 0; VertIndex < NumVerts; VertIndex++ )
	{
		FVector3f EdgeNormal = EdgeNormals[ VertIndex ];
		float Weight = EdgeNormals[ VertIndex ].W;
		if( Weight > 1e-6f )
		{
			//Scale * Perimeter / NumEdges = Weight * 0.5f;
			//float Scale = 2.0f * Perimeter / ( NumEdges * Weight );
			float Scale = Random.GetFraction() * 0.5f + 0.75f;	// 轻微随机缩放
			EdgeNormal /= Weight;
			float LengthSqr = EdgeNormal.SizeSquared();
			if( LengthSqr > 0.1f )
			{
				GetPosition( VertIndex ) += EdgeNormal * ( Scale * DilateDistance / LengthSqr );
			}
		}
	}
}
```

**总结**：

找到面积损失最多的材质，把这个材质对应的几何的边界做一次外扩，用近似方式补回损失的表面积

##### 4.1.1.8. 重建简化后的 Cluster 网格数据

- 重建 `Verts` 、 `Indexes` 和 `MaterialIndexes`
- 根据前面保存的 `LockedEdges` 重建简化后的 Cluster 的 `ExternalEdges`
- 还原模型尺度，更新 Bounds
- 清除 `MaterialIndexes` 高 8 位中存储的 UV 镜像标记

#### 4.1.2. 切分父 Cluster `SplitCluster`

```cpp
void FCluster::Split( FGraphPartitioner& Partitioner, const FAdjacency& Adjacency ) const
{
	// 根据 Cluster 内所有有向边的对向共享边信息, 使用并查集将 Cluster 内的三角形 Mesh 划分为不同的连通块, 构建三角形 Mesh 之间的拓扑邻接关系
	FDisjointSet DisjointSet( NumTris );
	for( int32 EdgeIndex = 0; EdgeIndex < Indexes.Num(); EdgeIndex++ )
	{
		Adjacency.ForAll( EdgeIndex,
			[ &DisjointSet ]( int32 EdgeIndex0, int32 EdgeIndex1 )
			{
				if( EdgeIndex0 > EdgeIndex1 )
					DisjointSet.UnionSequential( EdgeIndex0 / 3, EdgeIndex1 / 3 );
			} );
	}

	auto GetCenter = [ this ]( uint32 TriIndex )
	{
		FVector3f Center;
		Center  = GetPosition( Indexes[ TriIndex * 3 + 0 ] );
		Center += GetPosition( Indexes[ TriIndex * 3 + 1 ] );
		Center += GetPosition( Indexes[ TriIndex * 3 + 2 ] );
		return Center * (1.0f / 3.0f);
	};
	// 构建三角形 Mesh 之间的空间邻近关系
	Partitioner.BuildLocalityLinks( DisjointSet, Bounds, MaterialIndexes, GetCenter );

	// 使用 METIS 库递归二分划分子 Partition
	auto* RESTRICT Graph = Partitioner.NewGraph( NumTris * 3 );

	for( uint32 i = 0; i < NumTris; i++ )
	{
		Graph->AdjacencyOffset[i] = Graph->Adjacency.Num();

		uint32 TriIndex = Partitioner.Indexes[i];

		// Add shared edges
		// 拓扑邻接的三角形 Mesh 之间设置一个大权重
		for( int k = 0; k < 3; k++ )
		{
			Adjacency.ForAll( 3 * TriIndex + k,
				[ &Partitioner, Graph ]( int32 EdgeIndex, int32 AdjIndex )
				{
					Partitioner.AddAdjacency( Graph, AdjIndex / 3, 4 * 65 );
				} );
		}

		// 空间上邻近的三角形 Mesh 之间设置一个小权重
		Partitioner.AddLocalityLinks( Graph, TriIndex, 1 );
	}
	Graph->AdjacencyOffset[ NumTris ] = Graph->Adjacency.Num();

	Partitioner.PartitionStrict( Graph, false );
}

template< typename FPartitioner, typename FPartitionFunc >
bool SplitCluster( FCluster& Merged, TArray< FCluster >& Clusters, TAtomic< uint32 >& NumClusters, uint32 MaxClusterSize, uint32& NumParents, uint32& ParentStart, uint32& ParentEnd, FPartitionFunc&& PartitionFunc )
{
	// 如果简化后的 Cluster 中三角形 Mesh 的材质数量 <= 每个 Cluster 允许的最大三角形 Mesh 数量, 那么简化后的 Cluster 直接当父 Cluster
	if( Merged.MaterialIndexes.Num() <= (int32)MaxClusterSize )
	{
		ParentEnd = ( NumClusters += 1 );
		ParentStart = ParentEnd - 1;

		Clusters[ ParentStart ] = Merged;
		Clusters[ ParentStart ].Bound();
		return true;
	}
	else if( NumParents > 1 )   // 如果需要多个父 Cluster
	{
		check( MaxClusterSize == FCluster::ClusterSize );

		// 构建简化后的 Cluster 中每条有向边的对向共享边信息
		FAdjacency Adjacency = Merged.BuildAdjacency();

		// PartitionFunc 最终调用 FCluster::Split 函数, 将简化后的 Cluster 递归二分划分为若干个子 Partition, 每个子 Partition 也就是一个父 Cluster
		FPartitioner Partitioner( Merged.MaterialIndexes.Num(), MaxClusterSize - 4, MaxClusterSize );
		PartitionFunc( Partitioner, Adjacency );

		if( Partitioner.Ranges.Num() <= (int32)NumParents )
		{
			// 实际生成的父 Cluster 数量
			NumParents = Partitioner.Ranges.Num();

			// 生成的父 Cluster 在 FClusterDAG::Clusters 数组中的起始 Offset 和结束 Offset
			ParentEnd = ( NumClusters += NumParents );
			ParentStart = ParentEnd - NumParents;

			// 创建父 Cluster
			int32 Parent = ParentStart;
			for( auto& Range : Partitioner.Ranges )
			{
				Clusters[ Parent ] = FCluster( Merged, Range.Begin, Range.End, Partitioner.Indexes, Partitioner.SortedTo, Adjacency );
				Parent++;
			}

			return true;
		}
	}

	return false;
}
```

## 5. 基于 `Settings.KeepPercentTriangles` 或 `Settings.TrimRelativeError` 做裁剪

```cpp
FBinaryHeap< float > FClusterDAG::FindCut(
	uint32 TargetNumTris,
	float  TargetError,
	uint32 TargetOvershoot,
	TBitArray<>* SelectedGroupsMask ) const
{
	// 从根 Cluster 开始
	const FClusterGroup&	RootGroup = Groups.Last();
	const FCluster&			RootCluster = Clusters[ RootGroup.Children[0] ];

	bool bHitTargetBefore = false;

	// Cluster DAG 的 LODError 由父到子层级是单调递减的, 这里用 MinError 记录最近一次 Pop 的最小误差
	float MinError = RootCluster.LODError;

	// 用来标记哪些 Group 被保留,初始全 false
	TBitArray<> VisitedGroups;
	VisitedGroups.Init(false, Groups.Num());

	// 根 Group 一定会用, 标记为 true
	VisitedGroups[Groups.Num() - 1] = true;
	
	// Heap 初始只包含根 Cluster
	FBinaryHeap< float > Heap;
	Heap.Add( -RootCluster.LODError, RootGroup.Children[0] );

	uint32 CurNumTris = RootCluster.NumTris;

	while( true )
	{
		// Grab highest error cluster to replace to reduce cut error
		const uint32 ClusterIndex = Heap.Top();
		const FCluster& Cluster = Clusters[ ClusterIndex ];
		const FClusterGroup& Group = Groups[ Cluster.GroupIndex ];
		const uint32 NumInstances = Group.AssemblyPartIndex == MAX_uint32 ? 1u : AssemblyPartData[ Group.AssemblyPartIndex ].NumTransforms;

		// 已到叶子层, 跳出循环
		if( Cluster.MipLevel == 0 )
			break;

		// 没有 Children 层, 跳出循环
		if( Cluster.GeneratingGroupIndex == MAX_uint32 )
			break;

		// 是否达到裁剪目标: 近似估算当前 cut 的三角形 Mesh 数量达到目标数量或者最小误差已经小于目标误差
		bool bHitTarget = CurNumTris > TargetNumTris || MinError < TargetError;

		// Overshoot the target by TargetOvershoot number of triangles. This allows granular edge collapses to better minimize error to the target.
		// Overshoot 机制: 第一次达到裁剪目标 (例如刚达到目标三角形 Mesh 数量), 允许再多展开一层, 使得颗粒状边缘的 Collapse 能够更好的最小化到目标误差
		if( TargetOvershoot > 0 && bHitTarget && !bHitTargetBefore )
		{
			TargetNumTris = CurNumTris + TargetOvershoot;
			bHitTarget = false;
			bHitTargetBefore = true;
		}

		if( bHitTarget && Cluster.LODError < MinError )
			break;
		
		// 把该 Cluster 从 cut 中移除, 用它的 Children 替换进 cut
		Heap.Pop();
		CurNumTris -= Cluster.NumTris * NumInstances;

		check( Cluster.LODError <= MinError );
		MinError = Cluster.LODError;

		// 标记该 Cluster 所在的 Group 不会被裁剪掉
		if (VisitedGroups[Cluster.GeneratingGroupIndex])
		{
			continue;
		}
		VisitedGroups[Cluster.GeneratingGroupIndex] = true;

		const FClusterGroup& NextGroup = Groups[ Cluster.GeneratingGroupIndex ];
		const uint32 NextNumInstances = NextGroup.AssemblyPartIndex == MAX_uint32 ? 1u : AssemblyPartData[ NextGroup.AssemblyPartIndex ].NumTransforms;

		// 把 Children 层加入 cut
		for( uint32 Child : NextGroup.Children )
		{
			if( !Heap.IsPresent( Child ) )
			{
				const FCluster& ChildCluster = Clusters[ Child ];

				check( ChildCluster.MipLevel < Cluster.MipLevel );
				check( ChildCluster.LODError <= MinError );
				Heap.Add( -ChildCluster.LODError, Child );
				CurNumTris += ChildCluster.NumTris * NextNumInstances;
			}
		}

		// TODO: Nanite-Assemblies: Double-check this. I think we have to handle the case where we cross the threshold from the mip tail
		// into the lower mips of assembly parts. I believe it's possible otherwise to get into a situation where some mip tail clusters
		// that were generated by assembly parts are still present on the heap and now overlap with an instanced, higher LOD of the part.
		// Maybe this can be solved simply by detecting when we're crossing that threshold here and removing all clusters from the heap
		// whose generating group == NextGroup like this? Not sure if it covers all cases though.
		if (Group.AssemblyPartIndex == MAX_uint32 && NextGroup.AssemblyPartIndex != MAX_uint32)
		{
			for (int32 OtherGroupIndex = 0; OtherGroupIndex < Groups.Num(); ++OtherGroupIndex)
			{
				const FClusterGroup& OtherGroup = Groups[OtherGroupIndex];
				if (OtherGroup.MipLevel < Group.MipLevel)
				{
					// Skip over higher mip groups
					continue;
				}
				
				for (uint32 OtherClusterIndex : OtherGroup.Children)
				{
					const FCluster& OtherCluster = Clusters[OtherClusterIndex];
					if (Heap.IsPresent(OtherClusterIndex) &&
						OtherCluster.GeneratingGroupIndex == Cluster.GeneratingGroupIndex)
					{
						Heap.Remove(OtherClusterIndex);
						CurNumTris -= OtherCluster.NumTris;
					}
				}
			}
		}
	}

	if (SelectedGroupsMask)
	{
		*SelectedGroupsMask = MoveTemp(VisitedGroups);
	}

	return Heap;
}

if( Settings.KeepPercentTriangles < 1.0f || Settings.TrimRelativeError > 0.0f )
{
	int32 TargetNumTris = int32((float)Resources.NumInputTriangles * Settings.KeepPercentTriangles);
	float TargetError = Settings.TrimRelativeError * 0.01f * FMath::Sqrt( FMath::Min( 2.0f * Resources.SurfaceArea, InputMeshData.VertexBounds.GetSurfaceArea() ) );

	TBitArray<> SelectedGroupsMask;
	FBinaryHeap< float > Heap = Resources.ClusterDAG.FindCut( TargetNumTris, TargetError, 0, &SelectedGroupsMask );

	for( int32 GroupIndex = 0; GroupIndex < SelectedGroupsMask.Num(); GroupIndex++ )
	{
		Resources.ClusterDAG.Groups[ GroupIndex ].bTrimmed = !SelectedGroupsMask[ GroupIndex ];
	}

	uint32 NumVerts = 0;
	uint32 NumTris = 0;
	for( uint32 i = 0; i < Heap.Num(); i++ )
	{
		FCluster& Cluster = Resources.ClusterDAG.Clusters[ Heap.Peek(i) ];

		Cluster.GeneratingGroupIndex = MAX_uint32;
		Cluster.EdgeLength = -FMath::Abs( Cluster.EdgeLength );
		NumVerts += Cluster.NumVerts;
		NumTris  += Cluster.NumTris;
	}

	Resources.NumInputVertices	= FMath::Min( NumVerts, Resources.NumInputVertices );
	Resources.NumInputTriangles	= NumTris;

	UE_LOG( LogStaticMesh, Log, TEXT("Trimmed to %u tris"), NumTris );
}
```

**总结**：

根据**目标三角形 Mesh 数量**或**目标误差（ LODError ）**, 在 ClusterDAG 中寻找一个**切割面（ Cut ）**，并据此对网格进行**裁剪（ Trim ）**

## 6. 编码数据并写入 Page

```cpp
void Encode(
	FResources& Resources,
	FClusterDAG& ClusterDAG,
	const FMeshNaniteSettings& Settings,
	uint32 NumMeshes,
	uint32* OutTotalGPUSize
)
{
	{
		// TODO: Nanite-Assemblies - Remove shear here by making matrices orthogonal?
		const int32 NumTransforms = ClusterDAG.AssemblyTransforms.Num();
		if (NumTransforms > 0)
		{
			check(NumTransforms <= NANITE_MAX_ASSEMBLY_TRANSFORMS); // should have been handled already
			Resources.AssemblyTransforms.SetNumUninitialized(NumTransforms);
			TransposeTransforms(Resources.AssemblyTransforms.GetData(), ClusterDAG.AssemblyTransforms.GetData(), NumTransforms);
		}
	}

	// 顶点数据清洗
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::SanitizeVertexData);
		for (FCluster& Cluster : ClusterDAG.Clusters)
		{
			Cluster.SanitizeVertexData();
		}
	}

	// 删除退化三角形 Mesh
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::RemoveDegenerateTriangles);	// TODO: is this still necessary?
		RemoveDegenerateTriangles( ClusterDAG.Clusters );
	}

	// 构建材质 Range, 并重排序三角形 Mesh 顺序
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::BuildMaterialRanges);
		BuildMaterialRanges( ClusterDAG.Clusters );
	}

	// 对 Cluster 添加硬约束
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::ConstrainClusters);
		ConstrainClusters( ClusterDAG.Groups, ClusterDAG.Clusters );
	}

	// 检查所有三角形 Mesh 的 3 个 Index 是否都满足: MaxIndex - Index < 32
#if DO_CHECK
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::VerifyClusterConstraints);
		VerifyClusterContraints( ClusterDAG.Clusters );
	}
#endif

	// 对每个 MaterialRange, 把它内部的三角形 Mesh 再划分成多个 batch
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::BuildVertReuseBatches);
		BuildVertReuseBatches(ClusterDAG.Clusters);
	}

	// 顶点位置量化, 决定每个 Cluster 的 Position 怎么变成整数并写入 bitstream
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::CalculateQuantizedPositions);
		Resources.PositionPrecision = CalculateQuantizedPositionsUniformGrid( ClusterDAG.Clusters, Settings );	// Needs to happen after clusters have been constrained and split.
	}

	
	int32 BoneWeightPrecision;
	{
		// Select appropriate Auto precision for Normals and Tangents
		// Just use hard-coded defaults for now.
		Resources.NormalPrecision = (Settings.NormalPrecision < 0) ? 8 : FMath::Clamp(Settings.NormalPrecision, 0, NANITE_MAX_NORMAL_QUANTIZATION_BITS);

		if (ClusterDAG.bHasTangents)
		{
			Resources.TangentPrecision = (Settings.TangentPrecision < 0) ? 7 : FMath::Clamp(Settings.TangentPrecision, 0, NANITE_MAX_TANGENT_QUANTIZATION_BITS);
		}
		else
		{
			Resources.TangentPrecision = 0;
		}

		BoneWeightPrecision = (Settings.BoneWeightPrecision < 0) ? 8u : (int32)FMath::Clamp(Settings.BoneWeightPrecision, 0, NANITE_MAX_BLEND_WEIGHT_BITS);
	}

	if (ClusterDAG.bHasSkinning)
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::QuantizeBoneWeights);
		QuantizeBoneWeights(ClusterDAG.Clusters, BoneWeightPrecision);
	}

	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::PrintMaterialRangeStats);
		PrintMaterialRangeStats( ClusterDAG.Clusters );
	}

	TArray<FPage> Pages;
	TArray<FClusterGroupPart> GroupParts;
	TArray<FClusterGroupPartInstance> GroupPartInstances;
	TArray<FEncodingInfo> EncodingInfos;

	// 计算每个 Cluster 的编码信息
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::CalculateEncodingInfos);
		CalculateEncodingInfos(EncodingInfos, ClusterDAG.Clusters, Resources.NormalPrecision, Resources.TangentPrecision, BoneWeightPrecision);
	}

	// 把 Cluster 按 Group 顺序打包进 GPU Page, 同时生成 GroupPart 和实例数据
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::AssignClustersToPages);
		const uint32 MaxRootPages = CalculateMaxRootPages(Settings.TargetMinimumResidencyInKB);
		AssignClustersToPages(ClusterDAG, EncodingInfos, Pages, GroupParts, GroupPartInstances, MaxRootPages, Resources.MeshBounds);
		Resources.NumRootPages = FMath::Min((uint32)Pages.Num(), MaxRootPages);
	}

	// 构建层次结构 BVH
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::BuildHierarchyNodes);
		BuildHierarchies(Resources, Pages, ClusterDAG.Groups, GroupParts, GroupPartInstances, ClusterDAG.AssemblyTransforms, NumMeshes);
	}

	// 写入 Page 数据
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::WritePages);
		WritePages(Resources, Pages, ClusterDAG.Groups, GroupParts, GroupPartInstances, ClusterDAG.Clusters, EncodingInfos, ClusterDAG.bHasSkinning, OutTotalGPUSize);
	}
}
```

在 Nanite 中， Page 分为两类：

1. **Root Page** ：始终驻留在 GPU ，保证最小可见行（始终有东西可以绘制）
2. **Streaming Page** ：按需流式加载

**不同种类的 Page 限制的大小也不同**，其中 Root Page 限制的大小是 `NANITE_ROOT_PAGE_GPU_SIZE = (1u << 15)` 字节（Bytes），而 Streaming Page 限制的大小是 `NANITE_STREAMING_PAGE_GPU_SIZE = (1u << 17)` 字节（Bytes）

**Root Page 数量也有上限**，最多允许多少个 Root Page 是根据 `Settings.TargetMinimumResidencyInKB` 计算（先转换为 Bytes ，再按 `NANITE_ROOT_PAGE_GPU_SIZE` 对齐计算）而来

下面摘取 `Encode` 函数中部分代码进行分析说明，帮助理解

### 6.1. 顶点数据清洗（ Sanitize ）

```cpp
static void SanitizeFloat( float& X, float MinValue, float MaxValue, float DefaultValue )
{
	if( X >= MinValue && X <= MaxValue )
		;
	else if( X < MinValue )
		X = MinValue;
	else if( X > MaxValue )
		X = MaxValue;
	else
		X = DefaultValue;
}

static void SanitizeVector( FVector3f& V, float MaxValue, FVector3f DefaultValue )
{
	if ( !(	V.X >= -MaxValue && V.X <= MaxValue &&
			V.Y >= -MaxValue && V.Y <= MaxValue &&
			V.Z >= -MaxValue && V.Z <= MaxValue ) )	// Don't flip condition. This is intentionally written like this to be NaN-safe.
	{
		V = DefaultValue;
	}
}

void FCluster::SanitizeVertexData()
{
	const float FltThreshold = NANITE_MAX_COORDINATE_VALUE;

	for( uint32 VertexIndex = 0; VertexIndex < NumVerts; VertexIndex++ )
	{
		FVector3f& Position = GetPosition( VertexIndex );
		SanitizeFloat( Position.X, -FltThreshold, FltThreshold, 0.0f );
		SanitizeFloat( Position.Y, -FltThreshold, FltThreshold, 0.0f );
		SanitizeFloat( Position.Z, -FltThreshold, FltThreshold, 0.0f );

		FVector3f& Normal = GetNormal( VertexIndex );
		SanitizeVector( Normal, FltThreshold, FVector3f::UpVector );

		if( VertexFormat.bHasTangents )
		{
			FVector3f& TangentX = GetTangentX( VertexIndex );
			SanitizeVector( TangentX, FltThreshold, FVector3f::ForwardVector );

			float& TangentYSign = GetTangentYSign( VertexIndex );
			TangentYSign = TangentYSign < 0.0f ? -1.0f : 1.0f;
		}

		if( VertexFormat.bHasColors )
		{
			FLinearColor& Color = GetColor( VertexIndex );
			SanitizeFloat( Color.R, 0.0f, 1.0f, 1.0f );
			SanitizeFloat( Color.G, 0.0f, 1.0f, 1.0f );
			SanitizeFloat( Color.B, 0.0f, 1.0f, 1.0f );
			SanitizeFloat( Color.A, 0.0f, 1.0f, 1.0f );
		}

		if( VertexFormat.NumTexCoords > 0 )
		{
			FVector2f* UVs = GetUVs( VertexIndex );
			for( uint32 UVIndex = 0; UVIndex < VertexFormat.NumTexCoords; UVIndex++ )
			{
				SanitizeFloat( UVs[ UVIndex ].X, -FltThreshold, FltThreshold, 0.0f );
				SanitizeFloat( UVs[ UVIndex ].Y, -FltThreshold, FltThreshold, 0.0f );
			}
		}

		if( VertexFormat.NumBoneInfluences > 0 )
		{
			FVector2f* BoneInfluences = GetBoneInfluences( VertexIndex );
			for( uint32 Influence = 0; Influence < VertexFormat.NumBoneInfluences; Influence++ )
			{
				SanitizeFloat( BoneInfluences[Influence].X, 0.0f, FltThreshold, 0.0f );
				SanitizeFloat( BoneInfluences[Influence].Y, 0.0f, FltThreshold, 0.0f );
			}
		}
	}
}
```

对顶点数据进行清洗，确保顶点属性值合法。 Nanite 后续对顶点数据要做： bitstream 压缩、通过 hash/memcmp 做顶点复用以及量化取整并写入固定 bit 数，如果有不合法（ NaN/Inf/异常值）顶点属性会很容易导致量化溢出、 Decode 出错等问题，所以需要先进行清洗

### 6.2. 构建材质 Range ，并重排序三角形 Mesh

```cpp
void FCluster::BuildMaterialRanges()
{
	check( MaterialRanges.Num() == 0 );
	check( NumTris * 3 == Indexes.Num() );

	TArray< int32, TInlineAllocator<128> > MaterialElements;
	TArray< int32, TInlineAllocator<64> > MaterialCounts;

	MaterialElements.AddUninitialized( MaterialIndexes.Num() );
	MaterialCounts.AddZeroed( NANITE_MAX_CLUSTER_MATERIALS );

	// Tally up number per material index
	// 根据 MaterialIndexes 统计每个材质的数量
	for( int32 i = 0; i < MaterialIndexes.Num(); i++ )
	{
		MaterialElements[i] = i;
		MaterialCounts[ MaterialIndexes[i] ]++;
	}

	// Sort by range count descending, and material index ascending.
	// This groups the material ranges from largest to smallest, which is
	// more efficient for evaluating the sequences on the GPU, and also makes
	// the minus one encoding work (the first range must have more than 1 tri).
	// 排序 MaterialElements
	MaterialElements.Sort(
		[&]( int32 A, int32 B )
		{
			int32 IndexA = MaterialIndexes[A];
			int32 IndexB = MaterialIndexes[B];
			int32 CountA = MaterialCounts[ IndexA ];
			int32 CountB = MaterialCounts[ IndexB ];

			if( CountA != CountB )
				return CountA > CountB;

			if( IndexA != IndexB )
				return IndexA < IndexB;

			return A < B;
		} );

	/*
	 * 根据排好序的 MaterialElements ，生成连续的 MaterialRanges
	 */
	FMaterialRange CurrentRange;
	CurrentRange.RangeStart = 0;
	CurrentRange.RangeLength = 0;
	CurrentRange.MaterialIndex = MaterialElements.Num() > 0 ? MaterialIndexes[ MaterialElements[0] ] : 0;

	for( int32 i = 0; i < MaterialElements.Num(); i++ )
	{
		int32 MaterialIndex = MaterialIndexes[ MaterialElements[i] ];

		// Material changed, so add current range and reset
		// 材质 ID 有变化，则将当前 CurrentRange 添加进 MaterialRanges ，并重置 CurrentRange
		if (CurrentRange.RangeLength > 0 && MaterialIndex != CurrentRange.MaterialIndex)
		{
			MaterialRanges.Add(CurrentRange);

			CurrentRange.RangeStart = i;
			CurrentRange.RangeLength = 1;
			CurrentRange.MaterialIndex = MaterialIndex;
		}
		else
		{
			++CurrentRange.RangeLength;	// 材质 ID 相同，则累计 RangeLength
		}
	}

	// Add last triangle to range
	if (CurrentRange.RangeLength > 0)
	{
		MaterialRanges.Add(CurrentRange);
	}

	// 把 Cluster 的 Indexes 和 MaterialIndexes 按照泡好序的 MaterialElements 顺序写回
	if( NumTris )
	{
		TArray< uint32 >	NewIndexes;
		TArray< int32 >		NewMaterialIndexes;
	
		NewIndexes.AddUninitialized( Indexes.Num() );
		NewMaterialIndexes.AddUninitialized( MaterialIndexes.Num() );
	
		for( uint32 NewIndex = 0; NewIndex < NumTris; NewIndex++ )
		{
			uint32 OldIndex = MaterialElements[ NewIndex ];
			NewIndexes[ NewIndex * 3 + 0 ] = Indexes[ OldIndex * 3 + 0 ];
			NewIndexes[ NewIndex * 3 + 1 ] = Indexes[ OldIndex * 3 + 1 ];
			NewIndexes[ NewIndex * 3 + 2 ] = Indexes[ OldIndex * 3 + 2 ];
			NewMaterialIndexes[ NewIndex ] = MaterialIndexes[ OldIndex ];
		}
		Swap( Indexes,			NewIndexes );
		Swap( MaterialIndexes,	NewMaterialIndexes );
	}
	else
	{
		const uint32 VertSize = GetVertSize();

		TArray< float >		NewVerts;
		TArray< int32 >		NewMaterialIndexes;
		TArray< FBrick >	NewBricks;
	
		NewVerts.AddUninitialized( Verts.Num() );
		NewMaterialIndexes.AddUninitialized( MaterialIndexes.Num() );
		NewBricks.AddUninitialized( Bricks.Num() );
		NumVerts = 0;

		for( int32 NewIndex = 0; NewIndex < MaterialElements.Num(); NewIndex++ )
		{
			int32 OldIndex = MaterialElements[ NewIndex ];

			NewMaterialIndexes[ NewIndex ] = MaterialIndexes[ OldIndex ];

			FBrick& OldBrick = Bricks[ OldIndex ];
			FBrick& NewBrick = NewBricks[ NewIndex ];

			uint32 NumVoxels = FMath::CountBits( OldBrick.VoxelMask );
			
			NewBrick = OldBrick;
			NewBrick.VertOffset = NumVerts;
			NumVerts += NumVoxels;

			FMemory::Memcpy( &NewVerts[ NewBrick.VertOffset * VertSize ], &GetPosition( OldBrick.VertOffset ), NumVoxels * VertSize * sizeof( float ) );
		}
		Swap( Verts,			NewVerts );
		Swap( MaterialIndexes,	NewMaterialIndexes );
		Swap( Bricks,			NewBricks );
	}
}
```

1. 根据 `MaterialIndexes` 统计每个材质的数量，其中**材质的数量表示的是此材质被多少个三角形 Mesh 使用**，通过 `MaterialElements` 记录材质索引， `MaterialCounts` 记录材质的数量
2. 排序 `MaterialElements` ，排序规则：
   1. MaterialCount 降序：大材质块优先
   2. MaterialIndex 升序：同材质块大小则按材质 index
   3. MaterialIndexes 数组中索引升序：同材质块大小并且同材质 index ，则按照 MaterialIndexes 数组中索引
3. 按排好序的 `MaterialElements` ，生成连续的 MaterialRanges ： `MaterialRanges = [ { RangeStart, RangeLength, MaterialIndex0 }, { RangeStart, RangeLength, MaterialIndex1 }, ... ]`
4. 重排 Cluster 的三角形 Mesh ：把 Cluster 的 Indexes 和 MaterialIndexes 按照排好序的 `MaterialElements` 顺序写回

#### 6.2.1. 为什么要把三角形 Mesh 聚类为大段连续的 Range

在后续 `PackMaterialInfo` 方法中有两种编码路径：

- **Fast path** ：总的材质数量小于等于 3 （ `Cluster.MaterialRanges.Num() <= 3` ），直接用 32-bit inline 编码（ `PackMaterialFastPath` 方法）
- **Slow path** ：总的材质数量大于 3 （ `Cluster.MaterialRanges.Num() > 3` ），将其写入 MaterialTable （ `PackMaterialTableRange` 方法 + `PackMaterialSlowPath` 方法）

将三角形 Mesh 聚类为大段连续的 Range 之后：

- GPU 解码、材质选择更高效，可以直接范围判断
- 更容易落入 Fast path ，就算是落入 Slow path ， MaterialTable 也更紧凑

### 6.3. 对 Cluster 添加硬约束

Nanite cluster 有一个硬约束：三角形 Mesh 访问顶点时，顶点引用必须落在一个固定大小的 trailing window 内，这里的窗口大小是 `#define CONSTRAINED_CLUSTER_CACHE_SIZE 32` ，通俗来说：**任意三角形 Mesh 的 Vertex Index 必须位于最近 32 个顶点的范围内**，也就是任意三角形 Mesh 的 3 个 Vertext Index 都满足 `MaxVertexIndex - Index < 32`

在 `ConstrainClusters` 方法中：按照 Cluster 中已经排好序的 MaterialRanges 的顺序重新排列三角形 Mesh 和顶点，并且在必要时复制顶点（这也就是为什么 `ConstrainClusters` 后 Cluster 的顶点数可能增加），使 Cluster 满足这个硬约束。需要注意的是：**三角形 Mesh 和顶点的重排序是在每个 MaterialRange 内进行的，不会跨 Material ，否则会破坏 Material Batching**

#### 6.3.1. 为什么要这个 trailing window 约束

Nanite 的 Index 编码不是普通的 8/16/32-bit Index Buffer ，而是：

1. 以 **Base Index + 两个 5-bit offsets** 的形式编码三角形 Mesh ： `const uint32 BitsPerTriangle = BitsPerIndex + 2 * 5;	// Base index + two 5-bit offsets`
2. 或 strip 的形式，用 5-bit delta 引用前面的顶点

所以**必须保证与引用顶点的相对偏移能用 5 bits （ 0..31 ）来表达**

#### 6.3.2. 添加约束后顶点数 > 256 的额外处理

如果对某个 Cluster 添加约束后其顶点数 > 256 ，则会对其进行拆分，拆分会：

1. 复制原 Cluster 的顶点数组（先粗复制三角形 Mesh 的范围）
2. 重建 MaterialRanges
3. 再跑 Constrain/Stripfy
4. 把新 Cluster 加回同一个 Group 的 Children 中

### 6.4. 对每个 MaterialRange, 把它内部的三角形 Mesh 再划分成多个 Batch

```cpp
static void BuildVertReuseBatches(FCluster& Cluster)
{
	// 将每个 MaterialRange 再划分成多个 Batch
	for (FMaterialRange& MaterialRange : Cluster.MaterialRanges)
	{
		TStaticBitArray<NANITE_MAX_CLUSTER_VERTICES> UsedVertMask;
		uint32 NumUniqueVerts = 0;
		uint32 NumTris = 0;
		
		// 每个 Batch 要满足 Unique Verts <= 32 **并且** 三角形 Mesh 数量 <= 32
		const uint32 MaxBatchVerts = 32;
		const uint32 MaxBatchTris = 32;

		const uint32 TriIndexEnd = MaterialRange.RangeStart + MaterialRange.RangeLength;

		MaterialRange.BatchTriCounts.Reset();

		for (uint32 TriIndex = MaterialRange.RangeStart; TriIndex < TriIndexEnd; ++TriIndex)
		{
			const uint32 VertIndex0 = Cluster.Indexes[TriIndex * 3 + 0];
			const uint32 VertIndex1 = Cluster.Indexes[TriIndex * 3 + 1];
			const uint32 VertIndex2 = Cluster.Indexes[TriIndex * 3 + 2];

			auto Bit0 = UsedVertMask[VertIndex0];
			auto Bit1 = UsedVertMask[VertIndex1];
			auto Bit2 = UsedVertMask[VertIndex2];

			// If adding this tri to the current batch will result in too many unique verts, start a new batch
			// 添加当前三角形 Mesh 后会达到单个 Batch 的 Unique Verts 数量上限, MaterialRange.BatchTriCounts 记录此 Batch 当前的三角形 Mesh 数量, 遍历回退到上一个 TriIndex
			const uint32 NumNewUniqueVerts = uint32(!Bit0) + uint32(!Bit1) + uint32(!Bit2);
			if (NumUniqueVerts + NumNewUniqueVerts > MaxBatchVerts)
			{
				check(NumTris > 0);
				MaterialRange.BatchTriCounts.Add(uint8(NumTris));
				NumUniqueVerts = 0;
				NumTris = 0;
				UsedVertMask = TStaticBitArray<NANITE_MAX_CLUSTER_VERTICES>();
				--TriIndex;
				continue;
			}

			Bit0 = true;
			Bit1 = true;
			Bit2 = true;
			NumUniqueVerts += NumNewUniqueVerts;
			++NumTris;

			// 达到单个 Batch 的三角形 Mesh 数量上限, MaterialRange.BatchTriCounts 记录此 Batch 的三角形 Mesh 数量
			if (NumTris == MaxBatchTris)
			{
				MaterialRange.BatchTriCounts.Add(uint8(NumTris));
				NumUniqueVerts = 0;
				NumTris = 0;
				UsedVertMask = TStaticBitArray<NANITE_MAX_CLUSTER_VERTICES>();
			}
		}

		if (NumTris > 0)
		{
			MaterialRange.BatchTriCounts.Add(uint8(NumTris));
		}
	}
}
```

对于每个 `MaterialRange` ，把它内部的三角形 Mesh 再划分成多个 **Batch** ，每个 Batch 同时满足：

1. Unique Verts 数量小于等于 32
2. 三角形 Mesh 数量小于等于 32

并且在 `MaterialRange.BatchTriCounts` 中记录每个 Batch 中的三角形 Mesh 数量

把 MaterialRange 再划分为 Batch ，主要是以下 4 个核心原因：

1. **提高 GPU 并行度（最重要）**

	每个 MaterialRange 中的三角形 Mesh 数量不一致，它们之间的差异可能会很大，如果直接执行，会导致 GPU 负载不均衡，部分线程空闲，而拆成 Batch 后， GPU 可以同时执行多个 Batch ，提高录用率

2. **匹配 GPU 得执行粒度（适配 Compute Shader ）**

	Nanite 使用 Compute Shader 光栅化，而 Compute Shader 喜欢固定大小的任务， MaterialRange 太大或不规则，不适合直接执行

3. **提高剔除精度（减少无效渲染）**

	将 MaterialRange 再划分为多个 Batch 后，意味着可以剔除部分 Batch ，而不是整个 MaterialRange ，可以提高性能

**总结**：

**MaterialRange 是材质分组单位，而 Batch 是 GPU 执行单位**。将 MaterialRange 再划分为多个 Batch 是为了**提高并行度、提高 GPU 利用率、提高剔除精度**

### 6.5. 顶点位置量化: 决定每个 Cluster 的 Position 怎么变成整数并写入 bitstream

```cpp
static int32 CalculateQuantizedPositionsUniformGrid(TArray< FCluster >& Clusters, const FMeshNaniteSettings& Settings)
{
	// Simple global quantization for EA
	// 最大允许量化值, 其中 NANITE_MAX_POSITION_QUANTIZATION_BITS = 21, 所以 MaxPositionQuantizedValue = 2^21 - 1 = 2097151
	// 即 Position 的 X/Y/Z 每个轴最多 21 bits
	const int32 MaxPositionQuantizedValue	= (1 << NANITE_MAX_POSITION_QUANTIZATION_BITS) - 1;

	{
		// Make sure the worst case bounding box fits with the position encoding settings. Ideally this would be a compile-time check.
		const float MaxValue = FMath::RoundToFloat(NANITE_MAX_COORDINATE_VALUE * FMath::Exp2((float)NANITE_MIN_POSITION_PRECISION));
		checkf(MaxValue <= FLT_INT_MAX && int64(MaxValue) - int64(-MaxValue) <= MaxPositionQuantizedValue, TEXT("Largest cluster bounds doesn't fit in position bits"));
	}
	
	// 选择 PositionPrecision
	int32 PositionPrecision = Settings.PositionPrecision;
	// Settings.PositionPrecision == MIN_int32 表示是 Auto 模式
	if (PositionPrecision == MIN_int32)
	{
		// Heuristic: We want higher resolution if the mesh is denser.
		// Use geometric average of cluster size as a proxy for density.
		// Alternative interpretation: Bit precision is average of what is needed by the clusters.
		// For roughly uniformly sized clusters this gives results very similar to the old quantization code.

		// Auto 模式下根据 MipLevel == 0 的 Cluster Size 估计合适精度
		double TotalLogSize = 0.0;
		int32 TotalNum = 0;
		for (const FCluster& Cluster : Clusters)
		{
			if (Cluster.MipLevel == 0)
			{
				// Bounds Extent Size
				float ExtentSize = Cluster.Bounds.GetExtent().Size();
				if (ExtentSize > 0.0)
				{
					// 累计 log2(Size)
					TotalLogSize += FMath::Log2(ExtentSize);
					TotalNum++;
				}
			}
		}
		// 算平均
		double AvgLogSize = TotalNum > 0 ? TotalLogSize / TotalNum : 0.0;
		// 使用 7 - RoundToInt(AvgLogSize) , 这里的 7 是经验值
		PositionPrecision = 7 - (int32)FMath::RoundToInt(AvgLogSize);

		// Clamp precision. The user now needs to explicitly opt-in to the lowest precision settings.
		// These settings are likely to cause issues and contribute little to disk size savings (~0.4% on test project),
		// so we shouldn't pick them automatically.
		// Example: A very low resolution road or building frame that needs little precision to look right in isolation,
		// but still requires fairly high precision in a scene because smaller meshes are placed on it or in it.
		const int32 AUTO_MIN_PRECISION = 4;	// 1/16cm
		// PositionPrecision 下限 Clamp 到 AUTO_MIN_PRECISION = 4
		PositionPrecision = FMath::Max(PositionPrecision, AUTO_MIN_PRECISION);
	}

	// PositionPrecision 最终 Clamp 到 [NANITE_MIN_POSITION_PRECISION, NANITE_MAX_POSITION_PRECISION]
	// 其中 NANITE_MIN_POSITION_PRECISION = -20 , NANITE_MAX_POSITION_PRECISION = 43
	// 对应 QuantizationScale = 2^-20 到 2^43
	PositionPrecision = FMath::Clamp(PositionPrecision, NANITE_MIN_POSITION_PRECISION, NANITE_MAX_POSITION_PRECISION);

	// QuantizationScale = 2^PositionPrecision
	float QuantizationScale = FMath::Exp2((float)PositionPrecision);

	// Make sure all clusters are encodable. A large enough cluster could hit the 21bpc limit. If it happens scale back until it fits.
	// 确保所有 Cluster 都能编码 ( Fits in 21 bits )
	for (const FCluster& Cluster : Clusters)
	{
		const FBounds3f& Bounds = Cluster.Bounds;
		
		int32 Iterations = 0;
		while (true)
		{
			// 每个 Cluster Bounds 的 Min/Max
			float MinX = FMath::RoundToFloat(Bounds.Min.X * QuantizationScale);
			float MinY = FMath::RoundToFloat(Bounds.Min.Y * QuantizationScale);
			float MinZ = FMath::RoundToFloat(Bounds.Min.Z * QuantizationScale);

			float MaxX = FMath::RoundToFloat(Bounds.Max.X * QuantizationScale);
			float MaxY = FMath::RoundToFloat(Bounds.Max.Y * QuantizationScale);
			float MaxZ = FMath::RoundToFloat(Bounds.Max.Z * QuantizationScale);

			// 检查 Min/Max 在 int 表示范围内, 并且 Max - Min <= MaxPositionQuantizedValue
			if (MinX >= FLT_INT_MIN && MinY >= FLT_INT_MIN && MinZ >= FLT_INT_MIN &&
				MaxX <= FLT_INT_MAX && MaxY <= FLT_INT_MAX && MaxZ <= FLT_INT_MAX &&
				((int64)MaxX - (int64)MinX) <= MaxPositionQuantizedValue && ((int64)MaxY - (int64)MinY) <= MaxPositionQuantizedValue && ((int64)MaxZ - (int64)MinZ) <= MaxPositionQuantizedValue)
			{
				break;
			}
			
			// 如果 Cluster 太大, 也就是量化后的范围超过 21 bits, 则降低精度, 直到满足
			QuantizationScale *= 0.5f;
			PositionPrecision--;
			check(PositionPrecision >= NANITE_MIN_POSITION_PRECISION);
			check(++Iterations < 100);	// Endless loop?
		}
	}

	// 真正量化每个 Vertex
	const float RcpQuantizationScale = 1.0f / QuantizationScale;

	// 对每个 Cluster
	ParallelFor( TEXT("NaniteEncode.QuantizeClusterPositions.PF"), Clusters.Num(), 256, [&](uint32 ClusterIndex)
	{
		FCluster& Cluster = Clusters[ClusterIndex];
		
		const uint32 NumClusterVerts = Cluster.NumVerts;
		Cluster.QuantizedPositions.SetNumUninitialized(NumClusterVerts);

		// Quantize positions
		FIntVector IntClusterMax = { MIN_int32,	MIN_int32, MIN_int32 };
		FIntVector IntClusterMin = { MAX_int32,	MAX_int32, MAX_int32 };

		for (uint32 i = 0; i < NumClusterVerts; i++)
		{
			const FVector3f Position = Cluster.GetPosition(i);

			// 写入量化后的 IntPosition: IntPosition = round(Position * QuantizationScale)
			FIntVector& IntPosition = Cluster.QuantizedPositions[i];
			float PosX = FMath::RoundToFloat(Position.X * QuantizationScale);
			float PosY = FMath::RoundToFloat(Position.Y * QuantizationScale);
			float PosZ = FMath::RoundToFloat(Position.Z * QuantizationScale);

			IntPosition = FIntVector((int32)PosX, (int32)PosY, (int32)PosZ);

			// 统计 Cluster 所有顶点 IntPosition 的 Min/Max
			IntClusterMax.X = FMath::Max(IntClusterMax.X, IntPosition.X);
			IntClusterMax.Y = FMath::Max(IntClusterMax.Y, IntPosition.Y);
			IntClusterMax.Z = FMath::Max(IntClusterMax.Z, IntPosition.Z);
			IntClusterMin.X = FMath::Min(IntClusterMin.X, IntPosition.X);
			IntClusterMin.Y = FMath::Min(IntClusterMin.Y, IntPosition.Y);
			IntClusterMin.Z = FMath::Min(IntClusterMin.Z, IntPosition.Z);
		}

		// Store in minimum number of bits
		// 计算 X/Y/Z 每个轴所需 bits
		const uint32 NumBitsX = FMath::CeilLogTwo(IntClusterMax.X - IntClusterMin.X + 1);
		const uint32 NumBitsY = FMath::CeilLogTwo(IntClusterMax.Y - IntClusterMin.Y + 1);
		const uint32 NumBitsZ = FMath::CeilLogTwo(IntClusterMax.Z - IntClusterMin.Z + 1);
		check(NumBitsX <= NANITE_MAX_POSITION_QUANTIZATION_BITS);
		check(NumBitsY <= NANITE_MAX_POSITION_QUANTIZATION_BITS);
		check(NumBitsZ <= NANITE_MAX_POSITION_QUANTIZATION_BITS);

		for (uint32 i = 0; i < NumClusterVerts; i++)
		{
			FIntVector& IntPosition = Cluster.QuantizedPositions[i];

			// Update float position with quantized data
			// 把 Position 写回量化后的 float (乘 RcpQuantizationScale )
			Cluster.GetPosition(i) = FVector3f((float)IntPosition.X * RcpQuantizationScale, (float)IntPosition.Y * RcpQuantizationScale, (float)IntPosition.Z * RcpQuantizationScale);
			
			// 关键优化: IntPosition -= IntClusterMin , 存储的 IntPosition 是相对 IntClusterMin 的位置, 而不是绝对位置, 极大降低 bit 需求
			IntPosition.X -= IntClusterMin.X;
			IntPosition.Y -= IntClusterMin.Y;
			IntPosition.Z -= IntClusterMin.Z;
			check(IntPosition.X >= 0 && IntPosition.X < (1 << NumBitsX));
			check(IntPosition.Y >= 0 && IntPosition.Y < (1 << NumBitsY));
			check(IntPosition.Z >= 0 && IntPosition.Z < (1 << NumBitsZ));
		}


		// Update bounds
		// 把 Bounds 写回量化后的 float (乘 RcpQuantizationScale )
		Cluster.Bounds.Min = FVector3f((float)IntClusterMin.X * RcpQuantizationScale, (float)IntClusterMin.Y * RcpQuantizationScale, (float)IntClusterMin.Z * RcpQuantizationScale);
		Cluster.Bounds.Max = FVector3f((float)IntClusterMax.X * RcpQuantizationScale, (float)IntClusterMax.Y * RcpQuantizationScale, (float)IntClusterMax.Z * RcpQuantizationScale);

		// 后续写入 Page 时, Position 按 QuantizedPosBits 写 bitstream
		Cluster.QuantizedPosBits = FIntVector(NumBitsX, NumBitsY, NumBitsZ);
		Cluster.QuantizedPosStart = IntClusterMin;
		Cluster.QuantizedPosPrecision = PositionPrecision;

	} );
	return PositionPrecision;
}
```

这个函数的本质是：**把 Cluster 的 float 顶点坐标量化为整数，并用最少 bit 存储，同时保证精度和范围安全**。主要做了以下 4 件事：

- 选择合适量化精度
- float -> int 量化
- 转换为 Cluster 局部坐标
- 为 X/Y/Z 每个轴计算最小 bit 数

如果量化后范围超过 21 bit 上限，则自动降低精度直到安全：

```cpp
QuantizationScale *= 0.5;
PositionPrecision--;
```

最终 Cluster 存储的数据：

```cpp
Cluster.QuantizedPositions	  // 量化整数位置
Cluster.QuantizedPosStart	   // Cluster 最小值
Cluster.QuantizedPosBits		// 每个轴 bit 数
Cluster.QuantizedPosPrecision   // 精度
```

GPU 端解码（恢复 float ）公式：

```cpp
FloatPosition = (StoredPosition + QuantizedPosStart) * (1 / QuantizationScale);
```

仅需一次 `add` 一次 `mul` ，非常快

#### 6.5.1. 选择合适量化精度

```cpp
QuantizationScale = 2^PositionPrecision
```

Auto 模式根据 Cluster 尺寸平均值估算合适精度值。 PositionPrecision 越大，精度越高； PositionPrecision 越小，存储越省。

#### 6.5.2. float -> int 量化

把浮点数变成整数，方便压缩存储

量化：

```cpp
IntPosition = round(FloatPosition * QuantizationScale)
```

反量化：

```cpp
FloatPosition = IntPosition / QuantizationScale
```

PositionPrecision 决定精度：

```cpp
精度 = 1 / PositionPrecision
```

#### 6.5.3. 转换为 Cluster 局部坐标

**将坐标转换为 Cluster 局部坐标**是压缩的关键，可以极大减少所需 bit 数。最终存储的坐标是：

```cpp
StoredPosition = IntPosition - IntClusterMin
```

举个例子：

```cpp
// 假如原始要存储的坐标是:
100000 -> 100300	// 需要 17 bit

// 转换后:
0 -> 300			// 只需要 9 bit
```

#### 6.5.4. 为 X/Y/Z 每个轴计算最小 bit 数

```cpp
NumBitsX = ceil(log2(RangeX));
NumBitsY = ceil(log2(RangeY));
NumBitsZ = ceil(log2(RangeZ));
```

### 6.6. 计算每个 Cluster 的编码信息

```cpp
static void CalculateEncodingInfo(FEncodingInfo& Info, const Nanite::FCluster& Cluster, int32 NormalPrecision, int32 TangentPrecision, int32 BoneWeightPrecision)
{
	const uint32 NumClusterVerts	= Cluster.NumVerts;
	const uint32 NumClusterTris		= Cluster.NumTris;
	const uint32 MaxBones			= Cluster.VertexFormat.NumBoneInfluences;

	FMemory::Memzero(Info);

	// Write triangles indices. Indices are stored in a dense packed bitstream using ceil(log2(NumClusterVerices)) bits per index. The shaders implement unaligned bitstream reads to support this.
	// 存储一个顶点索引 Index 需要多少 bit （根据 Cluster 顶点数量大小计算）
	const uint32 BitsPerIndex = NumClusterVerts > 1 && NumClusterTris > 1 ? (FGenericPlatformMath::FloorLog2(NumClusterVerts - 1) + 1) : 1;
	// 存储一个三角形 Mesh 的 3 个顶点需要多少 bit
	// 需要注意的是: Nanite 中并不是直接存储三角形 Mesh 的 3 个顶点索引 Index , 而是存储一个 BaseIndex + 2 个 5-bit 的 Offsets
	const uint32 BitsPerTriangle = BitsPerIndex + 2 * 5;	// Base index + two 5-bit offsets
	Info.BitsPerIndex = BitsPerIndex;

	FPageSections& GpuSizes = Info.GpuSizes;

	// 固定结构部分:
	// Cluster Header: 包含 Bounding Box, LOD Info, Offsets, Counts
	GpuSizes.Cluster = sizeof(FPackedCluster);
	// MaterialIndex 表
	GpuSizes.MaterialTable = CalcMaterialTableSize(Cluster) * sizeof(uint32);
	// MaterialRange Batches
	GpuSizes.VertReuseBatchInfo = Cluster.NumTris && Cluster.MaterialRanges.Num() > 3 ? CalcVertReuseBatchInfoSize(Cluster.MaterialRanges) * sizeof(uint32) : 0;
	// UV Decode Header
	GpuSizes.DecodeInfo = Cluster.VertexFormat.NumTexCoords * sizeof(FPackedUVHeader) + (MaxBones > 0 ? sizeof(FPackedBoneInfluenceHeader) : 0);

	// 最终 Index Buffer 大小, 以 32 对齐并转换为 byte
	GpuSizes.Index = (NumClusterTris * BitsPerTriangle + 31) / 32 * 4;

	GpuSizes.BrickData = Cluster.Bricks.Num() * sizeof(FPackedBrick);

#if NANITE_USE_UNCOMPRESSED_VERTEX_DATA
	const uint32 AttribBytesPerVertex = (3 * sizeof(float) + (Cluster.VertexFormat.bHasTangents ? (4 * sizeof(float)) : 0) + sizeof(uint32) + Cluster.VertexFormat.NumTexCoords * 2 * sizeof(float));

	Info.BitsPerAttribute = AttribBytesPerVertex * 8;
	Info.ColorMin = FIntVector4(0, 0, 0, 0);
	Info.ColorBits = FIntVector4(8, 8, 8, 8);
	Info.ColorMode = NANITE_VERTEX_COLOR_MODE_VARIABLE;
	Info.NormalPrecision = 0;
	Info.TangentPrecision = 0;

	// TODO: Nanite-Skinning: Implement uncompressed path

	GpuSizes.Position = NumClusterVerts * 3 * sizeof(float);
	GpuSizes.Attribute = NumClusterVerts * AttribBytesPerVertex;
#else

	// Info.BitsPerAttribute 中记录每个 vertex attribute 占多少 bit

	// Normal: Nanite 使用 Octahedral encoding, 例如 NormalPrecision = 8, 那么 Normal = 16 bits (原 float3 则是 4 * 8 * 3 = 96 bits)
	Info.BitsPerAttribute = 2 * NormalPrecision;

	// Tangent: Sign bit + Tangent direction, 例如 TangentPrecision = 8, 那么 Tangent = 1 + 8 = 9 bits
	if (Cluster.VertexFormat.bHasTangents)
	{
		Info.BitsPerAttribute += 1 + TangentPrecision;
	}

	check(NumClusterVerts > 0);
	const bool bIsLeaf = (Cluster.GeneratingGroupIndex == MAX_uint32);

	// Normals
	Info.NormalPrecision = NormalPrecision;
	Info.TangentPrecision = TangentPrecision;

	// Vertex colors
	// 顶点颜色与 Position 类似, 使用 delta encoding, 先找 Min/Max , 然后计算 delta, 根据 delta 计算需要的 bit 数
	Info.ColorMode = NANITE_VERTEX_COLOR_MODE_CONSTANT;
	Info.ColorMin = FIntVector4(255, 255, 255, 255);
	if (Cluster.VertexFormat.bHasColors)
	{
		FIntVector4 ColorMin = FIntVector4( 255, 255, 255, 255);
		FIntVector4 ColorMax = FIntVector4( 0, 0, 0, 0);
		for (uint32 i = 0; i < NumClusterVerts; i++)
		{
			FColor Color = Cluster.GetColor(i).ToFColor(false);
			ColorMin.X = FMath::Min(ColorMin.X, (int32)Color.R);
			ColorMin.Y = FMath::Min(ColorMin.Y, (int32)Color.G);
			ColorMin.Z = FMath::Min(ColorMin.Z, (int32)Color.B);
			ColorMin.W = FMath::Min(ColorMin.W, (int32)Color.A);
			ColorMax.X = FMath::Max(ColorMax.X, (int32)Color.R);
			ColorMax.Y = FMath::Max(ColorMax.Y, (int32)Color.G);
			ColorMax.Z = FMath::Max(ColorMax.Z, (int32)Color.B);
			ColorMax.W = FMath::Max(ColorMax.W, (int32)Color.A);
		}

		const FIntVector4 ColorDelta = ColorMax - ColorMin;
		const int32 R_Bits = FMath::CeilLogTwo(ColorDelta.X + 1);
		const int32 G_Bits = FMath::CeilLogTwo(ColorDelta.Y + 1);
		const int32 B_Bits = FMath::CeilLogTwo(ColorDelta.Z + 1);
		const int32 A_Bits = FMath::CeilLogTwo(ColorDelta.W + 1);
		
		uint32 NumColorBits = R_Bits + G_Bits + B_Bits + A_Bits;
		Info.BitsPerAttribute += NumColorBits;
		Info.ColorMin = ColorMin;
		Info.ColorBits = FIntVector4(R_Bits, G_Bits, B_Bits, A_Bits);
		if (NumColorBits > 0)
		{
			Info.ColorMode = NANITE_VERTEX_COLOR_MODE_VARIABLE;
		}
	}

	const int NumMantissaBits = NANITE_UV_FLOAT_NUM_MANTISSA_BITS;	//TODO: make this a build setting
	for( uint32 UVIndex = 0; UVIndex < Cluster.VertexFormat.NumTexCoords; UVIndex++ )
	{
		FUintVector2 UVMin = FUintVector2(0xFFFFFFFFu, 0xFFFFFFFFu);
		FUintVector2 UVMax = FUintVector2(0u, 0u);

		for (uint32 i = 0; i < NumClusterVerts; i++)
		{
			const FVector2f& UV = Cluster.GetUVs(i)[UVIndex];

			// 将 UV 通过 EncodeUVFloat 方法转换为 fixed point integer
			const uint32 EncodedU = EncodeUVFloat(UV.X, NumMantissaBits);
			const uint32 EncodedV = EncodeUVFloat(UV.Y, NumMantissaBits);

			// 找 Min/Max
			UVMin.X = FMath::Min(UVMin.X, EncodedU);
			UVMin.Y = FMath::Min(UVMin.Y, EncodedV);
			UVMax.X = FMath::Max(UVMax.X, EncodedU);
			UVMax.Y = FMath::Max(UVMax.Y, EncodedV);
		}

		// 存 delta
		const FUintVector2 UVDelta = UVMax - UVMin;

		// 最终存储 UVMin 和 delta
		FUVInfo& UVInfo = Info.UVs[UVIndex];
		UVInfo.Min				= UVMin;
		UVInfo.NumBits.X		= FMath::CeilLogTwo(UVDelta.X + 1u);
		UVInfo.NumBits.Y		= FMath::CeilLogTwo(UVDelta.Y + 1u);

		Info.BitsPerAttribute	+= UVInfo.NumBits.X + UVInfo.NumBits.Y;
	}

	// Bone Influence
	if (MaxBones > 0)
	{
		CalculateInfluences(Info.BoneInfluence, Cluster, BoneWeightPrecision);

		// TODO: Nanite-Skinning: Make this more compact. Range of indices? Palette of indices? Omit the last weight?
		const uint32 VertexInfluenceSize	= ( NumClusterVerts * Info.BoneInfluence.NumVertexBoneInfluences * ( Info.BoneInfluence.NumVertexBoneIndexBits + Info.BoneInfluence.NumVertexBoneWeightBits ) + 31) / 32 * 4;
		GpuSizes.BoneInfluence				= VertexInfluenceSize;

		check(GpuSizes.BoneInfluence % 4 == 0);
	}

	// Position
	const uint32 PositionBitsPerVertex = Cluster.QuantizedPosBits.X + Cluster.QuantizedPosBits.Y + Cluster.QuantizedPosBits.Z;
	GpuSizes.Position = (NumClusterVerts * PositionBitsPerVertex + 31) / 32 * 4;
	GpuSizes.Attribute = (NumClusterVerts * Info.BitsPerAttribute + 31) / 32 * 4;
#endif
}
```

#### 6.6.1. UV 压缩

下面是 Nanite UV 压缩的核心编码函数之一，它实现了一种**自定义浮点压缩格式（ Custom Float Type ）**，目的是将 32-bit float 压缩为 bit 更少的整数，同时保持：

- 在 `[0, 1]` 范围提供均匀精度
- 保证编码后的 `uint32` 排序顺序与原 float 一致
- 支持 delta 压缩
- GPU 快速解码

```cpp
static uint32 EncodeUVFloat(float Value, uint32 NumMantissaBits)
{
	// Encode UV floats as a custom float type where [0,1] is denormal, so it gets uniform precision.
	// As UVs are encoded in clusters as ranges of encoded values, a few modifications to the usual
	// float encoding are made to preserve the original float order when the encoded values are interpreted as uints:
	// 1. Positive values use 1 as sign bit.
	// 2. Negative values use 0 as sign bit and have their exponent and mantissa bits inverted.

	checkSlow(FMath::IsFinite(Value));

	// 计算 sign bit 位置
	// 这里的 custom float: [sign][exponent][mantissa], 其中 NANITE_UV_FLOAT_NUM_EXPONENT_BITS = 5, NANITE_UV_FLOAT_NUM_MANTISSA_BITS = 14, 所以 SignBitPosition = 19
	const uint32 SignBitPosition = NANITE_UV_FLOAT_NUM_EXPONENT_BITS + NumMantissaBits;
	// 将 float 重新解释为 uint
	const uint32 FloatUInt = (uint32&)Value;
	// 提取 exponent
	const uint32 Exponent = (FloatUInt >> 23) & 0xFFu;
	// 提取 mantissa
	const uint32 Mantissa = FloatUInt & 0x7FFFFFu;

	// 提取绝对值, 去掉 sign bit
	const uint32 AbsFloatUInt = FloatUInt & 0x7FFFFFFFu;

	uint32 Result;
	// 1.0f == 0x3F800000u, 这里是判断 abs(Value) < 1.0, UV 通常在 [0, 1], 所以大多数都是走这里
	if (AbsFloatUInt < 0x3F800000u)
	{
		// Denormal encoding
		// Note: Mantissa can overflow into first non-denormal value (1.0f),
		// but that is desirable to get correct round-to-nearest behavior.

		// [0, 1] 之间, 走非规格化数编码(Denormal encoding)路径
		// 恢复 float
		const float AbsFloat = (float&)AbsFloatUInt;
		// 这里等价于 Result = round(AbsFloat * 2^NumMantissaBits), 将 [0, 1] 转换到 [0, 2^NumMantissaBits], 保证均匀精度
		Result = uint32(double(AbsFloat * float(1u << NumMantissaBits)) + 0.5);	// Cast to double to make sure +0.5 is lossless
	}
	else
	{
		// Normal encoding
		// Extract exponent and mantissa bits from 32-bit float-
		const uint32 Shift = (23 - NumMantissaBits);
		const uint32 Tmp = (AbsFloatUInt - 0x3F000000u) + (1u << (Shift - 1));	// Bias to round to nearest
		Result = FMath::Min(Tmp >> Shift, (1u << SignBitPosition) - 1u);		// Clamp to largest UV float value
	}

	// Produce a mask that for positive values only flips the sign bit
	// and for negative values only flips the exponent and mantissa bits.

	// (FloatUInt >> 31u )是符号位, 正数时为 0, 负数时为 1, 所以
	// positive: SignMask = (1u << SignBitPosition)
	// negative: SignMask = (1u << SignBitPosition) - 1
	const uint32 SignMask = (1u << SignBitPosition) - (FloatUInt >> 31u);

	// 异或 XOR
	// positive: sign bit 被设置为 1
	// negative: exponent 和 mantissa 全部取反
	// 保证: float a < float b <-> uint encode(a) < uint encode(b), 这是支持 delta encoding/range compression 的关键
	Result ^= SignMask;

#if DO_GUARD_SLOW
	VerifyUVFloatEncoding(Value, Result, NumMantissaBits);
#endif
	return Result;
}
```

### 6.7. 把 Cluster 按 Group 顺序打包进 GPU Page, 同时生成 GroupPart 和实例数据

```cpp
static void SortGroupClusters(TArray<FClusterGroup>& ClusterGroups, const TArray<FCluster>& Clusters)
{
	for (FClusterGroup& Group : ClusterGroups)
	{
		FVector3f SortDirection = FVector3f(1.0f, 1.0f, 1.0f);
		Group.Children.Sort([&Clusters, SortDirection](uint32 ClusterIndexA, uint32 ClusterIndexB) {
			const FCluster& ClusterA = Clusters[ClusterIndexA];
			const FCluster& ClusterB = Clusters[ClusterIndexB];
			// 按 Cluster.SphereBounds.Center 与 (1,1,1) 点积排序(本质是沿对角方向排序)
			float DotA = FVector3f::DotProduct(ClusterA.SphereBounds.Center, SortDirection);
			float DotB = FVector3f::DotProduct(ClusterB.SphereBounds.Center, SortDirection);
			return DotA < DotB;
		});
	}
}

// Generate a permutation of cluster groups that is sorted first by mip level and then by Morton order x, y and z.
// Sorting by mip level first ensure that there can be no cyclic dependencies between formed pages.
static TArray<uint32> CalculateClusterGroupPermutation( const TArray< FClusterGroup >& ClusterGroups )
{
	struct FClusterGroupSortEntry {
		int32	AssemblyPartIndex;
		int32	MipLevel;
		uint32	MortonXYZ;
		uint32	OldIndex;
	};

	uint32 NumClusterGroups = ClusterGroups.Num();
	TArray< FClusterGroupSortEntry > ClusterGroupSortEntries;
	ClusterGroupSortEntries.SetNumUninitialized( NumClusterGroups );

	FVector3f MinCenter = FVector3f( FLT_MAX, FLT_MAX, FLT_MAX );
	FVector3f MaxCenter = FVector3f( -FLT_MAX, -FLT_MAX, -FLT_MAX );
	for( const FClusterGroup& ClusterGroup : ClusterGroups )
	{
		const FVector3f& Center = ClusterGroup.LODBounds.Center;
		MinCenter = FVector3f::Min( MinCenter, Center );
		MaxCenter = FVector3f::Max( MaxCenter, Center );
	}

	const float Scale = 1023.0f / (MaxCenter - MinCenter).GetMax();
	for( uint32 i = 0; i < NumClusterGroups; i++ )
	{
		const FClusterGroup& ClusterGroup = ClusterGroups[ i ];
		FClusterGroupSortEntry& SortEntry = ClusterGroupSortEntries[ i ];

		// Group 中心点归一化到 [0, 1023]
		const FVector3f& Center = ClusterGroup.LODBounds.Center;
		const FVector3f ScaledCenter = ( Center - MinCenter ) * Scale + 0.5f;
		uint32 X = FMath::Clamp( (int32)ScaledCenter.X, 0, 1023 );
		uint32 Y = FMath::Clamp( (int32)ScaledCenter.Y, 0, 1023 );
		uint32 Z = FMath::Clamp( (int32)ScaledCenter.Z, 0, 1023 );

		// 计算 Group 中心点的 Morton 3D 编码
		SortEntry.AssemblyPartIndex = ClusterGroup.AssemblyPartIndex;
		SortEntry.MipLevel = ClusterGroup.MipLevel;
		SortEntry.MortonXYZ = ( FMath::MortonCode3(Z) << 2 ) | ( FMath::MortonCode3(Y) << 1 ) | FMath::MortonCode3(X);
		if ((ClusterGroup.MipLevel & 1) != 0)
		{
			SortEntry.MortonXYZ ^= 0xFFFFFFFFu;	// Alternate order so end of one level is near the beginning of the next
		}
		SortEntry.OldIndex = i;
	}

	// Group 排序:
	// 优先按 MipLevel 降序 (大 MipLevel/粗 LOD 优先), 先按照 MipLevel 排序确保 Page 之间不会存在循环依赖关系
	// 同 MipLevel 内按 Group 中心点的 Morton 编码排序
	ClusterGroupSortEntries.Sort( []( const FClusterGroupSortEntry& A, const FClusterGroupSortEntry& B ) {
		if (A.AssemblyPartIndex != B.AssemblyPartIndex)
			return A.AssemblyPartIndex < B.AssemblyPartIndex;
		if( A.MipLevel != B.MipLevel )
			return A.MipLevel > B.MipLevel;
		return A.MortonXYZ < B.MortonXYZ;
	} );

	TArray<uint32> Permutation;
	Permutation.SetNumUninitialized( NumClusterGroups );
	for( uint32 i = 0; i < NumClusterGroups; i++ )
		Permutation[ i ] = ClusterGroupSortEntries[ i ].OldIndex;
	return Permutation;
}

static bool TryAddClusterToPage(FPage& Page, const FCluster& Cluster, const FEncodingInfo& EncodingInfo, bool bRootPage)
{
	// 临时拷贝当前 Page
	FPage UpdatedPage = Page;

	// Page.NumClusters++
	// 累加 Page.GpuSizes
	UpdatedPage.NumClusters++;
	UpdatedPage.GpuSizes += EncodingInfo.GpuSizes;

	// Calculate sizes that don't just depend on the individual cluster
	if(Cluster.NumTris != 0)
	{
		UpdatedPage.MaxClusterBoneInfluences = FMath::Max(UpdatedPage.MaxClusterBoneInfluences, (uint32)EncodingInfo.BoneInfluence.ClusterBoneInfluences.Num());
	}
	else
	{
		UpdatedPage.MaxVoxelBoneInfluences = FMath::Max(UpdatedPage.MaxVoxelBoneInfluences, (uint32)EncodingInfo.BoneInfluence.VoxelBoneInfluences.Num());
	}
	
	UpdatedPage.GpuSizes.ClusterBoneInfluence = UpdatedPage.NumClusters * UpdatedPage.MaxClusterBoneInfluences * sizeof(FClusterBoneInfluence);
	UpdatedPage.GpuSizes.VoxelBoneInfluence = UpdatedPage.NumClusters * UpdatedPage.MaxVoxelBoneInfluences * sizeof(FPackedVoxelBoneInfluence);

	// 判断 Page 的 GPU Size 是否达到上限或 Page 中的 Cluster 数量是否达到上限
	// Page 的 GPU Size(单位是 Byte) 上限: Root Page 是 NANITE_ROOT_PAGE_GPU_SIZE(2^15), Streaming Page 是 NANITE_STREAMING_PAGE_GPU_SIZE(2^17)
	// Page 中 Cluster 数量上限: Root Page 是 NANITE_ROOT_PAGE_MAX_CLUSTERS(2^6), Streaming Page 是 NANITE_STREAMING_PAGE_MAX_CLUSTERS(2^8)
	if (UpdatedPage.GpuSizes.GetTotal() <= (bRootPage ? NANITE_ROOT_PAGE_GPU_SIZE : NANITE_STREAMING_PAGE_GPU_SIZE) &&
		UpdatedPage.NumClusters <= (bRootPage ? NANITE_ROOT_PAGE_MAX_CLUSTERS : NANITE_STREAMING_PAGE_MAX_CLUSTERS))
	{
		// 成功添加才赋值当前 Page
		Page = UpdatedPage;
		return true;
	}

	return false;
}

static void AssignClustersToPages(
	FClusterDAG& ClusterDAG,
	const TArray<FEncodingInfo>& EncodingInfos,
	TArray<FPage>& Pages,
	TArray<FClusterGroupPart>& Parts,
	TArray<FClusterGroupPartInstance>& PartInstances,
	const uint32 MaxRootPages,
	FBoxSphereBounds3f& OutFinalBounds
	)
{
	check(Pages.Num() == 0);
	check(Parts.Num() == 0);
	check(PartInstances.Num() == 0);

	TArray<FCluster>& Clusters = ClusterDAG.Clusters;
	TArray<FClusterGroup>& ClusterGroups = ClusterDAG.Groups;

	// 初始化, 创建第一个 Page, 保证至少一个 Page 存在
	const uint32 NumClusterGroups = ClusterGroups.Num();
	Pages.AddDefaulted();

	// 先对 Group 内 Cluster 排序, 让 Group 内 Cluster 顺序更稳定
	SortGroupClusters(ClusterGroups, Clusters);

	// 再对 Group 排序, 决定填 Page 的顺序
	TArray<uint32> ClusterGroupPermutation = CalculateClusterGroupPermutation(ClusterGroups);

	OutFinalBounds.Origin		= ClusterDAG.TotalBounds.GetCenter();
	OutFinalBounds.BoxExtent	= ClusterDAG.TotalBounds.GetExtent();
	OutFinalBounds.SphereRadius = 0.0f;

	// 按顺序逐 Group 填 Page:
	// 遍历排好序的 ClusterGroupPermutation
	for (uint32 i = 0; i < NumClusterGroups; i++)
	{
		// Pick best next group			// TODO
		uint32 GroupIndex = ClusterGroupPermutation[i];
		FClusterGroup& Group = ClusterGroups[GroupIndex];
		if( Group.bTrimmed )
			continue;

		uint32 GroupStartPage = MAX_uint32;
	
		// 对 Group 内每个 Cluster
		for (uint32 ClusterIndex : Group.Children)
		{
			// Pick best next cluster		// TODO
			FCluster& Cluster = Clusters[ClusterIndex];
			const FEncodingInfo& EncodingInfo = EncodingInfos[ClusterIndex];

			// Add to page
			// 当前 Page
			FPage* Page = &Pages.Top();
			// 当前 Page 是否是 Root Page
			bool bRootPage =  (Pages.Num() - 1u) < MaxRootPages;

			// Try adding cluster to current page
			// 尝试将 Cluster 填入当前 Page
			if (!TryAddClusterToPage(*Page, Cluster, EncodingInfo, bRootPage))
			{
				// Page is full. Start a new page.
				// 如果失败, 则新开一个 Page
				Pages.AddDefaulted();
				Page = &Pages.Top();

				bool bResult = TryAddClusterToPage(*Page, Cluster, EncodingInfo, bRootPage);
				check(bResult);
			}

			// 因为是逐 Group填 Page, 并且只根据 Page GPU Size 大小和 Page 中 Cluster 数量断是否要新开一个 Page, 这意味着：
			// 1. 一个 Page 中可能会有属于不同 Group 的 Cluster
			// 2. 一个 Group 可能跨多个 Page
			// 不管上面哪种情况, 都会按照 Cluster 所属的 Group 将其划分为多个 GroupPart

			// 每个 GroupPart 记录:
			// 1. 包含的 Cluster 属于哪个 Group
			// 2. 属于哪个 Page
			// 3. 在 Page 中的 Cluster Offset (Page->NumClusters - 1)
			// 4. 包含哪些 Cluster (记录 Cluster Indexes)
			
			// Start a new part?
			if (Page->PartsNum == 0 || Parts[Page->PartsStartIndex + Page->PartsNum - 1].GroupIndex != GroupIndex)
			{
				if (Page->PartsNum == 0)
				{
					Page->PartsStartIndex = Parts.Num();
				}
				Page->PartsNum++;

				FClusterGroupPart& Part = Parts.AddDefaulted_GetRef();
				Part.GroupIndex = GroupIndex;
			}

			// Add cluster to page
			uint32 PageIndex = Pages.Num() - 1;
			uint32 PartIndex = Parts.Num() - 1;

			FClusterGroupPart& Part = Parts.Last();
			if (Part.Clusters.Num() == 0)
			{
				Part.PageClusterOffset = Page->NumClusters - 1;
				Part.PageIndex = PageIndex;
			}
			Part.Clusters.Add(ClusterIndex);
			check(Part.Clusters.Num() <= NANITE_MAX_CLUSTERS_PER_GROUP);

			Cluster.GroupPartIndex = PartIndex;
			
			if (GroupStartPage == MAX_uint32)
			{
				GroupStartPage = PageIndex;
			}
		}

		// 每个 Group 跨越的 Page范围:
		// Group 的起始 PageIndex
		Group.PageIndexStart = GroupStartPage;
		// 以及此 Group 一共跨了多少 Page(一个 Group 中的 Cluster 可能会填入多个 Page)
		Group.PageIndexNum = Pages.Num() - GroupStartPage;
		check(Group.PageIndexNum >= 1);
		check(Group.PageIndexNum <= NANITE_MAX_GROUP_PARTS_MASK);
	}

	// 生成 PartInstances
	// Generate group part instances and calculate their bounds
	uint32 ClusterGroupPartIndex = 0;
	// 遍历所有 GroupPart
	for (FClusterGroupPart& Part : Parts)
	{
		check(Part.Clusters.Num() <= NANITE_MAX_CLUSTERS_PER_GROUP);
		check(Part.PageIndex < (uint32)Pages.Num());

		Part.FirstInstanceIndex = PartInstances.Num();
		Part.NumInstances = 0;

		const FClusterGroup& Group = ClusterGroups[Part.GroupIndex];
		if (Group.AssemblyPartIndex == INDEX_NONE)  // 非 instanced mesh, 直接生成 PartInstance
		{
			// 合并 Cluster 的 AABB
			FBounds3f Bounds;
			for (uint32 ClusterIndex : Part.Clusters)
			{
				Bounds += Clusters[ClusterIndex].Bounds;

				const FSphere3f SphereBounds = Clusters[ClusterIndex].SphereBounds;
				const float Radius = (SphereBounds.Center - OutFinalBounds.Origin).Length() + SphereBounds.W;
				OutFinalBounds.SphereRadius = FMath::Max(OutFinalBounds.SphereRadius, Radius);
			}

			// 添加此 GroupPart 到 PartInstances, 包含:
			// 1. 此 GroupPart 在 PartInstances 中的位置 Index
			// 2. 此 GroupPart 的 Bounds
			PartInstances.Add(
				{
					.PartIndex = ClusterGroupPartIndex,
					.AssemblyTransformIndex = MAX_uint32,
					.Bounds = Bounds
				}
			);
			++Part.NumInstances;
		}
		else	// instanced mesh, 生成多个 PartInstance
		{
			
			const FAssemblyPartData& AssemblyPart = ClusterDAG.AssemblyPartData[Group.AssemblyPartIndex];
			for (uint32 TransformIndex = 0; TransformIndex < AssemblyPart.NumTransforms; ++TransformIndex)
			{
				// Calculate the bounds of all clusters in their instanced location
				const uint32 AssemblyTransformIndex = AssemblyPart.FirstTransform + TransformIndex;
				const FMatrix44f& Transform = ClusterDAG.AssemblyTransforms[AssemblyTransformIndex];
				const FVector3f AbsBasisX = FVector3f(Transform.M[0][0], Transform.M[0][1], Transform.M[0][2]).GetAbs();
				const FVector3f AbsBasisY = FVector3f(Transform.M[1][0], Transform.M[1][1], Transform.M[1][2]).GetAbs();
				const FVector3f AbsBasisZ = FVector3f(Transform.M[2][0], Transform.M[2][1], Transform.M[2][2]).GetAbs();

				// 合并 Cluster 的 AABB
				FBounds3f Bounds;
				for (uint32 ClusterIndex : Part.Clusters)
				{
					Bounds += Clusters[ClusterIndex].Bounds;
					
					FSphere3f SphereBounds = Clusters[ClusterIndex].SphereBounds.TransformBy(Transform);
					const float Radius = (SphereBounds.Center - OutFinalBounds.Origin).Length() + SphereBounds.W;
					OutFinalBounds.SphereRadius = FMath::Max(OutFinalBounds.SphereRadius, Radius);
				}

				const FVector3f Center = Transform.TransformPosition(FVector3f(Bounds.GetCenter()));
				
				FVector3f Extent = Bounds.GetExtent();
				Extent = Extent.X * AbsBasisX + Extent.Y * AbsBasisY + Extent.Z * AbsBasisZ;

				Bounds.Min = FVector4f(Center - Extent, 0.0f);
				Bounds.Max = FVector4f(Center + Extent, 0.0f);

				PartInstances.Add(
					{
						.PartIndex = ClusterGroupPartIndex,
						.AssemblyTransformIndex = AssemblyTransformIndex,
						.Bounds = Bounds
					}
				);
				++Part.NumInstances;	// 累计 PartInstance 数量
			}
		}

		++ClusterGroupPartIndex;
	}
}
```

**总结**：

把 Cluster 按 Group 顺序打包进 GPU Page, 同时建立： `Group <-> GroupPart <-> Page <-> Cluster` 的完整映射关系，并生成 PartInstance 和 Bounds 供 Streaming 和 Rendering 使用

下面是完整结构图，助于理解：

```
Pages
 ├── Page 0
 │	├── GroupPart 0
 │	│	 ├── Cluster 0
 │	│	 └── Cluster 1
 │	│
 │	└── GroupPart 1
 │		  └── Cluster 2
 │
 └── Page 1
	  └── GroupPart 2
			├── Cluster 3
			└── Cluster 4
```

```
Group
 ├── GroupPart 0
 ├── GroupPart 1
 └── GroupPart 2
```

```
GroupPart
 └── PartInstances
```

### 6.8. 构建层次结构 BVH

```cpp
static void BuildHierarchies(
	FResources& Resources,
	TArray<FPage>& Pages,
	const TArray<FClusterGroup>& Groups,
	const TArray<FClusterGroupPart>& Parts,
	TArray<FClusterGroupPartInstance>& PartInstances,
	const TArray<FMatrix44f>& AssemblyTransforms,
	uint32 NumMeshes)
{
	TArray<TArray<uint32>> PartInstancesByMesh;
	PartInstancesByMesh.SetNum(NumMeshes);

	// Assign group part instances to the meshes they belong to
	// 按 Mesh 分组 PartInstance
	const uint32 NumTotalPartInstances = PartInstances.Num();
	for (uint32 PartInstanceIndex = 0; PartInstanceIndex < NumTotalPartInstances; PartInstanceIndex++)
	{
		const FClusterGroupPartInstance& PartInstance = PartInstances[PartInstanceIndex];
		const FClusterGroupPart& Part = Parts[PartInstance.PartIndex];
		const FClusterGroup& Group = Groups[Part.GroupIndex];
		PartInstancesByMesh[Group.MeshIndex].Add(PartInstanceIndex);
	}

	for (uint32 MeshIndex = 0; MeshIndex < NumMeshes; MeshIndex++)
	{
		const TArray<uint32>& PartInstanceIndices = PartInstancesByMesh[MeshIndex];
		const uint32 NumPartInstances = PartInstanceIndices.Num();
		
		// 计算每个 Mesh 的最大 MipLevel
		int32 MaxMipLevel = 0;
		for (uint32 i = 0; i < NumPartInstances; i++)
		{
			const FClusterGroupPartInstance& PartInstance = PartInstances[PartInstanceIndices[i]];
			const FClusterGroupPart& Part = Parts[PartInstance.PartIndex];
			const FClusterGroup& Group = Groups[Part.GroupIndex];
			MaxMipLevel = FMath::Max(MaxMipLevel, Group.MipLevel);
		}

		TArray< FIntermediateNode >	Nodes;
		Nodes.SetNum(NumPartInstances);

		// Build leaf nodes for each LOD level of the mesh
		// 创建 Leaf Nodes, 并且按照 MipLevel 分类
		TArray<TArray<uint32>> NodesByMip;
		NodesByMip.SetNum(MaxMipLevel + 1);
		for (uint32 i = 0; i < NumPartInstances; i++)
		{
			const uint32 PartInstanceIndex = PartInstanceIndices[i];
			const FClusterGroupPartInstance& PartInstance = PartInstances[PartInstanceIndex];
			const FClusterGroupPart& Part = Parts[PartInstance.PartIndex];
			const FClusterGroup& Group = Groups[Part.GroupIndex];

			const int32 MipLevel = Group.MipLevel;
			FIntermediateNode& Node = Nodes[i];
			Node.Bound = PartInstance.Bounds;
			Node.PartInstanceIndex = PartInstanceIndex;
			Node.AssemblyTransformIndex = PartInstance.AssemblyTransformIndex;
			Node.MipLevel = Group.MipLevel;
			Node.bLeaf = true;
			NodesByMip[Group.MipLevel].Add(i);  // 按照 MipLevel 分类
		}

		uint32 RootIndex = 0;
		// 特殊情况: Mesh 为空, 创建 dummy root
		if (Nodes.Num() == 0)
		{
			// Completely empty mesh. This can happen for submeshes of existing geometry collections. 
			// The caller expects the submesh to have a valid hierarchy offset, so we provide an empty node with no children.
			Nodes.AddDefaulted();
		}
		else if (Nodes.Num() == 1)  // 特殊情况: Mesh 只有 1 个 PartInstance
		{
			// Just a single leaf.
			// Needs to be special-cased as root should always be an inner node.
			FIntermediateNode& Node = Nodes.AddDefaulted_GetRef();
			Node.Children.Add(0);
			Node.Bound = Nodes[0].Bound;
			RootIndex = 1;
		}
		else
		{
			// Build hierarchy:
			// Nanite meshes contain cluster data for many levels of detail. Clusters from different levels
			// of detail can vary wildly in size, which can already be challenge for building a good hierarchy. 
			// Apart from the visibility bounds, the hierarchy also tracks conservative LOD error metrics for the child nodes.
			// The runtime traversal descends into children as long as they are visible and the conservative LOD error is not
			// more detailed than what we are looking for. We have to be very careful when mixing clusters from different LODs
			// as less detailed clusters can easily end up bloating both bounds and error metrics.

			// We have experimented with a bunch of mixed LOD approached, but currently, it seems, building separate hierarchies
			// for each LOD level and then building a hierarchy of those hierarchies gives the best and most predictable results.

			// TODO: The roots of these hierarchies all share the same visibility and LOD bounds, or at least close enough that we could
			//	   make a shared conservative bound without losing much. This makes a lot of the work around the root node fairly
			//	   redundant. Perhaps we should consider evaluating a shared root during instance cull instead and enable/disable
			//	   the per-level hierarchies based on 1D range tests for LOD error.

			// 为每个 MipLevel 单独构建 BVH
			// LevelRoots 最终存储的是每个 MipLevel BVH 的 Root
			TArray<uint32> LevelRoots;
			for (int32 MipLevel = 0; MipLevel <= MaxMipLevel; MipLevel++)
			{
				if (NodesByMip[MipLevel].Num() > 0)
				{
					// Build a hierarchy for the mip level
					uint32 NodeIndex = BuildHierarchyTopDown(Nodes, NodesByMip[MipLevel], true);

					if (Nodes[NodeIndex].bLeaf || Nodes[NodeIndex].Children.Num() == NANITE_MAX_BVH_NODE_FANOUT)
					{
						// Leaf or filled node. Just add it.
						LevelRoots.Add(NodeIndex);
					}
					else
					{
						// Incomplete node. Discard the code and add the children as roots instead.
						LevelRoots.Append(Nodes[NodeIndex].Children);
					}
				}
			}
			// Build top hierarchy. A hierarchy of MIP hierarchies.
			// 把 Mip BVH Root 再建 BVH
			RootIndex = BuildHierarchyTopDown(Nodes, LevelRoots, false);
		}

		check(Nodes.Num() > 0);

#if BVH_BUILD_WRITE_GRAPHVIZ
		WriteDotGraph(Nodes);
#endif

		// 转换为 GPU 格式 HierarchyNodes
		TArray< FHierarchyNode > HierarchyNodes;
		BuildHierarchyRecursive(Pages, HierarchyNodes, Nodes, Groups, Parts, PartInstances, AssemblyTransforms, RootIndex, 0);

		// Convert hierarchy to packed format
		const uint32 NumHierarchyNodes = HierarchyNodes.Num();
		const uint32 PackedBaseIndex = Resources.HierarchyNodes.Num();
		Resources.HierarchyRootOffsets.Add(PackedBaseIndex);
		Resources.HierarchyNodes.AddDefaulted(NumHierarchyNodes);
		for (uint32 i = 0; i < NumHierarchyNodes; i++)
		{
			PackHierarchyNode(Resources.HierarchyNodes[PackedBaseIndex + i], HierarchyNodes[i], Groups, Parts, PartInstances, Resources.NumRootPages);
		}
	}
}
```
**总结**：

```
Cluster Instances
	↓
按 Mesh 分组
	↓
每个 Cluster → 一个 BVH Leaf Node
	↓
按 MipLevel 分组
	↓
每个 MipLevel 单独建 BVH
	↓
再把各 MipLevel 的 Root 建一个总 BVH（BVH of BVHs）
	↓
转换为 GPU buffer
```

### 6.9. 写入 Page 数据

```cpp
static TArray<TMap<FVariableVertex, FVertexMapEntry>> BuildVertexMaps(const TArray<FPage>& Pages, const TArray<FCluster>& Clusters, const TArray<FClusterGroupPart>& Parts)
{
	TArray<TMap<FVariableVertex, FVertexMapEntry>> VertexMaps;
	VertexMaps.SetNum(Pages.Num());

	// 遍历每个 Page
	ParallelFor( TEXT("NaniteEncode.BuildVertexMaps.PF"), Pages.Num(), 1, [&VertexMaps, &Pages, &Clusters, &Parts](int32 PageIndex)
	{
		const FPage& Page = Pages[PageIndex];
		ProcessPageClusters(Page, Parts, [&](uint32 LocalClusterIndex, uint32 ClusterIndex)
		{
			// Page 内所有 Cluster
			const FCluster& Cluster = Clusters[ClusterIndex];

			// Cluster 的所有顶点
			for (uint32 VertexIndex = 0; VertexIndex < Cluster.NumVerts; VertexIndex++)
			{
				// Key: 顶点字节码
				FVariableVertex Vertex;
				Vertex.Data = &Cluster.Verts[VertexIndex * Cluster.GetVertSize()];
				Vertex.SizeInBytes = Cluster.GetVertSize() * sizeof(float);

				// Value: { LocalClusterIndex, VertexIndex }
				FVertexMapEntry Entry;
				Entry.LocalClusterIndex = LocalClusterIndex;
				Entry.VertexIndex = VertexIndex;

				// 添加进 TMap<FVariableVertex, FVertexMapEntry> 中, 用于后续 "跨 Page 顶点去重"
				VertexMaps[PageIndex].Add(Vertex, Entry);
			}
		});
	});
	return VertexMaps;
}

static uint32 MarkRelativeEncodingPagesRecursive(TArray<FPage>& Pages, TArray<uint32>& PageDependentsDepth, const TArray<TArray<uint32>>& PageDependents, uint32 PageIndex)
{
	if (PageDependentsDepth[PageIndex] != MAX_uint32)
	{
		return PageDependentsDepth[PageIndex];
	}

	uint32 Depth = 0;
	for (const uint32 DependentPageIndex : PageDependents[PageIndex])
	{
		const uint32 DependentDepth = MarkRelativeEncodingPagesRecursive(Pages, PageDependentsDepth, PageDependents, DependentPageIndex);
		Depth = FMath::Max(Depth, DependentDepth + 1u);
	}

	FPage& Page = Pages[PageIndex];
	Page.bRelativeEncoding = true;

	if (Depth >= MAX_DEPENDENCY_CHAIN_FOR_RELATIVE_ENCODING)
	{
		// Using relative encoding for this page would make the dependency chain too long. Use direct coding instead and reset depth.
		Page.bRelativeEncoding = false;
		Depth = 0;
	}
	
	PageDependentsDepth[PageIndex] = Depth;
	return Depth;
}

static uint32 MarkRelativeEncodingPages(const FResources& Resources, TArray<FPage>& Pages, const TArray<FClusterGroup>& Groups)
{
	const uint32 NumPages = Resources.PageStreamingStates.Num();

	// Build list of dependents for each page
	TArray<TArray<uint32>> PageDependents;
	PageDependents.SetNum(NumPages);

	// Memorize how many levels of dependency a given page has
	TArray<uint32> PageDependentsDepth;
	PageDependentsDepth.Init(MAX_uint32, NumPages);

	TBitArray<> PageHasOnlyRootDependencies(false, NumPages);

	for (uint32 PageIndex = 0; PageIndex < NumPages; PageIndex++)
	{
		const FPageStreamingState& PageStreamingState = Resources.PageStreamingStates[PageIndex];

		bool bHasRootDependency = false;
		bool bHasStreamingDependency = false;
		for (uint32 i = 0; i < PageStreamingState.DependenciesNum; i++)
		{
			const uint32 DependencyPageIndex = Resources.PageDependencies[PageStreamingState.DependenciesStart + i];
			if (Resources.IsRootPage(DependencyPageIndex))
			{
				bHasRootDependency = true;
			}
			else
			{
				PageDependents[DependencyPageIndex].AddUnique(PageIndex);
				bHasStreamingDependency = true;
			}
		}

		PageHasOnlyRootDependencies[PageIndex] = (bHasRootDependency && !bHasStreamingDependency);
	}

	uint32 NumRelativeEncodingPages = 0;
	for (uint32 PageIndex = 0; PageIndex < NumPages; PageIndex++)
	{
		FPage& Page = Pages[PageIndex];

		MarkRelativeEncodingPagesRecursive(Pages, PageDependentsDepth, PageDependents, PageIndex);
		
		if (Resources.IsRootPage(PageIndex))
		{
			// Root pages never use relative encoding
			Page.bRelativeEncoding = false;
		}
		else if (PageHasOnlyRootDependencies[PageIndex])
		{
			// Root pages are always resident, so dependencies on them shouldn't count towards dependency chain limit.
			// If a page only has root dependencies, always code it as relative.
			Page.bRelativeEncoding = true;
		}

		if (Page.bRelativeEncoding)
		{
			NumRelativeEncodingPages++;
		}
	}

	return NumRelativeEncodingPages;
}

static void WritePages(
	FResources& Resources,
	TArray<FPage>& Pages,
	const TArray<FClusterGroup>& Groups,
	const TArray<FClusterGroupPart>& Parts,
	const TArray<FClusterGroupPartInstance>& PartInstances,
	TArray<FCluster>& Clusters,
	const TArray<FEncodingInfo>& EncodingInfos,
	const bool bHasSkinning,
	uint32* OutTotalGPUSize)
{
	check(Resources.PageStreamingStates.Num() == 0);

	TArray< uint8 > StreamableBulkData;
	
	// 为每个 page 分配 streaming state, 用于运行时 streaming
	const uint32 NumPages = Pages.Num();
	Resources.PageStreamingStates.SetNum(NumPages);

	// Add external fixups to pages
	TArray<TArray<FClusterFixup>> ClusterFixupsPerPage;
	ClusterFixupsPerPage.SetNum(NumPages);

	// 遍历所有 GroupPart 中的 Cluster
	for (const FClusterGroupPart& Part : Parts)
	{
		check(Part.PageIndex < NumPages);

		const FClusterGroup& Group = Groups[Part.GroupIndex];
		check(!Group.bTrimmed);
		for (uint32 ClusterPositionInPart = 0; ClusterPositionInPart < (uint32)Part.Clusters.Num(); ClusterPositionInPart++)
		{
			const FCluster& Cluster = Clusters[Part.Clusters[ClusterPositionInPart]];
			// 如果该 Cluster 是通过 Group 生成(也就是 ReduceGroup ), 也意味着这是一个 **非叶子 Cluster**
			// 说明它有依赖的 Group
			if (Cluster.GeneratingGroupIndex != MAX_uint32)
			{
				const FClusterGroup& GeneratingGroup = Groups[Cluster.GeneratingGroupIndex];
				check(!GeneratingGroup.bTrimmed);
				check(GeneratingGroup.PageIndexNum >= 1);
				
				// Group 所处的 Page 范围
				uint32 PageDependencyStart = GeneratingGroup.PageIndexStart;
				uint32 PageDependencyNum = GeneratingGroup.PageIndexNum;

				// 去掉 Root Page, Root Page 常驻 GPU
				RemoveRootPagesFromRange(PageDependencyStart, PageDependencyNum, Resources.NumRootPages);
				// 去掉自己的 Page, 自己的 Page 不应该当作依赖 Page
				RemovePageFromRange(PageDependencyStart, PageDependencyNum, Part.PageIndex);
				
				// 移除 Root Page 和自己的 Page 之后, 不存在依赖 Page 了, 跳过当前 Cluster
				if (PageDependencyNum == 0)
					continue;	// Dependencies already met by current page and/or root pages
				
				// 构建 FClusterFixup, FClusterFixup 包含:
				// 1. Cluster 所在的 Page Index
				// 2. Cluster Index
				// 3. Cluster 所依赖的 Group 跨越的 Page 范围
				const FClusterFixup ClusterFixup = FClusterFixup(Part.PageIndex, Part.PageClusterOffset + ClusterPositionInPart, PageDependencyStart, PageDependencyNum);

				// 填充 ClusterFixupsPerPage, Key: PageIndex , Value: FClusterFixup 列表
				for (uint32 i = 0; i < GeneratingGroup.PageIndexNum; i++)
				{
					//TODO: Implement some sort of FFixupPart to not redundantly store PageIndexStart/PageIndexNum?
					ClusterFixupsPerPage[GeneratingGroup.PageIndexStart + i].Add(ClusterFixup);
				}
			}
		}
	}

	// 构建 FixupChunks, 记录每个 Page 中的所有 FClusterFixup
	uint32 NumReferencedClusters = 0;
	FFixupChunkBuffer FixupChunks;
	FixupChunks.Reserve(NumPages);
	for (uint32 PageIndex = 0; PageIndex < NumPages; PageIndex++)
	{
		const FPage& Page = Pages[PageIndex];
		NumReferencedClusters += Page.NumClusters;

		uint32 NumHierarchyFixups = 0;
		for (uint32 i = 0; i < Page.PartsNum; i++)
		{
			const FClusterGroupPart& Part = Parts[Page.PartsStartIndex + i];
			const FClusterGroup& Group = Groups[Part.GroupIndex];
			NumHierarchyFixups += Group.PageIndexNum * Part.NumInstances;
		}

		// Allocate fixup chunk and write cluster fixups
		const TArray<FClusterFixup>& ClusterFixups = ClusterFixupsPerPage[PageIndex];

		const uint32 NumClusterFixups = ClusterFixups.Num();
		FFixupChunk& FixupChunk = FixupChunks.Add_GetRef(Page.NumClusters, NumHierarchyFixups, NumClusterFixups);
		for (uint32 i = 0; i < NumClusterFixups; ++i)
		{
			FixupChunk.GetClusterFixup(i) = ClusterFixups[i];
		}
	}

	check(NumReferencedClusters <= (uint32)Clusters.Num());	// There can be unused clusters when trim is used
	Resources.NumClusters = NumReferencedClusters;

	// Generate page dependencies
	// 从 FixupChunks 反推每个 Page 的依赖列表
	for (uint32 PageIndex = 0; PageIndex < NumPages; PageIndex++)
	{
		// 每个 Page 的 FixupChunk
		const FFixupChunk& FixupChunk = FixupChunks[PageIndex];
		FPageStreamingState& PageStreamingState = Resources.PageStreamingStates[PageIndex];
		PageStreamingState.DependenciesStart = Resources.PageDependencies.Num();
		PageStreamingState.MaxHierarchyDepth = uint8(Pages[PageIndex].MaxHierarchyDepth);

		// 取 FixupChunk 中每个 Cluster 的 Page Index, 去重后写入 Resources.PageDependencies
		for (uint32 i = 0; i < FixupChunk.Header.NumClusterFixups; i++)
		{
			uint32 FixupPageIndex = FixupChunk.GetClusterFixup(i).GetPageIndex();
			check(FixupPageIndex < NumPages);
			if (FixupPageIndex == PageIndex)	// Never emit dependencies to ourselves
				continue;

			// Only add if not already in the set.
			// O(n^2), but number of dependencies should be tiny in practice.
			bool bFound = false;
			for (uint32 j = PageStreamingState.DependenciesStart; j < (uint32)Resources.PageDependencies.Num(); j++)
			{
				if (Resources.PageDependencies[j] == FixupPageIndex)
				{
					bFound = true;
					break;
				}
			}

			if (bFound)
				continue;

			Resources.PageDependencies.Add(FixupPageIndex);
		}
		// PageStreamingState.DependenciesNum 记录索引范围
		PageStreamingState.DependenciesNum = uint16(Resources.PageDependencies.Num() - PageStreamingState.DependenciesStart);
	}

	// 构建 Page 的顶点查找表
	auto PageVertexMaps = BuildVertexMaps(Pages, Clusters, Parts);

	const uint32 NumRelativeEncodingPages = MarkRelativeEncodingPages(Resources, Pages, Groups);
	
	// Process pages
	TArray< TArray<uint8> > PageResults;
	PageResults.SetNum(NumPages);

	// 并行编码每个 Page
	ParallelFor(TEXT("NaniteEncode.BuildPages.PF"), NumPages, 1, [&Resources, &Pages, &Groups, &Parts, &PartInstances, &Clusters, &EncodingInfos, &FixupChunks, &PageVertexMaps, &PageResults, bHasSkinning](int32 PageIndex)
	{
		const FPage& Page = Pages[PageIndex];
		FFixupChunk& FixupChunk = FixupChunks[PageIndex];

		Resources.PageStreamingStates[PageIndex].Flags = Page.bRelativeEncoding ? NANITE_PAGE_FLAG_RELATIVE_ENCODING : 0;

		// Add hierarchy fixups
		{
			// Parts include the hierarchy fixups for all the other parts of the same group.
			uint32 NumHierarchyFixups = 0;
			// 对 Page 内每个 GroupPart
			for (uint32 i = 0; i < Page.PartsNum; i++)
			{
				const FClusterGroupPart& Part = Parts[Page.PartsStartIndex + i];
				const FClusterGroup& Group = Groups[Part.GroupIndex];
				const uint32 HierarchyRootOffset = Resources.HierarchyRootOffsets[Group.MeshIndex];

				// 找到该 Group 跨越的 Page 范围
				uint32 PageDependencyStart = Group.PageIndexStart;
				uint32 PageDependencyNum = Group.PageIndexNum;

				// 移除 Root Page
				RemoveRootPagesFromRange(PageDependencyStart, PageDependencyNum, Resources.NumRootPages);

				// Add fixups to all part instances of the group
				for (uint32 j = 0; j < Group.PageIndexNum; j++)
				{
					// 该 Group 跨越的每个 Page
					const FPage& Page2 = Pages[Group.PageIndexStart + j];

					// 遍历跨越的每个 Page 的 GroupPart
					for (uint32 k = 0; k < Page2.PartsNum; k++)
					{
						const FClusterGroupPart& Part2 = Parts[Page2.PartsStartIndex + k];
						// 如果是同一个 Group
						if (Part2.GroupIndex == Part.GroupIndex)
						{
							for (uint32 InstanceIndex = 0; InstanceIndex < Part2.NumInstances; ++InstanceIndex)
							{
								// TODO???
								// 对跨越的 Page 的 GroupPart 的每个 PartInstance 写入 FHierarchyFixup, 这些 fixup 用于 runtime 时把 hierarchy leaf 指向正确的 Page Cluster Offset 位置
								const FClusterGroupPartInstance& PartInstance = PartInstances[Part2.FirstInstanceIndex + InstanceIndex];
								const uint32 GlobalHierarchyNodeIndex = HierarchyRootOffset + PartInstance.HierarchyNodeIndex;
								FixupChunk.GetHierarchyFixup(NumHierarchyFixups++) = FHierarchyFixup(Part2.PageIndex, GlobalHierarchyNodeIndex, PartInstance.HierarchyChildIndex, Part2.PageClusterOffset, PageDependencyStart, PageDependencyNum);
							}
							break;
						}
					}
				}
			}
			check(NumHierarchyFixups == FixupChunk.Header.NumHierarchyFixups);
		}

		// Pack clusters and generate material range data
		TArray<uint32>				CombinedStripBitmaskData;
		TArray<uint32>				CombinedPageClusterPairData;
		TArray<uint32>				CombinedVertexRefBitmaskData;
		TArray<uint16>				CombinedVertexRefData;
		TArray<uint8>				CombinedIndexData;
		TArray<uint8>				CombinedAttributeData;
		TArray<uint8>				BoneInfluenceData;
		TArray<uint8>				BrickData;
		TArray<uint32>				ExtendedData;
		TArray<uint32>				MaterialRangeData;
		TArray<uint32>				VertReuseBatchInfo;
		TArray<uint16>				CodedVerticesPerCluster;
		TArray<uint32>				NumPageClusterPairsPerCluster;
		TArray<FPackedCluster>		PackedClusters;
		TArray<FPackedBoneInfluenceHeader>	PackedBoneInfluenceHeaders;

		TArray<uint8>				LowByteStream;
		TArray<uint8>				MidByteStream;
		TArray<uint8>				HighByteStream;

		struct FByteStreamCounters
		{
			uint32 Low = 0;
			uint32 Mid = 0;
			uint32 High = 0;
		};

		TArray<FByteStreamCounters> ByteStreamCounters;
		ByteStreamCounters.SetNumUninitialized(Page.NumClusters);

		PackedClusters.SetNumUninitialized(Page.NumClusters);
		CodedVerticesPerCluster.SetNumUninitialized(Page.NumClusters);
		NumPageClusterPairsPerCluster.SetNumUninitialized(Page.NumClusters);

		if(bHasSkinning)
		{
			PackedBoneInfluenceHeaders.SetNumUninitialized(Page.NumClusters);
		}
		
		check((Page.GpuSizes.GetMaterialTableOffset() & 3) == 0);
		const uint32 MaterialTableStartOffsetInDwords = Page.GpuSizes.GetMaterialTableOffset() >> 2;

		FPageSections GpuSectionOffsets = Page.GpuSizes.GetOffsets();
		TMap<FVariableVertex, uint32> UniqueVertices;

		ProcessPageClusters(Page, Parts, [&](uint32 LocalClusterIndex, uint32 ClusterIndex)
		{
			const FCluster& Cluster = Clusters[ClusterIndex];
			const FEncodingInfo& EncodingInfo = EncodingInfos[ClusterIndex];

			FPackedCluster& PackedCluster = PackedClusters[LocalClusterIndex];
			PackCluster(PackedCluster, Cluster, EncodingInfos[ClusterIndex], Cluster.VertexFormat.bHasTangents, Cluster.VertexFormat.NumTexCoords);

			check((GpuSectionOffsets.Index & 3) == 0);
			check((GpuSectionOffsets.Position & 3) == 0);
			check((GpuSectionOffsets.Attribute & 3) == 0);
			PackedCluster.SetIndexOffset(GpuSectionOffsets.Index);
			PackedCluster.SetPositionOffset(GpuSectionOffsets.Position);
			PackedCluster.SetAttributeOffset(GpuSectionOffsets.Attribute);
			PackedCluster.SetDecodeInfoOffset(GpuSectionOffsets.DecodeInfo);
			PackedCluster.SetHasSkinning(bHasSkinning);

			if(bHasSkinning)
			{
				FPackedBoneInfluenceHeader& PackedBoneInfluenceHeader = PackedBoneInfluenceHeaders[LocalClusterIndex];
				PackBoneInfluenceHeader(PackedBoneInfluenceHeader, EncodingInfo.BoneInfluence);
				check((GpuSectionOffsets.BoneInfluence & 3) == 0);
				PackedBoneInfluenceHeader.SetDataOffset(GpuSectionOffsets.BoneInfluence);
			}

			if( Cluster.Bricks.Num() > 0 )
			{
				PackedCluster.SetBrickDataOffset( GpuSectionOffsets.BrickData );
				PackedCluster.SetBrickDataNum( Cluster.Bricks.Num() );

				for( const FCluster::FBrick& Brick : Cluster.Bricks )
				{
					FPackedBrick PackedBrick;
					PackBrick(PackedBrick, Brick);
					BrickData.Append( (uint8*)&PackedBrick, sizeof(PackedBrick));
				}
			}


			// No effect if unused
			if( Cluster.ExtendedData.Num() > 0 )
			{
				PackedCluster.SetExtendedDataOffset( GpuSectionOffsets.ExtendedData );
				PackedCluster.SetExtendedDataNum( Cluster.ExtendedData.Num() );
				ExtendedData.Append( Cluster.ExtendedData );
			}

			PackedCluster.PackedMaterialInfo = PackMaterialInfo(Cluster, MaterialRangeData, MaterialTableStartOffsetInDwords);
				
			if( Cluster.NumTris )
			{
				TArray<uint32> LocalVertReuseBatchInfo;
				PackVertReuseBatchInfo(MakeArrayView(Cluster.MaterialRanges), LocalVertReuseBatchInfo);
	
				PackedCluster.SetVertResourceBatchInfo(LocalVertReuseBatchInfo, GpuSectionOffsets.VertReuseBatchInfo, Cluster.MaterialRanges.Num());
				if (Cluster.MaterialRanges.Num() > 3)
				{
					VertReuseBatchInfo.Append(MoveTemp(LocalVertReuseBatchInfo));
				}
			}
				
			GpuSectionOffsets += EncodingInfo.GpuSizes;

			const uint32 PrevLow = LowByteStream.Num();
			const uint32 PrevMid = MidByteStream.Num();
			const uint32 PrevHigh = HighByteStream.Num();

			const FPageStreamingState& PageStreamingState = Resources.PageStreamingStates[PageIndex];
			const uint32 DependenciesNum = (PageStreamingState.Flags & NANITE_PAGE_FLAG_RELATIVE_ENCODING) ? PageStreamingState.DependenciesNum : 0u;
			const TArrayView<uint32> PageDependencies = TArrayView<uint32>(Resources.PageDependencies.GetData() + PageStreamingState.DependenciesStart, DependenciesNum);
			const uint32 PrevPageClusterPairs = CombinedPageClusterPairData.Num();
			uint32 NumCodedVertices = 0;
			EncodeGeometryData(	LocalClusterIndex, Cluster, EncodingInfo, 
								CombinedStripBitmaskData, CombinedIndexData,
								CombinedPageClusterPairData, CombinedVertexRefBitmaskData, CombinedVertexRefData,
								LowByteStream, MidByteStream, HighByteStream,
								BoneInfluenceData,
								PageDependencies, PageVertexMaps,
								UniqueVertices, NumCodedVertices);

			ByteStreamCounters[LocalClusterIndex].Low	= LowByteStream.Num() - PrevLow;
			ByteStreamCounters[LocalClusterIndex].Mid	= MidByteStream.Num() - PrevMid;
			ByteStreamCounters[LocalClusterIndex].High	= HighByteStream.Num() - PrevHigh;

			NumPageClusterPairsPerCluster[LocalClusterIndex] = CombinedPageClusterPairData.Num() - PrevPageClusterPairs;
			CodedVerticesPerCluster[LocalClusterIndex] = uint16(NumCodedVertices);
		});
		check(GpuSectionOffsets.Cluster							== Page.GpuSizes.GetClusterBoneInfluenceOffset());
		check(Align(GpuSectionOffsets.MaterialTable, 16)		== Page.GpuSizes.GetVertReuseBatchInfoOffset());
		check(Align(GpuSectionOffsets.VertReuseBatchInfo, 16)	== Page.GpuSizes.GetBoneInfluenceOffset());
		check(Align(GpuSectionOffsets.BoneInfluence, 16)		== Page.GpuSizes.GetBrickDataOffset());
		check(Align(GpuSectionOffsets.BrickData, 16)			== Page.GpuSizes.GetExtendedDataOffset());
		check(Align(GpuSectionOffsets.ExtendedData, 16)			== Page.GpuSizes.GetDecodeInfoOffset());
		check(Align(GpuSectionOffsets.DecodeInfo, 16)			== Page.GpuSizes.GetIndexOffset());
		check(GpuSectionOffsets.Index							== Page.GpuSizes.GetPositionOffset());
		check(GpuSectionOffsets.Position						== Page.GpuSizes.GetAttributeOffset());
		check(GpuSectionOffsets.Attribute						== Page.GpuSizes.GetTotal());

		// Dword align index data
		CombinedIndexData.SetNumZeroed((CombinedIndexData.Num() + 3) & -4);

		// Perform page-internal fix up directly on PackedClusters
		for (uint32 LocalPartIndex = 0; LocalPartIndex < Page.PartsNum; LocalPartIndex++)
		{
			const FClusterGroupPart& Part = Parts[Page.PartsStartIndex + LocalPartIndex];
			const FClusterGroup& Group = Groups[Part.GroupIndex];
			
			bool bRootGroup = false;
			{
				uint32 PageDependencyStart = Group.PageIndexStart;
				uint32 PageDependencyNum = Group.PageIndexNum;
				RemoveRootPagesFromRange(PageDependencyStart, PageDependencyNum, Resources.NumRootPages);
				bRootGroup = (PageDependencyNum == 0);
			}
			
			for (uint32 ClusterPositionInPart = 0; ClusterPositionInPart < (uint32)Part.Clusters.Num(); ClusterPositionInPart++)
			{
				const FCluster& Cluster = Clusters[Part.Clusters[ClusterPositionInPart]];
				FPackedCluster& PackedCluster = PackedClusters[Part.PageClusterOffset + ClusterPositionInPart];
				uint32 ClusterFlags = PackedCluster.GetFlags();

				if (bRootGroup)
				{
					ClusterFlags |= NANITE_CLUSTER_FLAG_ROOT_GROUP;
				}

				if (Cluster.GeneratingGroupIndex != MAX_uint32)
				{
					const FClusterGroup& GeneratingGroup = Groups[Cluster.GeneratingGroupIndex];
					uint32 PageDependencyStart = GeneratingGroup.PageIndexStart;
					uint32 PageDependencyNum = GeneratingGroup.PageIndexNum;
					RemoveRootPagesFromRange(PageDependencyStart, PageDependencyNum, Resources.NumRootPages);
					if (PageDependencyNum == 0)
					{
						// Dependencies met by root pages
						ClusterFlags &= ~NANITE_CLUSTER_FLAG_ROOT_LEAF;
					}

					RemovePageFromRange(PageDependencyStart, PageDependencyNum, PageIndex);
					
					if (PageDependencyNum == 0)
					{
						// Dependencies met by current page and/or root pages
						ClusterFlags &= ~NANITE_CLUSTER_FLAG_STREAMING_LEAF;
					}
				}
				else
				{
					ClusterFlags |= NANITE_CLUSTER_FLAG_FULL_LEAF;
				}
				PackedCluster.SetFlags(ClusterFlags);
			}
		}

		// Begin page
		TArray<uint8>& PageResult = PageResults[PageIndex];
		PageResult.Reset(NANITE_ESTIMATED_MAX_PAGE_DISK_SIZE);

		FPageWriter PageWriter(PageResult);

		// Disk header
		const uint32 PageDiskHeaderOffset = PageWriter.Append_Offset<FPageDiskHeader>(1);

		// 16-byte align material range data to make it easy to copy during GPU transcoding
		MaterialRangeData.SetNum(Align(MaterialRangeData.Num(), 4));
		VertReuseBatchInfo.SetNum(Align(VertReuseBatchInfo.Num(), 4));
		BoneInfluenceData.SetNum(Align(BoneInfluenceData.Num(), 16));
		BrickData.SetNum(Align(BrickData.Num(), 16));
		ExtendedData.SetNum(Align(ExtendedData.Num(), 4));

		static_assert(sizeof(FPageGPUHeader) % 16 == 0, "sizeof(FGPUPageHeader) must be a multiple of 16");
		static_assert(sizeof(FPackedCluster) % 16 == 0, "sizeof(FPackedCluster) must be a multiple of 16");
		
		// Cluster headers
		const uint32 ClusterDiskHeadersOffset = PageWriter.Append_Offset<FClusterDiskHeader>(Page.NumClusters);
		TArray<FClusterDiskHeader> ClusterDiskHeaders;
		ClusterDiskHeaders.SetNum(Page.NumClusters);

		const uint32 RawFloat4StartOffset = PageWriter.Offset();
		{
			// GPU page header
			FPageGPUHeader& GPUPageHeader = *PageWriter.Append_Ptr<FPageGPUHeader>(1);
			GPUPageHeader = FPageGPUHeader();
			GPUPageHeader.SetNumClusters(Page.NumClusters);
			GPUPageHeader.SetMaxClusterBoneInfluences(Page.MaxClusterBoneInfluences);
			GPUPageHeader.SetMaxVoxelBoneInfluences(Page.MaxVoxelBoneInfluences);
		}

		// Write clusters in SOA layout
		{
			const uint32 NumClusterFloat4Properties = sizeof(FPackedCluster) / 16;
			uint8* Dst = PageWriter.Append_Ptr<uint8>(NumClusterFloat4Properties * 16 * PackedClusters.Num());
			for (uint32 float4Index = 0; float4Index < NumClusterFloat4Properties; float4Index++)
			{
				for (const FPackedCluster& PackedCluster : PackedClusters)
				{
					FMemory::Memcpy(Dst, (uint8*)&PackedCluster + float4Index * 16, 16);
					Dst += 16;
				}
			}
		}


		// Cluster bone data in SOA layout
		{
			const uint32 ClusterBoneInfluenceOffset = PageWriter.Offset();
			FClusterBoneInfluence* Ptr = PageWriter.Append_Ptr<FClusterBoneInfluence>(Page.NumClusters * Page.MaxClusterBoneInfluences);
			
			ProcessPageClusters(Page, Parts, [&](uint32 LocalClusterIndex, uint32 ClusterIndex)
			{
				const TArray<FClusterBoneInfluence>& ClusterBoneInfluences = EncodingInfos[ClusterIndex].BoneInfluence.ClusterBoneInfluences;

				const uint32 NumInfluences = FMath::Min((uint32)ClusterBoneInfluences.Num(), Page.MaxClusterBoneInfluences);
				for (uint32 i = 0; i < NumInfluences; i++)
				{
					Ptr[Page.NumClusters * i + LocalClusterIndex] = ClusterBoneInfluences[i];
				}
			});

			PageWriter.AlignRelativeToOffset(ClusterBoneInfluenceOffset, 16u);
			check(PageWriter.Offset() - ClusterBoneInfluenceOffset == Page.GpuSizes.GetClusterBoneInfluenceSize());
		}

		// Voxel bone data in SOA layout
		{
			const uint32 VoxelBoneInfluenceOffset = PageWriter.Offset();

			uint32* Ptr = PageWriter.Append_Ptr<uint32>(Page.NumClusters * Page.MaxVoxelBoneInfluences);
			
			ProcessPageClusters(Page, Parts, [&](uint32 LocalClusterIndex, uint32 ClusterIndex)
			{
				const TArray<FPackedVoxelBoneInfluence>& VoxelBoneInfluences = EncodingInfos[ClusterIndex].BoneInfluence.VoxelBoneInfluences;

				const uint32 NumInfluences = FMath::Min((uint32)VoxelBoneInfluences.Num(), Page.MaxVoxelBoneInfluences);
				for (uint32 k = 0; k < NumInfluences; k++)
				{
					Ptr[Page.NumClusters * k + LocalClusterIndex] = VoxelBoneInfluences[k].Weight_BoneIndex;
				}
			});
			
			PageWriter.AlignRelativeToOffset(VoxelBoneInfluenceOffset, 16u);
			check(PageWriter.Offset() - VoxelBoneInfluenceOffset == Page.GpuSizes.GetVoxelBoneInfluenceSize());
		}
		
		{
			// Material table
			uint32 MaterialTableSize = MaterialRangeData.Num() * MaterialRangeData.GetTypeSize();
			uint8* MaterialTable = PageWriter.Append_Ptr<uint8>(MaterialTableSize);
			FMemory::Memcpy(MaterialTable, MaterialRangeData.GetData(), MaterialTableSize);
			check(MaterialTableSize == Page.GpuSizes.GetMaterialTableSize());
		}

		{
			// Vert reuse batch info
			const uint32 VertReuseBatchInfoSize = VertReuseBatchInfo.Num() * VertReuseBatchInfo.GetTypeSize();
			uint8* VertReuseBatchInfoData = PageWriter.Append_Ptr<uint8>(VertReuseBatchInfoSize);
			FMemory::Memcpy(VertReuseBatchInfoData, VertReuseBatchInfo.GetData(), VertReuseBatchInfoSize);
			check(VertReuseBatchInfoSize == Page.GpuSizes.GetVertReuseBatchInfoSize());
		}

		{
			// Bone data
			const uint32 DataSize = BoneInfluenceData.Num() * BoneInfluenceData.GetTypeSize();
			uint8* Ptr = PageWriter.Append_Ptr<uint8>(DataSize);
			FMemory::Memcpy(Ptr, BoneInfluenceData.GetData(), DataSize);
			check(DataSize == Page.GpuSizes.GetBoneInfluenceSize());
		}

		{
			// Brick data
			uint32 BrickDataSize = BrickData.Num() * BrickData.GetTypeSize();
			uint8* BrickDataPtr = PageWriter.Append_Ptr<uint8>(BrickDataSize);
			FMemory::Memcpy(BrickDataPtr, BrickData.GetData(), BrickDataSize);
			check(BrickDataSize == Page.GpuSizes.GetBrickDataSize());
		}

		{
			// Extended data
			uint32 ExtendedDataSize = ExtendedData.Num() * ExtendedData.GetTypeSize();
			uint8* ExtendedDataPtr = PageWriter.Append_Ptr<uint8>(ExtendedDataSize);
			FMemory::Memcpy(ExtendedDataPtr, ExtendedData.GetData(), ExtendedDataSize);
			check(ExtendedDataSize == Page.GpuSizes.GetExtendedDataSize());
		}

		// Decode information
		const uint32 DecodeInfoOffset = PageWriter.Offset();
		ProcessPageClusters(Page, Parts, [&](uint32 LocalClusterIndex, uint32 ClusterIndex)
		{
			const FCluster& Cluster = Clusters[ClusterIndex];
			FPackedUVHeader* UVHeaders = PageWriter.Append_Ptr<FPackedUVHeader>(Cluster.VertexFormat.NumTexCoords);

			for (uint32 i = 0; i < Cluster.VertexFormat.NumTexCoords; i++)
			{
				PackUVHeader(UVHeaders[i], EncodingInfos[ClusterIndex].UVs[i]);
			}

			if (bHasSkinning)
			{
				FPackedBoneInfluenceHeader* BoneInfluenceHeader = PageWriter.Append_Ptr<FPackedBoneInfluenceHeader>(1);
				*BoneInfluenceHeader = PackedBoneInfluenceHeaders[LocalClusterIndex];
			}
		});
		
		PageWriter.AlignRelativeToOffset(DecodeInfoOffset, 16u);
		check(PageWriter.Offset() - DecodeInfoOffset == Page.GpuSizes.GetDecodeInfoSize());

		const uint32 RawFloat4EndOffset = PageWriter.Offset();
		
		uint32 StripBitmaskOffset = 0u;
		// Index data
		{
			const uint32 StartOffset = PageWriter.Offset();
			uint32 NextOffset = StartOffset;
#if NANITE_USE_STRIP_INDICES
			ProcessPageClusters(Page, Parts, [&](uint32 LocalClusterIndex, uint32 ClusterIndex)
			{
				const FCluster& Cluster = Clusters[ClusterIndex];

				FClusterDiskHeader& ClusterDiskHeader = ClusterDiskHeaders[LocalClusterIndex];
				ClusterDiskHeader.IndexDataOffset = NextOffset;
				ClusterDiskHeader.NumPrevNewVerticesBeforeDwords = Cluster.StripDesc.NumPrevNewVerticesBeforeDwords;
				ClusterDiskHeader.NumPrevRefVerticesBeforeDwords = Cluster.StripDesc.NumPrevRefVerticesBeforeDwords;
					
				NextOffset += Cluster.StripIndexData.Num();
			});

			const uint32 Size = NextOffset - StartOffset;
			uint8* IndexDataPtr = PageWriter.Append_Ptr<uint8>(Size);
			FMemory::Memcpy(IndexDataPtr, CombinedIndexData.GetData(), Size);
			PageWriter.Align(sizeof(uint32));

			StripBitmaskOffset = PageWriter.Offset();

			{
				uint32 StripBitmaskDataSize = CombinedStripBitmaskData.Num() * CombinedStripBitmaskData.GetTypeSize();
				uint8* StripBitmaskData = PageWriter.Append_Ptr<uint8>(StripBitmaskDataSize);
				FMemory::Memcpy(StripBitmaskData, CombinedStripBitmaskData.GetData(), StripBitmaskDataSize);
			}
			
#else
			for (uint32 i = 0; i < Page.NumClusters; i++)
			{
				ClusterDiskHeaders[i].IndexDataOffset = NextOffset;
				NextOffset += PackedClusters[i].GetNumTris() * 3;
			}
			PageWriter.Align(sizeof(uint32));

			const uint32 Size = NextOffset - StartOffset;
			check(Size == CombinedIndexData.Num() * CombinedIndexData.GetTypeSize());
			uint8* IndexDataPtr = PageWriter.Append_Ptr<uint8>(Size);
			FMemory::Memcpy(IndexDataPtr, CombinedIndexData.GetData(), CombinedIndexData.Num() * CombinedIndexData.GetTypeSize());
#endif
		}

		// Write PageCluster Map
		{
			const uint32 StartOffset = PageWriter.Offset();
			uint32 NextOffset = StartOffset;
			for (uint32 i = 0; i < Page.NumClusters; i++)
			{
				ClusterDiskHeaders[i].PageClusterMapOffset = NextOffset;
				NextOffset += NumPageClusterPairsPerCluster[i] * sizeof(uint32);
			}
			const uint32 Size = NextOffset - StartOffset;
			check(Size == CombinedPageClusterPairData.Num() * CombinedPageClusterPairData.GetTypeSize());
			check(Size % 4 == 0);
			uint32* PageClusterMapPtr = PageWriter.Append_Ptr<uint32>(Size / 4);
			FMemory::Memcpy(PageClusterMapPtr, CombinedPageClusterPairData.GetData(), CombinedPageClusterPairData.Num() * CombinedPageClusterPairData.GetTypeSize());
		}

		// Write Vertex Reference Bitmask
		const uint32 VertexRefBitmaskOffset = PageWriter.Offset();
		{
			const uint32 VertexRefBitmaskSize = Page.NumClusters * (NANITE_MAX_CLUSTER_VERTICES / 8);
			uint8* VertexRefBitmask = PageWriter.Append_Ptr<uint8>(VertexRefBitmaskSize);
			FMemory::Memcpy(VertexRefBitmask, CombinedVertexRefBitmaskData.GetData(), VertexRefBitmaskSize);
			check(CombinedVertexRefBitmaskData.Num() * CombinedVertexRefBitmaskData.GetTypeSize() == VertexRefBitmaskSize);
		}

		// Write Vertex References
		{
			const uint32 StartOffset = PageWriter.Offset();
			uint32 NextOffset = StartOffset;
			for (uint32 i = 0; i < Page.NumClusters; i++)
			{
				const uint32 NumVertexRefs = PackedClusters[i].GetNumVerts() - CodedVerticesPerCluster[i];
				ClusterDiskHeaders[i].VertexRefDataOffset	= NextOffset;
				ClusterDiskHeaders[i].NumVertexRefs			= NumVertexRefs;
				NextOffset += NumVertexRefs;
			}
			const uint32 Size = NextOffset - StartOffset;
			uint8* VertexRefs = PageWriter.Append_Ptr<uint8>(Size * 2); // * 2 to also allocate space for the high bytes that follow
			PageWriter.Align(sizeof(uint32));

			// Split low and high bytes for better compression
			for (int32 i = 0; i < CombinedVertexRefData.Num(); i++)
			{
				VertexRefs[i] = CombinedVertexRefData[i] >> 8;
				VertexRefs[i + CombinedVertexRefData.Num()] = CombinedVertexRefData[i] & 0xFF;
			}
		}
		
		// Write low/mid/high byte streams
		{
			const uint32 StartOffset = PageWriter.Offset();
			uint32 NextLowOffset = StartOffset;
			uint32 NextMidOffset = NextLowOffset + LowByteStream.Num();
			uint32 NextHighOffset = NextMidOffset + MidByteStream.Num();
			for (uint32 i = 0; i < Page.NumClusters; i++)
			{
				ClusterDiskHeaders[i].LowBytesOffset = NextLowOffset;
				ClusterDiskHeaders[i].MidBytesOffset = NextMidOffset;
				ClusterDiskHeaders[i].HighBytesOffset = NextHighOffset;
				NextLowOffset += ByteStreamCounters[i].Low;
				NextMidOffset += ByteStreamCounters[i].Mid;
				NextHighOffset += ByteStreamCounters[i].High;
			}

			const uint32 Size = NextHighOffset - StartOffset;
			check(Size == LowByteStream.Num() + MidByteStream.Num() + HighByteStream.Num());

			uint8* Ptr = PageWriter.Append_Ptr<uint8>(Size);
			FMemory::Memcpy(Ptr, LowByteStream.GetData(), LowByteStream.Num());
			Ptr += LowByteStream.Num();
			FMemory::Memcpy(Ptr, MidByteStream.GetData(), MidByteStream.Num());
			Ptr += MidByteStream.Num();
			FMemory::Memcpy(Ptr, HighByteStream.GetData(), HighByteStream.Num());
		}

		const uint32 NumRawFloat4Bytes = RawFloat4EndOffset - RawFloat4StartOffset;
		check((NumRawFloat4Bytes & 15u) == 0u);

		// Write page header
		{
			FPageDiskHeader PageDiskHeader;
			PageDiskHeader.NumClusters = Page.NumClusters;
			PageDiskHeader.NumRawFloat4s = NumRawFloat4Bytes / 16u;
			PageDiskHeader.NumVertexRefs = CombinedVertexRefData.Num();
			PageDiskHeader.DecodeInfoOffset = DecodeInfoOffset;
			PageDiskHeader.StripBitmaskOffset = StripBitmaskOffset;
			PageDiskHeader.VertexRefBitmaskOffset = VertexRefBitmaskOffset;
			FMemory::Memcpy(PageResult.GetData() + PageDiskHeaderOffset, &PageDiskHeader, sizeof(PageDiskHeader));
		}

		// Write cluster headers
		FMemory::Memcpy(PageResult.GetData() + ClusterDiskHeadersOffset, ClusterDiskHeaders.GetData(), ClusterDiskHeaders.Num()* ClusterDiskHeaders.GetTypeSize());

		PageWriter.Align(sizeof(uint32));
#if 0
		FILE* File = nullptr;
		char Filename[128];
		sprintf(Filename, "f:\\test\\newnew\\%d.dat", PageIndex);
		fopen_s(&File, Filename, "wb");
		fwrite(PageResult.GetData(), PageResult.Num(), 1, File);
		fclose(File);
#endif
	});

	// Write pages
	uint32 NumRootPages = 0;
	uint32 TotalRootGPUSize = 0;
	uint32 TotalRootDiskSize = 0;
	uint32 NumStreamingPages = 0;
	uint32 TotalStreamingGPUSize = 0;
	uint32 TotalStreamingDiskSize = 0;
	
	uint32 TotalFixupSize = 0;
	for (uint32 PageIndex = 0; PageIndex < NumPages; PageIndex++)
	{
		const FPage& Page = Pages[PageIndex];
		const bool bRootPage = Resources.IsRootPage(PageIndex);
		FFixupChunk& FixupChunk = FixupChunks[PageIndex];
		TArray<uint8>& BulkData = bRootPage ? Resources.RootData : StreamableBulkData;

		FPageStreamingState& PageStreamingState = Resources.PageStreamingStates[PageIndex];
		PageStreamingState.BulkOffset = BulkData.Num();

		// Write fixup chunk
		uint32 FixupChunkSize = FixupChunk.GetSize();
		BulkData.Append((uint8*)&FixupChunk, FixupChunkSize);
		TotalFixupSize += FixupChunkSize;

		// Copy page to BulkData
		TArray<uint8>& PageData = PageResults[PageIndex];
		BulkData.Append(PageData.GetData(), PageData.Num());
		
		if (bRootPage)
		{
			TotalRootGPUSize += Page.GpuSizes.GetTotal();
			TotalRootDiskSize += PageData.Num();
			NumRootPages++;
		}
		else
		{
			TotalStreamingGPUSize += Page.GpuSizes.GetTotal();
			TotalStreamingDiskSize += PageData.Num();
			NumStreamingPages++;
		}

		PageStreamingState.BulkSize = BulkData.Num() - PageStreamingState.BulkOffset;
		PageStreamingState.PageSize = PageData.Num();
	}

	const uint32 TotalPageGPUSize = TotalRootGPUSize + TotalStreamingGPUSize;
	const uint32 TotalPageDiskSize = TotalRootDiskSize + TotalStreamingDiskSize;
	UE_LOG(LogStaticMesh, Log, TEXT("WritePages:"), NumPages);
	UE_LOG(LogStaticMesh, Log, TEXT("  Root: GPU size: %d bytes. %d Pages. %.3f bytes per page (%.3f%% utilization)."), TotalRootGPUSize, NumRootPages, (float)TotalRootGPUSize / (float)NumRootPages, (float)TotalRootGPUSize / (float(NumRootPages * NANITE_ROOT_PAGE_GPU_SIZE)) * 100.0f);
	if(NumStreamingPages > 0)
	{
		UE_LOG(LogStaticMesh, Log, TEXT("  Streaming: GPU size: %d bytes. %d Pages (%d with relative encoding). %.3f bytes per page (%.3f%% utilization)."), TotalStreamingGPUSize, NumStreamingPages, NumRelativeEncodingPages, (float)TotalStreamingGPUSize / float(NumStreamingPages), (float)TotalStreamingGPUSize / (float(NumStreamingPages * NANITE_STREAMING_PAGE_GPU_SIZE)) * 100.0f);
	}
	else
	{
		UE_LOG(LogStaticMesh, Log, TEXT("  Streaming: 0 bytes."));
	}
	UE_LOG(LogStaticMesh, Log, TEXT("  Page data disk size: %d bytes. Fixup data size: %d bytes."), TotalPageDiskSize, TotalFixupSize);
	UE_LOG(LogStaticMesh, Log, TEXT("  Total GPU size: %d bytes, Total disk size: %d bytes."), TotalPageGPUSize, TotalPageDiskSize + TotalFixupSize);

	// Store PageData
	Resources.StreamablePages.Lock(LOCK_READ_WRITE);
	uint8* Ptr = (uint8*)Resources.StreamablePages.Realloc(StreamableBulkData.Num());
	FMemory::Memcpy(Ptr, StreamableBulkData.GetData(), StreamableBulkData.Num());
	Resources.StreamablePages.Unlock();
	Resources.StreamablePages.SetBulkDataFlags(BULKDATA_Force_NOT_InlinePayload);

	if(OutTotalGPUSize)
	{
		*OutTotalGPUSize = TotalRootGPUSize + TotalStreamingGPUSize;
	}
}
```
