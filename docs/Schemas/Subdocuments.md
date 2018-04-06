## Sub Docs

SubDocuments是嵌套在其他documents中的documents，这意味着你可以在Schemas中嵌套Schemas。Mongoose支持两种不同形式的subdocuments：subdocuments数组和单独嵌套的subdocuments（mongoose版本大于等于4.2.0）。

    var childSchema = new Schema({ name: 'string' });

        var parentSchema = new Schema({
        // Array of subdocuments
        children: [childSchema],
        // Single nested subdocuments. Caveat: single nested subdocs only work
        // in mongoose >= 4.2.0
        child: childSchema
    });

sub-document享有所有与普通document相同的特征。嵌套的Schemas也可以有[middleware][http://mongoosejs.com/docs/middleware.html]和自定义的[validation][]，逻辑和使用与顶层schemas相同。subdocument最大的不同是不能单独保存，当它们的顶层父document保存时它们才被保存。

    var Parent = mongoose.model('Parent', parentSchema);
    var parent = new Parent({ children: [{ name: 'Matt' }, { name: 'Sarah' }] })
    parent.children[0].name = 'Matthew';

    // `parent.children[0].save()` is a no-op, it triggers middleware but
    // does **not** actually save the subdocument. You need to save the parent
    // doc.
    parent.save(callback);

Subdocuments也有类似于顶层documents的`save`和`validate`中间件。调用`save()`保存父级document会触发所有的subdocuments的`save()`和`validate()`。

    childSchema.pre('save', function (next) {
        if ('invalid' == this.name) {
            return next(new Error('#sadpanda'));
        }
        next();
        });

        var parent = new Parent({ children: [{ name: 'invalid' }] });
        parent.save(function (err) {
        console.log(err.message) // #sadpanda
    });

Subdocuments的`pre('save')`和`pre('validate')`中间件会在顶层ducument的`pre('save')`之前和`pre('validate')`之后执行。这是因为validating是一种内置的中间件，会在`save()`之前执行。

    // Below code will print out 1-4 in order
    var childSchema = new mongoose.Schema({ name: 'string' });

    childSchema.pre('validate', function(next) {
        console.log('2');
        next();
    });

    childSchema.pre('save', function(next) {
        console.log('3');
        next();
    });

    var parentSchema = new mongoose.Schema({
    child: childSchema,
        });

    parentSchema.pre('validate', function(next) {
        console.log('1');
        next();
    });

    parentSchema.pre('save', function(next) {
        console.log('4');
        next();
    });

### 查找sub-document

每个subdocument都有一个`_id`,DocumentArrays有特殊的 id 方法来通过_id来查找document。。
### 向数组中追加sub-docs

mongoose数组方法如[push][]、[unshift][]、[addToSet][]等将参数显式转换成恰当的类型。

    var Parent = mongoose.model('Parent');
    var parent = new Parent;

    // create a comment
    parent.children.push({ name: 'Liesl' });
    var subdoc = parent.children[0];
    console.log(subdoc) // { _id: '501d86090d371bab2c0341c5', name: 'Liesl' }
    subdoc.isNew; // true

    parent.save(function (err) {
        if (err) return handleError(err)
        console.log('Success!');
    });

也可以通过MongooseArrays的[create][]方法直接创建sub-docs，无需添加的数组中。

    var newdoc = parent.children.create({ name: 'Aaron' });

### 删除subdocs

移除subdocuments可以使用[remove][]方法，对于subdocument数组来说，该方法等同于使用`.pull()`，在单独嵌套的subdocument使用该方法，相当于设置subdocument为`null`。

    // Equivalent to `parent.children.pull(_id)`
    parent.children.id(_id).remove();
    // Equivalent to `parent.child = null`
    parent.child.remove();
    parent.save(function (err) {
        if (err) return handleError(err);
        console.log('the subdocs were removed');
    });

### 数组的交替声明语法

如果你在Schema中声明一个对象数组的属性，mongoose会自动的将对象字面量转换成Schema（如果你不需要访问的子文档schena的实例，你也可以通过简单传递一个对象字面量声明子文档）。

    var parentSchema = new Schema({
        children: [{ name: 'string' }]
    });
    // Equivalent
    var parentSchema = new Schema({
        children: [new Schema({ name: 'string' })]
    });

## 下一章 —— [Queries][]

[Queries]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Queries.md
[remove]:http://mongoosejs.com/docs/api.html#types_array_MongooseArray.remove
[push]:http://mongoosejs.com/docs/api.html#types_array_MongooseArray.push
[unshift]:http://mongoosejs.com/docs/api.html#types_array_MongooseArray.unshift
[addToSet]:http://mongoosejs.com/docs/api.html#types_array_MongooseArray.addToSet
[id]:http://mongoosejs.com/docs/api.html#types_documentarray_MongooseDocumentArray-id
[middleware]: http://mongoosejs.com/docs/middleware.html
[validation]: http://mongoosejs.com/docs/validation.html