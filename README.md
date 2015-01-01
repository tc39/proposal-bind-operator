## ECMAScript Function Bind Syntax ##

### Motivation and Overview ###

TODO

### Examples ###

Using method extraction to print the eventual value of a promise to the console:

```js
Promise.resolve(123).then(::console.log);
```

Using method extraction to call an object method when a DOM event occurs:

```js
$(".some-link").on("click", ::view.reset);
```

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


### Runtime Semantics:  Method Extraction ###

    CallExpresion :
        :: MemberExpression

- Let `targetReference` be the result of evaluating `MemberExpression`.
- Let `baseValue` be GetBase(`targetReference`).
- ReturnIfAbrupt(`baseValue`).
- Let `bv` be RequireObjectCoercible(`baseValue`).
- ReturnIfAbrupt(`bv`).
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
