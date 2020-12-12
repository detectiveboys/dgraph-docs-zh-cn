在Dgraph中添加或删除数据称为mutation。

## 三元组

用`set`关键字完成一个增加三元组的mutation。

```graphql
{
  set {
    # triples in here
  }
}
```

输入语言为W3C标准[RDF N-Quad格式](https://www.w3.org/TR/n-quads/)的三元组。(绝对不是标准的！译者注)

每个三元组具有以下形式

```turtle
<subject> <predicate> <object> .
```

意味着由 `subject` 标识的图节点通过有向边 `predicate` 链接到 `object`。每个三元组以一个句号结尾。三元组的 `subject` 始终是图中的一个节点，而`object` 可以是节点或值（字面量）。

例如，这个三元组

```
<0x01> <name> "Alice" .
<0x01> <dgraph.type> "Person" .
```

表示ID为 `0x01` 的图节点的 `name` 具有字符串类型的值“Alice”。

这个三元组

```
<0x01> <friend> <0x02> .
```

表示ID为 `0x01` 的图节点通过 `friend` 边链接到节点 `0x02` 。

Dgraph为mutation中的每个空白节点创建唯一的64位标识符 - 节点的UID。mutation可以包括空白节点作为对象或对象的标识符，或者来自先前mutation的已知UID。

## 空白节点和UID

Blank nodes in mutations, written `_:identifier`, identify nodes within a mutation.  Dgraph creates a UID identifying each blank node and returns the created UIDs as the mutation result.  For example, mutation:
mutation中的空白节点，写为 `_:identifier`，标识mutation中的节点。 Dgraph为每个空白节点创建一个UID，并返回创建的UID作为mutation结果。
例如，mutation：

```graphql
{
 set {
    _:class <student> _:x .
    _:class <student> _:y .
    _:class <name> "awesome class" .
    _:class <dgraph.type> "Class" .
    _:x <name> "Alice" .
    _:x <dgraph.type> "Person" .
    _:x <dgraph.type> "Student" .
    _:x <planet> "Mars" .
    _:x <friend> _:y .
    _:y <name> "Bob" .
    _:y <dgraph.type> "Person" .
    _:y <dgraph.type> "Student" .
 }
}
```

结果输出（在此mutation的每次运行后，实际的UID都会有所不同）

```json
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {
      "class": "0x2712",
      "x": "0x2713",
      "y": "0x2714"
    }
  }
}
```

该图已更新，就好像它存储了这些三元组一样

```
<0x6bc818dc89e78754> <student> <0xc3bcc578868b719d> .
<0x6bc818dc89e78754> <student> <0xb294fb8464357b0a> .
<0x6bc818dc89e78754> <name> "awesome class" .
<0x6bc818dc89e78754> <dgraph.type> "Class" .
<0xc3bcc578868b719d> <name> "Alice" .
<0xc3bcc578868b719d> <dgraph.type> "Person" .
<0xc3bcc578868b719d> <dgraph.type> "Student" .
<0xc3bcc578868b719d> <planet> "Mars" .
<0xc3bcc578868b719d> <friend> <0xb294fb8464357b0a> .
<0xb294fb8464357b0a> <name> "Bob" .
<0xb294fb8464357b0a> <dgraph.type> "Person" .
<0xb294fb8464357b0a> <dgraph.type> "Student" .
```

空白节点标签`_:class`，`_:x` 和 `_:y` 不能标识mutation后的节点，可以安全地重用以标识之后的mutation中的新节点。

以后的mutation可以更新现有UID的数据。
例如，以下将新学生添加到班级

```graphql
{
 set {
    <0x6bc818dc89e78754> <student> _:x .
    _:x <name> "Chris" .
    _:x <dgraph.type> "Person" .
    _:x <dgraph.type> "Student" .
 }
}
```

查询也可以直接使用UID

```graphql
{
 class(func: uid(0x6bc818dc89e78754)) {
  name
  student {
   name
   planet
   friend {
    name
   }
  }
 }
}
```

## 外部ID

Dgraph's input language, RDF, also supports triples of the form `<a_fixed_identifier> <predicate> literal/node` and variants on this, where the label `a_fixed_identifier` is intended as a unique identifier for a node.  For example, mixing [schema.org](http://schema.org) identifiers, [the movie database](https://www.themoviedb.org/) identifiers and blank nodes:
Dgraph的输入语言RDF也支持 `<a_fixed_identifier> <predicate> literal/node` 形式的三元组和其上的变变种，其中标签 `a_fixed_identifier` 旨在作为节点的唯一标识符。例如，混合使用schema.org标识符、电影数据标识符和空白节点：

```
_:userA <http://schema.org/type> <http://schema.org/Person> .
_:userA <dgraph.type> "Person" .
_:userA <http://schema.org/name> "FirstName LastName" .
<https://www.themoviedb.org/person/32-robin-wright> <http://schema.org/type> <http://schema.org/Person> .
<https://www.themoviedb.org/person/32-robin-wright> <http://schema.org/name> "Robin Wright" .
```

由于Dgraph本身不支持诸如节点标识符之类的外部ID。相反，可以将外部ID存储为具有 `xid` 边的节点的属性。例如，上面三元组，谓词名称在Dgraph中有效，但是用 `<http://schema.org/Person>` 标识的节点可以在Dgraph中标识为UID `0x123` 和边。

```
<0x123> <xid> "http://schema.org/Person" .
<0x123> <dgraph.type> "ExternalType" .
```

虽然 Robin Wright 可能会获得UID `0x321` 和三元组

```
<0x321> <xid> "https://www.themoviedb.org/person/32-robin-wright" .
<0x321> <http://schema.org/type> <0x123> .
<0x321> <http://schema.org/name> "Robin Wright" .
<0x321> <dgraph.type> "Person" .
```

schema应该如下

```
xid: string @index(exact) .
<http://schema.org/type>: uid @reverse .
```

查询示例：所有人

```graphql
{
  var(func: eq(xid, "http://schema.org/Person")) {
    allPeople as <~http://schema.org/type>
  }

  q(func: uid(allPeople)) {
    <http://schema.org/name>
  }
}
```

查询示例：通过外部ID查询Robin Wright。

```graphql
{
  robin(func: eq(xid, "https://www.themoviedb.org/person/32-robin-wright")) {
    expand(_all_) { expand(_all_) }
  }
}

```

**注意**
*xid边不会自动添加到mutation中。通常，用户有责任检查现有的xid，并在必要时添加节点和xid边。 Dgraph将此类xid的唯一性检查留给了外部程序。*

## 外部ID和Upsert块

upsert块使管理外部ID变得容易

设置schema

```
xid: string @index(exact) .
<http://schema.org/name>: string @index(exact) .
<http://schema.org/type>: [uid] @reverse .
```

首先设置类型

```graphql
{
  set {
    _:blank <xid> "http://schema.org/Person" .
    _:blank <dgraph.type> "ExternalType" .
  }
}
```

现在，您可以创建一个新person，并使用upsert块附上其类型。

```graphql
upsert {
  query {
    var(func: eq(xid, "http://schema.org/Person")) {
      Type as uid
    }
    var(func: eq(<http://schema.org/name>, "Robin Wright")) {
      Person as uid
    }
  }
  mutation {
      set {
        uid(Person) <xid> "https://www.themoviedb.org/person/32-robin-wright" .
        uid(Person) <http://schema.org/type> uid(Type) .
        uid(Person) <http://schema.org/name> "Robin Wright" .
        uid(Person) <dgraph.type> "Person" .
      }
  }
}
```

您还可以删除一个person节点并分离type和person节点之间的关系。与上面相同，但您使用的是关键字 “`delete`” 而不是 “`set`”。 “`http://schema.org/Person`” 将保留，但 “`Robin Wright`” 将被删除。

```graphql
upsert {
  query {
    var(func: eq(xid, "http://schema.org/Person")) {
      Type as uid
    }
    var(func: eq(<http://schema.org/name>, "Robin Wright")) {
      Person as uid
    }
  }
  mutation {
      delete {
        uid(Person) <xid> "https://www.themoviedb.org/person/32-robin-wright" .
        uid(Person) <http://schema.org/type> uid(Type) .
        uid(Person) <http://schema.org/name> "Robin Wright" .
        uid(Person) <dgraph.type> "Person" .
      }
  }
}
```

按person查询

```graphql
{
  q(func: eq(<http://schema.org/name>, "Robin Wright")) {
    uid
    xid
    <http://schema.org/name>
    <http://schema.org/type> {
      uid
      xid
    }
  }
}
```

## 多语言和RDF类型

RDF N-Quad允许为字符串值和RDF类型指定一种语言。语言是使用`@lang`编写的。例如

```
<0x01> <name> "Adelaide"@en .
<0x01> <name> "Аделаида"@ru .
<0x01> <name> "Adélaïde"@fr .
<0x01> <dgraph.type> "Person" .
```

另请参阅[查询中如何处理语言字符串](/query-language/index?id=多语言支持)。

使用标准的^^分隔符将RDF类型附加到文字上。例如
```
<0x01> <age> "32"^^<xs:int> .
<0x01> <birthdate> "1985-06-08"^^<xs:dateTime> .
```

支持的[RDF数据类型](https://www.w3.org/TR/rdf11-concepts/#section-Datatypes)和存储数据的相应内部类型如下。

| Storage Type                                            | Dgraph type    |
| -------------                                           | :------------: |
| `<xs:string>`                                     | `string`         |
| `<xs:dateTime>`                                   | `dateTime`       |
| `<xs:date>`                                       | `datetime`       |
| `<xs:int>`                                        | `int`            |
| `<xs:boolean>`                                    | `bool`           |
| `<xs:double>`                                     | `float`          |
| `<xs:float>`                                      | `float`          |
| `<geo:geojson>`                                   | `geo`            |
| `<http://www.w3.org/2001/XMLSchema#string>`   | `string`         |
| `<http://www.w3.org/2001/XMLSchema#dateTime>` | `dateTime`       |
| `<http://www.w3.org/2001/XMLSchema#date>`     | `dateTime`       |
| `<http://www.w3.org/2001/XMLSchema#int>`      | `int`            |
| `<http://www.w3.org/2001/XMLSchema#boolean>`  | `bool`           |
| `<http://www.w3.org/2001/XMLSchema#double>`   | `float`          |
| `<http://www.w3.org/2001/XMLSchema#float>`    | `float`          |

请参阅有关[RDF schema 类型](/query-language/index?id=RDF类型)的部分，以了解RDF类型如何影响mutation和存储。

## 批量mutation

每个mutation可能包含多个RDF三元组。对于大数据加载，可以并行处理许多此类mutation。 `dgraph live`命令就是这样做的。默认情况下，将1000条RDF行批处理到一个查询中，同时并行运行100条这样的查询。

`dgraph live` takes as input gzipped N-Quad files (that is triple lists without `{ set {`) and batches mutations for all triples in the input.  The tool has documentation of options.
`dgraph live`将gzip压缩的N-Quad文件（即没有`{set {`的三元组列表）作为输入，并对输入中所有三元组进行批量mutation。该工具具有options文档。

```sh
dgraph live --help
```

另请参阅[快速数据加载]。
See also [Bulk Data Loading](/deploy/index?id=快速数据加载).

## 删除

用`delete`关键字表示的delete mutation将从存储中删除三元组。

例如，如果存储中包含以下内容：

```
<0xf11168064b01135b> <name> "Lewis Carrol"
<0xf11168064b01135b> <died> "1998"
<0xf11168064b01135b> <dgraph.type> "Person" .
```

然后，以下delete mutation将删除指定的错误数据，并将其从所有索引中删除：

```graphql
{
  delete {
     <0xf11168064b01135b> <died> "1998" .
  }
}
```


### 通配符删除

在许多情况下，您将需要删除谓词的多种数据。对于特定的节点N，谓词P的所有数据（以及所有对应的索引）都以模式`S P *`删除。

```graphql
{
  delete {
     <0xf11168064b01135b> <author.of> * .
  }
}
```

模式`S * *`从节点中删除所有已知边，与已删除边相对应的任何反向边以及已删除数据的任何索引。

**注意** 
*对于适合`S * *`模式的mutation，仅删除与给定节点关联的类型（使用`dgraph.type`）中的谓词。 `S * *` delete mutation后，所有与该节点类型之一不匹配的谓词都将保留。*

```graphql
{
  delete {
     <0xf11168064b01135b> * * .
  }
}
```

如果删除模式`S * *`中的节点S仅具有由dgraph.type定义的类型的少数谓词，则仅删除具有类型谓词的那些三元组。 `S * *` delete mutation后，包含无类型谓词的节点仍将存在。

**注意**
*不支持模式`* P O`和`* * O`，因为它们存储和查找所有传入边的效率很低。

### 删除非列表谓词

删除非列表谓词的值（即一对一关系）可以通过两种方式完成。
- 使用上一节中提到的[通配符删除](###通配符删除)（星号）。
- 将对象设置为特定值。如果传递的值不是当前值，则该突变将成功但不会生效。如果传递的值是当前值，则该mutation将成功并删除非列表谓词。

对于带有语言标签的值，支持以下特殊语法：

```graphql
{
  delete {
    <0x12345> <name@es> * .
  }
}
```

在此示例中，使用`es`语言标记的名称字段的值被删除。其他标记的值保持不变。

## 具有RDF的列表类型的Facets

Schema:

```
<name>: string @index(exact).
<nickname>: [string] .
```

在RDF中创建包含facets的列表非常简单。

```graphql
{
  set {
    _:Julian <name> "Julian" .
    _:Julian <nickname> "Jay-Jay" (kind="first") .
    _:Julian <nickname> "Jules" (kind="official") .
    _:Julian <nickname> "JB" (kind="CS-GO") .
  }
}
```

```graphql
{
  q(func: eq(name,"Julian")){
    name
    nickname @facets
  }
}
```

结果：

```json
{
  "data": {
    "q": [
      {
        "name": "Julian",
        "nickname|kind": {
          "0": "first",
          "1": "official",
          "2": "CS-GO"
        },
        "nickname": [
          "Jay-Jay",
          "Jules",
          "JB"
        ]
      }
    ]
  }
}
```

## 使用CURL进行mutation

可以通过向Alpha的`/mutate`端点发出POST请求来进行mutation。在命令行上，可以使用curl来完成。要提交mutation，请在URL中传递参数`commitNow=true`。

运行 `set` mutation:

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
{
  set {
    _:alice <name> "Alice" .
    _:alice <dgraph.type> "Person" .
  }
}'
```

运行 `delete` mutation:

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
{
  delete {
    # Example: The UID of Alice is 0x56f33
    <0x56f33> <name> * .
  }
}'
```

要运行存在文件中的RDF mutation，请使用curl的--data-binary选项，与-d选项不同的是，该数据不经过URL编码。

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true --data-binary @mutation.txt
```

## JSON mutation 格式

也可以使用JSON对象指定mutation。这可以使mutation以更自然的方式表达。由于大多数语言已经具有JSON编组库，因此也消除了应用程序具有自定义序列化代码的需要。

当Dgraph接收到作为JSON对象的mutation时，它首先转换为多个RDF，然后按常规进行处理。

每个JSON对象代表图中的单个节点。

**注意**
*JSON mutation可通过gRPC客户端（例如Go客户端，JS客户端和Java客户端）获得，并且可用于具有dgraph-js-http和cURL的HTTP客户端。在[此处](https://github.com/dgraph-io/dgraph-js-http)查看有关cURL的更多信息*

### 设置文字值

设置新值时，`mutation`消息中的`set_json`字段应包含一个JSON对象。

可以通过将key/value添加到JSON对象来设置文字值。key代表谓词，value代表对象。

例如:

```json
{
  "name": "diggy",
  "food": "pizza",
  "dgraph.type": "Mascot"
}
```

将转换为RDF：

```
_:blank-0 <name> "diggy" .
_:blank-0 <food> "pizza" .
_:blank-0 <dgraph.type> "Mascot" .
```

mutation的结果还将包含一个映射，该映射将分配与键`blank-0`相对应的uid。您可以指定自己的密钥，例如

```json
{
  "uid": "_:diggy",
  "name": "diggy",
  "food": "pizza"
}
```

在这种情况下，分配的uid映射将具有一个称为diggy的键，其值为分配给它的uid。

### 多语言支持

RDF和JSON mutation之间的重要区别在于指定字符串值的语言。在JSON中，语言标签会附加到边名称中，而不是像RDF中那样附加到值中。

例如，JSON mutation：

```json
{
  "food": "taco",
  "rating@en": "tastes good",
  "rating@es": "sabe bien",
  "rating@fr": "c'est bon",
  "rating@it": "è buono",
  "dgraph.type": "Food"
}
```

等效于以下RDF：

```json
_:blank-0 <food> "taco" .
_:blank-0 <dgraph.type> "Food" .
_:blank-0 <rating> "tastes good"@en .
_:blank-0 <rating> "sabe bien"@es .
_:blank-0 <rating> "c'est bon"@fr .
_:blank-0 <rating> "è buono"@it .
```

### Geo-location支持

JSON中提供了对地理位置数据的支持。Geo-location数据作为JSON对象输入，其键为“type”和“coordinates”。请记住，我们仅支持对Point，Polygon和MultiPolygon类型进行索引，但是我们可以存储其他类型的地理位置数据。下面是一个示例：

```json
{
  "food": "taco",
  "location": {
    "type": "Point",
    "coordinates": [1.0, 2.0]
  }
}
```

### 引用现有节点

如果JSON对象包含一个名为`"uid"`的字段，则该字段将被解释为图形中现有节点的UID。此机制使您可以引用现有节点。

例如：

```json
{
  "uid": "0x467ba0",
  "food": "taco",
  "rating": "tastes good",
  "dgraph.type": "Food"
}
```

将转换为RDF：

```
<0x467ba0> <food> "taco" .
<0x467ba0> <rating> "tastes good" .
<0x467ba0> <dgraph.type> "Food" .
```

### 节点之间的边

节点之间的边以与文字值相似的方式表示，只是对象变成了JSON对象。

例如：

```json
{
  "name": "Alice",
  "friend": {
    "name": "Betty"
  }
}
```

将转换为RDF：

```
_:blank-0 <name> "Alice" .
_:blank-0 <friend> _:blank-1 .
_:blank-1 <name> "Betty" .
```

mutation的结果将包含分配给`blank-0`和`blank-1`节点的uid。如果要在其他键下返回这些`uid`，则可以将uid字段指定为空白节点。

```json
{
  "uid": "_:alice",
  "name": "Alice",
  "friend": {
    "uid": "_:bob",
    "name": "Betty"
  }
}
```

将转换为：

```
_:alice <name> "Alice" .
_:alice <friend> _:bob .
_:bob <name> "Betty" .
```

现有节点的引用方式与添加文字值时相同。例如，链接两个现有节点：

```json
{
  "uid": "0x123",
  "link": {
    "uid": "0x456"
  }
}
```

将转换为：

```
<0x123> <link> <0x456> .
```

**注意**
*常见的错误是尝试使用`{"uid":"0x123","link":"0x456"}`。这将导致错误。 Dgraph将此JSON对象解释为将`link`谓词设置为字符串`"0x456"`，这通常并是不想要的。*

### 删除文字值

删除mutation也可以JSON格式发送。要发送删除mutation，请使用`delete_json`字段而不是`mutation`消息中的`set_json`字段。

**注意**
*如果您使用的是dgraph-js-http客户端或Ratel UI，请查看[使用Raw HTTP或Ratel UI的JSON语法](###使用Raw-HTTP或Ratel-UI的JSON语法)部分。*

使用删除mutation时，始终必须引用现有节点。因此，必须存在每个JSON对象的`"uid"`字段。应该删除的谓词应设置为JSON值null。

例如，要删除food rating：

```json
{
  "uid": "0x467ba0",
  "rating": null
}
```

### 删除边

删除单个边需要与创建该边相同的JSON对象。例如，删除`link`链接从`"0x123"`到`"0x456"`：

```json
{
  "uid": "0x123",
  "link": {
    "uid": "0x456"
  }
}
```

可以一次删除谓词从单个节点发出的所有边（对应于删除`S P *`）：

```json
{
  "uid": "0x123",
  "link": null
}
```

如果未指定谓词，则删除节点的所有已知出站边（到其他节点和文字值）（相当于删除`S * *`）。使用类型系统派生要删除的谓词。有关更多信息，请参考[RDF格式](##删除)文档和[类型系统](/query-language/index?id=类型系统)部分。

```json
{
  "uid": "0x123"
}
```

### Facets

可以使用`|`创建Facets字符，用于分隔JSON对象字段名称中的谓词和Facets键。这与用于在查询结果中显示Facets的编码模式相同。例如：

```json
{
  "name": "Carol",
  "name|initial": "C",
  "dgraph.type": "Person",
  "friend": {
    "name": "Daryl",
    "friend|close": "yes",
    "dgraph.type": "Person"
  }
}
```

产生以下RDF：

```
_:blank-0 <name> "Carol" (initial=C) .
_:blank-0 <dgraph.type> "Person" .
_:blank-0 <friend> _:blank-1 (close=yes) .
_:blank-1 <name> "Daryl" .
_:blank-1 <dgraph.type> "Person" .
```

Facets不包含类型信息，但Dgraph将尝试从输入中猜测类型。如果Facets的值可以解析为数字，则它将转换为浮点数或整数。如果可以将其解析为布尔值，则将其存储为布尔值。如果该值为字符串，则如果该字符串与Dgraph可以识别的一种时间格式（YYYY，MM-YYYY，DD-MM-YYYY，RFC339等）匹配，则它将被存储为日期时间，否则为字符串。如果您不想冒将Facets数据误解为时间值的风险，最好将数字数据存储为int或float。

### 删除Facets

删除Facets的最简单方法是覆盖它。当您为没有Facets的同一实体创建新mutation时，现有Facets将被自动删除。

例如：

```
<0x1> <name> "Carol" .
<0x1> <friend> <0x2> .
```

另一种方法是使用Upsert块。
在下面的查询中，我们将删除`Name`和`Friend`谓词中的Facets。要覆盖，我们需要收集执行此操作的边的值，并使用函数`val(var)`来完成覆盖。

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    user as var(func: eq(name, "Carol")){
      Name as name
      Friends as friend
    }
 }

  mutation {
    set {
      uid(user) <name> val(Name) .
      uid(user) <friend> uid(Friends) .
    }
  }
}' | jq
```

### 使用JSON创建列表并与之交互

Schema:

```JSON
testList: [string] .
```

```JSON
{
  "testList": [
    "Grape",
    "Apple",
    "Strawberry",
    "Banana",
    "watermelon"
  ]
}
```

然后，从此列表中删除`"Apple"`（请记住，区分大小写）：

```graphql
{
  q(func: has(testList)) {
    uid
    testList
  }
}
```

```json
{
  "delete": {
    "uid": "0x6", #列表的UID
    "testList": "Apple"
  }
}
```

您也可以删除多个值

```json
{
  "delete": {
    "uid": "0x6",
    "testList": [
          "Strawberry",
          "Banana",
          "watermelon"
        ]
  }
}
```

**注意**
*如果您使用的是dgraph-js-http客户端或Ratel UI，请查看[使用Raw HTTP或Ratel UI的JSON语法](###使用Raw-HTTP或Ratel-UI的JSON语法)部分。*

添加另一种水果：

```JSON
{
   "uid": "0xd", #列表的UID
   "testList": "Pineapple"
}
```

### 具有JSON的列表类型的Facets

Schema:

```
<name>: string @index(exact).
<nickname>: [string] .
```

要创建列表类型谓词，您需要在一个列表中指定所有值。所有谓词值的Facets应一起指定。它以map格式完成，列表内的谓词值索引是map键，其各自的Facets值作为map值。没有Facets值的谓词值将从Facets映射中丢失。例如：

```json
{
  "set": [
    {
      "uid": "_:Julian",
      "name": "Julian",
      "nickname": ["Jay-Jay", "Jules", "JB"],
      "nickname|kind": {
        "0": "first",
        "1": "official",
        "2": "CS-GO"
      }
    }
  ]
}
```

在上面您会看到我们有三个值来输入带有各自Facets的列表。您可以运行此查询来检查包含Facets的列表：

```graphql
{
   q(func: eq(name,"Julian")) {
    uid
    nickname @facets
   }
}
```

之后，如果要使用Facets添加更多值，只需执行相同的过程，但是这次必须使用实际节点的UID来代替空白节点。

```json
{
  "set": [
    {
      "uid": "0x3",
      "nickname|kind": "Internet",
      "nickname": "@JJ"
    }
  ]
}
```

最终结果是：

```json
{
  "data": {
    "q": [
      {
        "uid": "0x3",
        "nickname|kind": {
          "0": "first",
          "1": "Internet",
          "2": "official",
          "3": "CS-GO"
        },
        "nickname": [
          "Jay-Jay",
          "@JJ",
          "Jules",
          "JB"
        ]
      }
    ]
  }
}
```

### 指定多个操作

当指定添加或删除mutation时，可以使用JSON数组同时指定多个节点。 
例如，以下JSON对象可用于添加两个新节点，每个节点都有一个`name`：

```json
[
  {
    "name": "Edward"
  },
  {
    "name": "Fredric"
  }
]
```

### 使用Raw HTTP或Ratel UI的JSON语法

此语法可以在Ratel的最新版本中，[dgraph-js-http](https://github.com/dgraph-io/dgraph-js-http)客户端中甚至通过cURL使用。

您也可以下载[适用于Linux，macOS或Windows的Ratel UI](https://discuss.dgraph.io/t/ratel-installer-for-linux-macos-and-windows-updated-2020/2884)。

Mutate:

```json
{
  "set": [
    {
      # 这里有一个JSON对象
    },
    {
      # 此处的另一个JSON对象用于多个操作
    }
  ]
}
```

删除:

删除操作与[删除文字值](###删除文字值)和[删除边](###删除边)相同。

Deletion operations are the same as [Deleting literal values]({{< relref "#deleting-literal-values">}}) and [Deleting edges]({{< relref "#deleting-edges">}}).

```json
{
  "delete": [
    {
      # 这里有一个JSON对象
    },
    {
      # 此处的另一个JSON对象用于多个操作
    }
  ]
}
```

### 通过cURL使用JSON操作

首先，您必须配置HTTP header以指定content-type。

```sh
-H 'Content-Type: application/json'
```

**注意**
*为了将jq用于JSON格式，您需要jq包。有关安装详细信息，请参见[jq下载](https://stedolan.github.io/jq/download/)页面。您还可以将Python内置的`json.tool`模块与`python -m json.tool`一起使用以进行JSON格式设置。*

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'
    {
      "set": [
        {
          "name": "Alice"
        },
        {
          "name": "Bob"
        }
      ]
    }' | jq

```

删除：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'
    {
      "delete": [
        {
          "uid": "0xa"
        }
      ]
    }' | jq
```

使用JSON文件进行mutation：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d @data.json
```

data.json的内容如下所示：

```json
{
  "set": [
    {
      "name": "Alice"
    },
    {
      "name": "Bob"
    }
  ]
}
```

对于通过HTTP进行的mutation，JSON文件必须遵循相同的格式：具有"set"或"delete"键的单个JSON对象，以及用于mutation的JSON对象数组。如果您已经具有包含数据数组的文件，则可以使用jq将数据转换为正确的格式。例如，如果您的data.json文件如下所示：

```json
[
  {
    "name": "Alice"
  },
  {
    "name": "Bob"
  }
]
```

那么您可以使用以下jq命令将数据转换为正确的格式，其中，`.`的代表data.json的内容：

```sh
cat data.json | jq '{set: .}'
```

```json
{
  "set": [
    {
      "name": "Alice"
    },
    {
      "name": "Bob"
    }
  ]
}
```

## Upsert块

upsert块允许在单个请求中执行查询和mutation。 upsert块包含一个查询块和一个或多个mutation块。可以使用`uid`和`val`函数在mutation块中使用查询块中定义的变量。

通常，upsert块的结构如下：

```graphql
upsert {
  query <query block>
  [fragment <fragment block>]
  mutation <mutation block 1>
  [mutation <mutation block 2>]
  ...
}
```

upsert块的执行还返回执行mutation之前在数据库状态上执行的查询的响应。为了获得最新结果，我们应该提交mutation并执行另一个查询。

### uid函数

uid函数允许从查询块中定义的变量中提取UID。根据执行查询块的结果，可能有两种结果：

- 如果变量为空，即没有节点与查询匹配，则在进行`set`操作的情况下，`uid`函数将返回新的UID，因此将其视为空白节点。另一方面，对于`delete`操作，它不返回UID，因此该操作变为无操作且被忽略。空白节点在所有mutation块中都将获得相同的UID。
- 如果变量存储一个或多个UID，则uid函数返回存储在变量中的所有UID。在这种情况下，将对返回的所有UID执行一次操作。

### val函数

`val`函数允许从值变量中提取值。值变量存储从UID到其对应值的映射。因此，将`val(v)`替换为N-Quad中UID（Subject）的映射中存储的值。如果变量v对于给定的UID没有值，则忽略该mutation。 `val`函数也可以与聚合变量的结果一起使用，在这种情况下，mutation中的所有UID都将使用聚合值进行更新。

### uid函数示例

考虑具有以下schema的示例：

```sh
curl localhost:8080/alter -X POST -d $'
  name: string @index(term) .
  email: string @index(exact, trigram) @upsert .
  age: int @index(int) .' | jq
```

现在，假设我们要使用电子邮件和名称信息创建一个新用户。我们还希望确保一封电子邮件在数据库中恰好有一个相应的用户。为此，我们需要先使用给定的电子邮件查询用户是否存在于数据库中。如果存在用户，我们将使用其UID更新名称信息。如果该用户不存在，我们将创建一个新用户并更新电子邮件和姓名信息。

我们可以使用upsert块执行此操作，如下所示：

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    q(func: eq(email, "user@company1.io")) {
      v as uid
      name
    }
  }

  mutation {
    set {
      uid(v) <name> "first last" .
      uid(v) <email> "user@company1.io" .
    }
  }
}' | jq
```

结果：

```json
{
  "data": {
    "q": [],
    "code": "Success",
    "message": "Done",
    "uids": {
      "uid(v)": "0x1"
    }
  },
  "extensions": {...}
}
```

upsert块的查询部分将用户的UID和提供的电子邮件存储在变量v中。然后，mutation部分从变量v提取UID，并将名称和电子邮件信息存储在数据库中。如果用户存在，则信息将更新。如果该用户不存在，则`uid(v)`被视为空白节点，并如上所述创建新用户。

如果我们再次运行相同的mutation，则数据将被覆盖，并且不会创建新的uid。请注意，再次执行该mutation时，`uids`在结果中为空，并且`data`中（`q`）包含上一个upsert中创建的uid。

```json
{
  "data": {
    "q": [
      {
        "uid": "0x1",
        "name": "first last"
      }
    ],
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

我们可以使用`json`数据实现相同的结果，如下所示：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '
{
  "query": "{ q(func: eq(email, \\"user@company1.io\\")) {v as uid\\n name} }",
  "set": {
    "uid": "uid(v)",
    "name": "first last",
    "email": "user@company1.io"
  }
}' | jq
```

现在，我们要添加具有相同电子邮件`user@company1.io`的同一用户的`age`信息。我们可以使用upsert块执行以下操作：

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    q(func: eq(email, "user@company1.io")) {
      v as uid
    }
  }

  mutation {
    set {
      uid(v) <age> "28" .
    }
  }
}' | jq
```

结果：

```json
{
  "data": {
    "q": [
      {
        "uid": "0x1"
      }
    ],
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

在这里，查询块查询`email`为`user@company1.io`的用户。它将用户的`uid`存储在变量`v`中。然后，mutation块通过使用`uid`函数从变量`v`中提取`uid`来更新用户的`age`。

我们可以使用json数据实现相同的结果，如下所示：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'
{
  "query": "{ q(func: eq(email, \\"user@company1.io\\")) {v as uid} }",
  "set":{
    "uid": "uid(v)",
    "age": "28"
  }
}' | jq
```

如果我们只想在用户存在时执行mutation，则可以使用[条件Upsert](##有条件的Upsert)。

### val函数示例

假设我们要将谓词`age`迁移到`other`，我们可以使用以下mutation来做到这一点：

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    v as var(func: has(age)) {
      a as age
    }
  }

  mutation {
    # we copy the values from the old predicate
    set {
      uid(v) <other> val(a) .
    }

    # and we delete the old predicate
    delete {
      uid(v) <age> * .
    }
  }
}' | jq
```

结果：

```json
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

在这里，变量`a`将存储从所有UID到其`age`的映射。然后，mutation块将每个UID的`age`相应值存储在`other`谓词中，并删除`age`谓词。

我们可以使用json数据实现相同的结果，如下所示：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d $'{
  "query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "delete": {
    "uid": "uid(v)",
    "age": null
  },
  "set": {
    "uid": "uid(v)",
    "other": "val(a)"
  }
}' | jq
```

### 批量删除示例

假设我们要从数据库中删除`company1`的所有用户。这可以使用upsert块在一个查询中完成，如下所示：

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    v as var(func: regexp(email, /.*@company1.io$/))
  }

  mutation {
    delete {
      uid(v) <name> * .
      uid(v) <email> * .
      uid(v) <age> * .
    }
  }
}' | jq
```

我们可以使用json数据实现相同的结果，如下所示：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '{
  "query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "delete": {
    "uid": "uid(v)",
    "name": null,
    "email": null,
    "age": null
  }
}' | jq
```

## 有条件的Upsert

upsert块还允许使用`@if`指令指定条件mutation块。仅当指定条件为true时，才执行mutation。如果条件为false，则忽略该mutation。有条件Upsert的一般结构如下所示：

```graphql
upsert {
  query <query block>
  [fragment <fragment block>]
  mutation [@if(<condition>)] <mutation block 1>
  [mutation [@if(<condition>)] <mutation block 2>]
  ...
}
```

`@if`指令接受查询块中定义的变量的条件，并且可以使用`AND`，`OR`和`NOT`进行连接。

### 有条件Upsert的示例

假设在前面的示例中，我们知道`company1`的员工人数少于100。为了安全起见，我们希望仅在变量`v`中存储的UID小于100但大于50时才执行该mutation。这可以通过以下方式实现：

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d  $'
upsert {
  query {
    v as var(func: regexp(email, /.*@company1.io$/))
  }

  mutation @if(lt(len(v), 100) AND gt(len(v), 50)) {
    delete {
      uid(v) <name> * .
      uid(v) <email> * .
      uid(v) <age> * .
    }
  }
}' | jq
```

我们可以使用json数据实现相同的结果，如下所示：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '{
  "query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "cond": "@if(lt(len(v), 100) AND gt(len(v), 50))",
  "delete": {
    "uid": "uid(v)",
    "name": null,
    "email": null,
    "age": null
  }
}' | jq
```

### 多个mutation块的例子

考虑具有以下schema的示例：

```sh
curl localhost:8080/alter -X POST -d $'
  name: string @index(term) .
  email: [string] @index(exact) @upsert .' | jq
```

假设我们在数据库中存储了许多用户，每个用户都有一个或多个电子邮件地址。现在，我们得到两个属于同一用户的电子邮件地址。如果电子邮件地址属于数据库中的不同节点，则我们要删除现有节点并创建一个新节点，并将这两个电子邮件都附加到该新节点上。否则，我们将使用这两封电子邮件创建新的或更新现有的节点。

```sh
curl -H "Content-Type: application/rdf" -X POST localhost:8080/mutate?commitNow=true -d $'
upsert {
  query {
    # filter is needed to ensure that we do not get same UIDs in u1 and u2
    q1(func: eq(email, "user_email1@company1.io")) @filter(not(eq(email, "user_email2@company1.io"))) {
      u1 as uid
    }

    q2(func: eq(email, "user_email2@company1.io")) @filter(not(eq(email, "user_email1@company1.io"))) {
      u2 as uid
    }

    q3(func: eq(email, "user_email1@company1.io")) @filter(eq(email, "user_email2@company1.io")) {
      u3 as uid
    }
  }

  # case when both emails do not exist
  mutation @if(eq(len(u1), 0) AND eq(len(u2), 0) AND eq(len(u3), 0)) {
    set {
      _:user <name> "user" .
      _:user <dgraph.type> "Person" .
      _:user <email> "user_email1@company1.io" .
      _:user <email> "user_email2@company1.io" .
    }
  }

  # case when email1 exists but email2 does not
  mutation @if(eq(len(u1), 1) AND eq(len(u2), 0) AND eq(len(u3), 0)) {
    set {
      uid(u1) <email> "user_email2@company1.io" .
    }
  }

  # case when email1 does not exist but email2 exists
  mutation @if(eq(len(u1), 0) AND eq(len(u2), 1) AND eq(len(u3), 0)) {
    set {
      uid(u2) <email> "user_email1@company1.io" .
    }
  }

  # case when both emails exist and needs merging
  mutation @if(eq(len(u1), 1) AND eq(len(u2), 1) AND eq(len(u3), 0)) {
    set {
      _:user <name> "user" .
      _:user <dgraph.type> "Person" .
      _:user <email> "user_email1@company1.io" .
      _:user <email> "user_email2@company1.io" .
    }

    delete {
      uid(u1) <name> * .
      uid(u1) <email> * .
      uid(u2) <name> * .
      uid(u2) <email> * .
    }
  }
}' | jq
```

结果（当数据库为空时）：

```json
{
  "data": {
    "q1": [],
    "q2": [],
    "q3": [],
    "code": "Success",
    "message": "Done",
    "uids": {
      "user": "0x1"
    }
  },
  "extensions": {...}
}
```

结果（两种电子邮件都存在并且被附加到不同的节点）：

```json
{
  "data": {
    "q1": [
      {
        "uid": "0x2"
      }
    ],
    "q2": [
      {
        "uid": "0x3"
      }
    ],
    "q3": [],
    "code": "Success",
    "message": "Done",
    "uids": {
      "user": "0x4"
    }
  },
  "extensions": {...}
}
```

结果（当两个电子邮件同时存在并且已经附加到同一节点时）：

```json
{
  "data": {
    "q1": [],
    "q2": [],
    "q3": [
      {
        "uid": "0x4"
      }
    ],
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {...}
}
```

我们可以使用json数据实现相同的结果，如下所示：

```sh
curl -H "Content-Type: application/json" -X POST localhost:8080/mutate?commitNow=true -d '{
  "query": "{q1(func: eq(email, \"user_email1@company1.io\")) @filter(not(eq(email, \"user_email2@company1.io\"))) {u1 as uid} \n q2(func: eq(email, \"user_email2@company1.io\")) @filter(not(eq(email, \"user_email1@company1.io\"))) {u2 as uid} \n q3(func: eq(email, \"user_email1@company1.io\")) @filter(eq(email, \"user_email2@company1.io\")) {u3 as uid}}",
  "mutations": [
    {
      "cond": "@if(eq(len(u1), 0) AND eq(len(u2), 0) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "_:user",
          "name": "user",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email1@company1.io",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email2@company1.io",
          "dgraph.type": "Person"
        }
      ]
    },
    {
      "cond": "@if(eq(len(u1), 1) AND eq(len(u2), 0) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "uid(u1)",
          "email": "user_email2@company1.io",
          "dgraph.type": "Person"
        }
      ]
    },
    {
      "cond": "@if(eq(len(u1), 1) AND eq(len(u2), 0) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "uid(u2)",
          "email": "user_email1@company1.io",
          "dgraph.type": "Person"
        }
      ]
    },
    {
      "cond": "@if(eq(len(u1), 1) AND eq(len(u2), 1) AND eq(len(u3), 0))",
      "set": [
        {
          "uid": "_:user",
          "name": "user",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email1@company1.io",
          "dgraph.type": "Person"
        },
        {
          "uid": "_:user",
          "email": "user_email2@company1.io",
          "dgraph.type": "Person"
        }
      ],
      "delete": [
        {
          "uid": "uid(u1)",
          "name": null,
          "email": null
        },
        {
          "uid": "uid(u2)",
          "name": null,
          "email": null
        }
      ]
    }
  ]
}' | jq
```
