feathers-mongoose-advanced Service
=========================

[![NPM](https://nodei.co/npm/feathers-mongoose-advanced.png?downloads=true&stars=true)](https://nodei.co/npm/feathers-mongoose-advanced/)


> Create a flexible [Mongoose](http://mongoosejs.com/) Service for [FeatherJS](https://github.com/feathersjs).

## Installation

```bash
npm install feathers-mongoose-advanced --save
```

## Getting Started

A feathers-mongoose-advanced service differs from a standard feathers-mongoose model in a couple of ways:

### Full Access to Mongoose Schema Features

Because the Mongoose model must be created before setting up the service (as opposed to inline when using the standard feathers-mongoose service) you have complete control of Mongoose features, such as [virtual fields](http://mongoosejs.com/docs/guide.html) and [custom validators](http://mongoosejs.com/docs/validation.html).

### Optimized for Client-side Frameworks

To work better with client-side frameworks, such as [CanJS](www.canjs.com), out of the box, feathers-mongoose-advanced allows access to query options such as limit, skip, sort, and select by using `'$limit'`, `'$skip'`, `'$sort'`, and `'$select'` directly in the query object.



## Example Usage: Todos Service

Create a Mongoose model the same way that you normally would.  Here is an example todos service:

```js
// ./server/services/todos.js

var mongoose = require('mongoose'),
  ObjectId = mongoose.Schema.Types.ObjectId,
  MongooseService = require('feathers-mongoose-advanced');

// Set up the schema
var schema = new mongoose.Schema({
  description: String,
  done: Boolean,
  userID: ObjectId
});

// Setup the model.
var model = mongoose.model('Todo', schema);

// Provide access to the service.
module.exports = new MongooseService(model);
```

### Use the Service

The service is now ready to be used by your feathers app:

```js
// server.js
var feathers = require('feathers'),
  bodyParser = require('body-parser'),
  mongoose = require('mongoose');

// Connect to the MongoDB server.
mongoose.connect('mongodb://localhost/feathers-example');

// Create a feathers instance.
var app = feathers()
  // Setup the public folder.
  .use(feathers.static(__dirname + '/public'));
  // Enable Socket.io
  .configure(feathers.socketio()) 
  // Enable REST services
  .configure(feathers.rest()); 
  // Turn on JSON parser for REST services
  .use(bodyParser.json())
  // Turn on URL-encoded parser for REST services
  .use(bodyParser.urlencoded({extended: true}))

// Register services.
app.use('todos', require('./services/todos.js'));

// Start the server.
var port = 80;
app.listen(port, function() {
	console.log('Feathers server listening on port ' + port);
});
```

Now you can use the todos example from [feathersjs.com](http://feathersjs.com) and place it in the public directory.  Fire up the server and watch your todos persist in the database.

## Using with the `feathers-hooks` plugin

Here is an example service that uses `feathers-hooks`:

```js
// todos.js
var mongoose = require('mongoose'),
  ObjectId = mongoose.Schema.Types.ObjectId,
  MongooseService = require('feathers-mongoose-advanced'),
  hooks = require('../hook-library'),
  events = require('../event-library');

/**
 * A hook that logs to the console for debugging.
 * before or after
 *
 * find, get, create, update, delete
 */
var hooks = {
  log : function(hook, next){
    console.log(hook);
    return next();
  }
};

/* * * Create the schema * * */
var schema = new mongoose.Schema({
  description: String,
  done: Boolean,
  userID: ObjectId
});

module.exports = function(app){
  /* * * Set up the url for this service * * */
  var url = 'api/todos';

  app.use(url, new MongooseService( mongoose.model('Todo', schema) ));

  var service = app.service(url);

  /* * * Set up 'before-update' hook * * */
  service.before({
    update: hooks.log
  });

  /* * * Set up 'after-find' hook * * */
  service.after({
    find: hooks.log
  });


  /* Example socket filter that requires auth to publish messages. */
  var events = {
    requireAuth: function(data, params, callback){
      if (params.user && data.userID) {

        // Admins get notifications
        if (params.user.admin) {
          callback(null, data);
        }

        // User gets own notifications
        if (params.user._id == data.userID) {
          callback(null, data);
        }
      }
    }
  };

  /* * * Filter socket announcements * * */
  service.created = service.updated = service.patched = service.removed = events.requireAuth;

};

```

Then to use the service:

```js
// server.js
var feathers = require('feathers'),
  bodyParser = require('body-parser'),
  mongoose = require('mongoose');

// Connect to the MongoDB server.
mongoose.connect('mongodb://localhost/feathers-example');

// Create a feathers instance.
var app = feathers()
  // Setup the public folder.
  .use(feathers.static(__dirname + '/public'));
  // Enable Socket.io
  .configure(feathers.socketio()) 
  // Enable REST services
  .configure(feathers.rest()); 
  // Turn on JSON parser for REST services
  .use(bodyParser.json())
  // Turn on URL-encoded parser for REST services
  .use(bodyParser.urlencoded({extended: true}))

// Register services. Pass the app in to keep everything together.
require('./todos')(app);

// Start the server.
var port = 80;
app.listen(port, function() {
  console.log('Feathers server listening on port ' + port);
});
```


## API

`feathers-mongoose-advanced` services comply with the standard [FeathersJS API](http://feathersjs.com/api/#).

### Virtual Field for id
A virtual field will be set up on each service to automatically create an `id` out of each document's `_id`.  This means that all documents will contain both an `_id` and an `id` field.

This is added as a convenience for working with client-side frameworks that watch for an `id` to be present on model data. You can turn this off by passing {noVirtualID:true} in the second argument when creating the service:

```js
exports.service = new MongooseService(model, {noVirtualID:true});
```

### Special Query Params
The `find` queries allow the use of `$limit`, `$skip`, `$sort`, and `$select` in the query.  These special MongoDB features can be passed directly inside the query object:

```js
// Find all recipes that include salt, limit to 10, only include name field.
{"ingredients":"salt", "$limit":10, "$select":"name:1"} // JSON
GET /?ingredients=salt&%24limit=10&%24select=name%3A1 // HTTP
```

As a result of allowing these to be put directly into the query string, you won't want to use `$limit`, `$skip`, `$sort`, or `$select` as the name of fields in your document/schema.

### Working with Client-side Javascript Frameworks
Although it probably works well with most client-side frameworks, `mongoose-advanced-service` was built with CanJS in mind.  If you're making a CanJS app, consider using the [canjs-feathers plugin](https://github.com/feathersjs/canjs-feathers).


## Changelog
### 0.1.4
* Various bug fixes.
* Better examples in readme.md

### 0.1.0
* `$select` support in a query allows you to pick which fields to include or exclude in the results.
* A virtual field for `id` is automatically added to every service.
* Removed support for filters in favor of using [feathers-hooks](https://www.npmjs.com/package/feathers-hooks).

### 0.0.4
You no longer have to pass in an id on findOne.  If an id is present, the query will be executed as a findById().  All other params will be ignored.  If no id is present, the params.query object will be used in a findOne().

## License

[MIT](LICENSE)

## Author

[Marshall Thompson](https://github.com/marshallswain)

Based on [feathers-mongoose](https://github.com/feathersjs/feathers-mongoose) by [Glavin Wiechert](https://github.com/Glavin001)
