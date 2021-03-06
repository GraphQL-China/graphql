# Type System/类型系统

GraphQL的类型系统用于描述服务器的能力以及判断一个查询是否有效。类系统也描述了查询参数的输入类型，用于运行时检查有效性。

GraphQL服务器的能力是同schema来描述，schema使用其支持的类型和指令来定义。

一个给定GraphQL schema其自身首先必须要通过内部有效性验证，本章节将会讲述这个验证过程的相关规则。

一个GraphQL schema使用每种操作的根级类型表示：query/查询、mutation/更改和subscription/订阅，这表示schema是这三种操作开始的地方。

所有GraphQL schema内的类型都必须要有唯一的名字，任意两个类型都不应该有相同的名字，任意类型也不应该有和内建类型冲突的名字（包含Scalar/标量和Introspection/内省类型）。

所有GraphQL schema内的指令也必须拥有唯一的名字，指令和类型可以拥有相同的名字，因为两者之间并没有歧义。

所有schema内定义的类型和指令都不能以{"__"}（双下划线）开头命名，因为这是GraphQL内省系统专用。


## Types/类型

任何GraphQL Schema的最基本单元都是类型，GraphQL中有8种类型。

最基本的类型是`Scalar`/标量，一个标量代表一个原始值，例如字符串或者整数。有时候，一个标量字段的返回值可能是可枚举的，对应这种场景，GraphQL提速了`Enum`/枚举类型，其指定了响应结果的有效范围。

标量和枚举型组成了响应结果树的叶子节点，而中间的分支节点则是`Object`/对象类型，其定义了一套字段，每个字段是系统中的另一个类型，从而能够定义任意层次的类型层级。

GraphQL支持两种抽象类型：interface/接口和union/联合。

`Interface`定义了一系列字段，`Object`类型通过实现了其中的字段来实现它。当类型系统表明要返回一个接口时，其返回的都是一个实现这个接口的类型。

`Union`定义了一个可能类型的列表，与接口相似，当系统表明要返回一个联合时，其返回的是联合中的一个类型。

这些类型都是可为空且为单数，譬如，一个标量字符串会返回一个null或者一个字符串。通常有需要表示某个类型的列表，于是GraphQL中提供了`List`类型，将其他类型封装在其中。类似的`Non-Null`类型也是封装其他类型，用以标注返回结果不可为空。这两种类型称为“封装类型”，非封装类型称为基础类型，每个封装类型里面都有一个基础类型，通过不断解封装来找到基础类型。

向GraphQL提供复杂结构作为输入参数是十分有用的，GraphQL为此提供了`Input Object`类型，让客户端从schema中获知服务端具体需要什么样的数据。


### Scalars/标量

如名字所示，GraphQL中一个标量代表这一个原始值，GraphQL的响应采用的是树形层级结构，其叶子节点即是标量。

所有GraphQL标量都能以字符串形式表示，虽然取决于所用的返回格式，可能有准确的原始类型来表示指定标量，服务器在适当的时候也应采取这种类型。

GraphQL提供了一些内建标量，类型系统也允许根据语义添加其他标量。假设GraphQL中要定义一个标量`Time`/时间，可将字符串转换成ISO-8601的格式，当查询一个`Time`字段时，客户端可以使用ISO-8601解析器，将这个字段类型转换成客户端特有的原始类型。另一个有潜在用途的标量是`Url`，通常会序列化成字符串，但是会由服务器保证是有效的URL。

服务器可能会在schema中省略内建标量，譬如，服务器并未使用浮点数，那么它可能并不会包含`Float`类型。 但是一旦schema包含了本规范所述的类型，那么一定会遵守本规范描述的对应行为，譬如，服务器一定不会使用名为`Int`的类型去表示128-bit的数字，或者国际化信息。


**Result Coercion/结果类型转换**

当GraphQL服务器准备一个标量字段的时候，必须遵守此标量的描述协议，或者强制转换原始值，或者抛出错误。

例如：当服务器在准备一个`Int`型标量时，收到的是一个浮点数，如果服务器直接输出这个值，那势必会打破协定，所以服务器可以剔除小数部分，只保留整数部分，然后返回整数值，如果服务器收到的是布尔型值`true`，那么就可以返回`1`，如果服务器收到的是字符串型，那么就尝试以10为底，解析字符串为整数，如果服务器没法将某些值转换成`Int`，那么就只能抛出错误。

