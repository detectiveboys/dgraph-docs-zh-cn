
## 实现方案

*所有的mutations和query都在事务的上下文中运行。这与v0.9版本之前的交互模型有很大不同。*

客户端可以通过两种不同的方式与服务器通信：

- **通过 [gRPC](http://www.grpc.io/)。** 在内部，它使用[Protocol Buffers](https://developers.google.com/protocol-buffers)（Graph使用的proto文件位于[api.proto](https://github.com/dgraph-io/dgo/blob/master/protos/api.proto)）。

- **通过 HTTP。** 有各种端点，每个端点都接收并返回JSON。 HTTP端点和gRPC服务方法之间存在一对一的对应关系。

可以通过gRPC或HTTP直接与Dgraph连接。但是，如果您使用的语言存在客户端库，则会更容易。

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

The official Java client [can be found here](https://github.com/dgraph-io/dgraph4j)
and it fully supports Dgraph v1.0.x. Follow the instructions in the
[README](https://github.com/dgraph-io/dgraph4j#readme)
to get it up and running.

We also have a [DgraphJavaSample] project, which contains an end-to-end
working example of how to use the Java client.

[DgraphJavaSample]:https://github.com/dgraph-io/dgraph4j/tree/master/samples/DgraphJavaSample

## Javascript

The official Javascript client [can be found here](https://github.com/dgraph-io/dgraph-js)
and it fully supports Dgraph v1.0.x. Follow the instructions in the
[README](https://github.com/dgraph-io/dgraph-js#readme) to get it up and running.

We also have a [simple example](https://github.com/dgraph-io/dgraph-js/tree/master/examples/simple)
project, which contains an end-to-end working example of how to use the Javascript client,
for Node.js >= v6.

## Python

The official Python client [can be found here](https://github.com/dgraph-io/pydgraph)
and it fully supports Dgraph v1.0.x and Python versions >= 2.7 and >= 3.5. Follow the
instructions in the [README](https://github.com/dgraph-io/pydgraph#readme) to get it
up and running.

We also have a [simple example](https://github.com/dgraph-io/pydgraph/tree/master/examples/simple)
project, which contains an end-to-end working example of how to use the Python client.

## Unofficial Dgraph Clients

{{% notice "note" %}}
These third-party clients are contributed by the community and are not officially supported by Dgraph.
{{% /notice %}}

### C\#

- https://github.com/AlexandreDaSilva/DgraphNet
- https://github.com/MichaelJCompton/Dgraph-dotnet

### Elixir

- https://github.com/ospaarmann/exdgraph

## Raw HTTP

{{% notice "warning" %}}
Raw HTTP needs more chops to use than our language clients. We wrote this to be a
guide to help you build Dgraph client in a new language.
{{% /notice %}}

It's also possible to interact with dgraph directly via its HTTP endpoints.
This allows clients to be built for languages that don't have access to a
working gRPC implementation.

In the examples shown here, regular command line tools such as `curl` and
[`jq`](https://stedolan.github.io/jq/) are used. However, the real intention
here is to show other programmers how they could implement a client in their
language on top of the HTTP API.

Similar to the Go client example, we use a bank account transfer example.

### Create the Client

A client built on top of the HTTP API will need to track state at two different
levels:

1. Per client. Each client will need to keep a linearized reads (`lin_read`)
   map. This is a map from dgraph group id to proposal id. This will be needed
for the system as a whole (client + server) to have
[linearizability](https://en.wikipedia.org/wiki/Linearizability). Whenever a
`lin_read` map is received in a server response (*for any transaction*), the
client should update its version of the map by merging the two maps together.
The merge operation is simple - the new map gets all key/value pairs from the
parent maps. Where a key exists in both maps, the max value is taken. The
client's initial `lin_read` is should be an empty map.

2. Per transaction. There are three pieces of state that need to be maintained
   for each transaction.

    1. Each transaction needs its own `lin_read` (updated independently of the
       client level `lin_read`). Any `lin_read` maps received in server
responses *associated with the transaction* should be merged into the
transactions `lin_read` map.
  
    2. A start timestamp (`start_ts`). This uniquely identifies a transaction,
       and doesn't change over the transaction lifecycle.
  
    3. The set of keys modified by the transaction (`keys`). This aids in
       transaction conflict detection.

{{% notice "note" %}}
On a dgraph set up with no replication, there is no need to track `lin_read`.
It can be ignored in responses received from dgraph and doesn't need to be sent
in any requests.
{{% /notice %}}

### Alter the database

The `/alter` endpoint is used to create or change the schema. Here, the
predicate `name` is the name of an account. It's indexed so that we can look up
accounts based on their name.

```sh
curl -X POST localhost:8080/alter -d 'name: string @index(term) .'
```

If all goes well, the response should be `{"code":"Success","message":"Done"}`.

Other operations can be performed via the `/alter` endpoint as well. A specific
predicate or the entire database can be dropped.

E.g. to drop the predicate `name`:
```sh
curl -X POST localhost:8080/alter -d '{"drop_attr": "name"}'
```
To drop all data and schema:
```sh
curl -X POST localhost:8080/alter -d '{"drop_all": true}'
```

### Start a transaction

Assume some initial accounts with balances have been populated. We now want to
transfer money from one account to the other. This is done in four steps:

1. Create a new transaction.

1. Inside the transaction, run a query to determine the current balances.

2. Perform a mutation to update the balances.

3. Commit the transaction.

Starting a transaction doesn't require any interaction with dgraph itself.
Some state needs to be set up for the transaction to use. The transaction's
`lin_read` is initialized by *copying* the client's `lin_read`. The `start_ts`
can initially be set to 0. `keys` can start as an empty set.

**For both query and mutation if the `start_ts` is provided as a path parameter, then the operation
is performed as part of the ongoing transaction else a new transaction is initiated.**

### Run a query

To query the database, the `/query` endpoint is used. We need to use the
transaction scoped `lin_read`. Assume that `lin_read` is `{"1": 12}`.

To get the balances for both accounts:

```sh
curl -X POST -H 'X-Dgraph-LinRead: {"1": 12}' localhost:8080/query -d $'
{
  balances(func: anyofterms(name, "Alice Bob")) {
    uid
    name
    balance
  }
}' | jq

```

The result should look like this:

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

Notice that along with the query result under the `data` field, there is some
additional data in the `extensions -> txn` field. This data will have to be
tracked by the client.

First, there is a `start_ts` in the response. This `start_ts` will need to be
used in all subsequent interactions with dgraph for this transaction, and so
should become part of the transaction state.

Second, there is a new `lin_read` map. The `lin_read` map should be merged with
both the client scoped and transaction scoped `lin_read` maps. Recall that both
the transaction scoped and client scoped `lin_read` maps are `{"1": 12}`. The
`lin_read` in the response is `{"1": 14}`. The merged result is `{"1": 14}`,
since we take the max all of the keys.

### Run a Mutation

Now that we have the current balances, we need to send a mutation to dgraph
with the updated balances. If Bob transfers $10 to Alice, then the RDFs to send
are:

```
<0x1> <balance> "110" .
<0x2> <balance> "60" .
```
Note that we have to to refer to the Alice and Bob nodes by UID in the RDF
format.

We now send the mutations via the `/mutate` endpoint. We need to provide our
transaction start timestamp as a path parameter, so that dgraph knows which
transaction the mutation should be part of.

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

The result:

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

We get another `lin_read` map, which needs to be merged (the new `lin_read` map
for **both the client and transaction** becomes `{"1": 17}`). We also get some
`keys`. These should be added to the set of `keys` stored in the transaction
state.

### Committing the transaction

{{% notice "note" %}}
It's possible to commit immediately after a mutation is made (without requiring
to use the `/commit` endpoint as explained in this section). To do this, add
the `X-Dgraph-CommitNow: true` header to the final `/mutate` call.
{{% /notice %}}

Finally, we can commit the transaction using the `/commit` endpoint. We need
the `start_ts` we've been using for the transaction along with the `keys`.
If we had performed multiple mutations in the transaction instead of the just
the one, then the keys provided during the commit would be the union of all
keys returned in the responses from the `/mutate` endpoint.

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
The transaction is now complete.

If another client were to perform another transaction concurrently affecting
the same keys, then it's possible that the transaction would *not* be
successful.  This is indicated in the response when the commit is attempted.

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

In this case, it should be up to the user of the client to decide if they wish
to retry the transaction.
