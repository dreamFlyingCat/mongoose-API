### Connections

你可以使用` mongoose.connect()`方法连接mongoDB数据库。

    mongoose.connect('mongodb://localhost/myapp');

这是连接运行在本地的Mongodb服务上（默认使用27017端口）的myapp数据库所需要的最少的参数，如果连接失败尝试修改localhost为127.0.0.1再试试，可能是因为改变了localhost的值导致了连接失败。

你可以在uri中指定更多的参数：

    mongoose.connect('mongodb://username:password@host:port/database?options...');

[点击查看][]更多入参详情。

### 缓冲操作

你可以在mongoose连接到mongoDB之前就使用model操作数据库：

    nongoose.connect('mongodb://localhost/myapp');
    var MyModel = mongoose.model('Test', new Schema({ name: String }));
    // Works
    MyModel.findOne(function(error, result) { /* ... */ });

因为mongoose内部调用了缓冲模型函数，这种缓冲很方便，但也是常见的混乱来源。如果你在连接mongoDB之前调用models，mongoose默认不会抛出任何的错误。

    var MyModel = mongoose.model('Test', new Schema({ name: String }));
    // Will just hang until mongoose successfully connects
    MyModel.findOne(function(error, result) { /* ... */ });

    setTimeout(function() {
         mongoose.connect('mongodb://localhost/myapp');
    }, 60000);

可以在你的[schema][]中传入`bufferCommands`选项禁用缓冲。如果你开启了缓冲而且你的连接挂掉了，可以尝试关闭缓冲然后查看你是否正确的建立了连接。也可以全局关闭缓冲：

    mongoose.set('bufferCommands', false);

#### 选项(Options)

Mongoose的`connect`方法第二个参数为`options`对象，该对象会被传递给底层驱动程序，选项中的设置优先于连接字符串中传递的选项。

    mongoose.connect(uri, options);

`connect()`的参数——options的完整的选项列表可以点击[mongoDB node驱动文档][]查看。除了下面的例外情况，mongoose会将options中的选项原封不动的传递给驱动。

* bufferCommands – 是否开启mongoose的buffering, 该属性并不会传递给MongoDB底层驱动。
* user/pass – 用于验证的用户名和密码。这两个选项为mongoose专用的，对应mongoDB驱动的auth.user和auth.password属性。
* autoIndex – mongoose默认在建立连接时为schema中定义index的字段自动创建索引，该特性虽然有利于开发，但是对性能有很大的影响，不利于大型项目生产部署。设置autoIndex为false，mongoose会禁用自动创建索引功能。
* dbName – 在连接字符串中指明连接和重写的数据库名称。如果你使用`mongodb+srv`语法连接[ MongoDB Atlas][]，需要使用`dbname`选项指明连接的数据库，因为在连接字符串中无法指明。

下面的选项对于mongoose的调试（tuning mongoose不知道如何翻译，望高人指点）非常重要：

 * `autoReconnect` - 连接断开时候底层的MongoDB驱动会自动尝试重新连接，除非你是想要管理自己的连接池的非常高级的用户，否则请勿将此选项设置为`false`。
 * `reconnectTries` - 如果到单个服务器或mongos代理（而不是副本集）的连接断开，MongoDB驱动程序将每隔reconnectInterval时间尝试重连，直到reconnectTries后放弃尝试，然后mongoose连接将触发reconnectFailed事件。 此选项对副本集连接不起任何作用。
 * `reconnectInterval` - 详情查看reconnectTries。
 * `promiseLibrary` - 设置底层驱动的[promise library][]
poolSize - MongoDB驱动程序为此连接保持打开的最大套接字数量。poolSize默认为5.请记住，从MongoDB 3.4开始，MongoDB一次只允许每个套接字执行一个操作，因此如果你增加操作，可能会导致一些慢的操作阻塞速度更快的查询。
 * `bufferMaxEntries` - 当驱动程序断开连接时，MongoDB驱动程序有自己的缓冲机制。如果你希望数据库操作在未连接时立即失败，请将此选项设置为0，并设置schema的`bufferCommands`为`false`，而不是等待重新连接。

#### 例子

    const options = {
        useMongoClient: true,
        autoIndex: false, // Don't build indexes
        reconnectTries: Number.MAX_VALUE, // Never stop trying to reconnect
        reconnectInterval: 500, // Reconnect every 500ms
        poolSize: 10, // Maintain up to 10 socket connections
        // If not connected, return errors immediately rather than waiting for reconnect
        bufferMaxEntries: 0
    };
    mongoose.connect(uri, options);

### 回调函数

`connect()`函数接受一个回调函数作为参数，返回一个[promise][]。

    mongoose.connect(uri, options, function(error) {
    // Check error in initial connection. There is no 2nd param to the callback.
    });

    // Or using promises
    mongoose.connect(uri, options).then(
        () => { /** ready to use. The `mongoose.connect()` promise resolves to undefined. */ },
        err => { /** handle initial connection error */ }
    );

