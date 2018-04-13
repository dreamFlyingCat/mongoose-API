> 版权声明：本文为博主辛苦翻译，未经博主允许不得转载。

## 中间件

中间件（也称为预置和后置挂钩）在执异步函数执行期间使用。中间件在schema级别上指定，用于编写插件。 Mongoose 4.x有4种中间件：文档中间件，模型中间件，聚合中间件和查询中间件。 文档中间件支持以下功能，在文档中间件中，this指向文档。

   * [init][]
   * [validate][]
   * [save][]
   * [remove][]


查询中间件支持以下Model和Query的函数，在查询中间件函数中，this指向查询实例。

   * [count][]
   * [find][]
   * [findOne][]
   * [findOneAndRemove][]
   * [findOneAndUpdate][]
   * [update][]

聚合中间件在`MyModel.aggregate()`中使用，当你在一个聚合对象上调用`exec()`时会执行聚合中间件。聚合中间件中,`this`指向[聚合对象][]。

   * [aggregate][]

以下model方法支持模型中间件，模型中间件函数中`this`指向model。
 
   * [insertMany][]

所有类型的中间间均支持pre和post钩子，往下查看中间件如何在pre和post中间件中工作。

>注意：model的`remove()`方法没有钩子函数，只有document的`remove()`方法才有，当你调用`myDoc.remove()`而不是`MyModel.remove()`时触发。

>注意：`create()`方法会触发`save()`钩子函数。

### pre

这有两种类型的预置钩子，串行和并行。

#### Serial （串行）

有多个中间件时会一个接着一个的执行，每个中间件内部调用`next`。

    var schema = new Schema(..);
    schema.pre('save', function(next) {
        // do stuff
        next();
    });

mongoose 5.x版本中你可以选择使用返回promise对象的函数而不是在函数中手动的调用`next()`。尤其是使用`async/await`特别方便。

    schema.pre('save', function() {
        return doStuff().
            then(() => doMoreStuff());
    });

    // Or, in Node.js >= 7.6.0:
    schema.pre('save', async function() {
        await doStuff();
        await doMoreStuff();
    });

`next()`调用后并不会阻止中间件函数中余下代码的执行。可以选择使用`return`语句阻止剩余代码的执行。

    var schema = new Schema(..);
    schema.pre('save', function(next) {
        if (foo()) {
            console.log('calling next!');
            // `return next();` will make sure the rest of this function doesn't run
            /*return*/ next();
        }
        // Unless you comment out the `return` above, 'after next' will print
        console.log('after next');
    });

#### Parallel （并行）

并行中间件提供更精细的流量控制

    var schema = new Schema(..);

    // `true` means this is a parallel middleware. You **must** specify `true`
    // as the second parameter if you want to use parallel middleware.
    schema.pre('save', true, function(next, done) {
        // calling next kicks off the next middleware in parallel
        next();
        setTimeout(done, 100);
    });

上例中save的钩子函数直到`done`被调用后才会执行。


### 用例

中间件对于原型化的模型逻辑很有用。 以下是其他的一些用处：

   * 复杂的验证
   * 删除相关文档（删除用户时删除他的所有博客帖子）
   * 异步默认值
   * 特定操作触发的异步任务

### 错误处理

如果调用`next`或者`done`时传入一个错误对象，之后的中间件将被阻断，错误会传递给callback（回调函数）。

    schema.pre('save', function(next) {
        var err = new Error('something went wrong');
        next(err);
    });

    // later...

    myDoc.save(function(err) {
        console.log(err.message) // something went wrong
    });

### 后置中间件

后置中间件在所有的钩子方法和预置中间件执行完毕后调用。

    schema.post('init', function(doc) {
        console.log('%s has been initialized from the db', doc._id);
    });
    schema.post('validate', function(doc) {
        console.log('%s has been validated (but not saved yet)', doc._id);
    });
    schema.post('save', function(doc) {
        console.log('%s has been saved', doc._id);
    });
    schema.post('remove', function(doc) {
        console.log('%s has been removed', doc._id);
    });


#### 异步后置中间件

如果你的函数的参数大于等于2个，mongoose会认为第二个参数是一个`next()`函数，你将会调用`next`触发执行之后队列中的中间件。

    // Takes 2 parameters: this is an asynchronous post hook
    schema.post('save', function(doc, next) {
        setTimeout(function() {
            console.log('post1');
            // Kick off the second post hook
            next();
        }, 10);
    });

    // Will not execute until the first middleware calls `next()`
    schema.post('save', function(doc, next) {
        console.log('post2');
        next();
    });

### 保存/验证钩子

