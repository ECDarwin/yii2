Models
======
模型是MVC架构的一部分。它是代表了业务数据、规则和逻辑的对象.

你可以通过扩展 [[yii\base\Model]] 或者它的子类来创建。基类 [[yii\base\Model]] 支持非常多有用的功能：

* [属性](#attributes): 代表业务数据并且能够像访问普通对象属性进行访问。
* [属性标签](#attribute-labels): 为属性定义展示的标签
* [块赋值](#massive-assignment): 支持在单个步骤填充多个属性
* [校验规则](#validation-rules): 确保输入数据基于声明的验证规则
* [数据输出](#data-exporting): 允许模型数据按照定制的数组格式被输出

`Model` 类同样也是高级模型的基类, 例如[活动记录](db-active-record.md).
关于更多有关高级模型的细节请参考相关文档.

> 信息；你的模型类不需要强制基于 [[yii\base\Model]]. 然而，因为有很多 Yii 组件支持 [[yii\base\Model]], 它通常是更可取的模型基类.

## 属性 <a name="attributes"></a>

模型依据 *attributes* 代表了业务数据. 每个属性就像模型一个可获取的公有属性。方法 [[yii\base\Model::attributes)()]] 指出了模型类所具有的属性。

You can access an attribute like accessing a normal object property:

```php
$model = new \app\models\ContactForm;

// "name" is an attribute of ContactForm
$model->name = 'example';
echo $model->name;
```

You can also access attributes like accessing array elements, thanks to the support for
[ArrayAccess](http://php.net/manual/en/class.arrayaccess.php) and [ArrayIterator](http://php.net/manual/en/class.arrayiterator.php)
by [[yii\base\Model]]:

```php
$model = new \app\models\ContactForm;

// accessing attributes like array elements
$model['name'] = 'example';
echo $model['name'];

// iterate attributes
foreach ($model as $name => $value) {
    echo "$name: $value\n";
}
```


### 定义属性 <a name="defining-attributes"></a>

默认情况下，如果你的模型类直接扩展自 [[yii\base\Model]], 它所有的 *non-static public* 成员变量都是属性。例如， 下面的 `ContactForm` 拥有四个属性：`name`, `email`, `subject`, 和 `body`. `ContactForm` 模型被用来代表页面form的输入数据。

```php
namespace app\models;

use yii\base\Model;

class ContactForm extends Model
{
    public $name;
    public $email;
    public $subject;
    public $body;
}
```

你可以以重写 [[yii\base\Model::attributes()]] 的方式来重新定义属性。这个方法需要返回模型所有属性的名称。例如, [[yii\db\ActiveRecord]] 通过返回关联的数据库表的列名来代表模型的属性。注意，你必须重写一些魔术方法如 `__get()`, `__set()` 以保证属性可以被访问到。

### 属性标签 <a name="attribute-labels"></a>

当为属性展示值或者获取输入时，常常需要展示一些标签来关联属性。例如，已知 `firstName` 属性, 你想在用户界面或者错误信息展示更加友好的展示为 `Frist Name`.

你可以通过调用 [[yii\base\Model::getAttributeLabel()]] 来获取属性的标签。例如：

```php
$model = new \app\models\ContactForm;

// displays "Name"
echo $model->getAttributeLabel('name');
```

默认情况，属性标签自动产生自属性名。生成通过 [[yii\base\Model::generateAttributeLabel()]] 方法来实现。它将把多个单词以驼峰变量命名法来返回，例如, `username` 变成 `Username`, `firstName` 变成 `First Name`.

如果你不想使用自动生成的标签，你可以重写  [[yii\base\Model::attributeLabels()]] 方法来重新定义属性标签。例如：

```php
namespace app\models;

use yii\base\Model;

class ContactForm extends Model
{
    public $name;
    public $email;
    public $subject;
    public $body;

    public function attributeLabels()
    {
        return [
            'name' => 'Your name',
            'email' => 'Your email address',
            'subject' => 'Subject',
            'body' => 'Content',
        ];
    }
}
```
当应用支持多语言，你可以在 [[yii\base\Model::attributeLabels()|attributeLabels()]] 方法中来实现反意思：

```php
public function attributeLabels()
{
    return [
        'name' => \Yii::t('app', 'Your name'),
        'email' => \Yii::t('app', 'Your email address'),
        'subject' => \Yii::t('app', 'Subject'),
        'body' => \Yii::t('app', 'Content'),
    ];
}
```

你甚至可以有条件地定义属性标签.例如, 基于模型中使用的 [scenario](#scenario),你可以为相同的属性返回不同的标签。

> 提示，严格来讲，属性标签是 [views](structure-views.md) 的一部分. 但是在模型中定义标签将会更加方便，整洁和可重用代码.

## 场景 <a name="scenarios"></a>

一个模型可以在不同的场景中使用。例如，`User` 模型可以在收集用户登录信息时使用，也可以用在用户注册。在不同的场景中，模型需要使用不同的业务规则和逻辑。例如，`email` 属性需要在用户注册时必填，但在登录时则不需要这么做。

模型使用 [[yii\base\Model::scenario]] 来记录正在使用的场景。默认情况下，模型支持一个名为 `default` 的场景。接下来的代码展示了两种设置场景的方方。

```php
// scenario is set as a property
$model = new User;
$model->scenario = 'login';

// scenario is set through configuration
$model = new User(['scenario' => 'login']);
```
默认情况下，场景是不是被一个模型所支持是由在模型中声明的 [validation rules](#validation-rules) 来决定的。然而，你可以通过重写 [[yii\base\Model::scenarios()]] 方法来定制这个行为。

```php
namespace app\models;

use yii\db\ActiveRecord;

class User extends ActiveRecord
{
    public function scenarios()
    {
        return [
            'login' => ['username', 'password'],
            'register' => ['username', 'email', 'password'],
        ];
    }
}
```

> 提示: 在以上的和接下来的列子中, 模型扩展自 [[yii\db\ActiveRecord]],  这是因为多个场景的使用通常发生在 [Active Record] 类中。
  because the usage of multiple scenarios usually happens to [Active Record](db-active-record.md) classes.

`scenarios()` 方法返回一个数组，键值是场景名称，对应的值为激活的属性。一个激活的属性能被 [大量分配](#massive-assignment)并受 [vallidaton](#validation-rules) 的支配。在上面的例子中， `username` 和 `password` 属性是激活的在 `login` 场景下。当在 `register` 场景下， 除了 `username` 和 `password` 之外，`email` 也是激活的。

默认的 `scenarios()` 的实现将会返回在验证规则声明方法中 [[yii\base\Model::rules()]] 找到的所有的场景。当重写 `scenarios()` 时,如果你想在默认的中引入新的场景，你可以像下面这样：

```php
namespace app\models;

use yii\db\ActiveRecord;

class User extends ActiveRecord
{
    public function scenarios()
    {
        $scenarios = parent::scenarios();
        $scenarios['login'] = ['username', 'password'];
        $scenarios['register'] = ['username', 'email', 'password'];
        return $scenarios;
    }
}
```
场景功能主要被 [validation](#validation-rules) 和 [massive attribute assignment](#massive-assignment) 所使用。
你可以以其他目的来使用。例如，你可以基友当前的场景来声明不同的 [attribute labels](#attribute-labels)。


## 验证规则 <a name="validation-rules"></a>

当一个模型从终端用户接收数据时，它应该被验证来确认是安全的。例如，`ContactForm` 模型，你想让所有的属性都不能为空,并且 `email` 属性包含一个正确的邮件地址。当某些属性的值不符合关联的业务规则时，应该展示适当的错误信息帮助用户修正错误。

你可以调用 [[yii\base\Model::validate()]] 来验证接收的数据。该方法将会使用在 [[yii\base\Model::rules()]] 定义的验证规则来验证相关属性。如果没有错误，它将返回true。否则将错误保存在 [[yii\base\Model::errors]] 中并且返回false。例如：

```php
$model = new \app\models\ContactForm;

// populate model attributes with user inputs
$model->attributes = \Yii::$app->request->post('ContactForm');

if ($model->validate()) {
    // all inputs are valid
} else {
    // validation failed: $errors is an array containing error messages
    $errors = $model->errors;
}
```

声明模型关联的验证规则，重写 [[yii\base\Model::rules()]] 方法，并在方法中返回模型属性需要满足的规则。接下来的例子展示了为 `ContactForm` 模型定义的验证规则:

```php
public function rules()
{
    return [
        // the name, email, subject and body attributes are required
        [['name', 'email', 'subject', 'body'], 'required'],

        // the email attribute should be a valid email address
        ['email', 'email'],
    ];
}
```

一个规则能被验证一个或者多个属性，并且一个属性能够被一个或者多个规则进行验证。请参考 [Validating Input](input-validation.md)章节来了解更多验证规则。

某些时候，你可能想要一个规则在特定的场景所应用。你可以定义一个规则的 `on` 属性：

```php
public function rules()
{
    return [
        // username, email and password are all required in "register" scenario
        [['username', 'email', 'password'], 'required', 'on' => 'register'],

        // username and password are required in "login" scenario
        [['username', 'password'], 'required', 'on' => 'login'],
    ];
}
```

如果你不定义 `on` 属性，规则将会在所有的场景中生效。如果一个规则能在当前的 [[yii\base\Model::scenario|scenario]] 生效则被称为 激活的规则。

一个属性将会被验证当且仅当它是一个在 `scenarios()` 中声明的激活属性 并且在 `rules()` 中有一个或者多个激活的规则。

## 块赋值 <a name="massive-assignment"></a>

块赋值一个方便的方式来通过用户输入来扩展模型的方法。
它通过将输入数据赋给 [[yii\base\Model::$attributes]] 来扩展模型属性。下面的两段代码是等价的，都将终端用户提交的表单数据赋值给 `ContactForm` 模型。很明显使用了块赋值的前者比后者更加清晰和少错：

```php
$model = new \app\models\ContactForm;
$model->attributes = \Yii::$app->request->post('ContactForm');
```

```php
$model = new \app\models\ContactForm;
$data = \Yii::$app->request->post('ContactForm', []);
$model->name = isset($data['name']) ? $data['name'] : null;
$model->email = isset($data['email']) ? $data['email'] : null;
$model->subject = isset($data['subject']) ? $data['subject'] : null;
$model->body = isset($data['body']) ? $data['body'] : null;
```


### 安全属性 <a name="safe-attributes"></a>

块赋值只应用模型当前场景中列出的属性。例如，如果 `User` 模型有以下的场景声明，那么当当前的场景为`login` 时，只有 `username` 和 `password` 属性能够被赋值。

```php
public function scenarios()
{
    return [
        'login' => ['username', 'password'],
        'register' => ['username', 'email', 'password'],
    ];
}
```

> Info: The reason that massive assignment only applies to safe attributes is because you want to
  control which attributes can be modified by end user data. For example, if the `User` model
  has a `permission` attribute which determines the permission assigned to the user, you would
  like this attribute to be modifiable by administrators through a backend interface only.

因为默认的 [[yii\base\Model::scenarios()]] 实现将会返回在 [[yii\base\Model::rules()]] 找到的所有的场景和属性，如果你没有重写这个方法，这意味着在激活的验证规则存在属性都是安全属性。

因为这个原因，一个特殊的名为 `safe` 的验证器被提供，你可以声明一个属性是安全的没有实际的验证它。例如，下面的规则声明 `title` 和 `description` 都是安全属性:

```php
public function rules()
{
    return [
        [['title', 'description'], 'safe'],
    ];
}
```

### 不安全属性 <a name="unsafe-attributes"></a>

正如以上所介绍的，[[yii\base\Model::scenarios()]] 方法有两个目的：定义哪些属性应该被验证，并且定义哪些属性是安全的。
在很少的情况下，你可能想要验证一个属性但是不想标记为安全属性。你可以在属性名前置一个感叹号：

```php
public function scenarios()
{
    return [
        'login' => ['username', 'password', '!secret'],
    ];
}
```
当模型在 `login` 场景，三个属性全被验证。然而，只有 `username` 和 `password` 属性可以被块赋值。将一个输入值赋值给 `secret` 属性，你可以像下面这样做：

```php
$model->secret = $secret;
```


## 数据导出 <a name="data-exporting"></a>

模型经常会议不同的格式进行导出。例如，你想将模型集合转化为JSON或者Excel。导出过程可以分为两步。
第一步，模型被转化为数组。第二部，数组转化为目标格式。你只需关注在第一步，第二部可以被通用的数据格式所获取，比如 [[yii\web\JsonResponseFormatter]].

最简单的方法将模型转化为数组是使用 [[yii\base\Model::$attributes]].例如：

```php
$post = \app\models\Post::findOne(100);
$array = $post->attributes;
```

默认的， [[yii\base\Model::$attributes]] 返回在 [[yii\base\Model::attributes()]] 声明的所有的属性。

更加灵活有力的方法是使用 [[yii\base\Model::toArray()]] 方法来转化为数组。它默认的行为和 [[yii\base\Model::$attributes]] 一样。然而，它允许你选择那些数据(字段)，被放到结果数组中并且他们怎样被格式化。
实际上，在RESTfull开发中它是默认的方法 [Response Formatting](rest-response-formatting.md).


### 字段 <a name="fields"></a>

A field is simply a named element in the array that is obtained by calling the [[yii\base\Model::toArray()]] method
of a model.

默认情况下，字段名称和属性名是一样的。然而，你可以通过重写 [[yii\base\Model::fields()|fields()]] 和/或者 [[yii\base\Model::extraFields|extraFields()]] 方法来改变这个默认的行为。两个方法都应该返回一个字段定义的列表。
The fields defined by `fields()` are default fields, meaning that
`toArray()` will return these fields by default. The `extraFields()` method defines additionally available fields
which can also be returned by `toArray()` as long as you specify them via the `$expand` parameter. For example,
the following code will return all fields defined in `fields()` and the `prettyName` and `fullAddress` fields
if they are defined in `extraFields()`.

```php
$array = $model->toArray([], ['prettyName', 'fullAddress']);
```

You can override `fields()` to add, remove, rename or redefine fields. The return value of `fields()`
should be an array. The array keys are the field names, and the array values are the corresponding
field definitions which can be either property/attribute names or anonymous functions returning the
corresponding field values. In the special case when a field name is the same as its defining attribute
name, you can omit the array key. For example,

```php
// explicitly list every field, best used when you want to make sure the changes
// in your DB table or model attributes do not cause your field changes (to keep API backward compatibility).
public function fields()
{
    return [
        // field name is the same as the attribute name
        'id',

        // field name is "email", the corresponding attribute name is "email_address"
        'email' => 'email_address',

        // field name is "name", its value is defined by a PHP callback
        'name' => function () {
            return $this->first_name . ' ' . $this->last_name;
        },
    ];
}

// filter out some fields, best used when you want to inherit the parent implementation
// and blacklist some sensitive fields.
public function fields()
{
    $fields = parent::fields();

    // remove fields that contain sensitive information
    unset($fields['auth_key'], $fields['password_hash'], $fields['password_reset_token']);

    return $fields;
}
```

> Warning: Because by default all attributes of a model will be included in the exported array, you should
> examine your data to make sure they do not contain sensitive information. If there is such information,
> you should override `fields()` to filter them out. In the above example, we choose
> to filter out `auth_key`, `password_hash` and `password_reset_token`.


## Best Practices <a name="best-practices"></a>

Models are the central places to represent business data, rules and logic. They often need to be reused
in different places. In a well-designed application, models are usually much fatter than
[controllers](structure-controllers.md).

In summary, models

* may contain attributes to represent business data;
* may contain validation rules to ensure the data validity and integrity;
* may contain methods implementing business logic;
* should NOT directly access request, session, or any other environmental data. These data should be injected
  by [controllers](structure-controllers.md) into models;
* should avoid embedding HTML or other presentational code - this is better done in [views](structure-views.md);
* avoid having too many [scenarios](#scenarios) in a single model.

You may usually consider the last recommendation above when you are developing large complex systems.
In these systems, models could be very fat because they are used in many places and may thus contain many sets
of rules and business logic. This often ends up in a nightmare in maintaining the model code
because a single touch of the code could affect several different places. To make the mode code more maintainable,
you may take the following strategy:

* Define a set of base model classes that are shared by different [applications](structure-applications.md) or
  [modules](structure-modules.md). These model classes should contain minimal sets of rules and logic that
  are common among all their usages.
* In each [application](structure-applications.md) or [module](structure-modules.md) that uses a model,
  define a concrete model class by extending from the corresponding base model class. The concrete model classes
  should contain rules and logic that are specific for that application or module.

For example, in the [Advanced Application Template](tutorial-advanced-app.md), you may define a base model
class `common\models\Post`. Then for the front end application, you define and use a concrete model class
`frontend\models\Post` which extends from `common\models\Post`. And similarly for the back end application,
you define `backend\models\Post`. With this strategy, you will be sure that the code in `frontend\models\Post`
is only specific to the front end application, and if you make any change to it, you do not need to worry if
the change may break the back end application.
