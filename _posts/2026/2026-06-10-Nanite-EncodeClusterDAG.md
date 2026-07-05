---
layout: post
title:  "Nanite - Encode Cluster DAG"
date:   2026-06-10 20:18:00 +800
category: Unreal Engine
---

- [1. 准备编码数据](#1-准备编码数据)
  - [1.1. 清理非法属性值以及删除退化三角形](#11-清理非法属性值以及删除退化三角形)
  - [1.2. 按材质重新排序 Cluster 内的三角形并构建材质段信息](#12-按材质重新排序-cluster-内的三角形并构建材质段信息)
  - [1.3. **Stripify** Cluster](#13-stripify-cluster)
  - [1.4. 为每个材质段切分 Batches](#14-为每个材质段切分-batches)
  - [1.5. 使用全局统一精度量化 Cluster 顶点 Position](#15-使用全局统一精度量化-cluster-顶点-position)
  - [1.6. 量化 Cluster 顶点 BoneWeight](#16-量化-cluster-顶点-boneweight)
- [2. 编码 Cluster DAG](#2-编码-cluster-dag)
  - [2.1. 计算 Cluster 编码信息和 GPU Page 数据大小](#21-计算-cluster-编码信息和-gpu-page-数据大小)
- [3. References](#3-references)

本篇笔记是 Unreal Engine 的 Nanite 系统中关于编码 Cluster DAG 数据的源码分析理解，基于引擎版本 5.7.3 release。

在 [Nanite - Build Cluster DAG](https://dipperc.github.io/2026/04/15/Nanite-BuildClusterDAG.html) 中我们知道了 Nanite 是怎么构建 cluster DAG 数据的，而在这篇笔记中，我们来了解一下 Nanite 是怎么将 cluster DAG 数据编码成运行时使用的 `Nanite::FResources` 数据并写到磁盘的。

## 1. 准备编码数据

既然要对 cluster DAG 数据进行编码，那么就要先规范化输入的 cluster DAG 数据，这里的规范化不仅仅包括清理非法值，还需要对 cluster 的相关数据进行整理，通过整套规范化的处理，是 cluster DAG 数据变成后续编码所需要的标准格式。

### 1.1. 清理非法属性值以及删除退化三角形

Nanite 首先会清理 cluster 中的非法顶点属性，避免 NaN 或者越界值进入后续编码，然后还会删除 cluster 中的退化三角形：

```cpp
// 清理非法 vertex 属性, 避免 NaN 或越界值进入后续编码
{
    TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::SanitizeVertexData);
    for (FCluster& Cluster : ClusterDAG.Clusters)
    {
        Cluster.Verts.Sanitize();
    }
}

// 删除退化三角形: 删除三个顶点里有重复顶点的三角形
{
    TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::RemoveDegenerateTriangles);
    RemoveDegenerateTriangles( ClusterDAG.Clusters );
}
```

### 1.2. 按材质重新排序 Cluster 内的三角形并构建材质段信息

Nanite 还会把每个 cluster 内的三角形**按材质重新排序**，最终使得 cluster 内**使用相同材质的三角形会被排序在一起，从而形成连续的区间，并且所在材质区间越大的三角形排的越靠前**。

首先会遍历 cluster 内的所有三角形，统计每个材质 index 对应的三角形数量：

```cpp
TArray< int32, TInlineAllocator<128> > MaterialElements;
TArray< int32, TInlineAllocator<64> > MaterialCounts;

MaterialElements.AddUninitialized( MaterialIndexes.Num() );
MaterialCounts.AddZeroed( NANITE_MAX_CLUSTER_MATERIALS );

for( int32 i = 0; i < MaterialIndexes.Num(); i++ )
{
    // MaterialElements[i] 表示 cluster 内第 i 个三角形
    MaterialElements[i] = i;
    // 当前材质 index 对应的三角形数量 + 1
    MaterialCounts[ MaterialIndexes[i] ]++;
}
```

然后根据统计的结果排序所有三角形，排序遵循的规则是：

1. 三角形数量最多的材质对应的三角形优先排前面；
2. 数量相同时，材质 index 小的排前面；
3. 数量相同，材质 index 也相同时，cluster 内三角形 index 小的排前面；

```cpp
MaterialElements.Sort(
    [&]( int32 A, int32 B )
    {
        // 材质 index
        int32 IndexA = MaterialIndexes[A];
        int32 IndexB = MaterialIndexes[B];

        // 材质数量
        int32 CountA = MaterialCounts[ IndexA ];
        int32 CountB = MaterialCounts[ IndexB ];

        // 优先材质数量多的排前面
        if( CountA != CountB )
            return CountA > CountB;

        // 其次是材质 index 小的排前面
        if( IndexA != IndexB )
            return IndexA < IndexB;

        // 最后是 cluster 内三角形 index 小的排前面
        return A < B;
    } );
```

下一步，则是根据排序后的 `MaterialElements` 构造 `MaterialRanges`：

```cpp
FMaterialRange CurrentRange;
CurrentRange.RangeStart = 0;
CurrentRange.RangeLength = 0;
CurrentRange.MaterialIndex = MaterialElements.Num() > 0 ? MaterialIndexes[ MaterialElements[0] ] : 0;

// 遍历排序后的三角形 index
for( int32 i = 0; i < MaterialElements.Num(); i++ )
{
    int32 MaterialIndex = MaterialIndexes[ MaterialElements[i] ];

    // 材质 index 发生变化, 则结束当前 range, 开始下一个 range
    if (CurrentRange.RangeLength > 0 && MaterialIndex != CurrentRange.MaterialIndex)
    {
        MaterialRanges.Add(CurrentRange);

        // 设置下一个 range 的起始 index 和材质 index
        CurrentRange.RangeStart = i;
        CurrentRange.RangeLength = 1;
        CurrentRange.MaterialIndex = MaterialIndex;
    }
    else
    {
        // range 数量 + 1
        ++CurrentRange.RangeLength;
    }
}

// Add last triangle to range
if (CurrentRange.RangeLength > 0)
{
    MaterialRanges.Add(CurrentRange);
}
```

`MaterialRanges` 数组中的每个材质段 `MaterialRange` 记录了 cluster 内从第 `RangeStart` 个三角形开始，连续 `RangeLength` 个三角形，都是使用的材质 `MaterialIndex`。

最后才真正重新排序 cluster 的顶点索引 `Indexes` 和材质索引 `MaterialIndexes`：

```cpp
// 记录 cluster 新的顶点索引数据
TArray< uint32 >    NewIndexes;
// 记录 cluster 新的材质索引数据
TArray< int32 >     NewMaterialIndexes;

NewIndexes.AddUninitialized( Indexes.Num() );
NewMaterialIndexes.AddUninitialized( MaterialIndexes.Num() );

for( uint32 NewIndex = 0; NewIndex < NumTris; NewIndex++ )
{
    // 按排序后的顺序获取原始三角形 index
    uint32 OldIndex = MaterialElements[ NewIndex ];
    // 按顺序写入新的顶点索引
    NewIndexes[ NewIndex * 3 + 0 ] = Indexes[ OldIndex * 3 + 0 ];
    NewIndexes[ NewIndex * 3 + 1 ] = Indexes[ OldIndex * 3 + 1 ];
    NewIndexes[ NewIndex * 3 + 2 ] = Indexes[ OldIndex * 3 + 2 ];
    // 按顺序写入新的材质索引
    NewMaterialIndexes[ NewIndex ] = MaterialIndexes[ OldIndex ];
}
// 将新的顶点索引和材质索引数据赋值给 cluster
Swap( Indexes,          NewIndexes );
Swap( MaterialIndexes,  NewMaterialIndexes );
```

### 1.3. **Stripify** Cluster

在看 stripify 的核心逻辑之前，我们先来看看几个关键方法 `BuildTables()`、`NewScoreVertex()` 和 `VisitTriangle()`。首先是 `BuildTables()`：

```cpp
void BuildTables( const FCluster& Cluster )
{
    // 每个三角形 Corner 的链表节点
    struct FEdgeNode
    {
        uint16 Corner;  // (Triangle << 2) | LocalCorner
        uint16 NextNode;
    };

    // 构建一个所有三角形的所有 Corner 的链表
    // 这里的 NANITE_MAX_CLUSTER_INDICES = 128 * 3, 一个 cluster 最多 128 个三角形, 每个三角形有 3 个 Corner
    FEdgeNode EdgeNodes[ NANITE_MAX_CLUSTER_INDICES ];

    // 注意: 每个三角形 Corner 其实表示的是一条三角形有向边

    // 记录每个三角形 Corner 链表的链表头, 主要用于处理非流形边被超过 2 个三角形共享的情况: EdgeNodeHeads 中只记录链表头, 后续每次匹配到邻接三角形后就将 EdgeNodeHeads 中记录的链表头指向下一个节点
    // 这里使用二维矩阵, 后续可以按顶点 index 直接寻址: EdgeNodeHeads[ index0 * NANITE_MAX_CLUSTER_INDICES + index1 ] 表示 index0 -> index1 这条有向边的链表头
    uint16 EdgeNodeHeads[ NANITE_MAX_CLUSTER_INDICES * NANITE_MAX_CLUSTER_INDICES ]; // Linked list per edge to support more than 2 triangles per edge.
    FMemory::Memset( EdgeNodeHeads, INVALID_NODE_MEMSET );

    FMemory::Memset( VertexToTriangleMasks, 0 );

    uint32 NumTriangles = Cluster.NumTris;
    uint32 NumVertices = Cluster.Verts.Num();

    // 遍历 cluster 内的所有三角形
    for( uint32 i = 0; i < NumTriangles; i++ )
    {
        // 取第 i 个三角形的 3 个顶点 index
        uint32 i0 = Cluster.Indexes[ i * 3 + 0 ];
        uint32 i1 = Cluster.Indexes[ i * 3 + 1 ];
        uint32 i2 = Cluster.Indexes[ i * 3 + 2 ];

        // 保证不是退化三角形, 并且 index 没有越界
        check( i0 != i1 && i1 != i2 && i2 != i0 );
        check( i0 < NumVertices && i1 < NumVertices && i2 < NumVertices );

        // 使用 4 个 32-triangle DWORD 一共 128 位来表示 cluster 中的每个三角形, ( i >> 5 ) 计算出三角形 i 属于第几个 DWORD; 1 << ( i & 31 ) 计算出三角形 i 在所属 DWORD 中的具体 bit 位
        // 在这里记录顶点 i0, i1 和 i2 关联了三角形 i
        VertexToTriangleMasks[ i0 ][ i >> 5 ] |= 1 << ( i & 31 );
        VertexToTriangleMasks[ i1 ][ i >> 5 ] |= 1 << ( i & 31 );
        VertexToTriangleMasks[ i2 ][ i >> 5 ] |= 1 << ( i & 31 );

        // 计算三角形 3 个顶点**位置和**的 X 分量值, 后续选 strip 起点时, 如果评分相同, 则用它决定谁优先
        FVector3f ScaledCenter = Cluster.Verts.GetPosition( i0 ) + Cluster.Verts.GetPosition( i1 ) + Cluster.Verts.GetPosition( i2 );
        TrianglePriorities[ i ] = ScaledCenter.X;

        // 为三角形 i 的 Corner 0 建立链表节点, 需要注意: 当前三角形 Corner 0 代表的是它的有向边 i1 -> i2
        FEdgeNode& Node0 = EdgeNodes[ i * 3 + 0 ];
        // 将三角形索引 i 和 Corner 索引 0 编码到 Corner
        Node0.Corner = (uint16)SetCorner( i, 0 );
        // 当前的节点插入链表前, 先把原来的链表头保存到其 NextNode
        Node0.NextNode = EdgeNodeHeads[ i1 * NANITE_MAX_CLUSTER_INDICES + i2 ];
        // 把当前 Corner 0 设置为有向边 i1 -> i2 的新头节点
        EdgeNodeHeads[ i1 * NANITE_MAX_CLUSTER_INDICES + i2 ] = uint16(i * 3 + 0);

        // 同样为三角形 i 的 Corner 1 建立链表节点, 并更新有向边 i2 -> i0 的新头节点
        FEdgeNode& Node1 = EdgeNodes[ i * 3 + 1 ];
        Node1.Corner = (uint16)SetCorner( i, 1 );
        Node1.NextNode = EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i0 ];
        EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i0 ] = uint16(i * 3 + 1);

        // 同样为三角形 i 的 Corner 2 建立链表节点, 并更新有向边 i0 -> i1 的新头节点
        FEdgeNode& Node2 = EdgeNodes[ i * 3 + 2 ];
        Node2.Corner = (uint16)SetCorner( i, 2 );
        Node2.NextNode = EdgeNodeHeads[ i0 * NANITE_MAX_CLUSTER_INDICES + i1 ];
        EdgeNodeHeads[ i0 * NANITE_MAX_CLUSTER_INDICES + i1 ] = uint16(i * 3 + 2);
    }

    // 找共享边, 并构建 OppositeCorner 查找表
    for( uint32 i = 0; i < NumTriangles; i++ )
    {
        // 取第 i 个三角形的 3 个顶点 index
        uint32 i0 = Cluster.Indexes[ i * 3 + 0 ];
        uint32 i1 = Cluster.Indexes[ i * 3 + 1 ];
        uint32 i2 = Cluster.Indexes[ i * 3 + 2 ];

        // 因为当前三角形 Corner 0 表示的是有向边 i1 -> i2, 而这里找共享边, 所以直接按 i2 -> i1 从 EdgeNodeHeads 中去取头节点
        uint16& Node0 = EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i1 ];
        // 同理 Corner 1 表示的是有向边 i2 -> i0, 而这里找共享边, 所以直接按 i0 -> i2 从 EdgeNodeHeads 中去取头节点
        uint16& Node1 = EdgeNodeHeads[ i0 * NANITE_MAX_CLUSTER_INDICES + i2 ];
        // 同理 Corner 2 表示的是有向边 i0 -> i1, 而这里找共享边, 所以直接按 i1 -> i0 从 EdgeNodeHeads 中去取头节点
        uint16& Node2 = EdgeNodeHeads[ i1 * NANITE_MAX_CLUSTER_INDICES + i0 ];

        // 如果 EdgeNodeHeads[i2, i1] 中取出的节点不为空, 则说明找到当前三角形有向边 i1 -> i2 的共享边
        if( Node0 != INVALID_NODE )
        {
            // EdgeNodes[Node0].Corner 就是找到的共享边其对应的三角形 Corner, 将其写入 OppositeCorner[i * 3 + 0]
            OppositeCorner[ i * 3 + 0 ] = EdgeNodes[ Node0 ].Corner;
            // 把 EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i1 ] 指向的链表头推进到下一个节点. 保证在非流形的情况下, 不会多次取到重复的共享边
            Node0 = EdgeNodes[ Node0 ].NextNode;
        }
        else
        {
            // 如果没有找到贡献边, 则置为 INVALID_CORNER
            OppositeCorner[ i * 3 + 0 ] = INVALID_CORNER;
        }

        // 如下同理
        if( Node1 != INVALID_NODE ) { OppositeCorner[ i * 3 + 1 ] = EdgeNodes[ Node1 ].Corner; Node1 = EdgeNodes[ Node1 ].NextNode; }
        else { OppositeCorner[ i * 3 + 1 ] = INVALID_CORNER; }
        if( Node2 != INVALID_NODE ) { OppositeCorner[ i * 3 + 2 ] = EdgeNodes[ Node2 ].Corner; Node2 = EdgeNodes[ Node2 ].NextNode; }
        else { OppositeCorner[ i * 3 + 2 ] = INVALID_CORNER; }
    }
}
```

在详细说 `BuildTables()` 方法之前，先要说明一下上面源码中的**三角形 `Corner` 代表的是什么**：

三角形 `Corner` 的数据类型是 `uint16`，其高 14 位编码了三角形索引 `i`，低 2 位编码了三角形 `i` 中的局部顶点索引 `0/1/2`，也就是说一个三角形 `Corner` 表示的是三角形 `i` 的 3 个顶点中的某一个。不过在上面的源码中，Nanite 也会用这个 `Corner` 来代表**它所表示的三角形顶点对面的那条有向边**。举个例子：`Corner(i, 0)` 表示的是三角形 `i` 的第 0 个顶点 `i0`，而它对应的对边是三角形 `i` 的有向边 `i1 -> i2`；同理，`Corner(i, 1)` 对应有向边 `i2 -> i0`，`Corner(i, 2)` 对应有向边 `i0 -> i1`。

`BuildTables()` 是在 stripify 之前的关键一步，它根据 cluster 的原始网格信息构建了 stripify 所需的数据：首先是 `VertexToTriangleMasks`，它记录 **cluster 内每个顶点关联了哪些三角形**，通过 `VertexToTriangleMasks[ vertex index ][ DWORD i ]` 可以快速知道某个 32-triangle DWORD 中有哪些三角形使用了顶点 `Verts[index]`；其次是 `OppositeCorner`，它记录**每个三角形 `Corner` 所代表的有向边，其反向共享边对应的是哪个三角形 `Corner`**，后续 stripify 时会根据此数据找左/右相邻三角形；最后 `TrianglePriorities` 则是保存每个三角形 3 个顶点**位置和**的 `X` 分量值，后续选 strip 起点三角形时，如果评分相同，则用它决定谁优先。

再来看看 `NewScoreVertex()`：

```cpp
auto NewScoreVertex = [ &Weights ] ( const FContext& Context, uint32 OldVertex, bool bStart, bool bHasOpposite, bool bHasLeft, bool bHasRight )
{
    // 当前旧顶点在新顶点序列中的索引
    uint16 NewIndex = Context.OldToNewVertex[ OldVertex ];

    // 默认分数为 0
    int32 CacheScore = 0;

    // 如果当前旧顶点已经对应到新顶点序列中的某个已输出新顶点
    if( NewIndex != INVALID_INDEX )
    {
        // 计算其对应的新顶点与当前新顶点序列末尾的相对距离
        uint32 CachePosition = ( Context.NumVertices - 1 ) - NewIndex;
        // 相对距离是否满足 5-bit 偏移约束
        if( CachePosition < NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE )
        {
            // 如果满足约束, 就从权重表取分数, 根据以下 5 个维度查表:
            // bStart: 当前候选三角形是否作为 strip 的起点
            // bHasOpposite: 当输入是 strip 起点三角形时, 表示邻接三角形是否可用; 当输入是非 strip 起点三角形时, 用于区分当前三角形是左延伸 (true) 还是右延伸 (false)
            // bHasLeft: 左侧邻接三角形是否可用
            // bHasRight: 右侧邻接三角形是否可用
            // CachePosition: 与当前新顶点序列末尾的相对距离
            CacheScore = Weights.Weights[ bStart ][ bHasOpposite ][ bHasLeft ][ bHasRight ][ CachePosition ];
        }
    }

    return CacheScore;
};
```

`NewScoreVertex()` 方法的作用是给一个候选旧顶点打分。它首先检查这个旧顶点是否已经对应到新顶点序列中的某个已输出新顶点，如果还没有对应的新顶点，则这个旧顶点当前不能作为 ref 顶点使用，分数就是 0；如果已经有对应的新顶点，则计算这个新顶点与当前新顶点序列末尾之间的相对距离，也就是 `CachePosition = ( Context.NumVertices - 1 ) - NewIndex`。如果这个相对距离仍在 5-bit 可表示范围内，就可以作为 ref 顶点候选，然后根据当前候选三角形是否是 strip 起点、它的邻接三角形情况，以及这个 ref 距离去权重表里取分数。

所以 `NewScoreVertex()` 的打分主要反映两件事：这个旧顶点作为 ref 顶点是否足够“近”，以及当前候选三角形所处的拓扑上下文是否适合继续 stripify。另外提一下 `NewScoreTriangle()` 方法，它就是分别给三角形的 3 个旧顶点打分，然后把 3 个分数相加作为这个候选三角形的评分。

在说 `VisitTriangle()` 方法之前，先要了解一下 `FContext` 类，它记录了整个 stripify 过程中的临时数据：

```cpp
class FContext
{
public:
    bool TriangleEnabled( uint32 TriangleIndex ) const
    {
        return ( TrianglesEnabled[ TriangleIndex >> 5 ] & ( 1u << ( TriangleIndex & 31u ) ) ) != 0u;
    }

    uint16 OldToNewVertex[NANITE_MAX_CLUSTER_TRIANGLES * 3 ];   // 旧顶点索引到新顶点序列索引的映射
    uint16 NewToOldVertex[NANITE_MAX_CLUSTER_TRIANGLES * 3 ];   // 新顶点序列索引到旧顶点索引的映射

    // 当前材质段内尚未输出、仍可作为候选的三角形 mask
    uint32 TrianglesEnabled[ MAX_CLUSTER_TRIANGLES_IN_DWORDS ]; // Enabled triangles are in the current material range and have not yet been visited.
    // 至少有一个顶点被访问过的三角形 mask
    uint32 TrianglesTouched[ MAX_CLUSTER_TRIANGLES_IN_DWORDS ]; // Touched triangles have had at least one of their vertices visited.

    // 记录每个三角形输出后的 strip 编码信息
    // 当三角形作为 strip 起点输出时, 3 列分别是: Reset = 1; 起点三角形 ref 顶点数高 bit; 起点三角形 ref 顶点数低 bit
    // 当三角形作为 strip 延伸输出时, 3 列分别是: Reset = 0; 是否从左侧延伸过来; 新引入的顶点是否是 ref 顶点
    uint32 StripBitmasks[ 4 ][ 3 ]; // [4][Reset, IsLeft, IsRef]

    uint32 NumTriangles;    // 已输出到 stripified 三角形序列中的三角形数
    uint32 NumVertices;     // 已输出到新顶点序列中的新顶点数
};
```

`VisitTriangle()` 方法的核心逻辑是：**将一个旧三角形输出到 stripify 之后的三角形序列中，并决定这个三角形相关的旧顶点在 stripify 之后应当编码为新顶点还是 ref 顶点，最后将此三角形的 strip 编码信息写入 `StripBitmasks`**。

需要注意的是，对于 strip 的起点三角形，最多需要处理其 3 个顶点的编码；而对于 strip 的延伸三角形，因为其前 2 个顶点通常由 strip 拓扑隐含复用，所以只需要处理其新引入的那个顶点的编码。

在这里单独说明一下什么是 5-bit 偏移约束：Nanite 在 `StripIndexData` 中使用 5 bits 存储 ref 顶点的相对距离，这个相对距离引用的是新顶点序列中已经输出过的某个新顶点，这个 5-bit 字段中写入的是 `BaseVertex - Index`，其中 `BaseVertex` 是当前三角形输出前新顶点序列末尾的索引，`Index` 是被引用的新顶点索引。因为只有 5 bits，所以这个距离必须在 5-bit 可表示范围内，也就是小于 32。如果某个旧顶点虽然已经对应到新顶点序列中的一个已输出顶点，但距离太远，就不能继续作为 ref 顶点编码，而需要重新作为新顶点编码。

在 `VisitTriangle()` 方法中，首先会通过传入的三角形 Corner 参数获取三角形的 3 个顶点，并将这 3 个顶点关联的三角形标记为 touched：

```cpp
// 按 Corner 0 -> Corner 1 -> Corner 2 -> Corner 0 的顺序获取三角形的 3 个旧顶点:

// 下一个 TriangleCorner 对应的旧顶点
const uint32 OldIndex0 = Cluster.Indexes[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
// 前一个 TriangleCorner 对应的旧顶点
const uint32 OldIndex1 = Cluster.Indexes[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];
// 当前 TriangleCorner 对应的旧顶点
const uint32 OldIndex2 = Cluster.Indexes[ CornerToIndex( TriangleCorner ) ];

// Mark incident triangles
for( uint32 i = 0; i < MAX_CLUSTER_TRIANGLES_IN_DWORDS; i++ )
{
    // 标记这 3 个顶点关联的三角形为 touched
    Context.TrianglesTouched[ i ] |= VertexToTriangleMasks[ OldIndex0 ][ i ] | VertexToTriangleMasks[ OldIndex1 ][ i ] | VertexToTriangleMasks[ OldIndex2 ][ i ];
}
```

第二步，对当前三角形中可能作为 ref 顶点编码的旧顶点执行 **5-bit 偏移约束**检查，通过约束检查的顶点可以作为 ref 顶点编码，否则需要重新作为新顶点编码：

```cpp
// 3 个旧顶点在新顶点序列中的索引
uint16& NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
uint16& NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
uint16& NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];

uint32 OrgIndex0 = NewIndex0;
uint32 OrgIndex1 = NewIndex1;
uint32 OrgIndex2 = NewIndex2;

// 根据 3 个旧顶点是否已经对应到新顶点序列，预估当前三角形处理完成后，下一个可分配的新顶点序列索引
uint32 NextVertexIndex = Context.NumVertices + ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );

while(true)
{
    // 如果旧顶点 OldIndex0 已经存在于新顶点序列中，说明它有机会作为 ref 顶点编码。
    // 但是如果它对应的新顶点距离当前三角形处理完成后的新顶点序列末尾太远，
    // 导致无法用 5 bits 记录这个相对距离，就需要把它重新作为新顶点编码。
    if( NewIndex0 != INVALID_INDEX && NextVertexIndex - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE )
    {
        // 此时把旧顶点重新作为一个新顶点编码
        NewIndex0 = INVALID_INDEX;
        // 新顶点序列中会新增一个顶点，所以下一个可分配的新顶点序列索引 + 1
        NextVertexIndex++;
        continue;
    }
    // 旧顶点 OldIndex1 和旧顶点 OldIndex2 同理
    if( NewIndex1 != INVALID_INDEX && NextVertexIndex - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex1 = INVALID_INDEX; NextVertexIndex++; continue; }
    if( NewIndex2 != INVALID_INDEX && NextVertexIndex - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex2 = INVALID_INDEX; NextVertexIndex++; continue; }
    break;
}
```

在这里先通过 `Context.OldToNewVertex` 查询当前三角形的 3 个旧顶点是否已经对应到新顶点序列中的某个已输出新顶点。已经存在于新顶点序列中的旧顶点，先视为 ref 顶点候选；还不存在于新顶点序列中的旧顶点，则需要作为新顶点编码。

然后对 ref 顶点候选执行 5-bit 偏移约束检查：分别检查每个 ref 顶点引用的新顶点，与当前三角形处理完成后的新顶点序列末尾之间的相对距离。如果这个相对距离 `>= 32`，说明无法用 5 bits 记录，则这个旧顶点不能继续作为 ref 顶点编码，而是会被重新作为新顶点编码。这里用的是 `NextVertexIndex` 做预检查，因为当前三角形本身也可能会新增新顶点，新增的新顶点会影响其他 ref 顶点的相对距离。

第三步，将当前三角形的 strip 编码信息写入 `StripBitmasks`：

```cpp
// 当前三角形输出后的新索引
uint32 NewTriangleIndex = Context.NumTriangles;
// 当前三角形输出后会新增多少个新顶点
uint32 NumNewVertices = ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );

// 当前三角形是作为 strip 起点输出
if( bStart )
{
    // ( NewIndex == INVALID_INDEX ) 表示旧顶点需要作为新顶点编码; ( NewIndex != INVALID_INDEX ) 表示旧顶点可以作为 ref 顶点编码
    // 这里的 check 表示: 当一个三角形作为 strip 起点输出时, 如果有新顶点, 它们必须按 OldIndex0 -> OldIndex1 -> OldIndex2 的顺序出现在后面
    check( ( NewIndex2 == INVALID_INDEX ) >= ( NewIndex1 == INVALID_INDEX ) );
    check( ( NewIndex1 == INVALID_INDEX ) >= ( NewIndex0 == INVALID_INDEX ) );

    // 当前起点三角形中的 ref 顶点数
    uint32 NumWrittenIndices = 3u - NumNewVertices;

    // 把 ref 顶点数拆成 2 bits 存储: 0 -> 00, 1 -> 01, 2 -> 10, 3 -> 11
    uint32 LowBit = NumWrittenIndices & 1u;
    uint32 HighBit = (NumWrittenIndices >> 1) & 1u;

    // 写入当前三角形的 strip 编码信息:
    // 对于 strip 起点三角形来说, bitmask 0 置为 1, 表示它是一个 strip 起点
    Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 0 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
    // 对于 strip 起点三角形来说, bitmask 1 表示其 ref 顶点数的高 bit
    Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 1 ] |= ( HighBit << ( NewTriangleIndex & 31u ) );
    // 对于 strip 起点三角形来说, bitmask 2 表示其 ref 顶点数的低 bit
    Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 2 ] |= ( LowBit << ( NewTriangleIndex & 31u ) );
}
else    // 当前三角形作为 strip 延伸输出
{
    // 这里的 check 表示: 当一个三角形作为 strip 延伸输出时, 前 2 个旧顶点必须已经存在于新顶点序列中，并且能够由 strip 拓扑隐含复用
    check( NewIndex0 != INVALID_INDEX );
    check( NewIndex1 != INVALID_INDEX );

    // 写入当前三角形的 strip 编码信息:
    // 对于 strip 延伸三角形来说, bitmask 0 置为 0, 这里直接不需要处理
    // 对于 strip 延伸三角形来说, bitmask 1 表示其是否从左侧延伸过来的, !bRight 也就是 IsLeft
    if( !bRight )
    {
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 1 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
    }

    // 对于 strip 延伸三角形来说, 这里的 bitmask 2 表示新引入的那个顶点是否作为 ref 顶点编码
    if(NewIndex2 != INVALID_INDEX)
    {
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 2 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
    }
}
```

这里详细说说 `StripBitmasks` 数据，它是一个 `4x3` 二维 `uint32` 数组，其中每 1 列的 `uint32` 中的每 1 bit 记录了其对应三角形输出后的一种 strip 编码信息，一共 3 列记录 `[Reset, IsLeft, IsRef]` 这 3 类信息。

当一个三角形作为 strip 起点输出时，此三角形所对应的第 1 列 bit 为 1，表示它是一个 strip 起点；然后将当前起点三角形中的 ref 顶点数拆成 2 bits 存储，高 bit 存储在第 2 列 bit 中，低 bit 存储在第 3 列 bit 中。

当一个三角形作为 strip 延伸输出时，此三角形所对应的第 1 列 bit 为 0，表示它是一个 strip 延伸三角形；第 2 列 bit 记录延伸方向，也就是它是从左侧还是右侧延伸过来的；第 3 列 bit 记录新引入的那个顶点是否作为 ref 顶点编码。

将当前三角形的 strip 编码信息写入 `StripBitmasks` 之后，Nanite 会为即将作为新顶点编码的旧顶点分配其在新顶点序列中的新索引，然后记录旧顶点索引到新顶点索引的映射，接着将当前三角形标记为已输出并且将其从候选集合中移除，最后返回当前三角形新增的新顶点数：

```cpp
if( NewIndex0 == INVALID_INDEX )
{
    // 给旧顶点 OldIndex0 分配新顶点索引, 并将已输出的新顶点序列中的新顶点数 + 1
    // 另外这里的 NewIndex0 是引用, 所以同步更新了旧顶点索引到新顶点索引的映射
    NewIndex0 = uint16(Context.NumVertices++);
    // 记录新顶点索引到旧顶点索引的映射
    Context.NewToOldVertex[ NewIndex0 ] = uint16(OldIndex0);
}
// OldIndex1, OldIndex2 同理
if( NewIndex1 == INVALID_INDEX ) { NewIndex1 = uint16(Context.NumVertices++); Context.NewToOldVertex[ NewIndex1 ] = uint16(OldIndex1); }
if( NewIndex2 == INVALID_INDEX ) { NewIndex2 = uint16(Context.NumVertices++); Context.NewToOldVertex[ NewIndex2 ] = uint16(OldIndex2); }

// 已输出的三角形数量 + 1
Context.NumTriangles++;

// 从 Corner 中解码出旧三角形索引
const uint32 OldTriangleIndex = CornerToTriangle( TriangleCorner );
// 把当前三角形标记为已输出, 防止后续再次被输出
Context.TrianglesEnabled[ OldTriangleIndex >> 5 ] &= ~( 1u << ( OldTriangleIndex & 31u ) );

// 返回当前三角形输出的新顶点数
return NumNewVertices;
```

现在总结一下上面说的 3 个关键方法的核心逻辑：

`BuildTables()` 方法主要构建 stripify 所需的几类辅助数据：`VertexToTriangleMasks` 记录每个旧顶点关联了哪些旧三角形；`OppositeCorner` 通过三角形 `Corner` 记录邻接三角形之间的反向共享边信息，Nanite 后续可以根据三角形 `Corner` 找到特定的左侧或者右侧邻接三角形；`TrianglePriorities` 保存每个旧三角形 3 个旧顶点位置和的 `X` 分量值，后续在评分相同的情况下，用它决定哪个三角形优先作为 strip 起点输出。

而 `NewScoreVertex()` 方法主要是根据一个旧顶点是否可以作为 ref 顶点、作为 ref 顶点时距离当前新顶点序列末尾有多远，以及候选三角形的拓扑上下文为这个旧顶点打分。`NewScoreTriangle()` 则是把三角形 3 个旧顶点的分数相加，用于决定 strip 起点和 strip 延伸时优先选择哪个候选三角形。

`VisitTriangle()` 是整个 stripify 过程中的核心方法，它的主要逻辑是将一个旧三角形输出到 stripify 之后的三角形序列中，并决定这个三角形相关的旧顶点在 stripify 之后应当编码为一个新顶点还是一个 ref 顶点。需要注意的是，对于 strip 的起点三角形，最多需要处理其 3 个顶点的编码，而对于 strip 的延伸三角形，因为其前 2 个顶点通常由 strip 拓扑隐含复用，所以只需要处理其新引入的那个顶点的编码。具体来说：

对于需要编码的旧顶点，首先检查它是否已经对应到新顶点序列中的某个已输出顶点，如果还不存在于新顶点序列中，则将其作为一个新顶点编码；而如果它已经存在于新顶点序列中，则优先考虑将其作为 ref 顶点编码：通过记录其对应的已输出新顶点与当前新顶点序列末尾之间的相对距离，来引用这个已输出的新顶点。又因为 Nanite 使用 5 bits 去存储这个相对距离，所以如果这个相对距离超过了 5-bit 可表示范围，那么仍然会将这个旧顶点重新作为一个新顶点编码。

说完了 `BuildTables()`、`NewScoreVertex()` 和 `VisitTriangle()` 这 3 个关键方法，下面来说说 stripify 的核心流程：

Nanite 是**按材质段 `MaterialRange` 分段**进行 stripify 的，这样可以保持材质分组不被打乱，避免 strip 跨材质。

首先会初始化相关数据：

```cpp
// 清掉旧的 Cluster.StripIndexData
Cluster.StripIndexData.Empty();
// 初始化 BitWriter, 绑定数组 Cluster.StripIndexData
FBitWriter BitWriter( Cluster.StripIndexData );
// 清掉旧的 strip 编码信息
FStripDesc& StripDesc = Cluster.StripDesc;
FMemory::Memset(StripDesc, 0);
// 按 32-triangle DWORD 记录每个 DWORD 内新增的新顶点数
uint32 NumNewVerticesInDword[ 4 ] = {};
// 按 32-triangle DWORD 记录每个 DWORD 内编码的 ref 顶点数
uint32 NumRefVerticesInDword[ 4 ] = {};
```

`Cluster.StripIndexData` 是一个连续、紧凑的 5-bit **bitstream（位流）**，Nanite 通过 `BitWriter.PutBits(BaseVertex - Index, 5)` 以**低位优先**的顺序向其中写入连续的 5-bit ref 偏移值，每满 8 bits 就会将其转换为一个 `uint8` 并添加进 `Cluster.StripIndexData` 数组中。

按材质段分段进行 stripify 时，首先会将当前材质段内的所有旧三角形标记为候选状态，后续 stripify 时只处理候选三角形：

```cpp
// 将当前材质段中的所有旧三角形标记为候选状态:
// Context.TrianglesEnabled 是 4 个 DWORD, 每个 DWORD 的 32 bits 用于标记 32 个三角形的候选状态, 一共 128 bits 标记 cluster 中的 128 个三角形
for( uint32 i = 0; i < MAX_CLUSTER_TRIANGLES_IN_DWORDS; i++ )
{
    // 当前材质段起点在每个 DWORD 内的 bit 偏移
    int32 RangeStartRelativeToDword = (int32)RangeStart - (int32)i * 32;
    // 当前材质段起点在每个 DWORD 内对应的 bit 位
    int32 BitStart = FMath::Max( RangeStartRelativeToDword, 0 );
    // 当前材质段终点在每个 DWORD 内对应的 bit 位
    int32 BitEnd = FMath::Max( RangeStartRelativeToDword + (int32)RangeLength, 0 );
    // BitStart >= 32 则说明材质段起点不在当前这个 DWORD 中, StartMask 直接使用 0xFFFFFFFFu; 否则将 [0, BitStart) 的 bits 置为 1
    uint32 StartMask = BitStart < 32 ? ( ( 1u << BitStart ) - 1u ) : 0xFFFFFFFFu;
    // BitEnd >= 32 则说明材质段终点不在当前这个 DWORD 中, EndMask 直接使用 0xFFFFFFFFu; 否则将 [0, BitEnd) 的 bits 置为 1
    uint32 EndMask = BitEnd < 32 ? ( ( 1u << BitEnd ) - 1u ) : 0xFFFFFFFFu;
    // 最后按位异或将当前材质段三角形所属 DWORD 对应的 bits 置为 1
    // 举个例子: StartMask = bits 0..9, EndMask = bits 0..14, 那么 StartMask ^ EndMask = bits 10..14
    Context.TrianglesEnabled[ i ] |= StartMask ^ EndMask;
}
```

然后通过 `NewScoreTriangle()` 方法为每个候选三角形打分，并选取评分最高的候选三角形将其作为当前 strip 起点输出：

```cpp
for( uint32 Corner = 0; Corner < 3; Corner++ )
{
    // 尝试以当前三角形的 3 个不同 Corner 作为 strip 起点, 不同 Corner 会产生不同的起点局部顶点顺序
    uint32 TriangleCorner = SetCorner( TriangleIndex, Corner );

    // 检查以当前 Corner 作为 strip 起点时是否满足编码约束
    {
        // 按 strip 起点编码使用的局部顺序获取三角形的 3 个旧顶点. 这个顺序由当前 TriangleCorner 决定, 不等同于原始三角形 index buffer 中的 0,1,2 顺序
        uint32 OldIndex0 = Cluster.Indexes[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
        uint32 OldIndex1 = Cluster.Indexes[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];
        uint32 OldIndex2 = Cluster.Indexes[ CornerToIndex( TriangleCorner ) ];

        // 查看这 3 个旧顶点是否已经对应到新顶点序列中, 已存在的 NewIndex 后续可能作为 ref 顶点编码; INVALID_INDEX 则需要作为新顶点编码
        uint32 NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
        uint32 NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
        uint32 NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];
        uint32 NumVerts = Context.NumVertices + ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );
        
        // 对可能作为 ref 顶点编码的新顶点进行 5-bit 偏移约束检查: 如果其对应的已输出新顶点与当前新顶点序列末尾之间的相对距离超过 5 bits 可表示范围, 就必须把对应旧顶点重新作为新顶点编码
        while(true)
        {
            if( NewIndex0 != INVALID_INDEX && NumVerts - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex0 = INVALID_INDEX; NumVerts++; continue; }
            if( NewIndex1 != INVALID_INDEX && NumVerts - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex1 = INVALID_INDEX; NumVerts++; continue; }
            if( NewIndex2 != INVALID_INDEX && NumVerts - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex2 = INVALID_INDEX; NumVerts++; continue; }
            break;
        }

        // 检查当前候选 Corner 朝向是否可以作为 strip 起点: Mask 的第 N 位表示局部顶点 N 是否作为新顶点编码
        uint32 Mask = ( NewIndex0 == INVALID_INDEX ? 1u : 0u ) | ( NewIndex1 == INVALID_INDEX ? 2u : 0u ) | ( NewIndex2 == INVALID_INDEX ? 4u : 0u );

        // Strip 起点要求 ref 顶点必须在前, 新顶点必须连续出现在后面, 因此只允许:
        // 0: ref 顶点,  ref 顶点,  ref 顶点
        // 4: ref 顶点,  ref 顶点,   新顶点
        // 6: ref 顶点,   新顶点,    新顶点
        // 7:  新顶点,    新顶点,    新顶点
        if( Mask != 0u && Mask != 4u && Mask != 6u && Mask != 7u )
        {
            continue;
        }
    }

    ...
}
```

对于 strip 起点输出的旧三角形，Nanite 要求其 3 个旧顶点的按当前 Corner 决定的局部顶点顺序输出时，编码组合必须满足：**ref 顶点在前，新顶点连续出现在后面**。因此只允许以下几种情况：

```text
1. ref 顶点,  ref 顶点,  ref 顶点
2. ref 顶点,  ref 顶点,   新顶点
3. ref 顶点,   新顶点,    新顶点
4.  新顶点,    新顶点,    新顶点
```

所以在上面的代码中，Nanite 会尝试以每个候选旧三角形的 3 个不同 Corner 作为 strip 起点。不同 Corner 会产生不同的起点局部顶点顺序；随后 Nanite 会检查以当前 Corner 作为 strip 起点时是否满足上述编码组合要求，只有满足要求的旧三角形 Corner，才会作为一个合法的 strip 起点方案进入后续打分。

对于所有满足编码组合要求的旧三角形 Corner，会通过 `NewScoreTriangle()` 方法为其打分，并记录得分最高的旧三角形 Corner：

```cpp
// 当前旧三角形 Corner 周围 3 条边对应的邻接 Corner
uint32 Opposite = OppositeCorner[ CornerToIndex( TriangleCorner ) ];                   // 当前 Corner 对边的邻接 Corner
uint32 LeftCorner = OppositeCorner[ CornerToIndex( NextCorner( TriangleCorner ) ) ];   // 后续从左侧延伸 strip 的邻接 Corner
uint32 RightCorner = OppositeCorner[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];  // 后续从右侧延伸 strip 的邻接 Corner

// 只考虑当前材质段内尚未输出的邻接旧三角形
bool bHasOpposite = Opposite != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( Opposite ) );
bool bHasLeft = LeftCorner != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( LeftCorner ) );
bool bHasRight = RightCorner != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( RightCorner ) );

// 给这个候选 strip 起点评分
int32 Score = NewScoreTriangle( Context, TriangleIndex, true, bHasOpposite, bHasLeft, bHasRight );
if( Score > BestScore )
{
    // 分数更高就更新起点
    StartCorner = TriangleCorner;
    BestScore = Score;
}
else if( Score == BestScore )
{
    // 分数相同时根据 3 个顶点位置和的 X 分量值决定选谁当起点
    float Priority = TrianglePriorities[ TriangleIndex ];
    if( Priority > BestPriority )
    {
        StartCorner = TriangleCorner;
        BestScore = Score;
        BestPriority = Priority;
    }
}
```

如果当前材质段最终没有找到合法作为 strip 起点的旧三角形，此时会结束当前材质段的 stripify：

```cpp
// 没有找到合法 strip 起点, 跳出当前材质段
if( StartCorner == INVALID_CORNER )
    break;
```

而如果最终找到合法作为 strip 起点的旧三角形时，则首先会将其作为 strip 起点输出到 stripify 之后的三角形序列中：

```cpp
{
    // 记录这个输出三角形属于第几个 DWORD
    uint32 TriangleDword = Context.NumTriangles >> 5;
    // 输出前的新顶点序列末尾索引, 用来计算 ref 顶点的 5-bit ref 偏移值
    uint32 BaseVertex = Context.NumVertices - 1;

    // 输出这个三角形
    uint32 NumNewVertices = VisitTriangle(Context, StartCorner, true, false);

    // 为 strip 起点三角形写 ref 顶点的 5-bit ref 偏移值:
    // 如果 NumNewVertices = 3, 则表示 3 个顶点都是新顶点, 不写任何 5-bit ref 偏移值
    // 如果 NumNewVertices = 2, 则表示第 1 个顶点是 ref 顶点, 写 1 个 5-bit ref 偏移值
    // 如果 NumNewVertices = 1, 则表示前 2 个顶点是 ref 顶点, 写 2 个 5-bit ref 偏移值
    // 如果 NumNewVertices = 0, 则表示 3 个顶点都是 ref 顶点, 写 3 个 5-bit ref 偏移值
    if (NumNewVertices < 3)
    {
        uint32 Index = Context.OldToNewVertex[Cluster.Indexes[CornerToIndex(NextCorner(StartCorner))]];
        BitWriter.PutBits(BaseVertex - Index, 5);
    }
    if (NumNewVertices < 2)
    {
        uint32 Index = Context.OldToNewVertex[Cluster.Indexes[CornerToIndex(PrevCorner(StartCorner))]];
        BitWriter.PutBits(BaseVertex - Index, 5);
    }
    if (NumNewVertices < 1)
    {
        uint32 Index = Context.OldToNewVertex[Cluster.Indexes[CornerToIndex(StartCorner)]];
        BitWriter.PutBits(BaseVertex - Index, 5);
    }
    // 统计所在 DWORD 新增顶点数
    NumNewVerticesInDword[TriangleDword] += NumNewVertices;
    // 统计所在 DWORD ref 顶点数
    NumRefVerticesInDword[TriangleDword] += 3u - NumNewVertices;
}
```

在 `VisitTriangle()` 方法中，如果一个旧顶点被编码为一个新顶点，那么 `Context.NumVertices++`，也就是说：**`Context.NumVertices` 中统计的是已经输出到新顶点序列中的顶点数量**。

<!-- 其中 `BaseVertex` 是当前三角形输出前新顶点序列末尾的索引，`Index` 是被引用的新顶点索引 -->

**另外**：根据上面的源码逻辑可以看到，在输出 strip 起点三角形之前，会先记录当前新顶点序列末尾顶点的索引 `BaseVertex`，而在 `VisitTriangle()` 方法中，如果如果一个旧顶点被编码为一个 ref 顶点，那么它引用的就是输出当前这个 strip 起点三角形之前新顶点序列中已经存在的某个顶点，引用的顶点在新顶点序列中的索引可以根据映射 `Context.OldToNewVertex` 获取。那么也就是说：对于一个作为 ref 顶点编码的旧顶点，它所对应的 5-bit ref 偏移值指的是：**从它所属的旧三角形输出前的新顶点序列末尾往前回退多少个位置，这个位置所对应的顶点就是它所引用的顶点**。

找到并输出当前 strip 的起点三角形后，下面就开始从 strip 起点三角形开始延伸扩展当前 strip。

首先，如果当前 strip 已经输出的三角形数量刚好达到 32-triangle DWORD 边界，则跳出当前 strip 的延伸扩展，重新开启一个新的 strip。因为 `StripBitmasks`、`NumNewVerticesInDword` 和 `NumRefVerticesInDword` 都是按 32 个三角形为一个 DWORD 来组织的，这么做可以保证**每个 strip 编码的元数据不会跨 DOWRD 存储，每个 DWORD 开头的三角形总是 strip 起点三角形**：

```cpp
// 从 strip 起点三角形开始延伸扩展 strip
uint32 CurrentCorner = StartCorner;
while (true)
{
    // 如果此时输出三角形数量刚好是 32 的倍数, 那么就停止延伸当前 strip
    // 每个 strip 编码的元数据是 **按 32 个三角形一个 DWORD 分块** 的, 保证 DWORD 开头的三角形总是 strip 起点, 不依赖上一个 DWORD 的延伸状态
    if ((Context.NumTriangles & 31u) == 0u)
        break;

    ...
}
```

对 strip 延伸扩展的核心逻辑就是找当前旧三角形 Corner 的左侧和右侧候选邻接旧三角形 Corner，然后分别给它们打分，最终选择分数更高的一侧进行延伸扩展：

```cpp
// 左侧候选邻接旧三角形 Corner
uint32 LeftCorner = OppositeCorner[CornerToIndex(NextCorner(CurrentCorner))];
// 右侧候选邻接旧三角形 Corner
uint32 RightCorner = OppositeCorner[CornerToIndex(PrevCorner(CurrentCorner))];
CurrentCorner = INVALID_CORNER;

int32 LeftScore = INT_MIN;
// 左侧候选邻接旧三角形在当前材质段内并且尚未输出
if (LeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftCorner)))
{
    // 左侧候选邻接的左侧邻接
    uint32 LeftLeftCorner = OppositeCorner[CornerToIndex(NextCorner(LeftCorner))];
    // 左侧候选邻接的右侧邻接
    uint32 LeftRightCorner = OppositeCorner[CornerToIndex(PrevCorner(LeftCorner))];
    // 左侧候选旧三角形 Corner 是否可以继续往左侧延伸扩展
    bool bLeftLeftCorner = LeftLeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftLeftCorner));
    // 左侧候选旧三角形 Corner 是否可以继续往右侧延伸扩展
    bool bLeftRightCorner = LeftRightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftRightCorner));

    // 给左侧候选邻接旧三角形打分
    LeftScore = NewScoreTriangle(Context, CornerToTriangle(LeftCorner), false, true, bLeftLeftCorner, bLeftRightCorner);
    CurrentCorner = LeftCorner;
}

bool bIsRight = false;
// 右侧候选邻接旧三角形在当前材质段内并且尚未输出
if (RightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightCorner)))
{
    // 右侧候选邻接的左侧邻接
    uint32 RightLeftCorner = OppositeCorner[CornerToIndex(NextCorner(RightCorner))];
    // 右侧候选邻接的右侧邻接
    uint32 RightRightCorner = OppositeCorner[CornerToIndex(PrevCorner(RightCorner))];
    // 右侧候选旧三角形 Corner 是否可以继续往左侧延伸扩展
    bool bRightLeftCorner = RightLeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightLeftCorner));
    // 右侧候选旧三角形 Corner 是否可以继续往右侧延伸扩展
    bool bRightRightCorner = RightRightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightRightCorner));

    // 给右侧候选邻接旧三角形打分
    int32 Score = NewScoreTriangle(Context, CornerToTriangle(RightCorner), false, false, bRightLeftCorner, bRightRightCorner);

    // 根据评分决定延伸扩展方向
    if (Score > LeftScore)
    {
        CurrentCorner = RightCorner;
        bIsRight = true;
    }
}

// 左侧右侧都没有找到可以合法延伸扩展的旧三角形 Corner, 则停止延伸当前 strip
if (CurrentCorner == INVALID_CORNER)
    break;
```

Nanite 通过 `OppositeCorner` 去找左侧或右侧候选邻接旧三角形去扩展延伸 strip，而我们知道 `OppositeCorner[CornerToIndex(CurrentCorner)]` 表示的就是**共享当前旧三角形 `CurrentCorner` 所代表的对边的邻接旧三角形的对应 `Corner`**，所以这里的 `LeftCorner` 和 `RightCorner` 并不是任意候选旧三角形，而是**已经沿着当前 strip 左右两条边接上来的邻接旧三角形**，后面的评分只是决定这次更适合往左侧还是右侧继续延伸。

也就是说，strip 延伸三角形总是沿着当前 strip 的共享边接上，当根据评分选择了要延伸的 `CurrentCorner` 后，按照 `VisitTriangle()` 中使用的局部顶点顺序，`NextCorner(CurrentCorner)` 和 `PrevCorner(CurrentCorner)` 对应的前 2 个旧顶点就是共享边上的两个顶点，`CurrentCorner` 对应的第 3 个旧顶点才是这次延伸新引入的 head 顶点。

因此对于 strip 延伸三角形来说，前 2 个共享边顶点必须编码为 ref 顶点，并且还必须满足 5-bit 偏移约束，否则只能结束当前 strip 的延伸扩展，重新选择新的 strip 起点。

所以 Nanite 在选择了要延伸的 `CurrentCorner` 后，还会预测输出延伸旧三角形后，会不会因为延伸引入新的顶点导致新顶点序列长度增加 1 从而使共享边的 2 个 ref 顶点不再能通过 5-bit ref 偏移值去表示：

```cpp
{
    const uint32 OldIndex0 = Cluster.Indexes[CornerToIndex(NextCorner(CurrentCorner))];
    const uint32 OldIndex1 = Cluster.Indexes[CornerToIndex(PrevCorner(CurrentCorner))];
    const uint32 OldIndex2 = Cluster.Indexes[CornerToIndex(CurrentCorner)];

    const uint32 NewIndex0 = Context.OldToNewVertex[OldIndex0];
    const uint32 NewIndex1 = Context.OldToNewVertex[OldIndex1];
    const uint32 NewIndex2 = Context.OldToNewVertex[OldIndex2];

    // 延伸旧三角形必须沿当前 strip 的边接上, 所以其前 2 个顶点必须作为 ref 顶点编码
    check(NewIndex0 != INVALID_INDEX);
    check(NewIndex1 != INVALID_INDEX);

    // 预估输出这个延伸旧三角形后的新顶点序列中的顶点数量
    const uint32 NextNumVertices = Context.NumVertices + ((NewIndex2 == INVALID_INDEX || Context.NumVertices - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE) ? 1u : 0u);

    // 判断共享边的 2 个 ref 顶点是否仍然满足 5-bit 偏移约束, 如果不满足则结束当前 strip 的延伸扩展, 重新选择新的 strip 起点
    if (NextNumVertices - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ||
        NextNumVertices - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE)
        break;
}
```

最后将选择的延伸旧三角形输出到 stripify 之后的三角形序列中：

```cpp
{
    // 输出这个延伸旧三角形
    uint32 TriangleDword = Context.NumTriangles >> 5;
    uint32 BaseVertex = Context.NumVertices - 1;
    uint32 NumNewVertices = VisitTriangle(Context, CurrentCorner, false, bIsRight);
    // 确保最多只会新增 1 个新顶点
    check(NumNewVertices <= 1u);
    // 如果第 3 个顶点也是 ref 顶点, 则写此 ref 顶点的 5-bit ref 偏移值
    if (NumNewVertices == 0)
    {
        uint32 Index = Context.OldToNewVertex[Cluster.Indexes[CornerToIndex(CurrentCorner)]];
        BitWriter.PutBits(BaseVertex - Index, 5);
    }

    // 统计所在 DWORD 新增顶点数
    NumNewVerticesInDword[TriangleDword] += NumNewVertices;
    // 统计所在 DWORD ref 顶点数
    NumRefVerticesInDword[TriangleDword] += 1u - NumNewVertices;
}

// strip 长度 + 1
StripLength++;
```

以上就是 stripify cluster 的核心逻辑。

整个 stripify 流程结束之后，会先调用 `BitWriter.Flush(sizeof(uint32))` 方法将还未满 8 bits 的已写 bits 作为一个 `uint8` 追加进 `Cluster.StripIndexData` 中，然后还会将 `Cluster.StripIndexData` 以 4 字节对齐：

```cpp
void Flush(uint32 Alignment=1)
{
    // 先把 PendingBits 中还没满 8 bits 的已写 bits 作为一个 uint8 追加进 Cluster.StripIndexData 中
    if (NumPendingBits > 0)
        Buffer.Add((uint8)PendingBits);
    // 如果 Cluster.StripIndexData.Num() 不是 4 的倍数, 就追加 (uint8)0, 直到 Cluster.StripIndexData 满足 4 字节对齐
    while (Buffer.Num() % Alignment != 0)
        Buffer.Add(0);

    // 清空 BitWriter 内部暂存数据
    PendingBits = 0;
    NumPendingBits = 0;
}
```

然后根据 stripify 后的数据重建 cluster 的顶点数据 `Verts`：

```cpp
// 最终 strip 编码产生的新顶点数量, 它可能大于旧顶点数，因为超过 5-bit 偏移约束的 ref 顶点会重新作为新顶点编码
const uint32 NumNewVertices = Context.NumVertices;

// 把原来的顶点数组拷贝到 OldVertices, 并清空 Cluster.Verts
FVertexArray OldVertices( Cluster.Verts.Format );
Swap( OldVertices, Cluster.Verts );

// 按 NewToOldVertex 映射重建 Cluster.Verts: 如果某个旧顶点因为 5-bit 偏移约束被重新作为新顶点编码，那么 NewToOldVertex 中会有多个新索引指向同一个旧顶点；这里会复制相同的 vertex attribute 到多个位置
Cluster.Verts.Reserve( NumNewVertices );
for( uint32 i = 0; i < NumNewVertices; i++ )
{
    Cluster.Verts.Add( &OldVertices.GetPosition( Context.NewToOldVertex[i] ) );
}

// Stripify 不能丢三角形，也不能重复输出三角形。最终输出数量必须等于原 cluster 三角形数
check( Context.NumTriangles == NumOldTriangles );
```

因为在 stripify 的过程中，有些 ref 顶点会因为超出 5-bit 偏移约束从而被重新编码输出成一个新顶点，所以 `Context.NewToOldVertex` 中会存在多个索引指向同一个旧顶点的情况，这也导致最终 `Cluster.Verts` 中会存在多个相同的旧顶点。

重建完新的 `Cluster.Verts` 后，Nanite 会继续写入用于解码 `Cluster.StripIndexData` 数据的描述信息 `Cluster.StripDesc`：

```cpp
// 计算每个 DWORD 之前 **累计** 的新增顶点数:
// DWORD 0 之前 = 0, 不用存
// DWORD 1 之前 = DWORD 0 新增顶点数
// DWORD 2 之前 = DWORD 0 + DWORD 1 新增顶点数
// DWORD 3 之前 = DWORD 0 + DWORD 1 + DWORD 2 新增顶点数
uint32 NumPrevNewVerticesBeforeDwords1 = NumNewVerticesInDword[0];
uint32 NumPrevNewVerticesBeforeDwords2 = NumNewVerticesInDword[1] + NumPrevNewVerticesBeforeDwords1;
uint32 NumPrevNewVerticesBeforeDwords3 = NumNewVerticesInDword[2] + NumPrevNewVerticesBeforeDwords2;
// 每个累计值用 10 bit 存，所以必须小于 1024
check(NumPrevNewVerticesBeforeDwords1 < 1024 && NumPrevNewVerticesBeforeDwords2 < 1024 && NumPrevNewVerticesBeforeDwords3 < 1024);
// 将 3 个累计值编码进 1 个 uint32
StripDesc.NumPrevNewVerticesBeforeDwords = (NumPrevNewVerticesBeforeDwords3 << 20) | (NumPrevNewVerticesBeforeDwords2 << 10) | NumPrevNewVerticesBeforeDwords1;

// 计算每个 DWORD 之前 **累计** 的 ref 顶点数:
// DWORD 0 之前 = 0, 不用存
// DWORD 1 之前 = DWORD 0 ref 顶点数
// DWORD 2 之前 = DWORD 0 + DWORD 1 ref 顶点数
// DWORD 3 之前 = DWORD 0 + DWORD 1 + DWORD 2 ref 顶点数
uint32 NumPrevRefVerticesBeforeDwords1 = NumRefVerticesInDword[0];
uint32 NumPrevRefVerticesBeforeDwords2 = NumRefVerticesInDword[1] + NumPrevRefVerticesBeforeDwords1;
uint32 NumPrevRefVerticesBeforeDwords3 = NumRefVerticesInDword[2] + NumPrevRefVerticesBeforeDwords2;
// 每个累计值用 10 bits 存，所以必须小于 1024
check(NumPrevRefVerticesBeforeDwords1 < 1024 && NumPrevRefVerticesBeforeDwords2 < 1024 && NumPrevRefVerticesBeforeDwords3 < 1024);
// 将 3 个累计值编码进 1 个 uint32
StripDesc.NumPrevRefVerticesBeforeDwords = (NumPrevRefVerticesBeforeDwords3 << 20) | (NumPrevRefVerticesBeforeDwords2 << 10) | NumPrevRefVerticesBeforeDwords1;

// 写入 strip bitmask
static_assert(sizeof(StripDesc.Bitmasks) == sizeof(Context.StripBitmasks), "");
FMemory::Memcpy(StripDesc.Bitmasks, Context.StripBitmasks, sizeof(StripDesc.Bitmasks));
```

在 `Cluster.StripDesc` 中，`Bitmasks` 以 32 个三角形为一组记录每个三角形的 strip 编码元数据；`NumPrevNewVerticesBeforeDwords` 和 `NumPrevRefVerticesBeforeDwords` 则分别记录每个 32-triangle DWORD 之前累计的新顶点数和 ref 顶点数。

其中，`NumPrevNewVerticesBeforeDwords` 从最低位开始，每 10 bits 存储一个 DWORD **之前累计的**新顶点数：由于 DWORD 0 之前的累计值必然为 0，因此不会显式存储；bits `[0, 9]` 记录 DWORD 1 之前的累计新顶点数，也就是 DWORD 0 内所有三角形新增的新顶点数；bits `[10, 19]` 记录 DWORD 2 之前的累计新增顶点数，也就是 DWORD 0 和 DWORD 1 内所有三角形新增的新顶点数；bits `[20, 29]` 记录 DWORD 3 之前的累计新增顶点数。类似的，`NumPrevRefVerticesBeforeDwords` 则是从最低位开始，每 10 bits 存储一个 DWORD 之前累计的 ref 顶点数。

`Bitmasks` 中，`TriangleIndex >> 5` 可以定位三角形所属的 DWORD，通过 `TriangleIndex & 31` 可以定位该三角形在对应 DWORD 内的 bit 位。每组 `Bitmasks` 包含 3 个 `uint32`，这 3 个 `uint32` 中相同 bit 位共同描述一个三角形的 strip 编码信息。

`Bitmasks[][0]` 的对应 bit 表示某个三角形是否为 strip 起点，若该 bit 为 1，则表示该三角形是 strip 起点，此时 `Bitmasks[][1]` 和 `Bitmasks[][2]` 的对应 bit 组合表示该起点三角形中的 ref 顶点数量，范围为 `0..3`，其中 `Bitmasks[][1]` 为高位，`Bitmasks[][2]` 为低位。

若该三角形不是 strip 起点，则表示它是一个 strip 的延伸三角形。此时 `Bitmasks[][1]` 的对应 bit 表示该三角形是左侧还是右侧延伸，`Bitmasks[][2]` 的对应 bit 表示新引入的 head 顶点是否为 ref 顶点。

**需要注意的是**：`Bitmasks` 本身只描述每个三角形的 strip 结构和 ref 顶点或新顶点分布，并不直接存储 ref 顶点的具体索引。具体的 ref 顶点索引以 5-bit ref 偏移值的形式存储在 `Cluster.StripIndexData` 中。解码时会结合 `Bitmasks`、`NumPrevNewVerticesBeforeDwords`、`NumPrevRefVerticesBeforeDwords` 以及当前 DWORD 内之前三角形的 bit 计数，计算出当前三角形的新顶点基准位置 `BaseVertex`，以及需要从 `Cluster.StripIndexData` 读取的 bit offset，随后读取对应的 5-bit ref 偏移值，从而还原完整的三角形索引。

写完 `Cluster.StripDesc` 之后，Nanite 会立刻用刚生成的 strip 编码数据反解一遍三角形索引，并用反解结果重建 `Cluster.Indexes`。这里的 `Indexes` 已经不再引用 stripify 前的旧顶点数组，而是引用上面刚按 `Context.NewToOldVertex` 重建出来的新 `Cluster.Verts`。

这个反解过程的核心逻辑在 `UnpackTriangleIndices()` 方法中。它会以输出后的三角形序号 `TriIndex` 为输入，根据 `StripDesc.Bitmasks` 判断当前三角形是 strip 起点还是 strip 延伸三角形，再结合 `NumPrevNewVerticesBeforeDwords`、`NumPrevRefVerticesBeforeDwords` 和当前 DWORD 内的 bit 计数，算出当前三角形之前已经输出过多少个新顶点和 ref 顶点。之后再从 `StripIndexData` 中读取对应的 5-bit ref 偏移值，最终还原当前三角形的 3 个新顶点序列索引，并写入 `Cluster.Indexes`。

源码中还有一个小细节：`UnpackTriangleIndices()` 和 GPU 侧实现保持 1:1，为了减少分支，它可能会多读一些最终不会使用的 index data。因此 CPU 侧在调用前会给 `Cluster.StripIndexData` 前面补 1 个字节、后面补若干 0，保证 `ReadUnalignedDword()` 做非对齐 DWORD 读取时不会越界：

```cpp
const uint32 PaddedSize = Cluster.StripIndexData.Num() + 5;
TArray<uint8> PaddedStripIndexData;
PaddedStripIndexData.Reserve( PaddedSize );

PaddedStripIndexData.Add( 0 );
PaddedStripIndexData.Append( Cluster.StripIndexData );

// UnpackTriangleIndices is 1:1 with the GPU implementation.
// It can end up over-fetching because it is branchless. The over-fetched data is never actually used.
// On the GPU index data is followed by other page data, so it is safe.

// Here we have to pad to make it safe to perform a DWORD read after the end.
PaddedStripIndexData.SetNumZeroed( PaddedSize );

// Unpack strip
for( uint32 i = 0; i < NumOldTriangles; i++ )
{
    UnpackTriangleIndices( StripDesc, (const uint8*)(PaddedStripIndexData.GetData() + 1), i, &Cluster.Indexes[ i * 3 ] );
}
```

```cpp
static void UnpackTriangleIndices( const FStripDesc& StripDesc, const uint8* StripIndexData, uint32 TriIndex, uint32* OutIndices )
{
    // 当前三角形属于第几个 32-triangle DWORD
    const uint32 DwordIndex = TriIndex >> 5;
    // 当前三角形在 DWORD 内的 bit 位置
    const uint32 BitIndex = TriIndex & 31u;

    // SMask: 某个三角形在 SMask 中对应 bit 位为 1 则表示该三角形是 strip 起点
    const uint32 SMask = StripDesc.Bitmasks[ DwordIndex ][ 0 ];
    // LMask:
    //   - 对于非 strip 起点三角形, 它表示的是该三角形是否是从左侧延伸扩展而来;
    //   - 而对于 strip 起点三角形, 它表示的是该三角形 ref 顶点数的高位 (与 WMask 中的低位组合成 2 bits 记录 strip 起点三角形的 ref 顶点数);
    const uint32 LMask = StripDesc.Bitmasks[ DwordIndex ][ 1 ];
    // WMask:
    //   - 对于非 strip 起点三角形, 它表示该三角形新引入的那个顶点是否是 ref 顶点;
    //   - 而对于 strip 起点三角形, 它表示的是该三角形 ref 顶点数的低位 (与 LMask 中的高位组合成 2 bits 记录 strip 起点三角形的 ref 顶点数);
    const uint32 WMask = StripDesc.Bitmasks[ DwordIndex ][ 2 ];
    // SLMask: 某个三角形在 SLMask 中对应的 bit 位为 1 则表示该三角形既是 strip 起点并且其 ref 顶点数高位为 1, 也就是说该三角形的 ref 顶点数 >= 2
    const uint32 SLMask = SMask & LMask;
    
    //const uint HeadRefVertexMask = ( SMask & LMask & WMask ) | ( ~SMask & WMask );
    // 上面注释是下面代码的展开版本
    // HeadRefVertexMask: 某个三角形的 head 顶点是否是 ref 顶点, 有以下两种情况:
    //   - 对于非 strip 起点三角形, 它的 head 顶点是 ref 顶点则表示此三角形新引入的那个顶点是 ref 顶点, 也就是 WMask = 1;
    //   - 而对于 strip 起点三角形, 它的 head 顶点是 ref 顶点则表示此三角形的 3 个顶点全部都是 ref 顶点, 也就是 ref 顶点数为 3;
    const uint32 HeadRefVertexMask = ( SLMask | ~SMask ) & WMask;   // 1 if head of triangle is ref. S case with 3 refs or L/R case with 1 ref.

    // PrevBitsMask: 当前 32-triangle DWORD 内, 当前三角形之前所有的三角形的 bit mask
    const uint32 PrevBitsMask = ( 1u << BitIndex ) - 1u;
    // 当前 32-triangle DWORD 之前累计的 ref 顶点数
    const uint32 NumPrevRefVerticesBeforeDword = DwordIndex ? BitFieldExtractU32(StripDesc.NumPrevRefVerticesBeforeDwords, 10u, DwordIndex * 10u - 10u) : 0u;
    // 当前 32-triangle DWORD 之前累计的新顶点数
    const uint32 NumPrevNewVerticesBeforeDword = DwordIndex ? BitFieldExtractU32(StripDesc.NumPrevNewVerticesBeforeDwords, 10u, DwordIndex * 10u - 10u) : 0u;

    // 在当前 32-triangle DWORD 内当前三角形之前累计的 ref 顶点数, 其中:
    //   - ( FMath::CountBits( SLMask & PrevBitsMask ) << 1 ) : 累计 strip 起点三角形 ref 顶点数的高位贡献, 因为是高位所以会左移 1 位乘以 2;
    //   - FMath::CountBits( WMask & PrevBitsMask ) : 累计 strip 起点三角形 ref 顶点数的低位贡献, 以及延伸三角形新引入顶点为 ref 顶点时的贡献;
    int32 CurrentDwordNumPrevRefVertices = ( FMath::CountBits( SLMask & PrevBitsMask ) << 1 ) + FMath::CountBits( WMask & PrevBitsMask );
    // 在当前 32-triangle DWORD 内当前三角形之前累计的新顶点数, 其中:
    //   - 首先我们知道: 对于 strip 起点三角形最多有 3 个新顶点; 而对于非 strip 起点三角形, 最多有 1 个新顶点, 所以:
    //     1. BitIndex                                          -> 表示当前三角形之前的每个三角形先按 1 个新顶点算
    //     2. ( FMath::CountBits( SMask & PrevBitsMask ) << 1 ) -> 然后再给每个 strip 起点三角形补 2 个新顶点
    //     3. 最后再减去累计的 ref 顶点数, 最终得到当前三角形之前累计的新顶点数
    int32 CurrentDwordNumPrevNewVertices = ( FMath::CountBits( SMask & PrevBitsMask ) << 1 ) + BitIndex - CurrentDwordNumPrevRefVertices;

    // 当前三角形之前累计的总的 ref 顶点数, 这个数量包括: 当前 32-triangle DWORD 之前累计的 ref 顶点数 以及 当前 32-triangle DWORD 内当前三角形之前累计的 ref 顶点数
    int32 NumPrevRefVertices    = NumPrevRefVerticesBeforeDword + CurrentDwordNumPrevRefVertices;
    // 当前三角形之前累计的总的新顶点数, 这个数量包括: 当前 32-triangle DWORD 之前累计的新顶点数 以及 当前 32-triangle DWORD 内当前三角形之前累计的新顶点数
    int32 NumPrevNewVertices    = NumPrevNewVerticesBeforeDword + CurrentDwordNumPrevNewVertices;

    // 当前三角形是否是 strip 起点三角形
    const int32 IsStart = BitFieldExtractI32( SMask, 1, BitIndex);      // -1: true, 0: false
    // 当前三角形的 LMask bit
    const int32 IsLeft  = BitFieldExtractI32( LMask, 1, BitIndex );     // -1: true, 0: false
    // 当前三角形的 WMask bit
    const int32 IsRef   = BitFieldExtractI32( WMask, 1, BitIndex );     // -1: true, 0: false

    // BaseVertex 是: 当前三角形之前, 最后一个输出的新顶点序列索引, 也就是最近新顶点的索引
    // 而 ref 顶点的 5-bit ref 偏移值编码的是: 被引用顶点相对 BaseVertex 的回退距离
    // 所以这里先算出最近新顶点的索引 BaseVertex
    const uint32 BaseVertex = NumPrevNewVertices - 1u;

    // StripIndexData 里每个 ref 顶点占 5-bit, 对于 strip 起点三角形, 它的第一个 ref 顶点的 5-bit ref 偏移值从 NumPrevRefVertices * 5 开始读
    /*
        因为 ～(-1) = 0, ~(0) = -1, 也就是说 ( NumPrevRefVertices + ~IsStart ) * 5 等价于:

        if (IsStart)
        {
            readOffset = NumPrevRefVertices * 5;
        }
        else
        {
            readOffset = ( NumPrevRefVertices - 1 ) * 5;
        }
    */
    // 对非 strip 起点三角形进行解码时, 可能不仅需要当前三角形自己的 head/ref 偏移值, 还可能需要前一个三角形的 head/ref 偏移值来恢复共享边上的顶点, 所以这里会额外往前读一个 5-bit ref 偏移值
    uint32 IndexData = ReadUnalignedDword( StripIndexData, ( NumPrevRefVertices + ~IsStart ) * 5 ); // -1 if not Start

    // 当前三角形是 strip 起点三角形, strip 起点三角形的 3 个顶点都要独立计算
    if( IsStart )
    {
        // 因为 IsLeft 和 IsRef 的值是 -1: true, 0: false
        // 所以这里是负数形式的 ref 顶点数量
        // 0/1/2/3 个 ref 顶点分别对应的 MinusNumRefVertices 值是 0/-1/-2/-3
        const int32 MinusNumRefVertices = ( IsLeft << 1 ) + IsRef;

        // 下一个可分配的新顶点序列索引
        uint32 NextVertex = NumPrevNewVertices;

        // 如果至少有 1 个 ref 顶点, 解码 OutIndices[ 0 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到新顶点序列索引; 否则 OutIndices[ 0 ] 是一个新顶点, 分配下一个可分配的新顶点序列索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -1 ) { OutIndices[ 0 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 0 ] = NextVertex++; }
        // 至少有 2 个 ref 顶点, 继续解码 OutIndices[ 1 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到新顶点序列索引; 否则 OutIndices[ 1 ] 是一个新顶点, 分配下一个可分配的新顶点序列索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -2 ) { OutIndices[ 1 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 1 ] = NextVertex++; }
        // 3 个顶点都是 ref 顶点, 继续解码 OutIndices[ 2 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到新顶点序列索引; 否则 OutIndices[ 2 ] 是一个新顶点, 分配下一个可分配的新顶点序列索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -3 ) { OutIndices[ 2 ] = BaseVertex - ( IndexData & 31u );                  } else { OutIndices[ 2 ] = NextVertex++; }
    }
    else
    {
        // 当前三角形是非 strip 起点三角形, 也就是说它是 strip 延伸扩展出来的, 它的前 2 个顶点来自已有边, 需要通过 strip 拓扑从前面的三角形推导出来

        // 前一个三角形在当前 DWORD 内的 bit 位置
        const uint32 PrevBitIndex = BitIndex - 1u;
        // 前一个三角形是否是 strip 起点三角形
        const int32 IsPrevStart = BitFieldExtractI32( SMask, 1, PrevBitIndex);
        // 前一个三角形的 head 顶点是否是 ref 顶点
        const int32 IsPrevHeadRef = BitFieldExtractI32( HeadRefVertexMask, 1, PrevBitIndex );
        //const int NumPrevNewVerticesInTriangle = IsPrevStart ? ( 3u - ( bfe_u32( /*SLMask*/ LMask, PrevBitIndex, 1 ) << 1 ) - bfe_u32( /*SMask &*/ WMask, PrevBitIndex, 1 ) ) : /*1u - IsPrevRefVertex*/ 0u;
        // 如果前一个三角形是 strip 起点, 计算它输出了多少个新顶点; 如果前一个三角形是 strip 延伸三角形, 这里不需要做起点三角形的偏移修正, 值为 0
        const int32 NumPrevNewVerticesInTriangle = IsPrevStart & ( 3u - ( (BitFieldExtractU32( /*SLMask*/ LMask, 1, PrevBitIndex) << 1 ) | BitFieldExtractU32( /*SMask &*/ WMask, 1, PrevBitIndex) ) );
        
        //OutIndices[ 1 ] = IsPrevRefVertex ? ( BaseVertex - ( IndexData & 31u ) + NumPrevNewVerticesInTriangle ) : BaseVertex; // BaseVertex = ( NumPrevNewVertices - 1 );
        // 解码 OutIndices[ 1 ]:
        //   - 如果前一个三角形的 head 顶点是 ref 顶点, 则用 ( IndexData & 31u ) 取前一个 ref 偏移值, 再加上 NumPrevNewVerticesInTriangle 做前一个三角形是 strip 起点情况的偏移修正
        //   - 如果前一个三角形的 head 顶点不是 ref 顶点, 则直接使用 BaseVertex
        OutIndices[ 1 ] = BaseVertex + ( IsPrevHeadRef & ( NumPrevNewVerticesInTriangle - ( IndexData & 31u ) ) );
        //OutIndices[ 2 ] = IsRefVertex ? ( BaseVertex - bfe_u32( IndexData, 5, 5 ) ) : NumPrevNewVertices;
        // 解码 OutIndices[ 2 ]:
        //   - 如果当前三角形的 head 顶点是 ref 顶点, 则用 IndexData 的第 2 个 5-bit ref 偏移值解码出其新顶点序列索引
        //   - 如果当前三角形的 head 顶点不是 ref 顶点, 则它就是当前三角形新引入的新顶点, 索引为 NumPrevNewVertices
        OutIndices[ 2 ] = NumPrevNewVertices + ( IsRef & ( -1 - BitFieldExtractU32( IndexData, 5, 5 ) ) );

        // We have to search for the third vertex. 
        // Left triangles search for previous Right/Start. Right triangles search for previous Left/Start.

        // 解码 OutIndices[ 0 ]: 如果是左扩展, 则要找之前最近的右扩展或者 strip 起点三角形; 如果是右扩展, 则需要找之前最近的左扩展或 strip 起点三角形;

        // SMask | ( LMask ^ IsLeft ):
        //   - SMask: 保证 strip 起点三角形总是候选;
        //   - ( LMask ^ IsLeft ): 如果当前三角形是左扩展, LMask ^ (-1) = ~LMask, 找右扩展; 如果当前三角形是右扩展, LMask ^ 0 = LMask, 找左扩展
        const uint32 SearchMask = SMask | ( LMask ^ IsLeft );               // SMask | ( IsRight ? LMask : RMask );

        // 当前三角形之前的候选里找最高 bit 位代表的三角形, 也就是离当前三角形最近的候选三角形
        const uint32 FoundBitIndex = FMath::FloorLog2( SearchMask & PrevBitsMask );
        // 这个候选三角形是不是 strip 起点三角形
        const int32 IsFoundCaseS = BitFieldExtractI32( SMask, 1, FoundBitIndex );       // -1: true, 0: false

        // 当前 32-triangle DWORD 内, 这个候选三角形之前所有的三角形的 bit mask
        const uint32 FoundPrevBitsMask = ( 1u << FoundBitIndex ) - 1u;
        // 在当前 32-triangle DWORD 内候选三角形之前累计的 ref 顶点数
        int32 FoundCurrentDwordNumPrevRefVertices = ( FMath::CountBits( SLMask & FoundPrevBitsMask ) << 1 ) + FMath::CountBits( WMask & FoundPrevBitsMask );
        // 在当前 32-triangle DWORD 内候选三角形之前累计的新顶点数
        int32 FoundCurrentDwordNumPrevNewVertices = ( FMath::CountBits( SMask & FoundPrevBitsMask ) << 1 ) + FoundBitIndex - FoundCurrentDwordNumPrevRefVertices;

        // 候选三角形之前累计的总的新顶点数, 这个数量包括: 当前 32-triangle DWORD 之前累计的新顶点数 以及 当前 32-triangle DWORD 内候选三角形之前累计的新顶点数
        int32 FoundNumPrevNewVertices = NumPrevNewVerticesBeforeDword + FoundCurrentDwordNumPrevNewVertices;
        // 候选三角形之前累计的总的 ref 顶点数, 这个数量包括: 当前 32-triangle DWORD 之前累计的 ref 顶点数 以及 当前 32-triangle DWORD 内候选三角形之前累计的 ref 顶点数
        int32 FoundNumPrevRefVertices = NumPrevRefVerticesBeforeDword + FoundCurrentDwordNumPrevRefVertices;

        // 如果候选三角形是 strip 起点, 这里表示候选三角形的 ref 顶点数量; 如果候选三角形不是 strip 起点, 这个值不按 ref 顶点数量来解释
        const uint32 FoundNumRefVertices = (BitFieldExtractU32( LMask, 1, FoundBitIndex ) << 1 ) + BitFieldExtractU32( WMask, 1, FoundBitIndex );
        // 候选三角形的前一个输出三角形的 head 顶点是否是 ref 顶点
        const uint32 IsBeforeFoundRefVertex = BitFieldExtractU32( HeadRefVertexMask, 1, FoundBitIndex - 1 );

        // ReadOffset: Where is the vertex relative to triangle we searched for?
        // 读取候选三角形的 5-bit ref 偏移值时, StripIndexData 中读取位置的调整:
        //   - 如果候选三角形是 strip 起点且当前三角形是左扩展, 则要读候选三角形的第 2 个 5-bit ref 偏移值, 所以 ReadOffset = -1, 后续 ( FoundNumPrevRefVertices - ReadOffset ) 也就是 ( FoundNumPrevRefVertices + 1 );
        //   - 如果候选三角形是 strip 起点且当前三角形是右扩展, 则读第 1 个 5-bit ref 偏移值, 所以 ReadOffset = 0;
        //   - 如果候选三角形不是 strip 起点, 则读候选三角形前一个 5-bit ref 偏移值, 所以 ReadOffset = 1;
        const int32 ReadOffset = IsFoundCaseS ? IsLeft : 1;
        const uint32 FoundIndexData = ReadUnalignedDword( StripIndexData, ( FoundNumPrevRefVertices - ReadOffset ) * 5 );

        // ( FoundNumPrevNewVertices - 1u ): 是 FoundBaseVertex
        // BitFieldExtractU32( FoundIndexData, 5, 0 ): 是 5-bit ref 偏移值
        // 解码候选三角形中被当前三角形复用的那个 ref 顶点的新顶点序列索引
        const uint32 FoundIndex = ( FoundNumPrevNewVertices - 1u ) - BitFieldExtractU32( FoundIndexData, 5, 0 );

        // 决定是否应该使用 FoundIndex
        // 如果候选三角形是 strip 起点, 则:
        //   - 如果当前三角形是右扩展, 则候选三角形至少要有 1 个 ref 顶点;
        //   - 如果当前三角形是左扩展, 则候选三角形至少要有 2 个 ref 顶点;
        // 而如果候选三角形不是 strip 起点, 则候选三角形的前一个三角形的 head 顶点必须是 ref 顶点
        bool bCondition = IsFoundCaseS ? ( (int32)FoundNumRefVertices >= 1 - IsLeft ) : (IsBeforeFoundRefVertex != 0u);
        // 如果不能用 FoundIndex, 那么应该使用哪个可由 strip 拓扑推导出来的新顶点序列索引:
        //   - 候选三角形是 strip 起点并且没有 ref 顶点时:
        //     - 当前三角形是右扩展, 新顶点序列索引是: FoundNumPrevNewVertices;
        //     - 当前三角形是左扩展, 新顶点序列索引是: FoundNumPrevNewVertices + 1;
        //   - 候选三角形是 strip 起点但有 ref 顶点时, 新顶点序列索引是: FoundNumPrevNewVertices;
        //   - 候选三角形不是 strip 起点, 新顶点序列索引是: FoundNumPrevNewVertices - 1;
        int32 FoundNewVertex = FoundNumPrevNewVertices + ( IsFoundCaseS ? ( IsLeft & ( FoundNumRefVertices == 0 ) ) : -1 );
        // 满足条件就使用 FoundIndex, 否则使用推导的新顶点 FoundNewVertex
        OutIndices[ 0 ] = bCondition ? FoundIndex : FoundNewVertex;

        // 如果当前三角形是左扩展, 那么前面解码出来的第 2, 3 个顶点的顺序需要反过来, 这样才能恢复正确的三角形顶点排列
        if( IsLeft )
        {
            // swap
            std::swap( OutIndices[ 1 ], OutIndices[ 2 ] );
        }
        check(OutIndices[0] != OutIndices[1] && OutIndices[0] != OutIndices[2] && OutIndices[1] != OutIndices[2]);
    }
}
```

简单总结一下 `UnpackTriangleIndices()` 的反解逻辑：

1. 对于 strip 起点三角形，`LMask/WMask` 组合记录的是这个起点三角形有多少个 ref 顶点。ref 顶点从 `StripIndexData` 中读取 5-bit ref 偏移值，并用 `BaseVertex - ref 偏移值` 还原为新顶点序列索引；剩下的顶点则按 `NumPrevNewVertices` 开始顺序分配为新顶点。
2. 对于 strip 延伸三角形，`LMask` 记录延伸方向，`WMask` 记录新引入的 head 顶点是否为 ref 顶点。它的其中两个顶点来自 strip 共享边：一个可以从前一个输出三角形的 head 顶点推导出来，另一个需要根据当前三角形的延伸方向，向前找到最近的反方向延伸三角形或者 strip 起点三角形再推导出来。最后如果当前三角形是左扩展，还需要交换 `OutIndices[1]` 和 `OutIndices[2]`，恢复正确的三角形绕序。

因此，stripify 之后 `Cluster.StripDesc` 和 `Cluster.StripIndexData` 是运行时真正用于解码三角形拓扑的紧凑表示，而这里重建出来的 `Cluster.Indexes` 更像是 CPU 侧的校验和后续构建流程需要的普通 index buffer 视图：它与运行时解码结果保持一致，并且引用的是 stripify 后的新 `Cluster.Verts`。

因为在 stripify 的过程中，有些 ref 顶点会因为超出 5-bit 偏移约束从而被重新编码输出成一个新顶点，也就是说 stripify 之后的顶点数可能会变多，所以 Nanite 最后还会检查 stripify 之后 cluster 的顶点数量是否超过 `NANITE_MAX_CLUSTER_VERTICES` 限制，对于超过限制的 cluster，Nanite 会先将其按三角形数量二分，然后分别对拆分后的 cluster 再次按材质排序分段三角形以及 stripify 的处理，具体逻辑在 `BuildClusterFromClusterTriangleRange()` 方法中：

```cpp
const uint32 NumOldClusters = Clusters.Num();
for( uint32 i = 0; i < NumOldClusters; i++ )
{
    TotalNewTriangles += Clusters[ i ].NumTris;
    TotalNewVertices += Clusters[ i ].Verts.Num();
    
    // Stripify 后 cluster 的顶点数量超过 NANITE_MAX_CLUSTER_VERTICES 限制
    if( Clusters[ i ].Verts.Num() > NANITE_MAX_CLUSTER_VERTICES && Clusters[i].NumTris )
    {
        FCluster ClusterA, ClusterB;
        // 按三角形数量二分
        uint32 NumTrianglesA = Clusters[ i ].NumTris / 2;
        uint32 NumTrianglesB = Clusters[ i ].NumTris - NumTrianglesA;
        // 对二分后的 2 个子 cluster 继续进行按材质排序分段三角形和 stripify 的处理
        BuildClusterFromClusterTriangleRange( Clusters[ i ], ClusterA, 0, NumTrianglesA );
        BuildClusterFromClusterTriangleRange( Clusters[ i ], ClusterB, NumTrianglesA, NumTrianglesB );
        // ClusterA 替代原 cluster
        Clusters[ i ] = ClusterA;
        // ASSEMBLYTODO Many groups might reference this cluster.
        // 将 ClusterB 添加到 Clusters 数组中, 并且将其在 Clusters 数组中的索引添加到原 cluster 所在的 group 中
        ClusterGroups[ ClusterB.GroupIndex ].Children.Add( FClusterRef( Clusters.Num() ) );
        Clusters.Add( ClusterB );
    }
}
```

### 1.4. 为每个材质段切分 Batches

Nanite 下一步会给每个 cluster 的每个材质段连续区间，按**最多 32 个唯一顶点、最多 32 个三角形**的限制切成若干 batch，并把每个 batch 包含的三角形数量写进 `Cluster.MaterialRange.BatchTriCounts` 中：

```cpp
static void BuildVertReuseBatches(FCluster& Cluster)
{
    // 逐个处理 cluster 内的每个材质段连续区间
    for (FMaterialRange& MaterialRange : Cluster.MaterialRanges)
    {
        // 当前材质段内已经出现过的顶点集合, 索引是 cluster 局部顶点索引
        TStaticBitArray<NANITE_MAX_CLUSTER_VERTICES> UsedVertMask;
        // 当前 batch 的唯一顶点数
        uint32 NumUniqueVerts = 0;
        // 当前 batch 的唯一三角形数
        uint32 NumTris = 0;
        // 每个 batch 最多 32 个唯一顶点
        const uint32 MaxBatchVerts = 32;
        // 每个 batch 最多 32 个三角形
        const uint32 MaxBatchTris = 32;
        // 当前材质段末尾三角形索引
        const uint32 TriIndexEnd = MaterialRange.RangeStart + MaterialRange.RangeLength;

        // 清空旧的分 batches 结果
        MaterialRange.BatchTriCounts.Reset();

        // 遍历当前材质段连续区间里的所有三角形
        for (uint32 TriIndex = MaterialRange.RangeStart; TriIndex < TriIndexEnd; ++TriIndex)
        {
            // 当前三角形的 3 个顶点索引
            const uint32 VertIndex0 = Cluster.Indexes[TriIndex * 3 + 0];
            const uint32 VertIndex1 = Cluster.Indexes[TriIndex * 3 + 1];
            const uint32 VertIndex2 = Cluster.Indexes[TriIndex * 3 + 2];

            // 当前 batch 中这 3 个顶点对应的 bit 位标记, 需要注意的是这里的 Bit0/Bit1/Bit2 是引用, 后续会直接赋值
            auto Bit0 = UsedVertMask[VertIndex0];
            auto Bit1 = UsedVertMask[VertIndex1];
            auto Bit2 = UsedVertMask[VertIndex2];

            // 先检查如果添加此三角形后, 当前 batch 的唯一顶点数是否超过限制的 32
            const uint32 NumNewUniqueVerts = uint32(!Bit0) + uint32(!Bit1) + uint32(!Bit2);
            if (NumUniqueVerts + NumNewUniqueVerts > MaxBatchVerts)
            {
                check(NumTris > 0);
                // 记录当前 batch 的三角形数量
                MaterialRange.BatchTriCounts.Add(uint8(NumTris));
                // 重置 batch 状态, 准备开启新 batch
                NumUniqueVerts = 0;
                NumTris = 0;
                UsedVertMask = TStaticBitArray<NANITE_MAX_CLUSTER_VERTICES>();
                --TriIndex;
                continue;
            }

            // 将此三角形添加进当前 batch, 并将其 3 个顶点标记为已使用
            Bit0 = true;
            Bit1 = true;
            Bit2 = true;
            NumUniqueVerts += NumNewUniqueVerts;
            ++NumTris;

            // 检查当前 batch 的三角形数量是否达到限制的 32
            if (NumTris == MaxBatchTris)
            {
                // 记录当前 batch 的三角形数量
                MaterialRange.BatchTriCounts.Add(uint8(NumTris));
                // 重置 batch 状态, 准备开启新 batch
                NumUniqueVerts = 0;
                NumTris = 0;
                UsedVertMask = TStaticBitArray<NANITE_MAX_CLUSTER_VERTICES>();
            }
        }

        // 记录最后一个还未满 32 个三角形的 batch
        if (NumTris > 0)
        {
            MaterialRange.BatchTriCounts.Add(uint8(NumTris));
        }
    }
}
```

### 1.5. 使用全局统一精度量化 Cluster 顶点 Position

首先说说什么是**量化（Quantize）**：量化就是**把连续、高精度的数值，按某个固定精度映射成离散的整数值**。Nanite 通过量化把浮点 Position 映射到可控精度的整数网格上，再转换成 cluster 局部整数偏移，并用尽可能少的 bit 数对顶点 Position 进行紧凑编码。

在 cluster 完成约束、拆分以及 stripify 后，此时每个 cluster 的几何数据已经稳定，Nanite 会**给整个 Nanite 网格选择一个全局统一的 Position 量化精度 `PositionPrecision`**，然后把每个 cluster 的浮点型顶点 Position 量化成 cluster 局部整数偏移，并记录运行时解码需要的元数据。

Nanite 首先会检查当前定义值并做一次最坏情况验证，验证 Nanite 定义的常量值组合本身是否是自洽的：

```cpp
// 计算顶点 Position 的每个轴最多能表示的局部量化值
// 当前 NANITE_MAX_POSITION_QUANTIZATION_BITS = 21, 也就是说每个轴的局部量化值要用 21 bits 表示, 所以允许的最大值是 2^21 - 1
const int32 MaxPositionQuantizedValue   = (1 << NANITE_MAX_POSITION_QUANTIZATION_BITS) - 1;

// 最坏情况检查
{
    // Nanite 使用最大支持的坐标值 NANITE_MAX_COORDINATE_VALUE 和数值最小的 NANITE_MIN_POSITION_PRECISION 算出最大量化整数
    const float MaxValue = FMath::RoundToFloat(NANITE_MAX_COORDINATE_VALUE * FMath::Exp2((float)NANITE_MIN_POSITION_PRECISION));
    // MaxValue <= FLT_INT_MAX: 最大量化整数是否还能安全转成 int32
    // int64(MaxValue) - int64(-MaxValue) <= MaxPositionQuantizedValue: 从负最大到正最大的跨度是否能被 21 bits 表示
    checkf(MaxValue <= FLT_INT_MAX && int64(MaxValue) - int64(-MaxValue) <= MaxPositionQuantizedValue, TEXT("Largest cluster bounds doesn't fit in position bits"));
}
```

然后是确定当前整个 Nanite 网格的全局统一量化精度，首先会取 `Settings.PositionPrecision` 中设置的量化精度，如果选择的是 Auto 模式，也就是 `Settings.PositionPrecision == MIN_int32`，那么 Nanite 会自动估算一个量化精度：

```cpp
// Settings 中设置的量化精度
int32 PositionPrecision = Settings.PositionPrecision;

// 如果选择了 Auto 模式, 则自动估算一个精度, 这里估算精度的逻辑是: 网格越密, 选择的精度应该越高
if (PositionPrecision == MIN_int32)
{
    double TotalLogSize = 0.0;
    int32 TotalNum = 0;
    for (const FCluster& Cluster : Clusters)
    {
        // 用 MipLevel == 0 && NumTris != 0 的 cluster 来估算, 也就是只看最细 LOD 并且有三角形的 cluster
        if (Cluster.MipLevel == 0 && Cluster.NumTris != 0)
        {
            // 取每个 cluster 的包围盒 extent 长度
            float ExtentSize = Cluster.Bounds.GetExtent().Size();
            if (ExtentSize > 0.0)
            {
                // 累加 log2(ExtentSize)
                TotalLogSize += FMath::Log2(ExtentSize);
                // 增加计数
                TotalNum++;
            }
        }
    }

    // 平均 log2(ExtentSize)
    double AvgLogSize = TotalNum > 0 ? TotalLogSize / TotalNum : 0.0;

    // 根据平均 log2(ExtentSize) 计算全局统一量化精度:
    //   - AvgLogSize 越小: 说明 cluster 平均尺寸小, 此时网格密集(更细), 需要使用更高的量化精度, 所以 PositionPrecision 越大
    //   - AvgLogSize 越大: 说明 cluster 平均尺寸大, 此时网格稀疏(更粗), 可以用更低的量化精度, 所以 PositionPrecision 越小
    PositionPrecision = 7 - (int32)FMath::RoundToInt(AvgLogSize);

    
    // 过低的量化精度容易在场景里出问题, 并且节省的磁盘体积很少, 所以 Nanite 在这里限制 Auto 模式下的最小量化精度
    // Auto 模式下不会主动选比 1/16 cm 更低的精度
    const int32 AUTO_MIN_PRECISION = 4; // 1/16cm
    PositionPrecision = FMath::Max(PositionPrecision, AUTO_MIN_PRECISION);
}

// 无论是 Auto 模式自动估算还是指定了具体的量化精度, 最终都 clamp 到 Nanite 支持的范围
PositionPrecision = FMath::Clamp(PositionPrecision, NANITE_MIN_POSITION_PRECISION, NANITE_MAX_POSITION_PRECISION);
```

Auto 模式下自动估算量化精度的核心逻辑是：统计所有最细 LOD 且有三角形的 cluster，并对每个 cluster 的包围盒 extent 长度取 `log2` 后求平均，用这个平均值来大致反映 cluster 的整体尺寸；cluster 尺寸越小，说明当前 cluster 的网格越密集，这时需要使用更高的量化精度；而 cluster 尺寸越大，则说明当前 cluster 的网格越稀疏，此时可以用更低的量化精度。另外 Nanite 发现过低的量化精度更容易在场景里面出问题，并且也只会节省很少的磁盘体积，所以在 Auto 模式下 Nanite 会限制最小量化精度。

Nanite 在量化之前还会检查所有 cluster 在当前选择的量化精度下是否可编码：

```cpp
// 计算量化 scale
float QuantizationScale = FMath::Exp2((float)PositionPrecision);

// 检查所有 cluster 在当前量化精度下是否可编码
for (const FCluster& Cluster : Clusters)
{
    // 跳过没有三角形的 cluster
    if (Cluster.NumTris == 0)
    {
        continue;
    }

    // cluster 的包围盒
    const FBounds3f& Bounds = Cluster.Bounds;
    
    int32 Iterations = 0;
    while (true)
    {
        // 将 cluster 包围盒的 Min 按当前量化精度转换到整数坐标空间
        float MinX = FMath::RoundToFloat(Bounds.Min.X * QuantizationScale);
        float MinY = FMath::RoundToFloat(Bounds.Min.Y * QuantizationScale);
        float MinZ = FMath::RoundToFloat(Bounds.Min.Z * QuantizationScale);

        // 将 cluster 包围盒的 Max 按当前量化精度转换到整数坐标空间
        float MaxX = FMath::RoundToFloat(Bounds.Max.X * QuantizationScale);
        float MaxY = FMath::RoundToFloat(Bounds.Max.Y * QuantizationScale);
        float MaxZ = FMath::RoundToFloat(Bounds.Max.Z * QuantizationScale);

        // 是否可编码的条件:
        // 1. 量化后的 cluster 包围盒的 Min/Max 是否可以安全转换成 int32
        // 2. 量化后的 cluster 包围盒每个轴的局部跨度不能超过 21 bits 可以表示的最大值
        if (MinX >= FLT_INT_MIN && MinY >= FLT_INT_MIN && MinZ >= FLT_INT_MIN &&
            MaxX <= FLT_INT_MAX && MaxY <= FLT_INT_MAX && MaxZ <= FLT_INT_MAX &&
            ((int64)MaxX - (int64)MinX) <= MaxPositionQuantizedValue && ((int64)MaxY - (int64)MinY) <= MaxPositionQuantizedValue && ((int64)MaxZ - (int64)MinZ) <= MaxPositionQuantizedValue)
        {
            break;
        }
        
        // 如果不可编码, 就把全局统一量化精度 -1, 直到所有 cluster 满足编码条件
        QuantizationScale *= 0.5f;
        PositionPrecision--;
        check(PositionPrecision >= NANITE_MIN_POSITION_PRECISION);
        check(++Iterations < 100);  // Endless loop?
    }
}
```

Nanite 会将每个 cluster 包围盒的 Min/Max Position 按当前全局统一量化精度映射到整数坐标空间，并检查该整数空间下的包围盒：只有当量化后的 Min/Max Position 值可以安全转换成 `int32`，并且每个轴上的局部跨度都不超过每轴 21 bits 可表示的最大值时，才认为当前 cluster 在当前全局统一量化精度下是可编码的。如果发现某些 cluster 不可编码，就降低全局统一量化精度，直到所有非空 cluster 都满足可编码条件。

确定好最终的全局统一量化精度后，Nanite 会开始并行量化每个 cluster 的顶点 Position：

```cpp
// 量化 scale 的倒数, 用于将量化后的整数 Position 值反量化为量化网格上的浮点 Position 值
const float RcpQuantizationScale = 1.0f / QuantizationScale;

// 初始化 Cluster.QuantizedPositions
const uint32 NumClusterVerts = Cluster.Verts.Num();
Cluster.QuantizedPositions.SetNumUninitialized(NumClusterVerts);

// 初始化当前 cluster 的整数包围盒
FIntVector IntClusterMax = { MIN_int32, MIN_int32, MIN_int32 };
FIntVector IntClusterMin = { MAX_int32, MAX_int32, MAX_int32 };

for (uint32 i = 0; i < NumClusterVerts; i++)
{
    // 读取浮点 Position
    const FVector3f Position = Cluster.Verts.GetPosition(i);

    FIntVector& IntPosition = Cluster.QuantizedPositions[i];

    // 将浮点 Position 的每个轴量化到整数坐标空间
    float PosX = FMath::RoundToFloat(Position.X * QuantizationScale);
    float PosY = FMath::RoundToFloat(Position.Y * QuantizationScale);
    float PosZ = FMath::RoundToFloat(Position.Z * QuantizationScale);

    // 先写入全局量化网格下的绝对整数 Position 值, 后续再减去 IntClusterMin 转成 cluster 局部整数偏移
    IntPosition = FIntVector((int32)PosX, (int32)PosY, (int32)PosZ);

    // 更新当前 cluster 包围盒每个轴的整数 Min/Max
    IntClusterMax.X = FMath::Max(IntClusterMax.X, IntPosition.X);
    IntClusterMax.Y = FMath::Max(IntClusterMax.Y, IntPosition.Y);
    IntClusterMax.Z = FMath::Max(IntClusterMax.Z, IntPosition.Z);
    IntClusterMin.X = FMath::Min(IntClusterMin.X, IntPosition.X);
    IntClusterMin.Y = FMath::Min(IntClusterMin.Y, IntPosition.Y);
    IntClusterMin.Z = FMath::Min(IntClusterMin.Z, IntPosition.Z);
}

// 根据包围盒每个轴的整数 Min/Max 计算每个轴的跨度需要的 bits 数
const uint32 NumBitsX = FMath::CeilLogTwo(IntClusterMax.X - IntClusterMin.X + 1);
const uint32 NumBitsY = FMath::CeilLogTwo(IntClusterMax.Y - IntClusterMin.Y + 1);
const uint32 NumBitsZ = FMath::CeilLogTwo(IntClusterMax.Z - IntClusterMin.Z + 1);
// 检查每个轴需要的 bits 数没有超过 NANITE_MAX_POSITION_QUANTIZATION_BITS
check(NumBitsX <= NANITE_MAX_POSITION_QUANTIZATION_BITS);
check(NumBitsY <= NANITE_MAX_POSITION_QUANTIZATION_BITS);
check(NumBitsZ <= NANITE_MAX_POSITION_QUANTIZATION_BITS);

// 第二次遍历, 回写量化后的浮点 Position, 并转成局部整数偏移
for (uint32 i = 0; i < NumClusterVerts; i++)
{
    FIntVector& IntPosition = Cluster.QuantizedPositions[i];

    // 将 Verts 中的浮点 Position 值更新成量化后的绝对浮点 Position 值
    Cluster.Verts.GetPosition(i) = FVector3f((float)IntPosition.X * RcpQuantizationScale, (float)IntPosition.Y * RcpQuantizationScale, (float)IntPosition.Z * RcpQuantizationScale);
    
    // 把绝对量化整数 Position 值转换成相对于 IntClusterMin 的 cluster 局部整数偏移
    IntPosition.X -= IntClusterMin.X;
    IntPosition.Y -= IntClusterMin.Y;
    IntPosition.Z -= IntClusterMin.Z;

    // 检查每个轴相对于 IntClusterMin 的局部整数偏移非负, 并且在当前轴的 bit 数可表示范围内
    check(IntPosition.X >= 0 && IntPosition.X < (1 << NumBitsX));
    check(IntPosition.Y >= 0 && IntPosition.Y < (1 << NumBitsY));
    check(IntPosition.Z >= 0 && IntPosition.Z < (1 << NumBitsZ));
}

// 同样将包围盒更新成量化后的绝对浮点 Position 值
Cluster.Bounds.Min = FVector3f((float)IntClusterMin.X * RcpQuantizationScale, (float)IntClusterMin.Y * RcpQuantizationScale, (float)IntClusterMin.Z * RcpQuantizationScale);
Cluster.Bounds.Max = FVector3f((float)IntClusterMax.X * RcpQuantizationScale, (float)IntClusterMax.Y * RcpQuantizationScale, (float)IntClusterMax.Z * RcpQuantizationScale);

// 记录每个轴保存局部整数偏移需要多少 bit 位
Cluster.QuantizedPosBits = FIntVector(NumBitsX, NumBitsY, NumBitsZ);
// 记录当前 cluster 量化整数包围盒的 Min，也就是局部整数偏移的起点 QuantizedPosStart
Cluster.QuantizedPosStart = IntClusterMin;
// 记录全局统一量化精度
Cluster.QuantizedPosPrecision = PositionPrecision;
```

Nanite 先将当前 cluster 中每个顶点的浮点 Position 值按最终确定的全局 `PositionPrecision` 量化成绝对整数 Position 值，并在这个过程中统计当前 cluster 在整数坐标空间下的 Min/Max；接着根据整数 Min/Max 计算每个轴保存局部整数偏移所需的 bit 数，然后将 `Cluster.QuantizedPositions` 中每个顶点的绝对整数 Position 值减去 `IntClusterMin`，也就是说：`Cluster.QuantizedPositions` 中记录的并不是量化后的绝对整数 Position 值，而是相对于 `IntClusterMin` 的 cluster 局部整数偏移；最后 Nanite 还会把 `Cluster.Verts` 和 `Cluster.Bounds` 更新成量化后的浮点 Position 值，并记录 `Cluster.QuantizedPosBits`、`Cluster.QuantizedPosStart` 和 `Cluster.QuantizedPosPrecision`，供后续编码和运行时解码使用。

### 1.6. 量化 Cluster 顶点 BoneWeight

如果 cluster 顶点格式中包含 BoneInfluence 数据，Nanite 还会在编码前对每个顶点的 BoneWeight 进行量化处理。这里需要注意，`BoneWeightPrecision` 控制的是 BoneWeight 的量化精度，并不控制 BoneIndex 的编码精度；BoneIndex 后续会根据 cluster 中实际使用到的最大 BoneIndex 单独计算需要的 bit 数。

Nanite 会先确定 BoneWeight 的量化精度。当 `Settings.BoneWeightPrecision < 0` 时表示 Auto 模式，此时默认使用 8 bits 来量化 BoneWeight；否则使用 `Settings.BoneWeightPrecision` 指定的 bit 数，并将其限制在 `[0, NANITE_MAX_BLEND_WEIGHT_BITS]` 范围内：

```cpp
BoneWeightPrecision = (Settings.BoneWeightPrecision < 0) ? 8u : (int32)FMath::Clamp(Settings.BoneWeightPrecision, 0, NANITE_MAX_BLEND_WEIGHT_BITS);
```

然后根据量化精度计算量化后的目标总权重，并逐顶点处理它们的 `BoneInfluences`：

```cpp
static void QuantizeBoneWeights(FCluster& Cluster, int32 BoneWeightPrecision)
{
    // 当前 cluster 顶点数
    const uint32 NumVerts           = Cluster.Verts.Num();
    // 当前 cluster 顶点格式中最多支持的 BoneInfluence 数量
    const uint32 NumBoneInfluences  = Cluster.Verts.Format.NumBoneInfluences;

    // 根据量化精度计算量化后的目标总权重, 如果 BoneWeightPrecision = 8, 那么 TargetTotalBoneWeight = (1 << 8) - 1 = 255
    const uint32 TargetTotalBoneWeight = BoneWeightPrecision ? ((1u << BoneWeightPrecision) - 1u) : 1u;

    // 遍历 cluster 内的每个顶点
    for (uint32 VertIndex = 0; VertIndex < NumVerts; VertIndex++)
    {
        // 当前顶点的 BoneInfluences 数组, 其中每个 BoneInfluence 的 X 是 BoneIndex, Y 是 BoneWeight
        FVector2f* BoneInfluences = Cluster.Verts.GetBoneInfluences(VertIndex);

        // 处理每个顶点的 BoneInfluences
        QuantizeAndSortBoneInfluenceWeights(TArrayView<FVector2f>(BoneInfluences, NumBoneInfluences), TargetTotalBoneWeight);
    }
}
```

`QuantizeBoneWeights()` 本身只负责逐顶点取出 `BoneInfluences`，真正处理单个顶点 BoneWeight 的逻辑在 `QuantizeAndSortBoneInfluenceWeights()` 中：它会先调用 `QuantizeWeights()` 得到量化后的整数权重，然后再将结果写回 `BoneInfluence.Y`，接着清理权重为 0 的 BoneIndex，并重新排序当前顶点的 BoneInfluences：

```cpp
void QuantizeAndSortBoneInfluenceWeights(TArrayView<FVector2f> BoneInfluences, uint32 TargetTotalQuantizedWeight)
{
    const uint32 NumBoneInfluences = BoneInfluences.Num();
    TArray<uint32, TInlineAllocator<64>> QuantizedWeights;

    // 量化顶点 BoneWeight
    QuantizeWeights(NumBoneInfluences, TargetTotalQuantizedWeight, QuantizedWeights, [BoneInfluences](uint32 Index) -> float
        {
            return BoneInfluences[Index].Y;
        });

    for (uint32 i = 0; i < NumBoneInfluences; ++i)
    {
        // 把量化后的整数 BoneWeight 写回 BoneInfluence.Y
        BoneInfluences[i].Y = (float)QuantizedWeights[i];
        
        // 如果某个 BoneInfluence 的量化 BoneWeight 为 0, 就把其 BoneIndex 也清成 0
        if (QuantizedWeights[i] == 0)
        {
            BoneInfluences[i].X = 0.0f; // Clear index when weight is 0
        }
    }

    // 对顶点的 BoneInfluences 进行排序: BoneWeight 大的排前面, BoneWeight 相同时 BoneIndex 大的排前面
    // 后续 CalculateInfluences() 方法中会从前往后读取当前顶点的 BoneInfluences，一旦遇到量化后的 BoneWeight 为 0，就认为后续 BoneInfluence 都无效，并直接跳出遍历
    BoneInfluences.Sort([](const FVector2f& A, const FVector2f& B) { return A.Y > B.Y || (A.Y == B.Y && A.X > B.X); });
}
```

重排序当前顶点的 BoneInfluences 遵循：BoneWeight 大的排前面，BoneWeight 相同时 BoneIndex 大的排前面。这里重排序的主要目的，是配合后续 `CalculateInfluences()` 的逻辑遍历：`CalculateInfluences()` 会从前往后读取当前顶点的 BoneInfluences，一旦遇到量化后的 BoneWeight 为 0，就认为后续 BoneInfluence 都无效，并直接跳出遍历。

最后看看真正执行 BoneWeight 量化和误差修正的 `QuantizeWeights()`：

```cpp
// 累加原始 BoneWeight 总和
float TotalWeight = 0.0f;
for (uint32 i = 0; i < N; i++)
{
    TotalWeight += (float)GetWeight(i);
}

// 如果当前这组权重总和接近 0，则无法按比例归一化，直接将所有量化权重置 0
if (FMath::IsNearlyZero(TotalWeight))
{
    // bail early on zero total weight
    QuantizedWeights.SetNumZeroed(N);
    return;
}

// FHeapEntry 用于后续修正已量化整数 BoneWeight 的四舍五入误差
struct FHeapEntry
{
    float Error;    // 某个已量化整数 BoneWeight 的四舍五入误差
    uint32 Index;   // BoneWeight 索引
};

TArray<FHeapEntry, TInlineAllocator<64>> ErrorHeap;
QuantizedWeights.SetNum(N);

// 当前已量化整数 BoneWeight 总和
uint32 TotalQuantizedWeight = 0;
// 逐个 BoneWeight 量化
for (uint32 i = 0; i < N; i++)
{
    // 把原始 BoneWeight 归一化后映射到目标整数区间
    const float Weight = ((float)GetWeight(i) * (float)TargetTotalQuantizedWeight) / TotalWeight;
    // 再四舍五入到整数
    const uint32 QuantizedWeight = FMath::RoundToInt(Weight);
    // 写入 QuantizedWeights
    QuantizedWeights[i] = QuantizedWeight;
    // 并记录本次四舍五入误差
    ErrorHeap.Emplace(FHeapEntry{ (float)QuantizedWeight - Weight, i });
    // 累加已量化整数 BoneWeight 总和
    TotalQuantizedWeight += QuantizedWeight;
}

// 如果当前已量化整数 BoneWeight 总和不等于目标总 BoneWeight, 则进行误差修正
if (TotalQuantizedWeight != TargetTotalQuantizedWeight)
{
    // 当前已量化整数 BoneWeight 总和是偏小还是偏大
    const bool bTooSmall = (TotalQuantizedWeight < TargetTotalQuantizedWeight);
    // 总和偏小, 后面每次修正给某个权重 +1; 总和偏大, 后面每次修正给某个权重 -1;
    const int32 Diff = bTooSmall ? 1 : -1;

    // ErrorHeap 中 FHeapEntry 的比较规则
    auto Predicate = [bTooSmall](const FHeapEntry& A, const FHeapEntry& B)
        {
            if (bTooSmall)
            {
                // 如果是总和偏小: 误差小的排前面, 误差小的最适合权重 +1
                return (A.Error != B.Error) ? (A.Error < B.Error) : (A.Index < B.Index);
            }
            else
            {
                // 而如果是总和偏大: 误差大的排前面, 误差大的最适合权重 -1
                return (A.Error != B.Error) ? (A.Error > B.Error) : (A.Index > B.Index);
            }
        };

    // 构建堆, 每次取出最适合修正的已量化整数 BoneWeight
    ErrorHeap.Heapify(Predicate);

    // 误差修正, 直到已量化整数 BoneWeight 总和等于目标总 BoneWeight
    while (TotalQuantizedWeight != TargetTotalQuantizedWeight)
    {
        // 确保堆中还有元素可取
        check(ErrorHeap.Num() > 0);

        // 每次取出当前堆中最适合修正的已量化整数 BoneWeight
        FHeapEntry Entry;
        ErrorHeap.HeapPop(Entry, Predicate, EAllowShrinking::No);
            
        // 进行修正
        QuantizedWeights[Entry.Index] += Diff;
        TotalQuantizedWeight += Diff;
    }
}
```

Nanite 首先累加顶点 BoneInfluences 中的原始 BoneWeight 总和，然后根据原始 BoneWeight 总和对每个 BoneWeight 进行归一化并映射到目标总 BoneWeight 区间，再对其进行四舍五入后写入 `QuantizedWeights`，同时记录每次四舍五入后产生的有符号误差；

将所有 BoneWeight 量化结束后，检查当前已量化整数 BoneWeight 总和是否等于目标总 BoneWeight，如果不相等则还需要进行误差修正，保证所有已量化整数 BoneWeight 总和等于目标总 BoneWeight。修正的核心逻辑是：首先根据当前已量化 BoneWeight 总和是大于还是小于目标总 BoneWeight，如果总和偏小，则后续修正需要增加已量化整数 BoneWeight 值；而如果总和偏大，则需要减少已量化整数 BoneWeight 值。

修正之前会先根据比较结果将已记录的每个 BoneWeight 量化的误差记录建堆，总和偏小时，误差小的优先；总和偏大时，误差大的优先；

修正时，每次只从堆中取一个最优先的已量化整数 BoneWeight，并根据比较结果对其进行 +1 或 -1，直到当前已量化整数 BoneWeight 总和等于目标总 BoneWeight。

## 2. 编码 Cluster DAG

进过前面的规范化处理后，cluster 数据已经被整理成适合编码的标准格式。接下来 Nanite 会开始为每个 cluster 计算具体的编码信息和 GPU 数据尺寸，并基于这些信息完成 page 分配、层级结构构建、page 依赖关系计算以及最终将 page 数据写入磁盘。

### 2.1. 计算 Cluster 编码信息和 GPU Page 数据大小

Nanite 为每个 cluster 计算它们的编码信息 `FEncodingInfo`。`FEncodingInfo` 中主要包含 3 类数据：**首先是编码 bit 数和量化精度**，例如单个顶点索引 bit 数 `BitsPerIndex`、每个顶点所有属性的总 bit 数 `BitsPerAttribute`、法线 octahedron 编码每个分量的 bit 数 `NormalPrecision`，以及切线角度量化 bit 数 `TangentPrecision`；**其次是顶点属性的解码元数据**，例如顶点颜色解码元数据 `ColorMode`、`ColorMin` 和 `ColorBits`，顶点 UV 解码元数据 `UVs`，以及骨骼解码元数据 `BoneInfluence`；**最后是 `GpuSizes`，它记录当前 cluster 写进 GPU page 时，各个数据段需要多少字节**。

Nanite 首先计算运行时 GPU page 中索引数据的布局。需要注意的是，**这里计算的是 cluster 加载到 GPU Page 后，三角形索引数据在运行时数据结构中占用的固定大小，而不是磁盘上 `StripIndexData` 的实际压缩大小**：

```cpp
// 根据当前 cluster 顶点数量, 计算运行时 GPU page 中单个顶点索引值至少需要多少 bit 数
const uint32 BitsPerIndex = NumClusterVerts > 1 && NumClusterTris > 1 ? (FGenericPlatformMath::FloorLog2(NumClusterVerts - 1) + 1) : 1;
// Cluster 被加载到 GPU page 时, 磁盘上的 Strip 索引会被解码并重新写成: 单个顶点索引值 + 两个 5-bit 顶点索引偏移值
const uint32 BitsPerTriangle = BitsPerIndex + 2 * 5;    // Base index + two 5-bit offsets
// 将运行时 GPU page 中单个顶点索引的 bit 数保存到 FEncodingInfo
Info.BitsPerIndex = BitsPerIndex;
```

根据当前 cluster 的总顶点数可以知道其最大顶点索引值，通过最大索引值就可以确定运行时 GPU page 中单个顶点索引值至少所需的 bit 数。例如当前 cluster 一共有 128 个顶点，那么其最大顶点索引值就是 127，也就表示单个顶点索引值至少需要 7 bits。

Cluster 加载到 GPU page 后会使用固定长度的三角形索引结构：1 个基础顶点索引 `BaseIndex`，加上 2 个相对 `BaseIndex` 的 5-bit 顶点索引偏移值。因此，在 GPU page 中单个三角形索引数据需要 `BitsPerIndex + 2 * 5` bits。

然后在 `Info.GpuSizes` 中记录 GPU page 中各个数据段大小：

```cpp
FPageSections& GpuSizes = Info.GpuSizes;
// 每个 cluster 的固定头信息大小
GpuSizes.Cluster = sizeof(FPackedCluster);
// 材质表大小: 如果 cluster 材质段数量超过 3, 此时材质段信息不能通过 fast path 内联表达, 需要额外的材质表数据
GpuSizes.MaterialTable = CalcMaterialTableSize(Cluster) * sizeof(uint32);
// 材质段 batch 信息大小: 如果当前 cluster 存在三角形且材质段数量超过 3, 则需要额外的材质段 batch 信息; 否则为 0
GpuSizes.VertReuseBatchInfo = Cluster.NumTris && Cluster.MaterialRanges.Num() > 3 ? CalcVertReuseBatchInfoSize(Cluster.MaterialRanges) * sizeof(uint32) : 0;
// DecodeInfo 中保存解码所需的辅助头信息: 每个 UV 通道的解码头信; 如果有骨骼数据, 还需要 BoneInfluence 的解码头信息
GpuSizes.DecodeInfo = Cluster.Verts.Format.NumTexCoords * sizeof(FPackedUVHeader) + (MaxBones > 0 ? sizeof(FPackedBoneInfluenceHeader) : 0);
// GPU page 中三角形索引 bitstream 的字节数: 每个三角形占 BitsPerTriangle bits, 向上取整到 32-bit 边界
GpuSizes.Index = (NumClusterTris * BitsPerTriangle + 31) / 32 * 4;
```

这里额外分别说说 `MaterialTable` 和 `VertReuseBatchInfo` 编码的 slow path 和 fast path：

先来看看材质表（`MaterialTable`）。**对于材质段数量不超过 3 的 cluster，Nanite 会将其材质段信息直接编码进 `PackedCluster.PackedMaterialInfo` 这个 `uint32` 中**，Nanite 将其叫做 fast path：

```cpp
PackedMaterialInfo = PackMaterialFastPath(Material0Length, Material0Index, Material1Length, Material1Index, Material2Index);
```

因为 Nanite 中每个 cluster 的最大材质数量是 `64 (NANITE_MAX_CLUSTER_MATERIALS)`，也就是说每个 cluster 使用到的材质的索引范围是 `[0, 63]`，那么每个材质段中记录的材质索引用使用 6 bits 就可以编码：

```cpp
// 每个材质段中记录的材质索引小于 64
check(Material0Index  <  64);
check(Material1Index  <  64);
check(Material2Index  <  64);
// 每个材质索引各编码进 6 bits 中
Packed |= Material0Index;
Packed |= Material1Index << 6;
Packed |= Material2Index << 12;
```

对于每个材质段的长度，首先我们知道，Nanite 中每个 cluster 的最大三角形数量是 `128 (NANITE_MAX_CLUSTER_TRIANGLES)`，另外在前面构建材质段信息时，**保证了每个材质段中的三角形数量大于 0，并且每个 cluster 中的所有材质段有按照其三角形数量进行降序排序**。那么先来看看 `Material0Length`，它的取值范围就是 `[1, 128]`，那么 `Material0Length - 1` 刚好可以编码进 7 bits 中：

```cpp
// Material0Length 的范围是 [1, 128]
check(Material0Length >= 1);
check(Material0Length <= 128);
// 所以 (Material0Length - 1) 刚好可以编码进 7 bits 中
Packed |= (Material0Length - 1u) << 18;
```

又因为材质段会按其三角形数量降序排序，所以 `Material1Length` 一定满足 `Material1Length <= Material0Length`，另外如果 `Material1Length` 大于 64，那么 `Material0Length` 也必须大于 64，此时前 2 个材质段中三角形数量总和已经超过 128，这与 Nanite 定义的每个 cluster 最多 128 个三角形矛盾，所以 `Material1Length` 的范围应该是 `[1, 64]`，也是可以编码进 7 bits 中：

```cpp
// 因为前面的排序逻辑, 所以 Material1Length 一定满足: Material1Length <= Material0Length
// 又因为 cluster 最大 128 个三角形, 计算可得 Material1Length <= 64
check(Material1Length <= 64);
check(Material1Length <= Material0Length);
// 对于 [1, 64] 的 Material1Length, 将其编码进最后的 7 bits 中
Packed |= Material1Length << 25;
```

对于 `Material2Length` 则可以不编码，因为它可以通过总三角形数量与前 2 个材质段中三角形数量反推出来：

```cpp
Material2Length = Cluster.MaterialIndexes.Num() - Material0Length - Material1Length;
```

而**对于材质段数量超过 3 的 cluster，Nanite 会将其每个材质段信息编码进全局的材质表中，并在 `PackedCluster.PackedMaterialInfo` 这个 `uint32` 中编码前 cluster 的材质段信息在运行时 GPU page 的材质表数据段中的起始 DWORD 偏移和材质段数量**，Nanite 将其叫做 slow path：

```cpp
// 把当前 GPU page 中材质表数据段的起始 byte 偏移转换成 DWORD 偏移
const uint32 MaterialTableStartOffsetInDwords = Page.GpuSizes.GetMaterialTableOffset() >> 2;
```

首先把当前 GPU page 中材质表数据段的起始 byte 偏移转换成 DWORD 偏移，然后在 `PackMaterialInfo()` 方法中，先将每个材质段信息编码进一个 `uint32`，并添加进 `Streams.MaterialRange` 数组中：

```cpp
// 当前 GPU page 中材质表数据段的起始 DWORD 偏移
uint32 MaterialTableOffset = OutMaterialTable.Num() + MaterialTableStartOffset;
// 当前 cluster 材质段数量
uint32 MaterialTableLength = Cluster.MaterialRanges.Num();
check(MaterialTableLength > 0);

// 将每个材质段信息编码进一个 `uint32`，并添加进 `Streams.MaterialRange` 数组中
for (int32 RangeIndex = 0; RangeIndex < Cluster.MaterialRanges.Num(); ++RangeIndex)
{
    const FMaterialRange& Material = Cluster.MaterialRanges[RangeIndex];
    OutMaterialTable.Add(PackMaterialTableRange(Material.RangeStart, Material.RangeLength, Material.MaterialIndex));
}
```

也就是说 `Streams.MaterialRange` 中的每个 `uint32` 表示的是一个材质段信息，从低位开始，bit 0 到 bit 7 编码 `RangeStart`；bit 8 到 bit 15编码 `RangeLength`；bit 16 到 bit 21 编码 `MaterialIndex`，剩下的 10 bits 是 Padding。

最后把当前 cluster 的材质段信息在运行时 GPU page 的材质表数据段中的起始 DWORD 偏移和材质段数量编码进 `PackedCluster.PackedMaterialInfo` 这个 `uint32` 中：

```cpp
PackedMaterialInfo = PackMaterialSlowPath(MaterialTableOffset, MaterialTableLength);
```

```cpp
// 19 bits 编码起始 DWORD 偏移
uint32 Packed = MaterialTableOffset;
// 6 bits 编码材质段数量-1
Packed |= (MaterialTableLength - 1u) << 19;
// 高 7 bits 编码 slow path 标记, 这里是吧 bit 25 到 bit 31 全部设置为 1
Packed |= (0xFE000000u);
```

需要注意的是，Nanite 在这里将 `PackedCluster.PackedMaterialInfo` 的高 7 bits 全部设置为 1 作为 slow path 的标记。因为当 cluster 的材质段数量不超过 3 时，fast path 会直接将 3 个材质段的信息编码进 `PackedCluster.PackedMaterialInfo` 中，此时其高 7 bits 编码的是 `Material1Length`，这个值最大只会是 64。所以在运行时 shader 在解码时可以通过一个阈值判断知道当前 cluster 的材质段信息编码是走的 slow path 还是 fast path：

```hlsl
if (MaterialEncoding < 0xFE000000u)
    // Fast path decoding
else
    // Slow path decoding
```

再来看看材质段 batch 信息（`VertReuseBatchInfo`）。首先来来看看 `CalcVertReuseBatchInfoSize()` 方法，它的核心逻辑是计算编码当前 cluster 的材质段 batch 信息需要多少个 `uint32`：

```cpp
uint32 CalcVertReuseBatchInfoSize(const TArrayView<const FMaterialRange>& MaterialRanges)
{
    // 每个材质段的 batch 数量用 4 bits 编码
    constexpr int32 NumBatchCountBits = 4;
    // 每个 batch 的三角形数量用 5 bits 编码
    constexpr int32 NumTriCountBits = 5;
    // 最坏情况下一个 batch 中的三角形数量, 因为构建 batch 时要求//todo
    constexpr int32 WorstCaseFullBatchTriCount = 10;

    int32 TotalNumBatches = 0;
    int32 NumBitsNeeded = 0;

    for (const FMaterialRange& MaterialRange : MaterialRanges)
    {
        const int32 NumBatches = MaterialRange.BatchTriCounts.Num();
        check(NumBatches > 0 && NumBatches < (1 << NumBatchCountBits));
        TotalNumBatches += NumBatches;
        NumBitsNeeded += NumBatchCountBits + NumBatches * NumTriCountBits;
    }
    NumBitsNeeded += FMath::Max(NumBatchCountBits * (3 - MaterialRanges.Num()), 0);
    check(TotalNumBatches < FMath::DivideAndRoundUp(NANITE_MAX_CLUSTER_TRIANGLES, WorstCaseFullBatchTriCount) + MaterialRanges.Num() - 1);

    return FMath::DivideAndRoundUp(NumBitsNeeded, 32);
}
```

**对于材质段数量不超过 3 的 cluster，Nanite 会将其材质段 batch 信息直接编码进 `PackedCluster.VertReuseBatchInfo` 这个长度为 4 的 `uint32` 数组中**，

而对于材质段数量超过 3 的 cluster，Nanite 会将

---

接下来 Nanite 继续计算顶点数据相关的编码信息。

## 3. References

- [Nanite: A Deep Dive](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)
- [GAMES 104: GPU-Driven Geometry Pipeline - Nanite](https://www.piccoloengine.com/merch/8)
