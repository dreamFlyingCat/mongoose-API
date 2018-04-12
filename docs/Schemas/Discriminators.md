## `model.discriminator`函数

鉴别器是一种schema继承机制。 它们使您可以拥有多个模型通过重叠相同基础MongoDB集合的schema。

假设你想在一个collection中追踪不同类型的事件，每个事件都有一个时间戳，但是代表点击链接的事件应该有一个URL。你可以使用`model.discriminator()`函数实现它。这个函数有两个参数，模型名称和鉴别器（discriminator）schema。它会返回一个基本schema和鉴别器schema联合的schema。


    var options = {discriminatorKey: 'kind'};

        var eventSchema = new mongoose.Schema({time: Date}, options);
        var Event = mongoose.model('Event', eventSchema);

        // ClickedLinkEvent is a special type of Event that has
        // a URL.
        var ClickedLinkEvent = Event.discriminator('ClickedLink',
        new mongoose.Schema({url: String}, options));

        // When you create a generic event, it can't have a URL field...
        var genericEvent = new Event({time: Date.now(), url: 'google.com'});
        assert.ok(!genericEvent.url);

        // But a ClickedLinkEvent can
        var clickedEvent =
        new ClickedLinkEvent({time: Date.now(), url: 'google.com'});
        assert.ok(clickedEvent.url);

### 辨别器保存到Event模型的集合中

假设您创建了另一个鉴别器来跟踪新用户注册的事件。 这些`SignedUpEvent`实例将与`ClickedLinkEvent`实例作为普通的`event`一起存储在相同的集合中。

    var event1 = new Event({time: Date.now()});
    var event2 = new ClickedLinkEvent({time: Date.now(), url: 'google.com'});
    var event3 = new SignedUpEvent({time: Date.now(), user: 'testuser'});

    var save = function (doc, callback) {
      doc.save(function (error, doc) {
        callback(error, doc);
      });
    };

    async.map([event1, event2, event3], save, function (error) {

      Event.count({}, function (error, count) {
        assert.equal(count, 3);
      });
    });
  
### 辨别器的键

mongoose使用'discriminator key'标识不同鉴别器模型之间的区，默认为__t。 mongoose给schema添加一个字符串类型的路径_t,用于跟踪该文档是哪个鉴别器的实例。

    var event1 = new Event({time: Date.now()});
    var event2 = new ClickedLinkEvent({time: Date.now(), url: 'google.com'});
    var event3 = new SignedUpEvent({time: Date.now(), user: 'testuser'});

    assert.ok(!event1.__t);
    assert.equal(event2.__t, 'ClickedLink');
    assert.equal(event3.__t, 'SignedUp');

### 鉴别器向查询中添加key

鉴别器模型比较特殊，他们可以向查询中附加鉴别器的key。 换句话说，`find()`、`count()`、`aggregate()`等方法均可以使用鉴别器。

    var event1 = new Event({time: Date.now()});
    var event2 = new ClickedLinkEvent({time: Date.now(), url: 'google.com'});
    var event3 = new SignedUpEvent({time: Date.now(), user: 'testuser'});

    var save = function (doc, callback) {
      doc.save(function (error, doc) {
        callback(error, doc);
      });
    };

    async.map([event1, event2, event3], save, function (error) {

      ClickedLinkEvent.find({}, function (error, docs) {
        assert.equal(docs.length, 1);
        assert.equal(docs[0]._id.toString(), event2._id.toString());
        assert.equal(docs[0].url, 'google.com');
      });
    });

### 鉴别赋值预置和后置钩子

  基本schema的前后中间件也会作用在鉴别器上。您还可以给鉴别器schema添加中间件，而不会影响基本的schema。


    var options = {discriminatorKey: 'kind'};

    var eventSchema = new mongoose.Schema({time: Date}, options);
    var eventSchemaCalls = 0;
    eventSchema.pre('validate', function (next) {
      ++eventSchemaCalls;
      next();
    });
    var Event = mongoose.model('GenericEvent', eventSchema);

    var clickedLinkSchema = new mongoose.Schema({url: String}, options);
    var clickedSchemaCalls = 0;
    clickedLinkSchema.pre('validate', function (next) {
      ++clickedSchemaCalls;
      next();
    });
    var ClickedLinkEvent = Event.discriminator('ClickedLinkEvent',
      clickedLinkSchema);

    var event1 = new ClickedLinkEvent();
    event1.validate(function() {
      assert.equal(eventSchemaCalls, 1);
      assert.equal(clickedSchemaCalls, 1);

      var generic = new Event();
      generic.validate(function() {
        assert.equal(eventSchemaCalls, 2);
        assert.equal(clickedSchemaCalls, 1);
      });
    });

### 处理自定义_id字段

鉴别器字段是基本schema字段和鉴别器schema字段的合集。鉴别器schema中的字段优先级较高，_id字段例外。
  
您可以通过在schema中将_id选项设置为false来解决此问题，如下所示。  

    var options = {discriminatorKey: 'kind'};

    // Base schema has a String `_id` and a Date `time`...
    var eventSchema = new mongoose.Schema({_id: String, time: Date},
      options);
    var Event = mongoose.model('BaseEvent', eventSchema);

    var clickedLinkSchema = new mongoose.Schema({
      url: String,
      time: String
    }, options);
    // But the discriminator schema has a String `time`, and an implicitly added
    // ObjectId `_id`.
    assert.ok(clickedLinkSchema.path('_id'));
    assert.equal(clickedLinkSchema.path('_id').instance, 'ObjectID');
    var ClickedLinkEvent = Event.discriminator('ChildEventBad',
      clickedLinkSchema);

    var event1 = new ClickedLinkEvent({ _id: 'custom id', time: '4pm' });
    // Woops, clickedLinkSchema overwrites the `time` path, but **not**
    // the `_id` path because that was implicitly added.
    assert.ok(typeof event1._id === 'string');
    assert.ok(typeof event1.time === 'string');

