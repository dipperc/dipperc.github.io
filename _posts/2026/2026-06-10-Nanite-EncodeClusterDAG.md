---
layout: post
title:  "Nanite - Encode Cluster DAG"
date:   2026-06-10 20:18:00 +800
category: Unreal Engine
---

- [1. 准备编码数据](#1-准备编码数据)
  - [1.1. 清理非法属性值以及删除退化三角形](#11-清理非法属性值以及删除退化三角形)
  - [1.2. 按材质重排序 Cluster 内的三角形](#12-按材质重排序-cluster-内的三角形)
  - [1.3. **Stripify** Cluster](#13-stripify-cluster)
- [2. 编码 Cluster DAG](#2-编码-cluster-dag)
- [2. References](#2-references)

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
    TRACE_CPUPROFILER_EVENT_SCOPE(Nanite::Build::RemoveDegenerateTriangles);    // TODO: is this still necessary?
    RemoveDegenerateTriangles( ClusterDAG.Clusters );
}
```

### 1.2. 按材质重排序 Cluster 内的三角形

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

`MaterialRanges` 数组中的每个 `MaterialRange` 元素记录了 cluster 内从第 `RangeStart` 个三角形开始，连续 `RangeLength` 个三角形，都是使用的材质 `MaterialIndex`。

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

<!-- 在 Nanite 编码前，还会对 cluster 做编码约束调整：在每个 `MaterialRange` 内对三角形进行重排，并按新的访问顺序重排顶点；当某个旧顶点超出 32 顶点窗口约束时，会复制该顶点以生成新的局部顶点索引；如果约束后 cluster 顶点数超过 256，还会将 cluster 按三角形范围拆分并重新约束。这样可以保证每个 cluster 满足 32 顶点窗口约束和 256 顶点上限，为后续 index 编码和 GPU transcoding 做准备。 -->

所谓的 **Stripify**，指的是：//todo: 总结

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

        // 计算三角形 3 个顶点**位置和**的 x 分量值, 后续选 strip 起点时, 如果评分相同, 则用它决定谁优先
        FVector3f ScaledCenter = Cluster.Verts.GetPosition( i0 ) + Cluster.Verts.GetPosition( i1 ) + Cluster.Verts.GetPosition( i2 );
        TrianglePriorities[ i ] = ScaledCenter.X;   //TODO: Find a good direction to sort by instead of just picking x?

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

`BuildTables()` 是在 stripify 之前的关键一步，它根据 cluster 的原始网格信息构建了 stripify 所需的数据：首先是 `VertexToTriangleMasks`，它记录 **cluster 内每个顶点关联了哪些三角形**，通过 `VertexToTriangleMasks[ vertex index ][ DWORD i ]` 可以快速知道某个 32-triangle DWORD 中有哪些三角形使用了顶点 `Verts[index]`；其次是 `OppositeCorner`，它记录**每个三角形 `Corner` 所代表的有向边，其反向共享边对应的是哪个三角形 `Corner`**，后续 stripify 时会根据此数据找左/右相邻三角形；最后 `TrianglePriorities` 则是保存每个三角形 3 个顶点**位置和**的 `x` 分量值，后续选 strip 起点三角形时，如果评分相同，则用它决定谁优先。

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
        // 相对距离是否满足 5 bits 偏移约束
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

`NewScoreVertex()` 方法的作用是给一个候选旧顶点打分。它首先检查这个旧顶点是否已经对应到新顶点序列中的某个已输出新顶点，如果还没有对应的新顶点，则这个旧顶点当前不能作为 ref 顶点使用，分数就是 0；如果已经有对应的新顶点，则计算这个新顶点与当前新顶点序列末尾之间的相对距离，也就是 `CachePosition = ( Context.NumVertices - 1 ) - NewIndex`。如果这个相对距离仍在 5 bits 可表示范围内，就可以作为 ref 顶点候选，然后根据当前候选三角形是否是 strip 起点、它的邻接三角形情况，以及这个 ref 距离去权重表里取分数。

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

在这里单独说明一下什么是 5 bits 偏移约束：Nanite 在 `StripIndexData` 中使用 5 bits 存储 ref 顶点的相对距离，这个相对距离引用的是新顶点序列中已经输出过的某个新顶点，5 bits 中写入的是 `BaseVertex - Index`，其中 `BaseVertex` 是当前三角形输出前新顶点序列末尾的索引，`Index` 是被引用的新顶点索引。因为只有 5 bits，所以这个距离必须在 5 bits 可表示范围内，也就是小于 32。如果某个旧顶点虽然已经对应到新顶点序列中的一个已输出顶点，但距离太远，就不能继续作为 ref 顶点编码，而需要重新作为新顶点编码。

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

第二步，对当前三角形中可能作为 ref 顶点编码的旧顶点执行 **5 bits 偏移约束**检查，通过约束检查的顶点可以作为 ref 顶点编码，否则需要重新作为新顶点编码：

```cpp
// 3 个旧顶点在新顶点序列中的索引
uint16& NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
uint16& NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
uint16& NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];

uint32 OrgIndex0 = NewIndex0;
uint32 OrgIndex1 = NewIndex1;
uint32 OrgIndex2 = NewIndex2;

// 根据 3 个旧顶点是否已经对应到新顶点序列，预估当前三角形处理完成后，下一个可分配的新顶点索引
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
        // 新顶点序列中会新增一个顶点，所以下一个可分配的新顶点索引 + 1
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

然后对 ref 顶点候选执行 5 bits 偏移约束检查：分别检查每个 ref 顶点引用的新顶点，与当前三角形处理完成后的新顶点序列末尾之间的相对距离。如果这个相对距离 `>= 32`，说明无法用 5 bits 记录，则这个旧顶点不能继续作为 ref 顶点编码，而是会被重新作为新顶点编码。这里用的是 `NextVertexIndex` 做预检查，因为当前三角形本身也可能会新增新顶点，新增的新顶点会影响其他 ref 顶点的相对距离。

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

`BuildTables()` 方法主要构建 stripify 所需的几类辅助数据：`VertexToTriangleMasks` 记录每个旧顶点关联了哪些旧三角形；`OppositeCorner` 通过三角形 `Corner` 记录邻接三角形之间的反向共享边信息，Nanite 后续可以根据三角形 `Corner` 找到特定的左侧或者右侧邻接三角形；`TrianglePriorities` 保存每个旧三角形 3 个旧顶点位置和的 `x` 分量值，后续在评分相同的情况下，用它决定哪个三角形优先作为 strip 起点输出。

而 `NewScoreVertex()` 方法主要是根据一个旧顶点是否可以作为 ref 顶点、作为 ref 顶点时距离当前新顶点序列末尾有多远，以及候选三角形的拓扑上下文为这个旧顶点打分。`NewScoreTriangle()` 则是把三角形 3 个旧顶点的分数相加，用于决定 strip 起点和 strip 延伸时优先选择哪个候选三角形。

`VisitTriangle()` 是整个 stripify 过程中的核心方法，它的主要逻辑是将一个旧三角形输出到 stripify 之后的三角形序列中，并决定这个三角形相关的旧顶点在 stripify 之后应当编码为一个新顶点还是一个 ref 顶点。需要注意的是，对于 strip 的起点三角形，最多需要处理其 3 个顶点的编码，而对于 strip 的延伸三角形，因为其前 2 个顶点通常由 strip 拓扑隐含复用，所以只需要处理其新引入的那个顶点的编码。具体来说：

对于需要编码的旧顶点，首先检查它是否已经对应到新顶点序列中的某个已输出顶点，如果还不存在于新顶点序列中，则将其作为一个新顶点编码；而如果它已经存在于新顶点序列中，则优先考虑将其作为 ref 顶点编码：通过记录其对应的已输出新顶点与当前新顶点序列末尾之间的相对距离，来引用这个已输出的新顶点。又因为 Nanite 使用 5 bits 去存储这个相对距离，所以如果这个相对距离超过了 5 bits 可以表示的范围，那么仍然会将这个旧顶点重新作为一个新顶点编码。

说完了 `BuildTables()`、`NewScoreVertex()` 和 `VisitTriangle()` 这 3 个关键方法，下面来说说 stripify 的核心流程：

Nanite 是**按材质段 `MaterialRange` 分段**进行 stripify 的，这样可以保持材质分组不被打乱，避免 strip 跨材质。

首先会初始化相关数据：

```cpp
// 清掉旧的 Cluster.StripIndexData 字节流
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

`Cluster.StripIndexData` 是一个连续、紧凑的 5 bits 字节流，Nanite 通过 `BitWriter.PutBits(BaseVertex - Index, 5)` 以**低位优先**的顺序向其中写入连续的 5 bits 偏移，每满 8 bits 就会将其转换为一个 `uint8` 并添加进 `Cluster.StripIndexData` 数组中。

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
        
        // 对可能作为 ref 顶点编码的新顶点进行 5 bits 偏移约束检查: 如果其对应的已输出新顶点与当前新顶点序列末尾之间的相对距离超过 5 bits 可表示范围, 就必须把对应旧顶点重新作为新顶点编码
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
    // 分数相同时根据 3 个顶点位置和的 x 分量值决定选谁当起点
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
    // 输出前的新顶点序列末尾索引, 用来计算 ref 顶点的 5 bits 相对距离
    uint32 BaseVertex = Context.NumVertices - 1;

    // 输出这个三角形
    uint32 NumNewVertices = VisitTriangle(Context, StartCorner, true, false);

    // 为 strip 起点三角形写 ref 顶点的 5 bits 相对距离:
    // 如果 NumNewVertices = 3, 则表示 3 个顶点都是新顶点, 不写任何 5 bits 相对距离
    // 如果 NumNewVertices = 2, 则表示第 1 个顶点是 ref 顶点, 写 1 个 5 bits 相对距离
    // 如果 NumNewVertices = 1, 则表示前 2 个顶点是 ref 顶点, 写 2 个 5 bits 相对距离
    // 如果 NumNewVertices = 0, 则表示 3 个顶点都是 ref 顶点, 写 3 个 5 bits 相对距离
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

**另外**：根据上面的源码逻辑可以看到，在输出 strip 起点三角形之前，会先记录当前新顶点序列末尾顶点的索引 `BaseVertex`，而在 `VisitTriangle()` 方法中，如果如果一个旧顶点被编码为一个 ref 顶点，那么它引用的就是输出当前这个 strip 起点三角形之前新顶点序列中已经存在的某个顶点，引用的顶点在新顶点序列中的索引可以根据映射 `Context.OldToNewVertex` 获取。那么也就是说：对于一个作为 ref 顶点编码的旧顶点，它所对应的 5 bits 偏移指的是：**从它所属的旧三角形输出前的新顶点序列末尾往前数多少偏移，这个偏移所对应的顶点就是它所引用的顶点**。

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

因此对于 strip 延伸三角形来说，前 2 个共享边顶点必须编码为 ref 顶点，并且还必须满足 5 bits 偏移约束，否则只能结束当前 strip 的延伸扩展，重新选择新的 strip 起点。

所以 Nanite 在选择了要延伸的 `CurrentCorner` 后，还会预测输出延伸旧三角形后，会不会因为延伸引入新的顶点导致新顶点序列长度增加 1 从而使共享边的 2 个 ref 顶点不再能通过 5 bits 偏移去表示：

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

    // 判断共享边的 2 个 ref 顶点是否仍然满足 5 bits 偏移约束, 如果不满足则结束当前 strip 的延伸扩展, 重新选择新的 strip 起点
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
    // 如果第 3 个顶点也是 ref 顶点, 则写此 ref 顶点的 5 bits 相对距离
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
// 最终 strip 编码产生的新顶点数量, 它可能大于旧顶点数，因为超过 5 bits 约束的 ref 顶点会重新作为新顶点编码
const uint32 NumNewVertices = Context.NumVertices;

// 把原来的顶点数组拷贝到 OldVertices, 并清空 Cluster.Verts
FVertexArray OldVertices( Cluster.Verts.Format );
Swap( OldVertices, Cluster.Verts );

// 按 NewToOldVertex 映射重建 Cluster.Verts: 如果某个旧顶点因为 5 bits 偏移约束被重新作为新顶点编码，那么 NewToOldVertex 中会有多个新索引指向同一个旧顶点；这里会复制相同的 vertex attribute 到多个位置
Cluster.Verts.Reserve( NumNewVertices );
for( uint32 i = 0; i < NumNewVertices; i++ )
{
    Cluster.Verts.Add( &OldVertices.GetPosition( Context.NewToOldVertex[i] ) );
}

// Stripify 不能丢三角形，也不能重复输出三角形。最终输出数量必须等于原 cluster 三角形数
check( Context.NumTriangles == NumOldTriangles );
```

因为在 stripify 的过程中，有些 ref 顶点会因为超出 5 bits 偏移约束从而被重新编码成一个新顶点，所以 `Context.NewToOldVertex` 中会存在多个索引指向同一个旧顶点的情况，这也导致最终 `Cluster.Verts` 中会存在多个相同的旧顶点。

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

在 `Cluster.StripDesc` 中，`Bitmasks` 记录每个三角形的 strip 编码元数据，而 `NumPrevNewVerticesBeforeDwords` 和 `NumPrevRefVerticesBeforeDwords` 则是记录每个 32-triangle DWORD 之前累计的新顶点数和 ref 顶点数。

## 2. 编码 Cluster DAG

## 2. References

- [Nanite: A Deep Dive](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)
- [GAMES 104: GPU-Driven Geometry Pipeline - Nanite](https://www.piccoloengine.com/merch/8)