`save()`函数会触发`validate()`钩子，因为mongoose内置了`pre('save')`会调用`validate()`。这意味着所有的`pre('validate')`和`post('validate')`中间件均会在`pre('save')`之前调用。

    schema.pre('validate', function() {
        console.log('this gets printed first');
    });
    schema.post('validate', function() {
        console.log('this gets printed second');
    });
    schema.pre('save', function() {
        console.log('this gets printed third');
    });
    schema.post('save', function() {
        console.log('this gets printed fourth');
    });

###  findAndUpdate()和Query中间件注意事项

调用`update()`和`findOneAndUpdate()`时预置和后置的`save()`钩子并不会被执行。可以在[github issue][]中查看更多的细节。mongoose 4.0版本中介绍了不同的hooks函数。

    schema.pre('find', function() {
        console.log(this instanceof mongoose.Query); // true
        this.start = Date.now();
    });

    schema.post('find', function(result) {
        console.log(this instanceof mongoose.Query); // true
        // prints returned documents
        console.log('find() returned ' + JSON.stringify(result));
        // prints number of milliseconds the query took
        console.log('find() took ' + (Date.now() - this.start) + ' millis');
    });

查询中间件与文档中间件有一个很小但是很重要的区别：在文档中间件中，this指向要被更新的文档，而在查询中间件中，this不必要指向更新的文档，它指向查询对象。

例如，如果你想要每次调用`update()`时给每个更新的文档都都增加`updatedAt`时间戳字段，你可以使用下面的预置钩子。

    schema.pre('update', function() {
        this.update({},{ $set: { updatedAt: new Date() } });
    });

### 错误处理中间件

> 4.5.0版本中新增

正常情况下，一旦某个中间件发生错误会通过`next()`传递错误，中间件会立即停止执行。但是，有一种特殊的后置中间件被称为'错误处理中间件'，当错误发生时会被执行。

错误处理中间件额外接受一个参数：'error'，报错时候作为第一个参数传给处理函数。该中间件可以任意转换错误。

    var schema = new Schema({
        name: {
            type: String,
            // Will trigger a MongoError with code 11000 when
            // you save a duplicate
            unique: true
        }
    });

    // Handler **must** take 3 parameters: the error that occurred, the document
    // in question, and the `next()` function
    schema.post('save', function(error, doc, next) {
        if (error.name === 'MongoError' && error.code === 11000) {
            next(new Error('There was a duplicate key error'));
        } else {
            next(error);
        }
    });

    // Will trigger the `post('save')` error handler
    Person.create([{ name: 'Axl Rose' }, { name: 'Axl Rose' }]);

错误处理中间件也适用于查询中间件。 您可以给`update()`定义后置钩子来捕获MongoDB重复键的错误。

    // The same E11000 error can occur when you call `update()`
    // This function **must** take 3 parameters. If you use the
    // `passRawResult` function, this function **must** take 4
    // parameters
    schema.post('update', function(error, res, next) {
        if (error.name === 'MongoError' && error.code === 11000) {
            next(new Error('There was a duplicate key error'));
        } else {
            next(error);
        }
    });

    var people = [{ name: 'Axl Rose' }, { name: 'Slash' }];
    Person.create(people, function(error) {
        Person.update({ name: 'Slash' }, { $set: { name: 'Axl Rose' } }, function(error) {
            // `error.message` will be "There was a duplicate key error"
        });
    });

## 下一章 —— [Populate][]


[Populate]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Populate.md
[github issue]:http://github.com/Automattic/mongoose/issues/964
[aggregate]:http://mongoosejs.com/docs/api.html#model_Model.aggregate
[insertMany]:http://mongoosejs.com/docs/api.html#model_Model.insertMany
[http://mongoosejs.com/docs/api.html#model_Model.aggregate]:http://mongoosejs.com/docs/api.html#model_Model.aggregate
[聚合对象]:http://mongoosejs.com/docs/api.html#model_Model.aggregate
[count]:http://mongoosejs.com/docs/api.html#query_Query-count
[find]:http://mongoosejs.com/docs/api.html#query_Query-find
[findOne]:http://mongoosejs.com/docs/api.html#query_Query-findOne
[findOneAndRemove]:http://mongoosejs.com/docs/api.html#query_Query-findOneAndRemove
[findOneAndUpdate]:http://mongoosejs.com/docs/api.html#query_Query-findOneAndUpdate
[update]:http://mongoosejs.com/docs/api.html#query_Query-update
[init]: http://mongoosejs.com/docs/api.html#document_Document-init
[validate]:http://mongoosejs.com/docs/api.html#document_Document-validate
[save]:http://mongoosejs.com/docs/api.html#model_Model-save
[remove]:http://mongoosejs.com/docs/api.html#model_Model-remove