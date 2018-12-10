
这里尝试比较Dgraph与其他流行的图数据库/数据存储之间的不同。以下是简要的说明，可以帮助您判断Dgraph是否符合您的需求。

# 基于批处理

基于批处理的图处理框架提供了非常高的吞吐量来定期处理数据。这对于将图数据转换为其他系统可随时使用的形态然后将数据提供给最终用户非常有用。

## Pregel

- [Pregel](https://kowshik.github.io/JPregel/pregel_paper.pdf)，是Google的一个大规模图形处理系统。您可以将其视为与MapReduce/Hadoop等效。
- Pregel不直接暴露给用户，即运行实时更新并执行任意复杂性查询。Dgraph能够以低延迟响应任意复杂的用户查询并允许用户交互。
- Pregel可以与Dgraph一起使用，用于于补充处理图，这样通过Dgraph运行需要一分钟以上的查询，或者产生太多数据的查询就可以被客户直接使用。

---

# 数据库

图数据库优化内部数据表示，以便能够有效地进行图形操作。

## Neo4j

[Neo4j](https://neo4j.com/)是2007年以来[db-engines.com](http://db-engines.com/en/ranking/graph+dbms)中最受欢迎的图数据库，Dgraph是一个比较新的图数据库，可以扩展到Google Web规模并作为主要数据库进行严格的生产使用。

### 语言

Neo4j支持Cypher和Gremlin查询语言。Dgraph支持[GraphQL+-](query-language/index.md)，是[GraphQL](https://facebook.github.io/graphql/)的变体，一种由Facebook创建的查询语言。与以简单列表格式生成结果的Cypher或Gremlin相反，GraphQL允许以子图格式生成结果，其具有更丰富的语义。 此外，GraphQL支持Schema验证，这有助于确保输入和输出期间的数据正确性。

虽然当前只支持GraphQL，但Gremlin和Cypher更受欢迎。Dgraph计划在v1.0版本之后支持它们。

### 可扩展性

Neo4j在单个服务器上运行。企业版Neo4j仅运行通用数据副本。随着数据的扩展，需要用户垂直扩展其服务器。 [垂直缩放是昂贵的](https://blog.openshift.com/best-practices-for-horizontal-application-scaling/)

Dgraph具有分布式架构。您可以在多个Dgraph服务器之间拆分数据以水平分发它。当您添加更多数据时，您可以添加更多商用硬件来为其提供服务。Dgraph提供了更多性能功能，如减少群集中的网络调用和高度并发的查询执行，以实现高查询吞吐量。 Dgraph对每个分片进行一致的复制，这使其具有崩溃弹性，并保护用户免受服务器停机影响。

### 事物

两个系统都提供ACID事物。NeN4J支持其单服务器体系结构中的ACID事物。尽管Dgraph是分布式且一致复制的系统，但它支持具有快照隔离的ACID事务。

### 复制

Neo4j的通用数据复制仅适用于购买它们的用户[企业证书](https://neo4j.com/subscriptions/#editions)。在Dgraph，我们将横向扩展和一致复制视为当今构建的任何应用程序的基本必需品。Dgraph不仅会自动对数据进行分片，还会移动数据以重新平衡这些分片，从而使用户实现最佳的机器利用率和查询延迟。

Dgraph一直在复制。 任何写入后的读取都将对客户端可见，而不管它属于哪个副本。简而言之，我们实现了线性化读取。

***有关Dgraph与Neo4j的更全面比较，您可以阅读我们的[博客](https://open.dgraph.io/post/benchmark-neo4j)***

---

# 数据存储

图数据存储就像一些其他SQL/NoSQL数据库之上的图层一样，可以为它们进行数据管理。其他数据库负责备份，快照，服务器故障和数据完整性。

## Cayley

- [Cayley](https://cayley.io/)和Dgraph都主要使用Go语言编写，受到Google不同项目的启发。
- Cayley就像一个图层，提供了一个可以由各种存储实现的存储接口，例如PostGreSQL，RocksDB用于单个机器，MongoDB允许分发。换句话说，Cayley将数据移交给其他数据库。Dgraph使用[Badger](https://github.com/dgraph-io/badger), 它拥有完整的数据所有权和紧密耦合的数据存储和管理，以实现高效的分布式查询。
- Cayley的设计受到高fan-out问题的困扰。其中，如果中间步骤导致返回大量结果，并且数据是分布的，那么它将导致Cayley和底层数据层之间的许多网络调用。 Dgraph的设计最小化了网络调用的数量，从而减少了响应查询所需的服务器数量。这种设计在集群中产生更好的、可预测的查询延迟，即使集群规模加大。

***要比较Dgraph和Cayley的查询和数据加载基准，可以阅读[Differences between Dgraph and Cayley](https://discuss.dgraph.io/t/differences-between-dgraph-and-cayley/23/3)***.
