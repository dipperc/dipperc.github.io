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

三角形 `Corner` 的数据类型是 `uint16`，其高 14 位编码了三角形索引 `i`，低 2 位编码了三角形 `i` 中的局部顶点索引 `0/1/2`，也就是说一个三角形 Corner 表示的是三角形 `i` 的 3 个顶点中的某一个，在上面的源码中 Nanite 使用三角形 `Corner` 代表的是**它所表示的三角形顶点对面的那条有向边**，举个例子：`Corner(i, 0)` 表示的是三角形 `i` 的第 1 个顶点 `i0`，而它代表是三角形 `i` 的有向边 `i1 -> i2`；同理，`Corner(i, 1)` 代表的是三角形 `i` 的有向边 `i2 -> i0`，`Corner(i, 2)` 代表的是三角形 `i` 的有向边 `i0 -> i1`。

`BuildTables()` 是在 stripify 之前的关键一步，它根据 cluster 的原始网格信息构建了 stripify 所需的数据：首先是 `VertexToTriangleMasks`，它记录 **cluster 内每个顶点关联了哪些三角形**，通过 `VertexToTriangleMasks[ vertex index ][ DWORD i ]` 可以快速知道某个 32-triangle DWORD 中有哪些三角形使用了顶点 `Verts[index]`；其次是 `OppositeCorner`，它记录**每个三角形 Corner 所代表的有向边的对象共享边所对应的三角形 Corner**，后续 stripify 时会根据此数据找左/右相邻三角形；最后 `TrianglePriorities` 则是保存每个三角形 3 个顶点**位置和**的 `x` 分量值，后续选 strip 起点三角形时，如果评分相同，则用它决定谁优先。

再来看看 `NewScoreVertex()`：

```cpp
auto NewScoreVertex = [ &Weights ] ( const FContext& Context, uint32 OldVertex, bool bStart, bool bHasOpposite, bool bHasLeft, bool bHasRight )
{
    // 当前旧顶点在新顶点序列中的索引
    uint16 NewIndex = Context.OldToNewVertex[ OldVertex ];

    // 默认分数为 0
    int32 CacheScore = 0;

    // 如果当前旧顶点已经输出到新顶点序列中
    if( NewIndex != INVALID_INDEX )
    {
        // 计算其对应的新顶点与目前已输出的最新顶点的偏移大小
        uint32 CachePosition = ( Context.NumVertices - 1 ) - NewIndex;
        // 偏移大小是否满足 5-bits 偏移约束
        if( CachePosition < NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE )
        {
            // 如果满足约束, 就从权重表取分数, 根据以下 5 个维度查表:
            // bStart: 当前候选三角形是否作为 strip 的起点
            // bHasOpposite: 当前 Corner 对面的邻接三角形是否可用
            // bHasLeft: 左侧邻接三角形是否可用
            // bHasRight: 右侧邻接三角形是否可用
            // CachePosition: 与目前已输出的最新顶点的偏移大小
            CacheScore = Weights.Weights[ bStart ][ bHasOpposite ][ bHasLeft ][ bHasRight ][ CachePosition ];
        }
    }

    return CacheScore;
};
```

`NewScoreVertex()` 方法的作用是给一个候选顶点打分，它根据这个顶点是否已经进入新顶点序列（是否可以作为 ref 顶点）、它所对应的新顶点与目前已经输出的最新顶点的偏移大小（作为 ref 顶点时的代价大不大）、以及它所属三角形的相邻三角形信息（所属三角形的拓扑上下文）来给这个顶点打分，打分主要依据**顶点作为 ref 顶点的代价**以及**顶点所属三角形的拓扑延伸能力**。另外提一下 `NewScoreTriangle()` 方法，它则是通过分别给三角形的 3 个顶点打分从而得到三角形的评分。

在说 `VisitTriangle()` 方法之前，先要了解一下 `FContext` 类，它记录了整个 stripify 过程中的临时数据：

