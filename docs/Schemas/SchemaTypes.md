> 版权声明：本文为博主辛苦翻译，未经博主允许不得转载。

### SchemaTypes

SchemaTypes定义路径（path）的[dufaults][]、[validation][]、[getters][]、[setters][]及查询时默认选择的[field][]等属性，还有Strings和Numbers的其他的一些特征。

下面是mongoose中可用的SchemaTypes：

  * String
  *	Number
  *	Date
  *	Buffer
  *	Boolean
  *	Mixed
  *	Objectid
  *	Array
  *	Decimal128

#### 例子

    var schema = new Schema({
      name:    String,
      binary:  Buffer,
      living:  Boolean,
      updated: { type: Date, default: Date.now },
      age:     { type: Number, min: 18, max: 65 },
      mixed:   Schema.Types.Mixed,
      _someId: Schema.Types.ObjectId,
      decimal: Schema.Types.Decimal128,
      array:      [],
      ofString:   [String],
      ofNumber:   [Number],
      ofDates:    [Date],
      ofBuffer:   [Buffer],
      ofBoolean:  [Boolean],
      ofMixed:    [Schema.Types.Mixed],
      ofObjectId: [Schema.Types.ObjectId],
      ofArrays:   [[]],
      ofArrayOfNumbers: [[Number]],
      nested: {
        stuff: { type: String, lowercase: true, trim: true }
      }
    })

    // example use

    var Thing = mongoose.model('Thing', schema);

    var m = new Thing;
    m.name = 'Statue of Liberty';
    m.age = 125;
    m.updated = new Date;
    m.binary = new Buffer(0);
    m.living = false;
    m.mixed = { any: { thing: 'i want' } };
    m.markModified('mixed');
    m._someId = new mongoose.Types.ObjectId;
    m.array.push(1);
    m.ofString.push("strings!");
    m.ofNumber.unshift(1,2,3,4);
    m.ofDates.addToSet(new Date);
    m.ofBuffer.pop();
    m.ofMixed = [1, [], 'three', { four: 5 }];
    m.nested.stuff = 'good';
    m.save(callback);

### SchemaType选项设置

你可以直接使用可用的type类型来声明一个schema type或者使用一个包含type字段的对象

    var schema1 = new Schema({
      test: String // `test` is a path of type String
    });

    var schema2 = new Schema({
      test: { type: String } // `test` is a path of type string
    });

除了type之外你还可以给path指定其他的一些属性，例如，你想要在保存字符串之前将它转化为小写。

    var schema2 = new Schema({
      test: {
        type: String,
        lowercase: true // Always convert `test` to lowercase
      }
    });

只有string类型的字段可以设置`lowercase`属性。有些选项适用于所有的schema类型，有些只适用于指定的schema类型。

#### 适用于所有schema类型的选项

  *	required：boolean或者function，如果值为true时可以为字段添加一个必须验证器；
  *	default：任意值或者function，用于给path设置默认值，如果值是一个函数，其返回值作为path的默认值；
  * select：boolean，指明查询时选择的字段；
  *	validate：function，为字段添加验证器函数；
  *	get：function，使用Object.defineProperty为字段自定义getter函数;
  *	set：function，使用Object.defineProperty为字段自定义setter函数;
  *	alias：stirng，mongoose>=4.10.0才可以使用，为给定的字段设置一个别名，通过`sets/gets`设置或获取值；

        var numberSchema = new Schema({
          integerOnly: {
            type: Number,
            get: v => Math.round(v),
            set: v => Math.round(v),
            alias: 'i'
          }
        });

        var Number = mongoose.model('Number', numberSchema);

        var doc = new Number();
        doc.integerOnly = 2.001;
        doc.integerOnly; // 2
        doc.i; // 2
        doc.i = 3.001;
        doc.integerOnly; // 3
        doc.i; // 3

#### indexes

可以通过设置schema类型的选项来定义mongoDB的索引；

  *	index：boolean，是否在该字段上建立索引（建立索引加快搜索速度，拖慢更新速度，因为更新数据时索引也要更新）；
  *	unique：boolean，是否在该字段上定义唯一的索引；
  *	sparse：boolean，是否在该字段上建立稀疏索引（稀疏索引会跳过键值不存在的文档）。

        var schema2 = new Schema({
        test: {
          type: String,
          index: true,
          unique: true // Unique index. If you specify `unique: true`
          // specifying `index: true` is optional if you do `unique: true`
        }
        });

