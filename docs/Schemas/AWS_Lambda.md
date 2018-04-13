## 在WS Lambda中使用Mongoose

AWS Lambda是一种流行的任意功能的服务，无需管理单个服务器。 在您的[AWS Lambda][]函数中使用Mongoose很容易。 下面是一个连接到MongoDB实例并找到单个文档的示例函数：

    // Currently, Lambda only supports node v6, so no native async/await.
    const co = require('co');
    const mongoose = require('mongoose');

    let conn = null;

    const uri = 'YOUR CONNECTION STRING HERE';

    exports.handler = function(event, context, callback) {
    // Make sure to add this so you can re-use `conn` between function calls.
    // See https://www.mongodb.com/blog/post/serverless-development-with-nodejs-aws-lambda-mongodb-atlas
    context.callbackWaitsForEmptyEventLoop = false;

    run().
        then(res => {
        callback(null, res);
        console.log('done');
        }).
        catch(error => callback(error));
    };

    function run() {
        return co(function*() {
            // Because `conn` is in the global scope, Lambda may retain it between
            // function calls thanks to `callbackWaitsForEmptyEventLoop`.
            // This means your Lambda function doesn't have to go through the
            // potentially expensive process of connecting to MongoDB every time.
            if (conn == null) {
            conn = yield mongoose.createConnection(uri);
            conn.model('Test', new mongoose.Schema({ name: String }));
            }

            const M = conn.model('Test');

            const doc = yield M.findOne();
            console.log(doc);

            return doc;
        });
    }

要将此函数导入Lambda，请转至[AWS Lambda控制台][]并单击“创建函数”。

![Atl AWS Lambda控制台](/static/FlAD0vT.png)

用下面的设置创建一个名为“mongoose-test”的函数：

![Atl AWS Lambda控制台](/static/mEtS2Ij.png)

将源代码复制到名为`lambda.js`的文件中。 然后运行`npm install mongoose co`。 最后，运行`zip -r mongoose-test.zip node_modules / lambda.js`创建一个zip文件，您可以使用"Function code"下的"Function code"选项上传到Lambda。 确保您还将“Handler”输入更改为`lambda.handler`以匹配lambda.js文件的处理函数。

![Atl AWS Lambda控制台](/static/IO9X570.png)

接下来，点击"Save"按钮，然后点击"Test"按钮。 "Test"按钮将要求您创建一个新的测试事件，只需创建一个测试事件，因为您的输入对此示例无关紧要。 然后，再次点击"Test"来实际运行你的功能：

![Atl AWS Lambda控制台](/static/2UKtWYq.png)

如果您在AWS Lambda中使用Mongoose来避免管理服务器的开销，我们推荐使用MongoDB的托管服务[MongoDB Atlas][]，这样您就不必管理自己的数据库服务。

[AWS Lambda ]:https://aws.amazon.com/cn/lambda/
[AWS Lambda控制台]:https://console.aws.amazon.com/lambda/home
[MongoDB Atlas]:https://cdn.augur.io/forward.html?url=https://mbsy.co/universal/redirect/aHR0cHM6Ly93d3cubW9uZ29kYi5jb20vY2xvdWQvYXRsYXM_bWJzeV9zb3VyY2U9ZmQxOGY4MWQtYmY2Yi00NTJlLWJjMzAtMjgyZGQ2MTIwODQyJmNhbXBhaWduaWQ9MzI0MzEmbWJzeT1sbE5RUg==