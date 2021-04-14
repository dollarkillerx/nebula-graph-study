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

### nGQL语法

- 创建图空间

``` 
CREATE SPACE [IF NOT EXISTS] <graph_space_name>
    [(partition_num = <partition_number>, 
    replica_factor = <replica_number>, 
    vid_type = {FIXED_STRING(<N>)) | INT64}];

partition_num	指定图空间的分片数量。建议设置为5倍的集群硬盘数量。例如集群中有3个硬盘，建议设置15个分片。
replica_factor	指定每个分片的副本数量。建议在生产环境中设置为3，在测试环境中设置为1。由于需要进行基于quorum的选举，副本数量必须是奇数。
vid_type	指定点ID的数据类型。可选值为FIXED_STRING(<N>)和INT64。FIXED_STRING(<N>)表示数据类型为字符串，最大长度为N，超出长度会报错；INT64表示数据类型为整数。默认值为FIXED_STRING(8)。

CREATE SPACE nba (partition_num=15, replica_factor=1, vid_type=fixed_string(30));
create space if not exists ssr (partition_num=5, replica_factor=1,vid_type=int64);
```

- 查看图空间

``` 
 SHOW SPACES;
```

- 检查分片的分布情况

``` 
SHOW HOSTS;
```

> 如果Leader distribution分布不均匀，请执行命令`BALANCE LEADER`重新分配

- 选择图空间

``` 
USE ssr;
```


#### 创建标签和边类型

``` 
CREATE {TAG | EDGE} {<tag_name> | <edge_type>}(<property_name> <data_type>
[, <property_name> <data_type> ...]);
```

名称	|类型	|属性
---|---|---
player	|Tag	|name (string), age (int)
team	|Tag	|name (string)
follow	|Edge type	|degree (int)
serve	|Edge type	|start_year (int), end_year (int)

``` 
创建标签player和team，以及边类型follow和serve。

// 创建两个标签
CREATE TAG player(name string, age int);
create tag team(name string);

show tags;

// 创建两个边类型
create edge follow(degree int);
create edge serve(start_year int, end_year int);

show edges;
```

#### 插入点和边
- 插入点

``` 
INSERT VERTEX <tag_name> (<property_name>[, <property_name>...])
[, <tag_name> (<property_name>[, <property_name>...]), ...]
{VALUES | VALUE} <vid>: (<property_value>[, <property_value>...])
[, <vid>: (<property_value>[, <property_value>...];
```

> VID是Vertex ID的缩写，VID在一个图空间中是唯一的。

- 插入边

``` 
INSERT EDGE <edge_type> (<property_name>[, <property_name>...])
{VALUES | VALUE} <src_vid> -> <dst_vid>[@<rank>] : (<property_value>[, <property_value>...])
[, <src_vid> -> <dst_vid>[@<rank> : (<property_name>[, <property_name>...]), ...];
```

- 插入代表NBA球员和球队的点。

``` 
// 插入3个队员
insert vertex player(name, age) values "player100":("Tim Duncan", 42);
insert vertex player(name, age) values "player101": ("Tony Parker", 36);
insert vertex player(name, age) values "player102": ("LaMarcus Aldridge", 33);

// 插入两个球队
insert vertex team(name) values "team200":("Warriors"), "team201": ("Nuggets");
```

- 插入代表NBA球队和球员之间的关系边 (有向边)

``` 
// 插入球员之间的关系
insert edge follow(degree) values "player100" -> "player101": (95);
insert edge follow(degree) values "player100" -> "player102": (90);
insert edge follow(degree) values "player102" -> "player101": (75);

// 插入球队和球员之间的关系
insert edge serve(start_year, end_year) values "player101" -> "team200": (1997, 2016), "payer101" -> "team201":(1999,2018);
```

#### 查询数据
- `GO`: 可以更具条件 遍历数据库。GO语句从一个或多个点开始, 沿一条或多条边遍历, 返回`YIELD`之句中指定的信息
- `FETCH`: 获得点或者边的属性
- `LOOKUP`: 基于索引, 和WHERE自句一起使用， 查找符合特定条件的数据
- `MATCH`: 依赖索引去匹配数据库中的数据模型 

##### `GO` 可以更具条件 遍历数据库

