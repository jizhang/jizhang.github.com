---
title: Why Use Lodash When ES6 Is Available
tags:
  - frontend
  - javascript
  - es6
  - lodash
categories: Programming
date: 2017-03-13 22:39:01
---


[Lodash](https://lodash.com/) is a well-known JavaScript utility library that makes it easy to manipulate arrays and objects, as well as functions, strings, etc. I myself enjoys its functional way to process collections, especially chaining and lazy evaluation. But as [ECMAScript 2015 Standard][1] (ES6) becomes widely supported by major browsers, and [Babel](https://babeljs.io/), the JavaScript compiler that transforms ES6 codes to ES5, plays a major role in today's frontend development, it seems that most Lodash utilities can be replaced by ES6. But should we? In my opinion, Lodash will remain popular, for it still has lots of useful features that could improve the way of programming.

## `_.map` and `Array#map` Are Different

`_.map`, `_.reduce`, `_.filter` and `_.forEach` are frequently used functions when processing collections, and ES6 provides direct support for them:

```js
_.map([1, 2, 3], (i) => i + 1)
_.reduce([1, 2, 3], (sum, i) => sum + i, 0)
_.filter([1, 2, 3], (i) => i > 1)
_.forEach([1, 2, 3], (i) => { console.log(i) })

// becomes
[1, 2, 3].map((i) => i + 1)
[1, 2, 3].reduce((sum, i) => sum + i, 0)
[1, 2, 3].filter((i) => i > 1)
[1, 2, 3].forEach((i) => { console.log(i) })
```

But Lodash's `_.map` is more powerful, in that it works on objects, has iteratee / predicate shorthands, lazy evaluation, guards against null parameter, and has better performance.

<!-- more -->

### Iterate over Objects

To iterate over an object in ES6, there're several approaches:

```js
for (let key in obj) { console.log(obj[key]) }
for (let key in Object.keys(obj)) { console.log(obj[key]) }
Object.keys(obj).forEach((key) => { console.log(obj[key]) })
```

With Lodash, there's a unified `_.forEach`, for both array and object:

```js
_.forEach((value, key) => { console.log(value) })
```

Although ES6 does [provide][2] `forEach` for the newly added `Map` type, it takes some effort to first convert an object into a `Map`:

```js
// http://stackoverflow.com/a/36644532/1030720
const buildMap = o => Object.keys(o).reduce((m, k) => m.set(k, o[k]), new Map());
```

### Iteratee / Predicate Shorthands

To extract some property from an array of objects:

```js
let arr = [{ n: 1 }, { n: 2 }]
// ES6
arr.map((obj) => obj.n)
// Lodash
_.map(arr, 'n')
```

This can be more helpful when it comes to complex objects:

```js
let arr = [
  { a: [ { n: 1 } ]},
  { b: [ { n: 1 } ]}
]
// ES6
arr.map((obj) => obj.a[0].n) // TypeError: property 'a' is not defined in arr[1]
// Lodash
_.map(arr, 'a[0].n') // => [1, undefined]
```

As we can see, Lodash not only provides conveniet shorthands, it also guards against undefined values. For `_.filter`, there's also predicate shorthand. Here are some examples from Lodash documentation:

```js
let users = [
  { 'user': 'barney', 'age': 36, 'active': true },
  { 'user': 'fred',   'age': 40, 'active': false }
];
// ES6
users.filter((o) => o.active)
// Lodash
_.filter(users, 'active')
_.filter(users, ['active', true])
_.filter(users, {'active': true, 'age': 36})
```

### Chain and Lazy Evaluation

Here comes the fun part. Processing collections with chaining, lazy evaluation, along with short, easy-to-test functions, is quite popular these days. Most Lodash functions regarding collections can be chained easily. The following is a wordcount example:

```js
let lines = `
an apple orange the grape
banana an apple melon
an orange banana apple
`.split('\n')

_.chain(lines)
  .flatMap(line => line.split(/\s+/))
  .filter(word => word.length > 3)
  .groupBy(_.identity)
  .mapValues(_.size)
  .forEach((count, word) => { console.log(word, count) })

// apple 3
// orange 2
// grape 1
// banana 2
// melon 1
```

## Destructuring, Spread and Arrow Function

ES6 introduces some useful syntaxes like destructuring, spread and arrow function, which can be used to replace a lot of Lodash functions. For instance:

```js
// Lodash
_.head([1, 2, 3]) // => 1
_.tail([1, 2, 3]) // => [2, 3]
// ES6 destructuring syntax
const [head, ...tail] = [1, 2, 3]

// Lodash
let say = _.rest((who, fruits) => who + ' likes ' + fruits.join(','))
say('Jerry', 'apple', 'grape')
// ES6 spread syntax
say = (who, ...fruits) => who + ' likes ' + fruits.join(',')
say('Mary', 'banana', 'orange')

// Lodash
_.constant(1)() // => 1
_.identity(2) // => 2
// ES6
(x => (() => x))(1)() // => 1
(x => x)(2) // => 2

// Partial application
let add = (a, b) => a + b
// Lodash
let add1 = _.partial(add, 1)
// ES6
add1 = b => add(1, b)

// Curry
// Lodash
let curriedAdd = _.curry(add)
let add1 = curriedAdd(1)
// ES6
curriedAdd = a => b => a + b
add1 = curriedAdd(1)
```

For collection related operations, I prefer Lodash functions for they are more concise and can be chained; for functions that can be rewritten by arrow function, Lodash still seems more simple and clear. And according to some arguments in the [references](#References), the currying, [operators][3] and [fp style][4] from Lodash are far more useful in scenarios like function composition.

## Conclusion

Lodash adds great power to JavaScript language. One can write concise and efficient codes with minor efforts. Besides, Lodash is fully [modularized][5]. Though some of its functions will eventually deprecate, but I believe it'll still bring many benifits to developers, while pushing the development of JS language as well.

## References

* [10 Lodash Features You Can Replace with ES6](https://www.sitepoint.com/lodash-features-replace-es6/)
* [Does ES6 Mean The End Of Underscore / Lodash?](https://derickbailey.com/2016/09/12/does-es6-mean-the-end-of-underscore-lodash/)
* [Why should I use lodash - reddit](https://www.reddit.com/r/javascript/comments/41fq2s/why_should_i_use_lodash_or_rather_what_lodash/)

[1]: http://www.ecma-international.org/ecma-262/6.0/
[2]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map
[3]: https://lodash.com/docs/#add
[4]: https://github.com/lodash/lodash/wiki/FP-Guide
[5]: https://lodash.com/custom-builds
