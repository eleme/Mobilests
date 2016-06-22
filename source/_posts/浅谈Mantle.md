title: 浅谈Mantle
date: 2016-06-20 17:12:15
tags:
- iOS
- Mantle
category: iOS
---

在公司使用`Mantle`也差不多有一年，其间有痛有笑，难一一道明。使用期间总算积累了一些经验，在此献丑，或能有益于人；由于是一些知识点的汇总，难免拉杂干涩，写的断断续续，行文并不流畅。所有索引都随文给出，不另外附录；建议核心部分，结合代码理解。另如有错漏，祈谅！

# 解析过程
`Mantle`的核心类是`MTLJSONAdapter`，负责整个解析过程。下面以`JSON To Object`为例粗略介绍一下的解析流程。

[相关代码见此](https://github.com/Mantle/Mantle/blob/master/Mantle/MTLJSONAdapter.m#L259), 这个函数前面部分是处理类似类簇的一类model，并不是很常见的需求，这里就不展开；接下来就是整个[常规的解析流程](https://github.com/Mantle/Mantle/blob/master/Mantle/MTLJSONAdapter.m#L284-L362), 无非是遍历整个`propertyKeys`,  依次解析。下面跟踪一遍单个`property`的解析。

单个`property`的解析前面一小半是处理花式`keypath`的写法，无关大体。真正的解析其实就是这[20行代码](https://github.com/Mantle/Mantle/blob/master/Mantle/MTLJSONAdapter.m#L316-L336)。 处理`keypath`拿到单个`property`对应的`JSON Value`, 然后通过`valueTransformersByPropertyKey`取出对应transformer，转化`Value`为相应的格式。单个`property`的解析完成。

解析完所有`propertyKeys`，就会通过转化后的dic集由`modelWithDictionary:`去初始化`ModelClass`，这个过程会走一下`validate`流程然后通过`KVC`去设置`property`。

# 三类Transformer
综上可以看出整个解析流程其实非常清晰明，问题核心在`transformer`。`transformer`的生成[相关代码在此](https://github.com/Mantle/Mantle/blob/master/Mantle/MTLJSONAdapter.m#L365)。

按性质大概可以分为三类`transformer`：
## keyTransformer
key为`property name`。首先根据`key`动态构造的`SEL`(`keyJSONTransformer`)去获取；未果则转入`JSONTransformerForKey:(NSSString *)key`获取；未果则继续获取第二类transformer。目前这类transformer主要集中在对`NSArray`（NSArray目前业界好像也有一些讨巧的做法，后面介绍）和一些非常规的`transformer`的处理，因为此处理论上你可以提供任意类型的`transformer`来满足复杂多样的需求。
## type transformer
`type`即为通过runtime获取`property`的类型，对象类型和基础类型会有不同的处理。
### 对象类型,
先根据`transformerForModelPropertiesOfClass:`去获取针对一些预置的,未实行`MTLJSONSerializing`类型的`transformer`,比如NSURL，NSDate（目前Mantle里面只实现了NSURL,至于为什么没有实行别的类型见这个[issue](https://github.com/Mantle/Mantle/issues/147)）；未果,当前`Class`若实现`MTLJSONSerializing`, 则为它生成一个transformer，这个是我们应用中的大部分场景，即一大堆自定义的`Model`，我们不需要做任何其他的事，只需要声明`property`时写上对应的类型就可以了，是不是人心大快！仍旧未果，就会通过`mtl_validatingTransformerForClass:`去生成一个`validate`的transformer（这类transformer的作用后面详述）。
### 基础类型
比较简单。首先通过`transformerForModelPropertiesOfObjCType:`获取一个`transformer`;未果也是生成一个`validate`的`transformer`。
## KVC也算一类transformer
当然除了这两类的`transformer`，因为最终`setValue`时会调用`KVC`去完成，`KVC`对基础类型的[自动解包](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/DataTypes.html#//apple_ref/doc/uid/20002171-BAJEAIEE)，也可以算作是一类`transformer`。

# Transformer使用
## 基本使用
结合上面的解析和`transformer`的生成流程，可以看出我们的生活确实简单了。首先在`property`里面声明好属性，然后通过`JSONKeyPathsByPropertyKey`做好`property`和`json key`的映射关系，基本上整个`Model`基本就完成了。
## Array的处理
但这个过程有个例外，就是对`NSArray`的处理。目前OC里面的`Array`泛型是编译时的，运行时获取不到数组中对象类型，这就需要我们通过通过`arrayTransformerWithModelClass：`构造一个上面的`typeTransformer`返回。但直到某一天看到有人给`YYModel`提了个[issue](https://github.com/ibireme/YYModel/issues/79)，发现这个问题似乎已经有了一个可能的解法。构造一个与`Model Class`同名的`protocol`让`NSArray` conform，这时通过`runtime`就可以取到这个`protocol name`，作为`Model Class`的name构造一个`type transformer`。这个feature JSONModel应该支持了一段时间，目前[YY已经集成](https://github.com/ibireme/YYModel/releases/tag/1.0.4)，`Mantle`何时支持，拭目以待！
## KVC里的小天地
解析完成后会调用KVC去setValue。KVC会对基础类型会做[自动wrapping和unwrapping](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/DataTypes.html#//apple_ref/doc/uid/20002171-BAJEAIEE)，省了很多事，但有个特性也需要注意。就是`- (void)setValue:(nullable id)value forKey:(NSString *)key`当key是基础类型而value是NSNull时会调用`- (void)setNilValueForKey:(NSString *)key`, 默认这个会崩溃，但Mantle并没有给你处理这种情况...所以需要我们在给基类给一个默认实现。
## type transformer是什么
从上面我们可以看到系统已经为NSURL添加了默认的transformer,即`NSURLJSONTransformer`，默认会对非空字符串调用[直接调用`[NSURL URLWithString:str]`构造字符串](https://github.com/Mantle/Mantle/blob/master/Mantle/NSValueTransformer%2BMTLPredefinedTransformerAdditions.m#L41)。有个问题是在字符串长度为0的情况下，这个函数还是会返回内容为空的URL对象。很多业务逻辑如果没有考虑到这一点是要犯错误的。对于UIColor，NSDate这类对象，如果整个应用的格式是统一的，按理我们也可以加上自己的这类transformer，这个顺理成章毋庸置疑。
## 巧用type transformer
结合`type transformer`其实我们还可以做一些更巧妙的事情。有一些类似ID的主键，服务器因为存储的是整形，出于方便和效率方面的考虑，服务端会直接返回一个整形的数据。但客户端在使用类型一直会把这个类型当做字符串使用，很多情况下就不免加上stringValue来转换类型或者加上显示的keyTransformer去显示转化。这时其实可以考虑加上一个NSStringJSONTransformer,转化可能的NSNumber为string，可谓一劳永逸。但这有个问题是，这个Model在转化为JSON时会丢失掉它原始的类型，但回传的需求，在应用中使用的太少，基本就忽略。

# Validate
validate从性质上也可以分为两类，一类为校验JSON数据是否返回了约定的格式，另一类校验返回的数据是否业务场景。
## 数据格式的校验（Mantle世界的麦田守望者）
校验数据格式的工作基本有上文提到的`validate transformer`提供，这个`transformer`位置重要，堪比麦田守望者。它会校验此时的`json object`是否是符合声明的类型，不是则及时报错。如果没有这个`transformer`，试想我们预期是`NSString`而拿到一个`NSNumber`会发生什么！数据格式错误往往是服务器出现了什么重大错误，或者http请求被恶意劫持，比较难以根治和解决，有了这个检验至少可以显著降低客户端的崩溃率。
## 业务逻辑的校验（后端不哭..）
业务逻辑的校验需要我们自己实现，返回的数据此时虽已确保类型无误。但我们还是不能确保它就是符合业务规范的，比如说关键的主键却为空，或者关联属性出现互相矛盾的结果。这时我们就需要我们根据业务需求去实现validate。理论上当然是逻辑越严密越好，但这时会导致代码异常繁琐，现实中一般做的并不多。因为这类错误，基本是服务端程序员的逻辑错误，所以只要他们能即使改正，客户端是可保无虞。
### 巧妙解决非空的判断
结合上面的NSArray的处理，其实我们也可以实现一个简单的Not NULL的实现，声明一个`NotNUll`的protocol，非空`NSString`，`NSNumber`属性conform一下。然后通过`runtime`去validate相关属性，是不是感觉清晰了不少...

# 一些实践
## 写好keypath
keypath推荐结合用[extobjc](https://github.com/jspahrsummers/libextobjc/tree/master/extobjc)结合code snippets，编译时检查，值得拥有。
## 加入默认值
有些时候，默认值是非空或0，可以在MTModel的基类里面的`- (instancetype)initWithDictionary:(NSDictionary *)dictionary error:(NSError **)error {
`里面加入一些默认值的处理。

算了，打住，不写了。
