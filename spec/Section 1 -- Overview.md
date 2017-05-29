# Overview/概览

GraphQL语言致力于提供一种直观的弹性语法系统，用以描述客户端程序设计时的数据需求以及数据交互行为。

例如，下面这个GraphQL请求将会从Facebook的GraphQL实现中获取id为4的用户的名字。

```GraphQL
{
  user(id: 4) {
    name
  }
}
```

将会产生如下JSON数据：

```json
{
  "user": {
    "name": "Mark Zuckerberg"
  }
}
```

GraphQL并不能像编程语言一样执行任意计算，但能针对具有本规范所述能力的应用服务器进行查询。GraphQL的实现并不要求应用服务器使用特定的编程语言或者存储系统，而是只需要应用服务器将他们的能力映射为符合GraphQL编码原理的统一语言和类型系统。这样既为产品开发提供了友好的统一接口，又为工具建设提供了强大平台。

GraphQL有若干设计原则：

 * **层次分明**：今时今日的大部分产品开发都涉及到创建和操作视图层次。为了满足应用结构的层次性，每个GraphQL查询也是层次构建的，每个查询和其数据共用了相同的形状，这样的方式在描述数据需求上更为直观。

 * **以产品为中心**：不可置辩的说，GraphQL是一种视图需求驱动的语言，因为主要是前端工程师书写它。GraphQL从前端工程师的思想和需求出发，再开发了语言和运行时库以满足这些需求。
 
 * **强类型**：每个GraphQL服务器都会构建一个针对应用的类型系统，查询语句就在这个类型系统上下文中执行。对于一个查询语句，GraphQL工具可以在执行以前通过类型系统检查这个查询语句的语法正确性和查询有效性，譬如在开发期，服务器就能保证返回值的形状和特性。

 * **客户端定制**：通过类型系统，GraphQL向客户端通告了自己那些可以被消费的能力。而客户端则专注于如何消费这些能力，其查询语句的粒度是字段级的。在大多数没有GraphQL的CS模型应用中，不同的服务端用不同的脚本和入口决定了返回的数据。而GraphQL查询则会返回客户端要求的数据，不多不少。

 * **内省**：GraphQL是内省的，一个GraphQL服务器的类型系统必须能用GraphQL语言自身来查询，本规范将后文描述此特性。GraphQL的内省特性使之能成为建造通用工具和客户端库的强大平台。

基于这些原则，GraphQL在建造客户端应用的时候就成了强大的生产环境。产品开发者和设计师在高质量工具的支撑下，无需阅读大量文档，只需一点或者无需正式训练就能根据GraphQL服务器建造客户端。当然为了完成这个目的，这些服务器和工具的建造者也必不可少。

下文的正式规范即作为这些建造者的参考指南，其描述了语言以及语法，接受查询的类型系统以及内省系统，执行引擎以及验证引擎的算法。本规范的目标是为GraphQL工具、客户端库、服务端实现提供了生态所需的基础和框架，无论组织还是平台，我们都希望和社区通力合作以完成上述目标。