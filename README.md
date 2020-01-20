# Private function arguments for JavaScript

Private function arguments for creating closures implicitly.

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

In order to avoid confusion between the order of parameters, private arguments should be specified after their public counterparts, even including rest parameters:

```js
// The public method signature seen by callers is `fn(x, y, ...rest)`
function fn(x, y, ...rest, #a = 0, #b) { /* ... */ }
```

Please take note that private arguments may not need to be initialized. They act like syntactic sugar for defining a new closure with multiple outer variables.

### Optimization opportunities for preprocessors

By detecting the operator usage sorrounding private arguments, a compiler may be able to optimize `const x = 20 * fact(3)` as follows:

```js
const x = fact(3, 20);

function fact(n, _acc) {
  if (n > 0) return fact(n - 1, n * _acc);
  return _acc;
}
```
