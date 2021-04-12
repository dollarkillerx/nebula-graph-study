# nebula-graph-study
nebula-graph-study

### 数据结构
- `Vertex` 点
    - 保存实体对象的实例
    - `VID` 为唯一表示
    - 点必须至少有一个 `Tag`
- `Tag` 标签
    - 用于对点进行区分。相同标签共享相同属性类型
- `Edge` 边
    - 描述 点与点之间的关系行为
    - 一条边有且仅有一条边类型
    - 起点 -> 边类型(`edge type`) -> `rank` -> 终点 可以唯一表示一条边
    - 边是有指向的 `->` 表示边的指向, 边可以沿任意方向遍历
    - 边必须有`rank`, `rank` 是一个不可更改,用户分配的, 64位符号整数。通过它才能标识两点之间具有相同边类型的边。边按它们的`rank`的排序,值最大的排在前面, default `rank` = 0 
- `Edge Type` 边类型
    - 用于对边进行区分。具有相同边类型的边共享相同的属性定义
- `properties` 属性
    - 属性的指向值对`key-value` 类型存储点或边的相关信息

### 有向属性图 `G = < V, E, PV, PE >`
指点和边构成的图，这些边是有方向的。

- V是点的集合。
- E是有向边的集合。
- PV 是点的属性。
- PE 是边的属性。

![](https://user-images.githubusercontent.com/42762957/64932536-51b1f800-d872-11e9-9016-c2634b1eeed6.png)
上述信息:
- 点
    - `team`
        * `id`
        * `name`
        * `age`
    - `player`
        * `id`
        * `name`
- 边
    - `serve`
        * `start_year`
        * `end_year`
    - `like`
        * `likeness`

### 服务组成
- Graph服务
    - 处理计算请求
- Meta服务 
    - 管理用户帐户 
    - 存储贺管理分片的位置信息, 并且保证分片的负载均衡
    - 管理图空间
    - 管理Schema
    - 管理TTL 贺数据回收
    - 管理作业，创建-排队-查询-删除
- Storage服务 `raft`
    - 数据存储

![](https://docs-cdn.nebula-graph.com.cn/docs-2.0/1.introduction/2.nebula-graph-architecture/nebula-graph-architecture-1.png)

### 基础语法
#### 链接
``` 
docker run --rm -ti --network nebuladockercompose_nebula-net --entrypoint=/bin/sh vesoft/nebula-console:v2-nightly

nebula-console -u user -p password --address=graphd --port=9669
```
#### 图空间和Schema

![](https://docs-cdn.nebula-graph.com.cn/docs-2.0/2.quick-start/demo-dataset-for-quick-start.png)

#### 异步实现创建和修改
Nebula Graph 中执行如下创建和修改操作, 为异步实现, 需要下一个心跳周期才同步数据
- `CREATE SPACE`
- `CREATE TAG`
- `CREATE EDGE`
- `ALTER TAG`
- `ALTER EDGE`
- `CREATE TAG INDEX`
- `CREATE EDGE INDEX`

> default 心跳10s。 修改参数 `heartbeat_interval_secs`