``` 
GO [[<M> TO] <N> STEPS ] FROM <vertex_list>
OVER <edge_type_list> [REVERSELY] [BIDIRECT]
[WHERE <expression> [AND | OR expression ...])]
YIELD [DISTINCT] <return_list>;
```

- 从VID为player100的球员开始，沿着边follow找到连接的球员

``` 
// 以player100点为基础 按follow边进行遍历
GO FROM "player100" OVER follow;

// 以player100为基础 按follow边进行遍历 条件player tag的age属性 >= 35 查询 player tag 的 name, age属性
GO FROM "player100" over follow where $$.player.age >= 35 yield $$.player.name as Teammate, $$.player.age AS Age;

// !!! 管道 将上此查询的结果  作为这次的输入
GO FROM "player100" OVER follow YIELD follow._dst AS id | GO FROM $-.id OVER serve YIELD $$.team.name AS Team,$^.player.name as player;

GO FROM "player100" OVER follow YIELD follow._dst as id ;
GO FROM "player102" OVER serve;
```

子句/符号	|说明
---|---
`$^`	|表示边的起点。
`$-`	|表示管道符前面的查询输出的结果集。

##### `FETCH`  获得点或者边的属性

``` 
FETCH PROP ON player "player100"
```

##### 修改点 或 边
- `UPDATE` `UPSERT`: UPSERT是UPDATE和INSERT的结合体。当您使用UPSERT更新一个点或边，如果它不存在，数据库会自动插入一个新的点或边。
- `UPDATE` 点

``` 
UPDATE VERTEX <vid> SET <properties to be updated>
[WHEN <condition>] [YIELD <columns>];

// 用UPDATE修改VID为player100的球员的name属性，然后用FETCH语句检查结果

UPDATE vertex "player100" SET player.name = "TimX";

FETCH PROP ON player "player100"; 
```

- `UPDATE` 边

``` 
UPDATE EDGE <source vid> -> <destination vid> [@rank] OF <edge_type>
SET <properties to be updated> [WHEN <condition>] [YIELD <columns to be output>];

// 两点确定一条边
UPDATE edge "player100" -> "player101" of follow set degree = 96;

FETCH PROP ON follow "player100" -> "player101";
```

- `UPSERT` 点 or 边

``` 
UPSERT {VERTEX <vid> | EDGE <edge_type>} SET <update_columns>
[WHEN <condition>] [YIELD <columns>];

INSERT VERTEX player(name, age) values "player111": ("Ben Simmons", 22);

UPSERT VERTEX "player111" SET player.name = "Dwight Howard", player.age = $^.player.age + 11 
WHEN $^.player.name == "Ben Simmons" AND $^.player.age > 20 
YIELD $^.player.name AS Name, $^.player.age AS Age;
```

#### 删除点和边
- 删除点 

``` 
DELETE VERTEX <vid1>[, <vid2>...]

delete vertex "team1", "team2";
```

- 删除边

``` 
DELETE EDGE <edge_type> <src_vid> -> <dst_vid>[@<rank>]
[, <src_vid> -> <dst_vid>...]

delete edge follow "team1" -> "team2";
```

#### 索引

可以通过`CREATE INDEX`语句为标签（`tag`）和边类型（`edge type`）增加索引。

- `MATCH`和`LOOKUP`语句的执行都依赖索引，但是索引会导致写性能大幅降低（`降低90%甚至更多`）。请不要随意在生产环境中使用索引，除非您很清楚使用索引对业务的影响。
- 您必须为已存在的数据重建索引，否则不能索引已存在的数据，导致无法在MATCH和LOOKUP语句中返回这些数据。更多信息，请参见重建索引。

- 创建索引

``` 
CREATE {TAG | EDGE} INDEX [IF NOT EXISTS] <index_name>
ON {<tag_name> | <edge_name>} (prop_name_list);

CREATE TAG INDEX player_index_0 on player(name(20));
```

- 重建索引

``` 
REBUILD {TAG | EDGE} INDEX <index_name>;

REBUILD TAG INDEX player_index_0;
```

> 说明：为没有指定长度的变量属性创建索引时，需要指定索引长度。在utf-8编码中，一个中文字符占3字节，请根据变量属性长度设置合适的索引长度。例如10个中文字符，索引长度需要为30。详情请参见创建索引。

#### LOOKUP
- 更具索引遍历数据
- 通过标签列出点: 检索指定标签的所有点ID。
- 通过边类型列出边：检索指定边类型的所有边的起始点、目的点和rank。
- 计数指定标签的点或指定边类型的边。