### `Model.create()`中使用鉴别器

当你使用`Model.create()`时，mongoose会从你的鉴别器键中提取正确的类型。
  
    var Schema = mongoose.Schema;
    var shapeSchema = new Schema({
      name: String
    }, { discriminatorKey: 'kind' });

    var Shape = db.model('Shape', shapeSchema);

    var Circle = Shape.discriminator('Circle',
      new Schema({ radius: Number }));
    var Square = Shape.discriminator('Square',
      new Schema({ side: Number }));

    var shapes = [
      { name: 'Test' },
      { kind: 'Circle', radius: 5 },
      { kind: 'Square', side: 10 }
    ];
    Shape.create(shapes, function(error, shapes) {
      assert.ifError(error);
      assert.ok(shapes[0] instanceof Shape);
      assert.ok(shapes[1] instanceof Circle);
      assert.equal(shapes[1].radius, 5);
      assert.ok(shapes[2] instanceof Square);
      assert.equal(shapes[2].side, 10);
    });


### 数组中嵌套鉴别器

您还可以在嵌套的文档数组上定义鉴别器。 嵌入式鉴别器的不同之处在于，不同的鉴别器类型存储在同一个文档数组中（一个文档中）而不是同一个集合。 换句话说，嵌入式鉴别器允许您将不同schema的子文档存储在同一个数组中。

作为一个通用的最佳实践，确保所有的钩子函数在·`discriminator()`之前声明。

    var eventSchema = new Schema({ message: String },
      { discriminatorKey: 'kind', _id: false });

    var batchSchema = new Schema({ events: [eventSchema] });

    // `batchSchema.path('events')` gets the mongoose `DocumentArray`
    var docArray = batchSchema.path('events');

    // The `events` array can contain 2 different types of events, a
    // 'clicked' event that requires an element id that was clicked...
    var clickedSchema = new Schema({
      element: {
        type: String,
        required: true
      }
    }, { _id: false });
    // Make sure to attach any hooks to `eventSchema` and `clickedSchema`
    // **before** calling `discriminator()`.
    var Clicked = docArray.discriminator('Clicked', clickedSchema);

    // ... and a 'purchased' event that requires the product that was purchased.
    var Purchased = docArray.discriminator('Purchased', new Schema({
      product: {
        type: String,
        required: true
      }
    }, { _id: false }));

    var Batch = db.model('EventBatch', batchSchema);

    // Create a new batch of events with different kinds
    var batch = {
      events: [
        { kind: 'Clicked', element: '#hero', message: 'hello' },
        { kind: 'Purchased', product: 'action-figure-1', message: 'world' }
      ]
    };

    Batch.create(batch).
      then(function(doc) {
        assert.equal(doc.events.length, 2);

        assert.equal(doc.events[0].element, '#hero');
        assert.equal(doc.events[0].message, 'hello');
        assert.ok(doc.events[0] instanceof Clicked);

        assert.equal(doc.events[1].product, 'action-figure-1');
        assert.equal(doc.events[1].message, 'world');
        assert.ok(doc.events[1] instanceof Purchased);

        doc.events.push({ kind: 'Purchased', product: 'action-figure-2' });
        return doc.save();
      }).
      then(function(doc) {
        assert.equal(doc.events.length, 3);

        assert.equal(doc.events[2].product, 'action-figure-2');
        assert.ok(doc.events[2] instanceof Purchased);

        done();
      }).
      catch(done);

### 数组中的递归嵌套鉴别器

递归嵌套鉴别器
  
    var singleEventSchema = new Schema({ message: String },
      { discriminatorKey: 'kind', _id: false });

    var eventListSchema = new Schema({ events: [singleEventSchema] });

    var subEventSchema = new Schema({
       sub_events: [singleEventSchema]
    }, { _id: false });

    var SubEvent = subEventSchema.path('sub_events').discriminator('SubEvent', subEventSchema)
    eventListSchema.path('events').discriminator('SubEvent', subEventSchema);

    var Eventlist = db.model('EventList', eventListSchema);

    // Create a new batch of events with different kinds
    var list = {
      events: [
        { kind: 'SubEvent', sub_events: [{kind:'SubEvent', sub_events:[], message:'test1'}], message: 'hello' },
        { kind: 'SubEvent', sub_events: [{kind:'SubEvent', sub_events:[{kind:'SubEvent', sub_events:[], message:'test3'}], message:'test2'}], message: 'world' }
      ]
    };

    Eventlist.create(list).
      then(function(doc) {
        assert.equal(doc.events.length, 2);

        assert.equal(doc.events[0].sub_events[0].message, 'test1');
        assert.equal(doc.events[0].message, 'hello');
        assert.ok(doc.events[0].sub_events[0] instanceof SubEvent);

        assert.equal(doc.events[1].sub_events[0].sub_events[0].message, 'test3');
        assert.equal(doc.events[1].message, 'world');
        assert.ok(doc.events[1].sub_events[0].sub_events[0] instanceof SubEvent);

        doc.events.push({kind:'SubEvent', sub_events:[{kind:'SubEvent', sub_events:[], message:'test4'}], message:'pushed'});
        return doc.save();
      }).
      then(function(doc) {
        assert.equal(doc.events.length, 3);

        assert.equal(doc.events[2].message, 'pushed');
        assert.ok(doc.events[2].sub_events[0] instanceof SubEvent);

        done();
      }).
      catch(done);
  

  ## 下一章 —— [Plugins][]


[Plugins]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Plugins.md