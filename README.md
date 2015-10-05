# Groot-
Groot 中文翻译
Groot

Groot  提供了一种简单地转化方式在Core Data的模型关系图和JSON之间。

Groot 所使用的就是 “标注节点注释” 在 Core Data的模型中实现转化的并且提供了下面一些功能：

- 属性和关系可以映射到JSON键值上(key paths).

- 不同类型的值转化方式使用NSValueTransformer对象。

- 保留模型关系图。

- 支持实体继承。


Getting started

看看下面这个JSON描述了一个很知名的咔嗵人物：蝙蝠侠 能力：敏捷 ，壕。出版公司：DC漫画

{
    "id": "1699",
    "name": "Batman",
    "real_name": "Bruce Wayne",
    "powers": [
        {
            "id": "4",
            "name": "Agility"
        },
        {
            "id": "9",
            "name": "Insanely Rich"
        }
    ],
    "publisher": {
        "id": "10",
        "name": "DC Comics"
    }
}


我们转化到Core Data 的三个实体就是  Character, Power 和Publisher.

如何映射属性和关系？
Groot 事先在属性的UserInfo里设定的( key-value)键值路径，才能把属性，实体，关系连接起来去做JSON和NSManagedObject的转换化，这些键值在这个文档中有所说明 annotations.

在我的例子当中,我们应该在属性 和 关系 里面设置正确的转化路径 就是它“JSONKeyPath” ：

- 设定id路径来获取实体的identifier属性

- 设定name路径来获取实体的name属性

- 设定real_name路径来获取实体的realName属性

- 设定powers路径来获取实体的powers属性

- 设定publisher路径来获取实体的publisher属性

那些没有JSONKeyPath键值的属性和关系在转化中是被忽略的。

 数据转换

当我们想用Integer 64类型来表示我们的identifier时，那么问题来了，JSON里的id可特么是Sring 咋办你说：

我们添加一个 JSONTransformerName  到每一个属性的 userInfo里。他来指明怎么转换当前的值

Groot 提供了一种简单地方式去生成和注册这种转换器

// Swift

func toString(value: Int) -> String? {
    return "\(value)"
}

func toInt(value: String) -> Int? {
    return value.toInt()
}

NSValueTransformer.setValueTransformerWithName("StringToInteger", transform: toString, reverseTransform: toInt)


// Objective-C

[NSValueTransformer grt_setValueTransformerWithName:@"StringToInteger" transformBlock:^id(NSString *value) {
    return @([value integerValue]);
} reverseTransformBlock:^id(NSNumber *value) {
    return [value stringValue];
}];


 维护对象模型关系图

为了维护对象模型关系图避免多余重复的信息在转换中生成，Groot需要知道你的对象唯一的id.

在我们的例子中我们应该添加  identityAttributes 到 Character, Power 和 Publisher实体的UserInfo中去，并且值为 identifier.

更详细的在这里 Annotations.

JSON转化到对象

现在我们的添加一些数据到我们的 Core Data里。

// Swift

let batmanJSON: JSONObject = [
    "name": "Batman",
    "id": "1699",
    "powers": [
        [
            "id": "4",
            "name": "Agility"
        ],
        [
            "id": "9",
            "name": "Insanely Rich"
        ]
    ],
    "publisher": [
        "id": "10",
        "name": "DC Comics"
    ]
]

let batman: Character = try objectFromJSONDictionary(batmanJSON, inContext: context)


// Objective-C

Character *batman = [GRTJSONSerialization objectWithEntityName:@"Character"
                                            fromJSONDictionary:batmanJSON
                                                    inContext:self.context
                                                        error:&error];


如果我们想更新我们刚刚生成的对象，Groot 帮助可以合并他们：

// Objective-C

NSDictionary *updateJSON = @{
    @"id": @"1699",
    @"real_name": @"Bruce Wayne",
};

// This will return the previously created managed object
Character *batman = [GRTJSONSerialization objectWithEntityName:@"Character"
                                            fromJSONDictionary:updateJSON
                                                    inContext:self.context
                                                        error:&error];


 通过identityers转换关系

假设我们的api没有返回关系的全部信息，仅仅有id怎么办

我们不用改变模型，就可以支持这种情况:

// Objective-C

NSDictionary *batmanJSON = @{
    @"name": @"Batman",
    @"real_name": @"Bruce Wayne",
    @"id": @"1699",
    @"powers": @[@"4", @"9"],
    @"publisher": @"10"
};

Character *batman = [GRTJSONSerialization objectWithEntityName:@"Character"
                                            fromJSONDictionary:batmanJSON
                                                    inContext:self.context
                                                        error:&error];


上面的代码会生成一个信息全面的 Character 实体并且很好地维护Power 和 Publisher 的关系通过被提供的 identifier。

我们能导入不同的JSON数据，Groot帮我们很好地合并了他们；

// Objective-C

NSArray *powersJSON = @[
    @{
        @"id": @"4",
        @"name": @"Agility"
    },
    @{
        @"id": @"9",
        @"name": @"Insanely Rich"
    }
];

[GRTJSONSerialization objectsWithEntityName:@"Power"
                              fromJSONArray:powersJSON
                                  inContext:self.context
                                      error:&error];

NSDictionary *publisherJSON = @{
    @"id": @"10",
    @"name": @"DC Comics"
};

