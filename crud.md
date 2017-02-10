##CRUD of Object

If the value of the primary key is already known, Read, Insert, Update, Delete can be used to manipulate the object.
```
o := orm.NewOrm()
user := new(User)
user.Name = "slene"

fmt.Println(o.Insert(user))

user.Name = "Your"
fmt.Println(o.Update(user))
fmt.Println(o.Read(user))
fmt.Println(o.Delete(user))
```

###Read
```
o := orm.NewOrm()
user := User{Id: 1}

err := o.Read(&user)

if err == orm.ErrNoRows {
    fmt.Println("No result found.")
} else if err == orm.ErrMissPK {
    fmt.Println("No primary key found.")
} else {
    fmt.Println(user.Id, user.Name)
}
```
Read uses primary key by default. But it can use other fields as well:
```
user := User{Name: "slene"}
err := o.Read(&user, "Name")
...
```
Other fields of the object are set to the default value according to the field type.


###ReadOrCreate

Try to read a row from the database, or insert one if it doesn’t exist.

At least one condition field must be supplied, multiple condition fields are also supported.
```
o := orm.NewOrm()
user := User{Name: "slene"}
// Three return values：Is Created，Object Id，Error
if created, id, err := o.ReadOrCreate(&user, "Name"); err == nil {
    if created {
        fmt.Println("New Insert an object. Id:", id)
    } else {
        fmt.Println("Get an object. Id:", id)
    }
}
```

###Insert

The first return value is auto inc Id value.
```
o := orm.NewOrm()
var user User
user.Name = "slene"
user.IsActive = true

id, err := o.Insert(&user)
if err == nil {
    fmt.Println(id)
}
```

###Update

The first return value is the number of affected rows.
```
o := orm.NewOrm()
user := User{Id: 1}
if o.Read(&user) == nil {
    user.Name = "MyName"
    if num, err := o.Update(&user); err == nil {
        fmt.Println(num)
    }
}
```
Update updates all fields by default. You can update specified fields:
```
// Only update Name
o.Update(&user, "Name")
// Update multiple fields
// o.Update(&user, "Field1", "Field2", ...)
...
```
For detailed object update, see One

###Delete

The first return value is the number of affected rows.
```
o := orm.NewOrm()
if num, err := o.Delete(&User{Id: 1}); err == nil {
    fmt.Println(num)
}
```
Delete will also manipulate reverse relationships. E.g.: Post has a foreign key to User. If on_delete is set to cascade, Post will be deleted while delete User.

After deleting it will clean up values for auto fields.


## Registering Model

This is compulsory if you use orm.QuerySeter for advanced query.

Otherwise you don't need to do this if you use raw SQL query and map struct only. [Raw SQL Query](rawsql.md)

#### RegisterModel

Register the Model you defined. The best practice is to have a single models.go file and register in it's init function.

Mini models.go

```go
package main

import "git.qasico.com/cuxs/orm"

type User struct {
    Id   int
    name string
}

func init(){
    orm.RegisterModel(new(User))
}
```

RegisterModel can register multiple models at the same time:

```go
orm.RegisterModel(new(User), new(Profile), new(Post))
```

For detailed struct definition, see [Model define](models.md)

#### Generate Tables

You may want Beego to automatically create your database tables.
One way to do this is by using the method described in the [cli](cmd.md) documentation. 
Alternatively you could choose to autogenerate your tables by including the following
in your main.go file in your main block. 

```go
// Database alias.
name := "default"

// Drop table and re-create.
force := true

// Print log.
verbose := true

// Error.
err := orm.RunSyncdb(name, force, verbose)
if err != nil {
    fmt.Println(err)
}
```
After the initial "bee run" command, change the values of force and verbose to false. 
The default behavior for Beego is to add additional columns when the model is updated.
You will need to manually handle dropping your columns if they are removed from your model. 


#### RegisterModelWithPrefix

Using table prefix

```go
orm.RegisterModelWithPrefix("prefix_", new(User))
```

The created table name is prefix_user

#### NewOrmWithDB

You may need to manage db pools by yourself. (eg: need two query in one connection)

But you want use orm awesome features. Bingo!

```go
var driverName, aliasName string
// driverName name of your driver (go-sql-driver: mysql)
// aliasName custom db alias name
var db *sql.DB
...
o := orm.NewOrmWithDB(driverName, aliasName, db)
```

#### GetDB

Get *sql.DB from registered databases. Will use `default` as default if you do not set.

```go
db, err := orm.GetDB()
if err != nil {
    fmt.Println("get default DataBase")
}

db, err := orm.GetDB("alias")
if err != nil {
    fmt.Println("get alias DataBase")
}
```

#### ResetModelCache

Reset registered models. Commonly used to write test cases.

```go
orm.ResetModelCache()
```

## ORM API Usage

Let's see how to use Ormer API:

