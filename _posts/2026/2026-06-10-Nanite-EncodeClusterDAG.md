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
- [//todo: Old](#todo-old)
- [//todo: New](#todo-new)

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
            // bHasOpposite: 当前 Corner 对面的邻接三角形是否可用
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

在这里单独说明一下什么是 5 bits 偏移约束：Nanite 在 `StripIndexData` 中使用 5 bits 存储 ref 顶点的相对距离。这个相对距离引用的是新顶点序列中已经输出过的某个新顶点，写入时类似 `BaseVertex - Index`，其中 `BaseVertex` 是当前三角形输出前新顶点序列末尾的索引，`Index` 是被引用的新顶点索引。因为只有 5 bits，所以这个距离必须小于 32，也就是必须在 5 bits 可表示范围内；如果某个旧顶点虽然已经对应到新顶点序列中的一个已输出顶点，但距离太远，就不能继续作为 ref 顶点编码，而需要重新作为新顶点编码。

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

首先 Nanite 是按材质段（`MaterialRange`）分段进行 stripify 的，这样可以保持材质分组不被打乱，避免 strip 跨材质。首先会初始化相关数据：

## 2. 编码 Cluster DAG

## 2. References

- [Nanite: A Deep Dive](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)
- [GAMES 104: GPU-Driven Geometry Pipeline - Nanite](https://www.piccoloengine.com/merch/8)

//todo: Old
---
Nanite 是按材质段（`MaterialRange`）分段进行 stripify 的，这样可以保持材质分组不被打乱，避免 strip 跨材质。首先会初始化相关数据：

```cpp
// 清掉旧的 strip index 字节流
Cluster.StripIndexData.Empty();
// 初始化 BitWriter, 绑定数组 Cluster.StripIndexData
FBitWriter BitWriter( Cluster.StripIndexData );
// 清掉旧的 strip 编码信息
FStripDesc& StripDesc = Cluster.StripDesc;
FMemory::Memset(StripDesc, 0);
// 记录整个 stripify 过程新增顶点数
uint32 NumNewVerticesInDword[ 4 ] = {};
// 记录整个 stripify 过程的 ref 顶点数
uint32 NumRefVerticesInDword[ 4 ] = {};
```

`Cluster.StripIndexData` 本质上是一个连续、紧凑的 5-bit deleta 流，Nanite 通过 `BitWriter.PutBits(BaseVertex - Index, 5)` 以低位优先的顺序向其中写入连续的 5-bit delta 字节流，每满 8 bits 就会吐出一个 `uint8` 并添加进 `Cluster.StripIndexData` 中。

按材质段 stripify 时，首先会将当前材质段的所有三角形标记为候选状态，后续 stripify 时只处理候选三角形：

```cpp
// 将当前材质段中的三角形标记为候选状态
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
    // BitEnd >= 32 则说明材质段终点不在当前这个 DWORD 中, EndMask 直接使用 0xFFFFFFFFu; 否则将 [0, EndMask) 的 bits 置为 1
    uint32 EndMask = BitEnd < 32 ? ( ( 1u << BitEnd ) - 1u ) : 0xFFFFFFFFu;
    // 最后按位异或将当前材质段三角形所属 DWORD 对应的 bits 置为 1.
    // 举个例子: StartMask = bits 0..9, EndMask = bits 0..14, 那么 StartMask ^ EndMask = bits 10..14
    Context.TrianglesEnabled[ i ] |= StartMask ^ EndMask;
}
```

然后会通过 `NewScoreTriangle` 为每个候选三角形打分，最后选取最高分的三角形作为当前 strip 起点。

另外，对于作为 strip 起点的三角形，Nanite 要求它的 3 个旧顶点按 strip 编码需要的局部顺序 stripify 后必须满足：**引用顶点必须在前，新顶点必须连续出现在后面**，也就是下面这几种情况：

```text
1. ref, ref, ref
2. ref, ref, new
3. ref, new, new
4. new, new, new
```

所以 Nanite 会先遍历所有的候选三角形，然后通过 `Context.OldToNewVertex` 获取它们的 3 个旧顶点在新顶点序列中的索引 `NewIndex`，根据 `NewIndex` 和 32 顶点窗口约束决定每个顶点是当作一个引用顶点还是新顶点 stripify，只有最终 stripify 顺序满足以上几种情况之一的三角形才会进入后续打分环节：

```cpp
// 计算当前顶点所对应的三角形 Corner
uint32 TriangleCorner = SetCorner( TriangleIndex, Corner );

{
    // Is it viable WRT the constraint that new vertices should always be at the end.
    // 按 stripify 编码顺序取三角形的 3 个旧顶点
    uint32 OldIndex0 = Cluster.Indexes[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
    uint32 OldIndex1 = Cluster.Indexes[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];
    uint32 OldIndex2 = Cluster.Indexes[ CornerToIndex( TriangleCorner ) ];

    // 查看这 3 个旧顶点是否已经在新顶点序列中
    uint32 NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
    uint32 NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
    uint32 NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];
    uint32 NumVerts = Context.NumVertices + ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );
    
    // 如果旧顶点已经在新顶点序列中, 则表示可以通过记录 5-bit delta 引用这个新顶点
    // 在这里模拟 32 顶点窗口约束: 如果 stripify 当前三角形后, 当前旧顶点引用的新顶点索引距离当前 strip 最新顶点超过 31 从而导致 5 bits 存储不下这个偏移, 则把它当作一个新顶点 stripify 
    while(true)
    {
        if( NewIndex0 != INVALID_INDEX && NumVerts - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex0 = INVALID_INDEX; NumVerts++; continue; }
        if( NewIndex1 != INVALID_INDEX && NumVerts - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex1 = INVALID_INDEX; NumVerts++; continue; }
        if( NewIndex2 != INVALID_INDEX && NumVerts - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex2 = INVALID_INDEX; NumVerts++; continue; }
        break;
    } 

    // Nanite 要求可以作为 strip 起点的三角形的 3 个旧顶点只有下面几种情况:
    // ref, ref, ref
    // ref, ref, new
    // ref, new, new
    // new, new, new
    // 所以这里检查当前候选三角形是否满足
    uint32 Mask = ( NewIndex0 == INVALID_INDEX ? 1u : 0u ) | ( NewIndex1 == INVALID_INDEX ? 2u : 0u ) | ( NewIndex2 == INVALID_INDEX ? 4u : 0u );

    // 跳过不满足的候选三角形
    if( Mask != 0u && Mask != 4u && Mask != 6u && Mask != 7u )
    {
        continue;
    }
}
```

对于满足要求的候选三角形，根据其拓扑上下文通过 `NewScoreTriangle` 为其打分，并记录得分最高的候选三角形：

```cpp
// 候选三角形对面的邻居三角形
uint32 Opposite = OppositeCorner[ CornerToIndex( TriangleCorner ) ];
// 候选三角形左侧的邻居三角形
uint32 LeftCorner = OppositeCorner[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
// 候选三角形右侧的邻居三角形
uint32 RightCorner = OppositeCorner[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];

// 对面的邻居三角形是否存在并且也是候选三角形
bool bHasOpposite = Opposite != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( Opposite ) );
// 左侧的邻居三角形是否存在并且也是候选三角形
bool bHasLeft = LeftCorner != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( LeftCorner ) );
// 右侧的邻居三角形是否存在并且也是候选三角形
bool bHasRight = RightCorner != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( RightCorner ) );

// 根据候选三角形拓扑上下文信息为其打分
int32 Score = NewScoreTriangle( Context, TriangleIndex, true, bHasOpposite, bHasLeft, bHasRight );
if( Score > BestScore )
{
    // 记录候选三角形
    StartCorner = TriangleCorner;
    // 记录最高分
    BestScore = Score;
}
else if( Score == BestScore )
{
    // 分数相同时根据三角形 3 个旧顶点位置和的 X 值决定谁更优先
    float Priority = TrianglePriorities[ TriangleIndex ];
    if( Priority > BestPriority )
    {
        // 记录候选三角形
        StartCorner = TriangleCorner;
        // 记录最高分
        BestScore = Score;
        BestPriority = Priority;
    }
}
```

如果当前材质段最终没有找到适合当 strip 起点的三角形，则跳过 stripify 当前材质段：

```cpp
// 当前材质段内没有找到合适的 strip 起点三角形, 则跳出当前材质段
if( StartCorner == INVALID_CORNER )
    break;
```

如果找到合适的候选三角形后，则继续 stripify 当前材质段，首先会 stripify 当前 strip 选中的起点三角形：

```cpp
{
    // 当前 strip 起点三角形 stripify 后属于第几个 DWORD
    uint32 TriangleDword = Context.NumTriangles >> 5;
    // stripify 前的最大新顶点索引, 用来计算 ref delta
    uint32 BaseVertex = Context.NumVertices - 1;

    // stripify 当前 strip 起点三角形
    uint32 NumNewVertices = VisitTriangle(Context, StartCorner, true, false);

    // 为当前 strip 起点三角形的每个 ref 顶点写 5-bit delta
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
    // 统计所属 DWORD 新增顶点数
    NumNewVerticesInDword[TriangleDword] += NumNewVertices;
    // 统计所属 DWORD 引用顶点数
    NumRefVerticesInDword[TriangleDword] += 3u - NumNewVertices;
}
```

我们知道，在 `VisitTriangle` 方法中如果一个旧顶点被 stripify 成一个新顶点，那么 `Context.NumVertices++`，也就是说：**`Context.NumVertices` 中统计的是 stripify 过程中产生的新顶点的数量**。

而根据上面的逻辑可以看到，在 stripify 起点三角形之前，会先记录当前已经输出的新顶点序列里的最后一个顶点的索引 `BaseVertex`，而在 `VisitTriangle` 方法中如果一个旧顶点被输出成一个 ref 顶点，那么它引用的就是 stripify 起点三角形之前新顶点序列中已经存在的顶点，引用的顶点的索引可以根据映射 `Context.OldToNewVertex` 获取，那么，我们也就知道了，对于一个三角形的 ref 顶点，它所对应的 5-bit delta 指的是：**在 stripify 这个三角形之前的最后一个已输出的新顶点往前数多个位置**，这个位置所对应的顶点就是它引用的顶点。

找到并 stripify 当前 strip 的起点三角形后，下面就从起点三角形开始延伸扩展当前 strip。首先，如果当前 strip 的三角形数量已经达到 32，则跳出当前 strip 的延伸扩展，重新去开启一个新的 strip：  //todo: why?

```cpp
// 从 strip 起点三角形开始延伸扩展当前 strip
uint32 CurrentCorner = StartCorner;
while (true)
{
    // 如果当前 strip 的三角形数量已经达到 32, 则跳出当前 strip 的延伸扩展, 重新去开启一个新的 strip
    if ((Context.NumTriangles & 31u) == 0u)
        break;

    ...
}
```

然后分别找左侧和右侧的邻居三角形，并根据它们的拓扑上下文给它们打分，最终选择分数更高的一侧对 strip 进行延伸扩展：

```cpp
// 左侧邻居三角形
uint32 LeftCorner = OppositeCorner[CornerToIndex(NextCorner(CurrentCorner))];
// 右侧邻居三角形
uint32 RightCorner = OppositeCorner[CornerToIndex(PrevCorner(CurrentCorner))];
CurrentCorner = INVALID_CORNER;

int32 LeftScore = INT_MIN;
// 左侧的邻居三角形是否存在并且也是候选三角形
if (LeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftCorner)))
{
    uint32 LeftLeftCorner = OppositeCorner[CornerToIndex(NextCorner(LeftCorner))];
    uint32 LeftRightCorner = OppositeCorner[CornerToIndex(PrevCorner(LeftCorner))];
    bool bLeftLeftCorner = LeftLeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftLeftCorner));
    bool bLeftRightCorner = LeftRightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftRightCorner));

    // 根据左侧邻居三角形的拓扑上下文给其打分
    LeftScore = NewScoreTriangle(Context, CornerToTriangle(LeftCorner), false, true, bLeftLeftCorner, bLeftRightCorner);
    CurrentCorner = LeftCorner;
}

bool bIsRight = false;
// 右侧的邻居三角形是否存在并且也是候选三角形
if (RightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightCorner)))
{
    uint32 RightLeftCorner = OppositeCorner[CornerToIndex(NextCorner(RightCorner))];
    uint32 RightRightCorner = OppositeCorner[CornerToIndex(PrevCorner(RightCorner))];
    bool bRightLeftCorner = RightLeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightLeftCorner));
    bool bRightRightCorner = RightRightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightRightCorner));

    // 根据右侧邻居三角形的拓扑上下文给其打分
    int32 Score = NewScoreTriangle(Context, CornerToTriangle(RightCorner), false, false, bRightLeftCorner, bRightRightCorner);

    // 根据评分决定往左侧还是右侧延伸扩展 strip
    if (Score > LeftScore)
    {
        CurrentCorner = RightCorner;
        bIsRight = true;
    }
}

// 没有找到合适的邻居三角形, 则跳出当前 strip 的延伸扩展
if (CurrentCorner == INVALID_CORNER)
    break;
```

再进行 32 顶点窗口约束的检查：

```cpp
{
    const uint32 OldIndex0 = Cluster.Indexes[CornerToIndex(NextCorner(CurrentCorner))];
    const uint32 OldIndex1 = Cluster.Indexes[CornerToIndex(PrevCorner(CurrentCorner))];
    const uint32 OldIndex2 = Cluster.Indexes[CornerToIndex(CurrentCorner)];

    const uint32 NewIndex0 = Context.OldToNewVertex[OldIndex0];
    const uint32 NewIndex1 = Context.OldToNewVertex[OldIndex1];
    const uint32 NewIndex2 = Context.OldToNewVertex[OldIndex2];

    // 延伸三角形必须沿当前 strip 的边接上, 所以前 2 个顶点必须输出成 ref 顶点
    check(NewIndex0 != INVALID_INDEX);
    check(NewIndex1 != INVALID_INDEX);

    // 预估计算 stripify 这个三角形后的已输出新顶点数量
    const uint32 NextNumVertices = Context.NumVertices + ((NewIndex2 == INVALID_INDEX || Context.NumVertices - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE) ? 1u : 0u);

    // 以及判断前 2 个 ref 顶点是否超过 32 顶点窗口约束
    if (NextNumVertices - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ||
        NextNumVertices - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE)
        break;
}
```

最后 stripify 选中的邻居三角形对 strip 进行延伸扩展：

```cpp
{
    // 选中的邻居三角形所属的 DWORD
    uint32 TriangleDword = Context.NumTriangles >> 5;
    // 当前已经输出的新顶点序列里的最后一个顶点的索引
    uint32 BaseVertex = Context.NumVertices - 1;
    // stripfy 选中的邻居三角形
    uint32 NumNewVertices = VisitTriangle(Context, CurrentCorner, false, bIsRight);
    // 延伸扩展的三角形最多只能输出 1 个新顶点
    check(NumNewVertices <= 1u);
    // 如果第 3 个顶点也是 ref 顶点, 则写 1 个 5-bit delta
    if (NumNewVertices == 0)
    {
        uint32 Index = Context.OldToNewVertex[Cluster.Indexes[CornerToIndex(CurrentCorner)]];
        BitWriter.PutBits(BaseVertex - Index, 5);
    }

    // 统计所在 DWORD 新增顶点数
    NumNewVerticesInDword[TriangleDword] += NumNewVertices;
    // 统计所在 DWORD 引用顶点数
    NumRefVerticesInDword[TriangleDword] += 1u - NumNewVertices;
}

// strip 长度 + 1
StripLength++;
```

以上就是整个 stripify 的逻辑，stripfy 结束之后，会先调用 `BitWriter.Flush(sizeof(uint32))` 方法将还未满 8 bits 的已写 bits 作为一个 `uint8` 追加进 `Cluster.StripIndexData` 中，然后还会将 `Cluster.StripIndexData` 以 4 字节对齐：

```cpp
void Flush(uint32 Alignment=1)
{
    // 先把 PendingBits 中还没满 8 bits 的已写 bits 作为一个 uint8 追加进 Cluster.StripIndexData 中
    if (NumPendingBits > 0)
        Buffer.Add((uint8)PendingBits);
    // 如果 Cluster.StripIndexData.Num() 不是 4 的倍数, 就追加 (uint8)0, 直到 4 字节对齐
    while (Buffer.Num() % Alignment != 0)
        Buffer.Add(0);

    // 清空 BitWriter 内部暂存数据
    PendingBits = 0;
    NumPendingBits = 0;
}
```

然后根据 stripify 的结果重建 cluster 的顶点数据 `Verts`：

```cpp
// Reorder vertices
// stripify 最终的新顶点数, 它可能大于旧顶点数, 因为超出 32 顶点窗口约束的 ref 顶点会被当作一个新的顶点输出
const uint32 NumNewVertices = Context.NumVertices;

// 把旧的 Cluster.Verts 拷贝到 OldVertices 并清空旧的 Cluster.Verts
FVertexArray OldVertices( Cluster.Verts.Format );
Swap( OldVertices, Cluster.Verts );

// 根据 Context.NewToOldVertex 映射重建 Cluster.Verts
// 如果某个旧顶点因为超出 32 顶点窗口约束被当作一个新顶点输出, 那么 Context.NewToOldVertex 中会有多个索引指向同一个旧顶点, 这里会复制多个相同的 Vert 到 Cluster.Verts 中
Cluster.Verts.Reserve( NumNewVertices );
for( uint32 i = 0; i < NumNewVertices; i++ )
{
    Cluster.Verts.Add( &OldVertices.GetPosition( Context.NewToOldVertex[i] ) );
}

// stripify 最终输出的三角形数必须等于原 cluster 的三角形数
check( Context.NumTriangles == NumOldTriangles );
```

需要注意的是，因为在 stripify 的过程中，有些旧顶点会因为超出 32 顶点窗口约束而被输出成一个新顶点，也就是说 stripify 后最终的新顶点数可能会超过原顶点数，并且 `Context.NewToOldVertex` 中会有多个索引指向同一个旧顶点，这也导致最终 `Cluster.Verts` 中会存在多个相同的旧 Vert。

重建完新的 `Cluster.Verts` 数据后，继续将 stripify 信息写进 `Cluster.StripDesc` 中：

```cpp
// 计算每个 DWORD 之前累计的新增顶点数:
// DWORD 0 之前 = 0, 不用存
// DWORD 1 之前 = DWORD 0 新增顶点数
// DWORD 2 之前 = DWORD 0 + DWORD 1 新增顶点数
// DWORD 3 之前 = DWORD 0 + DWORD 1 + DWORD 2 新增顶点数
uint32 NumPrevNewVerticesBeforeDwords1 = NumNewVerticesInDword[0];
uint32 NumPrevNewVerticesBeforeDwords2 = NumNewVerticesInDword[1] + NumPrevNewVerticesBeforeDwords1;
uint32 NumPrevNewVerticesBeforeDwords3 = NumNewVerticesInDword[2] + NumPrevNewVerticesBeforeDwords2;
// 每个累计值用 10 bit 存，所以必须小于 1024
check(NumPrevNewVerticesBeforeDwords1 < 1024 && NumPrevNewVerticesBeforeDwords2 < 1024 && NumPrevNewVerticesBeforeDwords3 < 1024);
// 编码进 1 个 uint32
StripDesc.NumPrevNewVerticesBeforeDwords = (NumPrevNewVerticesBeforeDwords3 << 20) | (NumPrevNewVerticesBeforeDwords2 << 10) | NumPrevNewVerticesBeforeDwords1;

// 计算每个 DWORD 之前累计的 ref 顶点数:
// DWORD 0 之前 = 0, 不用存
// DWORD 1 之前 = DWORD 0 ref 顶点数
// DWORD 2 之前 = DWORD 0 + DWORD 1 ref 顶点数
// DWORD 3 之前 = DWORD 0 + DWORD 1 + DWORD 2 ref 顶点数
uint32 NumPrevRefVerticesBeforeDwords1 = NumRefVerticesInDword[0];
uint32 NumPrevRefVerticesBeforeDwords2 = NumRefVerticesInDword[1] + NumPrevRefVerticesBeforeDwords1;
uint32 NumPrevRefVerticesBeforeDwords3 = NumRefVerticesInDword[2] + NumPrevRefVerticesBeforeDwords2;
// 每个累计值用 10 bit 存，所以必须小于 1024
check(NumPrevRefVerticesBeforeDwords1 < 1024 && NumPrevRefVerticesBeforeDwords2 < 1024 && NumPrevRefVerticesBeforeDwords3 < 1024);
// 编码进 1 个 uint32
StripDesc.NumPrevRefVerticesBeforeDwords = (NumPrevRefVerticesBeforeDwords3 << 20) | (NumPrevRefVerticesBeforeDwords2 << 10) | NumPrevRefVerticesBeforeDwords1;

// 复制 strip bitmask
static_assert(sizeof(StripDesc.Bitmasks) == sizeof(Context.StripBitmasks), "");
FMemory::Memcpy(StripDesc.Bitmasks, Context.StripBitmasks, sizeof(StripDesc.Bitmasks));
```

首先 Nanite 一共使用 4 个 DWORD 表示 1 个 cluster 的 128 个三角形，其中每个 DWORD 的 32 bits 表示 32 个三角形。

从 `StripDesc.NumPrevNewVerticesBeforeDwords` 的最低位开始，每 10 bits 记录了**每个 DWORD 之前累计的新增顶点数**，也就是说：`StripDesc.NumPrevNewVerticesBeforeDwords` 的 0-9 bit 记录了 DWORD 0 中表示的三角形累计的新增顶点数；10-19 bit 记录了 DWORD 0 和 DOWRD 1 中表示的三角形累计的新增顶点数；20-31 bit 记录了 DWORD0、DWORD1 和 DOWRD2 中表示的三角形累计的新增顶点数。类似的 `StripDesc.NumPrevRefVerticesBeforeDwords` 则是从最低位开始，每 10 bits 记录了**每个 DWORD 之前累积的 ref 顶点数**。

`StripDesc.Bitmasks` 则是记录每个三角形的 strip 编码信息，首先 `[4]` 表示的是 4 个 DWORD，根据三角形索引 `TriangleIndex >> 5` 可以快速得到这个三角形所属的 DWORD，而通过 `( NewTriangleIndex & 31u )` 又可以得到这个三角形在 `StripDesc.Bitmasks[TriangleIndex >> 5]` 这 3 个 `uint32` 中对应的 bit 位，每个 `uint32` 中对应 bit 位中记录了这个三角形的 strip 编码信息：

其中 `StripDesc.Bitmasks[TriangleIndex >> 5][0]` 对应的 bit 位中记录此三角形是否是一个 strip 的起点，bit 位为 1 则代表是一个 strip 的起点；

如果此三角形是一个 strip 的起点，那么对应的在 `StripDesc.Bitmasks[TriangleIndex >> 5][1]` 和 `StripDesc.Bitmasks[TriangleIndex >> 5][2]` 中这 2 bit 组合起来记录了这个 strip 起点三角形中 ref 顶点数量（`0..2`），其中 `StripDesc.Bitmasks[TriangleIndex >> 5][1]` 是高位，`StripDesc.Bitmasks[TriangleIndex >> 5][2]` 是低位。

而如果此三角形不是一个 strip 的起点，也就是说它是一个 strip 的延伸，那么对应的在 `StripDesc.Bitmasks[TriangleIndex >> 5][1]` 中的 bit 位记录这个三角形是从左边还是右边延伸过来的，对应的在 `StripDesc.Bitmasks[TriangleIndex >> 5][2]` 中的 bit 位则记录这个三角形的第 3 个顶点是否是 ref 顶点，因为延伸的三角形的前面 2 个顶点肯定是 ref 顶点。

最后，则是 Nanite 通过 stripify 信息重建 cluster 的顶点索引 `Indexes`，核心的逻辑在 `UnpackTriangleIndices()` 方法中，下面通过对重建 `Indexes` 的源码分析加深对 stripify 后数据结构的理解：

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
    //   - 而对于 strip 起点三角形, 它表示的是该三角形 ref 顶点数的高位 (与 IsRef mask 中的低位组合成 2 bits 记录 strip 起点三角形的 ref 顶点数);
    const uint32 LMask = StripDesc.Bitmasks[ DwordIndex ][ 1 ];
    // WMask:
    //   - 对于非 strip 起点三角形, 它表示该三角形第 3 个顶点是否是 ref 顶点;
    //   - 而对于 strip 起点三角形, 它表示的是该三角形 ref 顶点数的低位 (与 IsLeft mask 中的高位组合成 2 bits 记录 strip 起点三角形的 ref 顶点数);
    const uint32 WMask = StripDesc.Bitmasks[ DwordIndex ][ 2 ];
    // SLMask: 某个三角形在 SLMask 中对应的 bit 位为 1 则表示该三角形既是 strip 起点并且其 ref 顶点数高位为 1, 也就是说该三角形的 ref 顶点数 >= 2
    const uint32 SLMask = SMask & LMask;
    
    //const uint HeadRefVertexMask = ( SMask & LMask & WMask ) | ( ~SMask & WMask );
    // 上面注释是下面代码的展开版本
    // HeadRefVertexMask: 某个三角形的 head 顶点是否是 ref 顶点, 有以下两种情况:
    //   - 对于非 strip 起点三角形, 它的 head 顶点是 ref 顶点则表示此三角形的第 3 个顶点是 ref 顶点, 也就是 WMask = 1;
    //   - 而对于 strip 起点三角形, 它的 head 顶点是 ref 顶点则表示此三角形的 3 个顶点全部都是 ref 顶点, 也就是 ref 顶点数为 3;
    const uint32 HeadRefVertexMask = ( SLMask | ~SMask ) & WMask;   // 1 if head of triangle is ref. S case with 3 refs or L/R case with 1 ref.

    // PrevBitsMask: 当前 32-triangle DWORD 内, 当前三角形之前所有的三角形的 bit mask
    const uint32 PrevBitsMask = ( 1u << BitIndex ) - 1u;
    // 当前 32-triangle DWORD 之前累计的 ref 顶点数
    const uint32 NumPrevRefVerticesBeforeDword = DwordIndex ? BitFieldExtractU32(StripDesc.NumPrevRefVerticesBeforeDwords, 10u, DwordIndex * 10u - 10u) : 0u;
    // 当前 32-triangle DWORD 之前累计的新顶点数
    const uint32 NumPrevNewVerticesBeforeDword = DwordIndex ? BitFieldExtractU32(StripDesc.NumPrevNewVerticesBeforeDwords, 10u, DwordIndex * 10u - 10u) : 0u;

    // 在当前 32-triangle DWORD 内当前三角形之前累计的 ref 顶点数, 其中:
    //   - ( FMath::CountBits( SLMask & PrevBitsMask ) << 1 ) : 累计是 strip 起点并且其 ref 顶点数高位为 1 的三角形的 ref 顶点数, 因为是是高位所以会有左移 1 位乘以 2 的计算;
    //   - FMath::CountBits( WMask & PrevBitsMask ) : 累计是 strip 起点并且其 ref 顶点数低位为 1 或者不是 strip 起点但其 head 顶点是 ref 顶点的三角形的 ref 顶点数;
    int32 CurrentDwordNumPrevRefVertices = ( FMath::CountBits( SLMask & PrevBitsMask ) << 1 ) + FMath::CountBits( WMask & PrevBitsMask );
    // 在当前 32-triangle DWORD 内当前三角形之前累计的新顶点数, 其中:
    //   - 首先我们知道: 对于 strip 起点三角形最多有 3 个新顶点; 而对于非 strip 起点三角形, 最多有 1 个新顶点, 所以:
    //     1. BitIndex                                          -> 表示当前三角形之前的每个三角形先按 1 个新顶点算
    //     2. ( FMath::CountBits( SMask & PrevBitsMask ) << 1 ) -> 然后再给每个 strip 起点三角形补 2 个新顶点
    //     3. 最后再减去累计的 ref 顶点数, 最终得到当前三角形之前累计的的新顶点数
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

    // BaseVertex 是: 当前三角形之前, 最后一个输出的新顶点的索引, 也就是最近新顶点的索引
    // 而 ref 顶点的 5-bit delta 编码的是: 引用的顶点相对最近新顶点的距离
    // 所以这里先算出最近新顶点的索引 BaseVertex
    const uint32 BaseVertex = NumPrevNewVertices - 1u;

    // StripIndexData 里每个 ref 顶点占 5-bit, 按直觉当前三角形的第一个 ref 顶点的 5-bit delta 应该从 NumPrevRefVertices * 5 开始读
    /*
        因为 ～(-1) = 0, ~(0) = -1, 也就是说 ( NumPrevRefVertices + ~IsStart ) * 5 等价于:

        if (IsStart)
        {
            readOffset = NumPrevRefVertices * 5;
        }
        else
        {
            NumPrevRefVertices = ( NumPrevRefVertices - 1 ) * 5;
        }
    */
    // 对非 strip 起点的三角形的 ref 顶点进行解码时, 可能不仅需要当前三角形自己的 ref 顶点, 还有可能需要前一个三角形的 head 顶点来恢复共享边上的顶点, 所以这里会额外往前读一个 5-bit delta
    uint32 IndexData = ReadUnalignedDword( StripIndexData, ( NumPrevRefVertices + ~IsStart ) * 5 ); // -1 if not Start

    // 当前三角形是 strip 起点三角形, strip 起点三角形的 3 个顶点都要独立计算
    if( IsStart )
    {
        // 因为 IsLeft 和 IsRef 的值是 -1: true, 0: false
        // 所以这里是负数形式的 ref 顶点数量
        // 0/1/2/3 个 ref 顶点分别对应的 MinusNumRefVertices 值是 0/-1/-2/-3
        const int32 MinusNumRefVertices = ( IsLeft << 1 ) + IsRef;

        // 当前最新的新顶点索引
        uint32 NextVertex = NumPrevNewVertices;

        // 如果至少有 1 个 ref 顶点, 解码 OutIndices[ 0 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到顶点索引; 否则 OutIndices[ 0 ] 是一个新顶点, 分配当前最新的新顶点索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -1 ) { OutIndices[ 0 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 0 ] = NextVertex++; }
        // 至少有 2 个 ref 顶点, 继续解码 OutIndices[ 1 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到顶点索引; 否则 OutIndices[ 1 ] 是一个新顶点, 分配当前最新的新顶点索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -2 ) { OutIndices[ 1 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 1 ] = NextVertex++; }
        // 3 个顶点都是 ref 顶点, 继续解码 OutIndices[ 2 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到顶点索引; 否则 OutIndices[ 2 ] 是一个新顶点, 分配当前最新的新顶点索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -3 ) { OutIndices[ 2 ] = BaseVertex - ( IndexData & 31u );                  } else { OutIndices[ 2 ] = NextVertex++; }
    }
    else
    {
        // Handle two first vertices

        // 当前三角形是非 strip 起点三角形, 也就是说它是 strip 延伸扩展出来的, 所以理论上它的前 2 个顶点来自已有边, 都是 ref 顶点

        // 前一个三角形在当前 DWORD 内的 bit 位置
        const uint32 PrevBitIndex = BitIndex - 1u;
        // 前一个三角形是否是 strip 起点三角形
        const int32 IsPrevStart = BitFieldExtractI32( SMask, 1, PrevBitIndex);
        // 前一个三角形的 head 顶点是否是 ref 顶点
        const int32 IsPrevHeadRef = BitFieldExtractI32( HeadRefVertexMask, 1, PrevBitIndex );
        //const int NumPrevNewVerticesInTriangle = IsPrevStart ? ( 3u - ( bfe_u32( /*SLMask*/ LMask, PrevBitIndex, 1 ) << 1 ) - bfe_u32( /*SMask &*/ WMask, PrevBitIndex, 1 ) ) : /*1u - IsPrevRefVertex*/ 0u;
        // 计算前一个三角形自身有多少个新顶点
        const int32 NumPrevNewVerticesInTriangle = IsPrevStart & ( 3u - ( (BitFieldExtractU32( /*SLMask*/ LMask, 1, PrevBitIndex) << 1 ) | BitFieldExtractU32( /*SMask &*/ WMask, 1, PrevBitIndex) ) );
        
        //OutIndices[ 1 ] = IsPrevRefVertex ? ( BaseVertex - ( IndexData & 31u ) + NumPrevNewVerticesInTriangle ) : BaseVertex; // BaseVertex = ( NumPrevNewVertices - 1 );
        // 解码 OutIndices[ 1 ]:
        //   - 如果前一个三角形的 head 顶点是 ref 顶点, 则用 ( IndexData & 31u ) 取前一个 ref delta, 再加上 NumPrevNewVerticesInTriangle 做前一个三角形是 strip 起点情况的偏移修正
        //   - 如果前一个三角形的 head 顶点不是 ref 顶点, 则直接使用 BaseVertex
        OutIndices[ 1 ] = BaseVertex + ( IsPrevHeadRef & ( NumPrevNewVerticesInTriangle - ( IndexData & 31u ) ) );
        //OutIndices[ 2 ] = IsRefVertex ? ( BaseVertex - bfe_u32( IndexData, 5, 5 ) ) : NumPrevNewVertices;
        // 解码 OutIndices[ 2 ]:
        //   - 如果当前三角形的 head 顶点是 ref 顶点, 则用 IndexData 的第 2 个 5-bit delta 解码
        //   - 如果当前三角形的 head 顶点不是 ref 顶点, 则直接是 NumPrevNewVertices
        OutIndices[ 2 ] = NumPrevNewVertices + ( IsRef & ( -1 - BitFieldExtractU32( IndexData, 5, 5 ) ) );

        // We have to search for the third vertex. 
        // Left triangles search for previous Right/Start. Right triangles search for previous Left/Start.

        // 解码 OutIndices[ 0 ]: 如果是左扩展, 则要找之前最近的右扩展或者 strip 起点三角形; 如果是右扩展, 则需要找之前最近的左扩展或 strip 起点三角形;

        // SMask | ( LMask ^ IsLeft ):
        //   - SMask: 保证 strip 起点三角形总是候选;
        //   - ( LMask ^ IsLeft ): 如果当前三角形是左扩展, LMask ^ (-1) = ~LMask, 找右扩展; 如果当期三角形是右扩展, LMask ^ 0 = LMask, 找左扩展
        const uint32 SearchMask = SMask | ( LMask ^ IsLeft );               // SMask | ( IsRight ? LMask : RMask );

        // 当前三角形之前的候选里找最高 bit 位代表的三角形, 也就是离当前三角形最近的候选三角形
        const uint32 FoundBitIndex = FMath::FloorLog2( SearchMask & PrevBitsMask );
        // 这个候选三角形是不是 strip 起点三角形
        const int32 IsFoundCaseS = BitFieldExtractI32( SMask, 1, FoundBitIndex );       // -1: true, 0: false

        // 当前 32-triangle DWORD 内, 这个候选三角形之前所有的三角形的 bit mask
        const uint32 FoundPrevBitsMask = ( 1u << FoundBitIndex ) - 1u;
        // 在当前 32-triangle DWORD 内候选三角形之前累计的 ref 顶点数
        int32 FoundCurrentDwordNumPrevRefVertices = ( FMath::CountBits( SLMask & FoundPrevBitsMask ) << 1 ) + FMath::CountBits( WMask & FoundPrevBitsMask );
        // 在当前 32-triangle DWORD 内候选三角形之前累计的 ref 顶点数
        int32 FoundCurrentDwordNumPrevNewVertices = ( FMath::CountBits( SMask & FoundPrevBitsMask ) << 1 ) + FoundBitIndex - FoundCurrentDwordNumPrevRefVertices;

        // 候选三角形之前累计的总的 ref 顶点数, 这个数量包括: 当前 32-triangle DWORD 之前累计的 ref 顶点数 以及 当前 32-triangle DWORD 内候选三角形之前累计的 ref 顶点数
        int32 FoundNumPrevNewVertices = NumPrevNewVerticesBeforeDword + FoundCurrentDwordNumPrevNewVertices;
        // 候选三角形之前累计的总的新顶点数, 这个数量包括: 当前 32-triangle DWORD 之前累计的新顶点数 以及 当前 32-triangle DWORD 内候选三角形之前累计的新顶点数
        int32 FoundNumPrevRefVertices = NumPrevRefVerticesBeforeDword + FoundCurrentDwordNumPrevRefVertices;

        // 候选三角形的 ref 顶点数量
        const uint32 FoundNumRefVertices = (BitFieldExtractU32( LMask, 1, FoundBitIndex ) << 1 ) + BitFieldExtractU32( WMask, 1, FoundBitIndex );
        // 候选三角形的前一个三角形的 head 顶点是否是 ref 顶点
        const uint32 IsBeforeFoundRefVertex = BitFieldExtractU32( HeadRefVertexMask, 1, FoundBitIndex - 1 );

        // ReadOffset: Where is the vertex relative to triangle we searched for?
        // 读取候选三角形的 5-bit delta 时的偏移:
        //   - 如果候选三角形是 strip 起点且当前三角形是左扩展, 则要读候选三角形的第 2 个 5-bit delta, 所以 ReadOffset = -1, 后续 ( FoundNumPrevRefVertices - ReadOffset ) 也就是 ( FoundNumPrevRefVertices + 1 );
        //   - 如果候选三角形是 strip 起点且当前三角形是右扩展, 则读第 1 个 5-bit delta, 所以 ReadOffset = 0;
        //   - 如果候选三角形不是 strip 起点, 则读候选三角形前一个 5-bit delta, 所以 ReadOffset = 1;
        const int32 ReadOffset = IsFoundCaseS ? IsLeft : 1;
        const uint32 FoundIndexData = ReadUnalignedDword( StripIndexData, ( FoundNumPrevRefVertices - ReadOffset ) * 5 );

        // ( FoundNumPrevNewVertices - 1u ): 是 FoundBaseVertex
        // BitFieldExtractU32( FoundIndexData, 5, 0 ): 是 5-bit delta
        // 解码真实顶点索引
        const uint32 FoundIndex = ( FoundNumPrevNewVertices - 1u ) - BitFieldExtractU32( FoundIndexData, 5, 0 );

        // 决定是否应该使用 FoundIndex
        // 如果候选三角形是 strip 起点, 则:
        //   - 如果当前三角形是右扩展, 则候选三角形至少要有 1 个 ref 顶点;
        //   - 如果当前三角形是左扩展, 则候选三角形至少要有 2 个 ref 顶点;
        // 而如果候选三角形不是 strip 起点, 则候选三角形的前一个三角形的 head 顶点必须是 ref 顶点
        bool bCondition = IsFoundCaseS ? ( (int32)FoundNumRefVertices >= 1 - IsLeft ) : (IsBeforeFoundRefVertex != 0u);
        // 如果不能用 FoundIndex, 那么应该使用哪个新顶点索引:
        //   - 候选三角形是 strip 起点并且没有 ref 顶点时:
        //     - 当前三角形是右扩展, 新顶点索引是: FoundNumPrevNewVertices;
        //     - 当前三角形是左扩展, 新顶点索引是: FoundNumPrevNewVertices + 1;
        //   - 候选三角形是 strip 起点但有 ref 顶点时, 新顶点索引是: FoundNumPrevNewVertices;
        //   - 候选三角形不是 strip 起点, 新顶点索引是: FoundNumPrevNewVertices - 1;
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

因为在 stripify 的过程中对每个输出的顶点做 32 顶点窗口约束可能会复制老顶点输出新顶点从而导致 cluster 的顶点数量增加，所以最后还会检查 stripify 之后的 cluster 中的顶点数量是否超过 Nanite 限制的 `NANITE_MAX_CLUSTER_VERTICES`，如果超过了则会将此 cluster 按三角形数量二分，另外还会对拆分后的 cluster 再次进行按材质排序三角形以及 stripify 的处理，具体逻辑在 `BuildClusterFromClusterTriangleRange()` 方法中，这里就不再展开分析源码了：

```cpp
// Constrain clusters
const uint32 NumOldClusters = Clusters.Num();
for( uint32 i = 0; i < NumOldClusters; i++ )
{
    TotalNewTriangles += Clusters[ i ].NumTris;
    TotalNewVertices += Clusters[ i ].Verts.Num();
    
    // 二分顶点数量超过 NANITE_MAX_CLUSTER_VERTICES 的 cluster
    if( Clusters[ i ].Verts.Num() > NANITE_MAX_CLUSTER_VERTICES && Clusters[i].NumTris )
    {
        FCluster ClusterA, ClusterB;
        // 按三角形数量二分
        uint32 NumTrianglesA = Clusters[ i ].NumTris / 2;
        uint32 NumTrianglesB = Clusters[ i ].NumTris - NumTrianglesA;
        // 分别构建 2 个子 cluster, 并且继续对它们进行按材质排序三角形和 stripify 的处理
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

//todo: New
---
Nanite 是按材质段（`MaterialRange`）分段进行 stripify 的，这样可以保持材质分组不被打乱，避免 strip 跨材质。首先会初始化相关数据：

```cpp
// 清掉旧的 strip index 字节流
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

`Cluster.StripIndexData` 本质上是一个连续、紧凑的 5-bit delta 流，Nanite 通过 `BitWriter.PutBits(BaseVertex - Index, 5)` 以低位优先的顺序向其中写入连续的 5-bit delta，每满 8 bits 就会吐出一个 `uint8` 并添加进 `Cluster.StripIndexData` 中。这里的 delta 引用的是新顶点序列中已经输出过的某个新顶点，也就是用当前三角形输出前的新顶点序列末尾索引减去被引用的新顶点索引。

按材质段 stripify 时，首先会将当前材质段的所有三角形标记为候选状态，后续 stripify 时只处理候选三角形：

```cpp
// 将当前材质段中的三角形标记为候选状态
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
    // 最后按位异或将当前材质段三角形所属 DWORD 对应的 bits 置为 1.
    // 举个例子: StartMask = bits 0..9, EndMask = bits 0..14, 那么 StartMask ^ EndMask = bits 10..14
    Context.TrianglesEnabled[ i ] |= StartMask ^ EndMask;
}
```

然后会通过 `NewScoreTriangle` 为每个候选三角形打分，最后选取最高分的三角形作为当前 strip 起点。

另外，对于作为 strip 起点的三角形，Nanite 要求它的 3 个旧顶点按 strip 编码需要的局部顺序 stripify 后必须满足：**ref 顶点必须在前，新顶点必须连续出现在后面**，也就是下面这几种情况：

```text
1. ref, ref, ref
2. ref, ref, new
3. ref, new, new
4. new, new, new
```

所以 Nanite 会先遍历所有的候选三角形，然后通过 `Context.OldToNewVertex` 获取它们的 3 个旧顶点在新顶点序列中的索引 `NewIndex`，根据 `NewIndex` 和 32 顶点窗口约束决定每个顶点是作为 ref 顶点还是新顶点编码，只有最终编码形态满足以上几种情况之一的三角形才会进入后续打分环节：

```cpp
// 计算当前顶点所对应的三角形 Corner
uint32 TriangleCorner = SetCorner( TriangleIndex, Corner );

{
    // Is it viable WRT the constraint that new vertices should always be at the end.
    // 按 stripify 编码顺序取三角形的 3 个旧顶点
    uint32 OldIndex0 = Cluster.Indexes[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
    uint32 OldIndex1 = Cluster.Indexes[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];
    uint32 OldIndex2 = Cluster.Indexes[ CornerToIndex( TriangleCorner ) ];

    // 查看这 3 个旧顶点是否已经对应到新顶点序列中的某个已输出新顶点
    uint32 NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
    uint32 NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
    uint32 NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];
    uint32 NumVerts = Context.NumVertices + ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );
    
    // 如果旧顶点已经在新顶点序列中, 则表示它有机会作为 ref 顶点编码
    // 在这里模拟 32 顶点窗口约束: 如果 stripify 当前三角形后, 当前旧顶点引用的新顶点距离新顶点序列末尾太远, 导致 5 bits 存储不下这个相对距离, 则把它重新作为一个新顶点编码
    while(true)
    {
        if( NewIndex0 != INVALID_INDEX && NumVerts - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex0 = INVALID_INDEX; NumVerts++; continue; }
        if( NewIndex1 != INVALID_INDEX && NumVerts - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex1 = INVALID_INDEX; NumVerts++; continue; }
        if( NewIndex2 != INVALID_INDEX && NumVerts - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex2 = INVALID_INDEX; NumVerts++; continue; }
        break;
    } 

    // Nanite 要求可以作为 strip 起点的三角形, 其 3 个旧顶点在编码形态上只有下面几种情况:
    // ref, ref, ref
    // ref, ref, new
    // ref, new, new
    // new, new, new
    // 所以这里检查当前候选三角形是否满足“ref 顶点在前, 新顶点连续出现在后面”
    uint32 Mask = ( NewIndex0 == INVALID_INDEX ? 1u : 0u ) | ( NewIndex1 == INVALID_INDEX ? 2u : 0u ) | ( NewIndex2 == INVALID_INDEX ? 4u : 0u );

    // 跳过不满足的候选三角形
    if( Mask != 0u && Mask != 4u && Mask != 6u && Mask != 7u )
    {
        continue;
    }
}
```

对于满足要求的候选三角形，根据其拓扑上下文通过 `NewScoreTriangle` 为其打分，并记录得分最高的候选三角形：

```cpp
// 候选三角形对面的邻居三角形
uint32 Opposite = OppositeCorner[ CornerToIndex( TriangleCorner ) ];
// 候选三角形左侧的邻居三角形
uint32 LeftCorner = OppositeCorner[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
// 候选三角形右侧的邻居三角形
uint32 RightCorner = OppositeCorner[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];

// 对面的邻居三角形是否存在并且也是候选三角形
bool bHasOpposite = Opposite != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( Opposite ) );
// 左侧的邻居三角形是否存在并且也是候选三角形
bool bHasLeft = LeftCorner != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( LeftCorner ) );
// 右侧的邻居三角形是否存在并且也是候选三角形
bool bHasRight = RightCorner != INVALID_CORNER && Context.TriangleEnabled( CornerToTriangle( RightCorner ) );

// 根据候选三角形拓扑上下文信息为其打分
int32 Score = NewScoreTriangle( Context, TriangleIndex, true, bHasOpposite, bHasLeft, bHasRight );
if( Score > BestScore )
{
    // 记录候选三角形
    StartCorner = TriangleCorner;
    // 记录最高分
    BestScore = Score;
}
else if( Score == BestScore )
{
    // 分数相同时根据三角形 3 个旧顶点位置和的 X 值决定谁更优先
    float Priority = TrianglePriorities[ TriangleIndex ];
    if( Priority > BestPriority )
    {
        // 记录候选三角形
        StartCorner = TriangleCorner;
        // 记录最高分
        BestScore = Score;
        BestPriority = Priority;
    }
}
```

如果当前材质段最终没有找到适合当 strip 起点的三角形，则说明当前材质段中已经没有可继续输出的候选三角形，此时会结束当前材质段的 stripify：

```cpp
// 当前材质段内没有找到合适的 strip 起点三角形, 则结束当前材质段
if( StartCorner == INVALID_CORNER )
    break;
```

如果找到合适的候选三角形后，则继续 stripify 当前材质段，首先会 stripify 当前 strip 选中的起点三角形：

```cpp
{
    // 当前 strip 起点三角形 stripify 后属于第几个 DWORD
    uint32 TriangleDword = Context.NumTriangles >> 5;
    // 当前三角形输出前的新顶点序列末尾索引, 用来计算 ref delta
    uint32 BaseVertex = Context.NumVertices - 1;

    // stripify 当前 strip 起点三角形
    uint32 NumNewVertices = VisitTriangle(Context, StartCorner, true, false);

    // 为当前 strip 起点三角形的每个 ref 顶点写 5-bit delta
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
    // 统计所属 DWORD 新增顶点数
    NumNewVerticesInDword[TriangleDword] += NumNewVertices;
    // 统计所属 DWORD ref 顶点数
    NumRefVerticesInDword[TriangleDword] += 3u - NumNewVertices;
}
```

我们知道，在 `VisitTriangle` 方法中如果一个旧顶点被编码为一个新顶点，那么 `Context.NumVertices++`，也就是说：**`Context.NumVertices` 中统计的是 stripify 过程中已经输出到新顶点序列中的新顶点数量**。

而根据上面的逻辑可以看到，在 stripify 起点三角形之前，会先记录当前已经输出的新顶点序列里的最后一个顶点的索引 `BaseVertex`，而在 `VisitTriangle` 方法中如果一个旧顶点被编码为一个 ref 顶点，那么它引用的就是 stripify 起点三角形之前新顶点序列中已经存在的顶点，引用的顶点的索引可以根据映射 `Context.OldToNewVertex` 获取。那么，我们也就知道了，对于一个三角形的 ref 顶点，它所对应的 5-bit delta 指的是：**从这个三角形输出前的新顶点序列末尾往前数多少个位置**，这个位置所对应的新顶点就是它引用的顶点。

找到并 stripify 当前 strip 的起点三角形后，下面就从起点三角形开始延伸扩展当前 strip。首先，如果当前已经输出的三角形数量刚好到达 32-triangle DWORD 边界，则跳出当前 strip 的延伸扩展，重新去开启一个新的 strip。这里不是单纯限制一个 strip 最多 32 个三角形，而是因为 `StripBitmasks`、`NumNewVerticesInDword` 和 `NumRefVerticesInDword` 都是按 32 个三角形为一个 DWORD 来组织的，当前实现不让一个 strip 跨过 DWORD 边界：

```cpp
// 从 strip 起点三角形开始延伸扩展当前 strip
uint32 CurrentCorner = StartCorner;
while (true)
{
    // 如果当前已经输出的三角形数量到达 32-triangle DWORD 边界, 则结束当前 strip, 重新开启一个新的 strip
    if ((Context.NumTriangles & 31u) == 0u)
        break;

    ...
}
```

然后分别找左侧和右侧的邻居三角形，并根据它们的拓扑上下文给它们打分，最终选择分数更高的一侧对 strip 进行延伸扩展：

```cpp
// 左侧邻居三角形
uint32 LeftCorner = OppositeCorner[CornerToIndex(NextCorner(CurrentCorner))];
// 右侧邻居三角形
uint32 RightCorner = OppositeCorner[CornerToIndex(PrevCorner(CurrentCorner))];
CurrentCorner = INVALID_CORNER;

int32 LeftScore = INT_MIN;
// 左侧的邻居三角形是否存在并且也是候选三角形
if (LeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftCorner)))
{
    uint32 LeftLeftCorner = OppositeCorner[CornerToIndex(NextCorner(LeftCorner))];
    uint32 LeftRightCorner = OppositeCorner[CornerToIndex(PrevCorner(LeftCorner))];
    bool bLeftLeftCorner = LeftLeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftLeftCorner));
    bool bLeftRightCorner = LeftRightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(LeftRightCorner));

    // 根据左侧邻居三角形的拓扑上下文给其打分
    LeftScore = NewScoreTriangle(Context, CornerToTriangle(LeftCorner), false, true, bLeftLeftCorner, bLeftRightCorner);
    CurrentCorner = LeftCorner;
}

bool bIsRight = false;
// 右侧的邻居三角形是否存在并且也是候选三角形
if (RightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightCorner)))
{
    uint32 RightLeftCorner = OppositeCorner[CornerToIndex(NextCorner(RightCorner))];
    uint32 RightRightCorner = OppositeCorner[CornerToIndex(PrevCorner(RightCorner))];
    bool bRightLeftCorner = RightLeftCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightLeftCorner));
    bool bRightRightCorner = RightRightCorner != INVALID_CORNER && Context.TriangleEnabled(CornerToTriangle(RightRightCorner));

    // 根据右侧邻居三角形的拓扑上下文给其打分
    int32 Score = NewScoreTriangle(Context, CornerToTriangle(RightCorner), false, false, bRightLeftCorner, bRightRightCorner);

    // 根据评分决定往左侧还是右侧延伸扩展 strip
    if (Score > LeftScore)
    {
        CurrentCorner = RightCorner;
        bIsRight = true;
    }
}

// 没有找到合适的邻居三角形, 则跳出当前 strip 的延伸扩展
if (CurrentCorner == INVALID_CORNER)
    break;
```

再进行 32 顶点窗口约束的检查：

```cpp
{
    const uint32 OldIndex0 = Cluster.Indexes[CornerToIndex(NextCorner(CurrentCorner))];
    const uint32 OldIndex1 = Cluster.Indexes[CornerToIndex(PrevCorner(CurrentCorner))];
    const uint32 OldIndex2 = Cluster.Indexes[CornerToIndex(CurrentCorner)];

    const uint32 NewIndex0 = Context.OldToNewVertex[OldIndex0];
    const uint32 NewIndex1 = Context.OldToNewVertex[OldIndex1];
    const uint32 NewIndex2 = Context.OldToNewVertex[OldIndex2];

    // 延伸三角形必须沿当前 strip 的边接上, 所以前 2 个顶点必须已经存在于新顶点序列中, 后续由 strip 拓扑隐含复用
    check(NewIndex0 != INVALID_INDEX);
    check(NewIndex1 != INVALID_INDEX);

    // 预估 stripify 这个三角形后的已输出新顶点数量
    const uint32 NextNumVertices = Context.NumVertices + ((NewIndex2 == INVALID_INDEX || Context.NumVertices - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE) ? 1u : 0u);

    // 判断前 2 个隐含复用的顶点在当前三角形输出后是否仍满足 32 顶点窗口约束
    if (NextNumVertices - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ||
        NextNumVertices - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE)
        break;
}
```

最后 stripify 选中的邻居三角形对 strip 进行延伸扩展：

```cpp
{
    // 选中的邻居三角形所属的 DWORD
    uint32 TriangleDword = Context.NumTriangles >> 5;
    // 当前三角形输出前的新顶点序列末尾索引
    uint32 BaseVertex = Context.NumVertices - 1;
    // stripify 选中的邻居三角形
    uint32 NumNewVertices = VisitTriangle(Context, CurrentCorner, false, bIsRight);
    // 延伸扩展的三角形最多只能输出 1 个新顶点
    check(NumNewVertices <= 1u);
    // 如果新引入的那个顶点也是 ref 顶点, 则写 1 个 5-bit delta
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

以上就是整个 stripify 的逻辑，stripify 结束之后，会先调用 `BitWriter.Flush(sizeof(uint32))` 方法将还未满 8 bits 的已写 bits 作为一个 `uint8` 追加进 `Cluster.StripIndexData` 中，然后还会将 `Cluster.StripIndexData` 以 4 字节对齐：

```cpp
void Flush(uint32 Alignment=1)
{
    // 先把 PendingBits 中还没满 8 bits 的已写 bits 作为一个 uint8 追加进 Cluster.StripIndexData 中
    if (NumPendingBits > 0)
        Buffer.Add((uint8)PendingBits);
    // 如果 Cluster.StripIndexData.Num() 不是 4 的倍数, 就追加 (uint8)0, 直到 4 字节对齐
    while (Buffer.Num() % Alignment != 0)
        Buffer.Add(0);

    // 清空 BitWriter 内部暂存数据
    PendingBits = 0;
    NumPendingBits = 0;
}
```

然后根据 stripify 的结果重建 cluster 的顶点数据 `Verts`：

```cpp
// Reorder vertices
// stripify 最终的新顶点数, 它可能大于旧顶点数, 因为超出 32 顶点窗口约束的旧顶点会被重新作为一个新顶点编码
const uint32 NumNewVertices = Context.NumVertices;

// 把旧的 Cluster.Verts 拷贝到 OldVertices 并清空旧的 Cluster.Verts
FVertexArray OldVertices( Cluster.Verts.Format );
Swap( OldVertices, Cluster.Verts );

// 根据 Context.NewToOldVertex 映射重建 Cluster.Verts
// 如果某个旧顶点因为超出 32 顶点窗口约束被重新作为一个新顶点编码, 那么 Context.NewToOldVertex 中会有多个索引指向同一个旧顶点, 这里会复制多个相同的 Vert 到 Cluster.Verts 中
Cluster.Verts.Reserve( NumNewVertices );
for( uint32 i = 0; i < NumNewVertices; i++ )
{
    Cluster.Verts.Add( &OldVertices.GetPosition( Context.NewToOldVertex[i] ) );
}

// stripify 最终输出的三角形数必须等于原 cluster 的三角形数
check( Context.NumTriangles == NumOldTriangles );
```

需要注意的是，因为在 stripify 的过程中，有些旧顶点会因为超出 32 顶点窗口约束而被重新编码成一个新顶点，也就是说 stripify 后最终的新顶点数可能会超过原顶点数，并且 `Context.NewToOldVertex` 中会有多个索引指向同一个旧顶点，这也导致最终 `Cluster.Verts` 中会存在多个相同的旧 Vert。

重建完新的 `Cluster.Verts` 数据后，继续将 stripify 信息写进 `Cluster.StripDesc` 中：

```cpp
// 计算每个 DWORD 之前累计的新增顶点数:
// DWORD 0 之前 = 0, 不用存
// DWORD 1 之前 = DWORD 0 新增顶点数
// DWORD 2 之前 = DWORD 0 + DWORD 1 新增顶点数
// DWORD 3 之前 = DWORD 0 + DWORD 1 + DWORD 2 新增顶点数
uint32 NumPrevNewVerticesBeforeDwords1 = NumNewVerticesInDword[0];
uint32 NumPrevNewVerticesBeforeDwords2 = NumNewVerticesInDword[1] + NumPrevNewVerticesBeforeDwords1;
uint32 NumPrevNewVerticesBeforeDwords3 = NumNewVerticesInDword[2] + NumPrevNewVerticesBeforeDwords2;
// 每个累计值用 10 bit 存，所以必须小于 1024
check(NumPrevNewVerticesBeforeDwords1 < 1024 && NumPrevNewVerticesBeforeDwords2 < 1024 && NumPrevNewVerticesBeforeDwords3 < 1024);
// 编码进 1 个 uint32
StripDesc.NumPrevNewVerticesBeforeDwords = (NumPrevNewVerticesBeforeDwords3 << 20) | (NumPrevNewVerticesBeforeDwords2 << 10) | NumPrevNewVerticesBeforeDwords1;

// 计算每个 DWORD 之前累计的 ref 顶点数:
// DWORD 0 之前 = 0, 不用存
// DWORD 1 之前 = DWORD 0 ref 顶点数
// DWORD 2 之前 = DWORD 0 + DWORD 1 ref 顶点数
// DWORD 3 之前 = DWORD 0 + DWORD 1 + DWORD 2 ref 顶点数
uint32 NumPrevRefVerticesBeforeDwords1 = NumRefVerticesInDword[0];
uint32 NumPrevRefVerticesBeforeDwords2 = NumRefVerticesInDword[1] + NumPrevRefVerticesBeforeDwords1;
uint32 NumPrevRefVerticesBeforeDwords3 = NumRefVerticesInDword[2] + NumPrevRefVerticesBeforeDwords2;
// 每个累计值用 10 bit 存，所以必须小于 1024
check(NumPrevRefVerticesBeforeDwords1 < 1024 && NumPrevRefVerticesBeforeDwords2 < 1024 && NumPrevRefVerticesBeforeDwords3 < 1024);
// 编码进 1 个 uint32
StripDesc.NumPrevRefVerticesBeforeDwords = (NumPrevRefVerticesBeforeDwords3 << 20) | (NumPrevRefVerticesBeforeDwords2 << 10) | NumPrevRefVerticesBeforeDwords1;

// 复制 strip bitmask
static_assert(sizeof(StripDesc.Bitmasks) == sizeof(Context.StripBitmasks), "");
FMemory::Memcpy(StripDesc.Bitmasks, Context.StripBitmasks, sizeof(StripDesc.Bitmasks));
```

首先 Nanite 一共使用 4 个 DWORD 表示 1 个 cluster 的 128 个三角形，其中每个 DWORD 的 32 bits 表示 32 个三角形。

从 `StripDesc.NumPrevNewVerticesBeforeDwords` 的最低位开始，每 10 bits 记录了**某个 DWORD 之前累计的新增顶点数**。因为 DWORD 0 之前的累计值必然是 0，所以不用存；`StripDesc.NumPrevNewVerticesBeforeDwords` 的 0-9 bit 记录的是 DWORD 1 之前的累计新增顶点数，也就是 DWORD 0 中表示的三角形累计新增的新顶点数；10-19 bit 记录的是 DWORD 2 之前的累计新增顶点数，也就是 DWORD 0 和 DWORD 1 的累计值；20-29 bit 记录的是 DWORD 3 之前的累计新增顶点数，也就是 DWORD 0、DWORD 1 和 DWORD 2 的累计值。类似的 `StripDesc.NumPrevRefVerticesBeforeDwords` 则是从最低位开始，每 10 bits 记录了**某个 DWORD 之前累计的 ref 顶点数**。

`StripDesc.Bitmasks` 则是记录每个三角形的 strip 编码信息，首先 `[4]` 表示的是 4 个 DWORD，根据三角形索引 `TriangleIndex >> 5` 可以快速得到这个三角形所属的 DWORD，而通过 `( NewTriangleIndex & 31u )` 又可以得到这个三角形在 `StripDesc.Bitmasks[TriangleIndex >> 5]` 这 3 个 `uint32` 中对应的 bit 位，每个 `uint32` 中对应 bit 位中记录了这个三角形的 strip 编码信息：

其中 `StripDesc.Bitmasks[TriangleIndex >> 5][0]` 对应的 bit 位中记录此三角形是否是一个 strip 的起点，bit 位为 1 则代表是一个 strip 的起点；

如果此三角形是一个 strip 的起点，那么对应的在 `StripDesc.Bitmasks[TriangleIndex >> 5][1]` 和 `StripDesc.Bitmasks[TriangleIndex >> 5][2]` 中这 2 bit 组合起来记录了这个 strip 起点三角形中 ref 顶点数量（`0..3`），其中 `StripDesc.Bitmasks[TriangleIndex >> 5][1]` 是高位，`StripDesc.Bitmasks[TriangleIndex >> 5][2]` 是低位。

而如果此三角形不是一个 strip 的起点，也就是说它是一个 strip 的延伸，那么对应的在 `StripDesc.Bitmasks[TriangleIndex >> 5][1]` 中的 bit 位记录这个三角形是从左边还是右边延伸过来的，对应的在 `StripDesc.Bitmasks[TriangleIndex >> 5][2]` 中的 bit 位则记录这个三角形新引入的那个顶点是否是 ref 顶点。延伸三角形的前 2 个顶点并不会在当前三角形里再次显式编码，而是由 strip 拓扑从已经输出过的三角形中隐含复用出来。

最后，则是 Nanite 通过 stripify 信息重建 cluster 的顶点索引 `Indexes`，核心的逻辑在 `UnpackTriangleIndices()` 方法中，下面通过对重建 `Indexes` 的源码分析加深对 stripify 后数据结构的理解：

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
    //   - 而对于 strip 起点三角形, 它表示的是该三角形 ref 顶点数的高位 (与 IsRef mask 中的低位组合成 2 bits 记录 strip 起点三角形的 ref 顶点数);
    const uint32 LMask = StripDesc.Bitmasks[ DwordIndex ][ 1 ];
    // WMask:
    //   - 对于非 strip 起点三角形, 它表示该三角形新引入的那个顶点是否是 ref 顶点;
    //   - 而对于 strip 起点三角形, 它表示的是该三角形 ref 顶点数的低位 (与 IsLeft mask 中的高位组合成 2 bits 记录 strip 起点三角形的 ref 顶点数);
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

    // BaseVertex 是: 当前三角形之前, 最后一个输出的新顶点的索引, 也就是最近新顶点的索引
    // 而 ref 顶点的 5-bit delta 编码的是: 引用的顶点相对最近新顶点的距离
    // 所以这里先算出最近新顶点的索引 BaseVertex
    const uint32 BaseVertex = NumPrevNewVertices - 1u;

    // StripIndexData 里每个 ref 顶点占 5-bit, 对于 strip 起点三角形, 它的第一个 ref 顶点的 5-bit delta 从 NumPrevRefVertices * 5 开始读
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
    // 对非 strip 起点三角形进行解码时, 可能不仅需要当前三角形自己的 head/ref delta, 还可能需要前一个三角形的 head/ref delta 来恢复共享边上的顶点, 所以这里会额外往前读一个 5-bit delta
    uint32 IndexData = ReadUnalignedDword( StripIndexData, ( NumPrevRefVertices + ~IsStart ) * 5 ); // -1 if not Start

    // 当前三角形是 strip 起点三角形, strip 起点三角形的 3 个顶点都要独立计算
    if( IsStart )
    {
        // 因为 IsLeft 和 IsRef 的值是 -1: true, 0: false
        // 所以这里是负数形式的 ref 顶点数量
        // 0/1/2/3 个 ref 顶点分别对应的 MinusNumRefVertices 值是 0/-1/-2/-3
        const int32 MinusNumRefVertices = ( IsLeft << 1 ) + IsRef;

        // 下一个可分配的新顶点索引
        uint32 NextVertex = NumPrevNewVertices;

        // 如果至少有 1 个 ref 顶点, 解码 OutIndices[ 0 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到顶点索引; 否则 OutIndices[ 0 ] 是一个新顶点, 分配下一个可分配的新顶点索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -1 ) { OutIndices[ 0 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 0 ] = NextVertex++; }
        // 至少有 2 个 ref 顶点, 继续解码 OutIndices[ 1 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到顶点索引; 否则 OutIndices[ 1 ] 是一个新顶点, 分配下一个可分配的新顶点索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -2 ) { OutIndices[ 1 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 1 ] = NextVertex++; }
        // 3 个顶点都是 ref 顶点, 继续解码 OutIndices[ 2 ], 从 IndexData 低 5 bit 取 delta, BaseVertex - delta 得到顶点索引; 否则 OutIndices[ 2 ] 是一个新顶点, 分配下一个可分配的新顶点索引, 然后 NextVertex++
        if( MinusNumRefVertices <= -3 ) { OutIndices[ 2 ] = BaseVertex - ( IndexData & 31u );                  } else { OutIndices[ 2 ] = NextVertex++; }
    }
    else
    {
        // Handle two first vertices

        // 当前三角形是非 strip 起点三角形, 也就是说它是 strip 延伸扩展出来的, 它的前 2 个顶点来自已有边, 需要通过 strip 拓扑从前面的三角形推导出来

        // 前一个三角形在当前 DWORD 内的 bit 位置
        const uint32 PrevBitIndex = BitIndex - 1u;
        // 前一个三角形是否是 strip 起点三角形
        const int32 IsPrevStart = BitFieldExtractI32( SMask, 1, PrevBitIndex);
        // 前一个三角形的 head 顶点是否是 ref 顶点
        const int32 IsPrevHeadRef = BitFieldExtractI32( HeadRefVertexMask, 1, PrevBitIndex );
        //const int NumPrevNewVerticesInTriangle = IsPrevStart ? ( 3u - ( bfe_u32( /*SLMask*/ LMask, PrevBitIndex, 1 ) << 1 ) - bfe_u32( /*SMask &*/ WMask, PrevBitIndex, 1 ) ) : /*1u - IsPrevRefVertex*/ 0u;
        // 计算前一个三角形自身有多少个新顶点
        const int32 NumPrevNewVerticesInTriangle = IsPrevStart & ( 3u - ( (BitFieldExtractU32( /*SLMask*/ LMask, 1, PrevBitIndex) << 1 ) | BitFieldExtractU32( /*SMask &*/ WMask, 1, PrevBitIndex) ) );
        
        //OutIndices[ 1 ] = IsPrevRefVertex ? ( BaseVertex - ( IndexData & 31u ) + NumPrevNewVerticesInTriangle ) : BaseVertex; // BaseVertex = ( NumPrevNewVertices - 1 );
        // 解码 OutIndices[ 1 ]:
        //   - 如果前一个三角形的 head 顶点是 ref 顶点, 则用 ( IndexData & 31u ) 取前一个 ref delta, 再加上 NumPrevNewVerticesInTriangle 做前一个三角形是 strip 起点情况的偏移修正
        //   - 如果前一个三角形的 head 顶点不是 ref 顶点, 则直接使用 BaseVertex
        OutIndices[ 1 ] = BaseVertex + ( IsPrevHeadRef & ( NumPrevNewVerticesInTriangle - ( IndexData & 31u ) ) );
        //OutIndices[ 2 ] = IsRefVertex ? ( BaseVertex - bfe_u32( IndexData, 5, 5 ) ) : NumPrevNewVertices;
        // 解码 OutIndices[ 2 ]:
        //   - 如果当前三角形的 head 顶点是 ref 顶点, 则用 IndexData 的第 2 个 5-bit delta 解码
        //   - 如果当前三角形的 head 顶点不是 ref 顶点, 则直接是 NumPrevNewVertices
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
        // 读取候选三角形的 5-bit delta 时的偏移:
        //   - 如果候选三角形是 strip 起点且当前三角形是左扩展, 则要读候选三角形的第 2 个 5-bit delta, 所以 ReadOffset = -1, 后续 ( FoundNumPrevRefVertices - ReadOffset ) 也就是 ( FoundNumPrevRefVertices + 1 );
        //   - 如果候选三角形是 strip 起点且当前三角形是右扩展, 则读第 1 个 5-bit delta, 所以 ReadOffset = 0;
        //   - 如果候选三角形不是 strip 起点, 则读候选三角形前一个 5-bit delta, 所以 ReadOffset = 1;
        const int32 ReadOffset = IsFoundCaseS ? IsLeft : 1;
        const uint32 FoundIndexData = ReadUnalignedDword( StripIndexData, ( FoundNumPrevRefVertices - ReadOffset ) * 5 );

        // ( FoundNumPrevNewVertices - 1u ): 是 FoundBaseVertex
        // BitFieldExtractU32( FoundIndexData, 5, 0 ): 是 5-bit delta
        // 解码真实顶点索引
        const uint32 FoundIndex = ( FoundNumPrevNewVertices - 1u ) - BitFieldExtractU32( FoundIndexData, 5, 0 );

        // 决定是否应该使用 FoundIndex
        // 如果候选三角形是 strip 起点, 则:
        //   - 如果当前三角形是右扩展, 则候选三角形至少要有 1 个 ref 顶点;
        //   - 如果当前三角形是左扩展, 则候选三角形至少要有 2 个 ref 顶点;
        // 而如果候选三角形不是 strip 起点, 则候选三角形的前一个三角形的 head 顶点必须是 ref 顶点
        bool bCondition = IsFoundCaseS ? ( (int32)FoundNumRefVertices >= 1 - IsLeft ) : (IsBeforeFoundRefVertex != 0u);
        // 如果不能用 FoundIndex, 那么应该使用哪个新顶点索引:
        //   - 候选三角形是 strip 起点并且没有 ref 顶点时:
        //     - 当前三角形是右扩展, 新顶点索引是: FoundNumPrevNewVertices;
        //     - 当前三角形是左扩展, 新顶点索引是: FoundNumPrevNewVertices + 1;
        //   - 候选三角形是 strip 起点但有 ref 顶点时, 新顶点索引是: FoundNumPrevNewVertices;
        //   - 候选三角形不是 strip 起点, 新顶点索引是: FoundNumPrevNewVertices - 1;
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

因为在 stripify 的过程中对每个输出的顶点做 32 顶点窗口约束可能会复制老顶点输出新顶点从而导致 cluster 的顶点数量增加，所以最后还会检查 stripify 之后的 cluster 中的顶点数量是否超过 Nanite 限制的 `NANITE_MAX_CLUSTER_VERTICES`，如果超过了则会将此 cluster 按三角形数量二分，另外还会对拆分后的 cluster 再次进行按材质排序三角形以及 stripify 的处理，具体逻辑在 `BuildClusterFromClusterTriangleRange()` 方法中，这里就不再展开分析源码了：

```cpp
// Constrain clusters
const uint32 NumOldClusters = Clusters.Num();
for( uint32 i = 0; i < NumOldClusters; i++ )
{
    TotalNewTriangles += Clusters[ i ].NumTris;
    TotalNewVertices += Clusters[ i ].Verts.Num();
    
    // 二分顶点数量超过 NANITE_MAX_CLUSTER_VERTICES 的 cluster
    if( Clusters[ i ].Verts.Num() > NANITE_MAX_CLUSTER_VERTICES && Clusters[i].NumTris )
    {
        FCluster ClusterA, ClusterB;
        // 按三角形数量二分
        uint32 NumTrianglesA = Clusters[ i ].NumTris / 2;
        uint32 NumTrianglesB = Clusters[ i ].NumTris - NumTrianglesA;
        // 分别构建 2 个子 cluster, 并且继续对它们进行按材质排序三角形和 stripify 的处理
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
---
