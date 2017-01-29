---
title: Difference Between Lodash _.assign and _.assignIn
tags:
  - lodash
  - javascript
  - frontend
categories:
  - Programming
date: 2017-01-29 14:18:29
---


In Lodash, both `_.assign` and `_.assignIn` are ways to copy source objects' properties into target object. According the [documentation][1], `_.assign` processes **own enumerable string keyed properties**, while `_.assignIn` processes both **own and inherited source properties**. There're also other companion functions like `_.forOwn` and `_.forIn`, `_.has` and `_.hasIn`. So what's the difference between them?

In brief, the `In` in latter methods implies the way `for...in` loop behaves, which [iterates all enumerable properties of the object itself and those the object inherits from its constructor's prototype][2]. JavaScript has an inheritance mechanism called prototype chain. When iterating an object's properties with `for...in` or `_.forIn`, all properties appeared in the object and its prototype are processed, until the prototype resolves to `null`. Here's the example code taken from Lodash's doc:

```javascript
function Foo() { this.a = 1; }
Foo.prototype.b = 2;
function Bar() { this.c = 3; }
Bar.prototype.d = 4;
_.assign({a: 0}, new Foo, new Bar); // => {a: 1, c: 3}
_.assignIn({a: 0}, new Foo, new Bar); // => {a:1, b:2, c:3, d:4}
```

<!-- more -->

## How `_.assign` Picks Properties

Let's dissect the phrase "own enumerable string-keys properties" into three parts. 

### Own Property

JavaScript is a prototype-based language, but there're several ways to simulate class and instance, like object literal, function prototype, `Object.create`, and the newly added `class` keyword. In either case, we can use `Object.prototype.hasOwnProperty()` to determine if the property is inherited or not.

```javascript
let foo = new Foo();
foo.hasOwnProperty('a'); // => true
Object.prototype.hasOwnProperty.call(foo, 'b'); // => false
```

`Object.getOwnPropertyNames()` and `Object.keys()` can retrieve all properties defined directly in the object, except that `Object.keys()` only returns enumerable keys (see next section).

```javascript
let o1 = {a: 1};
let o2 = Object.create(o1);
o2.b = 2;
Object.getOwnPropertyNames(o2); // => ['b']
Object.keys(o2); // => ['b']
```

### Enumerable Property

Object property can be defined with either data descriptor or accessor descriptor. Among data descriptor options, the `enumerable` boolean indicates whether this property shows in `for...in` or `Object.keys()`. 

```javascript
let o = {};
Object.defineProperty(o, 'a', { enumerable: false, value: 1 });
Object.keys(o); // => []
o.propertyIsEnumerable('a'); // => false
```

You can refer to [Object.defineProperty()][3] for more information.

### String-keyed Property

Before ES6, object's keys are always String. ES6 introduces a new primitive type [Symbol][4], which can be used as a key for private property. Symbol property is non-enumerable.

```javascript
let s = Symbol();
let o = {};
o[s] = 1;
Object.keys(o); // => []
```

There's a nice [Detection Table][5] to help you figure out which built-in methods process enumerable or inherited properties.

## `_.assign` and `_.assignIn` Implementation

Both methods calls `_.keys` and `_.keysIn` respectively. `_.keys` calls `Object.keys()` and `_.keysIn` uses `for...in` loop. Actually `Object.keys()` is not difficult to implement. As mentioned above, `for...in` can be used to retrieve both own and inherited properties, while `hasOwnProperty` determines whether this property is defined in the object itself.

```javascript
function keys(object) {
  let result = [];
  for (let key in Object(object)) {
    if (Object.prototype.hasOwnProperty.call(object, key)) {
      result.push(key);
    }
  }
  return result;
}
```

`Object.assign()` does the same thing as `_.assign()`. Use Lodash if you need to run your code on older browsers.


## References

* [Object.assign() - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)
* [Inheritance and The Prototype Chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

[1]: https://lodash.com/docs/
[2]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...in
[3]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty
[4]: https://developer.mozilla.org/en-US/docs/Glossary/Symbol
[5]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Enumerability_and_ownership_of_properties#Detection_Table
