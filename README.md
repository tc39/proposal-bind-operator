## ECMAScript Function Bind Syntax ##

This proposal introduces a new operator `::` which performs function binding and
method extraction.

It is a more detailed description of the [bind operator strawman](http://wiki.ecmascript.org/doku.php?id=strawman:bind_operator).

### Examples ###

Using an iterator library implemented as a module of "virtual methods":

```js
import { map, takeWhile, forEach } from "iterlib";

getPlayers()
::map(x => x.character())
::takeWhile(x => x.strength > 100)
::forEach(x => console.log(x));
```

Using a jquery-like library of virtual methods:

```js
// Create bindings for just the methods that we need
let { find, html } = jake;

// Find all the divs with class="myClass", then get all of the "p"s and
// replace their content.
document.querySelectorAll("div.myClass")::find("p")::html("hahaha");
```

Using method extraction to print the eventual value of a promise to the console:

```js
Promise.resolve(123).then(::console.log);
```

Using method extraction to call an object method when a DOM event occurs:

```js
$(".some-link").on("click", ::view.reset);
```

### Motivation and Overview ###

With the introduction of arrow functions in ECMAScript 6, the need for explicitly
binding closures to the lexical `this` value has been dramatically reduced, resulting
in a significant increase in language usability.  However, there are still two use cases
where explicit `this` binding or injection is both common and awkward.

**Calling a known function with a supplied `this` argument:**

```js
function getX() { return this.x }
var obj = { x: 100 };
getX.call(obj);
```

**Extracting a method from an object:**

```js
Promise.resolve(123).then(console.log.bind(console));
```

This proposal introduces a new operator `::` which can be used as syntactic sugar
for these use cases.

In its binary form, the `::` operator creates a bound function such that the left
side of the operator is bound as the `this` variable to the target function on
the right hand side.

In its unary prefix form, the `::` operator creates a bound function such that
the base of the supplied reference is bound as the `this` variable to the target
function.

If the bound function is immediately called, then the bound function itself is not
observable and can be optimized to a simple and efficient function call.

By providing syntactic sugar for these use cases we will enable a new class of
"virtual method" libraries, which will have usability advantages over the standard
adapter patterns in use today.


### Syntax ###


    CallExpression[Yield] :
        MemberExpression[?Yield] Arguments[?Yield]
        super Arguments[?Yield]
        CallExpression[?Yield] Arguments[?Yield]
        CallExpression[?Yield] [ Expression[In, ?Yield] ]
        CallExpression[?Yield] . IdentifierName
        CallExpression[?Yield] TemplateLiteral[?Yield]
        CallExpression[?Yield] :: MemberExpression[?Yield]
        :: MemberExpression[?Yield]


### Runtime Semantics ###

    CallExpression :
        CallExpression :: MemberExpression

- Let `baseReference` be the result of evaluating `CallExpression`.
- Let `baseValue` be GetValue(`baseReference`).
- ReturnIfAbrupt(`baseValue`).
- Let `bv` be RequireObjectCoercible(`baseValue`).
- ReturnIfAbrupt(`bv`).
- Let `targetReference` be the result of evaluating `MemberExpression`
- Let `target` be GetValue(`targetReference`).
- If IsCallable(`target`) is false, throw a TypeError exception.
- Let `F` be BoundFunctionCreate(`target`, `baseValue`, ()).
- Let `targetHasLength` be HasOwnProperty(`target`, "length").
- ReturnIfAbrupt(`targetHasLength`).
- If `targetHasLength` is true, then
    - Let `targetLen` be Get(`target`, "length").
    - ReturnIfAbrupt(`targetLen`).
    - If Type(`targetLen`) is not Number, then let `L` be 0.
    - Else, let `L` be ToInteger(`targetLen`).
- Else let `L` be 0.
- Let `status` be DefinePropertyOrThrow(`F`, "length", PropertyDescriptor {[[Value]]: `L`,
  [[Writable]]: false, [[Enumerable]]: false, [[Configurable]]: true}).
- ReturnIfAbrupt(`status`).
- Let `targetName` be Get(`target`, "name").
- ReturnIfAbrupt(`targetName`).
- If Type(`targetName`) is not String, then let `targetName` be the empty string.
- Let `status` be SetFunctionName(`F`, `targetName`, "bound").
- ReturnIfAbrupt(`status`).
- Return `F`.


----

    CallExpresion :
        :: MemberExpression

- Let `targetReference` be the result of evaluating `MemberExpression`.
- If Type(`targetReference`) is Reference, then
    - Let `baseValue` be GetBase(`targetReference`).
    - ReturnIfAbrupt(`baseValue`).
    - Let `bv` be RequireObjectCoercible(`baseValue`).
    - ReturnIfAbrupt(`bv`).
- Else let `baseValue` be undefined.
- Let `target` be GetValue(`targetReference`).
- If IsCallable(`target`) is false, throw a TypeError exception.
- Let `F` be BoundFunctionCreate(`target`, `baseValue`, ()).
- Let `targetHasLength` be HasOwnProperty(`target`, "length").
- ReturnIfAbrupt(`targetHasLength`).
- If `targetHasLength` is true, then
    - Let `targetLen` be Get(`target`, "length").
    - ReturnIfAbrupt(`targetLen`).
    - If Type(`targetLen`) is not Number, then let `L` be 0.
    - Else, let `L` be ToInteger(`targetLen`).
- Else let `L` be 0.
- Let `status` be DefinePropertyOrThrow(`F`, "length", PropertyDescriptor {[[Value]]: `L`,
  [[Writable]]: false, [[Enumerable]]: false, [[Configurable]]: true}).
- ReturnIfAbrupt(`status`).
- Let `targetName` be Get(`target`, "name").
- ReturnIfAbrupt(`targetName`).
- If Type(`targetName`) is not String, then let `targetName` be the empty string.
- Let `status` be SetFunctionName(`F`, `targetName`, "bound").
- ReturnIfAbrupt(`status`).
- Return `F`.
