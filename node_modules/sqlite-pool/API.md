# API

## Class: Sqlite

### new Sqlite([filename], [options])

Returns a new Sqlite object and automatically opens `options.min` connections to the database. There is no separate method to open the database. Inherits from `EventEmitter`.

* `filename`: Valid values are filenames, `':memory:'` for an anonymous in-memory database and `''` (the empty string) for an anonymous disk-based database. Anonymous databases are not persisted and when closing the database handle, their contents are lost. Default: `':memory:'`.

* `options`:
  * `mode`: One or more of `Sqlite.OPEN_READONLY`, `Sqlite.OPEN_READWRITE` and `Sqlite.OPEN_CREATE`. See [documentation](https://www.sqlite.org/c3ref/open.html) for details. Default: `OPEN_READWRITE | OPEN_CREATE`.
  * `verbose`: Sets the execution mode for each new connection to verbose to produce long stack traces. See the [node-sqlite3 wiki page on debugging](https://github.com/developmentseed/node-sqlite3/wiki/Debugging) for more information. Default: `false`.
  * `busyTimeout`: Sets the [busy timeout](https://www.sqlite.org/c3ref/busy_timeout.html) in milliseconds. Default: `1000`.
  * `foreignKeys`: Enables [foreign key constraints](https://www.sqlite.org/foreignkeys.html) by executing the command `PRAGMA foreign_keys = ON;` for each new connection created. Default: `true`.
  * `walMode`: Enables [write-ahead logging](https://www.sqlite.org/wal.html) (WAL) mode by executing the command `PRAGMA journal_mode = WAL;` for each new connection created. Default: `true`.
  * `loadExtensions`: Array of [extension library names](https://www.sqlite.org/c3ref/load_extension.html) to load for each connection. Default: `[]`.
  * `min`: Sets the minimum number of connections in the pool. Will be silently set to `1` if the in-memory (`':memory:'`) or anonymous disk-based (`''`) filenames are set. Default: `1`.
  * `max`: Sets the maximum number of connections in the pool. Will be silently increased to the value of `min` if `max` is lower. Will be silently set to `1` if the in-memory (`':memory:'`) or anonymous disk-based (`''`) filenames are set. Default: `1`.
  * `acquireTimeout`: Sets the maximum time to wait to acquire a new connection, in milliseconds. Default: `1000`.
  * `trxImmediate`: Enables starting transactions with `BEGIN IMMEDIATE` instead of `BEGIN`. Can reduce [lock escalation deadlocks](https://www.sqlite.org/lang_transaction.html#immediate), especially in conjunction with WAL mode. Default: `true`.
  * `delayRelease`: Enables using `setImmediate()` to delay releasing connections back to the pool. This allows a Promise chain to continue before the next queued request is processed. Default: `true`.
  * `Promise`: Promise library to use. Default: `global.Promise`.

### Event: 'error'

* `'error' <Error>`

Emitted if an error occurs in an underlying `sqlite3` object outside of a Promise chain, or when the connection pool encounters an error when creating or destroying a connection.

### Event: 'open'

* `'filename' <String>`
* `'driver' <sqlite3>`

Emitted when an underlying `sqlite3` object successfully opens the given database file.

### Event: 'close'

* `'filename' <String>`
* `'driver' <sqlite3>`

Emitted when an underlying `sqlite3` object successfully closes the given database file.

### Event: 'trace'
### Event: 'profile'

These events are emitted as per the [`sqlite3` debugging API](https://github.com/mapbox/node-sqlite3/wiki/Debugging) when the `verbose` option is `true`.

### sqlite.close()

Closes the database. Returns a Promise.

### sqlite.run(sql, [param, ...])

Acquires a connection from the pool as a Database object, calls `database.run()` with the given arguments, then releases the connection to the pool. Returns a Promise which resolves with a finalized Statement object or rejects with an error object.

### sqlite.exec(sql)

Acquires a connection from the pool as a Database object, calls `database.exec()` with the given arguments, then releases the connection to the pool. Returns a Promise which resolves with `undefined` if execution is successful, or rejects with an error object.

### sqlite.get(sql, [param, ...])

Acquires a connection from the pool as a Database object, calls `database.get()` with the given arguments, then releases the connection to the pool. Returns a Promise which resolves with `undefined` if the result set is empty, otherwise with an object containing the values for the first result row, or rejects with an error object.

### sqlite.all(sql, [param, ...])

Acquires a connection from the pool as a Database object, calls `database.all()` with the given arguments, then releases the connection to the pool. Returns a Promise which resolves with an empty array if the result set is empty, otherwise with an array of objects, one for each result row which in turn contains the values of that row, like `database.get()`, or rejects with an error object.

### sqlite.each(sql, [param, ...], [callback])

Acquires a connection from the pool as a Database object, calls `database.each()` with the given arguments, then releases the connection to the pool. Returns a Promise which resolves with the number of retrieved rows, or rejects with an error object.

### sqlite.use(callback)

Acquires a connection from the pool as a Database object, calls the callback, then releases the connection to the pool. Returns a Promise which resolves with the return value of the callback, or rejects with an error object.

The signature of the callback is `function (database) {}`. If there is an error acquiring a connection, the callback will not be called. If the return value of the callback is a Promise, the connection will not be released until the Promise is resolved or rejected.

### sqlite.useAsync(generator)

As with `sqlite.use()`, but taking a generator function with signature `function* (database) {}`, which is then called and iterated over with an executor derived from Babel's [async to generator transform](https://babeljs.io/docs/plugins/transform-async-to-generator/). This allows async/await style code using `yield` instead of `await`, where execution will suspend on yielded Promises, and resumed when resolved or rejected, with the advantage that the Promise library configured with the `Promise` option given to the Sqlite object will be used. Returns a Promise which resolves with the return value of the generator, or rejects with an error object.

### sqlite.transaction(callback, immediate)

Acquires a connection from the pool as a Database object, calls `database.transaction()` with the given arguments, then releases the connection to the pool. Returns a Promise resolving with the return value of `database.transaction()`.

### sqlite.transactionAsync(generator, immediate)

As with `sqlite.transaction()`, but taking a generator function with signature `function* (database) {}`, as with `sqlite.useAsync()`. Returns a Promise which resolves with the return value of the generator, or rejects with an error object.

### sqlite.migrate([options])

Parses and applies SQL-based migrations. Each filename must be in the format `<id><separator><name>.sql`, eg `001-initial.sql` or `3.new-feature.sql`. Each file must have an 'up' and 'down' section, separated by a line consisting of `-- down` (case insensitive). SQL statements in the 'up' section will be executed when applying the migration, and those in the 'down' section when rolling back the migration.

* `options`:
  * `force`: Specified migration id to apply up to or roll back down to, if an integer, or `'last'` to rollback and re-apply the latest migration. No default value (will apply up to and including the latest).
  * `table`: Name to use for the table used to track applied migrations. Default `'migrations'`.
  * `migrationsPath`: Directory in which the migration files can be found. Default `'./migrations'`.

### Constants

`Sqlite.OPEN_READONLY`  
`Sqlite.OPEN_READWRITE`  
`Sqlite.OPEN_CREATE`  
`Sqlite.VERSION`  
`Sqlite.SOURCE_ID`  
`Sqlite.VERSION_NUMBER`  
`Sqlite.OK`  
`Sqlite.ERROR`  
`Sqlite.INTERNAL`  
`Sqlite.PERM`  
`Sqlite.ABORT`  
`Sqlite.BUSY`  
`Sqlite.LOCKED`  
`Sqlite.NOMEM`  
`Sqlite.READONLY`  
`Sqlite.INTERRUPT`  
`Sqlite.IOERR`  
`Sqlite.CORRUPT`  
`Sqlite.NOTFOUND`  
`Sqlite.FULL`  
`Sqlite.CANTOPEN`  
`Sqlite.PROTOCOL`  
`Sqlite.EMPTY`  
`Sqlite.SCHEMA`  
`Sqlite.TOOBIG`  
`Sqlite.CONSTRAINT`  
`Sqlite.MISMATCH`  
`Sqlite.MISUSE`  
`Sqlite.NOLFS`  
`Sqlite.AUTH`  
`Sqlite.FORMAT`  
`Sqlite.RANGE`  
`Sqlite.NOTADB`


## Class: Sqlite.Database

### database.run(sql, [param, ...])

Runs the SQL query with the specified parameters. It does not retrieve any result data. Returns a Promise which resolves with a finalized Statement object, or rejects with an error object.

If execution was successful, the Statement object will contain properties named `lastID` and `changes` which contain the value of the last inserted row ID and the number of rows affected by this query respectively. Note that `lastID` **only** contains valid information when the query was a successfully completed `INSERT` statement, and `changes` **only** contains valid information when the query was a successfully completed `UPDATE` or `DELETE` statement. In all other cases, the content of these properties is inaccurate and should not be used. The `.run()` function is the only query method that returns these two values; all other query methods such as `.all()` or `.get()` don't retrieve these values.

* `sql`: The SQL query to run. If the SQL query is invalid, the Promise will be rejected with an error object containing the error message from SQLite.

* `param, ...` *(optional)*: When the SQL statement contains placeholders, you can pass them in here. They will be bound to the statement before it is executed. There are three ways of passing bind parameters: directly in the function's arguments, as an array, and as an object for named parameters. This automatically sanitizes inputs RE: [node-sqlite3 issue #57](https://github.com/mapbox/node-sqlite3/issues/57).

```javascript
    // Directly in the function arguments.
    db.run("UPDATE tbl SET name = ? WHERE id = ?", "bar", 2);

    // As an array.
    db.run("UPDATE tbl SET name = ? WHERE id = ?", [ "bar", 2 ]);

    // As an object with named parameters.
    db.run("UPDATE tbl SET name = $name WHERE id = $id", {
        $id: 2,
        $name: "bar"
    });
```

  Named parameters can be prefixed with `:name`, `@name` and `$name`. We recommend using `$name` since JavaScript allows using the dollar sign as a variable name without having to escape it. You can also specify a numeric index after a `?` placeholder. These correspond to the position in the array. Note that placeholder indexes start at 1 in SQLite. `node-sqlite3` maps arrays to start with one so that you don't have to specify an empty value as the first array element (with index 0). You can also use numeric object keys to bind values. Note that in this case, the first index is 1:
```javascript
    db.run("UPDATE tbl SET name = ?5 WHERE id = ?", {
        1: 2,
        5: "bar"
    });
```

  This binds the first placeholder (`$id`) to `2` and the placeholder with index `5` to `"bar"`. While this is valid in SQLite and `node-sqlite3`, it is not recommended to mix different placeholder types.

  If you use an array or an object to bind parameters, it must be the first value in the bind arguments list. If any other object is before it, an error will be thrown. Additional bind parameters after an array or object will be ignored.

### database.exec(sql)

Runs all SQL queries in the supplied string. No result rows are retrieved. Returns a Promise. If a query fails, no subsequent statements will be executed (wrap it in a transaction if you want all or none to be executed). When all statements have been executed successfully, the Promise is resolved, or when an error occurs, the Promise will be rejected with an error object.

Note: This function will only execute statements up to the first NULL byte. **Comments are not allowed** and will lead to runtime errors.

### database.get(sql, [param, ...])

Runs the SQL query with the specified parameters as with `database.run()`. Returns a Promise which resolves with `undefined` if the result set is empty, otherwise it is an object containing the values for the first result row. The property names correspond to the column names of the result set. It is impossible to access them by column index; the only supported way is by column name.

### database.all(sql, [param, ...])

Runs the SQL query with the specified parameters as with `database.run()`. Returns a Promise which resolves with an empty array if the result set is empty, otherwise with an array of objects, one for each result row which in turn contains the values of that row, like `database.get()`, or rejects with an error object.

Note that it first retrieves all result rows and stores them in memory. For queries that have potentially large result sets, use the `database.each()` function to retrieve all rows, or use `database.prepare()` to create a new Statement object and multiple `statement.get()` calls without new parameters to retrieve rows individually.

### database.each(sql, [param, ...], callback)

Runs the SQL query with the specified parameters as with `database.run()`, and calls the callback for each result row. Returns a Promise which resolves with the number of retrieved rows, if successful, or rejects with an error object.

The signature of the callback is `function(row) {}`. If the result set succeeds but is empty, the callback is never called. Otherwise, the callback is called once for every retrieved row, unless the callback throws an error, and the Promise will be rejected with the thrown error object once the query is complete. The order of calls correspond exactly to the order of rows in the result set.

If you know that a query only returns a very limited number of rows, it might be more convenient to use `database.all()` to retrieve all rows at once.

There is currently no way to abort execution of the query, but if the callback throws an error, subsequent calls will be skipped.

### database.prepare(sql, [param, ...])

Prepares the SQL statement and optionally binds the specified parameters and calls the callback when done. Returns a Promise which resolves with a Statement object when preparing was successful, otherwise rejects with an error object. When bind parameters are supplied, they are bound to the prepared statement before resolving.

### database.transaction(callback, immediate)

Begins a transaction, calls the callback, and either commits the transaction if the callback is executed successfully, or rolls back the transaction if an error is thrown by the callback. Returns a Promise which resolves to the return value of the callback if successful, otherwise rejects with an error object.

The signature of the callback is `function (database) {}`. If there is an error beginning the transaction, the callback will not be called. If the return value of the callback is a Promise, the transaction will not be committed or rolled back until the Promise is resolved or rejected.

If `immediate` is `true`, an immediate transaction will be started with `BEGIN IMMEDIATE`, otherwise a deferred transaction will be started with `BEGIN`. Default is the value of the `trxImmediate` option given to the parent Sqlite object, which defaults to `true` to avoid lock escalation deadlocks. Please consult the [SQLite documentation](https://www.sqlite.org/lang_transaction.html#immediate) for details.

### database.transactionAsync(generator, immediate)

As with `database.transaction()`, but taking a generator function with signature `function* (database) {}`, which is then called and iterated over with an executor derived from Babel's [async to generator transform](https://babeljs.io/docs/plugins/transform-async-to-generator/). This allows async/await style code using `yield` instead of `await`, where execution will suspend on yielded Promises, and resumed when resolved or rejected, with the advantage that the Promise library configured with the `Promise` option given to the parent Sqlite object will be used.


## Class: Sqlite.Statement

### statement.sql

Read-only getter property. Contains the SQL query string of the statement.

### statement.lastID

Read-only getter property. Contains the last inserted row ID. Only valid when the query was a successfully completed `INSERT` statement. In all other cases, the content of this property is inaccurate and should not be used.

### statement.changes

Read-only getter property. Contains the number of rows affected by this query. Only valid when the query was a successfully completed `UPDATE` or `DELETE` statement. In all other cases, the content of this property is inaccurate and should not be used.

### statement.bind([param, ...])

Binds parameters to the prepared statement. Returns a Promise which resolves with the same Statement object when binding was successful, otherwise rejects an error object.

Binding parameters with this function completely resets the statement object and row cursor and removes all previously bound parameters, if any.

### statement.reset()

Resets the row cursor of the statement and preserves the parameter bindings. Use this function to re-execute the same query with the same bindings. Returns a Promise which resolves with the same Statement object.

### statement.finalize()

Finalizes the statement. Returns a Promise which resolves with the same Statement object.

This is typically optional, but if you experience long delays before the next query is executed, explicitly finalizing your statement might be necessary. After the statement is finalized, all further function calls on that statement object will throw errors.

### statement.run([param, ...])

Binds the specified parameters and executes the SQL query as with `database.run()`. Returns a Promise which resolves with the same Statement object, or rejects with an error object. This behaves identically to `database.run()` except that the Statement object will not be finalized after execution. This means you can run it multiple times.

If you specify bind parameters, they will be bound to the statement before it is executed. Note that the bindings and the row cursor are reset when you specify even a single bind parameter.

### statement.get([param, ...])

Binds the specified parameters and executes the SQL query as with `statement.run()`. Returns a Promise which resolves with `undefined` if the result set is empty, otherwise with an object containing the values for the first result row. The property names correspond to the column names of the result set. It is impossible to access them by column index; the only supported way is by column name.

Like with `statement.run()`, the statement will not be finalized after executing this function.

If no parameters are given, and the statement has already been bound using `database.prepare()`, `statement.bind()`, or a previous call to `statement.get()`, the next row will be retrieved.

### statement.all([param, ...])

Binds the specified parameters and executes the SQL query as with `statement.run()`. Returns a Promise which resolves with an empty array if the result set is empty, otherwise with an array of objects, one for each result row which in turn contains the values of that row, like `database.get()`, or rejects with an error object.

Note that it first retrieves all result rows and stores them in memory. For queries that have potentially large result sets, use the `statement.each()` function to retrieve all rows, or bind parameters with `statement.bind()` and use multiple `statement.get()` calls without new parameters to retrieve rows individually.

Like with `statement.run()`, the statement will not be finalized after executing this function.

### statement.each([param, ...], callback)

Binds the specified parameters and executes the SQL query as with `statement.run()`, and calls the callback for each result row. Returns a Promise which resolves with the number of retrieved rows, if successful, or rejects with an error object.

The signature of the callback is `function(row) {}`. If the result set succeeds but is empty, the callback is never called. Otherwise, the callback is called once for every retrieved row, unless the callback throws an error, and the Promise will be rejected with the thrown error object once the query is complete. The order of calls correspond exactly to the order of rows in the result set.

Like with `statement.run()`, the statement will not be finalized after executing this function.

If you know that a query only returns a very limited number of rows, it might be more convenient to use `statement.all()` to retrieve all rows at once.

There is currently no way to abort execution of the query, but if the callback throws an error, subsequent calls of the callback will be skipped.