``` 
LOOKUP ON {<vertex_tag> | <edge_type>} [WHERE <expression> [AND <expression> ...]] [YIELD <return_list>];

<return_list>
    <prop_name> [AS <col_alias>] [, <prop_name> [AS <prop_alias>] ...];
```

``` 
CREATE TAG INDEX player_name_0 on player(name(10));

REBUILD TAG INDEX player_name_0

LOOKUP ON player WHERE player.name == "Tony Parker" \
YIELD player.name, player.age;

MATCH (v:player{name:"Tony Parker"}) RETURN v;
```

#### Match
- 基于模式（pattern）匹配的搜索功能
- 一个MATCH语句定义了一个搜索模式，用该模式匹配存储在Nebula Graph中的数据，然后用RETURN子句检索数据。

``` 
MATCH <pattern> [<WHERE clause>] RETURN <output>
```

- 匹配所有 tag为player 的点

``` 
// 匹配所有 tag为player 的点
MATCH (v:player) RETURN v;

```

- 条件匹配

```
// select * from player where name = "Tim Duncan"
MATCH(v: player) where v.name == "Tim Duncan" RETURN v;
MATCH(v: player{name: "Tim Duncan"}) RETURN v;

```

- 匹配点
 
```
// 匹配点
MATCH(v) WHERE id(v) == 'player101' RETURN v;

``` 

- 匹配多个点 

```
// 匹配多个点  select * from player where name = 'Dwight Howard' and id in ['player111']
MATCH (v:player { name: 'Dwight Howard' })
WHERE id(v) in ["player111"]
RETURN v;

``` 

- 匹配链接的点

``` 
// 匹配链接的点  `--符号表示两个方向的边，并匹配这些边连接的点。`
MATCH (v:player{name:"Tim Duncan"})--(v2) 
        RETURN v2.name AS Name;

// 匹配链接的点 加上指向
MATCH (v:player{name:"Tim Duncan"})-->(v2)<--(v3) \
        RETURN v3.name AS Name;

```

-  匹配路径

``` 
// 匹配路径
MATCH p=(v:player{name: 'TimX'})-->(v2) 
        RETURN v2 AS Name, p;

resp:
name, p
("team200" :team{name: "Warriors"})	<("player100" :player{age: 42, name: "TimX"})-[:serve@0 {end_year: 2016, start_year: 1997}]->("team200" :team{name: "Warriors"})>
("player102" :player{age: 33, name: "LaMarcus Aldridge"})	<("player100" :player{age: 42, name: "TimX"})-[:follow@0 {degree: 90}]->("player102" :player{age: 33, name: "LaMarcus Aldridge"})>
("player101" :player{age: 36, name: "Tony Parker"})	<("player100" :player{age: 42, name: "TimX"})-[:follow@0 {degree: 96}]->("player101" :player{age: 36, name: "Tony Parker"})>


``` 

- 匹配边

``` 
// 匹配边 `除了用--、-->、<--表示未命名的边之外，您还可以在方括号中使用自定义变量命名边。例如-[e]-。`
MATCH (v:player{name: 'TimX'})-[e]->(v2) 
        RETURN e;

// 匹配边类型和属性
MATCH (v:player{name:"Tim Duncan"})-[e:serve]-(v2) 
        RETURN e;

// 匹配多个边类型
MATCH (v:player{name:"Tim Duncan"})-[e:follow|:serve]->(v2) \
        RETURN e;

``` 

- 匹配多条边

``` 
// 匹配多条边
MATCH (v:player{name:"Tim Duncan"})-[]->(v2)<-[e:serve]-(v3) 
        RETURN v2, v3;
```

- 匹配定长路径
在模式中使用`:<edge_type>*<hop>`匹配定长路径。`hop`必须是一个非负整数

``` 
MATCH p=(v:player{name:"Tim Duncan"})-[e:follow*2]->(v2) 
        RETURN DISTINCT v2 AS Friends;
```

- 匹配1～3长度 所有节点
``` 
MATCH (v:saic)-[e:saic_to_investment*1..3]->(v2) where id(v) == "1025417406" 
         RETURN v.name as src_name, v2.name as dst_name, id(v) as src, id(v2) as dst,e.share as share;
```