```cpp
class FContext
{
public:
    bool TriangleEnabled( uint32 TriangleIndex ) const
    {
        return ( TrianglesEnabled[ TriangleIndex >> 5 ] & ( 1u << ( TriangleIndex & 31u ) ) ) != 0u;
    }

    uint16 OldToNewVertex[NANITE_MAX_CLUSTER_TRIANGLES * 3 ];   // 旧顶点索引到新顶点索引的映射
    uint16 NewToOldVertex[NANITE_MAX_CLUSTER_TRIANGLES * 3 ];   // 新顶点索引到旧顶点索引的映射

    // 当前材质段内的候选三角形 mask
    uint32 TrianglesEnabled[ MAX_CLUSTER_TRIANGLES_IN_DWORDS ]; // Enabled triangles are in the current material range and have not yet been visited.
    // 被访问顶点关联的三角形 mask
    uint32 TrianglesTouched[ MAX_CLUSTER_TRIANGLES_IN_DWORDS ]; // Touched triangles have had at least one of their vertices visited.

    // 记录每个三角形输出后的 strip 编码信息
    // 当三角形作为 strip 起点输出时, 3 列分别是: Reset = 1; 三角形 ref 顶点数高 bit 位; 三角形 ref 顶点数低 bit 位(ref 顶点数 0..3 存储在 2 bits 中, 这里分别存储高 bit 和低 bit)
    // 当三角形作为 strip 延伸输出时, 3 列分别时: Reset = 0; 是否是从左边延伸过来的; 第 3 个顶点是否是 ref 顶点
    uint32 StripBitmasks[ 4 ][ 3 ]; // [4][Reset, IsLeft, IsRef]

    uint32 NumTriangles;    // 已输出的新三角形数
    uint32 NumVertices;     // 已输出的新顶点序列中的新顶点数
};
```

`VisitTriangle()` 方法的核心逻辑是：**读取输入三角形 Corner 所表示三角形的 3 个顶点，根据 5-bits 偏移约束决定每个顶点应该输出成一个 ref 顶点还是一个新顶点，最后将此三角形的 strip 编码信息写入 `StripBitmasks`**。

在这里单独说明一下什么是 5-bits 偏移约束：//todo: 理解一下再说

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

第二步，对当前三角形所有可能输出的 ref 顶点执行 **5-bits 偏移约束**检查，通过约束检查的顶点输出为 ref 顶点，否则输出为新顶点：

```cpp
// 3 个旧顶点在新顶点序列中的索引
uint16& NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
uint16& NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
uint16& NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];

uint32 OrgIndex0 = NewIndex0;
uint32 OrgIndex1 = NewIndex1;
uint32 OrgIndex2 = NewIndex2;

// 根据 3 个旧顶点是否已经输出到新顶点序列中预估输出当前三角形的 3 个旧顶点后新顶点序列中的最新顶点索引
uint32 NextVertexIndex = Context.NumVertices + ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );

while(true)
{
    // 如果旧顶点 OldIndex0 已经存在于新顶点序列中, 也就意味着当前这个旧顶点可以当作 ref 顶点输出. 但是它所对应的新顶点距离当前新顶点序列中的最新顶点太远, 导致无法用 5-bits 去记录这个偏移
    if( NewIndex0 != INVALID_INDEX && NextVertexIndex - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE )
    {
        // 此时把旧顶点当作一个新顶点输出
        NewIndex0 = INVALID_INDEX;
        // 新顶点序列中新增了一个顶点, 所以最新顶点索引 + 1
        NextVertexIndex++;
        continue;
    }
    // 旧顶点 OldIndex1 和旧顶点 OldIndex2 同理
    if( NewIndex1 != INVALID_INDEX && NextVertexIndex - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex1 = INVALID_INDEX; NextVertexIndex++; continue; }
    if( NewIndex2 != INVALID_INDEX && NextVertexIndex - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex2 = INVALID_INDEX; NextVertexIndex++; continue; }
    break;
}
```

在这里先通过 `Context.OldToNewVertex` 查询当前三角形的 3 个旧顶点是否已经输出到新顶点序列中，已经进入新顶点序列的旧顶点先视为 ref 顶点输出候选，否则直接视为新顶点输出。

