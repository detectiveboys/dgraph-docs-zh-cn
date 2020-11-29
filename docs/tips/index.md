
## 获取样本数据

使用 `has`函数获取一些样本节点。

```graphql
{
  result(func: has(director.film), first: 10) {
    uid
    expand(_all_)
  }
}
```

## 统计连接节点数

使用`expand(_all_)` 展开节点的边，然后将展开的uid分配给变量。现在可以使用该变量迭代唯一的相邻节点。然后使用`count(uid)`来计算变量块中的节点数。

```graphql
{
  uids(func: has(director.film), first: 1) {
    uid
    expand(_all_) { u as uid }
  }

  result(func: uid(u)) {
    count(uid)
  }
}
```

## 搜索无索引谓词

在值变量中使用`has`函数搜索无索引的谓词。

```graphql
{
  var(func: has(festival.date_founded)) {
    p as festival.date_founded
  }
  query(func: eq(val(p), "1961-01-01T00:00:00Z")) {
      uid
      name@en
      name@ru
      name@pl
      festival.date_founded
      festival.focus { name@en }
      festival.individual_festivals { total : count(uid) }
  }
}
```
