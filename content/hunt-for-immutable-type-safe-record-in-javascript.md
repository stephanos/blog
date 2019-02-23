+++
author = "Stephan Behnke"
categories = ["javascript", "flow", "babel"]
date = 2016-03-06T15:29:39Z
description = "javascript immutablejs babel"
draft = false
slug = "hunt-for-immutable-type-safe-record-in-javascript"
tags = ["javascript", "flow", "babel"]
title = "The hunt for an immutable, type safe data record in JavaScript"

+++

Ever since working with Scala's `case classes` I was hooked on the idea of having a type safe data record that was also immutable. What's not to like? It's type safe _and_ immutable (duh). So I wanted to see if I can get the same thing in JavaScript - the most mutable and dynamic language known to man.

```javascript
class Person {
  givenName;
  familyName;
}
```

This will serve as our starting point: a simple class in JavaScript. It contains two class instance fields according to the [ES Class Fields & Static Properties ES7](https://github.com/jeffmo/es-class-fields-and-static-properties) proposal. Also, it is mutable. After an instance of `Person` is created, its fields can be changed.

## immutable.js

Facebook's JavaScript library [immutable](http://facebook.github.io/immutable-js/) offers an immutable `Record` data type. Let's see how far it takes us.

```javascript
const Person = Record({ givenName: '', familyName: '' });
```

That's how we define our record. If you ask me, it seems strange to be required to provide default values in order to define the record structure. It mixes two concerns: structure definition and default values. Consequently, we can't require a property to be provided; there is always a default value.

Anyway, it's nice that we can access each property directly and are not forced to use something like `.get()`.

```javascript
const dad = new Person({ givenName: 'Homer', familyName: 'Simpson' });
dad.givenName // 'Homer'
```

What is very surprising is that there is no runtime error when constructing a record with any additional properties.

```javascript
const dad = new Person({ givenName: 'Homer', age: 40 });
dad.age // undefined
```

I find this quite dangerous. Maybe we can make it less problematic by adding type safety?

> **Type safety in JavaScript?** There are currently two serious efforts to bring type safety to JavaScript: [TypeScript](http://www.typescriptlang.org) and [Flow](http://flowtype.org). From what I've seen so far, the syntax of them is almost identical.

I've picked Flow for this since it seemed easier to get started with and also works well with Babel. TypeScript might work as well though.

There is an [pending issue](https://github.com/facebook/immutable-js/issues/203) to have proper Flow type definitions for `immutable` but using its current state already works quite well.

```javascript
const dad = new Person({ givenName: 42 });
// type error: number is incompatible with string

const dad = new Person({ givenName: 'Homer', age: 42 });
// type error: property `age` not found
```

That's quite nice! We get checks for wrong data types and unknown initialisation properties.

Unfortunately, the property access doesn't seem to get checked properly.

```javascript
dad.age
// type checks OK, returns undefined
```

All in all, we can get quite far with `Record`. But it's not perfect.


## Babel

It seems we can't reach our goal by writing code, we need to transform code. Since JavaScript doesn't have macros to do that, we take the next best thing: a Babel plugin.

> **What's Babel?**
[Babel](https://babeljs.io) was born as a transpiler that takes modern JavaScript code and emits code that can run on older platforms where some of the latest feature weren't supported yet. But it has since grown to become a more general code transformation and code generation platform.

The plugin needs to know which class to make immutable and which to ignore. So we create a new decorator to mark the class for transformation:

```javascript
@Record()
class Person {
  givenName;
  familyName;
}
```

Our Babel plugin will look for `@Record` and transform the code into this:

```javascript
@Record()
class Person {
  constructor(init) {
    this.__givenName = init.givenName;
    this.__familyName = init.familyName;
  }

  __givenName;
  __familyName;

  get givenName() {
    return this.__givenName;
  }

  get familyName() {
    return this.__familyName;
  }
}
```

Here's what happened:

 - the fields were renamed to be 'private'
 - getters were created to only allow read access
 - a constructor was added to initialise the fields

This is how we could use it:

```javascript
const dad = new Person({ givenName: 'Homer', familyName: 'Simpson' });
console.log(`created dad ${dad.givenName} ${dad.familyName}`);
// will output "created dad Homer Simpson"
```

But it's not terribly useful, yet. How do we change it? By creating a copy! Since this can be quite cumbersome and error-prone for larger objects, we generate a method to help us with that:

```javascript
@Record()
class Person {
  // ...

  update(update) {
    return new Person({
      givenName: update.givenName || this.__givenName,
      familyName: update.familyName || this.__familyName
    });
  }
}
```

`update` gets an object and creates a new `Person` by using the provided data or falls back to the existing data, if none is provided.

Now we can easily create a copy of our record:

```javascript
const son = dad.update({ givenName: 'Bart' });
console.log(`created son ${son.givenName} ${son.familyName}`);
// will output "created son Bart Simpson"
```

The next step is to make it type safe. Basically, our original code simply receives type annotations for its fields. That's it.

```javascript
@Record()
class Person {
  givenName: string;
  familyName: string;
}
```

The plugin now mostly just copies the new type annotations to the right places but also creates two new types:

```javascript
@Record()
class Person {
  constructor(init: PersonInit) {
    this.__givenName = init.givenName;
    this.__familyName = init.familyName;
  }

  __givenName: string;
  __familyName: string;

  get givenName(): string {
    return this.__givenName;
  }

  get familyName(): string {
    return this.__familyName;
  }

  update(update: PersonUpdate): Person {
    return new Person({
      givenName: update.givenName || this.__givenName,
      familyName: update.familyName || this.__familyName
    });
  }
}

type PersonInit = { 
  givenName: string;
  familyName: string;
};

type PersonUpdate = { 
  givenName?: string;
  familyName?: string;
};
```

The first one, `PersonInit`, defines the type of the object used to initialise the record. The second one, `PersonUpdate`, defines the type of the object used to create a copy of the record. It is important to notice that it contains optional properties (marked by `?` at the end). This means that the client doesn't have to specify any of them.

**We now a have a type safe, immutable record.**

```javascript
new Person({ givenName: 'Homer', familyName: 'Simpson' });
// OK

new Person({ givenName: 'Homer' });
// type error: property `familyName` not found

new Person({ givenName: 'Homer', familyName: true });
// type error: boolean is incompatible with string

new Person({ givenName: 'Homer', familyName: 'Simpson' }).update({});
// OK
```

Unfortunately, the type checker does _not_ fail when providing unknown properties during initialisation or update. This means we could easily have a typo for an optional field and wonder why nothing happens.

```javascript
const daughter = dad.update({givnam: 'Lisa'};
// OK
```

Apparently, this is a known limitation in Flow. The [workaround](https://github.com/facebook/flow/issues/1164) is to _seal_ the object type by adding a 'catch-all property' with a `void` type.

```javascript
type PersonInit = { 
  givenName: string;
  familyName: string;
  [key: string]: void;
};

type PersonUpdate = { 
  givenName?: string;
  familyName?: string;
  [key: string]: void;
};
```

Now when we use an invalid property key, it fails.

```javascript
const daughter = dad.update({givnam: 'Lisa'};
// type error: string is incompatible with undefined
```

This admittedly slightly cryptic error message now tells us that `givnam` is not part of the update data type.

*By the way, to make this type check in Flow the configuration file `.flowconfig` needs the flag `unsafe.enable_getters_and_setters=true` to process the getters.*

---

The source code for this code generator can be found on [GitHub]( https://github.com/stephanos/babel-plugin-immutable-record)