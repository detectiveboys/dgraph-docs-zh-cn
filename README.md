# dgraph中文文档

## 访问地址

https://detectiveboys.github.io/dgraph-docs-zh-cn

**访问不了的问题**

现在github.io由于众所周知的原因访问不了，可以将自己的网络DNS设置为阿里的

```sh
223.5.5.5
223.6.6.6
```

## 翻译进度

| Page   |   Progress   | Dog  | Finish |
| :-----------------: | :-------------: | :-----------: | :------: |
| [首页](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/home/index.md)      | :uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn: | [Valdanito](https://github.com/valdanitooooo)  | 2018-11-29      |
| [快速开始](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/get-started/index.md) |:uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn: | [Valdanito](https://github.com/valdanitooooo)   |  2018-11-29   |
| [查询语言](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/query-language/index.md)  |:uk: :sunny: :sunny: :cloud: :cloud: :cloud: :cn:  | [JustinRong](https://github.com/JustinRong) [zhenghaoyang](https://github.com/zhenghaoyang)  | - |
| [GraphQL+- 小技巧](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/tips/index.md)      |:uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn:  |  [Valdanito](https://github.com/valdanitooooo)  | 2020-11-29 |
| [Mutations](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/mutations/index.md)      |:uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn:  | [Valdanito](https://github.com/valdanitooooo)       | 2020-12-12 |
| [客户端](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/clients/index.md)   |:uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn:  | [Valdanito](https://github.com/valdanitooooo)  | 2018-12-01 |
| [集群部署](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/deploy/index.md)                  |:uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn:   | [Valdanito](https://github.com/valdanitooooo)  | 2018-12-09   |
| [常见问题](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/faq/index.md)                  |:uk: :cloud: :cloud: :cloud: :cloud: :cloud: :cn:  | - | -      |
| [具体问题指引](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/howto/index.md)    |:uk: :cloud: :cloud: :cloud: :cloud: :cloud: :cn:  | -       |   -    |
| [设计理念](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/design-concepts/index.md)        |:uk: :cloud: :cloud: :cloud: :cloud: :cloud: :cn: | -   | -      |
| [与同类产品比较](https://github.com/detectiveboys/dgraph-docs-zh-cn/blob/master/docs/dgraph-compared-to-other-databases/index.md) |:uk: :sunny: :sunny: :sunny: :sunny: :sunny: :cn: | [Valdanito](https://github.com/valdanitooooo)     | 2018-12-01     |

## 翻译风格

- 不予翻译的名词或术语

    dgraph、Zero、Alpha、Ratel、query、mutations、schema

- 语法高亮标记

    graphql语句：```graphql

    docker-compose语句：```yaml

    golang语句：```go

    json数据：```json

- 将一些原有的标记替换掉

    如{{% notice "note" %}}、{{% notice "tip" %}} 、{{% notice "warning" %}} 等，这些dgraph文档原有的标记markdown无法直接解释，替换成如下格式：

    ```html
    **注意** *note内容......*
    **提示** *tip内容......*
    **警告** *warning内容......*
    ```

## 翻译工具推荐

>[谷歌翻译](https://translate.google.com)

>[百度翻译](https://fanyi.baidu.com/translate)

>[Bing词典](http://cn.bing.com/dict/)

>[词都](http://www.dictall.com/)

>[词博](http://www.cibo.cn/)

## 运行项目

- clone项目源码

    ```bash
    git clone git@github.com:detectiveboys/dgraph-docs-zh-cn.git
    ```

- 方式一：本机运行（需要node环境）
  
    安装docsify-cli, 关于[docsify](https://docsify.js.org)

    ```bash
    npm i docsify-cli -g
    ```

    在项目根目录下执行：

    ```bash
    docsify serve ./docs
    ```

    访问项目主页http://localhost:3000

- 方式二：docker容器运行（需要docker环境）
  
    在项目根目录下执行：

    ```bash
    docker-compose up -d
    ```

    访问项目主页http://localhost:3000