```go
var o Ormer
o = orm.NewOrm() // create a Ormer // While running NewOrm, it will run orm.BootStrap (only run once in the whole app lifetime) to validate the definition between models and cache it.
```
Switching database or using transactions will affect Ormer object and all its queries.

So don't use a global Ormer object if you need to switch databases or use transactions.


* type Ormer interface {
    * [Read(interface{}, ...string) error](object.md#read)
    * [ReadOrCreate(interface{}, string, ...string) (bool, int64, error)](object.md#readorcreate)
    * [Insert(interface{}) (int64, error)](object.md#insert)
    * [InsertMulti(int, interface{}) (int64, error)](object.md#insertmulti)
    * [Update(interface{}, ...string) (int64, error)](object.md#update)
    * [Delete(interface{}) (int64, error)](object.md#delete)
    * [LoadRelated(interface{}, string, ...interface{}) (int64, error)](query.md#)
    * [QueryM2M(interface{}, string) QueryM2Mer](query.md#)
    * [QueryTable(interface{}) QuerySeter](#querytable)
    * [Using(string) error](#using)
    * [Begin() error](transaction.md)
    * [Commit() error](transaction.md)
    * [Rollback() error](transaction.md)
    * [Raw(string, ...interface{}) RawSeter](#raw)
    * [Driver() Driver](#driver)
* }


#### QueryTable

Pass in a table name or a Model object and return a [QuerySeter](query.md#queryseter)

```go
o := orm.NewOrm()
var qs QuerySeter
qs = o.QueryTable("user")
// Panics if the table can't be found
```

#### Using

Switch to  another database:

```go
orm.RegisterDataBase("db1", "mysql", "root:root@/orm_db2?charset=utf8")
orm.RegisterDataBase("db2", "sqlite3", "data.db")

o1 := orm.NewOrm()
o1.Using("db1")

o2 := orm.NewOrm()
o2.Using("db2")

// After switching database
// The API calls of this Ormer will use the new database
```

Use `default` database, no need to use `Using`

#### Raw

Use raw SQL query:

Raw function will return a [RawSeter](rawsql.md) to execute a query with the SQL and params provided:

```go
o := NewOrm()
var r RawSeter
r = o.Raw("UPDATE user SET name = ? WHERE name = ?", "testing", "slene")
```

#### Driver

The current db infomation used by ORM

```go
type Driver interface {
    Name() string
    Type() DriverType
}
```

```go
orm.RegisterDataBase("db1", "mysql", "root:root@/orm_db2?charset=utf8")
orm.RegisterDataBase("db2", "sqlite3", "data.db")

o1 := orm.NewOrm()
o1.Using("db1")
dr := o1.Driver()
fmt.Println(dr.Name() == "db1") // true
fmt.Println(dr.Type() == orm.DRMySQL) // true

o2 := orm.NewOrm()
o2.Using("db2")
dr = o2.Driver()
fmt.Println(dr.Name() == "db2") // true
fmt.Println(dr.Type() == orm.DRSqlite) // true
```

## Print out SQL query in debugging mode

Setting `orm.Debug` to true will print out SQL queries

It may cause performance issues. It's not recommend to be used in a production env.

```go
func main() {
    orm.Debug = true
...
```
Prints to `os.Stderr` by default.

You can change it to your own `io.Writer`

```go
var w io.Writer
...
// Use your `io.Writer`
...
orm.DebugLog = orm.NewLog(w)
```

Logs formatting

```go
[ORM] - time - [Queries/database name] - [operation/executing time] - [SQL query] - separate params with `,`  -errors 
```

```go
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [    db.Exec /     0.4ms] - [INSERT INTO `user` (`name`) VALUES (?)] - `slene`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [    db.Exec /     0.5ms] - [UPDATE `user` SET `name` = ? WHERE `id` = ?] - `astaxie`, `14`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [db.QueryRow /     0.4ms] - [SELECT `id`, `name` FROM `user` WHERE `id` = ?] - `14`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [    db.Exec /     0.4ms] - [INSERT INTO `post` (`user_id`,`title`,`content`) VALUES (?, ?, ?)] - `14`, `beego orm`, `powerful amazing`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [   db.Query /     0.4ms] - [SELECT T1.`name` `User__Name`, T0.`user_id` `User`, T1.`id` `User__Id` FROM `post` T0 INNER JOIN `user` T1 ON T1.`id` = T0.`user_id` WHERE T0.`id` = ? LIMIT 1000] - `68`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [    db.Exec /     0.4ms] - [DELETE FROM `user` WHERE `id` = ?] - `14`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [   db.Query /     0.3ms] - [SELECT T0.`id` FROM `post` T0 WHERE T0.`user_id` IN (?) ] - `14`
[ORM] - 2013-08-09 13:18:16 - [Queries/default] - [    db.Exec /     0.4ms] - [DELETE FROM `post` WHERE `id` IN (?)] - `68`
```

The log contains all the database operations, transactions, prepare etc.

