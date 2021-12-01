## Sqlite3 PHP Class
Full operation requires Sqlite3 built with support for
SQLITE_ENABLE_UPDATE_DELETE_LIMIT Tested on version 3.11.0

## Installation

```
composer require ufee/sqlite3
```

## Structure
Object of the BD
```php
\Ufee\Sqlite3\Database;
$db = \Ufee\Sqlite3\Sqlite::database(string $path, array $options = [
	'flags' => SQLITE3_OPEN_READWRITE | SQLITE3_OPEN_CREATE,
	'encryption_key' => null,
	'busy_timeout' => 15,
	'journal_mode' => 'WAL',
	'synchronous' => 'NORMAL',
	'exceptions' => true
]);
```
Object Table
```php
\Ufee\Sqlite3\Table;
$table = $db->table(string $name);
```
Object Request

```php
\Ufee\Sqlite3\Query\Insert;
$insert = $table->insert(array $columns);

\Ufee\Sqlite3\Query\Select;
$select = $table->select(array $columns);

\Ufee\Sqlite3\Query\Update;
$update = $table->update(array $columns);

\Ufee\Sqlite3\Query\Delete;
$delete = $table->delete();
```
Query Collection Object
```php
\Ufee\Sqlite3\Queries;
$queries = $db->queries();
$queries = $table->queries();
```
Debugging Queries 
```php
$db->queries()->listen(function($data) {
	echo 'Table: '.$data['table']."\n";
	echo '  Sql: '.$data['sql']."\n";
	echo ' Time: '.$data['time']."\n";
});
```

## Working with the Class
Working with the Class
```php
$db = Sqlite::database('path/to/file.db');

$temp_db_path = tempnam(sys_get_temp_dir(), 'Sqlite3');
$db = Sqlite::database($temp_db_path);

$db = Sqlite::database(':memory:');
```
Checking for the existence of the database and creating

```php
if (!$db->exists()) {
	$db->create();
}
```Or check for file existence (faster)

```php
if (!$db->fileExists()) {
	$db->create();
}
```
Opening a connection
to the DS In practice is not required, it is performed automatically

```php
$db->open();
```
Execute arbitrary queries and commands

```php
$result = $db->query($query); // return Query\Result
$rows = $result->getRows($mode = SQLITE3_ASSOC);

$result = $db->single($query, $entire = true); // return array
$result = $db->exec($command); // return bool
$result = $db->pragma($key, $val); // return bool
```
Closing the connection to the BD

```php
$db->close();
```
Get a Table Object

```php
$tables = $db->tables();
$table = $db->table('test_table');
```
Checking for existence and creation

```php
if (!$table->exists()) {
	// с указанием типов данных
	$table->create([
		'id' => 'INTEGER PRIMARY KEY', 
		'amount' => 'REAL', 
		'data1' => 'TEXT', 
		'data2' => 'BLOB', 
		'other' => 'INTEGER DEFAULT 5'
	]);
	// или без привязки к типу данных
	$table->create([
		'id, 
		'data', 
		'other'
	]);
}
```
Get information about a table

```php
$info = $table->info($key = null);
// [type, name, tbl_name, rootpage, sql]
```
Get information about a table

```php
$culumns = $table->columns($name = null);
// [name => [cid, name, type, notnull, dflt_value, pk]]
```
Set your column data type for subsequent queries
```php
$table->setColumnType(string $name, string $type); // integer, real, text, blob, null
$table->setColumnType('category', 'integer');
$table->setColumnType('title', 'text');
```
Delete a table
```php
$table->drop();
```

 ## Examples of queries
queries are executed using prepared statements (automatically). 
Condition operators: [=|>|<|<=|>=|!=|BETWEEN|NOT BETWEEN|IN|NOT IN|LIKE|NOT LIKE|GLOB|NOT GLOB]  
For the examples described below, create a database and tables: 
```php
$db = \Ufee\Sqlite3\Sqlite::database(
	tempnam(sys_get_temp_dir(), 'Sqlite3')
);
$goods = $db->table('goods');
$meta = $db->table('goods_meta');

if (!$goods->exists()) {
	$goods->create([
		'id' => 'INTEGER PRIMARY KEY', 
		'category' => 'INTEGER DEFAULT 1', 
		'price' => 'REAL DEFAULT 0.00', 
		'title' => 'TEXT', 
		'hot' => 'INTEGER DEFAULT NULL',
		'created_at' => 'INTEGER DEFAULT (DATETIME(\'now\'))'
	]);
	$meta->create([
		'good_id' => 'INTEGER UNIQUE', 
		'sale' => 'REAL DEFAULT 0.00',
		'descr' => 'TEXT', 
		'star' => 'INTEGER DEFAULT 0', 
	]);
}
```

