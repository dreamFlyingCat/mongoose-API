> 版权声明：本文为博主辛苦翻译，未经博主允许不得转载。

## 插件

模式是可插入的，也就是说，它们允许应用预先打包的功能来扩展其功能。 这是一个非常强大的功能。

假设我们的数据库中有多个集合，并且想为每个集合添加最后修改的功能。使用插件很容易实现，只需要一次性的创建一个插件并应用到每个schema上即可。

    // lastMod.js
    module.exports = exports = function lastModifiedPlugin (schema, options) {
    schema.add({ lastMod: Date });

    schema.pre('save', function (next) {
        this.lastMod = new Date();
        next();
    });

    if (options && options.index) {
        schema.path('lastMod').index(options.index);
    }
    }

    // game-schema.js
    var lastMod = require('./lastMod');
    var Game = new Schema({ ... });
    Game.plugin(lastMod, { index: true });

    // player-schema.js
    var lastMod = require('./lastMod');
    var Player = new Schema({ ... });
    Player.plugin(lastMod);

我们将最后修改的行为添加到了Game和Player的schemas中并且在`lastMod`路径上声明了索引，几行代码就搞定了。

### 全局插件

如何给所有的schema注册一个插件呢？mongoose有一个`.plugin()`函数可以为每一个schema注册插件，例如：

    var mongoose = require('mongoose');
    mongoose.plugin(require('./lastMod'));

    var gameSchema = new Schema({ ... });
    var playerSchema = new Schema({ ... });
    // `lastModifiedPlugin` gets attached to both schemas
    var Game = mongoose.model('Game', gameSchema);
    var Player = mongoose.model('Player', playerSchema);

### 社区

您不仅可以在自己的项目中重复使用schema的功能，还可以从Mongoose社区中获益。 任何发布到[npm][]并带有mongoose[标签][]的插件都会显示在我们的[搜索结果][]页面上。

## 下一章 —— [AWS_Lambda][]

[AWS_Lambda]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/AWS_Lambda.md

[标签]:https://www.npmjs.com/doc/tag.html
[npm]:https://www.npmjs.com/
[搜索结果]:http://plugins.mongoosejs.io/
