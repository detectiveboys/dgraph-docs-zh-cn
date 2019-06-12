Dgraph的 GraphQL+- 基于Facebook的[GraphQL](https://facebook.github.io/graphql/)。GraphQL不是为Graph数据库开发的，但它的图形式查询语法，模式验证和子图形状响应使其成为一种很好的语言选择。我们修改了语言以更好地支持图形操作，添加和删除功能以最适合图形数据库。我们称这种简化的，功能丰富的语言为“GraphQL+- ”。

GraphQL+-正在开发中。我们正在添加更多功能，我们可能会进一步简化现有功能。

## 浏览一下 - https://tour.dgraph.io

本文档是Dgraph查询参考资料。这不是一个教程。它被设计为已经知道如何在GraphQL+-中编写查询的用户的参考，但需要检查语法，索引或函数等。

**注意** *如果您是Dgraph的新手并想学习如何使用Dgraph和GraphQL+-，请浏览一下 * - https://tour.dgraph.io


### 运行示例

该参考文献中的示例使用了关于电影和演员的2100万三倍的数据库。示例查询运行并返回结果。查询由运行在 https://play.dgraph.io/ 的Dgraph实例执行。要在本地运行查询或进行更多实验，请参阅[入门]({{< relref "get-started/index.md" >}}) 指南，该指南还说明如何加载此处示例中使用的数据集。

## GraphQL+-基本原理

GraphQL+- 查询根据搜索条件查找节点，匹配图中的模式并返回图形作为结果。

查询由嵌套块组成，从查询根开始。根找到初始节点集，应用以下图形匹配和过滤。


### 返回结果

每个查询都有一个名称，在查询根目录中指定，并且相同的名称标识结果。

如果边是值类型，则可以通过给出边名来返回该值。

查询示例：在示例数据集中，将电影链接到导演和演员的边，电影具有许多众所周知的电影数据库的名称，发布日期和标识符。这个名为`bladerunner`的查询，以及与电影名称匹配的根，返回80年代早期科幻经典“Blade Runner”的值。

```graphql
{
  bladerunner(func: eq(name@en, "Blade Runner")) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

对于所有“name”边缘等于“Blade Runner”的节点，查询首先使用索引搜索图形，以使搜索更有效。对于找到的节点，查询然后返回列出的传出边。

每个节点都有一个唯一的64位标识符。上面查询中的`uid`边缘返回该标识符。如果已知所需节点，则函数“uid”找到该节点。

查询示例：通过UID找到的“Blade Runner”电影数据。

```
{
  bladerunner(func: uid(0x107b2c)) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

查询可以匹配许多节点并返回每个节点的值。

查询示例：名称中包含“Blade”或“Runner”的所有节点。

```
{
  bladerunner(func: anyofterms(name@en, "Blade Runner")) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

可以在列表中为`uid`函数指定多个ID。

查询示例：
```
{
  movies(func: uid(0x107b2c, 0x85f961)) {
    uid
    name@en
    initial_release_date
    netflix_id
    ~director.film {
      uid
      name@en
    }
  }
}
```

**注意** *如果你的谓词有特殊字符，那么你应该在查询中询问它时用尖括号包装它。 例如 `<first:name>`*


### 扩展图形边缘

查询通过使用 `{ }` 嵌套查询块来扩展节点之间的边缘。

Query Example: 查询在“Blade Runner”中扮演的演员和角色。 查询首先找到名为“Blade Runner”的节点, 然后将传出的“starring”边缘跟随表示actor作为角色的表现的节点。 从那里扩展`performance.actor` 和 `performance,character` 的边，以找到电影中每个演员的演员姓名和角色。
```
{
  brCharacters(func: eq(name@en, "Blade Runner")) {
    name@en
    initial_release_date
    starring {
      performance.actor {
        name@en  # actor name
      }
      performance.character {
        name@en  # character name
      }
    }
  }
}
```


### 注释

`#` 后面的任何内容都是注释。

### 应用过滤器

查询根找到一组初始节点，查询通过返回值和后续边缘继续进一步节点继续进行 - 查询中到达的任何节点都是在根搜索后通过遍历找到的。 找到的节点可以通过在根之后或任何边缘应用 `@ filter` 进行过滤。

查询示例: 查询导演雷德利斯科特在2000年之前发布的电影“Blade Runner”。
```
{
  scott(func: eq(name@en, "Ridley Scott")) {
    name@en
    initial_release_date
    director.film @filter(le(initial_release_date, "2000")) {
      name@en
      initial_release_date
    }
  }
}
```

查询示例：查询标题中带有“Blade”或“Runner”，并在2000年之前发行的电影。

```
{
  bladerunner(func: anyofterms(name@en, "Blade Runner")) @filter(le(initial_release_date, "2000")) {
    uid
    name@en
    initial_release_date
    netflix_id
  }
}
```

### 语言支持

**注意** *这 `@lang` 指令 必须明确指定对应schema 的 query 或者 mutate
的谓语后面跟随的特定的语言标记.*

Dgraph 支持 UTF-8 字符.

在一个查询中, 对于一个字符类型的 `edge`, 语法如下
```
edge@lang1:...:langN
```
按照如下规则，依据指定的顺序与语言返回

* 最多一个返回.
* 列表返回顺序，优先级从左到右: 一个没找到就往下找，找到就返回.
* 这个列表都没匹配到，就不返回.
* 最后一个 `.`意味着没有指定语言或者对应的值没有匹配到特定的语言, 某个被匹配到的语言会返回.

举例如下:

- `name`   => 查找标识值为空的语言; 如果标识值不存在，什么都不返回.
- `name@.` => 查找标识值为空的语言,然后查找任意一种语言.
- `name@en` => 查找标识值为 `en`的语言; 如果标识值 `en` 没匹配到，什么都不返回.
- `name@en:.` =>  查找标识值为 `en`的语言, 然后是未标识的语言, 最后是任意一种语言.
- `name@en:pl` => 查找标识值为 `en`的语言, 然后是 `pl`, 没有匹配到,什么都不返回.
- `name@en:pl:.` => 查找标识值为`en`的语言, 然后是`pl`的语言, 接着找未标识的语言, 最后是任意一种语言.


**注意** *在函数中,不支持多种语言.只能查找一种语言, `.` 符号 和 属性名也不支持多语言*


**注意** *在全文搜索函数中(`alloftext`, `anyoftext`), 没有指定特定语言的标识值，默认使用英文标识器tokenizer*


查询案例: 查找演员 Farhan Akhtar's 导演的电影 按照下面三种语言分类 俄语 印度语 英语 显示,其他语言不显示.

```
{
  q(func: allofterms(name@en, "Farhan Akhtar")) {
    name@hi
    name@en

    director.film {
      name@ru:hi:en
      name@en
      name@hi
      name@ru
    }
  }
}
```




## 函数

**注意** *函数只能被应用在已经建立索引的谓词上*

函数允许基于节点或变量的属性进行筛选。函数可以应用于查询根或过滤器中。

对于字符串值谓词上的函数，如果没有提供语言首选项，则将该函数应用于所有没有语言标记的语言和字符串;如果给定了语言首选项，则该函数仅应用于给定语言的字符串.


### 词匹配


#### 所有的词 allofterms 

语法: `allofterms(predicate, "space-separated term list")`

Schema 类型: `string`

索引 要求: `term`


匹配以任何顺序包含所有指定项的字符串;不区分大小写.

##### 在根节点使用 Usage at root

查询案例: 在全部节点中查找 名字`name` 包含词`indiana` 和 `jones`, 返回 英语名字 和英语电影类型.

```
{
  me(func: allofterms(name@en, "jones indiana")) {
    name@en
    genre {
      name@en
    }
  }
}
```

##### 使用过滤器

查询案例: 所有史蒂文·斯皮尔伯格的电影中 `indiana` and `jones`这两个词.  这个过滤器`@filter(has(director.film))` 删除不是Steven Spielberg导演的节点 --- 这些数据还包括一部名为史蒂文·斯皮尔伯格(Steven Spielberg)的电影中的一个角色.

```
{
  me(func: eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
    name@en
    director.film @filter(allofterms(name@en, "jones indiana"))  {
      name@en
    }
  }
}
```


#### 任意一个词 anyofterms


语法: `anyofterms(predicate, "space-separated term list")`

Schema 类型: `string`

索引 要求: `term`


匹配具有任意顺序指定项的字符串;不区分大小写.

##### 在根节点使用 Usage at root

查询案例: 所有节点名字`name`包含 `poison` 或者 `peacock`. 返回的许多节点是影片, 但是像琼·皮科克(Joan Peacock)这样的人也符合搜索条件,因为级联指令没有指定查询需要类型。

```
{
  me(func:anyofterms(name@en, "poison peacock")) {
    name@en
    genre {
      name@en
    }
  }
}
```


##### Usage as filter

查询案例:  所有史蒂文·斯皮尔伯格的电影都包含战争或间谍 `war` or `spies`。过滤器 `@filter(has(director.film))`过滤掉名为Steven Spielberg的节点，但这些这些节点不是导演———这些数据包含一个名为Steven Spielberg的电影中的角色。

```
{
  me(func: eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
    name@en
    director.film @filter(anyofterms(name@en, "war spies"))  {
      name@en
    }
  }
}
```


### 正则表达式


语法: `regexp(predicate, /regular-expression/)` 或不区分大小写 `regexp(predicate, /regular-expression/i)`

Schema 类型: `string`

索引 要求: `trigram`


通过正则表达式匹配字符串。正则表达式语言是[go语言的正则表达式](https://golang.org/pkg/regexp/syntax/).

查询案例: 从根节点开始，将节点与名称开头为 `Steven Sp`匹配，后面跟着任何字符。对于每个匹配的uid，匹配包含 `ryan`的影片。注意与 `allofterms`函数的不同, `allofterms`函数 只匹配`ryan`但是正则表达式会所有包含`ryan`, 例如 `bryan`.

```
{
  directors(func: regexp(name@en, /^Steven Sp.*$/)) {
    name@en
    director.film @filter(regexp(name@en, /ryan/i)) {
      name@en
    }
  }
}
```


####  技术细节

一个三元组是三个连续符文的子串。例如 `Dgraph` 的三元组有 `Dgr`, `gra`, `rap`, `aph`。

为保证正则表达式匹配的效率, Dgraph使用三元组索引[trigram indexing](https://swtch.com/~rsc/regexp/regexp4.html).也就是说，Dgraph将正则表达式转换为三元组查询，使用三元组索引和三元组查询查找可能的匹配项，并仅对可能的项应用完整的正则表达式搜索。

#### 编写有效的正则表达式和限制条件

当你设计正则表达式查询语句是把如下建议记在脑中

- 至少一个三元组必须与正则表达式匹配 (不支持正则模式少于三个字符的)。也就是说, Dgraph 要求查询的正则表达式要能被转换为一个三元组.
- 正则表达式匹配的备选三元组的数量应该尽可能少(`[a-zA-Z][a-zA-Z][0-9]` 这样的正则不是一个好选择)。许多可能的匹配意味着对要对许多字符串检查完整的正则表达式; 然而,如果正则表达式强制匹配更多的三元组，Dgraph 最好地使用索引，并根据更小的可能匹配集代替检查完整的正则表达式。
- 因此，正则表达式应该尽可能精确。匹配较长的字符串意味着需要更多的三元组，这有助于有效地使用索引。
- 如果使用重复的正则符号(`*`, `+`, `?`, `{n,m}`), 整个正则表达式必须不匹配空字符串或任意字符串，例如, `*` 可以这样使用`[Aa]bcd*` 但不能这样 `(abcd)*` 或者这样 `(abcd)|((defg)*)`
- 重复的正则符号在括号表达式后面(例如. `[fgh]{7}`, `[0-9]+` or `[a-z]{3,5}`) 通常被认为匹配任意字符串因为他们匹配太多元组。
- 如果部分结果(对于三元组的子集) 超过 1000000 uids 在索引扫描时,这条查询由于过于查询代价过于昂贵阻止被禁止。


### 全文检索

语法:  `alloftext(predicate, "space-separated text")` 和 `anyoftext(predicate, "space-separated text")`

Schema 类型: `string`

索引 要求:  `fulltext`


应用词干分析和停止词的全文检索来查找匹配所有或任何给定文本的字符串。

在索引生成和处理全文搜索参数时，应用以下步骤:

1. 标记化(根据Unicode单词边界)。
1. 转换为小写的。
1. 单点标准化(以KC形式标准化)(to [Normalization Form KC](http://unicode.org/reports/tr15/#Norm_Forms)).
1. 使用特定于语言的词干分析器进行词干分析(如果有语言支持)。
1. 停止词删除(如果有语言支持)。

Dgraph使用[bleve](https://github.com/blevesearch/bleve)作为全文搜索索引。参见bleve语言特定的[停止单词列表](https://github.com/blevesearch/bleve/tree/master/analysis/lang)。

下表包含所有支持的语言，对应的国家代码，词干提取和停止过滤支持。

|  Language  | Country Code | Stemming | Stop words |
| :--------: | :----------: | :------: | :--------: |
|   Arabic   |      ar      | &#10003; |  &#10003;  |
|  Armenian  |      hy      |          |  &#10003;  |
|   Basque   |      eu      |          |  &#10003;  |
| Bulgarian  |      bg      |          |  &#10003;  |
|  Catalan   |      ca      |          |  &#10003;  |
|  Chinese   |      zh      | &#10003; |  &#10003;  |
|   Czech    |      cs      |          |  &#10003;  |
|   Danish   |      da      | &#10003; |  &#10003;  |
|   Dutch    |      nl      | &#10003; |  &#10003;  |
|  English   |      en      | &#10003; |  &#10003;  |
|  Finnish   |      fi      | &#10003; |  &#10003;  |
|   French   |      fr      | &#10003; |  &#10003;  |
|   Gaelic   |      ga      |          |  &#10003;  |
|  Galician  |      gl      |          |  &#10003;  |
|   German   |      de      | &#10003; |  &#10003;  |
|   Greek    |      el      |          |  &#10003;  |
|   Hindi    |      hi      | &#10003; |  &#10003;  |
| Hungarian  |      hu      | &#10003; |  &#10003;  |
| Indonesian |      id      |          |  &#10003;  |
|  Italian   |      it      | &#10003; |  &#10003;  |
|  Japanese  |      ja      | &#10003; |  &#10003;  |
|   Korean   |      ko      | &#10003; |  &#10003;  |
| Norwegian  |      no      | &#10003; |  &#10003;  |
|  Persian   |      fa      |          |  &#10003;  |
| Portuguese |      pt      | &#10003; |  &#10003;  |
|  Romanian  |      ro      | &#10003; |  &#10003;  |
|  Russian   |      ru      | &#10003; |  &#10003;  |
|  Spanish   |      es      | &#10003; |  &#10003;  |
|  Swedish   |      sv      | &#10003; |  &#10003;  |
|  Turkish   |      tr      | &#10003; |  &#10003;  |


查询案例: 所有名字有`run`, `running`, 等词 和 `man`。消除停止字 `the` 和 `maybe`

```
{
  movie(func:alloftext(name@en, "the man maybe runs")) {
	 name@en
  }
}
```


### 不等式

#### 等于

语法例子:

* `eq(predicate, value)`
* `eq(val(varName), value)`
* `eq(predicate, val(varName))`
* `eq(count(predicate), value)`
* `eq(predicate, [val1, val2, ..., valN])`

Schema 类型: `int`, `float`, `bool`, `string`, `dateTime`

索引 要求:  `eq(predicate, ...)`  需要一个索引 (请参阅下面的表)。 对于在查询根 `count(predicate)`,需要`@count`上有索引.对于变量，值是作为查询的一部分计算的，因此不需要索引。

| Type       | Index Options   |
| :--------- | :-------------- |
| `int`      | `int`           |
| `float`    | `float`         |
| `bool`     | `bool`          |
| `string`   | `exact`, `hash` |
| `dateTime` | `dateTime`      |

测试谓词或变量的值是否相等或能否与列表中的值对应。

布尔常量是 `true` and `false`, 因此对于 `eq` , 就变成了, `eq(boolPred, true)`.

查询示例: 恰好属于有13种类型的电影.

```
{
  me(func: eq(count(genre), 13)) {
    name@en
    genre {
    	name@en
    }
  }
}
```


查询示例: 名字叫史蒂文且执导过1部、2部或3部电影。

```
{
  steve as var(func: allofterms(name@en, "Steven")) {
    films as count(director.film)
  }

  stevens(func: uid(steve)) @filter(eq(val(films), [1,2,3])) {
    name@en
    numFilms : val(films)
  }
}
```


#### 小于，小于或等于，大于，大于或等于

语法示例:不等式 `IE`

* `IE(predicate, value)`
* `IE(val(varName), value)`
* `IE(predicate, val(varName))`
* `IE(count(predicate), value)`

`IE` 可以替换成下面这些

* `le`  小于或等于
* `lt` 小于
* `ge` 大于或等于
* `gt` 大于

Schema 类型: `int`, `float`, `string`, `dateTime`

索引 要求: `IE(predicate, ...)` 需要一个索引 (请参阅下面的表)。 对于在查询根 `count(predicate)`,需要`@count`上有索引.对于变量，值是作为查询的一部分计算的，因此不需要索引。

| Type       | Index Options |
| :--------- | :------------ |
| `int`      | `int`         |
| `float`    | `float`       |
| `string`   | `exact`       |
| `dateTime` | `dateTime`    |


查询示例: 1980年以前上映的雷德利·斯科特电影。

```
{
  me(func: eq(name@en, "Ridley Scott")) {
    name@en
    director.film @filter(lt(initial_release_date, "1980-01-01"))  {
      initial_release_date
      name@en
    }
  }
}
```


查询示例:电影导演名字含有 `Steven` 同时指导超过100名演员。

```
{
  ID as var(func: allofterms(name@en, "Steven")) {
    director.film {
      num_actors as count(starring)
    }
    total as sum(val(num_actors))
  }

  dirs(func: uid(ID)) @filter(gt(val(total), 100)) {
    name@en
    total_actors : val(total)
  }
}
```



查询示例:每类电影超过30000部。因为这边没有指定电影种类返回按照什么顺序排序 将使用UID排序。count 索引记录节点外的边数，并进行更多的查询。

```
{
  genre(func: gt(count(~genre), 30000)){
    name@en
    ~genre (first:1) {
      name@en
    }
  }
}
```

查询示例:查找名字为斯蒂芬·斯皮尔伯格导演的电影，同时要求initial_release_date大于（大于就是晚于）电影《少数派报告》的initial_release_date(首次发布日期)。

```
{
  var(func: eq(name@en,"Minority Report")) {
    d as initial_release_date
  }

  me(func: eq(name@en, "Steven Spielberg")) {
    name@en
    director.film @filter(ge(initial_release_date, val(d))) {
      initial_release_date
      name@en
    }
  }
}
```


### uid

语法示例:

* `q(func: uid(<uid>)) `
* `predicate @filter(uid(<uid1>, ..., <uidn>))`
* `predicate @filter(uid(a))` 使用变量 `a`
* `q(func: uid(a,b))` 使用变量 `a` 和 `b`


将当前查询级别的节点过滤到给定uid集中的节点。

对于查询变量 `a`, `uid(a)`表示存储在 `a` 其中的一组uid。 对于值变量 `b`, `uid(b)` 表示从UID到值映射的UID.  有两个或两个以上的变量, `uid(a,b,...)`表示所有变量的并集。

`uid(<uid>)`, 像标识函数一样, 即使节点没有任何边, 也会返回请求的 UID。

查询示例: 如果已知节点的UID，则可以直接读取该节点的值。如已知电影普里扬卡·乔普的UID 为 0x878110，可以通过UID直接查

```
{
  films(func: uid(0x878110)) {
    name@hi
    actor.film {
      performance.film {
        name@hi
      }
    }
  }
}
```



查询示例: 塔拉吉·汉森的电影按类型划分
```
{
  var(func: allofterms(name@en, "Taraji Henson")) {
    actor.film {
      F as performance.film {
        G as genre
      }
    }
  }

  Taraji_films_by_genre(func: uid(G)) {
    genre_name : name@en
    films : ~genre @filter(uid(F)) {
      film_name : name@en
    }
  }
}
```



查询示例: 塔拉吉·汉森的电影按类型划分然后排序，最后统计每种类型电影的。
```
{
  var(func: allofterms(name@en, "Taraji Henson")) {
    actor.film {
      F as performance.film {
        G as count(genre)
        genre {
          C as count(~genre @filter(uid(F)))
        }
      }
    }
  }

  Taraji_films_by_genre_count(func: uid(G), orderdesc: val(G)) {
    film_name : name@en
    genres : genre (orderdesc: val(C)) {
      genre_name : name@en
    }
  }
}
```


### uid_in


语法 例子:

* `q(func: ...) @filter(uid_in(predicate, <uid>)`
* `predicate1 @filter(uid_in(predicate2, <uid>)`

Schema 类型: UID

索引 要求: 无

 `uid` 函数则根据uid过滤当前级别的节点,函数 `uid_in` 允许沿着边缘向前查看，以检查它是否指向特定的UID。这通常可以保存一个额外的查询块，并避免返回边缘。

`uid_in` 不能在根节点下使用，它接受一个UID常量作为参数(而不是变量)。

查询示例: Marc Caro和Jean-PierreJeunet(UID 0x6777ba)的合作。如果Jean-Pierre Jeunet的UID是已知的, 通过这种`~director.film`方式进行查询，就不需要一个块将其UID提取到变量中，也不需要额外的边缘遍历和过滤器 .
```
{
  caro(func: eq(name@en, "Marc Caro")) {
    name@en
    director.film @filter(uid_in(~director.film, 0x6777ba)){
      name@en
    }
  }
}
```


### has

语法 例子: `has(predicate)`

Schema 类型: all

确定节点是否具有特定谓词。

查询示例: 前五位导演和他们所有的电影都有上映日期的记录。导演至少导演过一部电影——相当于 `gt(count(director.film), 0)`.
```
{
  me(func: has(director.film), first: 5) {
    name@en
    director.film @filter(has(initial_release_date))  {
      initial_release_date
      name@en
    }
  }
}
```

### 定位

**注意** *到目前为止，我们只支持索引点、多边形和多边形集合类型。*


注意，对于定位查询，任何带有孔的多边形都将被替换为外部循环，忽略孔洞。另外，对于0.7.7版本，多边形包含检查是近似的。


#### Mutations（变化）

要使用geo函数，谓词上需要一个索引。
```
loc: geo @index(geo) .
```

下面是如何添加一个点

```
{
  set {
    <_:0xeb1dde9c> <loc> "{'type':'Point','coordinates':[-122.4220186,37.772318]}"^^<geo:geojson> .
    <_:0xeb1dde9c> <name> "Hamon Tower" .
  }
}
```

下面是如何将“多边形”与节点关联。添加一个“多边形集合”也是类似的。

```
{
  set {
    <_:0xf76c276b> <loc> "{'type':'Polygon','coordinates':[[[-122.409869,37.7785442],[-122.4097444,37.7786443],[-122.4097544,37.7786521],[-122.4096334,37.7787494],[-122.4096233,37.7787416],[-122.4094004,37.7789207],[-122.4095818,37.7790617],[-122.4097883,37.7792189],[-122.4102599,37.7788413],[-122.409869,37.7785442]],[[-122.4097357,37.7787848],[-122.4098499,37.778693],[-122.4099025,37.7787339],[-122.4097882,37.7788257],[-122.4097357,37.7787848]]]}"^^<geo:geojson> .
    <_:0xf76c276b> <name> "Best Western Americana Hotel" .
  }
}
```

上面的例子是从我们的[SF旅游](https://github.com/dgraph-io/benchmarks/blob/master/data/sf.tourism.gz?raw=true)数据集中挑选出来的。

#### 查询

##### 接近

语法 例子: `near(predicate, [long, lat], distance)`

Schema 类型: `geo`

索引 要求: `geo`

匹配'谓词'所给出的位置在`distance`米的geojson`[long,lat]`坐标的所有实体。

查询示例: 旅游景点在1公里内的一个点在旧金山的金门公园。

```
{
  tourist(func: near(loc, [-122.469829, 37.771935], 1000) ) {
    name
  }
}
```


##### 在…之内

语法 例子: `within(predicate, [[[long1, lat1], ..., [longN, latN]]])`

Schema 类型: `geo`

索引 要求: `geo`

匹配“谓词”给出的位置位于geojson坐标数组指定的多边形中的所有实体。

查询示例: 旧金山金门公园指定区域内的旅游景点。

```
{
  tourist(func: within(loc, [[[-122.47266769409178, 37.769018558337926 ], [ -122.47266769409178, 37.773699921075135 ], [ -122.4651575088501, 37.773699921075135 ], [ -122.4651575088501, 37.769018558337926 ], [ -122.47266769409178, 37.769018558337926]]] )) {
    name
  }
}
```


##### 包含

语法 例子: `contains(predicate, [long, lat])` or `contains(predicate, [[long1, lat1], ..., [longN, latN]])`

Schema 类型: `geo`

索引 要求: `geo`

匹配“谓词”给出的坐标的多边形`[long, lat]`或给定的geojson多边形的所有实体

查询示例 :所有实体中包含一个点在火烈鸟围场的旧金山动物园。
```
{
  tourist(func: contains(loc, [ -122.50326097011566, 37.73353615592843 ] )) {
    name
  }
}
```


##### 交叉

语法 例子: `intersects(predicate, [[[long1, lat1], ..., [longN, latN]]])`

Schema 类型: `geo`

索引 要求: `geo`

匹配"谓词"给定位置的多边形与给定geojson多边形相交的所有实体。


```
{
  tourist(func: intersects(loc, [[[-122.503325343132, 37.73345766902749 ], [ -122.503325343132, 37.733903134117966 ], [ -122.50271648168564, 37.733903134117966 ], [ -122.50271648168564, 37.73345766902749 ], [ -122.503325343132, 37.73345766902749]]] )) {
    name
  }
}
```



## 连接过滤器

在`@filter`中，多个函数可以与布尔连接词一起使用。

### AND, OR and NOT

连接词 `AND`, `OR` 和 `NOT` 连接过滤器，可以构建到任意复杂的过滤器中, 比如 `(NOT A OR B) AND (C AND NOT (D OR E))`。 注意, `NOT`与 `AND` 绑定比 `NOT` 与 `OR` 更紧密。

查询示例 :所有史蒂文斯皮尔伯格电影包含'印第安纳'和'琼斯'或“侏罗纪”和“公园”。

```
{
  me(func: eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
    name@en
    director.film @filter(allofterms(name@en, "jones indiana") OR allofterms(name@en, "jurassic park"))  {
      uid
      name@en
    }
  }
}
```


## 别名

语法 例子:

* `aliasName : predicate`
* `aliasName : predicate { ... }`
* `aliasName : varName as ...`
* `aliasName : count(predicate)`
* `aliasName : max(val(varName))`

别名可以结果中提供另一个名称。谓词，变量和聚合可以通过添加`:`来添加别名。别名不必与原始谓词名不同，但是，在一个作用域内， 别名必须与同一作用域中返回的谓词名和其他别名不同。别名可用于在一个作用域内多次返回相同的谓词。

查询示例 : 名称与“Steven”相匹配的导演，他们的UID，英文名，每部电影的平均演员人数，每部电影的总数量以及每部电影的英文和法文名称。

```
{
  ID as var(func: allofterms(name@en, "Steven")) @filter(has(director.film)) {
    director.film {
      num_actors as count(starring)
    }
    average as avg(val(num_actors))
  }

  films(func: uid(ID)) {
    director_id : uid
    english_name : name@en
    average_actors : val(average)
    num_films : count(director.film)

    films : director.film {
      name : name@en
      english_name : name@en
      french_name : name@fr
    }
  }
}
```


## 分页

分页允许只返回部分结果集，而不是返回整个结果集。这对于top-k风格的查询非常有用同时也减小结果集的大小对于 客户端处理或允许分页访问结果。for client side processing or to allow paged access to results.

分页通常用于排序。

**注意** *在没有指定排序顺序的情况下，结果按“uid”进行排序，uid是随机分配的。因此，虽然顺序是确定的，但可能不是您所期望的。*

### First

语法 例子:

* `q(func: ..., first: N)`
* `predicate (first: N) { ... }`
* `predicate @filter(...) (first: N) { ... }`

对于正数`N`， '`first: N`根据排序或UID顺序检索第一个`N`结果。

对于负数 `N`, `first: N` 根据排序或UID顺序检索最后一个`N`结果,目前,负数只有在排序后才支持。要通过排序实现负数查找的效果，请颠倒排序的顺序后使用正数`N`

查询示例 : 史蒂文斯皮尔伯格的导演和的前三种电影类型的最后两部电影，通过UID排序，按英文名称的字母顺序排序。

```
{
  me(func: allofterms(name@en, "Steven Spielberg")) {
    director.film (first: -2) {
      name@en
      initial_release_date
      genre (orderasc: name@en) (first: 3) {
          name@en
      }
    }
  }
}
```



查询示例 : 所有名字含有叫史蒂文的导演中执导过最多演员的前三位导演

```
{
  ID as var(func: allofterms(name@en, "Steven")) @filter(has(director.film)) {
    director.film {
      stars as count(starring)
    }
    totalActors as sum(val(stars))
  }

  mostStars(func: uid(ID), orderdesc: val(totalActors), first: 3) {
    name@en
    stars : val(totalActors)

    director.film {
      name@en
    }
  }
}
```

### Offset 偏移量

语法 例子:

* `q(func: ..., offset: N)`
* `predicate (offset: N) { ... }`
* `predicate (first: M, offset: N) { ... }`
* `predicate @filter(...) (offset: N) { ... }`

使用`offset: N`时，不会返回第一个 `N` 结果。组合使用`first: M, offset: N` ，从第`N`项后开始，返回`M`项数据,第`N`会被跳过。 

查询示例 : 徐克的电影所有按英文名排序,跳过前4条数据返回后续的6条数据。

```
{
  me(func: allofterms(name@en, "Hark Tsui")) {
    name@zh
    name@en
    director.film (orderasc: name@en) (first:6, offset:4)  {
      genre {
        name@en
      }
      name@zh
      name@en
      initial_release_date
    }
  }
}
```

### 在...之后 After 

语法 例子:

* `q(func: ..., after: UID)`
* `predicate (first: N, after: UID) { ... }`
* `predicate @filter(...) (first: N, after: UID) { ... }`

Another way to get results after skipping over some results is to use the default UID ordering and skip directly past a node specified by UID.  For example, a first query could be of the form `predicate (after: 0x0, first: N)`, or just `predicate (first: N)`, with subsequent queries of the form `predicate(after: <uid of last entity in last result>, first: N)`.


Query Example: The first five of Baz Luhrmann's films, sorted by UID order.

```
{
  me(func: allofterms(name@en, "Baz Luhrmann")) {
    name@en
    director.film (first:5) {
      uid
      name@en
    }
  }
}
```

The fifth movie is the Australian movie classic Strictly Ballroom.  It has UID `0x264ce8`.  The results after Strictly Ballroom can now be obtained with `after`.

```
{
  me(func: allofterms(name@en, "Baz Luhrmann")) {
    name@en
    director.film (first:5, after: 0x264ce8) {
      uid
      name@en
    }
  }
}
```


## Count

Syntax Examples:

* `count(predicate)`
* `count(uid)`

The form `count(predicate)` counts how many `predicate` edges lead out of a node.

The form `count(uid)` counts the number of UIDs matched in the enclosing block.

Query Example: The number of films acted in by each actor with `Orlando` in their name.

```
{
  me(func: allofterms(name@en, "Orlando")) @filter(has(actor.film)) {
    name@en
    count(actor.film)
  }
}
```

Count can be used at root and [aliased]({{< relref "#alias">}}).

Query Example: Count of directors who have directed more than five films.  When used at the query root, the [count index]({{< relref "#count-index">}}) is required.

```
{
  directors(func: gt(count(director.film), 5)) {
    totalDirectors : count(uid)
  }
}
```


Count can be assigned to a [value variable]({{< relref "#value-variables">}}).

Query Example: The actors of Ang Lee's "Eat Drink Man Woman" ordered by the number of movies acted in.

```
{
	var(func: allofterms(name@en, "eat drink man woman")) {
    starring {
      actors as performance.actor {
        totalRoles as count(actor.film)
      }
    }
  }

  edmw(func: uid(actors), orderdesc: val(totalRoles)) {
    name@en
    name@zh
    totalRoles : val(totalRoles)
  }
}
```


## Sorting

Syntax Examples:

* `q(func: ..., orderasc: predicate)`
* `q(func: ..., orderdesc: val(varName))`
* `predicate (orderdesc: predicate) { ... }`
* `predicate @filter(...) (orderasc: N) { ... }`
* `q(func: ..., orderasc: predicate1, orderdesc: predicate2)`

Sortable Types: `int`, `float`, `String`, `dateTime`, `id`, `default`

Results can be sorted in ascending, `orderasc` or decending `orderdesc` order by a predicate or variable.

For sorting on predicates with [sortable indices]({{< relref "#sortable-indices">}}), Dgraph sorts on the values and with the index in parallel and returns whichever result is computed first.

Sorted queries retrieve up to 1000 results by default. This can be changed with [first]({{< relref "#first">}}).


Query Example: French director Jean-Pierre Jeunet's movies sorted by release date.

```
{
  me(func: allofterms(name@en, "Jean-Pierre Jeunet")) {
    name@fr
    director.film(orderasc: initial_release_date) {
      name@fr
      name@en
      initial_release_date
    }
  }
}
```

Sorting can be performed at root and on value variables.

Query Example: All genres sorted alphabetically and the five movies in each genre with the most genres.

```
{
  genres as var(func: has(~genre)) {
    ~genre {
      numGenres as count(genre)
    }
  }

  genres(func: uid(genres), orderasc: name@en) {
    name@en
    ~genre (orderdesc: val(numGenres), first: 5) {
      name@en
    	genres : val(numGenres)
    }
  }
}
```

Sorting can also be performed by multiple predicates as shown below. If the values are equal for the
first predicate, then they are sorted by the second predicate and so on.

Query Example: Find all nodes which have type Person, sort them by their first_name and among those
that have the same first_name sort them by last_name in descending order.

```
{
  me(func: eq(type, "Person", orderasc: first_name, orderdesc: last_name)) {
    first_name
    last_name
  }
}
```

## Multiple Query Blocks

Inside a single query, multiple query blocks are allowed.  The result is all blocks with corresponding block names.

Multiple query blocks are executed in parallel.

The blocks need not be related in any way.

Query Example: All of Angelina Jolie's films, with genres, and Peter Jackson's films since 2008.

```
{
 AngelinaInfo(func:allofterms(name@en, "angelina jolie")) {
  name@en
   actor.film {
    performance.film {
      genre {
        name@en
      }
    }
   }
  }

 DirectorInfo(func: eq(name@en, "Peter Jackson")) {
    name@en
    director.film @filter(ge(initial_release_date, "2008"))  {
        Release_date: initial_release_date
        Name: name@en
    }
  }
}
```


If queries contain some overlap in answers, the result sets are still independent

Query Example: The movies Mackenzie Crook has acted in and the movies Jack Davenport has acted in.  The results sets overlap because both have acted in the Pirates of the Caribbean movies, but the results are independent and both contain the full answers sets.

```
{
  Mackenzie(func:allofterms(name@en, "Mackenzie Crook")) {
    name@en
    actor.film {
      performance.film {
        uid
        name@en
      }
      performance.character {
        name@en
      }
    }
  }

  Jack(func:allofterms(name@en, "Jack Davenport")) {
    name@en
    actor.film {
      performance.film {
        uid
        name@en
      }
      performance.character {
        name@en
      }
    }
  }
}
```


### Var Blocks

Var blocks start with the keyword `var` and are not returned in the query results.

Query Example: Angelina Jolie's movies ordered by genre.

```
{
  var(func:allofterms(name@en, "angelina jolie")) {
    name@en
    actor.film {
      A AS performance.film {
        B AS genre
      }
    }
  }

  films(func: uid(B), orderasc: name@en) {
    name@en
    ~genre @filter(uid(A)) {
      name@en
    }
  }
}
```


## Query Variables

Syntax Examples:

* `varName as q(func: ...) { ... }`
* `varName as var(func: ...) { ... }`
* `varName as predicate { ... }`
* `varName as predicate @filter(...) { ... }`

Types : `uid`

Nodes (UID's) matched at one place in a query can be stored in a variable and used elsewhere.  Query variables can be used in other query blocks or in a child node of the defining block.

Query variables do not affect the semantics of the query at the point of definition.  Query variables are evaluated to all nodes matched by the defining block.

In general, query blocks are executed in parallel, but variables impose an evaluation order on some blocks.  Cycles induced by variable dependence are not permitted.

If a variable is defined, it must be used elsewhere in the query.

A query variable is used by extracting the UIDs in it with `uid(var-name)`.

The syntax `func: uid(A,B)` or `@filter(uid(A,B))` means the union of UIDs for variables `A` and `B`.

Query Example: The movies of Angelia Jolie and Brad Pitt where both have acted on movies in the same genre.  Note that `B` and `D` match all genres for all movies, not genres per movie.
```
{
 var(func:allofterms(name@en, "angelina jolie")) {
   actor.film {
    A AS performance.film {  # All films acted in by Angelina Jolie
     B As genre  # Genres of all the films acted in by Angelina Jolie
    }
   }
  }

 var(func:allofterms(name@en, "brad pitt")) {
   actor.film {
    C AS performance.film {  # All films acted in by Brad Pitt
     D as genre  # Genres of all the films acted in by Brad Pitt
    }
   }
  }

 films(func: uid(D)) @filter(uid(B)) {   # Genres from both Angelina and Brad
  name@en
   ~genre @filter(uid(A, C)) {  # Movies in either A or C.
     name@en
   }
 }
}
```


## Value Variables

Syntax Examples:

* `varName as scalarPredicate`
* `varName as count(predicate)`
* `varName as avg(...)`
* `varName as math(...)`

Types : `int`, `float`, `String`, `dateTime`, `id`, `default`, `geo`, `bool`

Value variables store scalar values.  Value variables are a map from the UIDs of the enclosing block to the corresponding values.

It therefore only makes sense to use the values from a value variable in a context that matches the same UIDs - if used in a block matching different UIDs the value variable is undefined.

It is an error to define a value variable but not use it elsewhere in the query.

Value variables are used by extracting the values with `val(var-name)`, or by extracting the UIDs with `uid(var-name)`.

[Facet]({{< relref "#facets-edge-attributes">}}) values can be stored in value variables.

Query Example: The number of movie roles played by the actors of the 80's classic "The Princess Bride".  Query variable `pbActors` matches the UIDs of all actors from the movie.  Value variable `roles` is thus a map from actor UID to number of roles.  Value variable `roles` can be used in the the `totalRoles` query block because that query block also matches the `pbActors` UIDs, so the actor to number of roles map is available.

```
{
  var(func:allofterms(name@en, "The Princess Bride")) {
    starring {
      pbActors as performance.actor {
        roles as count(actor.film)
      }
    }
  }
  totalRoles(func: uid(pbActors), orderasc: val(roles)) {
    name@en
    numRoles : val(roles)
  }
}
```


Value variables can be used in place of UID variables by extracting the UID list from the map.

Query Example: The same query as the previous example, but using value variable `roles` for matching UIDs in the `totalRoles` query block.

```
{
  var(func:allofterms(name@en, "The Princess Bride")) {
    starring {
      performance.actor {
        roles as count(actor.film)
      }
    }
  }
  totalRoles(func: uid(roles), orderasc: val(roles)) {
    name@en
    numRoles : val(roles)
  }
}
```


### Variable Propagation

Like query variables, value variables can be used in other query blocks and in blocks nested within the defining block.  When used in a block nested within the block that defines the variable, the value is computed as a sum of the variable for parent nodes along all paths to the point of use.  This is called variable propagation.

For example:
```
{
  q(func: uid(0x01)) {
    myscore as math(1)          # A
    friends {                   # B
      friends {                 # C
        ...myscore...
      }
    }
  }
}
```
At line A, a value variable `myscore` is defined as mapping node with UID `0x01` to value 1.  At B, the value for each friend is still 1: there is only one path to each friend.  Traversing the friend edge twice reaches the friends of friends. The variable `myscore` gets propagated such that each friend of friend will receive the sum of its parents values:  if a friend of a friend is reachable from only one friend, the value is still 1, if they are reachable from two friends, the value is two and so on.  That is, the value of `myscore` for each friend of friends inside the block marked C will be the number of paths to them.

**The value that a node receives for a propagated variable is the sum of the values of all its parent nodes.**

This propagation is useful, for example, in normalizing a sum across users, finding the number of paths between nodes and accumulating a sum through a graph.



Query Example: For each Harry Potter movie, the number of roles played by actor Warwick Davis.
```
{
	num_roles(func: eq(name@en, "Warwick Davis")) @cascade @normalize {

    paths as math(1)  # records number of paths to each character

    actor : name@en

    actor.film {
      performance.film @filter(allofterms(name@en, "Harry Potter")) {
        film_name : name@en
        characters : math(paths)  # how many paths (i.e. characters) reach this film
      }
    }
  }
}
```


Query Example: Each actor who has been in a Peter Jackson movie and the fraction of Peter Jackson movies they have appeared in.
```
{
	movie_fraction(func:eq(name@en, "Peter Jackson")) @normalize {

    paths as math(1)
    total_films : num_films as count(director.film)
    director : name@en

    director.film {
      starring {
        performance.actor {
          fraction : math(paths / (num_films/paths))
          actor : name@en
        }
      }
    }
  }
}
```

More examples can be found in two Dgraph blog posts about using variable propagation for recommendation engines ([post 1](https://open.dgraph.io/post/recommendation/), [post 2](https://open.dgraph.io/post/recommendation2/)).

## Aggregation

Syntax Example: `AG(val(varName))`

For `AG` replaced with

* `min` : select the minimum value in the value variable `varName`
* `max` : select the maximum value
* `sum` : sum all values in value variable `varName`
* `avg` : calculate the average of values in `varName`

Schema Types:

| Aggregation   | Schema Types                                    |
| :------------ | :---------------------------------------------- |
| `min` / `max` | `int`, `float`, `string`, `dateTime`, `default` |
| `sum` / `avg` | `int`, `float`                                  |

Aggregation can only be applied to [value variables]({{< relref "#value-variables">}}).  An index is not required (the values have already been found and stored in the value variable mapping).

An aggregation is applied at the query block enclosing the variable definition.  As opposed to query variables and value variables, which are global, aggregation is computed locally.  For example:
```
A as predicateA {
  ...
  B as predicateB {
    x as ...some value...
  }
  min(val(x))
}
```
Here, `A` and `B` are the lists of all UIDs that match these blocks.  Value variable `x` is a mapping from UIDs in `B` to values.  The aggregation `min(val(x))`, however, is computed for each UID in `A`.  That is, it has a semantics of: for each UID in `A`, take the slice of `x` that corresponds to `A`'s outgoing `predicateB` edges and compute the aggregation for those values.

Aggregations can themselves be assigned to value variables, making a UID to aggregation map.


### Min

#### Usage at Root

Query Example: Get the min initial release date for any Harry Potter movie.

The release date is assigned to a variable, then it is aggregated and fetched in an empty block.
```
{
  var(func: allofterms(name@en, "Harry Potter")) {
    d as initial_release_date
  }
  me() {
    min(val(d))
  }
}
```

#### Usage at other levels.

Query Example:  Directors called Steven and the date of release of their first movie, in ascending order of first movie.

```
{
  stevens as var(func: allofterms(name@en, "steven")) {
    director.film {
      ird as initial_release_date
      # ird is a value variable mapping a film UID to its release date
    }
    minIRD as min(val(ird))
    # minIRD is a value variable mapping a director UID to their first release date
  }

  byIRD(func: uid(stevens), orderasc: val(minIRD)) {
    name@en
    firstRelease: val(minIRD)
  }
}
```

### Max

#### Usage at Root

Query Example: Get the max initial release date for any Harry Potter movie.

The release date is assigned to a variable, then it is aggregated and fetched in an empty block.
```
{
  var(func: allofterms(name@en, "Harry Potter")) {
    d as initial_release_date
  }
  me() {
    max(val(d))
  }
}
```

#### Usage at other levels.

Query Example: Quentin Tarantino's movies and date of release of the most recent movie.

```
{
  director(func: allofterms(name@en, "Quentin Tarantino")) {
    director.film {
      name@en
      x as initial_release_date
    }
    max(val(x))
  }
}
```

### Sum and Avg

#### Usage at Root

Query Example: Get the sum and average of number of count of movies directed by people who have
Steven or Tom in their name.

```
{
  var(func: anyofterms(name@en, "Steven Tom")) {
    a as count(director.film)
  }

  me() {
    avg(val(a))
    sum(val(a))
  }
}
```

#### Usage at other levels.

Query Example: Steven Spielberg's movies, with the number of recorded genres per movie, and the total number of genres and average genres per movie.

```
{
  director(func: eq(name@en, "Steven Spielberg")) {
    name@en
    director.film {
      name@en
      numGenres : g as count(genre)
    }
    totalGenres : sum(val(g))
    genresPerMovie : avg(val(g))
  }
}
```


### Aggregating Aggregates

Aggregations can be assigned to value variables, and so these variables can in turn be aggregated.

Query Example: For each actor in a Peter Jackson film, find the number of roles played in any movie.  Sum these to find the total number of roles ever played by all actors in the movie.  Then sum the lot to find the total number of roles ever played by actors who have appeared in Peter Jackson movies.  Note that this demonstrates how to aggregate aggregates; the answer in this case isn't quite precise though, because actors that have appeared in multiple Peter Jackson movies are counted more than once.

```
{
  PJ as var(func:allofterms(name@en, "Peter Jackson")) {
    director.film {
      starring {  # starring an actor
        performance.actor {
          movies as count(actor.film)
          # number of roles for this actor
        }
        perf_total as sum(val(movies))
      }
      movie_total as sum(val(perf_total))
      # total roles for all actors in this movie
    }
    gt as sum(val(movie_total))
  }

  PJmovies(func: uid(PJ)) {
    name@en
  	director.film (orderdesc: val(movie_total), first: 5) {
    	name@en
    	totalRoles : val(movie_total)
  	}
    grandTotal : val(gt)
  }
}
```


## Math on value variables

Value variables can be combined using mathematical functions.  For example, this could be used to associate a score which is then be used to order or perform other operations, such as might be used in building newsfeeds, simple recommendation systems and the likes.

Math statements must be enclosed within `math( <exp> )` and must be stored to a value variable.

The supported operators are as follows:

|            Operators             |                   Types accepted                   |                         What it does                         |
| :------------------------------: | :------------------------------------------------: | :----------------------------------------------------------: |
|       `+` `-` `*` `/` `%`        |                   `int`, `float`                   |             performs the corresponding operation             |
|           `min` `max`            | All types except `geo`, `bool`  (binary functions) |           selects the min/max value among the two            |
|   `<` `>` `<=` `>=` `==` `!=`    |           All types except `geo`, `bool`           |          Returns true or false based on the values           |
| `floor` `ceil` `ln` `exp` `sqrt` |          `int`, `float` (unary function)           |             performs the corresponding operation             |
|             `since`              |                     `dateTime`                     | Returns the number of seconds in float from the time specified |
|           `pow(a, b)`            |                   `int`, `float`                   |                  Returns `a to the power b`                  |
|          `logbase(a,b)`          |                   `int`, `float`                   |               Returns `log(a)` to the base `b`               |
|         `cond(a, b, c)`          |          first operand must be a boolean           |             selects `b` if `a` is true else `c`              |


Query Example:  Form a score for each of Steven Spielberg's movies as the sum of number of actors, number of genres and number of countries.  List the top five such movies in order of decreasing score.

```
{
	var(func:allofterms(name@en, "steven spielberg")) {
		films as director.film {
			p as count(starring)
			q as count(genre)
			r as count(country)
			score as math(p + q + r)
		}
	}

	TopMovies(func: uid(films), orderdesc: val(score), first: 5){
		name@en
		val(score)
	}
}
```

Value variables and aggregations of them can be used in filters.

Query Example: Calculate a score for each Steven Spielberg movie with a condition on release date to penalize movies that are more than 10 years old, filtering on the resulting score.

```
{
  var(func:allofterms(name@en, "steven spielberg")) {
    films as director.film {
      p as count(starring)
      q as count(genre)
      date as initial_release_date
      years as math(since(date)/(365*24*60*60))
      score as math(cond(years > 10, 0, ln(p)+q-ln(years)))
    }
  }

  TopMovies(func: uid(films), orderdesc: val(score)) @filter(gt(val(score), 2)){
    name@en
    val(score)
    val(date)
  }
}
```


Values calculated with math operations are stored to value variables and so can be aggreated.

Query Example: Compute a score for each Steven Spielberg movie and then aggregate the score.

```
{
	steven as var(func:eq(name@en, "Steven Spielberg")) @filter(has(director.film)) {
		director.film {
			p as count(starring)
			q as count(genre)
			r as count(country)
			score as math(p + q + r)
		}
		directorScore as sum(val(score))
	}

	score(func: uid(steven)){
		name@en
		val(directorScore)
	}
}
```


## GroupBy

Syntax Examples:

* `q(func: ...) @groupby(predicate) { min(...) }`
* `predicate @groupby(pred) { count(uid) }``


A `groupby` query aggregates query results given a set of properties on which to group elements.  For example, a query containing the block `friend @groupby(age) { count(uid) }`, finds all nodes reachable along the friend edge, partitions these into groups based on age, then counts how many nodes are in each group.  The returned result is the grouped edges and the aggregations.

Inside a `groupby` block, only aggregations are allowed and `count` may only be applied to `uid`.

If the `groupby` is applied to a `uid` predicate, the resulting aggregations can be saved in a variable (mapping the grouped UIDs to aggregate values) and used elsewhere in the query to extract information other than the grouped or aggregated edges.

Query Example: For Steven Spielberg movies, count the number of movies in each genre and for each of those genres return the genre name and the count.  The name can't be extracted in the `groupby` because it is not an aggregate, but `uid(a)` can be used to extract the UIDs from the UID to value map and thus organize the `byGenre` query by genre UID.


```
{
  var(func:allofterms(name@en, "steven spielberg")) {
    director.film @groupby(genre) {
      a as count(uid)
      # a is a genre UID to count value variable
    }
  }

  byGenre(func: uid(a), orderdesc: val(a)) {
    name@en
    total_movies : val(a)
  }
}
```

Query Example: Actors from Tim Burton movies and how many roles they have played in Tim Burton movies.
```
{
  var(func:allofterms(name@en, "Tim Burton")) {
    director.film {
      starring @groupby(performance.actor) {
        a as count(uid)
        # a is an actor UID to count value variable
      }
    }
  }

  byActor(func: uid(a), orderdesc: val(a)) {
    name@en
    val(a)
  }
}
```



## Expand Predicates

Keyword `_predicate_` retrieves all predicates out of nodes at the level used.

Query Example: All predicates from actor Geoffrey Rush.
```
{
  director(func: eq(name@en, "Geoffrey Rush")) {
    _predicate_
  }
}
```

The number of predicates from a node can be counted and be aliased.

Query Example: All predicates from actor Geoffrey Rush and the count of such predicates.
```
{
  director(func: eq(name@en, "Geoffrey Rush")) {
    num_predicates: count(_predicate_)
    my_predicates: _predicate_
  }
}
```

Predicates can be stored in a variable and passed to `expand()` to expand all the predicates in the variable.

If `_all_` is passed as an argument to `expand()`, all the predicates at that level are retrieved. More levels can be specfied in a nested fashion under `expand()`.
If `_forward_` is passed as an argument to `expand()`, all predicates at that level (minus any reverse predicates) are retrieved.
If `_reverse_` is passed as an argument to `expand()`, only the reverse predicates are retrieved.

Query Example: Predicates saved to a variable and queried with `expand()`.
```
{
  var(func: eq(name@en, "Lost in Translation")) {
    pred as _predicate_
    # expand(_all_) { expand(_all_)}
  }

  director(func: eq(name@en, "Lost in Translation")) {
    name@.
    expand(val(pred)) {
      expand(_all_)
    }
  }
}
```

`_predicate_` returns string valued predicates as a name without language tag.  If the predicate has no string without a language tag, `expand()` won't expand it (see [language preference]({{< relref "#language-support" >}})).  For example, above `name` generally doesn't have strings without tags in the dataset, so `name@.` is required.

## Cascade Directive

With the `@cascade` directive, nodes that don't have all predicates specified in the query are removed. This can be useful in cases where some filter was applied or if nodes might not have all listed predicates.


Query Example: Harry Potter movies, with each actor and characters played.  With `@cascade`, any character not played by an actor called Warwick is removed, as is any Harry Potter movie without any actors called Warwick.  Without `@cascade`, every character is returned, but only those played by actors called Warwick also have the actor name.
```
{
  HP(func: allofterms(name@en, "Harry Potter")) @cascade {
    name@en
    starring{
        performance.character {
          name@en
        }
        performance.actor @filter(allofterms(name@en, "Warwick")){
            name@en
         }
    }
  }
}
```

## Normalize directive

With the `@normalize` directive, only aliased predicates are returned and the result is flattened to remove nesting.

Query Example: Film name, country and first two actors (by UID order) of every Steven Spielberg movie, without `initial_release_date` because no alias is given and flattened by `@normalize`
```
{
  director(func:allofterms(name@en, "steven spielberg")) @normalize {
    director: name@en
    director.film {
      film: name@en
      initial_release_date
      starring(first: 2) {
        performance.actor {
          actor: name@en
        }
        performance.character {
          character: name@en
        }
      }
      country {
        country: name@en
      }
    }
  }
}
```


## Ignorereflex directive

The `@ignorereflex` directive forces the removal of child nodes that are reachable from themselves as a parent, through any path in the query result

Query Example: All the coactors of Rutger Hauer.  Without `@ignorereflex`, the result would also include Rutger Hauer for every movie.

```
{
  coactors(func: eq(name@en, "Rutger Hauer")) @ignorereflex {
    actor.film {
      performance.film {
        starring {
          performance.actor {
            name@en
          }
        }
      }
    }
  }
}
```

## Debug

For the purposes of debugging, you can attach a query parameter `debug=true` to a query. Attaching this parameter lets you retrieve the `uid` attribute for all the entities along with the `server_latency` information.

Query with debug as a query parameter
```
curl "http://localhost:8080/query?debug=true" -XPOST -d $'{
  tbl(func: allofterms(name@en, "The Big Lebowski")) {
    name@en
  }
}' | python -m json.tool | less
```

Returns `uid` and `server_latency`
```
{
  "data": {
    "tbl": [
      {
        "uid": "0x41434",
        "name@en": "The Big Lebowski"
      },
      {
        "uid": "0x145834",
        "name@en": "The Big Lebowski 2"
      },
      {
        "uid": "0x2c8a40",
        "name@en": "Jeffrey \"The Big\" Lebowski"
      },
      {
        "uid": "0x3454c4",
        "name@en": "The Big Lebowski"
      }
    ],
    "server_latency": {
      "parsing": "101µs",
      "processing": "802ms",
      "json": "115µs",
      "total": "802ms"
    }
  }
}
```


## Schema

For each predicate, the schema specifies the target's type.  If a predicate `p` has type `T`, then for all subject-predicate-object triples `s p o` the object `o` is of schema type `T`.

* On mutations, scalar types are checked and an error thrown if the value cannot be converted to the schema type.

* On query, value results are returned according to the schema type of the predicate.

If a schema type isn't specified before a mutation adds triples for a predicate, then the type is inferred from the first mutation.  This type is either:

* type `uid`, if the first mutation for the predicate has nodes for the subject and object, or

* derived from the [rdf type]({{< relref "#rdf-types" >}}), if the object is a literal and an rdf type is present in the first mutation, or

* `default` type, otherwise.


### Schema Types

Dgraph supports scalar types and the UID type.

#### Scalar Types

For all triples with a predicate of scalar types the object is a literal.

| Dgraph Type | Go type                                                      |
| ----------- | :----------------------------------------------------------- |
| `default`   | string                                                       |
| `int`       | int64                                                        |
| `float`     | float                                                        |
| `string`    | string                                                       |
| `bool`      | bool                                                         |
| `dateTime`  | time.Time (RFC3339 format [Optional timezone] eg: 2006-01-02T15:04:05.999999999+10:00 or 2006-01-02T15:04:05.999999999) |
| `geo`       | [go-geom](https://github.com/twpayne/go-geom)                |
| `password`  | string (encrypted)                                           |


{{% notice "note" %}}Dgraph supports date and time formats for `dateTime` scalar type only if they
are RFC 3339 compatible which is different from ISO 8601(as defined in the RDF spec). You should
convert your values to RFC 3339 format before sending them to Dgraph.{{% /notice  %}}

#### UID Type

The `uid` type denotes a node-node edge; internally each node is represented as a `uint64` id.

| Dgraph Type | Go type |
| ----------- | :------ |
| `uid`       | uint64  |


### Adding or Modifying Schema

Schema mutations add or modify schema.

Multiple scalar values can also be added for a `S P` by specifying the schema to be of
list type. Occupations in the example below can store a list of strings for each `S P`.

An index is specified with `@index`, with arguments to specify the tokenizer. When specifying an
index for a predicate it is mandatory to specify the type of the index. For example:

```
name: string @index(exact, fulltext) @count .
multiname: string @lang .
age: int @index(int) .
friend: uid @count .
dob: dateTime .
location: geo @index(geo) .
occupations: [string] @index(term) .
```

If no data has been stored for the predicates, a schema mutation sets up an empty schema ready to receive triples.

If data is already stored before the mutation, existing values are not checked to conform to the new schema.  On query, Dgraph tries to convert existing values to the new schema types, ignoring any that fail conversion.

If data exists and new indices are specified in a schema mutation, any index not in the updated list is dropped and a new index is created for every new tokenizer specified.

Reverse edges are also computed if specified by a schema mutation.

{{% notice "note" %}} If your predicate is a URI or has special characters, then you should wrap
it with angular brackets while doing the schema mutation. E.g. `<first:name>`{{% /notice %}}


### Upsert directive

Predicates can specify the `@upsert` directive if you want to do upsert operations against it.
If the `@upsert` directive is specified then the index key for the predicate would be checked for
conflict while committing a transaction, which would allow upserts.

This is how you specify the upsert directive for a predicate. This replaces the `IgnoreIndexConflict`
field which was part of the mutation object in previous releases.
```
email: string @index(exact) @upsert .
```

### RDF Types

Dgraph supports a number of [RDF types in mutations]({{< relref "mutations/index.md#language-and-rdf-types" >}}).

As well as implying a schema type for a [first mutation]({{< relref "#schema" >}}), an RDF type can override a schema type for storage.

If a predicate has a schema type and a mutation has an RDF type with a different underlying Dgraph type, the convertibility to schema type is checked, and an error is thrown if they are incompatible, but the value is stored in the RDF type's corresponding Dgraph type.  Query results are always returned in schema type.

For example, if no schema is set for the `age` predicate.  Given the mutation
```
{
 set {
  _:a <age> "15"^^<xs:int> .
  _:b <age> "13" .
  _:c <age> "14"^^<xs:string> .
  _:d <age> "14.5"^^<xs:string> .
  _:e <age> "14.5" .
 }
}
```
Dgraph:

* sets the schema type to `int`, as implied by the first triple,
* converts `"13"` to `int` on storage,
* checks `"14"` can be converted to `int`, but stores as `string`,
* throws an error for the remaining two triples, because `"14.5"` can't be converted to `int`.

### Extended Types

The following types are also accepted.

#### Password type

A password for an entity is set with setting the schema for the attribute to be of type `password`.  Passwords cannot be queried directly, only checked for a match using the `checkpwd` function.

For example: to set a password, first set schema, then the password:
```
pass: password .
```

```
{
  set {
    <0x123> <name> "Password Example"
    <0x123> <pass> "ThePassword" .
  }
}
```

to check a password:
```
{
  check(func: uid(0x123)) {
    name
    checkpwd(pass, "ThePassword")
  }
}
```

output:
```
{
  "check": [
    {
      "name": "Password Example",
      "pass": [
        {
          "checkpwd": true
        }
      ]
    }
  ]
}
```

### Indexing

{{% notice "note" %}}Filtering on a predicate by applying a [function]({{< relref "#functions" >}}) requires an index.{{% /notice %}}

When filtering by applying a function, Dgraph uses the index to make the search through a potentially large dataset efficient.

All scalar types can be indexed.

Types `int`, `float`, `bool` and `geo` have only a default index each: with tokenizers named `int`, `float`, `bool` and `geo`.

Types `string` and `dateTime` have a number of indices.

#### String Indices
The indices available for strings are as follows.

| Dgraph function            | Required index / tokenizer             | Notes                                                        |
| :------------------------- | :------------------------------------- | :----------------------------------------------------------- |
| `eq`                       | `hash`, `exact`, `term`, or `fulltext` | The most performant index for `eq` is `hash`. Only use `term` or `fulltext` if you also require term or full text search. If you're already using `term`, there is no need to use `hash` or `exact` as well. |
| `le`, `ge`, `lt`, `gt`     | `exact`                                | Allows faster sorting.                                       |
| `allofterms`, `anyofterms` | `term`                                 | Allows searching by a term in a sentence.                    |
| `alloftext`, `anyoftext`   | `fulltext`                             | Matching with language specific stemming and stopwords.      |
| `regexp`                   | `trigram`                              | Regular expression matching. Can also be used for equality checking. |

{{% notice "warning" %}}
Incorrect index choice can impose performance penalties and an increased
transaction conflict rate. Use only the minimum number of and simplest indexes
that your application needs.
{{% /notice %}}


#### DateTime Indices

The indices available for `dateTime` are as follows.

| Index name / Tokenizer | Part of date indexed               |
| :--------------------- | :--------------------------------- |
| `year`                 | index on year (default)            |
| `month`                | index on year and month            |
| `day`                  | index on year, month and day       |
| `hour`                 | index on year, month, day and hour |

The choices of `dateTime` index allow selecting the precision of the index.  Applications, such as the movies examples in these docs, that require searching over dates but have relatively few nodes per year may prefer the `year` tokenizer; applications that are dependent on fine grained date searches, such as real-time sensor readings, may prefer the `hour` index.


All the `dateTime` indices are sortable.


#### Sortable Indices

Not all the indices establish a total order among the values that they index. Sortable indices allow inequality functions and sorting.

* Indexes `int` and `float` are sortable.
* `string` index `exact` is sortable.
* All `dateTime` indices are sortable.

For example, given an edge `name` of `string` type, to sort by `name` or perform inequality filtering on names, the `exact` index must have been specified.  In which case a schema query would return at least the following tokenizers.

```
{
  "predicate": "name",
  "type": "string",
  "index": true,
  "tokenizer": [
    "exact"
  ]
}
```

#### Count index

For predicates with the `@count` Dgraph indexes the number of edges out of each node.  This enables fast queries of the form:
```
{
  q(func: gt(count(pred), threshold)) {
    ...
  }
}
```

### List Type

Predicate with scalar types can also store a list of values if specified in the schema. The scalar
type needs to be enclosed within `[]` to indicate that its a list type. These lists are like an
unordered set.

```
occupations: [string] .
score: [int] .
```

* A set operation adds to the list of values. The order of the stored values is non-deterministic.
* A delete operation deletes the value from the list.
* Querying for these predicates would return the list in an array.
* Indexes can be applied on predicates which have a list type and you can use [Functions]({{<ref
  "#functions">}}) on them.
* Sorting is not allowed using these predicates.


### Reverse Edges

A graph edge is unidirectional. For node-node edges, sometimes modeling requires reverse edges.  If only some subject-predicate-object triples have a reverse, these must be manually added.  But if a predicate always has a reverse, Dgraph computes the reverse edges if `@reverse` is specified in the schema.

The reverse edge of `anEdge` is `~anEdge`.

For existing data, Dgraph computes all reverse edges.  For data added after the schema mutation, Dgraph computes and stores the reverse edge for each added triple.

### Querying Schema

A schema query queries for the whole schema:

```
schema {}
```

{{% notice "note" %}}
Unlike regular queries, the schema query is not surrounded by curly braces.
{{% /notice %}}

You can query for particular schema fields in the query body.

```
schema {
  type
  index
  reverse
  tokenizer
  list
  count
  upsert
  lang
}
```

You can also query for particular predicates:

```
schema(pred: [name, friend]) {
  type
  index
  reverse
  tokenizer
  list
  count
  upsert
  lang
}
```

## Facets : Edge attributes

Dgraph supports facets --- **key value pairs on edges** --- as an extension to RDF triples. That is, facets add properties to edges, rather than to nodes.
For example, a `friend` edge between two nodes may have a boolean property of `close` friendship.
Facets can also be used as `weights` for edges.

Though you may find yourself leaning towards facets many times, they should not be misused.  It wouldn't be correct modeling to give the `friend` edge a facet `date_of_birth`. That should be an edge for the friend.  However, a facet like `start_of_friendship` might be appropriate.  Facets are however not first class citizen in Dgraph like predicates.

Facet keys are strings and values can be `string`, `bool`, `int`, `float` and `dateTime`.
For `int` and `float`, only decimal integers upto 32 signed bits, and 64 bit float values are accepted respectively.

The following mutation is used throughout this section on facets.  The mutation adds data for some peoples and, for example, records a `since` facet in `mobile` and `car` to record when Alice bought the car and started using the mobile number.

First we add some schema.
```sh
curl localhost:8080/alter -XPOST -d $'
    name: string @index(exact, term) .
    rated: uid @reverse @count .
' | python -m json.tool | less

```

```sh
curl localhost:8080/mutate -H "X-Dgraph-CommitNow: true" -XPOST -d $'
{
  set {

    # -- Facets on scalar predicates
    _:alice <name> "Alice" .
    _:alice <mobile> "040123456" (since=2006-01-02T15:04:05) .
    _:alice <car> "MA0123" (since=2006-02-02T13:01:09, first=true) .

    _:bob <name> "Bob" .
    _:bob <car> "MA0134" (since=2006-02-02T13:01:09) .

    _:charlie <name> "Charlie" .
    _:dave <name> "Dave" .


    # -- Facets on UID predicates
    _:alice <friend> _:bob (close=true, relative=false) .
    _:alice <friend> _:charlie (close=false, relative=true) .
    _:alice <friend> _:dave (close=true, relative=true) .


    # -- Facets for variable propagation
    _:movie1 <name> "Movie 1" .
    _:movie2 <name> "Movie 2" .
    _:movie3 <name> "Movie 3" .

    _:alice <rated> _:movie1 (rating=3) .
    _:alice <rated> _:movie2 (rating=2) .
    _:alice <rated> _:movie3 (rating=5) .

    _:bob <rated> _:movie1 (rating=5) .
    _:bob <rated> _:movie2 (rating=5) .
    _:bob <rated> _:movie3 (rating=5) .

    _:charlie <rated> _:movie1 (rating=2) .
    _:charlie <rated> _:movie2 (rating=5) .
    _:charlie <rated> _:movie3 (rating=1) .
  }
}' | python -m json.tool | less
```

### Facets on scalar predicates


Querying `name`, `mobile` and `car` of Alice gives the same result as without facets.

```
{
  data(func: eq(name, "Alice")) {
     name
     mobile
     car
  }
}
```


The syntax `@facets(facet-name)` is used to query facet data. For Alice the `since` facet for `mobile` and `car` are queried as follows.

```
{
  data(func: eq(name, "Alice")) {
     name
     mobile @facets(since)
     car @facets(since)
  }
}
```


Facets are retuned at the same level as the corresponding edge and have keys like edge|facet.

All facets on an edge are queried with `@facets`.

```
{
  data(func: eq(name, "Alice")) {
     name
     mobile @facets
     car @facets
  }
}
```


### Alias with facets

Alias can be specified while requesting specific predicates. Syntax is similar to how would request
alias for other predicates. `orderasc` and `orderdesc` are not allowed as alias as they have special
meaning. Apart from that anything else can be set as alias.

Here we set `car_since`, `close_friend` alias for `since`, `close` facets respectively.
```
{
   data(func: eq(name, "Alice")) {
     name
     mobile
     car @facets(car_since: since)
     friend @facets(close_friend: close) {
       name
     }
   }
}
```



### Facets on UID predicates

Facets on UID edges work similarly to facets on value edges.

For example, `friend` is an edge with facet `close`.
It was set to true for friendship between Alice and Bob
and false for friendship between Alice and Charlie.

A query for friends of Alice.

```
{
  data(func: eq(name, "Alice")) {
    name
    friend {
      name
    }
  }
}
```

A query for friends and the facet `close` with `@facets(close)`.

```
{
   data(func: eq(name, "Alice")) {
     name
     friend @facets(close) {
       name
     }
   }
}
```


For uid edges like `friend`, facets go to the corresponding child under the key edge|facet. In the above
example you can see that the `close` facet on the edge between Alice and Bob appears with the key `friend|close`
along with Bob's results.

```
{
  data(func: eq(name, "Alice")) {
    name
    friend @facets {
      name
      car @facets
    }
  }
}
```

Bob has a `car` and it has a facet `since`, which, in the results, is part of the same object as Bob
under the key car|since.
Also, the `close` relationship between Bob and Alice is part of Bob's output object.
Charlie does not have `car` edge and thus only UID facets.

### Filtering on facets

Dgraph supports filtering edges based on facets.
Filtering works similarly to how it works on edges without facets and has the same available functions.


Find Alice's close friends
```
{
  data(func: eq(name, "Alice")) {
    friend @facets(eq(close, true)) {
      name
    }
  }
}
```


To return facets as well as filter, add another `@facets(<facetname>)` to the query.

```
{
  data(func: eq(name, "Alice")) {
    friend @facets(eq(close, true)) @facets(relative) { # filter close friends and give relative status
      name
    }
  }
}
```


Facet queries can be composed with `AND`, `OR` and `NOT`.

```
{
  data(func: eq(name, "Alice")) {
    friend @facets(eq(close, true) AND eq(relative, true)) @facets(relative) { # filter close friends in my relation
      name
    }
  }
}
```


### Sorting using facets

Sorting is possible for a facet on a uid edge. Here we sort the movies rated by Alice, Bob and
Charlie by their `rating` which is a facet.

```
{
  me(func: anyofterms(name, "Alice Bob Charlie")) {
    name
    rated @facets(orderdesc: rating) {
      name
    }
  }
}
```



### Assigning Facet values to a variable

Facets on UID edges can be stored in [value variables]({{< relref "#value-variables" >}}).  The variable is a map from the edge target to the facet value.

Alice's friends reported by variables for `close` and `relative`.
```
{
  var(func: eq(name, "Alice")) {
    friend @facets(a as close, b as relative)
  }

  friend(func: uid(a)) {
    name
    val(a)
  }

  relative(func: uid(b)) {
    name
    val(b)
  }
}
```


### Facets and Variable Propagation

Facet values of `int` and `float` can be assigned to variables and thus the [values propagate]({{< relref "#variable-propagation" >}}).


Alice, Bob and Charlie each rated every movie.  A value variable on facet `rating` maps movies to ratings.  A query that reaches a movie through multiple paths sums the ratings on each path.  The following sums Alice, Bob and Charlie's ratings for the three movies.

```
{
  var(func: anyofterms(name, "Alice Bob Charlie")) {
    num_raters as math(1)
    rated @facets(r as rating) {
      total_rating as math(r) # sum of the 3 ratings
      average_rating as math(total_rating / num_raters)
    }
  }
  data(func: uid(total_rating)) {
    name
    val(total_rating)
    val(average_rating)
  }

}
```



### Facets and Aggregation

Facet values assigned to value variables can be aggregated.

```
{
  data(func: eq(name, "Alice")) {
    name
    rated @facets(r as rating) {
      name
    }
    avg(val(r))
  }
}
```


Note though that `r` is a map from movies to the sum of ratings on edges in the query reaching the movie.  Hence, the following does not correctly calculate the average ratings for Alice and Bob individually --- it calculates 2 times the average of both Alice and Bob's ratings.

```

{
  data(func: anyofterms(name, "Alice Bob")) {
    name
    rated @facets(r as rating) {
      name
    }
    avg(val(r))
  }
}
```


Calculating the average ratings of users requires a variable that maps users to the sum of their ratings.

```

{
  var(func: has(rated)) {
    num_rated as math(1)
    rated @facets(r as rating) {
      avg_rating as math(r / num_rated)
    }
  }

  data(func: uid(avg_rating)) {
    name
    val(avg_rating)
  }
}
```


## K-Shortest Path Queries

The shortest path between a source (`from`) node and destination (`to`) node can be found using the keyword `shortest` for the query block name. It requires the source node UID, destination node UID and the predicates (atleast one) that have to be considered for traversal. A `shortest` query block does not return any results and requires the path has to be stored in a variable which is used in other query blocks.

By default the shortest path is returned. With `numpaths: k`, the k-shortest paths are returned. With `depth: n`, the shortest paths up to `n` hops away are returned.

{{% notice "note" %}}
- If no predicates are specified in the `shortest` block, no path can be fetched as no edge is traversed.
- If you're seeing queries take a long time, you can set a [gRPC deadline](https://grpc.io/blog/deadlines) to stop the query after a certain amount of time.
{{% /notice %}}

For example:
```sh
curl localhost:8080/alter -XPOST -d $'
    name: string @index(exact) .
' | python -m json.tool | less
```

```sh
curl localhost:8080/mutate -H "X-Dgraph-CommitNow: true" -XPOST -d $'
{
  set {
    _:a <friend> _:b (weight=0.1) .
    _:b <friend> _:c (weight=0.2) .
    _:c <friend> _:d (weight=0.3) .
    _:a <friend> _:d (weight=1) .
    _:a <name> "Alice" .
    _:b <name> "Bob" .
    _:c <name> "Tom" .
    _:d <name> "Mallory" .
  }
}' | python -m json.tool | less
```

The shortest path between Alice and Mallory (assuming UIDs 0x2 and 0x5 respectively) can be found with query:
```
curl localhost:8080/query -XPOST -d $'{
 path as shortest(from: 0x2, to: 0x5) {
  friend
 }
 path(func: uid(path)) {
   name
 }
}' | python -m json.tool | less
```

Which returns the following results. (Note, without considering the `weight` facet, each edges' weight is considered as 1)
```
{
  "data": {
    "path": [
      {
        "name": "Alice"
      },
      {
        "name": "Mallory"
      }
    ],
    "_path_": [
      {
        "uid": "0x2",
        "friend": [
          {
            "uid": "0x5"
          }
        ]
      }
    ]
  }
}
```

The shortest two paths are returned with:
```
curl localhost:8080/query -XPOST -d $'{
 path as shortest(from: 0x2, to: 0x5, numpaths: 2) {
  friend
 }
 path(func: uid(path)) {
   name
 }
}' | python -m json.tool | less
```



Edges weights are included by using facets on the edges as follows.

{{% notice "note" %}}One facet per predicate in the shortest query block is allowed.{{% /notice %}}
```
curl localhost:8080/query -XPOST -d $'{
 path as shortest(from: 0x2, to: 0x5) {
  friend @facets(weight)
 }

 path(func: uid(path)) {
  name
 }
}' | python -m json.tool | less
```



```
{
  "data": {
    "path": [
      {
        "name": "Alice"
      },
      {
        "name": "Bob"
      },
      {
        "name": "Tom"
      },
      {
        "name": "Mallory"
      }
    ],
    "_path_": [
      {
        "uid": "0x2",
        "friend": [
          {
            "uid": "0x3",
            "friend|weight": 0.1,
            "friend": [
              {
                "uid": "0x4",
                "friend|weight": 0.2,
                "friend": [
                  {
                    "uid": "0x5",
                    "friend|weight": 0.3
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

Constraints can be applied to the intermediate nodes as follows.
```
curl localhost:8080/query -XPOST -d $'{
  path as shortest(from: 0x2, to: 0x5) {
    friend @filter(not eq(name, "Bob")) @facets(weight)
    relative @facets(liking)
  }

  relationship(func: uid(path)) {
    name
  }
}' | python -m json.tool | less
```


## Recurse Query

`Recurse` queries let you traverse a set of predicates (with filter, facets, etc.) until we reach all leaf nodes or we reach the maximum depth which is specified by the `depth` parameter.

To get 10 movies from a genre that has more than 30000 films and then get two actors for those movies we'd do something as follows:
```
{
	me(func: gt(count(~genre), 30000), first: 1) @recurse(depth: 5, loop: true) {
		name@en
		~genre (first:10) @filter(gt(count(starring), 2))
		starring (first: 2)
		performance.actor
	}
}
```
Some points to keep in mind while using recurse queries are:

- You can specify only one level of predicates after root. These would be traversed recursively. Both scalar and entity-nodes are treated similarly.
- Only one recurse block is advised per query.
- Be careful as the result size could explode quickly and an error would be returned if the result set gets too large. In such cases use more filters, limit results using pagination, or provide a depth parameter at root as shown in the example above.
- Loop parameter can be set to false, in which case paths which lead to a loops would be ignored
  while traversing.


## Fragments

`fragment` keyword allows you to define new fragments that can be referenced in a query, as per [GraphQL specification](https://facebook.github.io/graphql/#sec-Language.Fragments). The point is that if there are multiple parts which query the same set of fields, you can define a fragment and refer to it multiple times instead. Fragments can be nested inside fragments, but no cycles are allowed. Here is one contrived example.

```
curl localhost:8080/query -XPOST -d $'
query {
  debug(func: uid(1)) {
    name@en
    ...TestFrag
  }
}
fragment TestFrag {
  initial_release_date
  ...TestFragB
}
fragment TestFragB {
  country
}' | python -m json.tool | less
```

## GraphQL Variables

`Variables` can be defined and used in queries which helps in query reuse and avoids costly string building in clients at runtime by passing a separate variable map. A variable starts with a `$` symbol.

```
query test($a: int, $b: int, $name: string) {
  me(func: allofterms(name@en, $name)) {
    name@en
    director.film (first: $a, offset: $b) {
      name @en
      genre(first: $a) {
        name@en
      }
    }
  }
}
```

* Variables can have default values. In the example below, `$a` has a default value of `2`. Since the value for `$a` isn't provided in the variable map, `$a` takes on the default value.
* Variables whose type is suffixed with a `!` can't have a default value but must have a value as part of the variables map.
* The value of the variable must be parsable to the given type, if not, an error is thrown.
* The variable types that are supported as of now are: `int`, `float`, `bool` and `string`.
* Any variable that is being used must be declared in the named query clause in the beginning.

```
query test($a: int = 2, $b: int!, $name: string) {
  me(func: allofterms(name@en, $name)) {
    director.film (first: $a, offset: $b) {
      genre(first: $a) {
        name@en
      }
    }
  }
}
```


{{% notice "note" %}}
If you want to input a list of uids as a GraphQL variable value, you can have the variable as string type and
have the value surrounded by square brackets like `["13", "14"]`.
{{% /notice %}}

## Indexing with Custom Tokenizers

Dgraph comes with a large toolkit of builtin indexes, but sometimes for niche
use cases they're not always enough.

Dgraph allows you to implement custom tokenizers via a plugin system in order
to fill the gaps.

### Caveats

The plugin system uses Go's [`pkg/plugin`](https://golang.org/pkg/plugin/).
This brings some restrictions to how plugins can be used.

- Plugins must be written in Go.

- As of Go 1.9, `pkg/plugin` only works on Linux. Therefore, plugins will only
  work on dgraph instances deployed in a Linux environment.

- The version of Go used to compile the plugin should be the same as the version
  of Go used to compile Dgraph itself. Dgraph always uses the latest version of
Go (and so should you!).

### Implementing a plugin

{{% notice "note" %}}
You should consider Go's [plugin](https://golang.org/pkg/plugin/) documentation
to be supplementary to the documentation provided here.
{{% /notice %}}

Plugins are implemented as their own main package. They must export a
particular symbol that allows Dgraph to hook into the custom logic the plugin
provides.

The plugin must export a symbol named `Tokenizer`. The type of the symbol must
be `func() interface{}`. When the function is called the result returned should
be a value that implements the following interface:

```
type PluginTokenizer interface {
    // Name is the name of the tokenizer. It should be unique among all
    // builtin tokenizers and other custom tokenizers. It identifies the
    // tokenizer when an index is set in the schema and when search/filter
    // is used in queries.
    Name() string

    // Identifier is a byte that uniquely identifiers the tokenizer.
    // Bytes in the range 0x80 to 0xff (inclusive) are reserved for
    // custom tokenizers.
    Identifier() byte

    // Type is a string representing the type of data that is to be
    // tokenized. This must match the schema type of the predicate
    // being indexde. Allowable values are shown in the table below.
    Type() string

    // Tokens should implement the tokenization logic. The input is
    // the value to be tokenized, and will always have a concrete type
    // corresponding to Type(). The return value should be a list of
    // the tokens generated.
    Tokens(interface{}) ([]string, error)
}
```

The return value of `Type()` corresponds to the concrete input type of
`Tokens(interface{})` in the following way:

 `Type()` return value | `Tokens(interface{})` input type
-----------------------|----------------------------------
 `"int"`               | `int64`
 `"float"`             | `float64`
 `"string"`            | `string`
 `"bool"`              | `bool`
 `"datetime"`          | `time.Time`

### Building the plugin

The plugin has to be built using the `plugin` build mode so that an `.so` file
is produced instead of a regular executable. For example:

```sh
go build -buildmode=plugin -o myplugin.so ~/go/src/myplugin/main.go
```

### Running Dgraph with plugins

When starting Dgraph, use the `--custom_tokenizers` flag to tell dgraph which
tokenizers to load. It accepts a comma separated list of plugins. E.g.

```sh
dgraph ...other-args... --custom_tokenizers=plugin1.so,plugin2.so
```

{{% notice "note" %}}
Plugin validation is performed on startup. If a problem is detected, Dgraph
will refuse to initialise.
{{% /notice %}}

### Adding the index to the schema

To use a tokenization plugin, an index has to be created in the schema.

The syntax is the same as adding any built-in index. To add an custom index
using a tokenizer plugin named `foo` to a `string` predicate named
`my_predicate`, use the following in the schema:

```sh
my_predicate: string @index(foo) .
```

### Using the index in queries

There are two functions that can use custom indexes:

| Mode    | Behaviour                                                 |
| ------- | --------------------------------------------------------- |
| `anyof` | Returns nodes that match on *any* of the tokens generated |
| `allof` | Returns nodes that match on *all* of the tokens generated |

The functions can be used either at the query root or in filters.

There behaviour here an analogous to `anyofterms`/`allofterms` and
`anyoftext`/`alloftext`.

### Examples

The following examples should make the process of writing a tokenization plugin
more concrete.

#### Unicode Characters

This example shows the type of tokenization that is similar to term
tokenization of full text search. Instead of being broken down into terms or
stem words, the text is instead broken down into its constituent unicode
codepoints (in Go terminology these are called *runes*).

{{% notice "note" %}}
This tokenizer would create a very large index that would be expensive to
manage and store. That's one of the reasons that text indexing usually occurs
at a higher level; stem words for full text search or terms for term search.
{{% /notice %}}

The implementation of the plugin looks like this:

```go
package main

import "encoding/binary"

func Tokenizer() interface{} { return RuneTokenizer{} }

type RuneTokenizer struct{}

func (RuneTokenizer) Name() string     { return "rune" }
func (RuneTokenizer) Type() string     { return "string" }
func (RuneTokenizer) Identifier() byte { return 0xfd }

func (t RuneTokenizer) Tokens(value interface{}) ([]string, error) {
	var toks []string
	for _, r := range value.(string) {
		var buf [binary.MaxVarintLen32]byte
		n := binary.PutVarint(buf[:], int64(r))
		tok := string(buf[:n])
		toks = append(toks, tok)
	}
	return toks, nil
}
```

**Hints and tips:**

- Inside `Tokens`, you can assume that `value` will have concrete type
  corresponding to that specified by `Type()`. It's safe to do a type
assertion.

- Even though the return value is `[]string`, you can always store non-unicode
  data inside the string. See [this blogpost](https://blog.golang.org/strings)
for some interesting background how string are implemented in Go and why they
can be used to store non-textual data. By storing arbitrary data in the string,
you can make the index more compact. In this case, varints are stored in the
return values.

Setting up the indexing and adding data:
```
name: string @index(rune) .
```


```
{
  set{
    _:ad <name> "Adam" .
    _:aa <name> "Aaron" .
    _:am <name> "Amy" .
    _:ro <name> "Ronald" .
  }
}
```
Now queries can be performed.

The only person that has all of the runes `A` and `n` in their `name` is Aaron:
```
{
  q(func: allof(name, rune, "An")) {
    name
  }
}
=>
{
  "data": {
    "q": [
      { "name": "Aaron" }
    ]
  }
}
```
But there are multiple people who have both of the runes `A` and `m`:
```
{
  q(func: allof(name, rune, "Am")) {
    name
  }
}
=>
{
  "data": {
    "q": [
      { "name": "Amy" },
      { "name": "Adam" }
    ]
  }
}
```
Case is taken into account, so if you search for all names containing `"ron"`,
you would find `"Aaron"`, but not `"Ronald"`. But if you were to search for
`"no"`, you would match both `"Aaron"` and `"Ronald"`. The order of the runes in
the strings doesn't matter.

It's possible to search for people that have *any* of the supplied runes in
their names (rather than *all* of the supplied runes). To do this, use `anyof`
instead of `allof`:
```
{
  q(func: anyof(name, rune, "mr")) {
    name
  }
}
=>
{
  "data": {
    "q": [
      { "name": "Adam" },
      { "name": "Aaron" },
      { "name": "Amy" }
    ]
  }
}
```
`"Ronald"` doesn't contain `m` or `r`, so isn't found by the search.

{{% notice "note" %}}
Understanding what's going on under the hood can help you intuitively
understand how `Tokens` method should be implemented.

When Dgraph sees new edges that are to be indexed by your tokenizer, it
will tokenize the value. The resultant tokens are used as keys for posting
lists. The edge subject is then added to the posting list for each token.

When a query root search occurs, the search value is tokenized. The result of
the search is all of the nodes in the union or intersection of the correponding
posting lists (depending on whether `anyof` or `allof` was used).
{{% /notice %}}

#### CIDR Range

Tokenizers don't always have to be about splitting text up into its constituent
parts. This example indexes [IP addresses into their CIDR
ranges](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing). This
allows you to search for all IP addresses that fall into a particular CIDR
range.

The plugin code is more complicated than the rune example. The input is an IP
address stored as a string, e.g. `"100.55.22.11/32"`. The output are the CIDR
ranges that the IP address could possibly fall into. There could be up to 32
different outputs (`"100.55.22.11/32"` does indeed have 32 possible ranges, one
for each mask size).

```go
package main

import "net"

func Tokenizer() interface{} { return CIDRTokenizer{} }

type CIDRTokenizer struct{}

func (CIDRTokenizer) Name() string     { return "cidr" }
func (CIDRTokenizer) Type() string     { return "string" }
func (CIDRTokenizer) Identifier() byte { return 0xff }

func (t CIDRTokenizer) Tokens(value interface{}) ([]string, error) {
	_, ipnet, err := net.ParseCIDR(value.(string))
	if err != nil {
		return nil, err
	}
	ones, bits := ipnet.Mask.Size()
	var toks []string
	for i := ones; i >= 1; i-- {
		m := net.CIDRMask(i, bits)
		tok := net.IPNet{
			IP:   ipnet.IP.Mask(m),
			Mask: m,
		}
		toks = append(toks, tok.String())
	}
	return toks, nil
}
```
An example of using the tokenizer:

Setting up the indexing and adding data:
```
ip: string @index(cidr) .

```

```
{
  set{
    _:a <ip> "100.55.22.11/32" .
    _:b <ip> "100.33.81.19/32" .
    _:c <ip> "100.49.21.25/32" .
    _:d <ip> "101.0.0.5/32" .
    _:e <ip> "100.176.2.1/32" .
  }
}
```
```
{
  q(func: allof(ip, cidr, "100.48.0.0/12")) {
    ip
  }
}
=>
{
  "data": {
    "q": [
      { "ip": "100.55.22.11/32" },
      { "ip": "100.49.21.25/32" }
    ]
  }
}
```
The CIDR ranges of `100.55.22.11/32` and `100.49.21.25/32` are both
`100.48.0.0/12`.  The other IP addresses in the database aren't included in the
search result, since they have different CIDR ranges for 12 bit masks
(`100.32.0.0/12`, `101.0.0.0/12`, `100.154.0.0/12` for `100.33.81.19/32`,
`101.0.0.5/32`, and `100.176.2.1/32` respectively).

Note that we're using `allof` instead of `anyof`. Only `allof` will work
correctly with this index. Remember that the tokenizer generates all possible
CIDR ranges for an IP address. If we were to use `anyof` then the search result
would include all IP addresses under the 1 bit mask (in this case, `0.0.0.0/1`,
which would match all IPs in this dataset).

#### Anagram

Tokenizers don't always have to return multiple tokens. If you just want to
index data into groups, have the tokenizer just return an identifying member of
that group.

In this example, we want to find groups of words that are
[anagrams](https://en.wikipedia.org/wiki/Anagram) of each
other.

A token to correspond to a group of anagrams could just be the letters in the
anagram in sorted order, as implemented below:

```go
package main

import "sort"

func Tokenizer() interface{} { return AnagramTokenizer{} }

type AnagramTokenizer struct{}

func (AnagramTokenizer) Name() string     { return "anagram" }
func (AnagramTokenizer) Type() string     { return "string" }
func (AnagramTokenizer) Identifier() byte { return 0xfc }

func (t AnagramTokenizer) Tokens(value interface{}) ([]string, error) {
	b := []byte(value.(string))
	sort.Slice(b, func(i, j int) bool { return b[i] < b[j] })
	return []string{string(b)}, nil
}
```
In action:

Setting up the indexing and adding data:
```
word: string @index(anagram) .
```

```
{
  set{
    _:1 <word> "airmen" .
    _:2 <word> "marine" .
    _:3 <word> "beat" .
    _:4 <word> "beta" .
    _:5 <word> "race" .
    _:6 <word> "care" .
  }
}
```
```
{
  q(func: allof(word, anagram, "remain")) {
    word
  }
}
=>
{
  "data": {
    "q": [
      { "word": "airmen" },
      { "word": "marine" }
    ]
  }
}
```

Since a single token is only ever generated, it doesn't matter if `anyof` or
`allof` is used. The result will always be the same.

#### Integer prime factors

All all of the custom tokenizers shown previously have worked with strings.
However, other data types can be used as well. This example is contrived, but
nonetheless shows some advanced usages of custom tokenizers.

The tokenizer creates a token for each prime factor in the input.

```
package main

import (
    "encoding/binary"
    "fmt"
)

func Tokenizer() interface{} { return FactorTokenizer{} }

type FactorTokenizer struct{}

func (FactorTokenizer) Name() string     { return "factor" }
func (FactorTokenizer) Type() string     { return "int" }
func (FactorTokenizer) Identifier() byte { return 0xfe }

func (FactorTokenizer) Tokens(value interface{}) ([]string, error) {
    x := value.(int64)
    if x <= 1 {
        return nil, fmt.Errorf("cannot factor int <= 1: %d", x)
    }
    var toks []string
    for p := int64(2); x > 1; p++ {
        if x%p == 0 {
            toks = append(toks, encodeInt(p))
            for x%p == 0 {
                x /= p
            }
        }
    }
    return toks, nil

}

func encodeInt(x int64) string {
    var buf [binary.MaxVarintLen64]byte
    n := binary.PutVarint(buf[:], x)
    return string(buf[:n])
}
```
{{% notice "note" %}}
Notice that the return of `Type()` is `"int"`, corresponding to the concrete
type of the input to `Tokens` (which is `int64`).
{{% /notice %}}

This allows you do do things like search for all numbers that share prime
factors with a particular number.

In particular, we search for numbers that contain any of the prime factors of
15, i.e. any numbers that are divisible by either 3 or 5.

Setting up the indexing and adding data:
```
num: int @index(factor) .
```

```
{
  set{
    _:2 <num> "2"^^<xs:int> .
    _:3 <num> "3"^^<xs:int> .
    _:4 <num> "4"^^<xs:int> .
    _:5 <num> "5"^^<xs:int> .
    _:6 <num> "6"^^<xs:int> .
    _:7 <num> "7"^^<xs:int> .
    _:8 <num> "8"^^<xs:int> .
    _:9 <num> "9"^^<xs:int> .
    _:10 <num> "10"^^<xs:int> .
    _:11 <num> "11"^^<xs:int> .
    _:12 <num> "12"^^<xs:int> .
    _:13 <num> "13"^^<xs:int> .
    _:14 <num> "14"^^<xs:int> .
    _:15 <num> "15"^^<xs:int> .
    _:16 <num> "16"^^<xs:int> .
    _:17 <num> "17"^^<xs:int> .
    _:18 <num> "18"^^<xs:int> .
    _:19 <num> "19"^^<xs:int> .
    _:20 <num> "20"^^<xs:int> .
    _:21 <num> "21"^^<xs:int> .
    _:22 <num> "22"^^<xs:int> .
    _:23 <num> "23"^^<xs:int> .
    _:24 <num> "24"^^<xs:int> .
    _:25 <num> "25"^^<xs:int> .
    _:26 <num> "26"^^<xs:int> .
    _:27 <num> "27"^^<xs:int> .
    _:28 <num> "28"^^<xs:int> .
    _:29 <num> "29"^^<xs:int> .
    _:30 <num> "30"^^<xs:int> .
  }
}
```
```
{
  q(func: anyof(num, factor, 15)) {
    num
  }
}
=>
{
  "data": {
    "q": [
      { "num": 3 },
      { "num": 5 },
      { "num": 6 },
      { "num": 9 },
      { "num": 10 },
      { "num": 12 },
      { "num": 15 },
      { "num": 18 }
      { "num": 20 },
      { "num": 21 },
      { "num": 25 },
      { "num": 24 },
      { "num": 27 },
      { "num": 30 },
    ]
  }
}
```