## Insert Request 
```php
$insert = $table->insert('id, category, data')
	->orRollback()
	->orAbort() // is default 
	->orFail()
	->orIgnore()
	->orRreplace();
$insert->rows(array $rows); // return bool
// or
$insert->row(array $row); // return bool|integer
```
Insert single row 
```php
$insert = $goods->insert('category, price, title');

$increment_id = $insert->orFail()->row([2, 7299.50, 'Notebook Pro']);
// INSERT OR FAIL INTO goods (category, price, title) VALUES (..., ..., ...)

$meta->insert([
	'good_id' => $increment_id, 
	'descr' => 'Professional device',
	'star' => 4
]);
// INSERT INTO goods_meta (good_id, descr, star) VALUES (..., ..., ...)

$increment_id =$insert->orAbort()->row([2, 6999.70, 'Notebook Adv']);
// INSERT OR ABORT INTO goods (category, price, title) VALUES (..., ..., ...)

$meta->insert([
	'good_id' => $increment_id, 
	'descr' => 'Advanced device',
	'star' => 5
]);
// INSERT INTO goods_meta (good_id, descr, star) VALUES (..., ..., ...)

$increment_id = $goods->insert([
	'category' => 3, 
	'title' => 'Mobile Pro',
	'price' => 3799.90
]);
// INSERT INTO goods (category, price, title) VALUES (..., ..., ...)

$meta->insert([
	'good_id' => $increment_id, 
	'sale' => 499.90,
	'descr' => 'Professional device',
	'star' => 3
]);
// INSERT INTO goods_meta (good_id, descr, star) VALUES (..., ..., ..., ....)

```
Вставка нескольких строк
```php
$insert = $goods->insert('category, title');
$result = $insert->rows([
	[4, 'TV 1000'],
	[4, 'TV 2000'],
	[4, 'TV 3000']
]);
// INSERT INTO goods (category, title) VALUES (..., ...), (..., ...), (..., ...)
```

## Запрос Select
```php
$select = $table->select()
	->distinct() // for unique rows
	->where($column, $value = false, $operator = '=')
	->orWhere($column, $value = false, $operator = '=')
	->short($short_table_name)
	->join($join_table_name, $on, $type = '')
	->leftJoin($join_table_name, $on)
	->innerJoin($join_table_name, $on)
	->groupBy($columns)
	->having($column, $value = false, $operator = '=')
	->orHaving($column, $value = false, $operator = '=')
	->orderBy($column, $by = 'DESC');
$count = $select->count(); // return integer
$row = $select->row($column = null); // return array row or string|integer column value
// or
$rows = $select->rows($limit = null, $offset = null); // return array
```
Произвольное условие (без подготовленных выражений)
```php
$select->where('id != another OR ...')
$select->having('other > another AND ...')
```
Получение количества строк
```php
$count = $goods->select()->count();
// SELECT COUNT(*) as rows_count FROM goods LIMIT 1
```
Получение строк и их количества с учетом условий
```php
$select = $goods->select()
	->where('id', 1, '>')
	->orderBy('hot');
$count = $select->count();
// SELECT COUNT(*) as rows_count FROM goods WHERE id > ... ORDER BY hot DESC LIMIT 1
$rows = $select->rows(3);
// SELECT * FROM goods WHERE id > ... ORDER BY hot DESC LIMIT 3

$select = $goods->select('id, category, title')
	->where('hot', null, 'IS NOT')
	->orderBy('hot');
$rows = $select->rows(2,2);
// SELECT id,category,title FROM goods WHERE hot IS NOT NULL ORDER BY hot DESC LIMIT 2 OFFSET 2
```
Получение значений одной строки
```php
$select = $goods->select()->where('id', 1);
$row = $select->row();
// SELECT * FROM goods WHERE id = ... LIMIT 1
```
Получение одного значения из одной строки
```php
$select = $goods->select('title')->where('id', 1);
$title = $select->row('title');
// SELECT * FROM goods WHERE id = ... LIMIT 1
```
Получение с использованием JOIN
```php
$select = $goods->select('g.id, g.category, g.title, m.descr, m.sale, g.hot, m.star, g.created_at')
	->short('g')->innerJoin('goods_meta AS m', 'm.good_id=g.id')
	->where('g.category', [1,2,3,4,5], 'IN')
	->where('m.star', 1, '>')
	->orderBy('m.star');
$count = $select->count();
// SELECT COUNT(*) as rows_count FROM goods AS g INNER JOIN goods_meta AS m ON m.good_id=g.id WHERE g.category IN (...,...,...,...,...) AND m.star > ... ORDER BY m.star DESC LIMIT 1
$rows = $select->rows();
// SELECT g.id,g.category,g.title,m.descr,m.sale,g.hot,m.star,g.created_at FROM goods AS g INNER JOIN goods_meta AS m ON m.good_id=g.id WHERE g.category IN (...,...,...,...,...) AND m.star > ... ORDER BY m.star DESC
```

