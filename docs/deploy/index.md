
本页讨论以分布式方式在各种部署模式下运行Dgraph，并涉及在集群中的多个服务器上运行Dgraph的多个实例。

**提示** *对于单个服务器设置，建议新用户使用，请参阅[快速开始](/get-started/index)页。*

## 安装Dgraph

#### Docker

```sh
docker pull dgraph/dgraph:latest

# 测试它运行是否良好，运行：
docker run -it dgraph/dgraph:latest dgraph
```

#### 自动下载

执行

```sh
curl https://get.dgraph.io -sSf | bash

# 测试它运行是否良好，运行：
dgraph
```

将`dgraph`二进制文件安装到您的系统中。

#### 手动下载[可选]

如果您不想遵循自动安装方法，可以从 **[Dgraph releases](https://github.com/dgraph-io/dgraph/releases)** 手动下载适用于您平台的tar包。 从Github下载适用于您的平台的tar包后，将二进制文件解压缩到`/usr/local/bin`目录下。

```sh
# 对于Linux系统
$ sudo tar -C /usr/local/bin -xzf dgraph-linux-amd64-VERSION.tar.gz

# 对于Mac系统
$ sudo tar -C /usr/local/bin -xzf dgraph-darwin-amd64-VERSION.tar.gz

# 测试它运行是否良好，运行：
dgraph
```

#### 从源码构建

**注意** *Ratel UI现在是闭源的，因此您无法从源代码构建它。但您可以通过使用上面列出的任何方法安装的Ratel UI连接到Dgraph实例。*

Make sure you have [Go](https://golang.org/dl/) (version >= 1.8) installed.
确保安装了[Go](https://golang.org/dl/)（版本> = 1.8）。

安装Go后，运行

```sh
# 你的$GOPATH/bin中安装dgraph二进制文件。

go get -u -v github.com/dgraph-io/dgraph/dgraph
```

If you get errors related to `grpc` while building them, your `go-grpc` version might be outdated. We don't vendor in `go-grpc`(because it causes issues while using the Go client). Update your `go-grpc` by running.
如果在构建它们时遇到与`grpc`相关的错误，那么你的`go-grpc`版本可能已经过时了。 我们不在`go-grpc`中供应（因为它在使用Go客户端时会引起问题）。通过运行一下命令更新你的`go-grpc`。

```sh
go get -u -v google.golang.org/grpc
```

#### 配置

可以通过使用 `--help` 标志调用dgraph来查看dgraph的配置选项的完整集合（以及简要描述）。例如，为了查看`dgraph alpha`可用的选项，运行`dgraph alpha --help`。

可以通过多种方式配置选项（从最高优先级到最低优先级）：

- 使用命令行标志（如帮助输出中所述）。

- 使用环境变量。

- 使用配置文件。

如果没有使用选项的配置，`--help`输出默认值。

可以同时使用多种配置方法。例如，可以在配置文件中设置一组核心选项，同时使用环境变量或命令行标志设置特定于实例的选项。

环境变量名称与标志名称相同，如`--help`输出中所示。
它们是`DGRAPH`连接，调用的子命令（`ALPHA`，`ZERO`，`LIVE`或`BULK`），然后是标志的名称（大写）。 例如，您可以使用`DGRAPH_ALPHA_LRU_MB = 8096 dgraph alpha`代替`dgraph alpha --lru_mb = 8096`。

支持的配置文件格式是JSON，TOML，YAML，HCL和Java properties（通过文件扩展名检测）。

可以使用`--config`标志或环境变量指定配置文件。例如。 `dgraph zero --config my_config.json`或`DGRAPH_ZERO_CONFIG = my_config.json dgraph zero`。

配置文件结构只是简单的键值对（与标志名称相同）。 例如。一个JSON配置文件，设置`--idx`，`--peer`和`--replicas`：

```json
{
  "idx": 42,
  "peer": 192.168.0.55:9080,
  "replicas": 2
}
```

## 集群设置

### 了解Dgraph集群

Dgraph是一个真正的分布式图形数据库 - 而不是通用数据集的主从复制。 它通过谓词进行分片并在群集中复制谓词，查询可以在任何节点上运行，连接通过分布式数据进行处理。对于节点存储的谓词本地解析查询，并通过分布式连接解析存储在其他节点上的谓词。

为了有效地运行Dgraph集群，了解分片，复制和重新平衡的工作原理非常重要。

**分片**

Dgraph为每个谓词编写数据（* P *，在RDF术语中），因此最小的数据单位是一个谓词。要对graph进行分片，请将一个或多个谓词分配为一组。群集中的每个Alpha节点都服务于一个组。 Dgraph Zero为每个Alpha节点分配一个组。

**重新平衡分片**

Dgraph Zero尝试根据每个组中的磁盘使用情况重新平衡群集。如果Zero节点检测到不平衡，它将尝试将谓词与索引和反向边一起移动到具有最小磁盘使用量的组。这使谓词暂时不可用。

Zero节点会不断尝试保持每台服务器上的数据量，通常以10分钟的频率运行此检查。因此，每个额外的Dgraph Alpha实例将允许Zero进一步从组中拆分谓词并将它们移动到新节点。

**相合复制**

如果`--replicas`标志设置为大于1的值，则Zero节点会将同一组分配给多个节点。 然后，这些节点将形成一个Raft组，即quorum。每次写入都会相合复制到quorum。 为了达成共识，quorum的数量要是奇数。 因此，我们建议将`--replicas`设置为1,3或5（不是2或4）。这允许服务于同一组的0,1或2个节点分别关闭，而不会影响该组的整体健康状况。

## 端口使用

Dgraph集群节点使用不同的端口通过GRPC和HTTP进行通信。由于每个端口需要不同的访问安全规则或防火墙，因此用户在选择这些端口时必须注意它们的拓扑结构和部署模式。

### 端口类型

- **gRPC-internal:** 在集群节点之间用于内部通信和消息交换的端口。
- **gRPC-external:** Dgraph客户端，Dgraph Live Loader和Dgraph Bulk loader用于通过gRPC访问API的端口。
- **http-external:** 客户端用于通过HTTP与其他监控和管理任务访问API的端口。

### 不同节点使用的端口

 Dgraph节点类型 | gRPC-内部  | gRPC-外部 | HTTP-外部
------------------|----------------|---------------|---------------
       zero       |  --Not Used--  |     5080      |     6080
       alpha      |      7080      |     9080      |     8080
       ratel      |  --Not Used--  | --Not Used--  |     8000

用户必须根据其底层网络修改安全规则或打开防火墙，以允许群集节点之间以及服务器和客户端之间的通信。 在开发期间，一般*-external（gRPC/HTTP）端口规则可以是公开的，gRPC-internal端口在集群节点内是开放的。

**Ratel UI** 访问http-external端口上的Dgraph Alpha 节点（默认localhost:8080），并且可以配置为与远程Dgraph集群通信。这样，您可以在本地计算机上运行Ratel并指向远程群集。但是如果你要将Ratel和Dgraph集群一起部署，那么你可能不得不向公众公开8000。

**Port Offset** 为了方便用户设置集群，Dgraph各节点使用默认的端口，并允许用户提供偏移量（通过命令选项`--port_offset`）来定义节点使用的实际端口。 在高可用集群设置中启动多个zero节点时也可以使用偏移。

例如，当用户通过设置`--port_offset 2`运行Dgraph Alpha 节点时，Alpha节点分别绑定到7082（gRPC-internal），8082（http-external）和9092（gRPC-external）。

**Ratel UI** 默认情况下监听端口8000.您可以使用`-port`标志配置为监听任何其他端口。

**提示**

**对于Dgraph v1.0.2 (或这之后版本)**

Zero 节点的默认端口为7080和8080.当按照以下不同设置指南的说明操作时，使用`--port_offset`覆盖Zero端口以匹配当前的默认端口。

```sh
# 使用端口5080和6080运行Zero
dgraph zero --idx=1 --port_offset -2000
# 使用端口5081和6081运行Zero
dgraph zero --idx=2 --port_offset -1999
```

同样，Ratel的默认端口是8081，因此使用`--port`将其覆盖到当前默认端口。

```sh
dgraph-ratel --port 8080
```

### 高可用集群设置

在高可用设置中，我们需要为Zero节点运行3或5个副本，同样地，为Alpha节点运行3或5个副本。

**注意** *如果副本数量为2K+1，则最多 **K个服务器** 可以关闭而不会对读取或写入产生任何影响。*
*避免将副本保持为2K（偶数）。 如果K个服务器出现故障，由于缺乏共识，这会阻止读写操作。*

**Dgraph Zero 节点**
运行三个Zero实例，通过`--idx`标志为每个实例分配一个唯一的ID（整数），并通过`--peer`标志传递健康的Zero实例的地址。

要为alpha节点运行三个副本，请设置`--replicas = 3`。每次添加新的Dgraph Alpha 节点时，Zero 节点都会检查现有的组并将它们分配给一个没有三个副本的组。

**Dgraph Alpha 节点**
根据需要运行尽可能多的Dgraph Alpha 节点。您可以手动设置`--idx`标志，或者您可以将该标志留空，Zero节点会自动为Alpha节点分配一个ID。此ID将在写前日志中保留，因此请注意不要删除它。

新的Alpha节点将通过与Zero节点通信来自动检测彼此，并建立相互连接。

Typically, Zero would first attempt to replicate a group, by assigning a new
Dgraph alpha to run the same group as assigned to another. Once the group has
been replicated as per the `--replicas` flag, Zero would create a new group.
通常，Zero节点会首先尝试复制一个组，方法是指定一个新的Alpha节点去运行指定的另一个相同的组。一旦该组按照`--replicas`标志复制，Zero节点将创建一个新组。

随着时间的推移，数据将在所有组中均匀分配。因此，确保Dgraph alpha节点的数量是replicas设置的倍数非常重要。 例如，如果在Zero节点中设置`--replicas = 3`，则运行三个Dgraph alpha节点，不进行分片，而是3x复制。 运行六个Dgraph alphas，将数据分成两组，复制3次。

## 单主机设置

### 直接在主机上运行

**运行 Dgraph Zero 节点**

```sh
dgraph zero --my=IPADDR:5080
```

`--my`标志是alpha节点与zero节点通话的连接。因此，端口“5080”和IP地址必须对所有Dgraph alpha都可见。

对于所有其他各种标志，运行`dgraph zero --help`。

**运行 Dgraph Alpha 节点**

```sh
dgraph alpha --lru_mb=<typically one-third the RAM> --my=IPADDR:7080 --zero=localhost:5080
dgraph alpha --lru_mb=<typically one-third the RAM> --my=IPADDR:7081 --zero=localhost:5080 -o=1
```

注意第二个Alpha使用`-o`来为使用的默认端口添加偏移量。Zero会自动为每个Alpha分配一个唯一ID，该ID保留在写入日志（wal）目录中，用户可以使用`--idx`选项指定索引。Dgraph Alphas使用两个目录来保存数据和wal日志，如果每个Alpha在同一主机上运行，则这些目录必须不同。 您可以使用`-p`和`-w`来更改数据和WAL目录的位置。

对于所有其他标志，运行`dgraph alpha --help`。

**运行 dgraph UI**

```sh
dgraph-ratel
```

### 使用Docker运行

可以将Dgraph集群设置为在单个主机上作为容器运行。首先，您需要确定主机IP地址。你通常可以通过

```sh
ip addr  # Arch Linux系统上
ifconfig # Ubuntu/Mac系统上
```

我们将通过`HOSTIPADDR`来引用主机IP地址。

**运行 Dgraph Zero**

```sh
mkdir ~/zero # 或存储数据的任何其他目录。

docker run -it -p 5080:5080 -p 6080:6080 -v ~/zero:/dgraph dgraph/dgraph:latest dgraph zero --my=HOSTIPADDR:5080
```

**运行 Dgraph Alpha**

```sh
mkdir ~/server1 # 或存储数据的任何其他目录。

docker run -it -p 7080:7080 -p 8080:8080 -p 9080:9080 -v ~/server1:/dgraph dgraph/dgraph:latest dgraph alpha --lru_mb=<typically one-third the RAM> --zero=HOSTIPADDR:5080 --my=HOSTIPADDR:7080

mkdir ~/server2 # 或存储数据的任何其他目录。

docker run -it -p 7081:7081 -p 8081:8081 -p 9081:9081 -v ~/server2:/dgraph dgraph/dgraph:latest dgraph alpha --lru_mb=<typically one-third the RAM> --zero=HOSTIPADDR:5080 --my=HOSTIPADDR:7081  -o=1
```

注意，server2使用-o覆盖server2的默认端口。

**运行 dgraph UI**

```sh
docker run -it -p 8000:8000 dgraph/dgraph:latest dgraph-ratel
```

### 使用Docker Compose运行（在单个AWS实例上）

我们将使用[Docker Machine](https://docs.docker.com/machine/overview/)。它是一个工具，可让您在虚拟机上安装Docker引擎，并轻松部署应用程序。

* [安装 Docker Machine](https://docs.docker.com/machine/install-machine/) 在你的机器上。

**注意** *这些说明适用于在没有TLS配置的情况下运行Dgraph Alpha。*
*使用TLS运行的说明参考[TLS instructions](#tls-configuration).*

Here we'll go through an example of deploying Dgraph Zero, Alpha and Ratel on an AWS instance.
在这里，我们将通过一个例子在AWS实例上部署Dgraph Zero，Alpha和Ratel。

* 确保按照[instructions](https://docs.docker.com/machine/install-machine/)中的方式安装了Docker Machine, 在AWS上配置实例只需一步之遥。您必须[配置您的AWS凭证](http://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html)才能以编程方式访问Amazon API。

* 创建一个新的 docker machine.

```sh
docker-machine create --driver amazonec2 aws01
```

输出应该是这样的

```sh
Running pre-create checks...
Creating machine...
(aws01) Launching instance...
...
...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env aws01
```

该命令将提供一个名为`docker-machine`的安全组的`t2-micro`实例（允许在2376和22上进行入站访问）。您可以编辑安全组以允许对`5080`, `8080`, `9080`（Dgraph Zero和Alpha的默认端口）的入站访问，或者您可以提供自己的安全组，允许在端口22,2376上进行入站访问（Docker Machine，5080,8080和9080要求。只有从外面运行Dgraph Live Loader或Dgraph Bulk Loader时，才需要记住端口*5080*。

[这里](https://docs.docker.com/machine/drivers/aws/#options) 是一个`amazonec2`驱动程序的完整选项列表，它允许您选择实例类型，安全组，AMI以及许多其他内容。

**提示** *Docker machine 支持 [其他驱动程序](https://docs.docker.com/machine/drivers/gce/)，如GCE，Azure等。*

* 使用docker-compose安装并运行Dgraph

Docker Compose是一个用于运行多容器Docker应用程序的工具。您可以按照[此处](https://docs.docker.com/compose/install/)的说明进行安装。

将以下文件复制到计算机上的目录中并命名`docker-compose.yml`.

```sh
version: "3.2"
services:
  zero:
    image: dgraph/dgraph:latest
    volumes:
      - /data:/dgraph
    ports:
      - 5080:5080
      - 6080:6080
    restart: on-failure
    command: dgraph zero --my=zero:5080
  server:
    image: dgraph/dgraph:latest
    volumes:
      - /data:/dgraph
    ports:
      - 8080:8080
      - 9080:9080
    restart: on-failure
    command: dgraph alpha --my=server:7080 --lru_mb=2048 --zero=zero:5080
  ratel:
    image: dgraph/dgraph:latest
    ports:
      - 8000:8000
    command: dgraph-ratel
```

**注意** *配置将实例上的`/data`（你可以挂载别的东西）目录挂载到容器中的`/dgraph`目录以持久化数据。*

* 连接到机器上运行的Docker引擎。

运行`docker-machine env aws01`告诉我们运行以下命令进行配置我们的shell。

```sh
eval $(docker-machine env aws01)
```

这会将我们的Docker客户端配置为与在AWS机器上运行的Docker引擎进行通信。

最后运行以下命令启动Zero和Alpha。

```sh
docker-compose up -d
```

这将启动3个Docker容器在同一台机器上运行Dgraph Zero，Alpha和Ratel。如果出现任何错误，Docker会重新启动容器。
您可以使用`docker-compose logs`命令查看日志。

## 多主机设置

### 使用Docker Swarm

#### 使用Docker Swarm进行群集设置

**注意** *这些说明适用于在没有TLS配置的情况下运行Dgraph Alpha。*
*使用TLS运行的说明参考[TLS instructions](#tls-configuration).*

在这里，我们将通过一个例子使用Docker Swarm在复制因子为3的三个不同AWS实例上部署3个Dgraph Alpha节点和1个Zero。

* .确保按照[instructions](https://docs.docker.com/machine/install-machine/)中的方式安装了Docker Machine。

```sh
docker-machine --version
```

* 在AWS上创建3个实例，并在其上安装[安装Docker Engine](https://docs.docker.com/engine/installation/)。可以手动完成或使用`docker-machine`完成。

您必须[配置您的AWS凭证](http://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html)才能使用Docker Machine创建实例。

考虑到您已设置AWS凭据，您可以使用以下命令启动安装了Docker Engine的3个AWS`t2-micro`实例。

```sh
docker-machine create --driver amazonec2 aws01
docker-machine create --driver amazonec2 aws02
docker-machine create --driver amazonec2 aws03
```

输出应该是这样的

```sh
Running pre-create checks...
Creating machine...
(aws01) Launching instance...
...
...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env aws01
```

该命令将提供一个名为`docker-machine`的安全组的`t2-micro`实例（允许在2376和22上进行入站访问）。

您需要编辑`docker-machine`安全组以打开以下端口上的入站流量。

1. 允许源为“docker machine”安全端口的所有端口上的所有入站流量，以便轻松进行与docker相关的通信。

2. 还可以在dgraph要求的以下端口上打开入站TCP流量：`5080`、`6080`、`8000`、`808[0-2]`、`908[0-2]`。请记住，只有从外部运行dgraph live loader或dgraph bulk loader时才需要端口*5080*。如果您没有打开1中的所有端口，则需要打开“7080”以启用Alpha-to-Alpha的通信。

如果您在AWS上，则在进行必要的更改后，安全组（**docker-machine**）如下。

![AWS Security Group](./images/aws.png)

[这里](https://docs.docker.com/machine/drivers/aws/#options)是一个`amazonec2`驱动程序的完整选项列表，它允许您选择实例类型，安全组，AMI以及许多其他内容。

**提示** *Docker machine 支持 [其他驱动程序](https://docs.docker.com/machine/drivers/gce/)，如GCE，Azure等。*

运行`docker-machine ps`显示我们启动的所有AWS EC2实例。

```sh
➜  ~ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
aws01   -        amazonec2    Running   tcp://34.200.239.30:2376            v17.11.0-ce
aws02   -        amazonec2    Running   tcp://54.236.58.120:2376            v17.11.0-ce
aws03   -        amazonec2    Running   tcp://34.201.22.2:2376              v17.11.0-ce
```

* 启动Swarm

Docker Swarm有manager和worker节点。可以在manager节点上启动和更新Swarm。我们将`aws01`设置为swarm manager。您可以先运行以下命令来初始化swarm。

我们将使用AWS提供的内部IP地址。运行以下命令获取`aws01`的内部IP。

```sh
docker-machine ssh aws01 ifconfig eth0
```

现在我们有了内部IP，让我们假设`172.31.64.18`是这种情况下的内部IP。让我们启动Swarm。

```sh
# 这会将我们的Docker客户端配置为与在aws01主机上运行的Docker引擎进行通信。
eval $(docker-machine env aws01)
docker swarm init --advertise-addr 172.31.64.18
```

输出:

```sh
Swarm initialized: current node (w9mpjhuju7nyewmg8043ypctf) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1y7lba98i5jv9oscf10sscbvkmttccdqtkxg478g3qahy8dqvg-5r5cbsntc1aamsw3s4h3thvgk \
    172.31.64.18:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

现在我们将使其他节点加入swarm。

```sh
eval $(docker-machine env aws02)
docker swarm join \
    --token SWMTKN-1-1y7lba98i5jv9oscf10sscbvkmttccdqtkxg478g3qahy8dqvg-5r5cbsntc1aamsw3s4h3thvgk \
    172.31.64.18:2377
```

输出:

```sh
This node joined a swarm as a worker.
```

aws03也一样

```sh
eval $(docker-machine env aws03)
docker swarm join \
    --token SWMTKN-1-1y7lba98i5jv9oscf10sscbvkmttccdqtkxg478g3qahy8dqvg-5r5cbsntc1aamsw3s4h3thvgk \
    172.31.64.18:2377
```

在Swarm manager`aws01`上，验证你的swarm是否正在运行。

```sh
docker node ls
```

输出:

```sh
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
ghzapjsto20c6d6l3n0m91zev     aws02               Ready               Active
rb39d5lgv66it1yi4rto0gn6a     aws03               Ready               Active
waqdyimp8llvca9i09k4202x5 *   aws01               Ready               Active              Leader
```

* 启动Dgraph集群

将以下文件复制到计算机上的目录中并命名`docker-compose.yml`

```sh
version: "3"
networks:
  dgraph:
services:
  zero:
    image: dgraph/dgraph:latest
    volumes:
      - data-volume:/dgraph
    ports:
      - 5080:5080
      - 6080:6080
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws01
    command: dgraph zero --my=zero:5080 --replicas 3
  alpha_1:
    image: dgraph/dgraph:latest
    hostname: "alpha_1"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8080:8080
      - 9080:9080
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws01
    command: dgraph alpha --my=alpha_1:7080 --lru_mb=2048 --zero=zero:5080
  alpha_2:
    image: dgraph/dgraph:latest
    hostname: "alpha_2"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8081:8081
      - 9081:9081
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws02
    command: dgraph alpha --my=alpha_2:7081 --lru_mb=2048 --zero=zero:5080 -o 1
  alpha_3:
    image: dgraph/dgraph:latest
    hostname: "alpha_3"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8082:8082
      - 9082:9082
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws03
    command: dgraph alpha --my=alpha_3:7082 --lru_mb=2048 --zero=zero:5080 -o 2
  ratel:
    image: dgraph/dgraph:latest
    hostname: "ratel"
    ports:
      - 8000:8000
    networks:
      - dgraph
    command: dgraph-ratel
volumes:
  data-volume:
```

在Swarm leader上运行以下命令以部署Dgraph Cluster。

```sh
eval $(docker-machine env aws01)
docker stack deploy -c docker-compose.yml dgraph
```

这样会运行三个Dgraph Alpha服务（由于我们的约束，每个VM上有一个），aws01上有一个Dgraph Zero服务和一个Dgraph Ratel。

这些约束（如compose文件中所示）非常重要，以便在重新启动任何容器时，swarm将相应的Dgraph Alpha或Zero容器放置在同一主机上以重新使用卷。 此外，如果运行的主机少于三台，请确保使用不同的卷或使用`-p p1 -w w1`选项运行Dgraph Alpha。

**注意**

*此设置将在实例上创建并使用名为`dgraph_data-volume`的本地卷。如果您计划替换实例，则应使用如[cloudstore](https://docs.docker.com/docker-for-aws/persistent-data-volumes)等远程存储而不是本地磁盘。*

您可以通过运行以下命令来验证是否已成功创建所有服务：

```sh
docker service ls
```

输出:

```sh
ID                  NAME                MODE                REPLICAS            IMAGE                PORTS
vp5bpwzwawoe        dgraph_ratel        replicated          1/1                 dgraph/dgraph:latest   *:8000->8000/tcp
69oge03y0koz        dgraph_alpha_2      replicated          1/1                 dgraph/dgraph:latest   *:8081->8081/tcp,*:9081->9081/tcp
kq5yks92mnk6        dgraph_alpha_3      replicated          1/1                 dgraph/dgraph:latest   *:8082->8082/tcp,*:9082->9082/tcp
uild5cqp44dz        dgraph_zero         replicated          1/1                 dgraph/dgraph:latest   *:5080->5080/tcp,*:6080->6080/tcp
v9jlw00iz2gg        dgraph_alpha_1      replicated          1/1                 dgraph/dgraph:latest   *:8080->8080/tcp,*:9080->9080/tcp
```

停止群集运行

```sh
docker stack rm dgraph
```

### 使用Docker Swarm建立高可用集群

下面是一个示例swarm配置，用于在6个不同的ec2实例上运行6个Dgraph Alpha节点和3个Zero节点。 类似于[使用Docker Swarm进行群集设置](#使用Docker-Swarm进行群集设置) 除了几处不同。 此设置将确保使用分片数据进行复制。 该文件假定有六个主机可用作docker-machine。 此外，如果您在少于六台主机上运行，请确保使用不同的卷或使用`-p p1 -w w1`选项运行Dgraph Alpha。

您需要编辑`docker-machine`安全组以打开以下端口上的入站流量。

1. 允许源为“docker machine”安全端口的所有端口上的所有入站流量，以便轻松进行与docker相关的通信。

2. 还可以在dgraph要求的以下端口上打开入站TCP流量：`5080`、`8000`、`808[0-5]`、`908[0-5]`。请记住，只有从外部运行dgraph live loader或dgraph bulk loader时才需要端口*5080*。如果您没有打开#1中的所有端口，则需要打开“7080”以启用Alpha到Alpha的通信。

如果您在AWS上，下面是经过必要更改后的安全组（**docker machine**）。

![AWS Security Group](./images/aws.png)

在主机上复制以下文件，并将其命名为docker-compose.yml

```yaml
version: "3"
networks:
  dgraph:
services:
  zero_1:
    image: dgraph/dgraph:latest
    volumes:
      - data-volume:/dgraph
    ports:
      - 5080:5080
      - 6080:6080
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws01
    command: dgraph zero --my=zero_1:5080 --replicas 3 --idx 1
  zero_2:
    image: dgraph/dgraph:latest
    volumes:
      - data-volume:/dgraph
    ports:
      - 5081:5081
      - 6081:6081
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws02
    command: dgraph zero -o 1 --my=zero_2:5081 --replicas 3 --peer zero_1:5080 --idx 2
  zero_3:
    image: dgraph/dgraph:latest
    volumes:
      - data-volume:/dgraph
    ports:
      - 5082:5082
      - 6082:6082
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws03
    command: dgraph zero -o 2 --my=zero_3:5082 --replicas 3 --peer zero_1:5080 --idx 3
  alpha_1:
    image: dgraph/dgraph:latest
    hostname: "alpha_1"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8080:8080
      - 9080:9080
    networks:
      - dgraph
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname == aws01
    command: dgraph alpha --my=alpha_1:7080 --lru_mb=2048 --zero=zero_1:5080
  alpha_2:
    image: dgraph/dgraph:latest
    hostname: "alpha_2"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8081:8081
      - 9081:9081
    networks:
      - dgraph
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname == aws02
    command: dgraph alpha --my=alpha_2:7081 --lru_mb=2048 --zero=zero_1:5080 -o 1
  alpha_3:
    image: dgraph/dgraph:latest
    hostname: "alpha_3"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8082:8082
      - 9082:9082
    networks:
      - dgraph
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.hostname == aws03
    command: dgraph alpha --my=alpha_3:7082 --lru_mb=2048 --zero=zero_1:5080 -o 2
  alpha_4:
    image: dgraph/dgraph:latest
    hostname: "alpha_4"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8083:8083
      - 9083:9083
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws04
    command: dgraph alpha --my=alpha_4:7083 --lru_mb=2048 --zero=zero_1:5080 -o 3
  alpha_5:
    image: dgraph/dgraph:latest
    hostname: "alpha_5"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8084:8084
      - 9084:9084
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws05
    command: dgraph alpha --my=alpha_5:7084 --lru_mb=2048 --zero=zero_1:5080 -o 4
  alpha_6:
    image: dgraph/dgraph:latest
    hostname: "alpha_6"
    volumes:
      - data-volume:/dgraph
    ports:
      - 8085:8085
      - 9085:9085
    networks:
      - dgraph
    deploy:
      placement:
        constraints:
          - node.hostname == aws06
    command: dgraph alpha --my=alpha_6:7085 --lru_mb=2048 --zero=zero_1:5080 -o 5
  ratel:
    image: dgraph/dgraph:latest
    hostname: "ratel"
    ports:
      - 8000:8000
    networks:
      - dgraph
    command: dgraph-ratel
volumes:
  data-volume:
```

**注意**
*1. 此设置假定您使用的是6台主机，但如果运行的主机少于6台，则必须在Dgraph alpha之间使用不同的卷，或使用`-p`和`-w`配置数据目录。*
*2. 此设置将在实例上创建并使用名为“dgraph_data-volume”的本地卷。 如果您计划替换实例，则应使用[cloudstore](https://docs.docker.com/docker-for-aws/persistent-data-volumes)等远程存储而不是本地磁盘。*

## 使用 Kubernetes (v1.8.4)

**注意**
*这些说明适用于在没有TLS配置的情况下运行Dgraph Alpha。 使用TLS运行的说明参考[TLS指令](#TLS配置).*

- 安装[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)，用于在kubernetes上部署和管理应用程序。

- 获取kubernetes群集并在您选择的云提供商上运行。您可以使用[kops](https://github.com/kubernetes/kops/blob/master/docs/aws.md)在AWS上进行设置。Kops默认在AWS上进行自动扩展，并为您创建卷和实例。

使用`kubectl get nodes`验证您的群集已启动并正在运行。 如果您使用带有默认选项的`kops`，则应准备好主节点和两个工作节点。

```sh
➜  kubernetes git:(master) ✗ kubectl get nodes
NAME                                          STATUS    ROLES     AGE       VERSION
ip-172-20-42-118.us-west-2.compute.internal   Ready     node      1h        v1.8.4
ip-172-20-61-179.us-west-2.compute.internal   Ready     master    2h        v1.8.4
ip-172-20-61-73.us-west-2.compute.internal    Ready     node      2h        v1.8.4
```

### 单服务器

一旦你的Kubernetes集群启动，你可以使用[dgraph-single.yaml](https://github.com/dgraph-io/dgraph/blob/master/contrib/config/kubernetes/dgraph-single.yaml)来启动Zero和Alpha。

- 在您的计算机上，运行以下命令以启动StatefulSet，该StatefulSet创建一个在其中运行Zero和Alpha的Pod。

```sh
kubectl create -f https://raw.githubusercontent.com/dgraph-io/dgraph/master/contrib/config/kubernetes/dgraph-single.yaml
```

输出:

```sh
service "dgraph-public" created
statefulset "dgraph" created
```

- 确认已成功创建pod。

```sh
kubectl get pods
```

输出:

```sh
NAME       READY     STATUS    RESTARTS   AGE
dgraph-0   3/3       Running   0          1m
```

**注意**
*您可以使用`kubectl logs -f dgraph-0 <container_name>`检查容器中容器的日志。例如，尝试使用`kubectl logs -f dgraph-0 alpha`得到服务器日志。*

- 测试设置

从本地机器向pod转发端口

```sh
kubectl port-forward dgraph-0 8080
kubectl port-forward dgraph-0 8000
```

转到`http://localhost:8000`并验证Dgraph是否按预期工作。

**注意** *您还可以在其外部IP地址上访问该服务。*

- 停止群集

删除所有资源

```sh
kubectl delete pods,statefulsets,services,persistentvolumeclaims,persistentvolumes -l app=dgraph
```

停止群集。如果您使用`kops`，则可以运行以下命令。

```sh
kops delete cluster ${NAME} --yes
```

### 使用Kubernets进行高可用群集设置

This setup allows you to run 3 Dgraph Alphas and 3 Dgraph Zeros. We start Zero with `--replicas
3` flag, so all data would be replicated on 3 Alphas and form 1 alpha group.

{{% notice "note" %}} Ideally you should have at least three worker nodes as part of your Kubernetes
cluster so that each Dgraph Alpha runs on a separate node.{{% /notice %}}

* Check the nodes that are part of the Kubernetes cluster.

```sh
kubectl get nodes
```

Output:

```sh
NAME                                          STATUS    ROLES     AGE       VERSION
ip-172-20-34-90.us-west-2.compute.internal    Ready     master    6m        v1.8.4
ip-172-20-51-1.us-west-2.compute.internal     Ready     node      4m        v1.8.4
ip-172-20-59-116.us-west-2.compute.internal   Ready     node      4m        v1.8.4
ip-172-20-61-88.us-west-2.compute.internal    Ready     node      5m        v1.8.4
```

Once your Kubernetes cluster is up, you can use [dgraph-ha.yaml](https://github.com/dgraph-io/dgraph/blob/master/contrib/config/kubernetes/dgraph-ha.yaml) to start the cluster.

* From your machine, run the following command to start the cluster.

```sh
kubectl create -f https://raw.githubusercontent.com/dgraph-io/dgraph/master/contrib/config/kubernetes/dgraph-ha.yaml
```

Output:
```sh
service "dgraph-zero-public" created
service "dgraph-alpha-public" created
service "dgraph-alpha-0-http-public" created
service "dgraph-ratel-public" created
service "dgraph-zero" created
service "dgraph-alpha" created
statefulset "dgraph-zero" created
statefulset "dgraph-alpha" created
deployment "dgraph-ratel" created
```

* Confirm that the pods were created successfully.

```sh
kubectl get pods
```

Output:
```sh
NAME                   READY     STATUS    RESTARTS   AGE
dgraph-ratel-<pod-id>  1/1       Running   0          9s
dgraph-alpha-0         1/1       Running   0          2m
dgraph-alpha-1         1/1       Running   0          2m
dgraph-alpha-2         1/1       Running   0          2m
dgraph-zero-0          1/1       Running   0          2m
dgraph-zero-1          1/1       Running   0          2m
dgraph-zero-2          1/1       Running   0          2m

```

{{% notice "tip" %}}You can check the logs for the containers in the pod using `kubectl logs -f dgraph-alpha-0` and `kubectl logs -f dgraph-zero-0`.{{% /notice %}}

* Test the setup

Port forward from your local machine to the pod

```sh
kubectl port-forward dgraph-alpha-0 8080
kubectl port-forward dgraph-ratel-<pod-id> 8000
```

Go to `http://localhost:8000` and verify Dgraph is working as expected.

{{% notice "note" %}} You can also access the service on its External IP address.{{% /notice %}}


* Stop the cluster

Delete all the resources

```sh
kubectl delete pods,statefulsets,services,persistentvolumeclaims,persistentvolumes -l app=dgraph-zero
kubectl delete pods,statefulsets,services,persistentvolumeclaims,persistentvolumes -l app=dgraph-alpha
kubectl delete pods,replicasets,services,persistentvolumeclaims,persistentvolumes -l app=dgraph-ratel
```

Stop the cluster. If you used `kops` you can run the following command.

```sh
kops delete cluster ${NAME} --yes
```

## More about Dgraph

On its HTTP port, a Dgraph Alpha exposes a number of admin endpoints.

* `/health` returns HTTP status code 200 and an "OK" message if the worker is running, HTTP 503 otherwise.
* `/admin/shutdown` initiates a proper [shutdown]({{< relref "#shutdown">}}) of the Alpha.
* `/admin/export` initiates a data [export]({{< relref "#export">}}).

By default the Alpha listens on `localhost` for admin actions (the loopback address only accessible from the same machine). The `--bindall=true` option binds to `0.0.0.0` and thus allows external connections.

{{% notice "tip" %}}Set max file descriptors to a high value like 10000 if you are going to load a lot of data.{{% /notice %}}

## More about Dgraph Zero

Dgraph Zero controls the Dgraph cluster. It automatically moves data between
different Dgraph Alpha instances based on the size of the data served by each Alpha instance.

It is mandatory to run at least one `dgraph zero` node before running any `dgraph alpha`.
Options present for `dgraph zero` can be seen by running `dgraph zero --help`.

* Zero stores information about the cluster.
* `--replicas` is the option that controls the replication factor. (i.e. number of replicas per data shard, including the original shard)
* When a new Alpha joins the cluster, it is assigned a group based on the replication factor. If the replication factor is 1 then each Alpha node will serve different group. If replication factor is 2 and you launch 4 Alphas, then first two Alphas would serve group 1 and next two machines would serve group 2.
* Zero also monitors the space occupied by predicates in each group and moves them around to rebalance the cluster.

Like Alpha, Zero also exposes HTTP on 6080 (+ any `--port_offset`). You can query it
to see useful information, like the following:

* `/state` Information about the nodes that are part of the cluster. Also contains information about
  size of predicates and groups they belong to.
* `/assignIds?num=100` This would allocate `num` ids and return a JSON map
containing `startId` and `endId`, both inclusive. This id range can be safely assigned
externally to new nodes, during data ingestion.
* `/removeNode?id=3&group=2` If a replica goes down and can't be recovered, you can remove it and add a new node to the quorum.
This endpoint can be used to remove a dead Zero or Dgraph alpha node. To remove dead Zero nodes, just pass `group=0` and the
id of the Zero node.
{{% notice "note" %}}
Before using the api ensure that the node is down and ensure that it doesn't come back up ever again.

You should not use the same `idx` as that of a node that was removed earlier.
{{% /notice %}}
* `/moveTablet?tablet=name&group=2` This endpoint can be used to move a tablet to a group. Zero
  already does shard rebalancing every 8 mins, this endpoint can be used to force move a tablet.


## TLS configuration

{{% notice "note" %}}
This section refers to the `dgraph cert` command which was introduced in v1.0.9. For previous releases, see the previous [TLS configuration documentation](https://docs.dgraph.io/v1.0.7/deploy/#tls-configuration).
{{% /notice %}}


Connections between client and server can be secured with TLS. Password protected private keys are **not supported**.

{{% notice "tip" %}}If you're generating encrypted private keys with `openssl`, be sure to specify encryption algorithm explicitly (like `-aes256`). This will force `openssl` to include `DEK-Info` header in private key, which is required to decrypt the key by Dgraph. When default encryption is used, `openssl` doesn't write that header and key can't be decrypted.{{% /notice %}}

### Self-signed certificates

The `dgraph cert` program creates and manages self-signed certificates using a generated Dgraph Root CA. The _cert_ command simplifies certificate management for you.

```sh
# To see the available flags.
$ dgraph cert --help

# Create Dgraph Root CA, used to sign all other certificates.
$ dgraph cert

# Create node certificate (needed for Dgraph Live Loader using TLS)
$ dgraph cert -n live

# Create client certificate
$ dgraph cert -c dgraphuser

# Combine all in one command
$ dgraph cert -n live -c dgraphuser

# List all your certificates and keys
$ dgraph cert ls
```

### File naming conventions

To enable TLS you must specify the directory path to find certificates and keys. The default location where the _cert_ command stores certificates (and keys) is `tls` under the Dgraph working directory; where the data files are found. The default dir path can be overridden using the `--dir` option.

```sh
$ dgraph cert --dir ~/mycerts
```

The following file naming conventions are used by Dgraph for proper TLS setup.

| File name | Description | Use |
|-----------|-------------|-------|
| ca.crt | Dgraph Root CA certificate | Verify all certificates |
| ca.key | Dgraph CA private key | Validate CA certificate |
| node.crt | Dgraph node certificate | Shared by all nodes for accepting TLS connections |
| node.key | Dgraph node private key | Validate node certificate |
| client._name_.crt | Dgraph client certificate | Authenticate a client _name_ |
| client._name_.key | Dgraph client private key | Validate _name_ client certificate |

The Root CA certificate is used for verifying node and client certificates, if changed you must regenerate all certificates.

For client authentication, each client must have their own certificate and key. These are then used to connect to the Dgraph node(s).

The node certificate `node.crt` can support multiple node names using multiple host names and/or IP address. Just separate the names with commas when generating the certificate.

```sh
$ dgraph cert -n localhost,104.25.165.23,dgraph.io,2400:cb00:2048:1::6819:a417
```

{{% notice "tip" %}}You must delete the old node cert and key before you can generate a new pair.{{% /notice %}}

{{% notice "note" %}}When using host names for node certificates, including _localhost_, your clients must connect to the matching host name -- such as _localhost_ not 127.0.0.1. If you need to use IP addresses, then add them to the node certificate.{{% /notice %}}

### Certificate inspection

The command `dgraph cert ls` lists all certificates and keys in the `--dir` directory (default 'tls'), along with details to inspect and validate cert/key pairs.

Example of command output:

```sh
-rw-r--r-- ca.crt - Dgraph Root CA certificate
    Issuer: Dgraph Labs, Inc.
       S/N: 3e468ac77ecd5017
Expiration: 23 Sep 28 19:10 UTC
  MD5 hash: 85B533D86B0DD689B9DBDAD6755B702F

-r-------- ca.key - Dgraph Root CA key
  MD5 hash: 85B533D86B0DD689B9DBDAD6755B702F

-rw-r--r-- client.srfrog.crt - Dgraph client certificate: srfrog
    Issuer: Dgraph Labs, Inc.
 CA Verify: PASSED
       S/N: 55cedf3c8606d98e
Expiration: 25 Sep 23 19:25 UTC
  MD5 hash: 445DCB276E29FA1000F79CAC376569BA

-rw------- client.srfrog.key - Dgraph Client key
  MD5 hash: 445DCB276E29FA1000F79CAC376569BA

-rw-r--r-- node.crt - Dgraph Node certificate
    Issuer: Dgraph Labs, Inc.
 CA Verify: PASSED
       S/N: 75aeb1ccd9a6f3fd
Expiration: 25 Sep 23 19:39 UTC
     Hosts: localhost
  MD5 hash: FA0FFC88F7AA654575CD48A493C3D65A

-rw------- node.key - Dgraph Node key
  MD5 hash: FA0FFC88F7AA654575CD48A493C3D65A
```

Important points:

* The cert/key pairs should always have matching MD5 hashes. Otherwise, the cert(s) must be regenerated. If the Root CA pair differ, all cert/key must be regenerated; the flag `--force` can help.
* All certificates must pass Dgraph CA verification.
* All key files should have the least access permissions, specially the `ca.key`, but be readable.
* Key files won't be overwritten if they have limited access, even with `--force`.
* Node certificates are only valid for the hosts listed.
* Client certificates are only valid for the named client/user.

### TLS options

The following configuration options are available for Alpha:

* `--tls_dir string` - TLS dir path; this enables TLS connections (usually 'tls').
* `--tls_use_system_ca` - Include System CA with Dgraph Root CA.
* `--tls_client_auth string` - TLS client authentication used to validate client connection. See [Client authentication](#client-authentication) for details.

```sh
# Default use for enabling TLS server (after generating certificates)
$ dgraph alpha --tls_dir tls
```

Dgraph Live Loader can be configured with following options:

* `--tls_dir string` - TLS dir path; this enables TLS connections (usually 'tls').
* `--tls_use_system_ca` - Include System CA with Dgraph Root CA.
* `--tls_server_name string` - Server name, used for validating the server's TLS host name.

```sh
# First, create a client certificate for live loader. This will create 'tls/client.live.crt'
$ dgraph cert -c live

# Now, connect to server using TLS
$ dgraph live --tls_dir tls -s 21million.schema -r 21million.rdf.gz
```

### Client authentication

The server option `--tls_client_auth` accepts different values that change the security policty of client certificate verification.

| Value | Description |
|-------|-------------|
| REQUEST | Server accepts any certificate, invalid and unverified (least secure) |
| REQUIREANY | Server expects any certificate, valid and unverified |
| VERIFYIFGIVEN | Client certificate is verified if provided (default) |
| REQUIREANDVERIFY | Always require a valid certificate (most secure) |

{{% notice "note" %}}REQUIREANDVERIFY is the most secure but also the most difficult to configure for remote clients. When using this value, the value of `--tls_server_name` is matched against the certificate SANs values and the connection host.{{% /notice %}}

## Cluster Checklist

In setting up a cluster be sure the check the following.

* Is at least one Dgraph Zero node running?
* Is each Dgraph Alpha instance in the cluster set up correctly?
* Will each Dgraph Alpha instance be accessible to all peers on 7080 (+ any port offset)?
* Does each instance have a unique ID on startup?
* Has `--bindall=true` been set for networked communication?

## Fast Data Loading

There are two different tools that can be used for fast data loading:

- `dgraph live` runs the Dgraph Live Loader
- `dgraph bulk` runs the Dgraph Bulk Loader

{{% notice "note" %}} Both tools only accept [RDF NQuad/Triple
data](https://www.w3.org/TR/n-quads/) in plain or gzipped format. Data
in other formats must be converted.{{% /notice %}}

### Live Loader

Dgraph Live Loader (run with `dgraph live`) is a small helper program which reads RDF NQuads from a gzipped file, batches them up, creates mutations (using the go client) and shoots off to Dgraph.

Dgraph Live Loader correctly handles assigning unique IDs to blank nodes across multiple files, and can optionally persist them to disk to save memory, in case the loader was re-run.

{{% notice "note" %}} Dgraph Live Loader can optionally write the xid->uid mapping to a directory specified using the `-x` flag, which can reused
given that live loader completed successfully in the previous run.{{% /notice %}}

```sh
$ dgraph live --help # To see the available flags.

# Read RDFs from the passed file, and send them to Dgraph on localhost:9080.
$ dgraph live -r <path-to-rdf-gzipped-file>

# Read RDFs and a schema file and send to Dgraph running at given address
$ dgraph live -r <path-to-rdf-gzipped-file> -s <path-to-schema-file> -d <dgraph-alpha-address:grpc_port> -z <dgraph-zero-address:grpc_port>
```

### Bulk Loader

{{% notice "note" %}}
It's crucial to tune the bulk loader's flags to get good performance. See the
section below for details.
{{% /notice %}}

Dgraph Bulk Loader serves a similar purpose to the Dgraph Live Loader, but can
only be used to load data into a new cluster. It cannot be run on an existing
live Dgraph cluster. Dgraph Bulk Loader is **considerably faster** than the
Dgraph Live Loader and is the recommended way to perform the initial import of
large datasets into Dgraph.

Only one or more Dgraph Zeros should be running for bulk loading. Dgraph Alphas
will be started later.

{{% notice "warning" %}}
Don't use bulk loader once the Dgraph cluster is up and running. Use it to import
your existing data to a new cluster.
{{% /notice %}}

You can [read some technical details](https://blog.dgraph.io/post/bulkloader/)
about the bulk loader on the blog.

See [Fast Data Loading]({{< relref "#fast-data-loading" >}}) for more info about
the expected N-Quads format.

**Reduce shards**: Before running the bulk load, you need to decide how many
Alpha groups will be running when the cluster starts. The number of Alpha groups
will be the same number of reduce shards you set with the `--reduce_shards`
flag. For example, if your cluster will run 3 Alpha with 3 replicas per group,
then there is 1 group and `--reduce_shards` should be set to 1. If your cluster
will run 6 Alphas with 3 replicas per group, then there are 2 groups and
`--reduce_shards` should be set to 2.

**Map shards**: The `--map_shards` option must be set to at least what's set for
`--reduce_shards`. A higher number helps the bulk loader evenly distribute
predicates between the reduce shards.

```sh
$ dgraph bulk -r goldendata.rdf.gz -s goldendata.schema --map_shards=4 --reduce_shards=2 --http localhost:8000 --zero=localhost:5080
{
	"RDFDir": "goldendata.rdf.gz",
	"SchemaFile": "goldendata.schema",
	"DgraphsDir": "out",
	"TmpDir": "tmp",
	"NumGoroutines": 4,
	"MapBufSize": 67108864,
	"ExpandEdges": true,
	"SkipMapPhase": false,
	"CleanupTmp": true,
	"NumShufflers": 1,
	"Version": false,
	"StoreXids": false,
	"ZeroAddr": "localhost:5080",
	"HttpAddr": "localhost:8000",
	"MapShards": 4,
	"ReduceShards": 2
}
The bulk loader needs to open many files at once. This number depends on the size of the data set loaded, the map file output size, and the level of indexing. 100,000 is adequate for most data set sizes. See `man ulimit` for details of how to change the limit.
Current max open files limit: 1024
MAP 01s rdf_count:176.0 rdf_speed:174.4/sec edge_count:564.0 edge_speed:558.8/sec
MAP 02s rdf_count:399.0 rdf_speed:198.5/sec edge_count:1.291k edge_speed:642.4/sec
MAP 03s rdf_count:666.0 rdf_speed:221.3/sec edge_count:2.164k edge_speed:718.9/sec
MAP 04s rdf_count:952.0 rdf_speed:237.4/sec edge_count:3.014k edge_speed:751.5/sec
MAP 05s rdf_count:1.327k rdf_speed:264.8/sec edge_count:4.243k edge_speed:846.7/sec
MAP 06s rdf_count:1.774k rdf_speed:295.1/sec edge_count:5.720k edge_speed:951.5/sec
MAP 07s rdf_count:2.375k rdf_speed:338.7/sec edge_count:7.607k edge_speed:1.085k/sec
MAP 08s rdf_count:3.697k rdf_speed:461.4/sec edge_count:11.89k edge_speed:1.484k/sec
MAP 09s rdf_count:71.98k rdf_speed:7.987k/sec edge_count:225.4k edge_speed:25.01k/sec
MAP 10s rdf_count:354.8k rdf_speed:35.44k/sec edge_count:1.132M edge_speed:113.1k/sec
MAP 11s rdf_count:610.5k rdf_speed:55.39k/sec edge_count:1.985M edge_speed:180.1k/sec
MAP 12s rdf_count:883.9k rdf_speed:73.52k/sec edge_count:2.907M edge_speed:241.8k/sec
MAP 13s rdf_count:1.108M rdf_speed:85.10k/sec edge_count:3.653M edge_speed:280.5k/sec
MAP 14s rdf_count:1.121M rdf_speed:79.93k/sec edge_count:3.695M edge_speed:263.5k/sec
MAP 15s rdf_count:1.121M rdf_speed:74.61k/sec edge_count:3.695M edge_speed:246.0k/sec
REDUCE 16s [1.69%] edge_count:62.61k edge_speed:62.61k/sec plist_count:29.98k plist_speed:29.98k/sec
REDUCE 17s [18.43%] edge_count:681.2k edge_speed:651.7k/sec plist_count:328.1k plist_speed:313.9k/sec
REDUCE 18s [33.28%] edge_count:1.230M edge_speed:601.1k/sec plist_count:678.9k plist_speed:331.8k/sec
REDUCE 19s [45.70%] edge_count:1.689M edge_speed:554.4k/sec plist_count:905.9k plist_speed:297.4k/sec
REDUCE 20s [60.94%] edge_count:2.252M edge_speed:556.5k/sec plist_count:1.278M plist_speed:315.9k/sec
REDUCE 21s [93.21%] edge_count:3.444M edge_speed:681.5k/sec plist_count:1.555M plist_speed:307.7k/sec
REDUCE 22s [100.00%] edge_count:3.695M edge_speed:610.4k/sec plist_count:1.778M plist_speed:293.8k/sec
REDUCE 22s [100.00%] edge_count:3.695M edge_speed:584.4k/sec plist_count:1.778M plist_speed:281.3k/sec
Total: 22s
```

The output will be generated in the `out` directory by default. Here's the bulk
load output from the example above:

```sh
$ tree ./out
./out
├── 0
│   └── p
│       ├── 000000.vlog
│       ├── 000002.sst
│       └── MANIFEST
└── 1
    └── p
        ├── 000000.vlog
        ├── 000002.sst
        └── MANIFEST

4 directories, 6 files
```

Because `--reduce_shards` was set to 2, there are two sets of p directories: one
in `./out/0` directory and another in the `./out/1` directory.

Once the output is created, they can be copied to all the servers that will run
Dgraph Alphas. Each Dgraph Alpha must have its own copy of the group's p
directory output. Each replica of the first group should have its own copy of
`./out/0/p`, each replica of the second group should have its own copy of
`./out/1/p`, and so on.

#### Tuning & monitoring

##### Performance Tuning

{{% notice "tip" %}}
We highly recommend [disabling swap
space](https://askubuntu.com/questions/214805/how-do-i-disable-swap) when
running Bulk Loader. It is better to fix the parameters to decrease memory
usage, than to have swapping grind the loader down to a halt.
{{% /notice %}}

Flags can be used to control the behaviour and performance characteristics of
the bulk loader. You can see the full list by running `dgraph bulk --help`. In
particular, **the flags should be tuned so that the bulk loader doesn't use more
memory than is available as RAM**. If it starts swapping, it will become
incredibly slow.

**In the map phase**, tweaking the following flags can reduce memory usage:

- The `--num_go_routines` flag controls the number of worker threads. Lowering reduces memory
  consumption.

- The `--mapoutput_mb` flag controls the size of the map output files. Lowering
  reduces memory consumption.

For bigger datasets and machines with many cores, gzip decoding can be a
bottleneck during the map phase. Performance improvements can be obtained by
first splitting the RDFs up into many `.rdf.gz` files (e.g. 256MB each). This
has a negligible impact on memory usage.

**The reduce phase** is less memory heavy than the map phase, although can still
use a lot.  Some flags may be increased to improve performance, *but only if
you have large amounts of RAM*:

- The `--reduce_shards` flag controls the number of resultant Dgraph alpha instances.
  Increasing this increases memory consumption, but in exchange allows for
higher CPU utilization.

- The `--map_shards` flag controls the number of separate map output shards.
  Increasing this increases memory consumption but balances the resultant
Dgraph alpha instances more evenly.

- The `--shufflers` controls the level of parallelism in the shuffle/reduce
  stage. Increasing this increases memory consumption.

## Monitoring
Dgraph exposes metrics via the `/debug/vars` endpoint in json format and the `/debug/prometheus_metrics` endpoint in Prometheus's text-based format. Dgraph doesn't store the metrics and only exposes the value of the metrics at that instant. You can either poll this endpoint to get the data in your monitoring systems or install **[Prometheus](https://prometheus.io/docs/introduction/install/)**. Replace targets in the below config file with the ip of your Dgraph instances and run prometheus using the command `prometheus -config.file my_config.yaml`.
```sh
scrape_configs:
  - job_name: "dgraph"
    metrics_path: "/debug/prometheus_metrics"
    scrape_interval: "2s"
    static_configs:
    - targets:
      - 172.31.9.133:6080 #For Dgraph zero, 6080 is the http endpoint exposing metrics.
      - 172.31.15.230:8080
      - 172.31.0.170:8080
      - 172.31.8.118:8080
```

{{% notice "note" %}}
Raw data exported by Prometheus is available via `/debug/prometheus_metrics` endpoint on Dgraph alphas.
{{% /notice %}}

Install **[Grafana](http://docs.grafana.org/installation/)** to plot the metrics. Grafana runs at port 3000 in default settings. Create a prometheus datasource by following these **[steps](https://prometheus.io/docs/visualization/grafana/#creating-a-prometheus-data-source)**. Import **[grafana_dashboard.json](https://github.com/dgraph-io/benchmarks/blob/master/scripts/grafana_dashboard.json)** by following this **[link](http://docs.grafana.org/reference/export_import/#importing-a-dashboard)**.

## Metrics

Dgraph metrics follow the [metric and label conventions for
Prometheus](https://prometheus.io/docs/practices/naming/).

### Disk Metrics

The disk metrics let you track the disk activity of the Dgraph process. Dgraph does not interact
directly with the filesystem. Instead it relies on [Badger](https://github.com/dgraph-io/badger) to
read from and write to disk.

 Metrics                          | Description
 -------                          | -----------
 `badger_disk_reads_total`        | Total count of disk reads in Badger.
 `badger_disk_writes_total`       | Total count of disk writes in Badger.
 `badger_gets_total`              | Total count of calls to Badger's `get`.
 `badger_memtable_gets_total`     | Total count of memtable accesses to Badger's `get`.
 `badger_puts_total`              | Total count of calls to Badger's `put`.
 `badger_read_bytes`              | Total bytes read from Badger.
 `badger_written_bytes`           | Total bytes written to Badger.

### Memory Metrics

The memory metrics let you track the memory usage of the Dgraph process. The idle and inuse metrics
gives you a better sense of the active memory usage of the Dgraph process. The process memory metric
shows the memory usage as measured by the operating system.

By looking at all three metrics you can see how much memory a Dgraph process is holding from the
operating system and how much is actively in use.

 Metrics                          | Description
 -------                          | -----------
 `dgraph_memory_idle_bytes`       | Estimated amount of memory that is being held idle that could be reclaimed by the OS.
 `dgraph_memory_inuse_bytes`      | Total memory usage in bytes (sum of heap usage and stack usage).
 `dgraph_memory_proc_bytes`       | Total memory usage in bytes of the Dgraph process. On Linux/macOS, this metric is equivalent to resident set size. On Windows, this metric is equivalent to [Go's runtime.ReadMemStats](https://golang.org/pkg/runtime/#ReadMemStats).

### LRU Cache Metrics

The LRU cache metrics let you track on how well the posting list cache is being used.

You can track `dgraph_lru_capacity_bytes`, `dgraph_lru_evicted_total`, and `dgraph_max_list_bytes`
(see the [Data Metrics]({{< relref "#data-metrics" >}})) to determine if the cache size should be
adjusted. A high number of evictions can indicate a large posting list that repeatedly is inserted
and evicted from the cache due to insufficient sizing. The LRU cache size can be tuned with the option
`--lru_mb`.

 Metrics                     | Description
 -------                     | -----------
 `dgraph_lru_hits_total`     | Total number of cache hits for posting lists in Dgraph.
 `dgraph_lru_miss_total`     | Total number of cache misses for posting lists in Dgraph.
 `dgraph_lru_race_total`     | Total number of cache races when getting posting lists in Dgraph.
 `dgraph_lru_evicted_total`  | Total number of posting lists evicted from LRU cache.
 `dgraph_lru_capacity_bytes` | Current size of the LRU cache. The max value should be close to the size specified by `--lru_mb`.
 `dgraph_lru_keys_total`     | Total number of keys in the LRU cache.
 `dgraph_lru_size_bytes`     | Size in bytes of the LRU cache.

### Data Metrics

The data metrics let you track the [posting list]({{< ref "/design-concepts/index.md#posting-list"
>}}) store.

 Metrics                          | Description
 -------                          | -----------
 `dgraph_max_list_bytes`          | Max posting list size in bytes.
 `dgraph_max_list_length`         | The largest number of postings stored in a posting list seen so far.
 `dgraph_posting_writes_total`    | Total number of posting list writes to disk.
 `dgraph_read_bytes_total`        | Total bytes read from Dgraph.

### Activity Metrics

The activity metrics let you track the mutations, queries, and proposals of an Dgraph instance.

 Metrics                          | Description
 -------                          | -----------
 `dgraph_goroutines_total`        | Total number of Goroutines currently running in Dgraph.
 `dgraph_active_mutations_total`  | Total number of mutations currently running.
 `dgraph_pending_proposals_total` | Total pending Raft proposals.
 `dgraph_pending_queries_total`   | Total number of queries in progress.
 `dgraph_num_queries_total`       | Total number of queries run in Dgraph.

### Health Metrics

The health metrics let you track to check the availability of an Dgraph Alpha instance.

 Metrics                          | Description
 -------                          | -----------
 `dgraph_alpha_health_status`     | **Only applicable to Dgraph Alpha**. Value is 1 when the Alpha is ready to accept requests; otherwise 0.

### Go Metrics

Go's built-in metrics may also be useful to measure for memory usage and garbage collection time.

 Metrics                        | Description
 -------                        | -----------
 `go_memstats_gc_cpu_fraction`  | The fraction of this program's available CPU time used by the GC since the program started.
 `go_memstats_heap_idle_bytes`  | Number of heap bytes waiting to be used.
 `go_memstats_heap_inuse_bytes` | Number of heap bytes that are in use.

### Unused Metrics

 Metrics                          | Description
 -------                          | -----------
 `dgraph_dirtymap_keys_total`     | Unused.
 `dgraph_posting_reads_total`     | Unused.

## Dgraph Administration

Each Dgraph Alpha exposes administrative operations over HTTP to export data and to perform a clean shutdown.

### Whitelist Admin Operations

By default, admin operations can only be initiated from the machine on which the Dgraph Alpha runs.
You can use the `--whitelist` option to specify whitelisted IP addresses and ranges for hosts from which admin operations can be initiated.

```sh
dgraph alpha --whitelist 172.17.0.0:172.20.0.0,192.168.1.1 --lru_mb <one-third RAM> ...
```
This would allow admin operations from hosts with IP between `172.17.0.0` and `172.20.0.0` along with
the server which has IP address as `192.168.1.1`.

### Secure Alter Operations

Clients can use alter operations to apply schema updates and drop particular or all predicates from the database.
By default, all clients are allowed to perform alter operations.
You can configure Dgraph to only allow alter operations when the client provides a specific token.
This can be used to prevent clients from making unintended or accidental schema updates or predicate drops.

You can specify the auth token with the `--auth_token` option for each Dgraph Alpha in the cluster.
Clients must include the same auth token to make alter requests.

```sh
$ dgraph alpha --lru_mb=2048 --auth_token=<authtokenstring>
```

```sh
$ curl -s localhost:8080/alter -d '{ "drop_all": true }'
# Permission denied. No token provided.
```

```sh
$ curl -s -H 'X-Dgraph-AuthToken: <wrongsecret>' localhost:8180/alter -d '{ "drop_all": true }'
# Permission denied. Incorrect token.
```

```sh
$ curl -H 'X-Dgraph-AuthToken: <authtokenstring>' localhost:8180/alter -d '{ "drop_all": true }'
# Success. Token matches.
```

{{% notice "note" %}}
To fully secure alter operations in the cluster, the auth token must be set for every Alpha.
{{% /notice %}}


### Export Database

An export of all nodes is started by locally accessing the export endpoint of any Alpha in the cluster.

```sh
$ curl localhost:8080/admin/export
```
{{% notice "warning" %}}By default, this won't work if called from outside the server where the Dgraph Alpha is running.
You can specify a list or range of whitelisted IP addresses from which export or other admin operations
can be initiated using the `--whitelist` flag on `dgraph alpha`.
{{% /notice %}}

This also works from a browser, provided the HTTP GET is being run from the same server where the Dgraph alpha instance is running.


{{% notice "note" %}}An export file would be created on only the server which is the leader for a group
and not on followers.{{% /notice %}}

This triggers an export of all the groups spread across the entire cluster. Each Alpha leader for a group writes output as a gzipped RDF file to the export directory specified on startup by `--export`. If any of the groups fail, the entire export process is considered failed and an error is returned.

{{% notice "note" %}}It is up to the user to retrieve the right export files from the Alphas in the cluster. Dgraph does not copy files to the Alpha that initiated the export.{{% /notice %}}

### Shutdown Database

A clean exit of a single Dgraph node is initiated by running the following command on that node.
{{% notice "warning" %}}This won't work if called from outside the server where Dgraph is running.
{{% /notice %}}

```sh
$ curl localhost:8080/admin/shutdown
```

This stops the Alpha on which the command is executed and not the entire cluster.

### Delete database

Individual triples, patterns of triples and predicates can be deleted as described in the [query languge docs](/query-language#delete).

To drop all data, you could send a `DropAll` request via `/alter` endpoint.

Alternatively, you could:

* [stop Dgraph]({{< relref "#shutdown" >}}) and wait for all writes to complete,
* delete (maybe do an export first) the `p` and `w` directories, then
* restart Dgraph.

### Upgrade Database

Doing periodic exports is always a good idea. This is particularly useful if you wish to upgrade Dgraph or reconfigure the sharding of a cluster. The following are the right steps safely export and restart.

- Start an [export]({{< relref "#export">}})
- Ensure it's successful
- Bring down the cluster
- Run Dgraph using new data directories.
- Reload the data via [bulk loader]({{< relref "#Bulk Loader" >}}).
- If all looks good, you can delete the old directories (export serves as an insurance)

These steps are necessary because Dgraph's underlying data format could have changed, and reloading the export avoids encoding incompatibilities.

### Post Installation

Now that Dgraph is up and running, to understand how to add and query data to Dgraph, follow [Query Language Spec](/query-language). Also, have a look at [Frequently asked questions](/faq).

## Troubleshooting

Here are some problems that you may encounter and some solutions to try.

#### Running OOM (out of memory)

During bulk loading of data, Dgraph can consume more memory than usual, due to high volume of writes. That's generally when you see the OOM crashes.

The recommended minimum RAM to run on desktops and laptops is 16GB. Dgraph can take up to 7-8 GB with the default setting `--lru_mb` set to 4096; so having the rest 8GB for desktop applications should keep your machine humming along.

On EC2/GCE instances, the recommended minimum is 8GB. It's recommended to set `--lru_mb` to one-third of RAM size.

You could also decrease memory usage of Dgraph by setting `--badger.vlog=disk`.

#### Too many open files

If you see an log error messages saying `too many open files`, you should increase the per-process file descriptors limit.

During normal operations, Dgraph must be able to open many files. Your operating system may set by default a open file descriptor limit lower than what's needed for a database such as Dgraph.

On Linux and Mac, you can check the file descriptor limit with `ulimit -n -H` for the hard limit and `ulimit -n -S` for the soft limit. The soft limit should be set high enough for Dgraph to run properly. A soft limit of 65535 is a good lower bound for a production setup. You can adjust the limit as needed.

## See Also

* [Product Roadmap to v1.0](https://github.com/dgraph-io/dgraph/issues/1)
