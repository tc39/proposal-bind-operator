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
var hasOwnProp = Object.prototype.hasOwnProperty;
var obj = { x: 100 };
hasOwnProp.call(obj, "x");
```

**Extracting a method from an object:**

```js
Promise.resolve(123).then(console.log.bind(console));
```

This proposal introduces a new operator `::` which can be used as syntactic sugar
for these use cases.

In its binary form, the `::` operator creates a bound function such that the left
hand side of the operator is bound as the `this` variable to the target function on
the right hand side.

In its unary prefix form, the `::` operator creates a bound function such that
the base of the supplied reference is bound as the `this` variable to the target
function.

If the bound function is immediately called, and the target function is strict
mode, then the bound function itself is not observable and the bind operator
can be optimized to an efficient direct function call.

_NOTE: If the target function is not strict, then it may use `function.callee` or
`arguments.callee`, which would result in an observable difference of behavior._

By providing syntactic sugar for these use cases we will enable a new class of
"virtual method" library, which will have usability advantages over the standard
adapter patterns in use today.


### Prototypes ###

- Parser: [esparse](https://github.com/zenparsing/esparse)
- Transpiler: [esdown](https://github.com/zenparsing/esdown)
- Demo: [esdown REPL](http://zenparsing.github.io/esdown/repl/)


### Syntax ###


    LeftHandSideExpression[Yield] :
        NewExpression[?Yield]
        CallExpression[?Yield]
        BindExpression[?Yield]

    BindExpression[Yield] :
        LeftHandSideExpression[?Yield] :: NewExpression[?Yield]
        :: NewExpression[?Yield]

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
        :: NewExpression

- It is a Syntax Error if the derived *NewExpression* is neither
  *MemberExpression : MemberExpression . Identifier* nor
  *MemberExpression : MemberExpression [ Expression ]*

- It is a Syntax Error if the derived *NewExpression* is
  *PrimaryExpression : CoverParenthesizedExpressionAndArrowParameterList* and
  *CoverParenthesizedExpressionAndArrowParameterList* ultimately derives a phrase that,
  if used in place of *NewExpression*, would produce a Syntax Error according to these rules.
  This rule is recursively applied.

NOTE:  The last rule means that expressions such as

    ::(((foo)))

produce early errors because of recursive application of the first rule.

### Runtime Semantics ###

    BindExpression :
        LeftHandSideExpression :: NewExpression

- Let _baseReference_ be the result of evaluating _LeftHandSideExpression_.
- Let _baseValue_ be GetValue(_baseReference_).
- ReturnIfAbrupt(_baseValue_).
- Let _targetReference_ be the result of evaluating _NewExpression_.
- Let _target_ be GetValue(_targetReference_).
- If IsCallable(_target_) is **false**, throw a **TypeError** exception.
- Let _F_ be BoundFunctionCreate(_target_, _baseValue_, ()).
- Let _targetHasLength_ be HasOwnProperty(_target_, "length").
- ReturnIfAbrupt(_targetHasLength_).
- If _targetHasLength_ is **true**, then
    - Let _targetLen_ be Get(_target_, "length").
    - ReturnIfAbrupt(_targetLen_).
    - If Type(_targetLen_) is not **Number**, then let _L_ be 0.
    - Else, let _L_ be ToInteger(_targetLen_).
- Else let _L_ be 0.
- Let _status_ be DefinePropertyOrThrow(_F_, "length", PropertyDescriptor {[[Value]]: _L_,
  [[Writable]]: **false**, [[Enumerable]]: **false**, [[Configurable]]: **true**}).
- ReturnIfAbrupt(_status_).
- Let _targetName_ be Get(_target_, "name").
- ReturnIfAbrupt(_targetName_).
- If Type(_targetName_) is not **String**, then let _targetName_ be the empty string.
- Let _status_ be SetFunctionName(_F_, _targetName_, "bound").
- ReturnIfAbrupt(_status_).
- Return _F_.


----

    BindExpression :
        :: NewExpression

- Let _targetReference_ be the result of evaluating _NewExpression_.
- Assert: Type(_targetReference_) is **Reference**
- Assert: IsSuperReference(_targetReference_) is **false**.
- Let _baseValue_ be GetBase(_targetReference_).
- Let _target_ be GetValue(_targetReference_).
- If IsCallable(_target_) is **false**, throw a **TypeError** exception.
- Let _F_ be BoundFunctionCreate(_target_, _baseValue_, ()).
- Let _targetHasLength_ be HasOwnProperty(_target_, "length").
- ReturnIfAbrupt(_targetHasLength_).
- If _targetHasLength_ is **true**, then
    - Let _targetLen_ be Get(_target_, "length").
    - ReturnIfAbrupt(_targetLen_).
    - If Type(_targetLen_) is not **Number**, then let _L_ be 0.
    - Else, let _L_ be ToInteger(_targetLen_).
- Else let _L_ be 0.
- Let _status_ be DefinePropertyOrThrow(_F_, "length", PropertyDescriptor {[[Value]]: _L_,
  [[Writable]]: **false**, [[Enumerable]]: **false**, [[Configurable]]: **true**}).
- ReturnIfAbrupt(_status_).
- Let _targetName_ be Get(_target_, "name").
- ReturnIfAbrupt(_targetName_).
- If Type(_targetName_) is not **String**, then let _targetName_ be the empty string.
- Let _status_ be SetFunctionName(_F_, _targetName_, "bound").
- ReturnIfAbrupt(_status_).
- Return _F_.