## Запрос Update
```php
$update = $table->update('category, data')
	->where($column, $value = false, $operator = '=')
	->orWhere($column, $value = false, $operator = '=')
	->orderBy($column, $by = 'DESC')
	->set([$category, $data]);
$update->rows($limit = null, $offset = null); return integer changed rows
// or
$update->row(); return bool
```
Обновление одной строки
```php
$update = $goods->update('price, title, hot')
	->where('id', 1)
	->set([6950.10, 'Notebook Pro (hot)', 1]);
$update->row();
// UPDATE goods SET price=..., title = ..., hot = ... WHERE id = ... LIMIT 1
```
Обновление нескольких строк
```php
$update = $goods->update('hot')
	->where('category', 3)
	->orWhere('price', 5000, '<')
	->set(1);
$update->rows();
// UPDATE goods SET hot = ... WHERE category = ... OR price < ...
$update->rows(3,2);
// UPDATE goods SET hot = ... WHERE category = ... OR price < ... LIMIT 3 OFFSET 2
```

## Запрос Delete
```php
$delete = $table->delete()
	->where($column, $value = false, $operator = '=')
	->orWhere($column, $value = false, $operator = '=')
	->orderBy($column, $by = 'DESC');
$delete->rows($limit = null, $offset = null); return integer changed rows
// or
$delete->row(); return bool
```
Удалние одной строки
```php
$result = $goods->delete()->where('id', 5)->row();
// DELETE FROM goods WHERE id = ... LIMIT 1

$delete = $goods->delete()
	->where('category', 4)
	->orderBy('id');
$result = $delete->row();
// DELETE FROM goods WHERE category = ... ORDER BY id DESC LIMIT 1
```
Удалние нескольких строк
```php
$delete = $goods->delete()
	->where('category', 4)
	->orderBy('id');
$delete->rows(3);
// DELETE FROM goods WHERE category = ... ORDER BY id DESC LIMIT 3
$delete->rows(3, 3);
// DELETE FROM goods WHERE category = ... ORDER BY id DESC LIMIT 3 OFFSET 3
$delete->rows(3, 6);
// DELETE FROM goods WHERE category = ... ORDER BY id DESC LIMIT 3 OFFSET 6
```

## Транзакции
```php
// DEFERRED|IMMEDIATE|EXCLUSIVE
$table->database()->transactionBegin($type = 'DEFERRED', $name = '');
$table->database()->transactionCommit($name = '');
$table->database()->transactionRollback($name = '');
$table->database()->transactionEnd($name = '');
```
Вставка с использованием транзакции
```php
$insert = $goods->insert('category, title');
$goods->database()->transactionBegin('IMMEDIATE');
// BEGIN IMMEDIATE TRANSACTION
$insert->row([5, 'PC Intel']);
// INSERT INTO goods (category, title) VALUES (..., ...)
$insert->row([5, 'PC AMD']);
// INSERT INTO goods (category, title) VALUES (..., ...)
$insert->row([5, 'PC Intel']);
// INSERT INTO goods (category, title) VALUES (..., ...)
$goods->database()->transactionCommit();
// COMMIT TRANSACTION
```
