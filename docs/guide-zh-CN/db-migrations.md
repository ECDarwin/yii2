数据库迁移
==================

> 注意: 该章节还在开发中。

像源代码一样，数据库的结构的演变作为一个数据库驱动的应用程序开发和维护。例如，在开发期间，可能会增加一张新表；或者，在应用上线之后，需要新增一个索引。记录数据库结构的改变非常重要（叫做 **迁移**），正如修改源代码跟踪使用版本控制。如果源代码和数据库不同步，就会发生bug，或者整个应用就会被破坏。因此，Yii提供了一个数据库迁移工具能够记录数据库的迁移历史，应用新的迁移，或者恢复已有的。

接下来的步骤展示了在团队开发中如何进行数据迁移：

1. 提姆创建一个新的迁移（例如，创建一张新表，改变字段的定义等等。）。
2. 提姆将新的迁移提交到源代码控制系统（例如，Git，Mercurial）。
3. 逗哥更新了版本库并接收到了新的迁移。
4. 逗哥在本地开发数据库中应用了迁移，从而同步他的数据库来反映提姆的改动。

Yii 通过 `yii migrate` 命令行工具来支持数据迁移。此工具支持：

* 新建迁移
* 应用，恢复和重做迁移
* 显示迁移历史和新的迁移

创建迁移
-------------------

执行以下的命令来创建新的迁移：

```
yii migrate/create <name>
```
必须要的 `name` 参数指定迁移简明的介绍。例如，如果迁移创建了一张名为 *news* 的新表，你应该使用命令：

```
yii migrate/create create_news_table
```

很快你就会看到，`name` 参数在迁移中被用来定义PHP类名的一部分。因此，它只能包含字母，数组和/或者下划线。

上面的命令会创建一个新的文件名为 `m101129_185401_create_news_table.php`。这个文件将会创建在 `@app/migrations` 目录。首先，迁移文件包含下代码：

```php
class m101129_185401_create_news_table extends \yii\db\Migration
{
    public function up()
    {
    }

    public function down()
    {
        echo "m101129_185401_create_news_table cannot be reverted.\n";
        return false;
    }
}
```

注意类名和文件名一致，并且满足 `m<timestamp>_name` 的规则，其中：

* `<timestamp>` 代表UTC时间戳（`yymmdd_hhmmss`格式）当迁移被创建，
* `<name>` 使用命令行中的 `name` 参数

在类中，`up()` 方法应该包含实现数据迁移的代码。换句话说，`up()` 方法执行改变数据的代码。`down()` 包含恢复由 `up()` 方法产生的改变。

有些时候，`down()` 撤销数据库迁移是不可能的。例如，如果迁移删除表的列或者删除整个表，数据就不能在 `down()` 方法中恢复。在这种情况下，迁移被称为不可逆的，意味着数据库不能被回滚到之前的状态。当迁移不可逆时，就像上面产生的代码，`down()` 方法返回 `false` 来指出迁移不可逆。

例如，展示一个创建新表的迁移。

```php

use yii\db\Schema;

class m101129_185401_create_news_table extends \yii\db\Migration
{
    public function up()
    {
        $this->createTable('news', [
            'id' => 'pk',
            'title' => Schema::TYPE_STRING . ' NOT NULL',
            'content' => Schema::TYPE_TEXT,
        ]);
    }

    public function down()
    {
        $this->dropTable('news');
    }

}
```

基类 [[yii\db\Migration]] 通过 `db` 属性暴露数据库连接。你可以使用它来操作数据和数据库的模式。

在这个例子中使用的字段类型将会被Yii根据DBMS来替换相应的类型。

例如 `pk` 在MySQL中将会被替换为 `int(11) NOT NULL AUTO_iINCREMENT PRIMARY KEY` ，在sqlite中被替换为 `int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY`。参考 [[yii\db\QueryBuilder::getColumnType()]] 文档获取更多详情和可用的类型。你也可以使用在 [[yii\db\Schema]] 中定义的常量类型。

