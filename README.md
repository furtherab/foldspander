# foldspander

Foldspander is a JavaScript utility to fold complex objects into simpler ones fit for serialization, and then expand them back into their originals.

Perfect for seamless object sharing between the client-side and the server-side!

[travis]: https://travis-ci.org/tedeh/foldspander
[travis-img]: https://travis-ci.org/tedeh/foldspander.png?branch=master

**Build status:** [![Build Status][travis-img]][travis] 

## Table of contents

- [Installation](#installation)
- [Usage](#usage)
  - [Native objects](#native-objects)
  - [Replacer and revivers](#replacer-and-revivers)
  - [Helpers and custom objects](#helpers-and-custom-objects)
    - [addClass](#addclass)
  - [Nested objects](#nested-objects)
- [Tests](#tests)
- [TODO](#todo)
- [Performance](#performance)
- [Contributing](#contributing)

## Installation

Install the lastest version of `foldspander` from [npm](https://www.npmjs.com/) by executing `npm install foldspander` in your shell. 

## Usage

Usage is simple and consists of creating an instance of foldspander and assigning different types to it.

Basic example showcasing many of the features in [examples/showcase.js](examples/showcase.js):

```javascript
var Foldspander = require(__dirname + '/../');
var should = require('should');

// Basic Class
function Fraction(numerator, denominator) {
  this.numerator = numerator;
  this.denominator = denominator;
};

var foldspand = new Foldspander({

  // option that indicates that we want to have automatic support for some native Javascript types
  native: true
  
});

// add Fraction as a foldspander type
foldspand.add(

  // handle that should be unique
  'fraction',

  // determines if an encountered object should be folded with this routine (basic instanceof matcher)
  function(obj) { return obj instanceof Fraction; },
  
  // folds the object (breaks it into a basic JSON type)
  function(frac) { return {numerator: frac.numerator, denominator: frac.denominator}; },
  
  // expands the object (instantiates the complex type)
  function(obj) { return new Fraction(obj.numerator, obj.denominator); }
  
);

var objs = [
  {id: 1, value: new Fraction(1,3), created_at: new Date('2015-03-23')},
  {id: 2, value: new Fraction(1,7), created_at: new Date('2015-03-24')}
];

// get a string of JSON from the list of objects
var str = foldspand.stringify(objs);

// parse the string and instantiate the contained objects
var exp = foldspand.parse(str);

// objs are equal to exp
exp.should.eql(objs);

// more verbose example to show that complex objects have indeed been instantiated
exp[0].value.should.be.instanceof(Fraction);
exp[1].created_at.should.be.instanceof(Date);
```

### Native objects

Foldspander provides support for dealing with some native Javascript objects. A Foldspander instance can be prepared to accept these types by passing the option `native` to the constuctor, or using the instance method `native`.

The currently supported objects are:

- `Date`
- `RegExp`
- `NaN`
- `Infinity`

With more to come!

### Replacer and Revivers

`JSON.stringify` and `JSON.parse` take optional arguments named `replacer` and `reviver` respectively. An instance of Foldspander provides hooks for these arguments that will apply the rules of the instance. Example in [examples/revive_and_replace.js](examples/revive_and_replace.js):

```javascript
var Foldspander = require(__dirname + '/../');
var should = require('should');

var foldspander = new Foldspander({native: true});
var obj = {id: 123, created_at: new Date()};

var str = JSON.stringify(obj, foldspander.getReplacer());
var exp = JSON.parse(str, foldspander.getReviver());

exp.should.eql(obj);
```

As a convenience, a Foldspander instance also provides a wrapper around `JSON.stringify` and `JSON.parse`. Example in [examples/stringify_and_parse.js](examples/stringify_and_parse.js):

```javascript
var Foldspander = require(__dirname + '/../');
var should = require('should');

var foldspander = new Foldspander({native: true});
var obj = {id: 123, created_at: new Date()};

var str = foldspander.stringify(obj);
var exp = foldspander.parse(str);

exp.should.eql(obj);
```

### Helpers and custom objects

It is simple to add your own custom objects. See [examples/showcase.js](examples/showcase.js)or look into the [Usage](#usage) section of this document.

In addition to the basic `add` method on a Foldspander instance, some helpers are provided.

#### addClass

`addClass` is a simple helper for basic types. It requires only a (named!) function (dubbed "class") as its first argument to work. It will add an object to a foldspander instance with the following properties:

- `name` - Name of the class
- `matcher` - Matches if an object is an instanceof the class
- `fold` - Picks every enumerable property of the object
- `expand` - Makes an instance of class and then copies every folded property into the instance

It works good for simple classes. Example in [examples/add_class.js](examples/add_class.js):

```javascript
var Foldspander = require(__dirname + '/../');
var should = require('should');

function Fraction(numerator, denominator) {
  this.numerator = numerator;
  this.denominator = denominator;
};

var foldspander = new Foldspander();

foldspander.addClass(Fraction);

var obj = new Fraction(1,3);

// 1) Every enumerable property was picked of obj
var fld = foldspander.fold(obj);

// 2) Fraction was instantiated with no arguments to the constructor
// 3) Every property picked in 1) was copied into instance from 2)
var exp = foldspander.expand(fld);

exp.should.eql(obj).and.not.equal(obj);
```

### Nested objects

Nested objects may be serialized as well. The object walker will descend again into a folded object. Example involving a Collection of Model objects in [examples/nested_objects.js](examples/nested_objects.js):

```javascript
var Foldspander = require(__dirname + '/../');
var should = require('should');

// a model class (loosely based on Backbone.Model)
function Model(attrs) {
  this.attributes = attrs || {};
}

// a collection of models class (loosely based on Backbone.Collection)
function Collection(models) {
  this.models = models || [];
}

var foldspander = new Foldspander({
  native: true
});

foldspander.add('collection', Foldspander.matchers.instanceof(Collection),
  function(collection) { return {models: collection.models}; },
  function(obj) { return new Collection(obj.models); }
);

foldspander.add('model', Foldspander.matchers.instanceof(Model),
  function(model) { return {attributes: model.attributes}; },
  function(obj) { return new Model(obj.attributes); }
);

var collection = new Collection([
  new Model({id: 1, created_at: new Date()}),
  new Model({id: 2, created_at: new Date()})
]);

var fld = foldspander.fold(collection);
var exp = foldspander.expand(fld);

// equal === strict equality (actually created a new object)
exp.should.eql(collection).and.not.equal(collection);

// verbose example to show that models are actually created
exp.models[0].should.be.eql(collection.models[0]).and.not.equal(collection.models[0]);
```

[backbone]: http://backbonejs.org/

A similar setup might be used when folding and expanding [Backbone][backbone] objects.

## Tests 

To run the [mocha.js](http://mochajs.org/) tests, go the the project folder and run:

```shell
npm install --dev
node_modules/.bin/mocha
```

## TODO

- Support more native types like WeakMap etc.
- Client versions
- Async expansion/folding

## Performance

## Contributing

Highlighting [issues](https://github.com/tedeh/foldspander/issues) or submitting pull
requests on [Github](https://github.com/tedeh/foldspander) is most welcome.
