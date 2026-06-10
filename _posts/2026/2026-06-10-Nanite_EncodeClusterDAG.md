---
layout: post
title:  "Nanite - Encode Cluster DAG"
date:   2026-06-10 20:18:00 +800
category: Unreal Engine
---

- [1. 编码 Cluster DAG](#1-编码-cluster-dag)
- [4. References](#4-references)

本篇笔记是 Unreal Engine 的 Nanite 系统中关于编码 Cluster DAG 的源码分析理解，基于引擎版本 5.7.3 release

## 1. 编码 Cluster DAG

到这里 Nanite 已经构建好 cluster DAG，而这一步要做的就是把 cluster DAG 编码成运行时可用的 `Nanite::FResources` 数据。

既然要对 cluster DAG 进行编码，那么就要先规范化输入的 cluster 数据，规范化不仅仅包括清理非法值，还需要对 cluster 相关数据进行整理，通过这套规划化的处理，使 cluster 数据变成后续编码所需的标准格式：

1. Nanite 首先会清理非法的 vertex 属性和删除退化三角形：

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

2. 然后会把 cluster 内的三角形按材质重新排序，并生成 cluster 内连续三角形区间对应的材质 index 数据 `MaterialRanges`：**每个 `MaterialRange` 记录了 cluster 内从第 `RangeStart` 个三角形开始，连续 `RangeLength` 个三角形，都使用的是材质 `MaterialIndex`**。核心逻辑在 `FCluster::BuildMaterialRanges()` 方法中：

    首先会遍历 cluster 中的每个三角形，统计每个材质 index 对应的三角形数量：

    ```cpp
    TArray< int32, TInlineAllocator<128> > MaterialElements;
    TArray< int32, TInlineAllocator<64> > MaterialCounts;

    MaterialElements.AddUninitialized( MaterialIndexes.Num() );
    MaterialCounts.AddZeroed( NANITE_MAX_CLUSTER_MATERIALS );

    // Tally up number per material index
    for( int32 i = 0; i < MaterialIndexes.Num(); i++ )
    {
        // MaterialElements[i] 表示 cluster 内第 i 个三角形
        MaterialElements[i] = i;
        // 当前材质 index 对应的三角形数量 + 1
        MaterialCounts[ MaterialIndexes[i] ]++;
    }
    ```

    然后排序 cluster 内的三角形 index，排序遵循的规则是：

    1. 使用三角形数量更多的材质排前面；
    2. 如果数量相同，则材质 index 小的排前面
    3. 材质 index 也相同，则三角形 index 小的排前面

    ```cpp
    // Sort by range count descending, and material index ascending.
    // This groups the material ranges from largest to smallest, which is
    // more efficient for evaluating the sequences on the GPU, and also makes
    // the minus one encoding work (the first range must have more than 1 tri).
    MaterialElements.Sort(
        [&]( int32 A, int32 B )
        {
            // 材质 index
            int32 IndexA = MaterialIndexes[A];
            int32 IndexB = MaterialIndexes[B];

            // 材质数量
            int32 CountA = MaterialCounts[ IndexA ];
            int32 CountB = MaterialCounts[ IndexB ];

            // 优先根据材质数量多的排前面
            if( CountA != CountB )
                return CountA > CountB;

            // 其次是材质 index 小的排前面
            if( IndexA != IndexB )
                return IndexA < IndexB;

            // 最后是 cluster 内三角形 index 小的排前面
            return A < B;
        } );
    ```

    排序后 `MaterialElements` 中的三角形 index 顺序满足：**使用相同材质的三角形会被排在一起，从而形成连续的区间，并且材质区间越大，它排的越靠前**。

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

        // Material changed, so add current range and reset
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

    最后才是真正重新排序 cluster 的顶点索引 `Indexes` 和材质索引 `MaterialIndexes`：

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

3. 在 Nanite 编码前，还会对 cluster 做编码约束调整：在每个 `MaterialRange` 内对三角形进行重排，并按新的访问顺序重排顶点；当某个旧顶点超出 32 顶点窗口约束时，会复制该顶点以生成新的局部顶点索引；如果约束后 cluster 顶点数超过 256，还会将 cluster 按三角形范围拆分并重新约束。这样可以保证每个 cluster 满足 32 顶点窗口约束和 256 顶点上限，为后续 index 编码和 GPU transcoding 做准备。

    在开始看 `ConstrainAndStripify` 的逻辑之前，先来看看 `BuildTables`、`NewScoreVertex`、`NewScoreTriangle` 和 `VisitTriangle` 这几个方法分别是在做什么：

    ```cpp
    void BuildTables( const FCluster& Cluster )
    {
        // 链表节点
        struct FEdgeNode
        {
            uint16 Corner;  // (Triangle << 2) | LocalCorner
            uint16 NextNode;
        };
        // 每条三角形有向边会挂一个链表, 链表里存的是: 哪个三角形的哪条有向边对应这条边

        // NANITE_MAX_CLUSTER_INDICES = 128 * 3, 一个 cluster 最多 128 个三角形, 一个三角形有 3 条有向边
        FEdgeNode EdgeNodes[NANITE_MAX_CLUSTER_INDICES ];
        
        // EdgeNodeHeads[a * NANITE_MAX_CLUSTER_INDICES + b] 表示有向边 a -> b 的链表头, 这里用二维矩阵按顶点 index 直接寻址
        // 如果是非流形边上挂了超过 2 个三角形, 会把它们全部挂进链表, 后面查询时每次 pop 一个
        uint16 EdgeNodeHeads[NANITE_MAX_CLUSTER_INDICES * NANITE_MAX_CLUSTER_INDICES ]; // Linked list per edge to support more than 2 triangles per edge.
        FMemory::Memset( EdgeNodeHeads, INVALID_NODE_MEMSET );

        FMemory::Memset( VertexToTriangleMasks, 0 );

        uint32 NumTriangles = Cluster.NumTris;
        uint32 NumVertices = Cluster.Verts.Num();

        // Add triangles to edge lists and update valence
        for( uint32 i = 0; i < NumTriangles; i++ )
        {
            // 取第 i 个三角形的 3 个顶点 index
            uint32 i0 = Cluster.Indexes[ i * 3 + 0 ];
            uint32 i1 = Cluster.Indexes[ i * 3 + 1 ];
            uint32 i2 = Cluster.Indexes[ i * 3 + 2 ];

            // 保证不是退化三角形, 并且 index 没有越界
            check( i0 != i1 && i1 != i2 && i2 != i0 );
            check( i0 < NumVertices && i1 < NumVertices && i2 < NumVertices );

            // 4 个 DWORD 一共 128 bits 来标记 cluster 中的每个三角形
            // i >> 5 计算出三角形 i 属于第几个 DWORD
            // 1 << ( i & 31 ) 计算出三角形 i 在所属 DWORD 中的具体 bit 位, 并将此 bit 置为 1
            VertexToTriangleMasks[ i0 ][ i >> 5 ] |= 1 << ( i & 31 );
            VertexToTriangleMasks[ i1 ][ i >> 5 ] |= 1 << ( i & 31 );
            VertexToTriangleMasks[ i2 ][ i >> 5 ] |= 1 << ( i & 31 );

            // 计算三角形 3 个顶点位置和的 X 值, 后续选 strip 起点时, 如果 score 相同, 则用它决定谁优先
            FVector3f ScaledCenter = Cluster.Verts.GetPosition( i0 ) + Cluster.Verts.GetPosition( i1 ) + Cluster.Verts.GetPosition( i2 );
            TrianglePriorities[ i ] = ScaledCenter.X;   //TODO: Find a good direction to sort by instead of just picking x?

            // 根据 stripify 编码的顺序为每条有向边建立一个链表节点:

            // 将三角形 i 的第 0 条边节点放在 EdgeNodes[ i * 3 + 0 ]
            FEdgeNode& Node0 = EdgeNodes[ i * 3 + 0 ];
            // 将三角形索引 i 和 LocalCorner 0 编码到 Corner
            Node0.Corner = (uint16)SetCorner( i, 0 );
            // 当前的节点插入链表前, 先把原来的链表头保存到 NextNode
            Node0.NextNode = EdgeNodeHeads[ i1 * NANITE_MAX_CLUSTER_INDICES + i2 ];
            // 把当前节点设为有向边 i1 * NANITE_MAX_CLUSTER_INDICES + i2 的新头节点
            EdgeNodeHeads[ i1 * NANITE_MAX_CLUSTER_INDICES + i2 ] = uint16(i * 3 + 0);

            // 同样为三角形 i 的第 1 条边建链表节点
            FEdgeNode& Node1 = EdgeNodes[ i * 3 + 1 ];
            Node1.Corner = (uint16)SetCorner( i, 1 );
            Node1.NextNode = EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i0 ];
            EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i0 ] = uint16(i * 3 + 1);

            // 同样为三角形 i 的第 2 条边建链表节点
            FEdgeNode& Node2 = EdgeNodes[ i * 3 + 2 ];
            Node2.Corner = (uint16)SetCorner( i, 2 );
            Node2.NextNode = EdgeNodeHeads[ i0 * NANITE_MAX_CLUSTER_INDICES + i1 ];
            EdgeNodeHeads[ i0 * NANITE_MAX_CLUSTER_INDICES + i1 ] = uint16(i * 3 + 2);
        }

        // Gather adjacency from edge lists
        for( uint32 i = 0; i < NumTriangles; i++ )
        {
            // 取第 i 个三角形的 3 个顶点 index
            uint32 i0 = Cluster.Indexes[ i * 3 + 0 ];
            uint32 i1 = Cluster.Indexes[ i * 3 + 1 ];
            uint32 i2 = Cluster.Indexes[ i * 3 + 2 ];


            // 当前三角形 corner 0 的边是 i1 -> i2，如果另一个三角形也共享这条边，它通常会以反方向 i2 -> i1 存在, 所以从 EdgeNodeHeads[i2, i1] 中取一个节点
            uint16& Node0 = EdgeNodeHeads[ i2 * NANITE_MAX_CLUSTER_INDICES + i1 ];
            // 当前三角形 corner 1 的边是 i2 -> i0，如果另一个三角形也共享这条边，它通常会以反方向 i0 -> i2 存在, 所以从 EdgeNodeHeads[i0, i2] 中取一个节点
            uint16& Node1 = EdgeNodeHeads[ i0 * NANITE_MAX_CLUSTER_INDICES + i2 ];
            // 当前三角形 corner 2 的边是 i0 -> i1，如果另一个三角形也共享这条边，它通常会以反方向 i1 -> i0 存在, 所以从 EdgeNodeHeads[i1, i0] 中取一个节点
            uint16& Node2 = EdgeNodeHeads[ i1 * NANITE_MAX_CLUSTER_INDICES + i0 ];

            // 如果 EdgeNodeHeads[i2, i1] 中取出的节点不为空, 说明找到了一个共享这条边的三角形
            if( Node0 != INVALID_NODE )
            {
                // EdgeNodes[Node0].Corner 就是那个邻居三角形对应的 Corner, 并将其写入 OppositeCorner[i * 3 + 0]
                OppositeCorner[ i * 3 + 0 ] = EdgeNodes[ Node0 ].Corner;
                // 然后把链表头推进到下一个节点. 这样在非流形的情况下, 同一条边上有多个候选三角形时, 不会每次查询都拿到同一个邻接三角形
                Node0 = EdgeNodes[ Node0 ].NextNode;
            }
            else
            {
                // 如果没有反向边, 则说明没找到邻居三角形, 置为 INVALID_CORNER
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

    在说 `BuildTables` 之前，先要说说**什么是三角形 Corner**：

    三角形 Corner 的数据类型是 `uint16`，其中高 14 位编码了一个三角形的 index，低 2 位编码了这个三角形其中一个顶点的局部索引 0..2，也就是说每个三角形 Corner 记录的是某个三角形的某个顶点数据。在这里，一个三角形 Corner 代表的是**它所记录的三角形顶点对面的那条有向边**，例如：对于三角形 i 得第 0 个顶点，Corner(i, i0) 表示的是三角形 i 的有向边 `i1 -> i2`。

    `BuildTables` 是在 stripify 之前的关键一步，它根据 cluster 的原始网格信息构建 stripify 所需的几张表数据。

    首先是 `VertexToTriangleMasks`，它记录**每个旧顶点关联了哪些三角形**，通过 `VertexToTriangleMasks[ vertex index ][ triangle i >> 5 ]` 可以快速知道三角形 `i` 是否使用了顶点 `Verts[index]`。

    其次是 `OppositeCorner`，它记录**以 strip 编码的顺序下，每个三角形 Corner 对面共享同一条边的相邻三角形的 Corner**，后续 stripify 时会根据此表找左/右相邻三角形。

    最后是 `TrianglePriorities`，它保存了每个三角形 3 个顶点位置和的 X 值, 后续选 strip 起点时, 如果 score 相同, 则用它决定谁优先。

    ```cpp
    auto NewScoreVertex = [ &Weights ] ( const FContext& Context, uint32 OldVertex, bool bStart, bool bHasOpposite, bool bHasLeft, bool bHasRight )
    {
        // 当前这个旧顶点是否已经进入新的顶点序列; NewIndex=INVALID_INDEX 表示还没有, 或者因为超过 32 顶点窗口约束而失效
        uint16 NewIndex = Context.OldToNewVertex[ OldVertex ];

        // 如果这个顶点还没有进入新的顶点序列, 它是一个新的顶点, 直接评分 0
        int32 CacheScore = 0;
        if( NewIndex != INVALID_INDEX )
        {
            // 计算这个顶点距离当前最新顶点的距离
            uint32 CachePosition = ( Context.NumVertices - 1 ) - NewIndex;
            // 判断这个旧顶点是否在 Nanite 允许的 32 顶点窗口内
            if( CachePosition < NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE )
            {
                // 如果顶点还在 32 顶点窗口内, 就从权重表取分数, 根据一下 5 个维度查表:
                // bStart: 当前候选三角形是否作为新 strip 的起点
                // bHasOpposite: 当前 Corner 对面的邻接三角形是否还可用
                // bHasLeft: 左侧邻接三角形是否还可用
                // bHasRight: 右侧邻接三角形是否还可用
                // CachePosition: 顶点在约束窗口里的相对位置
                CacheScore = Weights.Weights[ bStart ][ bHasOpposite ][ bHasLeft ][ bHasRight ][ CachePosition ];
            }
        }

        return CacheScore;
    };

    auto NewScoreTriangle = [ &Cluster, &NewScoreVertex ] ( const FContext& Context, uint32 TriangleIndex, bool bStart, bool bHasOpposite, bool bHasLeft, bool bHasRight )
    {
        // 获取三角形的三个顶点 index
        const uint32 OldIndex0 = Cluster.Indexes[ TriangleIndex * 3 + 0 ];
        const uint32 OldIndex1 = Cluster.Indexes[ TriangleIndex * 3 + 1 ];
        const uint32 OldIndex2 = Cluster.Indexes[ TriangleIndex * 3 + 2 ];

        // 通过 NewScoreVertex 分别对 3 个顶点打分, 得到当前三角形的分数
        return  NewScoreVertex( Context, OldIndex0, bStart, bHasOpposite, bHasLeft, bHasRight ) +
                NewScoreVertex( Context, OldIndex1, bStart, bHasOpposite, bHasLeft, bHasRight ) +
                NewScoreVertex( Context, OldIndex2, bStart, bHasOpposite, bHasLeft, bHasRight );
    };
    ```

    `NewScoreVertex` 是一个给候选顶点打分的函数，它根据这个顶点是否已经进入新顶点序列（是否可以复用）、它在 32 顶点窗口内的相对位置（复用的代价大不大）、以及它所在三角形的相邻三角形信息（所在三角形的拓扑上下文）来给这个顶点打分，打分同时考虑了**顶点的复用价值**和**其所在三角形的拓扑延伸潜力**。

    而 `NewScoreTriangle` 则是通过分别对三角形的 3 个顶点打分从而得到三角形的分数。

    在说 `VisitTriangle` 之前，先要了解一下 `FContext`，它记录了整个 stripify 过程中的临时状态：

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

        // 当前材质段内的候选三角形
        uint32 TrianglesEnabled[ MAX_CLUSTER_TRIANGLES_IN_DWORDS ]; // Enabled triangles are in the current material range and have not yet been visited.
        // 被访问顶点关联到的三角形标记
        uint32 TrianglesTouched[ MAX_CLUSTER_TRIANGLES_IN_DWORDS ]; // Touched triangles have had at least one of their vertices visited.

        // 每 32 个三角形一组的 bitmask
        // 当三角形是 strip 起点时, 3 列分别是: Reset = 1; ref 顶点数高 bit; ref 顶点数低 bit (ref 顶点数 0..3 被存储在 2 bit中, 这里分别存储高 bit 和低 bit)
        // 当三角形是 strip 延伸时, 3 列分别时: Reset = 0; 是否是从左边延伸过来的 ; 第 3 个顶点是否是 ref 顶点
        uint32 StripBitmasks[ 4 ][ 3 ]; // [4][Reset, IsLeft, IsRef]

        uint32 NumTriangles;    // 已经 stripify 的三角形数量
        uint32 NumVertices;     // 已经 stripify 的新顶点数量
    };
    ```

    而在 `VisitTriangle` 方法中，首先会通过 `TriangleCorner` **按 strip 编码需要的局部顺序**获取三角形的 3 个旧顶点，并将这 3 个旧顶点关联的三角形标记为 touched：

    ```cpp
    // 按 i0 -> i1, i1 -> i2, i2 -> i0 的顺序取下一个顶点
    const uint32 OldIndex0 = Cluster.Indexes[ CornerToIndex( NextCorner( TriangleCorner ) ) ];
    // 按 i0 <- i1, i1 <- i2, i2 <- i0 的顺序取上一个顶点
    const uint32 OldIndex1 = Cluster.Indexes[ CornerToIndex( PrevCorner( TriangleCorner ) ) ];
    // 当前 TriangleCorner 对应的顶点
    const uint32 OldIndex2 = Cluster.Indexes[ CornerToIndex( TriangleCorner ) ];

    // Mark incident triangles
    for( uint32 i = 0; i < MAX_CLUSTER_TRIANGLES_IN_DWORDS; i++ )
    {
        // 标记这 3 个顶点关联的三角形为 touched
        Context.TrianglesTouched[ i ] |= VertexToTriangleMasks[ OldIndex0 ][ i ] | VertexToTriangleMasks[ OldIndex1 ][ i ] | VertexToTriangleMasks[ OldIndex2 ][ i ];
    }
    ```

    第二步，对当前三角形的所有可能 ref 顶点执行 32 顶点窗口约束：

    ```cpp
    // 这 3 个旧顶点在新的顶点序列中的 index
    uint16& NewIndex0 = Context.OldToNewVertex[ OldIndex0 ];
    uint16& NewIndex1 = Context.OldToNewVertex[ OldIndex1 ];
    uint16& NewIndex2 = Context.OldToNewVertex[ OldIndex2 ];

    uint32 OrgIndex0 = NewIndex0;
    uint32 OrgIndex1 = NewIndex1;
    uint32 OrgIndex2 = NewIndex2;

    // 预估 stripify 这个三角形之后, 新的顶点总数会是多少
    // 等于 INVALID_INDEX 意味着还不存在于新的顶点序列中, 所以新的顶点数会 + 1
    uint32 NextVertexIndex = Context.NumVertices + ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );
    
    while(true)
    {
        // 如果 OldIndex0 本来可以复用, 但是它离访问后的尾部太远了, 无法用 5-bit delta 引用顶点
        if( NewIndex0 != INVALID_INDEX && NextVertexIndex - NewIndex0 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE )
        {
            // 则将其置为 INVALID_INDEX
            NewIndex0 = INVALID_INDEX;
            // 此时需要新增一个新顶点, 所以 NextVertexIndex + 1
            NextVertexIndex++;
            continue;
        }
        // OldIndex1, OldIndex2 同理
        if( NewIndex1 != INVALID_INDEX && NextVertexIndex - NewIndex1 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex1 = INVALID_INDEX; NextVertexIndex++; continue; }
        if( NewIndex2 != INVALID_INDEX && NextVertexIndex - NewIndex2 >= NANITE_CONSTRAINED_CLUSTER_CACHE_SIZE ) { NewIndex2 = INVALID_INDEX; NextVertexIndex++; continue; }
        break;
    }
    ```

    先通过 `Context.OldToNewVertex` 查询当前三角形的 3 个旧顶点是否已经进入新顶点序列，已经进入新顶点序列的顶点先视为 ref 顶点候选，没有进入的则视为一个新的顶点。

    然后对所有 ref 顶点候选**执行 32 顶点窗口约束**：根据当前三角形实际 stripify 后会增加的顶点数，检查所有 ref 顶点候选是否仍然在 32 顶点窗口内，如果距离太远导致无法用 5-bit delta 引用，那么就把它修改为新顶点，在新顶点序列中重新拷贝一份，最终得到当前三角形的每个顶点是 ref 顶点还是新顶点。

    第三步，写当前三角形的 strip bitmasks：

    ```cpp
    // 当前要 stripify 的三角形的新 index
    uint32 NewTriangleIndex = Context.NumTriangles;
    // 这个新三角形会引入几个新顶点
    uint32 NumNewVertices = ( NewIndex0 == INVALID_INDEX ) + ( NewIndex1 == INVALID_INDEX ) + ( NewIndex2 == INVALID_INDEX );

    // 新三角形是 strip 的起点
    if( bStart )
    {
        // NewIndexX == INVALID_INDEX 时: 表示旧顶点 X 已经有新顶点索引, 是 ref 顶点
        // NewIndexX != INVALID_INDEX 时: 表示旧顶点 X 还没出现, 需要作为新顶点 stripify
        // 这里要求: 当一个三角形作为 strip 起点时, 它的三个顶点需要满足: isNew2 >= isNew1 >= isNew0, 也就是说 strip 起点的三角形的顶点必须满足: ref 顶点在前, 新顶点在后
        check( ( NewIndex2 == INVALID_INDEX ) >= ( NewIndex1 == INVALID_INDEX ) );
        check( ( NewIndex1 == INVALID_INDEX ) >= ( NewIndex0 == INVALID_INDEX ) );

        // ref 顶点数, 复用顶点需要写 5-bit delta
        uint32 NumWrittenIndices = 3u - NumNewVertices;
        
        // 把 ref 顶点数拆成 2 bits 存储: 0 -> 00, 1 -> 01, 2 -> 10, 3 -> 11
        uint32 LowBit = NumWrittenIndices & 1u;
        uint32 HighBit = (NumWrittenIndices >> 1) & 1u;

        // 写 bitmask
        // bitmask 0: 设置为 1, 表示这个三角形是 strip start, 也就是 Reset = 1
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 0 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
        // 对于 strip start 三角形来说, bitmask 1 表示的是 ref 数量的高 bit
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 1 ] |= ( HighBit << ( NewTriangleIndex & 31u ) );
        // // 对于 strip start 三角形来说, bitmask 1 表示的是 ref 数量的低 bit
        Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 2 ] |= ( LowBit << ( NewTriangleIndex & 31u ) );
    }
    else    // 新三角形是 strip 的延伸
    {
        // strip 延伸三角形必须复用前 2 个顶点
        // 这就是 triangle strip 的核心: 后续延伸三角形一般只需要新增 1 个顶点，另外两个来自前面的 strip
        check( NewIndex0 != INVALID_INDEX );
        check( NewIndex1 != INVALID_INDEX );
        
        // bitmask 1 中记录这个三角形是从左边还是右边延伸过来的
        if( !bRight )
        {
            // !bRight 也就是将 IsLeft 的 bit 置为 1
            Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 1 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
        }

        // 对于延伸三角形, 第 3 个顶点通常是新顶点, 如果 NewIndex2 != INVALID_INDEX 则说明第 3 个顶点不是新顶点, 而是引用已有顶点, 所以设置 IsRef bit. 调用方随后会写 5-bit delta
        if(NewIndex2 != INVALID_INDEX)
        {
            Context.StripBitmasks[ NewTriangleIndex >> 5 ][ 2 ] |= ( 1u << ( NewTriangleIndex & 31u ) );
        }
    }
    ```

    对于 strip 起点的三角形：strip bitmask 0 记录 Reset 标记为 1；然后将当前三角形中的 ref 顶点数 0..3 拆成 2 bits 存储，将高 bit 存储在 strip bitmask 1 中，低 bit 存储在 strip bitmask 2 中。

    而对于 strip 延伸的三角形：strip bitmask 0 记录 Reset 标记为 0；strip bitmask 1 标记延伸方向；对于延伸三角形，它的前 2 个顶点肯定是 ref 顶点，所以 strip bitmask 2 标记第 3 个顶点是否是 ref 顶点。

    写完当前三角形的 strip bitmasks 后，Nanite 会给新顶点分配新索引，并记录旧顶点到新顶点的索引映射，并标记当前三角形已 stripify 且将其从可选集合中移除，最后输出新增的顶点数：

    ```cpp
    if( NewIndex0 == INVALID_INDEX )
    {
        // 给 OldIndex0 分配新顶点索引, 并将 stripify 的新顶点数量 + 1
        NewIndex0 = uint16(Context.NumVertices++);
        // 并记录新顶点索引对应旧顶点 OldIndex0 索引的映射
        Context.NewToOldVertex[ NewIndex0 ] = uint16(OldIndex0);
    }
    // OldIndex1, OldIndex2 同理
    if( NewIndex1 == INVALID_INDEX ) { NewIndex1 = uint16(Context.NumVertices++); Context.NewToOldVertex[ NewIndex1 ] = uint16(OldIndex1); }
    if( NewIndex2 == INVALID_INDEX ) { NewIndex2 = uint16(Context.NumVertices++); Context.NewToOldVertex[ NewIndex2 ] = uint16(OldIndex2); }

    // Output triangle
    // stripify 的三角形数量 + 1
    Context.NumTriangles++;

    // Disable selected triangle
    // 从 Corner 中取回旧三角形 index
    const uint32 OldTriangleIndex = CornerToTriangle( TriangleCorner );
    // 把这个旧三角形从 TrianglesEnabled 集合里清掉, 也就是说它已经被访问, 不能再被后面的 strip 重复使用
    Context.TrianglesEnabled[ OldTriangleIndex >> 5 ] &= ~( 1u << ( OldTriangleIndex & 31u ) );

    // 返回这个三角形实际新增了几个新顶点
    return NumNewVertices;
    ```

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

    最后，则是 Nanite 通过 stripify 信息重建 cluster 的顶点索引 `Indexes`，通过下面的逻辑也会对 stripify 后的数据结构有一个更深的理解：

    ```cpp
    static void UnpackTriangleIndices( const FStripDesc& StripDesc, const uint8* StripIndexData, uint32 TriIndex, uint32* OutIndices )
    {
        const uint32 DwordIndex = TriIndex >> 5;
        const uint32 BitIndex = TriIndex & 31u;

        //Bitmask.x: bIsStart, Bitmask.y: bIsRight, Bitmask.z: bIsNewVertex
        const uint32 SMask = StripDesc.Bitmasks[ DwordIndex ][ 0 ];
        const uint32 LMask = StripDesc.Bitmasks[ DwordIndex ][ 1 ];
        const uint32 WMask = StripDesc.Bitmasks[ DwordIndex ][ 2 ];
        const uint32 SLMask = SMask & LMask;
        
        //const uint HeadRefVertexMask = ( SMask & LMask & WMask ) | ( ~SMask & WMask );
        const uint32 HeadRefVertexMask = ( SLMask | ~SMask ) & WMask;   // 1 if head of triangle is ref. S case with 3 refs or L/R case with 1 ref.

        const uint32 PrevBitsMask = ( 1u << BitIndex ) - 1u;
        const uint32 NumPrevRefVerticesBeforeDword = DwordIndex ? BitFieldExtractU32(StripDesc.NumPrevRefVerticesBeforeDwords, 10u, DwordIndex * 10u - 10u) : 0u;
        const uint32 NumPrevNewVerticesBeforeDword = DwordIndex ? BitFieldExtractU32(StripDesc.NumPrevNewVerticesBeforeDwords, 10u, DwordIndex * 10u - 10u) : 0u;

        int32 CurrentDwordNumPrevRefVertices = ( FMath::CountBits( SLMask & PrevBitsMask ) << 1 ) + FMath::CountBits( WMask & PrevBitsMask );
        int32 CurrentDwordNumPrevNewVertices = ( FMath::CountBits( SMask & PrevBitsMask ) << 1 ) + BitIndex - CurrentDwordNumPrevRefVertices;

        int32 NumPrevRefVertices    = NumPrevRefVerticesBeforeDword + CurrentDwordNumPrevRefVertices;
        int32 NumPrevNewVertices    = NumPrevNewVerticesBeforeDword + CurrentDwordNumPrevNewVertices;

        const int32 IsStart = BitFieldExtractI32( SMask, 1, BitIndex);      // -1: true, 0: false
        const int32 IsLeft  = BitFieldExtractI32( LMask, 1, BitIndex );     // -1: true, 0: false
        const int32 IsRef   = BitFieldExtractI32( WMask, 1, BitIndex );     // -1: true, 0: false

        const uint32 BaseVertex = NumPrevNewVertices - 1u;

        uint32 IndexData = ReadUnalignedDword( StripIndexData, ( NumPrevRefVertices + ~IsStart ) * 5 ); // -1 if not Start

        if( IsStart )
        {
            const int32 MinusNumRefVertices = ( IsLeft << 1 ) + IsRef;
            uint32 NextVertex = NumPrevNewVertices;

            if( MinusNumRefVertices <= -1 ) { OutIndices[ 0 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 0 ] = NextVertex++; }
            if( MinusNumRefVertices <= -2 ) { OutIndices[ 1 ] = BaseVertex - ( IndexData & 31u ); IndexData >>= 5; } else { OutIndices[ 1 ] = NextVertex++; }
            if( MinusNumRefVertices <= -3 ) { OutIndices[ 2 ] = BaseVertex - ( IndexData & 31u );                  } else { OutIndices[ 2 ] = NextVertex++; }
        }
        else
        {
            // Handle two first vertices
            const uint32 PrevBitIndex = BitIndex - 1u;
            const int32 IsPrevStart = BitFieldExtractI32( SMask, 1, PrevBitIndex);
            const int32 IsPrevHeadRef = BitFieldExtractI32( HeadRefVertexMask, 1, PrevBitIndex );
            //const int NumPrevNewVerticesInTriangle = IsPrevStart ? ( 3u - ( bfe_u32( /*SLMask*/ LMask, PrevBitIndex, 1 ) << 1 ) - bfe_u32( /*SMask &*/ WMask, PrevBitIndex, 1 ) ) : /*1u - IsPrevRefVertex*/ 0u;
            const int32 NumPrevNewVerticesInTriangle = IsPrevStart & ( 3u - ( (BitFieldExtractU32( /*SLMask*/ LMask, 1, PrevBitIndex) << 1 ) | BitFieldExtractU32( /*SMask &*/ WMask, 1, PrevBitIndex) ) );
            
            //OutIndices[ 1 ] = IsPrevRefVertex ? ( BaseVertex - ( IndexData & 31u ) + NumPrevNewVerticesInTriangle ) : BaseVertex; // BaseVertex = ( NumPrevNewVertices - 1 );
            OutIndices[ 1 ] = BaseVertex + ( IsPrevHeadRef & ( NumPrevNewVerticesInTriangle - ( IndexData & 31u ) ) );
            //OutIndices[ 2 ] = IsRefVertex ? ( BaseVertex - bfe_u32( IndexData, 5, 5 ) ) : NumPrevNewVertices;
            OutIndices[ 2 ] = NumPrevNewVertices + ( IsRef & ( -1 - BitFieldExtractU32( IndexData, 5, 5 ) ) );

            // We have to search for the third vertex. 
            // Left triangles search for previous Right/Start. Right triangles search for previous Left/Start.
            const uint32 SearchMask = SMask | ( LMask ^ IsLeft );               // SMask | ( IsRight ? LMask : RMask );
            const uint32 FoundBitIndex = FMath::FloorLog2( SearchMask & PrevBitsMask );
            const int32 IsFoundCaseS = BitFieldExtractI32( SMask, 1, FoundBitIndex );       // -1: true, 0: false

            const uint32 FoundPrevBitsMask = ( 1u << FoundBitIndex ) - 1u;
            int32 FoundCurrentDwordNumPrevRefVertices = ( FMath::CountBits( SLMask & FoundPrevBitsMask ) << 1 ) + FMath::CountBits( WMask & FoundPrevBitsMask );
            int32 FoundCurrentDwordNumPrevNewVertices = ( FMath::CountBits( SMask & FoundPrevBitsMask ) << 1 ) + FoundBitIndex - FoundCurrentDwordNumPrevRefVertices;

            int32 FoundNumPrevNewVertices = NumPrevNewVerticesBeforeDword + FoundCurrentDwordNumPrevNewVertices;
            int32 FoundNumPrevRefVertices = NumPrevRefVerticesBeforeDword + FoundCurrentDwordNumPrevRefVertices;

            const uint32 FoundNumRefVertices = (BitFieldExtractU32( LMask, 1, FoundBitIndex ) << 1 ) + BitFieldExtractU32( WMask, 1, FoundBitIndex );
            const uint32 IsBeforeFoundRefVertex = BitFieldExtractU32( HeadRefVertexMask, 1, FoundBitIndex - 1 );

            // ReadOffset: Where is the vertex relative to triangle we searched for?
            const int32 ReadOffset = IsFoundCaseS ? IsLeft : 1;
            const uint32 FoundIndexData = ReadUnalignedDword( StripIndexData, ( FoundNumPrevRefVertices - ReadOffset ) * 5 );
            const uint32 FoundIndex = ( FoundNumPrevNewVertices - 1u ) - BitFieldExtractU32( FoundIndexData, 5, 0 );

            bool bCondition = IsFoundCaseS ? ( (int32)FoundNumRefVertices >= 1 - IsLeft ) : (IsBeforeFoundRefVertex != 0u);
            int32 FoundNewVertex = FoundNumPrevNewVertices + ( IsFoundCaseS ? ( IsLeft & ( FoundNumRefVertices == 0 ) ) : -1 );
            OutIndices[ 0 ] = bCondition ? FoundIndex : FoundNewVertex;

            if( IsLeft )
            {
                // swap
                std::swap( OutIndices[ 1 ], OutIndices[ 2 ] );
            }
            check(OutIndices[0] != OutIndices[1] && OutIndices[0] != OutIndices[2] && OutIndices[1] != OutIndices[2]);
        }
    }

    const uint32 PaddedSize = Cluster.StripIndexData.Num() + 5;
    TArray<uint8> PaddedStripIndexData;
    PaddedStripIndexData.Reserve( PaddedSize );

    PaddedStripIndexData.Add( 0 );  // TODO: Workaround for empty list and reading from negative offset
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

## 4. References

- [Nanite: A Deep Dive](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)
- [GAMES 104: GPU-Driven Geometry Pipeline - Nanite](https://www.piccoloengine.com/merch/8)