> Note: You can add constraints and other custom table options at the end of the table description by
> specifying them as simple string. For example in the above migration, after `content` attribute definition
> you can write `'CONSTRAINT ...'` or other custom options.


事务处理迁移
------------------------

当进行复杂DB的迁移时，总的来说我们通常想要每个迁移成功或者失败，来确保数据库的一致性和完整性。为了达到这个目的，我们可以利用数据事物。我们可以使用特殊的方法 `safeUp` 和 `safeDown` 来达到这个目的。

```php

use yii\db\Schema;

class m101129_185401_create_news_table extends \yii\db\Migration
{
    public function safeUp()
    {
        $this->createTable('news', [
            'id' => 'pk',
            'title' => Schema::TYPE_STRING . ' NOT NULL',
            'content' => Schema::TYPE_TEXT,
        ]);

        $this->createTable('user', [
            'id' => 'pk',
            'login' => Schema::TYPE_STRING . ' NOT NULL',
            'password' => Schema::TYPE_STRING . ' NOT NULL',
        ]);
    }

    public function safeDown()
    {
        $this->dropTable('news');
        $this->dropTable('user');
    }

}
```

当你的代码使用多一个查询建议使用 `safeUp` 和 `safeDown`。

> 注意：并不是所有的DBMS都支持事物。并且一些数据库查询不能被
> 放到事物中。在这种情况下，你不得不实现 `up()` 和 `down()` 来替代
> 对于MySQL，一些SQL语句可能引起 []()
> Note: Not all DBMS support transactions. And some DB queries cannot be put
> into a transaction. In this case, you will have to implement `up()` and
> `down()`, instead. And for MySQL, some SQL statements may cause [内含提交](http://dev.mysql.com/doc/refman/5.1/en/implicit-commit.html)

应用迁移
-------------------

应用所有的可用的新迁移（例如，使本地数据保持最新），执行以下命令：

```
yii migrate
```

这个命令显示所有新的迁移。如果你确认应用迁移，这将以类名的时间戳值为顺序来运行每个迁移类中的 `up()` 方法。

在应用迁移后，迁移工具在 `migration` 表中进行记录。这使工具分辨哪些迁移被应用哪些没有被应用。如果 `migration` 表不存在，工具会自动在当前 `db` 中创建。

有些时候，我们只想应用一个或者几个迁移。我们可以使用一下命令：

```
yii migrate/up 3
```

这个命令将会应用3个新的迁移。改变3的值允许改变应用的迁移数。

我们也可以迁移数据库到指定的版本：

```
yii migrate/to 101129_185401
```

即，我们使用迁移名的时间戳部分来指定我们想要迁移数据库的版本。如果在最新应用的迁移和指定的迁移中有多个迁移，所有的迁移都会被应用。如果指定的迁移之前应用过，那么所有的迁移会在恢复后被应用（下章节会讲到）。

恢复迁移
--------------------

恢复上一次或者几次迁移，我们可以使用下面的命令：

```
yii migrate/down [step]
```

`step` 参数指定恢复多少个迁移。默认为1，回滚到上一次迁移。

正如我们之前描述的，并不是所有的迁移都可以被回滚。回滚这样的迁移会抛出异常并且停止整个回滚进程。

重新迁移
------------------

重新迁移意思是首先回滚然后应用指定的迁移。这可以用下面的命令：

```
yii migrate/redo [step]
```

可选的`step`参数指定多少迁移被重做。默认为1，重做上一次迁移。

展示迁移信息
-----------------------------

除了应用和回滚迁移之外，迁移工具同样能够展示迁移历史和最新应用的迁移。

```
yii migrate/history [limit]
yii migrate/new [limit]
```

可选的 `limit` 参数指定了展示迁移的个数。如果 `limit` 没被指定，那么所有的迁移信息被展示。

第一个命令展示被应用的迁移，第二个命名展示了没用被应用的迁移。

修改迁移历史
---------------------------

有些时候，我们想修改迁移历史来指定迁移版本但不应用或者回滚相关的迁移。这在开发新的迁移时常常发生。我们可以使用下面的命令来实现这个目的。

```
yii migrate/mark 101129_185401
```

这个命令和 `yii migrate/to` 相似，除了它仅仅修改迁移历史表来指定版本而不会应用或者回滚迁移。


定制迁移命令
-----------------------------

有几个方法来定制迁移命令。

### 使用命令行选项

迁移命令有几个选项能够在命令行中指定：

* `interactive`: boolean, 指定是否在交互的模式下执行迁移。默认为ture，意味着在执行特定迁移时会提示用户。你可以设置为false让迁移在后台运行
* `migrationPath`: string, 指定存贮迁移类文件的目录。必须指定为路劲别名，并且相关的目录必须存在。如果没指定，将会使用应用目录下的 `migrations` 子目录。
* `migrationTable`: string, 指定保存迁移历史信息的数据库表。表的结构为 `version varchar(255) primary key, apply_time integer`。
* `db`: string, 指定数据库的ID[应用组件](structure-application-components.md)。默认为 "db"。
* `templateFile`: string, 指定存储生成的迁移类文件的路径。必须指定为路径别名（例如，`application.migrations.template`）。如果没指定，将会使用内部模板。在模板里面，`{ClassName}` 将会被替换为实际的类名。

以下面的格式的来执行迁移命令来指定选项。

```
yii migrate/up --option1=value1 --option2=value2 ...
```

例如，如果我们想迁移 `forum` 模块下的 `migrations` 目录的迁移文件，我们可以使用下面的命令：

```
yii migrate/up --migrationPath=@app/modules/forum/migrations
```


### 配置全局命令

命令行的选项让我们在迁移的时候手忙脚乱，有些时候我们想一次性配置所有的命令。例如，我们想要使用不同的表来存储迁移历史，或者我们想要使用定制的迁移模板。我们可以像下面这样来修改命令行应用的配置文件，

```php
'controllerMap' => [
    'migrate' => [
        'class' => 'yii\console\controllers\MigrateController',
        'migrationTable' => 'my_custom_migrate_table',
    ],
]
```
现在我们执行 `migrate` 命令，上面的配置将会起作用就不用每次在命令行中输入选项。其他的命令选项也可以像这样配置。

### 迁移多个数据库

默认情况下，迁移应用到 `db` [application componet](structure-application-components.md) 指定的数据库。

```
yii migrate --db=db2
```

上面的命令将会应用所有的的迁移文件到 `db2` 数据库。

如果你的应用使用多个数据库，那么就有可能一些迁移应用到一个数据库，另外一些迁移应用到另外一个数据库。既然这样，建议你为每个通用的数据创建一个迁移基类，然后像下面这样重写 [[yii\db\Migration::init()]] 方法，

```php
public function init()
{
    $this->db = 'db2';
    parent::init();
}
```

创建一个应用到特别数据的迁移，扩展相应的迁移基类即可。现在如果你运行 `yii migrate` ，每个迁移都会应用相应的数据库。

> 提示: 因为每个迁移使用了硬编码的DB连接， `--db` 选项将不会起作用。同样注意迁移历史会存储在默认的 `db` 数据库。

如果你希望支持通过 `--db` 选项改变数据连接，你可以使用下面供选择的方法来使用多个数据库。

每个数据库，创建一个迁移路径，并且保存所有相关的迁移类。指向下面的命令来应用迁移：

```
yii migrate --migrationPath=@app/migrations/db1 --db=db1
yii migrate --migrationPath=@app/migrations/db2 --db=db2
...
```

> 提示：上面的方法将迁移历史存储到通过 `--db` 选项指定的不同的数据库。
