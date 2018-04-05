### 新手入门

首先请确定你已经安装了[mongoDB][]和[Node.js][]。
接下来使用`npm`安装mongoose。

    $ npm install mongoose

假设我们喜欢毛绒绒的猫咪，想要在mongoDB中记录我们遇见过的每一只猫咪。首先我们要在项目中引入mongoose并且连接到运行在本地MongoDB实例上的`test`数据库。

    // getting-started.js
    var mongoose = require('mongoose');
    mongoose.connect('mongodb://localhost/test');       

我们需要知道连接到运行在本地的数据库的结果是成功还是失败：

    var db = mongoose.connection;
    db.on('error', console.error.bind(console, 'connection error:'));
    db.once('open', function() {
    // we're connected!
    });

一旦我们发起连接，回调函数会被调用。简洁起见，我们假定下面的代码都写在回调函数中。

在Mongoose中，所有事物都源于模式。参考下面的代码定义kittens。

    var kittySchema = mongoose.Schema({
    name: String
    });

目前为止，我们创建了一个schema，它只有一个属性——字符串类型的`name`。下一步要将schema编译为一个[Model][]。

    var Kitten = mongoose.model('Kitten', kittySchema);

model是一个class用于构造documents。在这种情况下，每个document都是Kitten的实例，都拥有在schema中声明的属性和行为。让我们创建一个document来代表在路边遇到的一只猫咪。

    var silence = new Kitten({ name: 'Silence' });
    console.log(silence.name); // 'Silence'

猫咪可以喵喵叫，我们学习如何给document添加"speak"功能：

    // NOTE: methods must be added to the schema before compiling it with mongoose.model()
    kittySchema.methods.speak = function () {
        var greeting = this.name
            ? "Meow name is " + this.name
            : "I don't have a name";
        console.log(greeting);
    }

    var Kitten = mongoose.model('Kitten', kittySchema);

将方法函数添加到schema实例的`methods`属性上，方法会被编译到`Model`的原型上从而让每个实例document都可以访问该方法。

    var fluffy = new Kitten({ name: 'fluffy' });
    fluffy.speak(); // "Meow name is fluffy"

这样猫咪就可以说话啦！到现在我们还没有向mongoDB中保存任何的数据，调用document的`save`方法保存文档。如果报错，错误对象会包含在回调函数的第一个参数中。

    fluffy.save(function (err, fluffy) {
        if (err) return console.error(err);
        fluffy.speak();
    });

我们可以通过操作Kitten [Model][]查看曾经遇到过的所有猫咪的文档。

    Kitten.find(function (err, kittens) {
        if (err) return console.error(err);
        console.log(kittens);
    })

我们将mongoDB中所有的猫咪文档全部打印出来了，我们也可以通过名字过滤猫咪，mongoose支持[mongoDB][]丰富的查询语法。

    Kitten.find({ name: /^fluff/ }, callback);

上例中会查询所有名字以'fluff'开头的猫咪，并将结果放在数组中返回给回调函数。

### Congratulations

快速入门到此结束。现在我们成功的创建了一个schema并添加了一个自定义的实例方法，使用Mongoose在MongoDB中保存并查询了kittens的文档。进阶学习前往查看[guide][]或者[API文档][]。

[guide]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Schema.md
[API文档]:http://mongoosejs.com/docs/api.html
[mongoDB]:https://www.mongodb.com/download-center
[Node.js]:https://nodejs.org/en/
[Model]:http://mongoosejs.com/docs/models.html