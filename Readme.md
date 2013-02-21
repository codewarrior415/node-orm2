## Object Relational Mapping

[![Build Status](https://secure.travis-ci.org/dresende/node-orm2.png)](http://travis-ci.org/dresende/node-orm2)

## Install

```sh
npm install orm
```

Current stable version: **2.0.2**

## DBMS Support

- MySQL
- PostgreSQL
- Amazon Redshift
- SQLite

## Features

- Create Models, sync, drop, bulk create, get, find, remove, count
- Create Model associations, find, check, create and remove
- Define custom validations (several builtin validations, check instance properties before saving)
- Instance singleton (table rows fetched twice are the same object, changes to one change all)

## Introduction

This is a node.js object relational mapping module.

Here is an example on how to use it:

```js
var orm = require("orm");

orm.connect("mysql://username:password@host/database", function (err, db) {
	if (err) throw err;

	var Person = db.define("person", {
		name      : String,
		surname   : String,
		age       : Number,
		male      : Boolean,
		continent : [ "Europe", "America", "Asia", "Africa", "Australia", "Antartica" ], // ENUM type
		photo     : Buffer, // BLOB/BINARY
		data      : Object // JSON encoded
	}, {
		methods: {
			fullName: function () {
				return this.name + ' ' + this.surname;
			}
		},
		validations: {
			age: orm.validators.rangeNumber(18, undefined, "under-age")
		}
	});

	Person.find({ surname: "Doe" }, function (err, people) {
		// SQL: "SELECT * FROM person WHERE surname = 'Doe'"

		console.log("People found: %d", people.length);
		console.log("First person: %s, age %d", people[0].fullName(), people[0].age);

		people[0].age = 16;
		people[0].save(function (err) {
			// err.msg = "under-age";
		});
	});
});
```

## Settings

You have a global settings object and one for each connection.

```js
var orm = require("orm");

orm.settings.set("some.deep.value", 123);

orm.connect("....", function (err, db) {
    // db.settings is a snapshot of the settings at the moment
    // of orm.connect(). changes to it don't affect orm.settings

	console.log(db.settings.get("some.deep.value")); // 123
	console.log(db.settings.get("some.deep"));       // { value: 123 }
});
```

## Models

A Model is a structure binded to one or more tables, depending on the associations. The model name is assumed to be the table name. After defining a model you can use it to manipulate the table.

After defining a Model you can get a specific element or find one or more based on some conditions.

## Defining Models

To define a model, you use the reference to the database connection and call `define`. The function will define a Model
and will return it to you. You can get it later by it's id directly from the database connection so you don't actually
need to store a reference to it.

```js
var Person = db.define('person', {        // 'person' will be the table in the database as well as the model id
	// properties
	name    : String,                     // you can use native objects to define the property type
	surname : { type: "text", size: 50 }  // or you can be specific and define aditional options
}, {
	// options (optional)
});
```

## Loading Models

If you prefer to have your models defined in separated files, you can define them in a function inside a module and
export the function has the entire module. You can have cascading loads.

```js
// your main file (after connecting)
db.load("./models", function (err) {
    // loaded!
    var Person = db.models.person;
    var Pet    = db.models.pet;
});

// models.js
module.exports = function (db, cb) {
    db.load("./models-extra", function (err) {
        if (err) {
            return cb(err);
        }

        db.define('person', {
            name : String
        });

        return cb();
    });
};

// models-extra.js
module.exports = function (db, cb) {
    db.define('pet', {
        name : String
    });

    return cb();
};
```

## Synching Models

If you don't have the tables on the database you have to call the `.sync()` on every Model. This will just create the
tables necessary for your Model. If you have more than one Model you can call `.sync()` directly on the database
connection to syncronize all Models.

```js
// db.sync() can also be used
Person.sync(function (err) {
	!err && console.log("done!");
});
```

## Dropping Models

If you want to drop a Model and remove all tables you can use the `.drop()` method.

```js
Person.drop(function (err) {
	!err && console.log("person model no longer exists!");
});
```

## Advanced Options

Using [Settings](#settings) or directly on Model definition you can tweak some options.
For example, each Model instance has a unique ID in the database. This table column is
by default "id" but you can change it.

```js
var Person = db.define("person", {
	name : String
}, {
	id   : "person_id"
});

// or just do it globally..
db.settings.set("properties.primary_key", "UID");

// ..and then define your Models
var Pet = db.define("pet", {
	name : String
});
```

**Pet** model will have 2 columns, an `UID` and a `name`.

Other options:

- `cache` : (default: `true`) Set it to `false` to disable Instance cache (Singletons) or set a timeout value (in seconds);
- `autoSave` : (default: `false`) Set it to `true` to save an Instance right after changing any property;
- `autoFetch` : (default: `false`) Set it to `true` to fetch associations when fetching an instance from the database;
- `autoFetchLimit` : (default: `1`) If `autoFetch` is enabled this defines how many hoops (associations of associations)
  you want it to automatically fetch.

## Hooks

If you want to listen for a type of event than occurs in instances of a Model, you can attach a function that
will be called when that event happens. There are some events possible:

- `afterLoad` : (no parameters) Right after loading and preparing an instance to be used;
- `beforeSave` : (no parameters) Right before trying to save;
- `afterSave` : (bool success) Right after saving;
- `beforeCreate` : (no parameters) Right before trying to save a new instance;
- `beforeRemove` : (no parameters) Right before trying to remove an instance.

All hook function are called with `this` as the instance so you can access anything you want related to it.

## Finding Items

### Model.get(id, [ options ], cb)

To get a specific element from the database use `Model.get`.

```js
Person.get(123, function (err, person) {
	// finds person with id = 123
});
```

### Model.find([ conditions ] [, options ] [, limit ] [, order ] [, cb ])

Finding one or more elements has more options, each one can be given in no specific parameter order. Only `options` has to be after `conditions` (even if it's an empty object).

```js
Person.find({ name: "John", surname: "Doe" }, 3, function (err, people) {
	// finds people with name='John' AND surname='Doe' and returns the first 3
});
```

If you need to sort the results because you're limiting or just because you want them sorted do:

```js
Person.find({ surname: "Doe" }, "name", function (err, people) {
	// finds people with surname='Doe' and returns sorted by name ascending
});
Person.find({ surname: "Doe" }, [ "name", "Z" ], function (err, people) {
	// finds people with surname='Doe' and returns sorted by name descending
	// ('Z' means DESC; 'A' means ASC - default)
});
```

There are more options that you can pass to find something. These options are passed in a second object:

```js
Person.find({ surname: "Doe" }, { offset: 2 }, function (err, people) {
	// finds people with surname='Doe', skips the first 2 and returns the others
});
```

### Model.count([ conditions, ] cb)

If you just want to count the number of items that match a condition you can just use `.count()` instead of finding all
of them and counting. This will actually tell the database server to do a count, the count is not done in javascript.

```js
Person.count({ surname: "Doe" }, function (err, count) {
	console.log("We have %d Does in our db", count);
});
```

### Model.exists([ conditions, ] cb)

Similar to `.count()`, this method just checks if the count is greater than zero or not.

```js
Person.exists({ surname: "Doe" }, function (err, exists) {
	console.log("We %s Does in our db", exists ? "have" : "don't have");
});
```

### Available options

- `offset`: discards the first `N` elements
- `limit`: although it can be passed as a direct argument, you can use it here if you prefer
- `only`: if you don't want all properties, you can give an array with the list of properties you want

#### Chaining

If you prefer another less complicated syntax you can chain `.find()` by not giving a callback parameter.

```js
Person.find({ surname: "Doe" }).limit(3).offset(2).only("name", "surname").run(function (err, people) {
    // finds people with surname='Doe', skips first 2 and limits to 3 elements,
    // returning only 'name' and 'surname' properties
});
```

You can also chain and just get the count in the end. In this case, offset, limit and order are ignored.

```js
Person.find({ surname: "Doe" }).count(function (err, people) {
    // people = number of people with surname="Doe"
});
```

Also available is the option to remove the selected items.

```js
Person.find({ surname: "Doe" }).remove(function (err) {
    // Does gone..
});
```

You can also make modifications to your instances using common Array traversal methods and save everything
in the end.

```js
Person.find({ surname: "Doe" }).each(function (person) {
	person.surname = "Dean";
}).save(function (err) {
	// done!
});

Person.find({ surname: "Doe" }).each().filter(function (person) {
	return person.age >= 18;
}).sort(function (person1, person2) {
	return person1.age < person2.age;
}).get(function (people) {
	// get all people with at least 18 years, sorted by age
});
```

Of course you could do this directly on `.find()`, but for some more complicated tasks this can be very usefull.

`Model.find()` does not return an Array so you can't just chain directly. To start chaining you have to call
`.each()` (with an optional callback if you want to traverse the list). You can then use the common functions
`.filter()`, `.sort()` and `.forEach()` more than once.

In the end (or during the process..) you can call:
- `.count()` if you just want to know how many items there are;
- `.get()` to retrieve the list;
- `.save()` to save all item changes.

#### Conditions

Conditions are defined as an object where every key is a property (table column). All keys are supposed
to be concatenated by the logical `AND`. Values are considered to match exactly, unless you're passing
an `Array`. In this case it is considered a list to compare the property with.

```js
{ col1: 123, col2: "foo" } // `col1` = 123 AND `col2` = 'foo'
{ col1: [ 1, 3, 5 ] } // `col1` IN (1, 3, 5)
```

If you need other comparisons, you have to use a special object created by some helper functions. Here are
a few examples to describe it:

```js
{ col1: orm.eq(123) } // `col1` = 123 (default)
{ col1: orm.ne(123) } // `col1` <> 123
{ col1: orm.gt(123) } // `col1` > 123
{ col1: orm.gte(123) } // `col1` >= 123
{ col1: orm.lt(123) } // `col1` < 123
{ col1: orm.lte(123) } // `col1` <= 123
{ col1: orm.between(123, 456) } // `col1` BETWEEN 123 AND 456
```

### Singleton

Each model instances is cached, so if you fetch the same record using 2 or more different queries, you will
get the same object. If you have other systems that can change your database (or you're developing and need
to make some manual changes) you should remove this feature by disabling cache. You do this when you're
defining each Model.

```js
var Person = db.define('person', {
	name    : String
}, {
	cache   : false
});
```

If you want singletons but want cache to expire after a period of time, you can pass a number instead of a
boolean. The number will be considered the cache timeout in seconds (you can use floating point).

## Associations

An association is a relation between one or more tables.

## hasOne vs. hasMany

Since this topic brings some confusion to many people including myself, here's a list of the possibilities
supported by both types of association.

- `hasOne` : it's a **Many-to-One** relationship. A.hasOne(B) means A will have one (or none) of B, but B can be
  associated with many A;
- `hasMany`: it's a **One-to-Many** relationship. A.hasMany(B) means A will have none, one or more of B. Actually
  B will be associated with possibly many A but you don't have how to find it easily (see next);
- `hasMany` + reverse: it's a **Many-to-Many** relationship. A.hasMany(B, { reverse: A }) means A can have none or
  many B and also B can have none or many A. Accessors will be created in both models so you can manage them from
  both sides.

If you have a relation of 1 to 0 or 1 to 1, you should use `hasOne` association. This assumes a column in the model that has the id of the other end of the relation.

```js
var Person = db.define('person', {
	name : String
});
var Animal = db.define('animal', {
	name : String
});
Animal.hasOne("owner", Person); // assumes column 'owner_id' in 'animal' table

// get animal with id = 123
Animal.get(123, function (err, Foo) {
	// Foo is the animal model instance, if found
	Foo.getOwner(function (err, John) {
		// if Foo animal has really an owner, John points to it
	});
});
```

If you prefer to use another name for the field (owner_id) you can change this parameter in the settings.

```js
db.settings.set("properties.association_key", "id_{name}"); // {name} will be replaced by 'owner' in this case
```

**Note: This has to be done prior to the association creation.**

For relations of 1 to many you have to use `hasMany` associations. This assumes another table that has 2 columns, one for each table in the association.

```js
var Person = db.define('person', {
	name : String
});
Person.hasMany("friends"); // omitting the other Model, it will assume self model

Person.get(123, function (err, John) {
	John.getFriends(function (err, friends) {
		// assumes table person_friends with columns person_id and friends_id
	});
});
```

The `hasMany` associations can have additional properties that are assumed to be in the association table.

```js
var Person = db.define('person', {
	name : String
});
Person.hasMany("friends", {
    rate : Number
});

Person.get(123, function (err, John) {
	John.getFriends(function (err, friends) {
		// assumes rate is another column on table person_friends
		// you can access it by going to friends[N].extra.rate
	});
});
```

If you prefer you can activate `autoFetch`. This way associations are automatically fetched when you get or find instances of a model.

```js
var Person = db.define('person', {
	name : String
});
Person.hasMany("friends", {
    rate : Number
}, {
    autoFetch : true
});

Person.get(123, function (err, John) {
    // no need to do John.getFriends() , John already has John.friends Array
});
```

You can also define this option globally instead of a per association basis.

```js
var Person = db.define('person', {
	name : String
}, {
    autoFetch : true
});
Person.hasMany("friends", {
    rate : Number
});
```

Associations can make calls to the associated Model by using the `reverse` option. For example, if you have an
association from ModelA to ModelB, you can create an accessor in ModelB to get instances from ModelA.
Confusing? Look at the next example.

```js
var Pet = db.define('pet', {
	name : String
});
var Person = db.define('person', {
	name : String
});
Pet.hasOne("owner", Person, {
	reverse : "pets"
});

Person(4).getPets(function (err, pets) {
	// although the association was made on Pet,
	// Person will have an accessor (getPets)
	//
	// In this example, ORM will fetch all pets
	// whose owner_id = 4
});
```

This makes even more sense when having `hasMany` associations since you can manage the Many-to-Many associations
from both sides.


```js
var Pet = db.define('pet', {
	name : String
});
var Person = db.define('person', {
	name : String
});
Person.hasMany("pets", Person, {
    bought  : Date
}, {
	reverse : "owners"
});

Person(1).getPets(...);
Pet(2).getOwners(...);
```
