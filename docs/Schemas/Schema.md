>官方原版：http://mongoosejs.com/docs/guide.html
> 版权声明：本文为博主辛苦翻译，未经博主允许不得转载。


## 纲要

如果你还不了解Mongoose如何工作，请先阅读[quickstart][]章节。如果你是从4.x版本迁移到5.x，请花点时间阅读[迁移指南][]。

### 定义你的schema 

在mongoose中一切都是从schema开始的，schema会被映射为mongoDB中的collection，并且定义了collection中documents的形式。

    var mongoose = require('mongoose');
    var Schema = mongoose.Schema;

    var blogSchema = new Schema({
        title:  String,
        author: String,
        body:   String,
        comments: [{ body: String, date: Date }],
        date: { type: Date, default: Date.now },
        hidden: Boolean,
        meta: {
        votes: Number,
        favs:  Number
        }
    });

如果你之后想额外的增加keys，可以使用[Schema#add][]方法。

在我们的blogSchema中每个key对应document中的一个字段并且有一个关联的[SchemaType][]。例如，我们定义的`title`字段的[SchemaType][]为String类型，`date`的[SchemaType][]是Date类型。keys也可以定义为嵌套的objects，包含了进一步的键/类型的定义，例如上面的`meta`属性。

下面是允许的schemaTypes类型：

   * String
   * Number
   * Date
   * Buffer
   * Boolean
   * Mixed
   * ObjectId
   * Array

点击查看更多的[SchemaType][]。

Schemas不仅定义了文档和其结构，还定义了model的[实例方法][]、[静态方法][]、[复合索引][]和属于文档[中间件][]的生命周期钩子函数。


[静态方法]:http://mongoosejs.com/docs/guide.html#statics
[实例方法]:http://mongoosejs.com/docs/guide.html#methods
[复合索引]:http://mongoosejs.com/docs/guide.html#indexes
[中间件]:http://mongoosejs.com/docs/middleware.html


### 创建一个model

我们需要将我们定义的`blogSchema`转换成一个model才能工作。我们将`blogSchema`传给`mongoose.model(modelName, schema)`;

      var Blog = mongoose.model('Blog', blogSchema);
    // ready to go!

### 实例方法

document是`model`的实例。document有许多内置的实例方法，我们也可以自定义document的实例方法。

    // define a schema
    var animalSchema = new Schema({ name: String, type: String });

    // assign a function to the "methods" object of our animalSchema
    animalSchema.methods.findSimilarTypes = function(cb) {
        return this.model('Animal').find({ type: this.type }, cb);
    };

现在所有的`animal`实例都拥有`findSimilarTypes`方法。

    var Animal = mongoose.model('Animal', animalSchema);
    var dog = new Animal({ type: 'dog' });

    dog.findSimilarTypes(function(err, dogs) {
        console.log(dogs); // woof
    });

   * 重写默认的mongoose document方法可能导致无法预知的结果，详情请点击[查看][]。
   * 不要使用箭头函数声明方法，因为箭头函数阻止绑定`this`，所以你的方法无法访问document，上例将无法正常工作。

### 静态方法

在`model`上绑定静态方法也很简单，继续上面的`animalSchema`示例：

    // assign a function to the "statics" object of our animalSchema
    animalSchema.statics.findByName = function(name, cb) {
        return this.find({ name: new RegExp(name, 'i') }, cb);
    };

    var Animal = mongoose.model('Animal', animalSchema);
    Animal.findByName('fido', function(err, animals) {
        console.log(animals);
    });

要使用箭头函数声明方法，因为箭头函数阻止绑定`this`，在上例中使用箭头函数将无法正常工作。

### 查询助手

你也可以像添加实例方法那样给mongoose的查询添加查询助手函数。查询助手方法可以帮你建立[链式查询][]模式。
    
    animalSchema.query.byName = function(name) {
        return this.find({ name: new RegExp(name, 'i') });
    };

    var Animal = mongoose.model('Animal', animalSchema);
    Animal.find().byName('fido').exec(function(err, animals) {
        console.log(animals);
    });

### 索引

MongoDB支持[secondary indexes][] 。在mongoose中，我们可以在schema的path级别和schema级别上定义索引。复合索引只能在schema级别上定义。

    var animalSchema = new Schema({
    name: String,
    type: String,
    tags: { type: [String], index: true } // field level
    });

    animalSchema.index({ name: 1, type: -1 }); // schema level

当应用程序启动时，mongoose会自动创建你在schema中定义的索引。mongoose为每个index顺序执行`createIndex`，当所有的`createIndex`执行成功或者发生错误时会触发model上的index事件。虽然这个特性有利于开发，但是建议在生成环境中关闭它，因为创建索引会对性能造成很大的影响。可以通过设置`autoIndex`选择值为false在schema级别上或者全局关闭自动创建索引。

    mongoose.connect('mongodb://user:pass@localhost:port/database', { autoIndex: false });
    // or
    mongoose.createConnection('mongodb://user:pass@localhost:port/database', { autoIndex: false });
    // or
    animalSchema.set('autoIndex', false);
    // or
    new Schema({..}, { autoIndex: false });

当所有index创建成功或者发生错误时会触发model上的index事件。

    // Will cause an error because mongodb has an _id index by default that
    // is not sparse
    animalSchema.index({ _id: 1 }, { sparse: true });
    var Animal = mongoose.model('Animal', animalSchema);

    Animal.on('index', function(error) {
        // "_id index cannot be sparse"
        console.log(error.message);
    });

点击查看[Model#ensureIndexes][]方法。
    
### 虚拟属性

document的虚拟属性可以被访问和设置，但是不会被存储到MongoDB中。getters用于格式化和组合字段，setter用于将一个值结构到数据库中的多个字段上。

    // define a schema
    var personSchema = new Schema({
        name: {
        first: String,
        last: String
        }
    });

    // compile our model
    var Person = mongoose.model('Person', personSchema);

    // create a document
    var axl = new Person({
        name: { first: 'Axl', last: 'Rose' }
    });

假设你想输出一个人的全名，你需要这样做：

    console.log(axl.name.first + ' ' + axl.name.last); // Axl Rose

每次都要连接first name和last name是很麻烦的事情。如果你想要对name做些其他的处理，例如移除符号该怎么办呢？你可以定义一个`fullname`的虚拟属性的getter，该值不会被保存到MongoDB中。

    personSchema.virtual('fullName').get(function () {
        return this.name.first + ' ' + this.name.last;
    });

现在你可以调用getter函数获取`fullName`字段的值：

    console.log(axl.fullName); // Axl Rose

mongoose不会输出虚拟的字段，当你使用`toJSON()`或者`toObject()`函数时（或者对document使用`JSON.stringify()`），可以给`toJSON()`或者[toObject()][]`传`{ virtuals: true }`获取虚拟字段。

你可以自定义虚拟字段的setter函数，通过虚拟的`fullName`字段一次性设置first name和last name的值。

    personSchema.virtual('fullName').
    get(function() { return this.name.first + ' ' + this.name.last; }).
    set(function(v) {
        this.name.first = v.substr(0, v.indexOf(' '));
        this.name.last = v.substr(v.indexOf(' ') + 1);
    });

    axl.fullName = 'William Rose'; // Now `axl.name.first` is "William"
虚拟字段的setters会在其他的validation之前运行。所以即使`first`和`last`字段值必须，上例依然会执行。

只有非虚拟字段可以用于查询和选择的字段，因为虚拟字段不会被存到MongoDB中，所以不能用于查询。

#### 别名

别名属于特殊的虚拟字段，其getter和setter用于设置和获取另一个字段。为了节省带宽，你可以使用简短的字段名称用于存储到数据库，使用更长的语义化的别名提高代码可读性。

    var personSchema = new Schema({
    n: {
        type: String,
        // Now accessing `name` will get you the value of `n`, and setting `n` will set the value of `name`
        alias: 'name'
    }
    });

    // Setting `name` will propagate to `n`
    var person = new Person({ name: 'Val' });
    console.log(person); // { n: 'Val' }
    console.log(person.toObject({ virtuals: true })); // { n: 'Val', name: 'Val' }
    console.log(person.name); // "Val"

    person.name = 'Not Val';
    console.log(person); // { n: 'Not Val' }
### 选项

schemas有一些可配置的选项可直接传递给构造器或者`set`：

    new Schema({..}, options);

    // or

    var schema = new Schema({..});
    schema.set(option, value);

有效的选项：

    * autoIndex
    * bufferCommands
    * capped
    * collection
    * id
    * _id
    * minimize
    * read
    * shardKey
    * strict
    * strictQuery
    * toJSON
    * toObject
    * typeKey
    * validateBeforeSave
    * versionKey
    * collation
    * skipVersioning
    * timeStamps
#### autoIndex

当应用程序启动时，mongoose会发出`createIndex`命令自动创建你在schema中定义的索引。mongoose v3版本索引默认在后台创建索引。如果你希望关闭自动创建选择手动创建索引时，设置`autoIndex`选择值为false，并调用model的[ensureIndexes][]方法。

    var schema = new Schema({..}, { autoIndex: false });
    var Clock = mongoose.model('Clock', schema);
    Clock.ensureIndexes(callback);

#### bufferCommands 

如果connection断掉，mongoose会默认缓冲命令，直到驱动重连才会执行命令。设置`bufferCommands`值为false关闭缓冲。

    var schema = new Schema({..}, { bufferCommands: false });

设置schema的`bufferCommands`选项会重写全局的`bufferCommands`选项：

    mongoose.set('bufferCommands', true);
    // Schema option below overrides the above, if the schema option is set.
    var schema = new Schema({..}, { bufferCommands: false });

#### capped

Mongoose支持设置MongoDB集合大小的上限值。设置`capped`属性限制MongoDB集合的最大值，单位bytes。

    new Schema({..}, { capped: 1024 });

如果你想传其他的选项，例如max或者autoIndexId，可以将`capped`值设置为一个对象。这种情况下，`size`属性为必传值。

    new Schema({..}, { capped: { size: 1024, max: 1000, autoIndexId: true } });

#### collection

Mongoose默认使用utils.tocollectionname方法生成集合的集合名称。该方法会生成复数形式的名称。如果你需要自定义集合名称，可以传入下面的选项。

    var dataSchema = new Schema({..}, { collection: 'data' });

#### id

mongoose默认为每一个schema分配一个虚拟的`id` getter，该方法返回document的`_id`字段，类型为string或者ObjectId（哈希字符串）。如果你的schema不需要`id` getter，给schema的构造函数传option禁用该功能。

    // default behavior
    var schema = new Schema({ name: String });
    var Page = mongoose.model('Page', schema);
    var p = new Page({ name: 'mongodb.org' });
    console.log(p.id); // '50341373e894ad16347efe01'

    // disabled id
    var schema = new Schema({ name: String }, { id: false });
    var Page = mongoose.model('Page', schema);
    var p = new Page({ name: 'mongodb.org' });
    console.log(p.id); // undefined

#### _id

如果没有禁用，mongoose默认为每一个schema分配`_id`字段，值为ObjectId类型，与MongoDB默认行为一致。如果你的schema不需要`id` getter，给schema的构造函数传option禁用该功能。

你只能在sub-document中使用此选项，因为如果没有`_id`，将document保存到数据库时会报错。

    // default behavior
    var schema = new Schema({ name: String });
    var Page = mongoose.model('Page', schema);
    var p = new Page({ name: 'mongodb.org' });
    console.log(p); // { _id: '50341373e894ad16347efe01', name: 'mongodb.org' }

    // disabled _id
    var childSchema = new Schema({ name: String }, { _id: false });
    var parentSchema = new Schema({ children: [childSchema] });

    var Model = mongoose.model('Model', parentSchema);

    Model.create({ children: [{ name: 'Luke' }] }, function(error, doc) {
    // doc.children[0]._id will be undefined
    });

#### minimize

mongoose默认不存储空对象，实现schemas最小化。

    var schema = new Schema({ name: String, inventory: {} });
    var Character = mongoose.model('Character', schema);

    // will store `inventory` field if it is not empty
    var frodo = new Character({ name: 'Frodo', inventory: { ringOfPower: 1 }});
    Character.findOne({ name: 'Frodo' }, function(err, character) {
    console.log(character); // { name: 'Frodo', inventory: { ringOfPower: 1 }}
    });

    // will not store `inventory` field if it is empty
    var sam = new Character({ name: 'Sam', inventory: {}});
    Character.findOne({ name: 'Sam' }, function(err, character) {
    console.log(character); // { name: 'Sam' }
    });

设置`minimize`选项重写默认行为，下例中会存储空对象：

    var schema = new Schema({ name: String, inventory: {} }, { minimize: false });
    var Character = mongoose.model('Character', schema);

    // will store `inventory` if empty
    var sam = new Character({ name: 'Sam', inventory: {}});
    Character.findOne({ name: 'Sam' }, function(err, character) {
    console.log(character); // { name: 'Sam', inventory: {}}
    });

#### read

mongoose允许在schema级别设置[查询＃读取][]选项，为我们提供一种将默认ReadPreferences应用于从模型派生的所有查询的方法。

    var schema = new Schema({..}, { read: 'primary' });            // also aliased as 'p'
    var schema = new Schema({..}, { read: 'primaryPreferred' });   // aliased as 'pp'
    var schema = new Schema({..}, { read: 'secondary' });          // aliased as 's'
    var schema = new Schema({..}, { read: 'secondaryPreferred' }); // aliased as 'sp'
    var schema = new Schema({..}, { read: 'nearest' });            // aliased as 'n'

每个pref允许使用简单的别名，不必须使用全拼，例如'secondaryPreferred'，防止拼写错误。

读选项允许我们设置tag，指明驱动应该试图读取的replicaSet。更多[tag设置][]详情点击查看。

>注意：你可以在连接数据库时，指明驱动的读选项的pref [strategy][]。

    // pings the replset members periodically to track network latency
    var options = { replset: { strategy: 'ping' }};
    mongoose.connect(uri, options);

    var schema = new Schema({..}, { read: ['nearest', { disk: 'ssd' }] });
    mongoose.model('JellyBean', schema);

#### shardKey

当我们有一个分片的MongoDB架构时，需要使用hardKey（片键）选项。每个分片集合都有一个分片键，它必须存在于所有插入/更新操作中。我们只需要在schema属性中配置shardKey。

    new Schema({ .. }, { shardKey: { tag: 1, name: 1 }})

>请注意，Mongoose不会为你发送`shardcollection`命令。 你必须自己配置你的碎片。

#### strict

mongoose默认开启strict模式，确保传递给model却没有在schema中声明的字段，不会被保存到数据库中。

    var thingSchema = new Schema({..})
    var Thing = mongoose.model('Thing', thingSchema);
    var thing = new Thing({ iAmNotInTheSchema: true });
    thing.save(); // iAmNotInTheSchema is not saved to the db

    // set to false..
    var thingSchema = new Schema({..}, { strict: false });
    var thing = new Thing({ iAmNotInTheSchema: true });
    thing.save(); // iAmNotInTheSchema is now saved to the db!!

该选项也影响`doc.set()`执行的结果。

    var thingSchema = new Schema({..})
    var Thing = mongoose.model('Thing', thingSchema);
    var thing = new Thing;
    thing.set('iAmNotInTheSchema', true);
    thing.save(); // iAmNotInTheSchema is not saved to the db

通过给model的实例传递第二个参数重写该选项。

    var Thing = mongoose.model('Thing');
    var thing = new Thing(doc, true);  // enables strict mode
    var thing = new Thing(doc, false); // disables strict mode

`strict`值也可以被设置为`throw`，如果保存坏数据到数据库会报错而不是忽略该数据。

>请注意，不设置schema的option时，model实例设置任何的未在schema中声明的key/val键值对都会被忽视。

    var thingSchema = new Schema({..})
    var Thing = mongoose.model('Thing', thingSchema);
    var thing = new Thing;
    thing.iAmNotInTheSchema = true;
    thing.save(); // iAmNotInTheSchema is never saved to the db

####  strictQuery

为了向后兼容，`strict`选项在查询时不会用于过滤字段。

    const mySchema = new Schema({ field: Number }, { strict: true });
    const MyModel = mongoose.model('Test', mySchema);

    // Mongoose will **not** filter out `notInSchema: 1`, despite `strict: true`
    MyModel.find({ notInSchema: 1 });

`strict`选项也会用于更新数据。

    // Mongoose will strip out `notInSchema` from the update if `strict` is
    // not `false`
    MyModel.updateMany({}, { $set: { notInSchema: 1 } });

mongoose提供了一个独立的选项`strictQuery`用于查询时过滤字段。

    const mySchema = new Schema({ field: Number }, {
        strict: true,
        strictQuery: true // Turn on strict mode for query filters
    });
    const MyModel = mongoose.model('Test', mySchema);

    // Mongoose will strip out `notInSchema: 1` because `strictQuery` is `true`
    MyModel.find({ notInSchema: 1 });

#### toJSON

与[toObject][]选项相同，但是`toJSON`方法只能被documents调用

    var schema = new Schema({ name: String });
    schema.path('name').get(function (v) {
    return v + ' is my name';
    });
    schema.set('toJSON', { getters: true, virtuals: false });
    var M = mongoose.model('Person', schema);
    var m = new M({ name: 'Max Headroom' });
    console.log(m.toObject()); // { _id: 504e0cd7dd992d9be2f20b6f, name: 'Max Headroom' }
    console.log(m.toJSON()); // { _id: 504e0cd7dd992d9be2f20b6f, name: 'Max Headroom is my name' }
    // since we know toJSON is called whenever a js object is stringified:
    console.log(JSON.stringify(m)); // { "_id": "504e0cd7dd992d9be2f20b6f", "name": "Max Headroom is my name" }

[点击查看][]所有可用的`toJSON/toObject`选项。

#### toObject

document可以调用[toObject][]方法将mongoose document转换为普通的JavaScript对象。该方法接受一些参数，我们可以声明这些选项并默认应用于所有的schema documents，而不是在每个文档的基础上应用这些选项。

想要在`console.log`中输出所有的虚拟字段，在`toObject`选项中设置`{ getters: true }`：

    var schema = new Schema({ name: String });
    schema.path('name').get(function (v) {
    return v + ' is my name';
    });
    schema.set('toObject', { getters: true });
    var M = mongoose.model('Person', schema);
    var m = new M({ name: 'Max Headroom' });
    console.log(m); // { _id: 504e0cd7dd992d9be2f20b6f, name: 'Max Headroom is my name' }

点击查看[这里][]所有可用的`toObject`选项。

#### typeKey

如果schema中的object定义了'type'字段，mongoose默认把它理解为字段类型声明。

    // Mongoose interprets this as 'loc is a String'
    var schema = new Schema({ loc: { type: String, coordinates: [Number] } });

然而有些应用中例如[geoJSON][]，'type'属性是非常重要的，你可以设置'typeKey'选项来控制哪个字段用于声明字段类型。

    var schema = new Schema({
    // Mongoose interpets this as 'loc is an object with 2 keys, type and coordinates'
    loc: { type: String, coordinates: [Number] },
    // Mongoose interprets this as 'name is a String'
    name: { $type: String }
    }, { typeKey: '$type' }); // A '$type' key means this object is a type declaration

#### validateBeforeSave

在document保存到数据库前会自动执行验证器，对于验证失败的文档会阻止保存。如果你想手动控制验证并且让验证失败的document也可以保存到数据库，可以设置`validateBeforeSave`值为false。

    var schema = new Schema({ name: String });
    schema.set('validateBeforeSave', false);
    schema.path('name').validate(function (value) {
        return v != null;
    });
    var M = mongoose.model('Person', schema);
    var m = new M({ name: null });
    m.validate(function(err) {
        console.log(err); // Will tell you that null is not allowed.
    });
    m.save(); // Succeeds despite being invalid

#### versionKey

mongoose默认在第一次创建文档时为每个document设置`versionKey`。此值包含了document的内部修订版本号。`versionkey`选项是一个字符串，用于表示版本控制的路径，默认值为`_v`。如果与你的应用有冲突，你可以如下例中修改：

    var schema = new Schema({ name: 'string' });
    var Thing = mongoose.model('Thing', schema);
    var thing = new Thing({ name: 'mongoose v3' });
    thing.save(); // { __v: 0, name: 'mongoose v3' }

    // customized versionKey
    new Schema({..}, { versionKey: '_somethingElse' })
    var Thing = mongoose.model('Thing', schema);
    var thing = new Thing({ name: 'mongoose v3' });
    thing.save(); // { _somethingElse: 0, name: 'mongoose v3' }

可以设置`versionKey`值为false，禁用document的版本号。不要禁用版本号除非[你知道你正在做什么][]。

#### collation

为每个查询和聚合设置默认的[collation][]。 这是一个初学者友好的[collation][][collation总览][]。

#### skipVersioning

`skipversioning`允许阻止版本号更新（例如，内部版本号不会增加即使这些路径更新）。不要这样做，除非你知道你在做什么。对于子文档，使用完全限定的路径将版本号包含在父文档中。

    new Schema({..}, { skipVersioning: { dontVersionMe: true } });
    thing.dontVersionMe.push('hey');
    thing.save(); // version is not incremented

#### timestamps

如果设置了`timestamps`，mongoose会为你的schema分配`createAt`和`updateAt`字段，值为`Date`类型。

默认情况下，两个字段的名称为`createAt`和`updateAt`，设置`timestamps.createdAt`和`timestamps.updatedAt`自定义字段名称。

    var thingSchema = new Schema({..}, { timestamps: { createdAt: 'created_at' } });
    var Thing = mongoose.model('Thing', thingSchema);
    var thing = new Thing();
    thing.save(); // `created_at` & `updatedAt` will be included

#### useNestedStrict

mongoose v4版本执行`update()`和`findOneAndUpdate()`操作时只会检查顶级schema的严格模式设置。

    var childSchema = new Schema({}, { strict: false });
    var parentSchema = new Schema({ child: childSchema }, { strict: 'throw' });
    var Parent = mongoose.model('Parent', parentSchema);
    Parent.update({}, { 'child.name': 'Luke Skywalker' }, function(error) {
    // Error because parentSchema has `strict: throw`, even though
    // `childSchema` has `strict: false`
    });

    var update = { 'child.name': 'Luke Skywalker' };
    var opts = { strict: false };
    Parent.update({}, update, opts, function(error) {
    // This works because passing `strict: false` to `update()` overwrites
    // the parent schema.
    });

如果你设置了`useNestedStrict`值为true，mongoose在执行updates时会使用child schema的`strict`选项的值。

    var childSchema = new Schema({}, { strict: false });
    var parentSchema = new Schema({ child: childSchema },
    { strict: 'throw', useNestedStrict: false });
    var Parent = mongoose.model('Parent', parentSchema);
    Parent.update({}, { 'child.name': 'Luke Skywalker' }, function(error) {
    // Works!
    });

#### Pluggable

schema允许使用[插件][]，这样我们将有用的功能打包成插件，在社区或者项目中实现共享。

## 下一章 —— [SchemaTypes][]

[schemaTypes]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/SchemaTypes.md
[geoJSON]:https://docs.mongodb.com/manual/reference/geojson/
[插件]:http://mongoosejs.com/docs/plugins.html
[collation总览]:http://thecodebarbarian.com/a-nodejs-perspective-on-mongodb-34-collations
[collation]:https://docs.mongodb.com/manual/reference/collation/
[你知道你正在做什么]:http://aaronheckmann.tumblr.com/post/48943525537/mongoose-v3-part-1-versioning
[这里]:http://mongoosejs.com/docs/api.html#document_Document-toObject
[点击查看]:http://mongoosejs.com/docs/api.html#document_Document-toObject
[toObject]:http://mongoosejs.com/docs/guide.html#toObject
[strategy]:http://mongodb.github.io/node-mongodb-native/api-generated/replset.html?highlight=strategy
[tag设置]:https://docs.mongodb.com/manual/applications/replication/#tag-sets
[查询＃读取]:http://mongoosejs.com/docs/api.html#query_Query-read
[ReadPreferences]:https://docs.mongodb.com/manual/applications/replication/#replica-set-read-preference
[ensureIndexes]:http://mongoosejs.com/docs/api.html#model_Model.ensureIndexes
[toObject()]:http://mongoosejs.com/docs/api.html#document_Document-toObject
[Model#ensureIndexes]:http://mongoosejs.com/docs/api.html#model_Model.ensureIndexes
[secondary indexes]:https://docs.mongodb.com/manual/indexes/
[链式查询]:http://mongoosejs.com/docs/queries.html
[查看]:http://mongoosejs.com/docs/api.html#schema_Schema.reserved
[SchemaType]:http://mongoosejs.com/docs/api.html#schematype_SchemaType
[quickstart]:http://mongoosejs.com/docs/index.html
[迁移指南]:https://github.com/Automattic/mongoose/blob/master/migrating_to_5.md
[Schema#add]:http://mongoosejs.com/docs/api.html#schema_Schema-add
