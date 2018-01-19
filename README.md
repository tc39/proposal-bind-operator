## ECMAScript This-Binding Syntax ##

This proposal introduces a new operator `::` which performs `this` binding and method extraction.

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

Using a jQuery-like library of virtual methods:

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

With the introduction of arrow functions in ECMAScript 6, the need for explicitly binding closures to the lexical `this` value has been dramatically reduced, resulting in a significant increase in language usability.  However, there are still two use cases where explicit `this` binding or injection is both common and awkward.

**Calling a known function with a supplied `this` argument:**

```js
let hasOwnProp = Object.prototype.hasOwnProperty;
let obj = { x: 100 };
hasOwnProp.call(obj, "x");
```

**Extracting a method from an object:**

```js
Promise.resolve(123).then(console.log.bind(console));
```

This proposal introduces a new operator `::` which can be used as syntactic sugar for these use cases.

In its binary form, the `::` operator creates a bound function such that the left hand side of the operator is bound as the `this` variable to the target function on the right hand side.

In its unary prefix form, the `::` operator creates a bound function such that the base of the supplied reference is bound as the `this` variable to the target function.

If the bound function is immediately called, and the target function is strict mode, then the bound function itself is not observable and the bind operator can be optimized to an efficient direct function call.

_NOTE: If the target function is not strict, then it may use `function.callee` or `arguments.callee`, which would result in an observable difference of behavior._

By providing syntactic sugar for these use cases we will enable a new class of "virtual method" library, which will have usability advantages over the standard adapter patterns in use today.

### Prototypes ###

- [Babel](https://github.com/babel/babel) - [REPL](https://babeljs.io/repl/)
- [esdown](https://github.com/zenparsing/esdown) - [REPL](http://zenparsing.github.io/esdown/repl/)

### Syntax ###

    LeftHandSideExpression[Yield] :
        NewExpression[?Yield]
        CallExpression[?Yield]
        BindExpression[?Yield]

    BindExpression[Yield] :
        LeftHandSideExpression[?Yield] :: [lookahead ≠ new] MemberExpression[?Yield]
        :: MemberExpression[?Yield]

    CallExpression[Yield] :
        MemberExpression[?Yield] Arguments[?Yield]
        super Arguments[?Yield]
        CallExpression[?Yield] Arguments[?Yield]
        CallExpression[?Yield] [ Expression[In, ?Yield] ]
        CallExpression[?Yield] . IdentifierName
        CallExpression[?Yield] TemplateLiteral[?Yield]
        BindExpression[?Yield] Arguments[?Yield]

### Early Errors ###

    BindExpression :
        :: MemberExpression

- It is a Syntax Error if the derived *MemberExpression* is not *MemberExpression : MemberExpression . Identifier*, *MemberExpression : MemberExpression [ Expression ]*, or *SuperProperty*

- It is a Syntax Error if the derived *NewExpression* is *PrimaryExpression : CoverParenthesizedExpressionAndArrowParameterList* and *CoverParenthesizedExpressionAndArrowParameterList* ultimately derives a phrase that, if used in place of *NewExpression*, would produce a Syntax Error according to these rules. This rule is recursively applied.

NOTE:  The last rule means that expressions such as

    ::(((foo)))

produce early errors because of recursive application of the first rule.

### Abstract Operation: InitializeBoundFunctionProperties ( F, target ) ###

The abstract operation InitializeBoundFunctionProperties with arguments _F_ and _target_ is used to set the "length" and "name" properties of a bound function _F_. It performs the following steps:

- Let _targetHasLength_ be HasOwnProperty(_target_, `"length"`).
- If _targetHasLength_ is `true`, then
    - Let _targetLen_ be ? Get(_target_, `"length"`).
    - If Type(_targetLen_) is not `Number`, then let _L_ be `0`.
    - Else, let _L_ be ToInteger(_targetLen_).
- Else let _L_ be `0`.
- Let _status_ be ? DefinePropertyOrThrow(_F_, `"length"`, PropertyDescriptor {[[Value]]: _L_, [[Writable]]: `false`, [[Enumerable]]: `false`, [[Configurable]]: `true`}).
- Let _targetName_ be ? Get(_target_, `"name"`).
- If Type(_targetName_) is not `String`, then let _targetName_ be the empty string.
- Let _status_ be ? SetFunctionName(_F_, _targetName_, `"bound"`).
- Return _F_.

### Runtime Semantics ###

    BindExpression :
        LeftHandSideExpression :: [lookahead ≠ new] MemberExpression

- Let _baseReference_ be the result of evaluating _LeftHandSideExpression_.
- Let _baseValue_ be GetValue(_baseReference_).
- Let _targetReference_ be the result of evaluating _MemberExpression_.
- Let _target_ be GetValue(_targetReference_).
- If IsCallable(_target_) is `false`, throw a `TypeError` exception.
- Let _F_ be ? BoundFunctionCreate(_target_, _baseValue_, «»).
- Return InitializeBoundFunctionProperties(_F_, _target_).

----

    BindExpression :
        :: MemberExpression

- Let _targetReference_ be the result of evaluating _MemberExpression_.
- Assert: IsPropertyReference(_targetReference_) is `true`.
- Let _thisValue_ be GetThisValue(_targetReference_).
- Let _target_ be GetValue(_targetReference_).
- If IsCallable(_target_) is `false`, throw a `TypeError` exception.
- Let _F_ be ? BoundFunctionCreate(_target_, _thisValue_, «»).
- Return ? InitializeBoundFunctionProperties(_F_, _target_).

### Future Extensions:  Bound Constructors ###

This syntax can be extended by introducing *bound constructors*.  When the binary `::` operator is followed by the `new` keyword, the constructor on the left hand side is wrapped by a callable function.

```js
class User {
    constructor(name) {
        this.name = name;
    }
}

let users = ["userA", "userB"].map(User::new);

console.log(users);

/*
[ (User) { name: "userA" },
  (User) { name: "userB" }]
*/
```
#### Runtime Semantics ####

    BindExpression:
        LeftHandSideExpression :: new

- Let _targetReference_ be the result of evaluating _LeftHandSideExpression_.
- Let _target_ be GetValue(_targetReference_).
- If IsConstructor(_target_) is `false`, throw a `TypeError` exception.
- Let _F_ be a new built-in function as defined in Bound Constructor Wrapper Functions.
- Set the [[BoundTargetConstructor]] internal slot of _F_ to _target_.
- Return ? InitializeBoundFunctionProperties(_F_, _target_).

#### Bound Constructor Wrapper Functions ####

A bound constructor wrapper function is an anonymous built-in function that has a [[BoundTargetConstructor]] internal slot.

When a bound constructor wrapper function _F_ is called with zero or more _args_, it performs the following steps:

- Assert: _F_ has a [[BoundTargetConstructor]] internal slot.
- Let _target_ be the value of _F_'s [[BoundTargetConstructor]] internal slot.
- Assert: IsConstructor(_target_).
- Let _args_ be a List consisting of all the arguments passed to this function.
- Return ? Construct(_target_, _args_).
