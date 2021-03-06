# Private function arguments for JavaScript

Private function arguments for creating closures implicitly.

[Discussion on TC39 Discourse](https://es.discourse.group/t/private-function-arguments-for-creating-closures-implicitly/199)

## Motivation

Tail recursive function parameters should be encapsulated in a closure to avoid modifying them from the outside. The following code demonstrates calculating the factorial of an integer:

```js
function fact(n) {
  // A closure is defined explicitly to protect `acc` from unwanted modification
  function fact_inner(n, acc) {
    if (n > 0) return fact_inner(n - 1, n * acc);
    return acc;
  }
  return fact_inner(n, 1);
}
```

While `fact` accounts for a simple mechanism, the explicit closure greatly increases the mental complexity of the code. As [private class fields](https://github.com/tc39/proposal-class-fields#private-fields) are already on the track of standardization, a similar syntax could be utilized for making function parameters inaccessible for callers:

```js
// From the outside, the function below only takes a single argument
function fact(n, #acc = 1) {
  if (n > 0) return fact(n - 1, n * #acc);
  return #acc;
}
```

Transformation between the presented code snippets is possible with transpilers.

## Details

### Private arguments shall be specified after public ones

Private arguments shall not be seen by callers. Extra parameters provided beyond the public ones shall be ignored. The value of private arguments may not be overrided from outside the scope of the function body. For example, calling `fact(3, 999, null)` should retain the value of `#acc = 1` by ignoring superfluous parameters.

In order to avoid confusion between the order of parameters, private arguments should be specified after their public counterparts:

```js
// The public method signature seen by callers is `fn(x, y)`
function fn(x, y, #a = 0, #b) { /* ... */ }
```

Please take note that private arguments act like syntactic sugar for defining a new closure with multiple outer variables.

### Rest parameters may not be mixed with private arguments

- Private rest parameters like `...#args` or `#...args` are out of consideration, as they provide no known value as accumulator variables. However, if a specific use-case was missed, please open a new issue regarding the case!

- Concerns were [raised](https://github.com/kripod/tc39-proposal-private-function-arguments/issues/1) about compatibility with public rest parameters. Being the least confusing approach to date, private arguments shall not be allowed alongside rest parameters.

### Default values for private arguments are mandatory

As [private arguments shall be specified after public ones](#private-arguments-shall-be-specified-after-public-ones), it would not make sense to allow private arguments without a default parameter, as the last public argument may also be optional.

```js
function fn1(x, y = 1, #a,     #b = 3) {} // Incorrect
function fn2(x, y = 1, #a = 2, #b = 3) {} // Correct
```

### `arguments` are not modified

To retain backwards compatibility, the function-specific `arguments` object shall not include private arguments.

### When transformed, the inner function should forward operations to the outer method

As observed in [issue #2](https://github.com/kripod/tc39-proposal-private-function-arguments/issues/2), the following code should always modify the outer function:

```js
fact.x = 'test';
function fact(n, #acc = 1) {
  if (n > 0) return fact(n - 1, n * #acc);
  console.log(fact.x); // 'test'
  return #acc;
}
```

However, a `Proxy` is required to bridge the gap between `fact_inner` and `fact.x` inside the function body:

```js
function fact(n) {
  function fact_inner(n, acc) {
    if (n > 0) return fact_inner(n - 1, n * acc);
    return acc;
  }
  const fact_inner_proxy = new Proxy(
    fact_inner,
    {
      set(_obj, prop, value) {
        fact[prop] = value;
        return true;
      },
      get(_obj, prop) {
        return fact[prop];
      },
      // Other traps may also be added
    },
  );
  return fact_inner_proxy(n, 1);
}
```

### Optimization opportunities for preprocessors

By detecting the operator usage sorrounding private arguments, a compiler may be able to optimize `const x = 20 * fact(3)` as follows:

```js
const x = fact(3, 20);

function fact(n, _acc) {
  if (n > 0) return fact(n - 1, n * _acc);
  return _acc;
}
```
