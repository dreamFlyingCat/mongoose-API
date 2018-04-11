## Populate

MongoDB>=3.2版本中具有类似连接`$lookup`的聚合操作符。 Mongoose有一个更强大的替代方法叫做`populate()`，它可以让你引用其他集合中的文档。

Populate是自动将文档中的指定path替换为来自其他集合的文档的过程。我们可以Populate单个文档、多个文档、普通对象，多个普通对象或从查询返回的所有对象。 我们来看一些例子。

    var mongoose = require('mongoose');
    var Schema = mongoose.Schema;

    var personSchema = Schema({
        _id: Schema.Types.ObjectId,
        name: String,
        age: Number,
        stories: [{ type: Schema.Types.ObjectId, ref: 'Story' }]
    });

    var storySchema = Schema({
        author: { type: Schema.Types.ObjectId, ref: 'Person' },
        title: String,
        fans: [{ type: Schema.Types.ObjectId, ref: 'Person' }]
    });

    var Story = mongoose.model('Story', storySchema);
    var Person = mongoose.model('Person', personSchema);

现在我们创建了两个模型，`Person`的模型有一个`stories`字段类，指定类型为`ObjectId`的数组。`ref`选项告诉mongoose `Story`模型在population期间会使用哪个模型。所有存储的`_id`必须是`Story`模型生成的document的`_id`。

>注意：`ObjectId`、`Number`、`String`、`Buffer`集中类型均可以作为refs。但是建议使用`ObjectId`，除非你是高级用户并且有合理的理由使用其他的类型作为refs。

### 保存refs

将refs保存到其他文档的工作方式与您通常保存属性的方式相同，只需指定_id值即可：

    var author = new Person({
        _id: new mongoose.Types.ObjectId(),
        name: 'Ian Fleming',
        age: 50
    });

    author.save(function (err) {
        if (err) return handleError(err);

        var story1 = new Story({
            title: 'Casino Royale',
            author: author._id    // assign the _id from the person
        });

        story1.save(function (err) {
            if (err) return handleError(err);
            // thats it!
        });
    });

### Population

到目前为止，我们没有做过太多的改变。 我们只创建了`Person `和`Story`的模型。 现在让我们看看使用查询构建器来关联故事的作者：

    Story.
        findOne({ title: 'Casino Royale' }).
        populate('author').
        exec(function (err, story) {
            if (err) return handleError(err);
            console.log('The author is %s', story.author.name);
            // prints "The author is Ian Fleming"
        });

populated的路径不再设置为它们的原始_id，而是由之前从数据库单独执行查询返回的mongoose文档替换。


数组形式的refs也是相同的工作方法，查询时调用`populate`方法会返回一个文档的数组替换掉原来的`_id`。

### 设置populated的字段

在mongoose>=4.0版本中，你可以手动的populate一个字段：

    Story.findOne({ title: 'Casino Royale' }, function(error, story) {
        if (error) {
            return handleError(error);
        }
        story.author = author;
        console.log(story.author.name); // prints "Ian Fleming"
    });

### 字段选择

如果我们只想返回populated文档中特定的一些字段怎么办？可以通过将需要的[字段名称][]作为第二个参数传递给populate方法来完成：

      Story.
        findOne({ title: /casino royale/i }).
        populate('author', 'name'). // only return the Persons name
        exec(function (err, story) {
            if (err) return handleError(err);

            console.log('The author is %s', story.author.name);
            // prints "The author is Ian Fleming"

            console.log('The authors age is %s', story.author.age);
            // prints "The authors age is null'
        });

### populating多个路径

如果你希望一次populate多个路径怎么办？

      Story.
        find(...).
        populate('fans').
        populate('author').
        exec();

如果你对相同的路径多次的populate，仅最后一次会生效。

      // The 2nd `populate()` call below overwrites the first because they
      // both populate 'fans'.
      Story.
        find().
        populate({ path: 'fans', select: 'name' }).
        populate({ path: 'fans', select: 'email' });
        // The above is equivalent to:
        Story.find().populate({ path: 'fans', select: 'email' });

### 查询条件和其他的选项

如果你想要基于fans的年龄做筛选，并只返回名字字段，最多返回5条怎么办呢？

     Story.
        find(...).
        populate({
            path: 'fans',
            match: { age: { $gte: 21 }},
            // Explicitly exclude `_id`, see http://bit.ly/2aEfTdB
            select: 'name -_id',
            options: { limit: 5 }
        }).
        exec();


