> 版权声明：本文为博主辛苦翻译，未经博主允许不得转载。

## Models
Modes是由Schema编译而成的假想（fancy）构造器，具有抽象属性和行为。Model的每一个实例（instance）就是一个document。document可以保存到数据库和从数据库返回。document的创建和搜索均可以通过操作model完成。

### 定义你的第一个Model

     var schema = new mongoose.Schema({ name: 'string', size: 'string' });
     var Tank = mongoose.model('Tank', schema);

第一个参数是Model对应的collection名称的但是形式。因此，对于上面的实例，Tank model对应名称为tanks的collection。Model是由Schema编译生产的，在调用.model()之前，需要先定义好Schema。

### 创建documents

Documents是Model的实例，创建和保存document到数据库非常的简单：

    var Tank = mongoose.model('Tank', yourSchema);

    var small = new Tank({ size: 'small' });
    small.save(function (err) {
    if (err) return handleError(err);
    // saved!
    })

    // or

    Tank.create({ size: 'small' }, function (err, small) {
    if (err) return handleError(err);
    // saved!
    })

Document的创建和删除必须在mongoose connection建立之后才会执行。每一个model会关联一个connection，如果你使用`mongoose.model()`，创建的model会使用默认的mongoose connection。

    mongoose.connect('localhost', 'gettingstarted');

如果你自定义了一个connection，可以使用该connection的`model()`函数。

    var connection = mongoose.createConnection('mongodb://localhost:27017/test');
    var Tank = connection.model('Tank', yourSchema);

### Querying

Mongoose查询文档非常的便利，它提供非常丰富的MongoDB查询语法。可以使用models的`find`,`findById`,`findOne`,或者`where`等静态语法查询。

    Tank.find({ size: 'small' }).where('createdDate').gt(oneYearAgo).exec(callback);

更详细的Query api使用方法请查看[querying][]章节。

[querying]: http://mongoosejs.com/docs/api.html#Query

### Removing

Models提供`remove`静态方法用来移除所有符合提交的documents。
   
    Tank.remove({ size: 'large' }, function (err) {
        if (err) return handleError(err);
        // removed!
    });

### Updating

每个model都有自己的`update`方法用于修改数据库中的documents，并且该方法不会返回被修改的文档。更多详情可查看[API][]文档。

[API]:http://mongoosejs.com/docs/api.html#model_Model.update

### 更多
[API docs][]还提供了很多其他的可用方法，例如[count][], [mapReduce][], [aggregate][], and [more][].

## 下一章 —— [Documents][]

[Documents]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Documents.md
[API docs]:http://mongoosejs.com/docs/api.html#model_Model
[count]:http://mongoosejs.com/docs/api.html#model_Model.count
[mapReduce]:http://mongoosejs.com/docs/api.html#model_Model.mapReduce
[aggregate]:http://mongoosejs.com/docs/api.html#model_Model.aggregate
[more]:http://mongoosejs.com/docs/api.html#model_Model.findOneAndRemove