然后对 ref 顶点输出候选执行 5-bits 偏移约束检查：分别检查每个 ref 顶点引用的新顶点与实际新顶点序列中的最新顶点间的相对距离，如果这个相对距离 `>= 32`，也就说明这个相对距离无法用 5-bits 去记录，则表示这个旧顶点应该被当作新顶点输出。**需要注意的是**：这里所说的实际新顶点序列中的最新顶点还额外考虑到了当前三角形输出后的新顶点序列的情况，即：当前三角形的 3 个旧顶点也有可能输出为新顶点。

第三步，将当前三角形的 strip 编码信息写入 `StripBitmasks`：

```cpp
// 当前三角形输出后的新索引
uint32 NewTriangleIndex = Context.NumTriangles;
// 当前三角形输出后会新增多少个新顶点
uint32 NumNewVertices = ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );

// 当前三角形是作为 strip 起点输出
if( bStart )
{
    // ( NewIndex == INVALID_INDEX ) 表示旧顶点是作为 ref 顶点输出; ( NewIndex != INVALID_INDEX ) 表示旧顶点作为新顶点输出
    // 这里的 check 表示: 当一个三角形作为 strip 起点输出时, 它的 3 个旧顶点输出需要满足 isNew2 >= isNew1 >= isNew0, 也就是说: 当一个三角形作为 strip 起点输出, 那么它的 3 个旧顶点必须满足: 前面的顶点
    check( ( NewIndex2 == INVALID_INDEX ) >= ( NewIndex1 == INVALID_INDEX ) );
    check( ( NewIndex1 == INVALID_INDEX ) >= ( NewIndex0 == INVALID_INDEX ) );

    // 当前三角形输出的 ref 顶点数
    uint32 NumWrittenIndices = 3u - NumNewVertices;

    // 把 ref 顶点数拆成 2 bits 存储: 0 -> 00, 1 -> 01, 2 -> 10, 3 -> 11
    uint32 LowBit = NumWrittenIndices & 1u;
    uint32 HighBit = (NumWrittenIndices >> 1) & 1u;

    // 写入当前三角形的 strip 编码信息:
    // 对于 strip 起点三角形来说, bitmask 0 置为 1, 表示它是一个 strip 起点
    Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 0 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
    // 对于 strip 起点三角形来说, bitmask 1 表示其输出的 ref 顶点数的高 bit
    Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 1 ] |= ( HighBit << ( NewTriangleIndex & 31u ) );
    // 对于 strip 起点三角形来说, bitmask 2 表示其输出的 ref 顶点数的低 bit
    Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 2 ] |= ( LowBit << ( NewTriangleIndex & 31u ) );
}
else    // 当前三角形作为 strip 延伸输出
{
    // 这里的 check 表示: 当一个三角形作为 strip 延伸输出时, 它的 3 个旧顶点输出需要满足: 前 2 个旧顶点需要作为 ref 顶点输出
    check( NewIndex0 != INVALID_INDEX );
    check( NewIndex1 != INVALID_INDEX );

    // 写入当前三角形的 strip 编码信息:
    // 对于 strip 延伸三角形来说, bitmask 0 置为 0, 这里直接不需要处理
    // 对于 strip 延伸三角形来说, bitmask 1 表示其是否从左侧延伸过来的, !bRight 也就是 IsLeft
    if( !bRight )
    {
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 1 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
    }

    // 对于 strip 延伸三角形来说, 其最后一个旧顶点通常是作为新顶点输出, 这里的 bitmask 2 表示最后一个顶点是否是作为 ref 顶点输出
    if(NewIndex2 != INVALID_INDEX)
    {
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 2 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
    }
}
```

这里详细说说 `StripBitmasks` 数据，它记录了每个三角形输出后的 strip 编码信息

这里详细说说 `StripBitmasks` 数据，它是一个 `4x3` 二维 `uint32` 数组，其中每 1 列的 `uint32` 中的每 1 bit 记录了其对应三角形输出后的 1 种 strip 编码信息，一共 3 列总共记录了 `[Reset, IsLeft, IsRef]` 这 3 种 strip 编码信息：