### refs to children

如果我们使用auther对象，可能无法获取stories列表，因为没有对象曾经被`pushed`盗author.storeis中。

这里有两点需要考虑。首先，你可能想要作者知道哪些故事是他们的。通常，你的schema应该建立‘一对多’的关系，通过使用一个父指针指向多个孩子。 但是，如果您有充足的理由使用子指针的数组，可以按如下所示将文档推送到数组中。

    author.stories.push(story1);
    author.save(callback);

这允许我们查找和populate组合结果：

      Person.
        findOne({ name: 'Ian Fleming' }).
        populate('stories'). // only works if we pushed refs to children
        exec(function (err, person) {
            if (err) return handleError(err);
            console.log(person);
        });

有争议的是，如果我们同时使用两组指针，他们可能会不同步。我们可以抛弃populate直接使用`find()`查询感兴趣的故事。

      Story.
        find({ author: author._id }).
        exec(function (err, stories) {
            if (err) return handleError(err);
            console.log('The stories are an array: ', stories);
        });

除非指定了[lean][]选项，否则populate查询返回的文档是功能齐全的文档，可删除、可保存。不要讲他们与子文档混淆。条用`removeing`删除文档时候请小心，因为您将直接从数据库中删除它，而不仅仅是从父指针的数组中。

### populate一个已近存在的文档

