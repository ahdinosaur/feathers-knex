feathers-knex
================

[![Build Status](https://travis-ci.org/feathersjs/feathers-knex.png?branch=master)](https://travis-ci.org/feathersjs/feathers-knex)

> A [Knex.js](http://knexjs.org/) service adapter for [FeathersJS](http://feathersjs.com)


## Installation

```bash
npm install feathers-knex --save
```


## Getting Started

You can create a SQL Knex service like this:

```js
var knex = require('knex')({
  client: 'sqlite3',
  connection: {
    filename: './db.sqlite'
  }
});
var knexService = require('feathers-knex');

app.use('/todos', knexService('todos', {
  Model: knex,
  name: 'todos'
}));
```

This will create a `todos` endpoint and connect to a local `todos` table on an SQLite database in `data.db`.


### Complete Example

Here's a complete example of a Feathers server with a `todos` SQLite service. We are using the [Knex schema builder](http://knexjs.org/#Schema)

```js
// server.js
var feathers = require('feathers');
var bodyParser = require('body-parser');
var knexService = require('../lib');
var knex = require('knex')({
  client: 'sqlite3',
  connection: {
    filename: './db.sqlite'
  }
});

// This create your database table
knex.schema.createTableIfNotExists('todos', function(table) {
  table.increments('id');
  table.string('text');
  table.boolean('complete');
});

// Create a feathers instance.
var app = feathers()
  // Setup the public folder.
  .use(feathers.static(__dirname + '/public'))
  // Enable Socket.io
  .configure(feathers.socketio())
  // Enable REST services
  .configure(feathers.rest())
  // Turn on JSON parser for REST services
  .use(bodyParser.json())
  // Turn on URL-encoded parser for REST services
  .use(bodyParser.urlencoded({extended: true}))

// Create Knex Feathers service with a default page size of 2 items
// and a maximum size of 4
app.use('/todos', knexService({
  Model: knex,
  name: 'todos',
  paginate: {
    default: 2,
    max: 4
  }
}));

// Start the server.
var port = 3030;
app.listen(port, function() {
    console.log('Feathers server listening on port ' + port);
});
```

You can run this example by using `node example/app` and going to [localhost:3030/todos](http://localhost:3030/todos). You should see an empty array. That's because you don't have any Todos yet but you now have full CRUD for your new todos service!

## Options

Creating a new Knex service currently offers the following options:

- `name` (**required**) - this is the table name for your resource.
- `id` (default: `id`) [optional] - The name of the id property
- `paginate` [optional] - A pagination object containing a `default` and `max` page size (see below)

_You will also need to pass additional Knex connection options. [See Knex docs](http://knexjs.org/#Installation-client)._

## Pagination

When initializing the service you can set the following pagination options in the `paginate` object:

- `default` - Sets the default number of items
- `max` - Sets the maximum allowed number of items per page (even if the `$limit` query parameter is set higher)

When `paginate.default` is set, `find` will return an object (instead of the normal array) in the following form:

```
{
  "total": "<total number of records>",
  "limit": "<max number of items per page>",
  "skip": "<number of skipped items (offset)>",
  "data": [/* data */]
}
```


## Extending

To extend the basic service there are two options. Either through using [Uberproto's](https://github.com/daffl/uberproto) inheritance mechanism or by using [feathers-hooks](https://github.com/feathersjs/feathers-hooks).


### With Uberproto

The basic Knex service is implemented using [Uberproto](https://github.com/daffl/uberproto), a small EcmaScript 5 inheritance library so you can use the Uberproto syntax to add your custom functionality.
For example, you might want `update` and `patch` to behave the same (the basic implementation of `update` replaces the entire object instead of merging it) and add an `updatedAt` and `createdAt` flag to your data:

```js
// myservice.js
var knex = require('feathers-knex');
var Proto = require('uberproto');

var TimestampPatchService = knex.Service.extend({
  create: function(data, params, callback) {
    data.createdAt = new Date();

    // Call the original `create`
    return this._super(data, params, callback);
  },

  update: function() {
    // Call `patch` instead so that PUT calls merge
    // instead of replace data, too
    this.patch(id, data, params, callback);
  },

  patch: function(id, data, params, callback) {
    data.updatedAt = new Date();

    // Call the original `patch`
    this._super(id, data, params, callback);
  }
});

// Export a simple function that instantiates this new service like
// var myservice = require('myservice');
// app.use('/users', myservice(options));
module.exports = function(options) {
  // We need to call `Proto.create` explicitly here since we are overriding
  // the original `create` method
  return Proto.create.call(TimestampPatchService, options);
}

module.exports.Service = TimestampPatchService;
```


### With hooks

Another option is to weave functionality into your existing services using [feathers-hooks](https://github.com/feathersjs/feathers-hooks), for example the above `createdAt` and `updatedAt` functionality:

```js
var feathers = require('feathers');
var hooks = require('feathers-hooks');
var knexService = require('feathers-knex');
var knex = require('knex')({
  client: 'sqlite3',
  connection: {
    filename: './db.sqlite'
  }
});

// Initialize a Knex service with the users table in an SQLite
var app = feathers()
  .configure(hooks())
  .use('/users', knexService('users', {
    Model: knex,
    name: 'todos'
  }));

app.service('users').before({
  create: function(hook, next) {
    hook.data.createdAt = new Date();
    next();
  },

  update: function(hook, next) {
    hook.data.updatedAt = new Date();
    next();
  }
});

app.listen(8080);
```


## Query Parameters

The `find` API allows the use of `$limit`, `$skip`, `$sort`, and `$select` in the query.  These special parameters can be passed directly inside the query object:

```js
// Find all recipes that include salt, limit to 10, only include name field.
{"ingredients":"salt", "$limit":10, "$select": { "name" :1 } } // JSON

GET /?ingredients=salt&$limit=10&$select[name]=1 // HTTP
```

As a result of allowing these to be put directly into the query string, you won't want to use `$limit`, `$skip`, `$sort`, or `$select` as the name of fields in your document schema.

### `$limit`

`$limit` will return only the number of results you specify:

```
// Retrieves the first two records found where age is 37.
query: {
  age: 37,
  $limit: 2
}
```

### `$skip`

`$skip` will skip the specified number of results:

```
// Retrieves all except the first two records found where age is 37.
query: {
  age: 37,
  $skip: 2
}
```

### `$sort`

`$sort` will sort based on the object you provide:

```
// Retrieves all where age is 37, sorted ascending alphabetically by name.
query: {
  age: 37,
  $sort: { name: 1 }
}

// Retrieves all where age is 37, sorted descending alphabetically by name.
query: {
  age: 37,
  $sort: { name: -1}
}
```

### `$select`

`$select` support in a query allows you to pick which fields to include or exclude in the results.

```
// Only retrieve name.
query: {
  name: 'Alice',
  $select: {'name': 1}
}

// Retrieve everything except age.
query: {
  name: 'Alice',
  $select: {'age': 0}
}
```


## Filter criteria

In addition to sorting and pagination, properties can also be filtered by criteria. Standard criteria can just be added to the query. For example, the following find all users with the name `Alice`:

```js
query: {
  name: 'Alice'
}
```

Additionally, the following advanced criteria are supported for each property.

### $in, $nin

Find all records where the property does (`$in`) or does not (`$nin`) contain the given values. For example, the following query finds every user with the name of `Alice` or `Bob`:

```js
query: {
  name: {
    $in: ['Alice', 'Bob']
  }
}
```

### $lt, $lte

Find all records where the value is less (`$lt`) or less and equal (`$lte`) to a given value. The following query retrieves all users 25 or younger:

```js
query: {
  age: {
    $lte: 25
  }
}
```

### $gt, $gte

Find all records where the value is more (`$gt`) or more and equal (`$gte`) to a given value. The following query retrieves all users older than 25:

```js
query: {
  age: {
    $gt: 25
  }
}
```

### $ne

Find all records that do not contain the given property value, for example anybody not age 25:

```js
query: {
  age: {
    $ne: 25
  }
}
```

### $or

Find all records that match any of the given objects. For example, find all users name Bob or Alice:

```js
query: {
  $or: [
    { name: 'Alice' },
    { name: 'Bob' }
  ]
}
```


## Changelog

__2.0.0__

- Making compatible with Feathers 2.0 and consistent with other adapters [#24](https://github.com/feathersjs/feathers-knex/issues/24)
- Updating example
- Adding s'more tests

__1.3.0__

- fixing docs
- adding an example app
- fixing queries so that they pass all tests
- adding mappings to feathers-errors for SQLite

__1.2.x__

- Babel 6 support, Object.assign polyfill and CommonJS module backwards compatibility

__1.1.0__

- Compatibility with latest service tests

__1.0.0__

- Initial release

## License

Copyright (c) 2015

Licensed under the [MIT license](LICENSE).
