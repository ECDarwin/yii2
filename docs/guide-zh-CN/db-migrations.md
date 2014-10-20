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

That is, we use the timestamp part of a migration name to specify the version
that we want to migrate the database to. If there are multiple migrations between
the last applied migration and the specified migration, all these migrations
will be applied. If the specified migration has been applied before, then all
migrations applied after it will be reverted (to be described in the next section).


Reverting Migrations
--------------------

To revert the last one or several applied migrations, we can use the following
command:

```
yii migrate/down [step]
```

where the optional `step` parameter specifies how many migrations to be reverted
back. It defaults to 1, meaning reverting back the last applied migration.

As we described before, not all migrations can be reverted. Trying to revert
such migrations will throw an exception and stop the whole reverting process.


Redoing Migrations
------------------

Redoing migrations means first reverting and then applying the specified migrations.
This can be done with the following command:

```
yii migrate/redo [step]
```

where the optional `step` parameter specifies how many migrations to be redone.
It defaults to 1, meaning redoing the last migration.


Showing Migration Information
-----------------------------

Besides applying and reverting migrations, the migration tool can also display
the migration history and the new migrations to be applied.

```
yii migrate/history [limit]
yii migrate/new [limit]
```

where the optional parameter `limit` specifies the number of migrations to be
displayed. If `limit` is not specified, all available migrations will be displayed.

The first command shows the migrations that have been applied, while the second
command shows the migrations that have not been applied.


Modifying Migration History
---------------------------

Sometimes, we may want to modify the migration history to a specific migration
version without actually applying or reverting the relevant migrations. This
often happens when developing a new migration. We can use the following command
to achieve this goal.

```
yii migrate/mark 101129_185401
```

This command is very similar to `yii migrate/to` command, except that it only
modifies the migration history table to the specified version without applying
or reverting the migrations.


Customizing Migration Command
-----------------------------

There are several ways to customize the migration command.

### Use Command Line Options

The migration command comes with few options that can be specified in command
line:

* `interactive`: boolean, specifies whether to perform migrations in an
  interactive mode. Defaults to true, meaning the user will be prompted when
  performing a specific migration. You may set this to false should the
  migrations be done in a background process.

* `migrationPath`: string, specifies the directory storing all migration class
  files. This must be specified in terms of a path alias, and the corresponding
  directory must exist. If not specified, it will use the `migrations`
  sub-directory under the application base path.

* `migrationTable`: string, specifies the name of the database table for storing
  migration history information. It defaults to `migration`. The table
  structure is `version varchar(255) primary key, apply_time integer`.

* `db`: string, specifies the ID of the database [application component](structure-application-components.md).
  Defaults to 'db'.

* `templateFile`: string, specifies the path of the file to be served as the code
  template for generating the migration classes. This must be specified in terms
  of a path alias (e.g. `application.migrations.template`). If not set, an
  internal template will be used. Inside the template, the token `{ClassName}`
  will be replaced with the actual migration class name.

To specify these options, execute the migrate command using the following format

```
yii migrate/up --option1=value1 --option2=value2 ...
```

For example, if we want to migrate for a `forum` module whose migration files
are located within the module's `migrations` directory, we can use the following
command:

```
yii migrate/up --migrationPath=@app/modules/forum/migrations
```


### Configure Command Globally

While command line options allow us to configure the migration command
on-the-fly, sometimes we may want to configure the command once for all.
For example, we may want to use a different table to store the migration history,
or we may want to use a customized migration template. We can do so by modifying
the console application's configuration file like the following,

```php
'controllerMap' => [
    'migrate' => [
        'class' => 'yii\console\controllers\MigrateController',
        'migrationTable' => 'my_custom_migrate_table',
    ],
]
```

Now if we run the `migrate` command, the above configurations will take effect
without requiring us to enter the command line options every time. Other command options
can be also configured this way.


### Migrating with Multiple Databases

By default, migrations will be applied to the database specified by the `db` [application component](structure-application-components.md).
You may change it by specifying the `--db` option, for example,

```
yii migrate --db=db2
```

The above command will apply *all* migrations found in the default migration path to the `db2` database.

If your application works with multiple databases, it is possible that some migrations should be applied
to one database while some others should be applied to another database. In this case, it is recommended that
you create a base migration class for each different database and override the [[yii\db\Migration::init()]]
method like the following,

```php
public function init()
{
    $this->db = 'db2';
    parent::init();
}
```

To create a migration that should be applied to a particular database, simply extend from the corresponding
base migration class. Now if you run the `yii migrate` command, each migration will be applied to its corresponding database.

> Info: Because each migration uses hardcoded DB connection, the `--db` option of the `migrate` command will
  have no effect. Also note that the migration history will be stored in the default `db` database.

If you want to support changing DB connection via the `--db` option, you may take the following alternative
approach to work with multiple databases.

For each database, create a migration path and save all corresponding migration classes there. To apply migrations,
run the command as follows,

```
yii migrate --migrationPath=@app/migrations/db1 --db=db1
yii migrate --migrationPath=@app/migrations/db2 --db=db2
...
```

> Info: The above approach stores the migration history in different databases specified via the `--db` option.
