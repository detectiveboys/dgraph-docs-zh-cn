
## 实现方案

**Note** *所有的mutations和query都在事务的上下文中运行。这与v0.9版本之前的交互模型有很大不同。*

客户端可以通过两种不同的方式与服务器通信：

- **通过 [gRPC](http://www.grpc.io/)**。在内部，它使用[Protocol Buffers](https://developers.google.com/protocol-buffers)（Graph使用的proto文件位于[api.proto](https://github.com/dgraph-io/dgo/blob/master/protos/api.proto)）。

- **通过 HTTP**。有各种端点，每个端点都接收并返回JSON。 HTTP端点和gRPC服务方法之间存在一对一的对应关系。

可以通过gRPC或HTTP直接与Dgraph连接。但是，如果您使用的语言存在客户端库，则会更容易。

**Tip**
*对于多节点，谓词将分配给首先看到该谓词的组。Dgraph还自动将谓词数据移动到不同的组，以平衡谓词分布。每10分钟自动移动一次。客户端可以通过与所有Dgraph实例通信来辅助这个过程。对于Go客户端，这意味着向每个Dgraph实例传入一个`grpc.ClientConn`。 Mutations将以循环方式进行，导致初始时谓词的半随机谓词分布。*

## Go

[![GoDoc](https://godoc.org/github.com/dgraph-io/dgo?status.svg)](https://godoc.org/github.com/dgraph-io/dgo)

go客户端在grpc端口上与服务器通信（默认端口9080）。

客户端可以通过`go get`以通常的方式获得：

```sh
go get -u -v github.com/dgraph-io/dgo
```

完整的[GoDoc](https://godoc.org/github.com/dgraph-io/dgo)包含客户端API的文档以及如何使用它的示例。

### 创建客户端

要创建客户端，请连接Dgraph外部Grpc端口（通常为9080）。以下代码段仅显示一个连接。您可以连接到多个Dgraph alpha节点以均匀分配工作负载。

```go
func newClient() *dgo.Dgraph {
	//拨打gRPC连接。设置dgraph集群时要拨打的地址。
	d, err := grpc.Dial("localhost:9080", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}

	return dgo.NewDgraphClient(
		api.NewDgraphClient(d),
	)
}
```

### 更改数据

要设置schema，请将其设置在`api.Operation`对象上，然后将其传递给`Alter`方法。

```go
func setup(c *dgo.Dgraph) {
	// 在dgraph中导入schema。 账户名称和余额。
	err := c.Alter(context.Background(), &api.Operation{
		Schema: `
			name: string @index(term) .
			balance: int .
		`,
	})
}
```

`api.Operation`也包含其他字段，包括drop predicate和drop all。如果你希望丢弃所有数据，从一个清空的状态开始，而不关闭实例，那drop all非常有用。

```go
	// 从dgraph实例中删除包括schema在内的所有数据。这对于像这样的小例子很有用，因为它将dgraph置于干净的状态。
	err := c.Alter(context.Background(), &api.Operation{DropAll: true})
```

### 创建事务

DGrave v0.9支持运行分布式的ACID事务。要创建事务，只需调用`c.NewTxn()`. 此操作不会引起网络调用。通常，您需要调用`defer txn.Discard()`以便在出现错误时自动回滚。在`Commit`之后调用`Discard`将是一个空操作。

```go
func runTxn(c *dgo.Dgraph) {
	txn := c.NewTxn()
	defer txn.Discard()
	...
}
```

### 执行query

您可以通过调用`txn.Query`来执行query。响应将包含一个`JSON`字段，该字段具有JSON编码结果。可以通过`json.Unmarshal`将其解构为GO struct。

```go
	// 查询Alice和Bob的余额
	const q = `
		{
			all(func: anyofterms(name, "Alice Bob")) {
				uid
				balance
			}
		}
	`
	resp, err := txn.Query(context.Background(), q)
	if err != nil {
		log.Fatal(err)
	}

	// 在得到余额后，我们必须将它们解码成struct，以便我们能够操作数据。
	var decode struct {
		All []struct {
			Uid     string
			Balance int
		}
	}
	if err := json.Unmarshal(resp.GetJson(), &decode); err != nil {
		log.Fatal(err)
	}
```

### 执行mutation

调用`txn.Mutate`可以执行mutation. 它接收一个`api.Mutation`对象，提供了两种主要的数据设置方式：JSON和RDF N-Quad。 您可以选择方便的方式。

我们将继续使用JSON。您可以修改从查询解析的Go struct，并将它们编组回JSON。

```go
	// 在这两个账户之间转移5美元。
	decode.All[0].Bal += 5
	decode.All[1].Bal -= 5

	out, err := json.Marshal(decode.All)
	if err != nil {
		log.Fatal(err)
	}

	_, err := txn.Mutate(context.Background(), &api.Mutation{SetJson: out})
```

有时，您只想提交mutation，而无需进一步查询。
在这种情况下，您可以在`api.Mutation`中使用`CommitNow`字段来指定必须立即提交mutation。

### 提交事物

完成所有query和mutation后，您可以提交事务。如果无法提交事务，则返回错误。

```go
	// 最后，我们提交事物。如果同时运行的其他事务修改了该事务中修改的相同数据，则将返回错误。
	// 在失败时重试事务是由依赖库使用者决定的。

	err := txn.Commit(context.Background())
```

### 完整例子

这是[GoDoc](https://godoc.org/github.com/dgraph-io/dgo)的一个例子。它展示了如何创建具有名称Alice的节点，同时创建她与其他节点的关系。注意，`loc`谓词属于`geo`类型，可以很容易地编组和解组为Go struct。GoDoc中有很多这样的例子。

```go
type School struct {
	Name string `json:"name,omitempty"`
}

type loc struct {
	Type   string    `json:"type,omitempty"`
	Coords []float64 `json:"coordinates,omitempty"`
}

// 如果没有设置omitempty，那么将为没有明确指定的值创建具有
// 空值的边（对于int/float为0、对于string为""、对于bool为false）

type Person struct {
		Uid      string     `json:"uid,omitempty"`
		Name     string     `json:"name,omitempty"`
		Age      int        `json:"age,omitempty"`
		Dob      *time.Time `json:"dob,omitempty"`
		Married  bool       `json:"married,omitempty"`
		Raw      []byte     `json:"raw_bytes,omitempty"`
		Friends  []Person   `json:"friend,omitempty"`
		Location loc        `json:"loc,omitempty"`
		School   []School   `json:"school,omitempty"`
}

conn, err := grpc.Dial("127.0.0.1:9080", grpc.WithInsecure())
if err != nil {
	log.Fatal("While trying to dial gRPC")
}
defer conn.Close()

dc := api.NewDgraphClient(conn)
dg := dgo.NewDgraphClient(dc)

op := &api.Operation{}
op.Schema = `
	name: string @index(exact) .
	age: int .
	married: bool .
	loc: geo .
	dob: datetime .
`

ctx := context.Background()
err = dg.Alter(ctx, op)
if err != nil {
	log.Fatal(err)
}

dob := time.Date(1980, 01, 01, 23, 0, 0, 0, time.UTC)
// 在设置对象时，如果struct具有Uid，那么在graph中更新它的属性，否则将创建一个新节点。
// 在下面的示例中，为Alice、Bob和Charlie以及school创建了新的节点（因为它们没有Uid）。
p := Person{
	Name:    "Alice",
	Age:     26,
	Married: true,
	Location: loc{
		Type:   "Point",
		Coords: []float64{1.1, 2},
	},
	Dob: &dob,
	Raw: []byte("raw_bytes"),
	Friends: []Person{{
		Name: "Bob",
		Age:  24,
	}, {
		Name: "Charlie",
		Age:  29,
	}},
	School: []School{{
		Name: "Crown Public School",
	}},
}

mu := &api.Mutation{
	CommitNow: true,
}
pb, err := json.Marshal(p)
if err != nil {
	log.Fatal(err)
}

mu.SetJson = pb
assigned, err := dg.NewTxn().Mutate(ctx, mu)
if err != nil {
	log.Fatal(err)
}

// 为创建的节点分配的uid将在resp.AssignedUids map中返回。
variables := map[string]string{"$id": assigned.Uids["blank-0"]}
q := `query Me($id: string){
	me(func: uid($id)) {
		name
		dob
		age
		loc
		raw_bytes
		married
		friend @filter(eq(name, "Bob")){
			name
			age
		}
		school {
			name
		}
	}
}`

resp, err := dg.NewTxn().QueryWithVars(ctx, q, variables)
if err != nil {
	log.Fatal(err)
}

type Root struct {
	Me []Person `json:"me"`
}

var r Root
err = json.Unmarshal(resp.Json, &r)
if err != nil {
	log.Fatal(err)
}
// fmt.Printf("Me: %+v\n", r.Me)
// R.Me 将和我们上面设定的人一样。

fmt.Println(string(resp.Json))
// 输出: {"me":[{"name":"Alice","dob":"1980-01-01T23:00:00Z","age":26,"loc":{"type":"Point","coordinates":[1.1,2]},"raw_bytes":"cmF3X2J5dGVz","married":true,"friend":[{"name":"Bob","age":24}],"school":[{"name":"Crown Public School"}]}]}


```


## Java

官方Java客户端可以在[这里](https://github.com/dgraph-io/dgraph4j)找到，它完全支持Dgraph v1.0.x. 按照[README](https://github.com/dgraph-io/dgraph4j#readme)中的说明进行运行即可。

We also have a [DgraphJavaSample] project, which contains an end-to-end
working example of how to use the Java client.
我们还有一个[DgraphJavaSample](https://github.com/dgraph-io/dgraph4j/tree/master/samples/DgraphJavaSample)项目，其中包含如何使用Java客户端的端到端工作示例。

## Javascript

官方Javascript客户端可以在[这里](https://github.com/dgraph-io/dgraph-js)找到，它完全支持Dgraph v1.0.x. 按照[README](https://github.com/dgraph-io/dgraph-js#readme) 中的说明启动并运行。

We also have a [simple example](https://github.com/dgraph-io/dgraph-js/tree/master/examples/simple)
project, which contains an end-to-end working example of how to use the Javascript client,
for Node.js >= v6.
我们还有一个[simple example](https://github.com/dgraph-io/dgraph-js/tree/master/examples/simple)项目，其中包含如何使用Javascript客户端的端到端工作示例，需要Node.js版本 >= v6。

## Python

官方Python客户端可以在[这里](https://github.com/dgraph-io/pydgraph)找到，它完全支持Dgraph v1.0.x，Python版本>= 2.7和>=3.5。按照[README](https://github.com/dgraph-io/pydgraph#readme)中的说明启动并运行。

我们还有一个[simple example](https://github.com/dgraph-io/pydgraph/tree/master/examples/simple)项目，其中包含如何使用Python客户端的端到端工作示例。

## 非官方Dgraph客户端

**Note** *这些第三方客户由社区提供，Dgraph没有正式支持。*

### C(C#)

- https://github.com/AlexandreDaSilva/DgraphNet
- https://github.com/MichaelJCompton/Dgraph-dotnet

### Elixir

- https://github.com/ospaarmann/exdgraph

## Raw HTTP

 **Warning** *Raw HTTP需要比我们的客户端语言更多的chop。我们写这篇文章是为了帮助您用新语言构建Dgraph客户端。*

可以通过其HTTP端点直接与dgraph交互。这让无法访问gRPC实现的语言也可以构建客户端。

在此处显示的示例中，使用常规命令行工具，例如`curl`和[`jq`](https://stedolan.github.io/jq/)。 然而，真正意图是向其他程序员展示他们如何在HTTP API之上使用他们的语言实现客户端。

与Go客户端示例类似，我们使用银行帐户转帐示例。

### 创建客户端

构建在HTTP API之上的客户端需要在两个不同级别跟踪状态：

1. 每个客户端。所有客户端都需要保持一个线性化的读取（`lin_read`）map。这是一个从group id到proposal id的map。整个系统（客户端 + 服务器）都需要[线性化](https://en.wikipedia.org/wiki/Linearizability)。每当在服务器响应（*for any transaction*）中接收到`lin_read`map时，客户端应及时通过合并两个map来更新版本。合并操作很简单 - 新map从父map中获取所有key/value。如果两个map中有相同的key，则采用值最大的。 客户端的初始`lin_read`应该是一个空map。

2. 每个事物。每个事务需要维护三个状态。
    1. 所有事务都需要自己的`lin_read`（独立于客户端级别的`lin_read`更新）。所有在服务器相应*associated with the transaction*中收到的`lin_read`map应该合并到事务`lin_read`map中。
  
    2. 开始时间戳（`start_ts`）。唯一地标识了一个事务，并且不会在事务生命周期中进行更改。
  
    3. 由事务修改的键集（`keys`）。有助于事务冲突检测。

**Note** *在没有多副本的dgraph设置中，不需要跟踪`lin_read`。在从dgraph收到的相应中可以忽略它，并且不需要在任何请求中发送它。*

### 更改数据库

`/alter`端点用于创建或更改schema。这里，谓词`name`是帐户的名称。它被编入索引，以便我们可以根据他们的名字查找帐户。

```sh
curl -X POST localhost:8080/alter -d 'name: string @index(term) .'
```

成功的话，响应应为`{"code":"Success","message":"Done"}`.

其他操作也可以通过`/alter`端点执行。可以删除特定谓词或整个数据库。

例如，删除谓词`name`：

```sh
curl -X POST localhost:8080/alter -d '{"drop_attr": "name"}'
```

要删除所有数据和schema：

```sh
curl -X POST localhost:8080/alter -d '{"drop_all": true}'
```

### 开始事物

假设已填充了一些具有余额的初始帐户。我们现在想把钱从一个帐户转移到另一个帐户。需要分四步完成：

1. 创建一个事物。

2. 在事务内部，运行query以确定当前余额。

3. 执行mutation以更新余额。

4. 提交事物。

启动事务不需要与dgraph本身进行任何交互。为要使用的事务设置某些状态。事务的`lin_read`通过拷贝客户端的`lin_read`初始化。`start_ts`最初可以设为0.`key`设为空集合。

**对于query和mutation，如果提供`start_ts`作为路径参数，则该操作作为正在进行的事务的一部分执行，否则启动新事务。**

### 执行query

要查询数据库，使用`/query`端点。我们需要使用事务作用域`lin_read`。假设`lin_read`是`{"1": 12}`。

获取两个帐户的余额：

```graphql
curl -X POST -H 'X-Dgraph-LinRead: {"1": 12}' localhost:8080/query -d $'
{
  balances(func: anyofterms(name, "Alice Bob")) {
    uid
    name
    balance
  }
}' | jq

```

结果应该是这样的：

```json
{
  "data": {
    "balances": [
      {
        "uid": "0x1",
        "name": "Alice",
        "balance": "100"
      },
      {
        "uid": "0x2",
        "name": "Bob",
        "balance": "70"
      }
    ]
  },
  "extensions": {
    "server_latency": {
      "parsing_ns": 70494,
      "processing_ns": 697140,
      "encoding_ns": 1560151
    },
    "txn": {
      "start_ts": 4,
      "lin_read": {
        "ids": {
          "1": 14
        }
      }
    }
  }
}
```

请注意，除了`data`字段下的查询结果，`extensions -> txn`字段中还有一些额外的数据。客户必须追踪此数据。

首先，响应中有一个`start_ts`。这个`start_ts`将需要在与该事务所有与dgraph后续交互中使用，因此应该成为事务状态的一部分。

其次，有一个新的`lin_read`map。 `lin_read`map应该与客户端范围和事务范围的`lin_read`map合并。回想一下，事务作用域和客户端作用域的`lin_read`映射都是`{"1": 12}`。响应中的`lin_read`是`{"1": 14}`。合并的结果是`{"1": 14}`，因为我们取所有key中值最大的。

### 执行mutation

现在我们有了当前的余额，我们需要使用更新的余额向dgraph发送mutation。如果Bob向Alice转账10美元，那么要发送的RDF是：

```sh
<0x1> <balance> "110" .
<0x2> <balance> "60" .
```

注意，我们必须用RDF格式中的UID引用Alice和Bob节点。

我们现在通过`/mutate`端点发送mutation。我们需要提供事务开始时间戳作为路径参数，以便dgraph知道mutation应该属于哪个事务。

```sh
curl -X POST localhost:8080/mutate/4 -d $'
{
  set {
    <0x1> <balance> "110" .
    <0x2> <balance> "60" .
  }
}
' | jq
```

返回结果:

```json
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {}
  },
  "extensions": {
    "txn": {
      "start_ts": 4,
      "keys": [
        "AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAI=",
        "AAAHYmFsYW5jZQAAAAAAAAAAAg==",
        "AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAE=",
        "AAAHYmFsYW5jZQAAAAAAAAAAAQ=="
      ],
      "lin_read": {
        "ids": {
          "1": 17
        }
      }
    }
  }
}
```

我们得到需要合并（对于**both the client and transaction**的新`lin_read`map变为`{"1": 17}`）的另一个`lin_read`map。我们也得到一些`keys`。这些`keys`应该添加到存储在事务状态中的`keys`集合中。

### 提交事物

**Note** *在发生mutation后可以立即提交（不需要使用本节中讲解的`/commit`端点）。 为了能这样做，需要将`X-Dgraph-CommitNow: true`请求头添加到最终的`/mutate`调用中。*

最后，我们可以使用`/commit`端点提交事务。需要我们一直用于事务`start_ts`以及`keys`。如果我们在事务中执行了多个mutation，那么在提交期间提供的`keys`将是来自`/mutate`端点的响应中返回的所有`keys`的并集。


```sh
curl -X POST localhost:8080/commit/4 -d $'
  [
    "AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAI=",
    "AAAHYmFsYW5jZQAAAAAAAAAAAg==",
    "AAALX3ByZWRpY2F0ZV8AAAAAAAAAAAE=",
    "AAAHYmFsYW5jZQAAAAAAAAAAAQ=="
  ]' | jq
```

```json
{
  "data": {
    "code": "Success",
    "message": "Done"
  },
  "extensions": {
    "txn": {
      "start_ts": 4,
      "commit_ts": 5
    }
  }
}
```

事物现已完成。

如果另一个客户端同时执行影响相同`keys`的另一个事务，则该事务可能*不会*成功。会在提交的响应中表明。

```json
{
  "errors": [
    {
      "code": "Error",
      "message": "Transaction aborted"
    }
  ]
}
```

在这种情况下，应该由客户端的用户决定他们是否希望重试事务。