如果你有一个已经存在的mongoose文档并且想要populate它的一些路径，mongoose>=3.6版本支持[document#populate()][]方法。

### populate多个存在的文档

如果你有一个或多个已储存在的mongoose文档或者是普通的对象(例如[mapReduce ](http://mongoosejs.com/docs/api.html#model_Model.mapReduce)输出)，我们可以在mongoose>=3.6版本中使用[Model.populate()][]方法。使用该方法，`document＃populate()`和`query＃populate()`可用于populate文档。

### populate多个级别

假设你有一个想要追踪用户的朋友的user schema。

    var userSchema = new Schema({
        name: String,
        friends: [{ type: ObjectId, ref: 'User' }]
    });

使用populate可以获取用户的朋友列表，但是如果你还想获取用户朋友的朋友该怎么办呢？ 可以指定`populate`选项告诉mongoose去populate用户朋友的朋友数组：

     User.
        findOne({ name: 'Val' }).
        populate({
            path: 'friends',
            // Get friends of friends - populate the 'friends' array for every friend
            populate: { path: 'friends' }
        });


### 跨数据库populate

假设你有一个代表events（事件）的schema和一个代表conversations（会话）的schema。每个事件都有相应的会话线程。

    var eventSchema = new Schema({
        name: String,
        // The id of the corresponding conversation
        // Notice there's no ref here!
        conversation: ObjectId
    });
    var conversationSchema = new Schema({
        numMessages: Number
    });

另外，假设事件和会话存储在不同的MongoDB实例中。  

    var db1 = mongoose.createConnection('localhost:27000/db1');
    var db2 = mongoose.createConnection('localhost:27001/db2');

    var Event = db1.model('Event', eventSchema);
    var Conversation = db2.model('Conversation', conversationSchema);

在这种情况下，你无法正常的populate，因为`conversation`字段总是null，因为`populate()`不知道使用哪个model。你可以[明确指明model][]解决这个问题。

      Event.
        find().
        populate({ path: 'conversation', model: Conversation }).
        exec(function(error, docs) { /* ... */ });

这被称为“cross-databse populate”，因为它让你能够跨mongoDB数据库甚至跨mongoDB实例populate。

### 动态参考

Mongoose也可以同时从多个集合中populate，假设您有一个user schema，它有一个"connections"的数组字段-用户可以连接到其他的用户或者组织。

    var userSchema = new Schema({
        name: String,
        connections: [{
            kind: String,
            item: { type: ObjectId, refPath: 'connections.kind' }
        }]
    });

    var organizationSchema = new Schema({ name: String, kind: String });

    var User = mongoose.model('User', userSchema);
    var Organization = mongoose.model('Organization', organizationSchema);

上面的refPath属性意味着mongoose会查看`connections.kind`路径以决定使用populate的模型。换句话说，refPath属性让ref属性为动态连接。

    // Say we have one organization:
    // `{ _id: ObjectId('000000000000000000000001'), name: "Guns N' Roses", kind: 'Band' }`
    // And two users:
    // {
    //   _id: ObjectId('000000000000000000000002')
    //   name: 'Axl Rose',
    //   connections: [
    //     { kind: 'User', item: ObjectId('000000000000000000000003') },
    //     { kind: 'Organization', item: ObjectId('000000000000000000000001') }
    //   ]
    // },
    // {
    //   _id: ObjectId('000000000000000000000003')
    //   name: 'Slash',
    //   connections: []
    // }
    User.
        findOne({ name: 'Axl Rose' }).
        populate('connections.item').
        exec(function(error, doc) {
            // doc.connections[0].item is a User doc
            // doc.connections[1].item is an Organization doc
        });

### 虚拟字段的populate

> 4.5.0版本中新增

到目前为止，您只基于`_id`字段进行populate。 但是，这有时不是正确的选择。 特别是一对多关系中，指向无边界增长的数组（是MongoDB的反模式）时。 使用mongoose的虚拟虚拟字段，你可以定义更复杂的文件之间的关系。

    var PersonSchema = new Schema({
        name: String,
        band: String
    });

    var BandSchema = new Schema({
        name: String
    });
    BandSchema.virtual('members', {
        ref: 'Person', // The model to use
        localField: 'name', // Find people where `localField`
        foreignField: 'band', // is equal to `foreignField`
        // If `justOne` is true, 'members' will be a single doc as opposed to
        // an array. `justOne` is false by default.
        justOne: false
    });

    var Person = mongoose.model('Person', PersonSchema);
    var Band = mongoose.model('Band', BandSchema);

    /**
    * Suppose you have 2 bands: "Guns N' Roses" and "Motley Crue"
    * And 4 people: "Axl Rose" and "Slash" with "Guns N' Roses", and
    * "Vince Neil" and "Nikki Sixx" with "Motley Crue"
    */
    Band.find({}).populate('members').exec(function(error, bands) {
        /* `bands.members` is now an array of instances of `Person` */
    });

请注意virtual默认不包含在`toJSON()`的输出中。如果在使用依赖`JSON.stringfy()`的函数时，例如express的`res.json()`函数，你想要显示populate的虚拟字段，设置schema的`toJSON`选项中的`virtual:true`。

    // Set `virtuals: true` so `res.json()` works
    var BandSchema = new Schema({
    name: String
    }, { toJSON: { virtuals: true } });

如果你使用populate做表的关联，请确保包含`foreignField`字段。

    Band.
        find({}).
        populate({ path: 'members', select: 'name' }).
        exec(function(error, bands) {
            // Won't work, foreign field `band` is not selected in the projection
        });

    Band.
        find({}).
        populate({ path: 'members', select: 'name band' }).
        exec(function(error, bands) {
            // Works, foreign field `band` is selected
        });

### 中间件中的populate

你可以在预置或者后置的钩子中使用populate。 如果您想始终populate某个字段，请查看[mongoose-autopopulate][]插件。

    // Always attach `populate()` to `find()` calls
    MySchema.pre('find', function() {
        this.populate('user');
    });

 

    // Always `populate()` after `find()` calls. Useful if you want to selectively populate
    // based on the docs found.
    MySchema.post('find', async function(docs) {
        for (let doc of docs) {
            if (doc.isPublic) {
            await doc.populate('user').execPopulate();
            }
        }
    }); 


    // `populate()` after saving. Useful for sending populated data back to the client in an
    // update API endpoint
    MySchema.post('save', function(doc, next) {
        doc.populate('user').execPopulate(function() {
            next();
        });
    });







## 下一章 —— [Discriminators][]

[Discriminators]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Discriminators.md
[mongoose-autopopulate]:http://npmjs.com/package/mongoose-autopopulate
[明确指明model]:http://mongoosejs.com/docs/api.html#model_Model.populate
[Model.populate()]:http://mongoosejs.com/docs/api.html#model_Model.populate
[document#populate()]:http://mongoosejs.com/docs/api.html#document_Document-populate
[lean]:http://mongoosejs.com/docs/api.html#query_Query-lean
[字段名称语法]:http://mongoosejs.com/docs/api.html#query_Query-select