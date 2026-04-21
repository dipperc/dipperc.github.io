---
layout: post
title:  "Unreal Engine - Nanite"
date:   2026-04-15 20:18:00 +800
category: Unreal Engine
---

- [1. 离线构建：Nanite Pages](#1-离线构建nanite-pages)
  - [1.1. 构建 Leaf Clusters](#11-构建-leaf-clusters)
    - [1.1.1. 为每条有向 Triangle Edge 建 Hash Key](#111-为每条有向-triangle-edge-建-hash-key)
    - [1.1.2. 查找每条有向 Triangle Edge 的反向共享边](#112-查找每条有向-triangle-edge-的反向共享边)
    - [1.1.3. 构建 Triangle 的连通岛](#113-构建-triangle-的连通岛)
    - [1.1.4. METIS 递归二分所有 Triangle](#114-metis-递归二分所有-triangle)
    - [1.1.5. 根据分区结果创建 Leaf Clusters](#115-根据分区结果创建-leaf-clusters)
    - [1.1.6. 每个 Cluster 中有哪些数据](#116-每个-cluster-中有哪些数据)

## 1. 离线构建：Nanite Pages

### 1.1. 构建 Leaf Clusters

#### 1.1.1. 为每条有向 Triangle Edge 建 Hash Key

在原始的 mesh 中，顶点索引数据 `Indexes` 中记录了所有三角形的 vertex index，每 3 个 vertex index 是一个三角形，Nanite 按 vertex index 取每条有向边，`EdgeIndex` 实际就是 `Indexes` 中的每个 vertex index。

第一步要做的，就是**并行遍历每条有向边并根据每条边的 2 个端点位置计算它的 hash**。

**需要注意的是**：Nanite 中是根据有向边的 2 个端点的**位置**来计算其 hash，而不是根据 vertex index 来计算。这是因为 mesh 可能由于 UV seam/normal seam/hard edge 等原因把同一个位置拆成多个顶点，通过位置来计算 hash 可以保证只要位置能匹配，就能在后续处理的过程中被识别为几何相邻。

同一个位置被拆成多个顶点的一个很典型的例子：一个立方体只有 8 个角点位置，但渲染时常常不是 8 个顶点，而是 24 个顶点，这是因为每个角会被 3 个面使用，而每个面都要有自己的法线、UV。

`Cycle3` 函数在三角形 3 个顶点内循环 + 1，对于任意传入的 `EdgeIndex`（有向边 `EdgeIndex` 的起点），次函数返回此有向边的终点：

```c++
FORCEINLINE uint32 Cycle3( uint32 Value )
{
    uint32 ValueMod3 = Value % 3;
    uint32 Value1Mod3 = ( 1 << ValueMod3 ) & 3;
    return Value - ValueMod3 + Value1Mod3;
}
```

对于有向边 EdgeIndex，分别取其 2 个端点的位置 `Position0` 和 `Position1`，以 `(Position0, Position1)` 顺序计算这条有向边的 hash，最后将结果添加到 `HashTable` 中，其中 Key 是有向边 EdgeIndex 的 hash，Value 是有向边的索引 EdgeIndex。

```c++
template< typename FGetPosition >
void Add_Concurrent( int32 EdgeIndex, FGetPosition&& GetPosition )
{
    // 对于有向边 EdgeIndex:
    // 起点位置
    const FVector3f& Position0 = GetPosition( EdgeIndex );
    // 终点位置
    const FVector3f& Position1 = GetPosition( Cycle3( EdgeIndex ) );

    // 以 { HashPosition( Position0 ), HashPosition( Position1 ) } 的顺序计算 hash
    uint32 Hash0 = HashPosition( Position0 );
    uint32 Hash1 = HashPosition( Position1 );
    uint32 Hash = Murmur32( { Hash0, Hash1 } );

    // 添加到 HashTable 中: key=hash, value=EdgeIndex
    HashTable.Add_Concurrent( Hash, EdgeIndex );
}
```

#### 1.1.2. 查找每条有向 Triangle Edge 的反向共享边

#### 1.1.3. 构建 Triangle 的连通岛

#### 1.1.4. METIS 递归二分所有 Triangle

#### 1.1.5. 根据分区结果创建 Leaf Clusters

#### 1.1.6. 每个 Cluster 中有哪些数据

<!-- ### 1.2. 构建 Cluster DAG

### 1.3. 构建 Hierarchy Nodes

### 1.4. 压缩

## 2. 运行时：渲染与 Streaming

### 2.1. 裁剪

### 2.2. 运行时 LoD 选择

### 2.3. Streaming -->