### 连接字符串选项

您还可以在连接字符串中指明驱动选项用于URI查询的参数。这仅适用于传递给MongoDB驱动程序的选项。您不能在查询字符串中设置特定于Mongoose的选项，如bufferCommands。

    mongoose.connect('mongodb://localhost:27017/test?connectTimeoutMS=1000&bufferCommands=false');
    // The above is equivalent to:
    mongoose.connect('mongodb://localhost:27017/test', {
    connectTimeoutMS: 1000
    // Note that mongoose will **not** pull `bufferCommands` from the query string
    });

将选项放入查询字符串的缺点是查询字符串选项难以阅读。 优点是你只需要一个配置选项，即URI，而不需要独立的配置socketTimeoutMS，connectTimeoutMS等多个选项。最佳实践是在连接字符串中放置开发和生产之间可能不同的选项，如replicaSet或ssl， 应该保持不变的选项，如connectTimeoutMS或poolSize放在options的对象中。

MongoDB文档有支持的连接字符串选项的[完整列表][]。

###keepAlive注意事项

对于长期运行的应用程序需要谨慎设置`keepAlive`的值（毫秒为单位）。如果不设置该参数，你可能隔三差五的看到`connection closed`的错误提示而且不知道原因。如果是这样，阅读完[此文][]后，您可能会决定启用keepAlive：

    mongoose.connect(uri, { keepAlive: 120 });

### 副本集连接

要连接到副本集，请传入以逗号分隔的主机列表而不是单个主机。

    mongoose.connect('mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]' [, options]);

### 支持多个multi-mongos

在集群中您还可以连接到多个mongos实例以实现高性能。对于mongoose 5.x，连接多个mongos您不需要任何特殊的选项。

    // Connect to 2 mongos servers
    mongoose.connect('mongodb://mongosA:27501,mongosB:27501', cb);

### 多连接

目前为止我们学习了使用mongoose默认连接连接到mongoDB。有时我们需要打开多个到mongo的连接，每个连接的读写设置不同，也可能连接到不同的数据库。这种情况下，可以使用`mongoose.createConnection()`方法，它接受上面提到的所有选项，并给你返回一个新的连接。

    const conn = mongoose.createConnection('mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]', options);

这个连接之后用于新建和检索模型。模型始终限定到其中的一个连接。

当您调用`mongoose.connect()`时，Mongoose会创建一个默认连接。您可以使用mongoose.connection访问默认连接。

### 连接池

每个连接无论是使用`mongoose.connect`还是`mongoose.createConnection`创建的，都需要内部可配置的连接池支持，默认最大为5。可以使用连接选项调整池大小：

    // With object options
    mongoose.createConnection(uri, { poolSize: 4 });

    const uri = 'mongodb://localhost/test?poolSize=4';
    mongoose.createConnection(uri);

### 5.x版本改变的选项

如果从4.x升级到5.x并且没有在4.x中使用`useMongoClient`选项，则可能会看到以下弃用警告：

    the server/replset/mongos options are deprecated, all their options are supported at the top level of the options object

 在较早版本的MongoDB驱动程序中，您必须为服务器连接、副本集连接和mongos连接指定不同的选项： 

    mongoose.connect(myUri, {
    server: {
        socketOptions: {
        socketTimeoutMS: 0,
        keepAlive: true
        },
        reconnectTries: 30
    },
    replset: {
        socketOptions: {
        socketTimeoutMS: 0,
        keepAlive: true
        },
        reconnectTries: 30
    },
    mongos: {
        socketOptions: {
        socketTimeoutMS: 0,
        keepAlive: true
        },
        reconnectTries: 30
    }
    });

在mongoose v5.x中，您可以在顶层声明这些选项，而无需额外的嵌套。 以下是[所有支持选项的列表][]。

    // Equivalent to the above code
    mongoose.connect(myUri, {
        socketTimeoutMS: 0,
        keepAlive: true,
        reconnectTries: 30
    });


## 下一章 —— [Models][]

[所有支持选项的列表]:http://mongodb.github.io/node-mongodb-native/2.2/api/MongoClient.html
[此文]:http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html
[完整列表]:https://docs.mongodb.com/manual/reference/connection-string/
[promise]:http://mongoosejs.com/docs/promises.html
[promise library]:http://mongodb.github.io/node-mongodb-native/2.1/api/MongoClient.html
[ MongoDB Atlas]:https://www.mongodb.com/cloud/atlas
[mongoDB node驱动文档]:http://mongodb.github.io/node-mongodb-native/2.2/api/MongoClient.html#connect
[schema]:http://mongoosejs.com/docs/guide.html#bufferCommands
[Models]:https://github.com/dreamFlyingCat/mongoose-API/blob/master/docs/Schemas/Models.md
[点击查看]:http://docs.mongodb.org/manual/reference/connection-string/
