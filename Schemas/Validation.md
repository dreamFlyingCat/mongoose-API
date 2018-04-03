## Validation

在我们使用确定的validation语法前，请先记住下面的规则：

   * Validation是在SchemaType中定义的额；
   * Validation是中间件的内部组件，schema默认使用`pre('save')`钩子函数注册validation；
   * 你可以使用`doc.validate(callback)`或者`doc.validateSync()`进行手动的validation。
   * 除了`required`验证器，validate不会在undefined值上运行；
   * validate是异步递归执行的，如果顶层document调用[Model#save][]，sub-document的验证也会执行。一旦发生错误，[Model#save][]的回调函数会收它。
   * validation支持自定义。

----- 

            var schema = new Schema({
            name: {
                type: String,
                required: true
            }
            });
            var Cat = db.model('Cat', schema);

            // This cat has no name :(
            var cat = new Cat();
            cat.save(function(error) {
            assert.equal(error.errors['name'].message,
                'Path `name` is required.');

            error = cat.validateSync();
            assert.equal(error.errors['name'].message,
                'Path `name` is required.');
            });
  
### 内置验证器

Mongoose提供了一些内置的验证器。

   * 所有的[SchemaType][]都有[required][]验证器，其使用`checkRequired()`函数判断值是否符合要求。
   * Numbers有专属的[min][]和[max][]验证器；
   * String有专属[enum][]、[match][]、[maxlength][]、[minlength][]验证器。 

上面的每一个验证器链接提供关于如何使用它们和定制错误信息。

       var breakfastSchema = new Schema({
        eggs: {
            type: Number,
            min: [6, 'Too few eggs'],
            max: 12
        },
        bacon: {
            type: Number,
            required: [true, 'Why no bacon?']
        },
        drink: {
            type: String,
            enum: ['Coffee', 'Tea'],
            required: function() {
                return this.bacon > 3;
            }
        }
    });
    var Breakfast = db.model('Breakfast', breakfastSchema);

    var badBreakfast = new Breakfast({
        eggs: 2,
        bacon: 0,
        drink: 'Milk'
    });
    var error = badBreakfast.validateSync();
    assert.equal(error.errors['eggs'].message,
      'Too few eggs');
    assert.ok(!error.errors['bacon']);
    assert.equal(error.errors['drink'].message,
      '`Milk` is not a valid enum value for path `drink`.');

    badBreakfast.bacon = 5;
    badBreakfast.drink = null;

    error = badBreakfast.validateSync();
    assert.equal(error.errors['drink'].message, 'Path `drink` is required.');

    badBreakfast.bacon = null;
    error = badBreakfast.validateSync();
    assert.equal(error.errors['bacon'].message, 'Why no bacon?');

### `unique`选项不是一个验证器

`unique`选项不是一个验证器，它用于快速建立MongoDB的unique indexes。更多信息查看[FAQ][]。

      var uniqueUsernameSchema = new Schema({
        username: {
            type: String,
            unique: true
        }
    });
    var U1 = db.model('U1', uniqueUsernameSchema);
    var U2 = db.model('U2', uniqueUsernameSchema);

    var dup = [{ username: 'Val' }, { username: 'Val' }];
    U1.create(dup, function(error) {
      // Race condition! This may save successfully, depending on whether
      // MongoDB built the index before writing the 2 docs.
    });

    // Need to wait for the index to finish building before saving,
    // otherwise unique constraints may be violated.
    U2.once('index', function(error) {
      assert.ifError(error);
      U2.create(dup, function(error) {
        // Will error, but will *not* be a mongoose validation error, it will be
        // a duplicate key error.
        assert.ok(error);
        assert.ok(!error.errors);
        assert.ok(error.message.indexOf('duplicate key error') !== -1);
      });
    });

    // There's also a promise-based equivalent to the event emitter API.
    // The `init()` function is idempotent and returns a promise that
    // will resolve once indexes are done building;
    U2.init().then(function() {
      U2.create(dup, function(error) {
        // Will error, but will *not* be a mongoose validation error, it will be
        // a duplicate key error.
        assert.ok(error);
        assert.ok(!error.errors);
        assert.ok(error.message.indexOf('duplicate key error') !== -1);
      });
    });

### 自定义验证器

如果内置的验证器满足不了你的需求，你可以使用自定义验证器。

通过传递一个validation函数实现自定义，具体细节查看`SchemaType#validate()`章节[API][]文档

     var userSchema = new Schema({
      phone: {
        type: String,
        validate: {
          validator: function(v) {
            return /\d{3}-\d{3}-\d{4}/.test(v);
          },
          message: '{VALUE} is not a valid phone number!'
        },
        required: [true, 'User phone number required']
      }
    });

    var User = db.model('user', userSchema);
    var user = new User();
    var error;

    user.phone = '555.0123';
    error = user.validateSync();
    assert.equal(error.errors['phone'].message,
      '555.0123 is not a valid phone number!');

    user.phone = '';
    error = user.validateSync();
    assert.equal(error.errors['phone'].message,
      'User phone number required');

    user.phone = '201-555-0123';
    // Validation succeeds! Phone number is defined
    // and fits `DDD-DDD-DDDD`
    error = user.validateSync();
    assert.equal(error, null);

### 自定义异步验证器

我们也可以自定义异步验证器。如果验证器函数返回一个promise实例（像一个async函数）,mongoose会等待promise状态确定。如果你更喜欢callbacks形式，设置`isAsync`选项并且给验证器函数传入callback作为第二个参数。

     var userSchema = new Schema({
      name: {
        type: String,
        // You can also make a validator async by returning a promise. If you
        // return a promise, do **not** specify the `isAsync` option.
        validate: function(v) {
          return new Promise(function(resolve, reject) {
            setTimeout(function() {
              resolve(false);
            }, 5);
          });
        }
      },
      phone: {
        type: String,
        validate: {
          isAsync: true,
          validator: function(v, cb) {
            setTimeout(function() {
              var phoneRegex = /\d{3}-\d{3}-\d{4}/;
              var msg = v + ' is not a valid phone number!';
              // First argument is a boolean, whether validator succeeded
              // 2nd argument is an optional error message override
              cb(phoneRegex.test(v), msg);
            }, 5);
          },
          // Default error message, overridden by 2nd argument to `cb()` above
          message: 'Default error message'
        },
        required: [true, 'User phone number required']
      }
    });

    var User = db.model('User', userSchema);
    var user = new User();
    var error;

    user.phone = '555.0123';
    user.name = 'test';
    user.validate(function(error) {
      assert.ok(error);
      assert.equal(error.errors['phone'].message,
        '555.0123 is not a valid phone number!');
      assert.equal(error.errors['name'].message,
        'Validator failed for path `name` with value `test`');
    });

### 错误验证

如果验证失败会返回错误包含了一个错误对象，值为`ValidatorError`对象。[ValidatorError][]对象有`kind`、`path`、`value`和`message`等属性。验证器也可能有`reason`属性，如果执行验证器时发生了错误，`reason`属性包含抛出的错误。

     var toySchema = new Schema({
      color: String,
      name: String
    });

    var validator = function(value) {
      return /red|white|gold/i.test(value);
    };
    toySchema.path('color').validate(validator,
      'Color `{VALUE}` not valid', 'Invalid color');
    toySchema.path('name').validate(function(v) {
      if (v !== 'Turbo Man') {
        throw new Error('Need to get a Turbo Man for Christmas');
      }
      return true;
    }, 'Name `{VALUE}` is not valid');

    var Toy = db.model('Toy', toySchema);

    var toy = new Toy({ color: 'Green', name: 'Power Ranger' });

    toy.save(function (err) {
      // `err` is a ValidationError object
      // `err.errors.color` is a ValidatorError object
      assert.equal(err.errors.color.message, 'Color `Green` not valid');
      assert.equal(err.errors.color.kind, 'Invalid color');
      assert.equal(err.errors.color.path, 'color');
      assert.equal(err.errors.color.value, 'Green');

      // This is new in mongoose 5. If your validator throws an exception,
      // mongoose will use that message. If your validator returns `false`,
      // mongoose will use the 'Name `Power Ranger` is not valid' message.
      assert.equal(err.errors.name.message,
        'Need to get a Turbo Man for Christmas');
      assert.equal(err.errors.name.value, 'Power Ranger');
      // If your validator threw an error, the `reason` property will contain
      // the original error thrown, including the original stack trace.
      assert.equal(err.errors.name.reason.message,
        'Need to get a Turbo Man for Christmas');

      assert.equal(err.name, 'ValidationError');
    });

### 嵌套对象中使用Required验证器

对值为嵌套对象的属性进行验证是比较麻烦的，因为嵌套对象不是一个完善的paths。

    var personSchema = new Schema({
      name: {
        first: String,
        last: String
      }
    });

    assert.throws(function() {
      // This throws an error, because 'name' isn't a full fledged path
      personSchema.path('name').required(true);
    }, /Cannot.*'required'/);

    // To make a nested object required, use a single nested schema
    var nameSchema = new Schema({
      first: String,
      last: String
    });

    personSchema = new Schema({
      name: {
        type: nameSchema,
        required: true
      }
    });

    var Person = db.model('Person', personSchema);

    var person = new Person();
    var error = person.validateSync();
    assert.ok(error.errors['name']);

### 更新验证

上面实例中，你学习了document的验证。Mongoose也支持执行`update()`或`findOneAndUpdate()`时的更新验证。更新验证默认关闭。设置`runValidators`选项，开启`update()`或者`findOneAndUpdate()`的更新验证。请注意，更新验证之所以默认关闭，因为会有一些警告信息。

    var toySchema = new Schema({
      color: String,
      name: String
    });

    var Toy = db.model('Toys', toySchema);

    Toy.schema.path('color').validate(function (value) {
      return /blue|green|white|red|orange|periwinkle/i.test(value);
    }, 'Invalid color');

    var opts = { runValidators: true };
    Toy.update({}, { color: 'bacon' }, opts, function (err) {
      assert.equal(err.errors.color.message,
        'Invalid color');
    });

### 更新验证和`this`

更新验证器和document validator会有些不同。下面示例中，color path的验证函数中this指向正在被验证的document。然而执行更新验证时，document可能不在服务器内存中更新，所以默认this未定义。

    var toySchema = new Schema({
      color: String,
      name: String
    });

    toySchema.path('color').validate(function(value) {
      // When running in `validate()` or `validateSync()`, the
      // validator can access the document using `this`.
      // Does **not** work with update validators.
      if (this.name.toLowerCase().indexOf('red') !== -1) {
        return value !== 'red';
      }
      return true;
    });

    var Toy = db.model('ActionFigure', toySchema);

    var toy = new Toy({ color: 'red', name: 'Red Power Ranger' });
    var error = toy.validateSync();
    assert.ok(error.errors['color']);

    var update = { color: 'red', name: 'Red Power Ranger' };
    var opts = { runValidators: true };

    Toy.update({}, update, opts, function(error) {
      // The update validator throws an error:
      // "TypeError: Cannot read property 'toLowerCase' of undefined",
      // because `this` is **not** the document being updated when using
      // update validators
      assert.ok(error);
    });
  
### `context`选项

设置context选项，使this值在更新验证时指向隐藏的query。

    var toySchema = new Schema({
      color: String,
      name: String
    });

    toySchema.path('color').validate(function(value) {
      // When running in `validate()` or `validateSync()`, the
      // validator can access the document using `this`.
      // Does **not** work with update validators.
      if (this.name.toLowerCase().indexOf('red') !== -1) {
        return value !== 'red';
      }
      return true;
    });

    var Toy = db.model('ActionFigure', toySchema);

    var toy = new Toy({ color: 'red', name: 'Red Power Ranger' });
    var error = toy.validateSync();
    assert.ok(error.errors['color']);

    var update = { color: 'red', name: 'Red Power Ranger' };
    var opts = { runValidators: true };

    Toy.update({}, update, opts, function(error) {
      // The update validator throws an error:
      // "TypeError: Cannot read property 'toLowerCase' of undefined",
      // because `this` is **not** the document being updated when using
      // update validators
      assert.ok(error);
    });
  


[ValidatorError]:http://mongoosejs.com/docs/api.html#error-validation-js
[API]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-validate
[FAQ]:http://mongoosejs.com/docs/faq.html
[Model#save]:http://mongoosejs.com/docs/api.html#model_Model-save
[SchemaType]:http://mongoosejs.com/docs/schematypes.html
[min]:http://mongoosejs.com/docs/api.html#schema_number_SchemaNumber-min
[max]:http://mongoosejs.com/docs/api.html#schema_number_SchemaNumber-max
[enum]:http://mongoosejs.com/docs/api.html#schema_string_SchemaString-enum
[match]:http://mongoosejs.com/docs/api.html#schema_string_SchemaString-match
[maxlength]:http://mongoosejs.com/docs/api.html#schema_string_SchemaString-maxlength
[minlength]:http://mongoosejs.com/docs/api.html#schema_string_SchemaString-minlength
[required]:http://mongoosejs.com/docs/api.html#schematype_SchemaType-required