[GRTJSONSerialization objectWithEntityName:@"Publisher"
                        fromJSONDictionary:publisherJSON
                                inContext:self.context
                                    error:&error];


注意一下哈，这个样子转化关系的时候 尽在 有一个字段描述 identityAttributes 时有用.

For more serialization methods check GRTJSONSerialization.h and Groot.swift.


Annotations

在NSManagedObject里的实体，属性，关系，能被准确的联系起来都是靠着 事先在UserInfo里设定好的键值路径

我们能在xcode的 Data Model inspector 添加userinfo的值：

这个文档罗列了许多默认的键值：

Property annotations

JSONKeyPath
这个键值描述了你的NSManagedObject的属性如何映射到JSON字典里的。

For example, consider this JSON modelling a famous comic book character:

{
    "id": "1699",
    "name": "Batman",
    "publisher": {
        "id": "10",
        "name": "DC Comics"
    }
}


我们在CoreData定义了两个相互关联的 Character and Publisher.

The Character entity could have identifier and name attributes, and a publisher to-one relationship.

The Publisher entity could have identifier and name attributes, and a characters to-many relationship.

这些属性都应该加一个JSONKeyPath在他们的UserInfo

- id for the identifier attribute,
- name for the name attribute,
- publisher for the publisher relationship,
- etc.

那些没有JSONKeyPath键值的属性和关系在转化中是被忽略的。

注意下，如果我们仅仅关心publisher's name,我们可以去掉 Publisher 对象 并且给Character 添加  apublisherName 属性 ，用 publisher.name 作为 JSONKeyPath的值.

JSONTransformerName

With this key you can specify the name of a value transformer that will be used to transform values when serializing from or into JSON.

Consider the id key in the previous JSON. Some web APIs send 64-bit integers as strings to support languages that have trouble consuming large integers.

We should store identifier values as integers instead of strings to save space.

First we need to change the identifier attribute's type to Integer 64 in both the Character and Publisher entities.

Then we add a JSONTransformerName entry to each identifier attribute's user info dictionary with the name of the value transformer: StringToInteger.

Finally we create the value transformer and give it the name we just used:

[NSValueTransformer grt_setValueTransformerWithName:@"StringToInteger" transformBlock:^id(NSString *value) {
    return @([value integerValue]);
} reverseTransformBlock:^id(NSNumber *value) {
    return [value stringValue];
}];


If we were not interested in serializing characters back into JSON we could omit the reverse transformation:

[NSValueTransformer grt_setValueTransformerWithName:@"StringToInteger" transformBlock:^id(NSString *value) {
    return @([value integerValue]);
}];


Entity annotations

identityAttributes

用这个键值描述那些需要用一个或者多个ID来确定唯一的实体

在我的们的例子中，我们应该添加一个 identityAttributes 给 Character 和 Publisher 实体值只有一个就是 identifier.

多个ID段的用逗号隔开，举个例子 我们的实体是一张扑克牌，我们能设置 identityAttributes 的值是 suit花色, value大小.

值得注意的是子实体可以扩展他父实体的 identityAttributes. 举个例子, 如果你定义UUID做父实体的identityAttributes ， email 是他的子实体, Groot 将会用 UUID, email 是识别子实体的唯一id。

指定 identityAttributes 给一个实体是很有必要的为了维护对象关系图的唯一不重复性

entityMapperName

If your model uses entity inheritance, use this key in the base entity to specify an entity mapper name.

An entity mapper is used to determine which sub-entity is used when deserializing an object from JSON.

For example, consider the following JSON:

{
    "messages": [
        {
            "id": 1,
            "type": "text",
            "text": "Hello there!"
        },
        {
            "id": 2,
            "type": "picture",
            "image_url": "http://example.com/risitas.jpg"
        }
    ]
}


We could model this in Core Data using an abstract base entity Message and two concrete sub-entities TextMessageand PictureMessage.

Then we need to add an entityMapperName entry to the Message entity's user info dictionary: MessageMapper.

Finally we create the entity mapper and give it the name we just used:

[NSValueTransformer grt_setEntityMapperWithName:@"MessageMapper" mapBlock:^NSString *(NSDictionary *JSONDictionary) {
    NSDictionary *entityMapping = @{
        @"text": @"TextMessage",
        @"picture": @"PictureMessage"
    };
    NSString *type = JSONDictionary[@"type"];
    return entityMapping[type];
}];


JSONDictionaryTransformerName JSON数据再转化成对象前的预处理

This is an optional key you can specify at entity level that contains the name of a value transformer that will be used to transform the JSON dictionaries before serializing them to the target entity.

Think about it as an optional preprocessing step for your JSON.

Consider the situation in which we need to support both legacy and current JSON specs for one of the entities in the model. We could add a JSONDictionaryTransformerName entry and create the corresponding dictionary transformer:

[NSValueTransformer grt_setDictionaryTransformerWithName:@“MyTransformer”
                                          transformBlock:^NSDictionary *(NSDictionary *JSONDictionary) {
                                              id legacyIdentifier = JSONDictionary[@“legacy_id”];
                                              if (legacyIdentifier != nil) {
                                                  NSMutableDictionary *dictionary = [JSONDictionary mutableCopy];
                                                  dictionary[@“id”] = legacyIdentifier;

                                                  return dictionary;
                                              }

                                              return JSONDictionary;
                                          }];


This will ‘upgrade’ our legacy JSON objects to the new version which is the one that correctly maps to our entity.
