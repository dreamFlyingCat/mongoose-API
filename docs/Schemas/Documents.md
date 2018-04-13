> 版权声明：本文为博主辛苦翻译，未经博主允许不得转载。

## Documents

Mongoose [documents][]和数据库中存储的document一一对应。每一个document都是它的Model的实例。

[documents]:http://mongoosejs.com/docs/api.html#document-js

### 检索

Mongoose提供了丰富的方法，从MongoDB中检索document，详情请查看[querying][]章节。

[quering]:http://mongoosejs.com/docs/queries.html

### 更新

有很多的方法可以更新documents，我们首先了解传统的方法`findById`：
    
    Tank.findById(id, function (err, tank) {
        if (err) return handleError(err);

        tank.size = 'large';
        tank.save(function (err, updatedTank) {
            if (err) return handleError(err);
            res.send(updatedTank);
        });
    });

你也可以使用`.set()`方法修改document。下例中，将`tank.size='large';`，修改为`tank.set({ size: 'large' })`；

    Tank.findById(id, function (err, tank) {
        if (err) return handleError(err);

        tank.set({ size: 'large' });
        tank.save(function (err, updatedTank) {
            if (err) return handleError(err);
            res.send(updatedTank);
        });
    });

上面的方法首先从mongo中检索文档，然后发出修改命令（调用`save`触发）。如果我们的应用程序不需要返回document，只想直接修改数据库中的document，[Model#update][]更适合我们：

[Model#update]:http://mongoosejs.com/docs/api.html#model_Model.update

    Tank.update({ _id: id }, { $set: { size: 'large' }}, callback);

如果我们的应用程序想要返回document，我们最好使用下面的方法会：

    Tank.findByIdAndUpdate(id, { $set: { size: 'large' }}, { new: true }, function (err, tank) {
        if (err) return handleError(err);
        res.send(tank);
    });


`findAndUpdate/Remove`静态方法最多只修改一个document，可以只调用一次就完成数据库中数据的修改，[findAndModify][]`主体有多种不同的使用方法，详情查看[API][]文档。

[findAndModify]:https://docs.mongodb.com/manual/reference/command/findAndModify/
[API]:http://mongoosejs.com/docs/api.html

请注意`findAndUpdate/Remove`修改documents之前并不会执行任何的hooks或者validation。你可以在该函数的option参数中设置`runValidators`属性为`true`开启本次更新的documents子集的验证。然而，如果你需要hooks和全部document validation，首先需要查询document，然后再`save()`它们。

### 验证

关于Documents保存之前的验证，详情请查看[API][]文档或者[validation][]章节。

[validation]:http://mongoosejs.com/docs/validation.html

### 重写

你可以通过`.set()`方法重写document。该方法修改数据库中保存的文档非常的方便。


    Tank.findById(id, function (err, tank) {
        if (err) return handleError(err);
        // Now `otherTank` is a copy of `tank`
        otherTank.set(tank);
    });

## 下一章 —— [Subdocuments][]

[Subdocuments]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Subdocuments.md