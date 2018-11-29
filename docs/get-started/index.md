
## Dgraph

Dgraph集群由不同的节点（Zero、Alpha、Ratel）组成，每个节点有不同的用途。

**Dgraph Zero** 控制Dgraph集群，将服务器分组，并在服务器组之间重新平衡数据。

**Dgraph Alpha** 托管谓词和索引。

**Dgraph Ratel** 为UI提供运行query，mutations和alter schema的功能。

您需要至少一个Dgraph Zero节点和一个Dgraph Alpha节点才能开始使用。

**这是一个3步教程，可以让您启动并运行Dgraph**

这是运行Dgraph的快速入门指南。 如需互动式指引，请参与[tour](https://tour.dgraph.io).

您可以在[此处](https://www.youtube.com/watch?v=QIIdSp2zLcs)查看随附的视频.

## Step 1: 安装Dgraph

Dgraph可以从安装脚本安装，也可以通过Docker运行。

### Docker镜像安装

从[这里](https://hub.docker.com/r/dgraph/dgraph/)拉取Dgraph镜像. 在终端执行:

```sh
docker pull dgraph/dgraph
```

### 脚本安装(Linux/Mac)

安装二进制文件

```sh
curl https://get.dgraph.io -sSf | bash
```

该脚本自动安装Dgraph。 完成后，直接跳到[step 2](#step-2-运行Dgraph).

**替代方案：**为了降低潜在的安全风险，请尝试：

```sh
curl https://get.dgraph.io > /tmp/get.sh
vim /tmp/get.sh  # Inspect the script
sh /tmp/get.sh   # Execute the script
```

您可以通过运行`dgraph`命令来检查Dgraph二进制文件是否正确安装
查看其输出，其中包括版本号。

### 在Windows系统安装

*注意：Windows的二进制文件可从`v0.8.3`版本获得。*

如果您希望在Windows上安装二进制文件，可以从[Github releases](https://github.com/dgraph-io/dgraph/releases)中获取, 手动提取并安装。 `dgraph-windows-amd64-v1.x.y.tar.gz` 中包含dgraph二进制文件.

## Step 2: 运行Dgraph

*这里只涉及一台机器的设置。对于多服务器设置，请转到[部署](/deploy/index)*

### Docker Compose

启动并运行Dgraph的最简单方法是使用Docker Compose。 如果您还没有安装Docker Compose，请按照[此处](https://docs.docker.com/compose/install/)的说明进行安装。

```yaml
version: "3.2"
services:
  zero:
    image: dgraph/dgraph:latest
    volumes:
      - type: volume
        source: dgraph
        target: /dgraph
        volume:
          nocopy: true
    ports:
      - 5080:5080
      - 6080:6080
    restart: on-failure
    command: dgraph zero --my=zero:5080
  server:
    image: dgraph/dgraph:latest
    volumes:
      - type: volume
        source: dgraph
        target: /dgraph
        volume:
          nocopy: true
    ports:
      - 8080:8080
      - 9080:9080
    restart: on-failure
    command: dgraph alpha --my=server:7080 --lru_mb=2048 --zero=zero:5080
  ratel:
    image: dgraph/dgraph:latest
    volumes:
      - type: volume
        source: dgraph
        target: /dgraph
        volume:
          nocopy: true
    ports:
      - 8000:8000
    command: dgraph-ratel

volumes:
  dgraph:
```

将上面的代码段的内容保存在名为`docker-compose.yml`的文件中，然后从包含该文件的文件夹中运行以下命令。

```sh
docker-compose up -d
```

这将启动Dgraph Alpha，Zero和Ratel。 您可以使用`docker-compose logs`命令查看日志。

### 二进制安装

**运行 Dgraph zero 节点**

执行`dgraph zero`启动Dgraph zero节点。 此过程控制Dgraph集群、维护成员信息、碎片分配和碎片移动等。

```sh
dgraph zero
```

**运行 Dgraph alpha 节点**

执行 `dgraph alpha` 命令启动 Dgraph alpha 节点.

```sh
dgraph alpha --lru_mb 2048 --zero localhost:5080
```

**运行 Ratel 节点**

执行 `dgraph-ratel` 命令启动Dgraph用户界面. 可以通过Dgraph用户界面做mutations和query.

```sh
dgraph-ratel
```

*你可以通过`lru_mb`字段设置Dgraph alpha占用的内存。 这只是对Dgraph alpha的示意，实际使用率会高于此值。建议将lru_mb设置为可用内存的三分之一。*

### Linux上的docker方式安装

```sh
# Directory to store data in. This would be passed to `-v` flag.
mkdir -p /tmp/data

# Run Dgraph Zero
docker run -it -p 5080:5080 -p 6080:6080 -p 8080:8080 -p 9080:9080 -p 8000:8000 -v /tmp/data:/dgraph --name diggy dgraph/dgraph dgraph zero

# Run Dgraph Alpha
docker exec -it diggy dgraph alpha --lru_mb 2048 --zero localhost:5080

# Run Dgraph Ratel
docker exec -it diggy dgraph-ratel
```

dgraph alpha监听端口8080和9080，并将日志输出到终端。

### 非Linux发行版上的docker方式安装

使用docker时，挂载的文件系统中的文件访问速度较慢。尝试在安装卷上运行命令`time dd if = / dev / zero of = test.dat bs = 1024 count = 100000`，您会注意到使用安装卷时速度非常慢。我们建议用户使用docker数据卷。 使用数据卷的唯一缺点是您无法从主机访问文件，您必须启动容器才能访问它。

*如果您在非linux发行版上使用docker，请使用docker数据卷。*

用dgraph/dgraph镜像创建一个名为*data*的数据容器。

```sh
docker create -v /dgraph --name data dgraph/dgraph
```

现在，如果我们使用`--volumes-from`参数启动Dgraph容器并使用以下命令运行Dgraph，那么我们写入Dgraph容器中/dgraph目录的任何内容都将被写入数据容器的/dgraph目录。

```sh
docker run -it -p 5080:5080 -p 6080:6080 --volumes-from data --name diggy dgraph/dgraph dgraph zero
docker exec -it diggy dgraph alpha --lru_mb 2048 --zero localhost:5080

# Run Dgraph Ratel
docker exec -it diggy dgraph-ratel
```

*If you are using Dgraph v1.0.2 (or older) then the default ports are 7080, 8080 for zero, so when following instructions for different setup guides override zero port using `--port_offset`.*

```sh
dgraph zero --lru_mb=<typically one-third the RAM> --port_offset -2000
```
Ratel's default port is 8081, so override it using -p 8000.

{{% /notice %}}


## Step 3: Run Queries
{{% notice "tip" %}}Once Dgraph is running, you can access Ratel at [`http://localhost:8000`](http://localhost:8000). It allows browser-based queries, mutations and visualizations.

The mutations and queries below can either be run from the command line using `curl localhost:8080/query -XPOST -d $'...'` or by pasting everything between the two `'` into the running user interface on localhost.{{% /notice %}}

### Dataset
The dataset is a movie graph, where and the graph nodes are entities of the type directors, actors, genres, or movies.

### Storing data in the graph
Changing the data stored in Dgraph is a mutation.  The following mutation stores information about the first three releases of the the ''Star Wars'' series and one of the ''Star Trek'' movies.  Running this mutation, either through the UI or on the command line, will store the data in Dgraph.


```sh
curl localhost:8080/mutate -H "X-Dgraph-CommitNow: true" -XPOST -d $'
{
  set {
   _:luke <name> "Luke Skywalker" .
   _:leia <name> "Princess Leia" .
   _:han <name> "Han Solo" .
   _:lucas <name> "George Lucas" .
   _:irvin <name> "Irvin Kernshner" .
   _:richard <name> "Richard Marquand" .

   _:sw1 <name> "Star Wars: Episode IV - A New Hope" .
   _:sw1 <release_date> "1977-05-25" .
   _:sw1 <revenue> "775000000" .
   _:sw1 <running_time> "121" .
   _:sw1 <starring> _:luke .
   _:sw1 <starring> _:leia .
   _:sw1 <starring> _:han .
   _:sw1 <director> _:lucas .

   _:sw2 <name> "Star Wars: Episode V - The Empire Strikes Back" .
   _:sw2 <release_date> "1980-05-21" .
   _:sw2 <revenue> "534000000" .
   _:sw2 <running_time> "124" .
   _:sw2 <starring> _:luke .
   _:sw2 <starring> _:leia .
   _:sw2 <starring> _:han .
   _:sw2 <director> _:irvin .

   _:sw3 <name> "Star Wars: Episode VI - Return of the Jedi" .
   _:sw3 <release_date> "1983-05-25" .
   _:sw3 <revenue> "572000000" .
   _:sw3 <running_time> "131" .
   _:sw3 <starring> _:luke .
   _:sw3 <starring> _:leia .
   _:sw3 <starring> _:han .
   _:sw3 <director> _:richard .

   _:st1 <name> "Star Trek: The Motion Picture" .
   _:st1 <release_date> "1979-12-07" .
   _:st1 <revenue> "139000000" .
   _:st1 <running_time> "132" .
  }
}
' | python -m json.tool | less
```

{{% notice "tip" %}}
To run an RDF mutation via curl, you can use the curl option `--data-binary @/path/to/mutation.txt` instead of `-d $''`. The `--data-binary` option skips curl's default URL-encoding.
{{% /notice %}}

### Adding indexes
Alter the schema to add indexes on some of the data so queries can use term matching, filtering and sorting.

```sh
curl localhost:8080/alter -XPOST -d $'
  name: string @index(term) .
  release_date: datetime @index(year) .
  revenue: float .
  running_time: int .
' | python -m json.tool | less
```

### Get all movies
Run this query to get all the movies. The query works below all the movies have a starring edge

```sh
curl localhost:8080/query -XPOST -d $'
{
 me(func: has(starring)) {
   name
  }
}
' | python -m json.tool | less
```

### Get all movies released after "1980"
Run this query to get "Star Wars" movies released after "1980".  Try it in the user interface to see the result as a graph.


```sh
curl localhost:8080/query -XPOST -d $'
{
  me(func:allofterms(name, "Star Wars")) @filter(ge(release_date, "1980")) {
    name
    release_date
    revenue
    running_time
    director {
     name
    }
    starring {
     name
    }
  }
}
' | python -m json.tool | less
```

Output

```json
{
  "data":{
    "me":[
      {
        "name":"Star Wars: Episode V - The Empire Strikes Back",
        "release_date":"1980-05-21T00:00:00Z",
        "revenue":534000000.0,
        "running_time":124,
        "director":[
          {
            "name":"Irvin Kernshner"
          }
        ],
        "starring":[
          {
            "name":"Han Solo"
          },
          {
            "name":"Luke Skywalker"
          },
          {
            "name":"Princess Leia"
          }
        ]
      },
      {
        "name":"Star Wars: Episode VI - Return of the Jedi",
        "release_date":"1983-05-25T00:00:00Z",
        "revenue":572000000.0,
        "running_time":131,
        "director":[
          {
            "name":"Richard Marquand"
          }
        ],
        "starring":[
          {
            "name":"Han Solo"
          },
          {
            "name":"Luke Skywalker"
          },
          {
            "name":"Princess Leia"
          }
        ]
      }
    ]
  }
}
```

That's it! In these three steps, we set up Dgraph, added some data, set a schema
and queried that data back.

## Where to go from here

- Go to [Clients]({{< relref "clients/index.md" >}}) to see how to communicate
with Dgraph from your application.
- Take the [Tour](https://tour.dgraph.io) for a guided tour of how to write queries in Dgraph.
- A wider range of queries can also be found in the [Query Language](/query-language) reference.
- See [Deploy](/deploy) if you wish to run Dgraph
  in a cluster.

## Need Help

* Please use [discuss.dgraph.io](https://discuss.dgraph.io) for questions, feature requests and discussions.
* Please use [Github Issues](https://github.com/dgraph-io/dgraph/issues) if you encounter bugs or have feature requests.
* You can also join our [Slack channel](http://slack.dgraph.io).

## Troubleshooting

### 1. Docker: Error response from daemon; Conflict. Container name already exists.

Remove the diggy container and try the docker run command again.
```
docker rm diggy
```