#### String

  *	lowercase：boolean，是否总是调用toLowerCase()函数将字段值转为小写；
  *	uppercase：boolean，是否总是调用toUpperCase()函数将字段值转为大写；
  *	trim：boolean，是否总是调用trim()去掉字段值中的空格；
  *	match：RegExp，创建一个验证器检查字段值是否符合给出的正则表达式；
  *	enum：Array，一个验证器用于检查字段值是否在给定的数组中。

#### Number

  * min：Number，创建一个验证器用于检查字段值是否大于或等于给定的最小值；
  * max：Number，创建一个验证器用于检查字段值是否小于或等于给定的最大值。

#### Date

  *	min：Date（日期需大于等于min值）；
  *	max：Date（日期需小于等于max值）。

### 使用注意事项

Date对象内置的方法并没有和Mongoose关联起来，这意味着如果你调用Date的方法修改Document的Date类型值并保存并不会起作用，因为mongoose意识不到Date值变了，所以`doc.save()`无法保存修改。如果你必须使用内置的方法修改Date类型的值，保存前请调用`doc.marckModified('pathToYourDate')`通知mongoose此更改。

    var Assignment = mongoose.model('Assignment', { dueDate: Date });
    Assignment.findOne(function (err, doc) {
      doc.dueDate.setMonth(3);
      doc.save(callback); // THIS DOES NOT SAVE YOUR CHANGE

      doc.markModified('dueDate');
      doc.save(callback); // works
    })

#### Mixed

Mixed类型接受任意类型的值，该类型虽然很灵活但是却难以维护，在schema中可以定义Mixed类型的字段。下面的几种方法是等价的，都可以声明字段为Mixed类型。

    var Any = new Schema({ any: {} });
    var Any = new Schema({ any: Object });
    var Any = new Schema({ any: Schema.Types.Mixed });

Mixed属于chema-less类型的一种，你可以改变值为任意你想要的类型，但是mongoose失去了自动检测值的修改并保存修改的能力。如果你修改了Mixed类型的值，需要调用`doc.markModified(path)`方法通知mongoose该修改。

    person.anything = { x: [3, 4, { y: "changed" }] };
    person.markModified('anything');
    person.save(); // anything will now get saved

#### ObjectIds

声明字段时，设置类型为`Schema.Types.ObjectId`，指定值为ObjectId的一种类型。

    var mongoose = require('mongoose');
    var ObjectId = mongoose.Schema.Types.ObjectId;
    var Car = new Schema({ driver: ObjectId });
    // or just Schema.ObjectId for backwards compatibility with v2

#### Arrays

设置字段值类型为[SchemaTypes][]或者[Sub-Documents][]类型的数组。

    var ToySchema = new Schema({ name: String });
    var ToyBox = new Schema({
      toys: [ToySchema],
      buffers: [Buffer],
      string:  [String],
      numbers: [Number]
      // ... etc
    });

>注意：如果数组为空等价于`Mixed`类型，下列方法均创建元素为`Mixed`类型的数组。

    var Empty1 = new Schema({ any: [] });
    var Empty2 = new Schema({ any: Array });
    var Empty3 = new Schema({ any: [Schema.Types.Mixed] });
    var Empty4 = new Schema({ any: [{}] });

设置默认值为`undefined`重写默认情况

    var ToySchema = new Schema({
      toys: {
        type: [ToySchema],
        default: undefined
      }
    });

### 创建自定义的类型

mongoose支持[plugins][]扩展自定义schemaTypes，查看[plugins][]网站查看可用的类型，例如[mongoose-long][]、[mongoose-int32][]还有其他的类型。

#### `schema.path()`函数

`schema.path()`函数返回指定path的实例化的schema类型。

    var sampleSchema = new Schema({ name: { type: String, required: true } });
    console.log(sampleSchema.path('name'));
    // Output looks like:
    /**
    * SchemaString {
    *   enumValues: [],
    *   regExp: null,
    *   path: 'name',
    *   instance: 'String',
    *   validators: ...
    */

可以使用该函数查看path的schema类型，包括验证器和字段的类型。

## 下一章 —— [Connections][]

[Connections]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Connections.md

[plugins]: http://plugins.mongoosejs.io/
[mongoose-long]:https://github.com/aheckmann/mongoose-long
[mongoose-int32]:https://github.com/vkarpov15/mongoose-int32
[SchemaTypes]:http://mongoosejs.com/docs/api.html#schema_Schema.Types
[Sub-Documents]:http://mongoosejs.com/docs/subdocs.html
[validation]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-validate
[dufaults]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-default
[getters]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-get
[setters]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-set
[field]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-select