当一个三角形作为 strip 起点输出时，此三角形所对应的第 1 列 `uint32` 中的 bit 位是 1，表示它是一个 strip 起点；然后将当前三角形输出的 ref 顶点数拆成 2 bits 存储，高 bit 存储在此三角形所对应的第 2 列 `uint32` 中的 bit 位中，高 bit 存储在此三角形所对应的第 3 列 `uint32` 中的 bit 位中；

而当一个三角形作为 strip 延伸输出时，此三角形所对应的第 1 列 `uint32` 中的 bit 位是 0，表示它是一个 strip 延伸；此三角形所对应的第 2 列 `uint32` 中的 bit 位记录延伸方向（左侧还是右侧）；另外对于输出为 strip 延伸的三角形，它的前 2 个顶点必须作为 ref 顶点输出，所以此三角形所对应的第 3 列 `uint32` 中的 bit 位记录其第 3 个顶点是否是作为 ref 顶点输出；

将当前三角形的 strip 编码信息写入 `StripBitmasks` 之后，Nanite 会为即将作为新顶点输出的旧顶点分配其在新顶点序列中的新索引，然后记录旧顶点索引到新顶点索引的映射，接着将当前三角形标记为已输出并且将其从候选集合中移除，最后返回当前三角形输出的新顶点数：

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

`BuildTables()` 方法主要构建了 2 张 stripify 所需的查找表：`OppositeCorner` 中通过三角形 Corner 记录邻接三角形之间的共享边信息，三角形 Corner 中不仅记录了旧三角形索引，还记录了旧三角形局部的顶点索引，根据三角形 Corner 中的局部顶点索引，Nanite 不仅可以找邻接三角形，还可以找特定左侧或者右侧的邻居三角形；`TrianglePriorities` 中则保存每个旧三角形 3 个旧顶点位置和的 x 分量值，后续此表决定评分相同的三角形谁优先作为 strip 起点输出；

而 `NewScoreVertex()` 方法主要是根据一个旧顶点作为 ref 顶点输出的代价以及其所在旧三角形的拓扑延伸能力为其打分，根据此评分决定 strip 起点和 strip 延伸三角形的选择；

`VisitTriangle()` 是整个 stripify 过程中的核心方法，它的主要逻辑是将一个旧三角形输出到 stripify 之后的三角形序列中，并决定这个三角形相关的旧顶点在 stripify 之后应当编码为一个新顶点还是一个 ref 顶点。

需要注意的是，对于 strip 的起点三角形，最多需要处理其 3 个顶点的编码，而对于 strip 的延伸三角形，因为其前 2 个顶点通常由 strip 拓扑隐含复用，所以只需要处理其新引入的那个顶点的编码。

具体来说，对于需要编码的旧顶点，首先检查它是否已经对应到新顶点序列中的某个已输出顶点，如果还不存在于新顶点序列中，则将其作为一个新顶点编码；而如果它已经存在于新顶点序列中，则优先考虑将其作为 ref 顶点编码：通过记录其对应的已输出新顶点与当前新顶点序列末尾之间的相对距离，来引用这个已输出的新顶点。又因为 Nanite 使用 5 bits 去存储这个相对距离，所以如果这个相对距离超过了 5 bits 可以表示的范围，那么仍然会将这个旧顶点重新作为一个新顶点编码。

//todo: Nanite 是按材质段（`MaterialRange`）分段进行 stripify 的，这样可以保持材质分组不被打乱，避免 strip 跨材质。首先会初始化相关数据：

<!-- 它会根据 cluster 的原始网格数据构建 stripify 所需的几张查找表数据 -->

<!-- 在说 `BuildTables()` 方法之前，先要说说 `BuildTables()` 方法中的 `FEdgeNode.Corner`： -->

<!-- `FEdgeNode.Corner` 数据类型是一个 `uint16`，其中高 14 位编码了三角形的 index，低 2 位编码了这个三角形某个顶点的局部索引（也就是内部顶点索引 0..2 ）, -->

## 2. 编码 Cluster DAG

## 2. References

- [Nanite: A Deep Dive](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)
- [GAMES 104: GPU-Driven Geometry Pipeline - Nanite](https://www.piccoloengine.com/merch/8)