因为这个转换行为对于客户端是不可见的，所以准确的转换规则全由实现指定，规范对其的唯一要求就是输出值需遵守标量协议。

**Input Coercion/输入类型转换**

如果GraphQL的某个参数要求标量作为输入，其类型转换显而易见地必须良好定义。如果输入值无法满足转换规则，则必须抛出错误、

GraphQL对于整数和浮点数输入值有不同的字面量表示方式，其类型转换也对应输入类型做转换。GraphQL可以使用变量作为参数，这些变量的值像是在HTTP的传输中通常会被序列化，由于有些序列化方法并不区分整数和浮点数（譬如JSON），可能因为一个数没有有效小数值而被当作整数而不是浮点数。

下列所有类型中，除了Non-Null之外，如果显式提供了{null}，那么其输入结果就会被转换成{null}。

**Built-in Scalars/内建标量**

GraphQL提供一套基本的定义良好的标量类型，每个GraphQL服务器都应该支持这些类型，并且使用这些名字的类型必须遵守下文描述的行为、


#### Int/整数型

整数型标量类型表示一个32位有符号的无小数部分的数值。响应格式应该使用一个支持32位的整数型或者数值类型来表示这个标量类型、

**Result Coercion/结果类型转换**

GraphQL服务器应该转换非整数型原始数据为整数型，如若不能，则必须抛出字段错误。例如，从浮点型`1.0`转换成`1`，或者从字符串型`"2"`转换成`2`。

**Input Coercion/输入类型转换**

当需要作为输入类型时，只接受整数型输入值。其它类型，包含字符串型数值内容，都要抛出类型不正确的查询错误。如果整数型输入值小于-2<sup>31</sup>或者大于2<sup>31</sup>，也要抛出查询错误。

Note: 超过32位的整数建议使用字符串或者自定义的标量类型，因为不是所有平台和传输协议都支持超过32位精度编码额整数型数值。


#### Float/浮点型

