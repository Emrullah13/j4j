## Pooled SQLite Client for Node.js Apps

A wrapper library that adds ES6 promises, an SQL-based migrations API, connection pooling, and managed transactions to [sqlite3](https://github.com/mapbox/node-sqlite3/) ([docs](https://github.com/mapbox/node-sqlite3/wiki)). Originally based on the [sqlite](https://github.com/kriasoft/node-sqlite/) library.

### How to Install

```sh
$ npm install sqlite-pool --save
```


### How to Use

This module has a similar API as the original `sqlite3` library ([docs](https://github.com/mapbox/node-sqlite3/wiki/API)), except that its equivalent API methods return ES6 Promises and do not accept callback arguments. Full API listings can be found in [API.md](https://github.com/rneilson/node-sqlite-pool/tree/master/API.md).

Below is an example of how to use it with [Node.js](https://nodejs.org) and [Express](http://expressjs.com/starter/hello-world.html), using [Bluebird](http://bluebirdjs.com/) instead of native Promises:

```javascript
const express = require('express');
const Promise = require('bluebird');
const Sqlite = require('sqlite-pool');

const app = express();
const port = process.env.PORT || 3000;
const db = new Sqlite('./database.sqlite', { Promise });

app.get('/post/:id', (req, res, next) => {
  return Promise.all([
    db.get('SELECT * FROM Post WHERE id = ?', req.params.id),
    db.all('SELECT * FROM Category')
  ]).then(([post, categories]) => {
    res.render('post', { post, categories });
  }).catch(next);
});

// Launch Node.js app
app.listen(port);
```

**NOTE**: For Node.js v5 and below use `var Sqlite = require('sqlite-pool/legacy');`.

### Multiple Connections

Due to the asynchronous interface which the [sqlite3](https://github.com/mapbox/node-sqlite3/) Node.js library provides, isolation and query ordering is not guaranteed within any given connection. This module uses the [generic-pool](https://github.com/coopernurse/node-pool) library to create multiple connections to an SQLite database if desired, and to queue requests made using methods of the `Sqlite` class.

The minimum/maximum number of open connections can be set with the `min` and `max` options when calling `new Sqlite()` (see the [API reference](https://github.com/rneilson/node-sqlite-pool/tree/master/API.md) for the full list of options).

Below is an example with the `Sqlite.use()` method of how one connection may perform a transaction isolated from other calls to the same `Sqlite` instance:

```javascript
const express = require('express');
const Promise = require('bluebird');
const Sqlite = require('sqlite-pool');

const app = express();
const port = process.env.PORT || 3000;
const db = new Sqlite('./database.sqlite', { Promise, min: 2, max: 4 });

app.get('/post/:id', (req, res, next) => {
  return Promise.all([
    // This will acquire a database connection, and release it
    // once the returned Promise is resolved or rejected
    db.use((conn) => {
      let id = req.params.id;
      // This Promise chain will begin a transaction, and either
      // commit if successful or rollback if an error is thrown
      return conn.exec('BEGIN IMMEDIATE')
        .then(() => conn.get('SELECT * FROM Post WHERE id = ?', id))
        .then((post) => {
          if (post === undefined) {
            throw new Error(`Post id ${id} not found`);
          }
          return conn.run('UPDATE Post SET views = views + 1 WHERE id = ?', id)
            .then(() => conn.exec('COMMIT'))
            .then(() => post);
        })
        .catch(err => conn.exec('ROLLBACK').then(() => Promise.reject(err)));
    }),
    // This query will run in a separate connection, outside of
    // the above transaction
    db.all('SELECT * FROM Category')
  ]).then(([post, categories]) => {
    res.render('post', { post, categories });
  }).catch(next);
});

// Launch Node.js app
app.listen(port);
```

### Transactions

This module also provides managed transactions, with automatic commit or rollback. Below is a modified version of the above example, using `Sqlite.transaction()`:

```javascript
const express = require('express');
const Promise = require('bluebird');
const Sqlite = require('sqlite-pool');

const app = express();
const port = process.env.PORT || 3000;
const db = new Sqlite('./database.sqlite', { Promise, min: 2, max: 4 });

app.get('/post/:id', (req, res, next) => {
  return Promise.all([
    // This will acquire a database connection, and release it
    // once the returned Promise is resolved or rejected
    db.transaction((trx) => {
      let id = req.params.id;
      // This Promise chain will begin a transaction, and either
      // commit if successful or rollback if an error is thrown
      return trx.get('SELECT * FROM Post WHERE id = ?', id)
        .then((post) => {
          if (post === undefined) {
            throw new Error(`Post id ${id} not found`);
          }
          return trx.run('UPDATE Post SET views = views + 1 WHERE id = ?', id)
            .then(() => post);
        });
    }),
    // This query will run in a separate connection, outside of
    // the above transaction
    db.all('SELECT * FROM Category')
  ]).then(([post, categories]) => {
    res.render('post', { post, categories });
  }).catch(next);
});

// Launch Node.js app
app.listen(port);
```

### Migrations

This module comes with a lightweight migrations API that works with [SQL-based migration files](https://github.com/rneilson/node-sqlite-pool/tree/master/migrations)
as the following example demonstrates:

##### `migrations/001-initial-schema.sql`

```sql
-- Up

CREATE TABLE Category (
  id   INTEGER PRIMARY KEY,
  name TEXT    NOT NULL
);

CREATE TABLE Post (
  id          INTEGER PRIMARY KEY,
  categoryId  INTEGER NOT NULL,
  title       TEXT    NOT NULL,
  views       NUMERIC NOT NULL DEFAULT 0,
  isPublished NUMERIC NOT NULL DEFAULT 0,
  CONSTRAINT Post_fk_categoryId FOREIGN KEY (categoryId)
    REFERENCES Category (id) ON UPDATE CASCADE ON DELETE CASCADE,
  CONSTRAINT Post_ck_isPublished CHECK (isPublished IN (0, 1))
);

INSERT INTO Category (id, name) VALUES (1, 'Business');
INSERT INTO Category (id, name) VALUES (2, 'Technology');

-- Down

DROP TABLE Post;
DROP TABLE Category;
```

##### `migrations/002-missing-index.sql`

```sql
-- Up
CREATE INDEX Post_ix_categoryId ON Post (categoryId);

-- Down
DROP INDEX Post_ix_categoryId;
```

##### `app.js` (Node.js/Express)

```js
const express = require('express');
const Promise = require('bluebird');
const Sqlite = require('sqlite-pool');

const app = express();
const port = process.env.PORT || 3000;
const db = new Sqlite('./database.sqlite', { Promise });

app.use(/* app routes */);

Promise.resolve()
  // First, try to update the database schema to the latest version
  .then(() => db.migrate({
    force: 'last',                  // Default: false
    table: 'migrations',            // Default: 'migrations'
    migrationsPath: './migrations'  // Default: './migrations'
  }))
  .catch(err => console.error(err.stack))
  // Finally, launch Node.js app
  .finally(() => app.listen(port));
```

**NOTE**: For the development environment, while working on the database schema, you may want to set `force: 'last'` (default `false`) that will force the migration API to rollback and re-apply the latest migration over again each time when Node.js app launches. 


### References

* [SQLite Documentation](https://www.sqlite.org/docs.html), e.g. [SQL Syntax](https://www.sqlite.org/lang.html), [Data Types](https://www.sqlite.org/datatype3.html) etc. on SQLite.org


### License

The MIT License © 2017 Raymond Neilson. All rights reserved.
Original work © 2015-2016 Kriasoft, LLC. All rights reserved.

Original library by Konstantin Tarkus ([@koistya](https://twitter.com/koistya)) and [contributors](https://github.com/rneilson/node-sqlite-pool/graphs/contributors)
