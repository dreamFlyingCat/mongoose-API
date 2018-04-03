### Queries

models提供了多个静态方法，用于检索Documents。任何涉及指定查询条件的models方法都能通过两个方法执行。根据是否传入回调函数，有以下两种情况：
   1. 传入callback，操作将立即执行，结果传给回调函数；
   2. 不传callback，返回一个查询实例，它将提供一个特殊的查询生成接口。
对于第二种情况，Query实例对象有`.then()`方法，所以可以用作promise实例对象。

执行查询时候传入callback回调函数，查询结果会以JSON格式返回。JSON文档的语法与[MongoDB shell][]相同。
    
    var Person = mongoose.model('Person', yourSchema);

    // find each person with a last name matching 'Ghost', selecting the `name` and `occupation` fields
    Person.findOne({ 'name.last': 'Ghost' }, 'name occupation', function (err, person) {
        if (err) return handleError(err);
        // Prints "Space Ghost is a talk show host".
        console.log('%s %s is a %s.', person.name.first, person.name.last,person.occupation);
    });

上例中，查询会被立即执行并将结果传入callback。Mongoose中所有的callbacks都遵循模式：`callback(error, result)`。如果查询时出错，`error`参数会包含一个错误对象且`result`为null。如果查询成功，`error`参数为null，`result`为查询结果。

Mongoose中任何情况下传入回调函数，都遵循`callback(error, results)`模式。查询结果取决于执行的操作：`findOne()`为可能为空的单document，`find()`为多个document，`count()`为document的个数，`update()`为受影响的document的数量。Model的[API][]文档详细介绍了回调函数的可能结果。

现在看看在没有回调的情况下会发生什么：：

    // find each person with a last name matching 'Ghost'
    var query = Person.findOne({ 'name.last': 'Ghost' });

    // selecting the `name` and `occupation` fields
    query.select('name occupation');

    // execute the query at a later time
    query.exec(function (err, person) {
        if (err) return handleError(err);
        // Prints "Space Ghost is a talk show host."
        console.log('%s %s is a %s.', person.name.first, person.name.last,person.occupation);
    });

上面代码中，`query`变量是Query的一种类型。`Query`允许你建立链式的查询语法，而不是指定JSON对象。下面两个例子是等价的：

     // With a JSON doc
     Person.
        find({
            occupation: /host/,
            'name.last': 'Ghost',
            age: { $gt: 17, $lt: 66 },
            likes: { $in: ['vaporizing', 'talking'] }
        }).
        limit(10).
        sort({ occupation: -1 }).
        select({ name: 1, occupation: 1 }).
        exec(callback);
    
     // Using query builder
     Person.
        find({ occupation: /host/ }).
        where('name.last').equals('Ghost').
        where('age').gt(17).lt(66).
        where('likes').in(['vaporizing', 'talking']).
        limit(10).
        sort('-occupation').
        select('name occupation').
        exec(callback);

可以在[API][]文档中查看全部的Query辅助函数列表。

### 引用其它的document

MongoDB没有joins功能，但是有时我们想引用其它collections中的文档。这就是[population][]的由来。[查看][]如何在查询结果中包含其他collection的document。

### 流

你可以流式从MongoDB中查询文档。调用[Query#cursor()][]会返回[QueryCursor][]实例。

[MongoDB shell]:http://docs.mongodb.org/manual/tutorial/query-documents/
[API]:http://mongoosejs.com/docs/api.html#model-js
[查看]:http://mongoosejs.com/docs/api.html#query_Query-populate
[Query#cursor()]:http://mongoosejs.com/docs/api.html#querycursor-js
[QueryCursor]:http://mongoosejs.com/docs/api.html#query_Query-cursor


        