浮点型标量类型表示一个有符号的双精度小数，见[IEEE 754](http://en.wikipedia.org/wiki/IEEE_floating_point)。响应格式应该使用一个合适的双精度数值类型来表示这个标量类型。

**Result Coercion/结果类型转换**

GraphQL服务器应该转换非浮点型原始数据为型，如若不能，则必须抛出字段错误。例如，从整数型`1`转换成`1.0`，或者是字符串型`"2"`转换成`2.0`。

**Input Coercion/输入类型转换**

当需要作为输入类型时，接受整数型和浮点型输入值。整数型会被转换成小数部分为空的浮点数，譬如整数型输入值`1`转换成`1.0`，其他类型，包含字符串型数值类型，都要抛出类型不正确的查询错误。如果整型输入值无法使用IEEE 754方式表示，也要抛出查询错误。


#### String/字符串型

字符串型标量表示UTF-8字符序列组成的文本数据。GraphQL一般使用字符串型来表示任意格式人类可读的文本。所有响应格式都必须支持字符串型表示，字符串型如下所述。

**Result Coercion/结果类型转换**

GraphQL服务器应该转换非字符串型原始数据为字符串型，如若不能，则必须抛出字段错误。例如，从布尔型true转换成`"true"`，从整型`1`转换成`"1"`。

**Input Coercion/输入类型转换**

当需要作为输入类型时，只接受有效的UTF-8字符串型输入值。其它类型都要抛出类型不正确的查询错误。


#### Boolean/布尔型

布尔型标量表示`true`或者`false`两个值。响应格式应该使用内建的布尔型，如不支持，则使用另外的表示法，整数型`1` 和`0`。

**Result Coercion/结果类型转换**

GraphQL服务器应该转换非布尔型原始数据为布尔型，如若不能，则必须抛出字段错误。例如，将任意不为零数值转换成`true`。

**Input Coercion/输入类型转换**

当需要作为输入类型时，只接受布尔型输入值。其它类型都要抛出类型不正确的查询错误。


#### ID

ID型标量表示一个唯一标识符，通常用于重取一个对象或者作为缓存的键。ID型使用`String`相同方式来序列化，但是它并不是为了人类可读，虽然它通常可能是数值型，但也总是序列化成`String`。

**Result Coercion/结果类型转换**

GraphQL对ID的格式是不干预的，将其序列化成字符串以保证ID的多种格式之间的相容性，可以是自增数值，可以是128位大数，也可是base64编码后的值和[GUID](http://en.wikipedia.org/wiki/Globally_unique_identifier)等格式的字符串值。

GraphQL服务器应该将给定ID格式转换成字符串，如若不能，则必须抛出字段错误。

**Input Coercion/输入类型转换**

当需要作为输入类型时，任意字符串（譬如`"4"`）或者整数（譬如`4`）都应该被转换成给定服务器支持的ID格式。其他的输入类型，包括浮点型（譬如`4.0`）都必须抛出类型不正确的查询错误。


### Objects/对象

GraphQL查询是层级式的可组装的，以树的形式描述了信息。其中标量类型描述了层级查询中叶子节点的值，对象则描述了中间层。

GraphQL对象表示一个具名字段列表，每个字段会产出一个特定类型的值。对象的值应该向有序映射集（ordered maps），其中查询字段名（或者别名）作为键，字段的结果作为值，以出现在查询中的顺序来排序。

对象中的所有字段都不应以{"__"}（双下划线）起头命名，因为这GraphQL内省系统专用的命名方式。

例如，`Person`类型可以如下描述：

```GraphQL
type Person {
  name: String
  age: Int
  picture: Url
}
```

其中`name`产生一个`String`值，`age`产生一个`Int`值，`picture`产生一个`Url`值。

对一个对象的查询必须至少指定一个字段，字段的选择集会产生一个有序映射集，其中包含被查询对象准确的子集，这个子集将以查询的顺序排序。只有在对象中声明过的字段才能被有效查询。

譬如，查询`Person`的所有字段：

```GraphQL
{
  name
  age
  picture
}
```

会产生如下对象：

```json
{
  "name": "Mark Zuckerberg",
  "age": 30,
  "picture": "http://some.cdn/picture.jpg"
}
```

当选择字段的子集：

```GraphQL
{
  age
  name
}
```

一定会产生准确的子集：

```json
{
  "age": 30,
  "name": "Mark Zuckerberg"
}
```

对象的字段可能是标量、枚举型、其他对象类型、接口、或者联合。也可能是其它封装类型，其内部类型是这五个之一。

例如：`Person`类型可能包含`relationship`：

```GraphQL
type Person {
  name: String
  age: Int
  picture: Url
  relationship: Person
}
```

对于字段对象的查询必须嵌套其字段集合，譬如下列查询就不是有效的：

```!graphql
{
  name
  relationship
}
```

然而，这个案例是有效的：

```GraphQL
{
  name
  relationship {
    name
  }
}
```

并会产生被查询的每个对象的子集：

```json
{
  "name": "Mark Zuckerberg",
  "relationship": {
    "name": "Priscilla Chan"
  }
}
```

**Field Ordering/字段排序**

当查询一个对象时，字段的结果映射在概念上的排序顺序应该是跟查询执行期间字段被执行的顺序一致，除了片段上并不适用于当前字段的类型和被`@skip`或`@include`指令跳过的字段。这个排序由{CollectFields()}算法正确完成。

表示有序映射集的响应序列化格式也应该采用同样的排序。只能表示无序映射集的序列化格式（譬如JSON）应该保证这个语法上的顺序。

响应中字段排序和请求中一致的表示方式，提升了调试过程中对人而言的可读性，也保证了响应属性顺序相关的解析效率。

如果一个片段在其他字段之前展开，那么片段的字段的顺位则在片段后续字段之前。

```GraphQL
{
  foo
  ...Frag
  qux
}

fragment Frag on Query {
  bar
  baz
}
```

产生下面排序结果：

```json
{
  "foo": 1,
  "bar": 2,
  "baz": 3,
  "qux": 4
}
```

如果一个字段在一个选择集中被多次查询，则以第一次被执行的顺序排序，片段中不适用的字段不会影响排序。

```GraphQL
{
  foo
  ...Ignored
  ...Matching
  bar
}

fragment Ignored on UnknownType {
  qux
  baz
}

fragment Matching on Query {
  bar
  qux
  foo
}
```

产生下面排序结果：

```json
{
  "foo": 1,
  "bar": 2,
  "qux": 3
}
```

如果一个字段被指令排除，那它也不会被纳入字段排序的考量之中。

```GraphQL
{
  foo @skip(if: true)
  bar
  foo
}
```

产生下面排序结果：

```json
{
  "bar": 1,
  "foo": 2
}
```

**Result Coercion/结果类型转换**

对象类型的结果类型转换判定机制是GraphQL执行器的核心，因此将在本规范的执行器那一章节覆盖这个内容。

**Input Coercion/输入类型转换**

对象类型不可作为有效输入类型。


#### Object Field Arguments/对象字段参数

概念上的对象字段是会产生值的函数，有时候对象字段能够接受参数来进一步指定返回值。对象字段参数在定义上是一个所有可能参数和参数输入类型的列表。

一个字段内的所有参数都不能以{"__"}（双下划线）起头命名，因为这GraphQL内省系统专用的命名方式。

例如，`Person`拥有一个`picture`字段，接受一个参数以返回特定大小的图片链接。

```GraphQL
type Person {
  name: String
  picture(size: Int): Url
}
```

GraphQL查询可选择性地指定参数，以让字段返回指定参数的结果。

譬如这个案例查询：

```GraphQL
{
  name
  picture(size: 600)
}
```

可能产生这个结果：

```json
{
  "name": "Mark Zuckerberg",
  "picture": "http://some.cdn/picture_600.jpg"
}
```

对象字段的参数可以是任何输入类型。


#### Object Field deprecation/对象字段弃用

应用在必要情况下会将对象字段标注为弃用。这样之后，查询弃用字段依然有效（为了保证既有客户端不被这个变更导致异常），但是这种字段应该在文档和工具中正确对待。


#### Object type validation/对象类型验证

对象类型可能因为定义的不严谨而导致潜在的无效性。GraphQL Schema中，以下规则必须被所有对象遵守。

1. 一个对象类型必须定义一个或多个字段。
2. 一个对象类型内的字段必须拥有这个对象类型内唯一的命名；任何两个字段都不可同名。
3. 一个对象类型的每个字段都不能以{"__"}（双下划线）起头命名。
4. 一个对象类型可以声明实现了一个或多个不同接口。
5. 一个对象类型必须是所有它所实现的接口的超集：
   1. 对象类型必须包含其接口内所有字段同名的字段。<small>译者案：此句翻译准确性待定</small>
      1. 这个对象字段类型必须是接口字段类型等价的类型或者其子类型（协变性）。
         1. 一个对象字段类型如果是这个接口字段类型等价（相同）的类型，那么它是一个有效子类型。
         2. 一个对象字段类型如果是一个对象类型，且其接口字段类型是一个接口类型或者联合类型，且这个对象字段类型是这个接口字段类型的可能类型，那么它是一个有效子类型。
         3. 一个对象字段类型如果是一个列表类型，且其接口字段类型也是一个列表类型，且这个对象字段类型的列表元素类型是接口字段类型的列表元素类型的子类型，那么它是一个有效子类型。
         4. 一个对象字段类型如果是其接口字段类型的子类型的Non-Null（非空）变体，那么它是一个有效子类型。
      2. 这个对象字段必须包含其接口字段上定一个所有参数的同名参数。
         1. 这个对象字段的参数必须接受其接口字段上参数同类型的参数（逆变性）。
      3. 这个对象字段可以包含其接口字段上未定义的附加参数，但附加参数并不是必须。


### Interfaces/接口

GraphQL接口表示一个具名字段列表以及其参数，GraphQL对象可以实现接口，并保证包含接口中的字段。

GraphQL接口上的字段拥有和GraphQL对象上相同的规则；字段类型可以是标量、对象、枚举型、接口或者联合，或者这五个类型作为基本类型的封装类型。

例如，一个接口可以描述某个必要字段的类型，譬如`Person`或者`Business`，随后实现这个接口。

```GraphQL
interface NamedEntity {
  name: String
}

type Person implements NamedEntity {
  name: String
  age: Int
}

type Business implements NamedEntity {
  name: String
  employeeCount: Int
}
```

产生接口额字段使用的场景为需要从多个对象找返回一个的情况，其中需要保证一定会有部分字段。

继续上述案例，`Contact`可能指代`NamedEntity`。

```GraphQL
type Contact {
  entity: NamedEntity
  phoneNumber: String
  address: String
}
```

这将使我们能够编写查询`Contact`中通用字段的语句。

```GraphQL
{
  entity {
    name
  }
  phoneNumber
}
```

当查询一个接口类型上的字段时，只有在接口上声明的字段可是被查询。上述案例中，`entity`返回`NamedEntity`，其中`NamedEntity`中定义了`name`，所以这个查询是有效的。所以，下面这个查询是无效的：

```!graphql
{
  entity {
    name
    age
  }
  phoneNumber
}
```

因为`entity`指代`NamedEntity`，这个接口中并没有定义`age`，所以只有在`entity`的结果为`Person`，查询`age`才有效。查询语句可以通过片段或者内联片段来表述这种情况：

```GraphQL
{
  entity {
    name
    ... on Person {
      age
    }
  },
  phoneNumber
}
```

**Result Coercion/结果类型转换**

接口类型应该能够判定给定结果对应哪一个对象类型，一旦确定，接口的结果类型转换和对象的接口转换采用一样的方法。

**Input Coercion/输入类型转换**

接口类型不可作为有效输入类型。


#### Interface type validation/接口类型验证

接口类型可能因为定义的不严谨而导致潜在的无效性。

1. 一个接口类型必须定义一个或多个字段。
2. 一个接口类型内的字段必须拥有这个接口类型内唯一的命名；任何两个字段都不可同名。
3. 一个对象类型的每个字段都不能以{"__"}（双下划线）起头命名。


### Unions/联合

GraphQL联合表示一个对象的类型是对象类型列表中之一，但不保证这些类型之间的字段。另一个区别于接口的方面是，对象会声明其实现的接口，而不知道它被包含的联合。

对于接口和对象，只可以直接查询在其中被定义的字段，如果要查询接口的其他字段，必须使用类型片段。对于联合也是一样，但是联合不定义任何字段，所以联合上**不**允许查询任何字段，除非使用类型片段。

例如，我们有以下类型系统：

```GraphQL
union SearchResult = Photo | Person

type Person {
  name: String
  age: Int
}

type Photo {
  height: Int
  width: Int
}

type SearchQuery {
  firstSearchResult: SearchResult
}
```

当查询`SearchQuery`类型的`firstSearchResult`字段时，查询可能需要片段的所有字段来判断类型。如果结果是`Person`，那请求其`name`，如果是`photo`，那请求`height`。下列案例是错的，因为联合上不定义任何字段：

```!graphql
{
  firstSearchResult {
    name
    height
  }
}
```

而正确的查询应该是：

```GraphQL
{
  firstSearchResult {
    ... on Person {
      name
    }
    ... on Photo {
      height
    }
  }
}
```

**Result Coercion/结果类型转换**

联合类型应该能够判定给定结果对应哪一个对象类型，一旦确定，联合的结果类型转换和对象的接口转换采用一样的方法。

**Input Coercion/输入类型转换**

联合类型不可作为有效输入类型。


#### Union type validation/联合类型验证

联合类型可能因为定义的不严谨而导致潜在的无效性。

1. 联合的成员类型必须是对象基础类型；标量、接口和联合都不能作为联合的成员类型。同样的，封装类型也不能是联合的成员类型。
2. 一个联合类型必须定义一个或者多个不同的成员类型。

### Enums/枚举型

GraphQL枚举型是基于标量类型的变体，其表示可能值的一个有限集。

GraphQL枚举型并不指代数值，而是正确的唯一值。他们序列化成字符串，字符串中用名来表示值。

**Result Coercion/结果类型转换**

GraphQL服务器必须返回定义中可能结果集的一个值。如果无法达成合理的类型转换，则抛出字段错误。

**Input Coercion/输入类型转换**

GraphQL用常量字面量表示枚举型输入值，GraphQL字符串型字面量不可作为枚举型输入值，否则将抛出查询错误。

查询变量的序列化传输方式中，如果对于非字符串符号值有区别于字符串的表示方法（譬如[EDN](https://github.com/edn-format/edn)），那么就应该采用那种方法。否则就像没有这个能力的大多数序列化传输方法一样，字符串型将被转义成同名的枚举型值。


### Input Objects/输入对象

字段上可能会定义参数，客户端将参数合在查询中传输，从而改变字段的行为。这些输入可能是字符串型或者枚举型，但是有时候需要比这个更复杂的类型结构。

上文中定义的`Object`并不适合在这儿重用，因为`Object`可能包含循环引用或者指代了接口类型或联合类型，这俩都不适合作为输入参数。因此，输入对象才成为了这个系统中单独的类型。

`Input Object`定义了输入字段的一个集合，输入字段并不是标量、枚举型或者其他输入对象，这使参数可以接受任意的复杂结构。

**Result Coercion/结果类型转换**

输入对象类型不可作为有效结果类型。

**Input Coercion/输入类型转换**

输入对象类型的值应该是输入对象的字面量或者无序映射集，否则将抛出错误。这个无序映射集不应该包含这个输入对象类型中未定义的字段，否则将抛出错误。

如果输入对象上定一个非空字段没有从原始值收到对应条目（譬如变量未赋值，或者赋予了{null}（空）值），则抛出错误。

类型转换的结果是环境特定的无序映射集，其定义了由输入对象类型和原始值构成的字段。

对于输入对象类型中的字段，如果原始值中有条目拥有相同的名字，条目的值是一个字面量或者变量（运行时值），这个条目就被添加到结果中同名的字段上。

结果中条目的值是原始值类型转换之后的结果，类型转换的规则由输入字段的类型决定。

下列是输入对象类型转换的案例：

```GraphQL
input ExampleInputObject {
  a: String
  b: Int!
}
```

原始值                   | 变量            | 转换后的值
----------------------- | --------------- | -----------------------------------
`{ a: "abc", b: 123 }`  | {null}          | `{ a: "abc", b: 123 }`
`{ a: 123, b: "123" }`  | {null}          | `{ a: "123", b: 123 }`
`{ a: "abc" }`          | {null}          | Error: Missing required field {b}
`{ a: "abc", b: null }` | {null}          | Error: {b} must be non-null.
`{ a: null, b: 1 }`     | {null}          | `{ a: null, b: 1 }`
`{ b: $var }`           | `{ var: 123 }`  | `{ b: 123 }`
`{ b: $var }`           | `{}`            | Error: Missing required field {b}.
`{ b: $var }`           | `{ var: null }` | Error: {b} must be non-null.
`{ a: $var, b: 1 }`     | `{ var: null }` | `{ a: null, b: 1 }`
`{ a: $var, b: 1 }`     | `{}`            | `{ b: 1 }`

Note: 输入值显式声明一个输入字段额度值为{null}和连输入字段都没声明两种情况存在语义差异。

#### Input Object type validation/输入对象类型验证

1. 一个输入对象类型必须定义一个或多个字段。
2. 一个输入对象类型内的字段必须拥有这个接口类型内唯一的命名；任何两个字段都不可同名。
3. 每个字段的返回类型必须是输入类型。


### Lists/列表型

GraphQL列表市一个特殊的集合类型，它声明了列表中元素的类型（下文指代为*元素类型*）。列表值的序列化结果是一个有序列表，列表中的元素根据元素类型序列化。一个字段使用了一个列表类型，其中封装了元素类型，其标注形式为：`pets: [Pet]`。

**Result Coercion/结果类型转换**

GraphQL服务器必须返回有序列表作为一个列表类型的结果，列表中的每一个元素都是其元素类型的结果类型转换的结果，如果无法达成合理的类型转换，则抛出字段错误。其中，如果返回的是非列表类型，类型转换也会失败，这是要抛出一个类型系统和实现不匹配的异常。

**Input Coercion/输入类型转换**

当作为输入类型的时候，要求所有列表值都符合列表元素类型。

如果作为输入类型传递给列表类型的值，既*不*是列表型也不是{null}空值，那么它将作为列表中的唯一元素作类型转换，这使输入能够在即便声明为"var args"类型参数数组，但只有一个参数被传入（常见场景），客户端可以直接传递值而不用封装成列表。

Note: 如果通过运行时变量传入给一个列表型的值是{null}，那么这个值会被解读为未传输任何列表，而不是一个长度为1的值为{null}的列表。


### Non-Null/非空型

默认情况下，GraphQL的所有类型都是可空的；{null}可作为上述所有类型的有效响应。如果要声明一个值不可为空，可以使用GraphQL的非空类型。这个类型封装一个内部类型，并表现为和被封装类型一样的行为，除了{null}不可作为其有效响应。非空类型使用一个感叹号跟着封装类型的方式来标注：`name: String!`。

**Nullable vs. Optional/可空 vs 可选**

查询上下文中的字段*总是*可选的，某个字段可以被省略而查询依然有效。然后，非空类型的字段绝不能对查询返回{null}。

输入（譬如字段参数）默认情况下总是可选的，然而如果一个参数是非空输入类型，那么它将不接受{null}也不能被省略。为了简明,(nullable)可空类型总是(optional)可选的，(non-null)非空类型总是(required)必须的。

**Result Coercion/结果类型转换**

上述所有结果类型转换中，{null}都被视为有效值，如果转换一个非空类型，那么会使用被封装类型的转换规则。如果非空类型的转换结果不为{null}空，那么就取这个结果。否则如果结果为{null}空，那么则抛出字段错误。

**Input Coercion/输入类型转换**

如果未提供非空类型的输入对象字段或者参数，则被认为是提供了字面量{null}，或者认为是提供了一个未赋值或者为{null}空值的运行时变量，然后抛出查询错误。

如果给非空类型提供的值是字面量而非{null}，或者一个非空变量值，则其会按照被封装类型的输入类型转换规则来转换。

案例：非空参数不可省略。

```!graphql
{
  fieldWithNonNullArg
}
```

案例：{null}空值不可作为非空参数的值。

```!graphql
{
  fieldWithNonNullArg(nonNullArg: null)
}
```

案例：一个可空的变量不能作为非空参数的值。

```GraphQL
query withNullableVariable($var: String) {
  fieldWithNonNullArg(nonNullArg: $var)
}
```

Note: 在验证章节的定义中，向可空类型提供非空的输入值是无效。

**Non-Null type validation/非空类型验证**

1. 一个非空类型不可封装另一个非空类型。


## Directives/指令

一个GraphQL Schema包含了执行引擎支持的指令表。

GraphQL的实现需要提供`@skip`和`@include`指令。


### @skip

`@skip`指令可用于字段、片段展开以及内联片段，从而能够在执行期间通过if参数完成条件性排除。

下列案例中，`experimentalField`只有在`$someTest`为`false`的时候才会被查询。

```GraphQL
query myQuery($someTest: Boolean) {
  experimentalField @skip(if: $someTest)
}
```


### @include

`@include`指令可用于字段、片段展开以及内联片段，从而能够在执行期间通过if参数完成条件性包含。

下列案例中，`experimentalField`只有在`$someTest`为`true`的时候才会被查询。

```GraphQL
query myQuery($someTest: Boolean) {
  experimentalField @include(if: $someTest)
}
```

Note: `@skip`和`@include`没有优先级差别，当`@skip`和`@include`同时应用于一个字段时，*当且仅当*`@skip`为`false`，`@include`为`true`的时候它才会被查询。相反的，在*仅有*`@skip`为`true`或者*仅有*`@include`为`false`的时候它*不会*被查询。


## Initial types/初始类型

一个GraphQL Schema包含了类型，表示query/查询、mutation/更改和subscription/订阅的起点，提供了进入类型系统的最初入口。其中query类型始终是必须的，并且是一个对象基础类型。mutation类型是可选的，如果其为空，那意味着此系统不支持mutation/更改，如果不为空，它必须是一个对象基础类型。同样的，subscription类型也是可选的，如果其为空，则此系统不支持subscription/订阅，如果不为空，它必须是一个对象基础类型。

query类型的字段表示GraphQL查询中最顶层可用的字段。例如：一个基本的GraphQL查询可能像这样：

```GraphQL
query getMe {
  me
}
```

当query对应的类型含有一个名为"me"的字段时，这个查询就是有效的。类似的：

```GraphQL
mutation setName {
  setName(name: "Zuck") {
    newName
  }
}
```

当mutation对应的类型不为空，这个类型有一个名为"setName"参数为"name"的字段时，这个更改就是有效的。

```GraphQL
subscription {
  newMessage {
    text
  }
}
```

当subscription对应的类型不为空，这个类型有一个名为"newMessage"的字段，且只有一个根字段时，这个订阅就是有效的。
