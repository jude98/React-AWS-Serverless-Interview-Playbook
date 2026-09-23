# JavaScript Type Casting Coercion vs. Conversion & Predict-the-Output

## Key Concepts

> [!summary] Type Conversion vs. Type Coercion
>
> - **Type Conversion (Explicit)**: The developer intentionally converts a value from one type to another using built-in functions or operators (e.g., `Number("42")`, `String(123)`, `Boolean(val)`).
>
> - **Type Coercion (Implicit)**: The JavaScript engine automatically converts types behind the scenes when evaluating expressions with mismatched types (e.g., `"5" + 2`, `if ("hello")`).


> [!abstract] The Three Target Types
>
> Coercion and conversion only ever convert values into three target types:
>
> 1. **To Boolean**: Evaluated via truthy/falsy rules.
>
> 2. **To String**: Triggered by string concatenation or string functions.
>
> 3. **To Number**: Triggered by arithmetic operators (`-`, `*`, `/`, `%`), bitwise operators, or numeric comparison (`>`, `<`).


> [!danger] The 8 Falsy Values
>
> Only eight values evaluate to `false` in JavaScript; **all** other values (including empty arrays `[]`, empty objects `{}`, and the string `"0"`) are truthy:
>
> - `false`, `0`, `-0`, `0n` (BigInt zero), `""` (empty string), `null`, `undefined`, `NaN`.


> [!info] Object-to-Primitive Algorithm (`ToPrimitive`)
>
> When an object is coerced into a primitive (string or number), the engine looks for:
>
> 1. `[Symbol.toPrimitive](hint)` if defined.
>
> 2. If hint is **"string"**: calls `.toString()` first, then `.valueOf()`.
>
> 3. If hint is **"number"** or **"default"**: calls `.valueOf()` first, then `.toString()`.


## Common Interview Questions

- "What is the difference between explicit type conversion and implicit type coercion?"

- "Why does `[] + []` equal `""`, but `[] + {}` equals `"[object Object]"`?"

- "What is the concrete difference between `==` (loose equality) and `===` (strict equality) under the hood?"

- "Why does `null == undefined` evaluate to `true`, but `null === undefined` evaluate to `false`?"

- "Explain what the unary `+` operator and the `!!` idiom do."

- "What happens step-by-step when evaluating `'5' - - '3'` or `true + false`?"

## Strong Answers / Talking Points

### 1. The Binary `+` Operator vs. Other Arithmetic Operators

- The binary `+` operator is overloaded: it performs **numeric addition** OR **string concatenation**.

- **Rule**: If _either_ operand evaluates to a string (or coerces to one via `ToPrimitive`), concatenation takes priority over addition.

- In contrast, operators like `-`, `*`, `/`, and `%` have no string overload. They _always_ coerce both operands to numbers using the `ToNumber` abstract operation.

### 2. Loose Equality (`==`) Algorithm (Abstract Equality)

- `===` checks both **type and value** without coercion (with exceptions: `NaN === NaN` is `false`, `-0 === +0` is `true`).

- `==` performs type coercion before comparison following ECMAScript rules:

    - If comparing `string` and `number`: string is coerced to number (`'5' == 5` -> `5 == 5`).

    - If comparing `boolean` to anything: boolean is coerced to number (`true` -> `1`, `false` -> `0`).

    - `null == undefined` is hard-coded to `true` (and neither loosely equals any other value).

    - If comparing an `object` to a `primitive`: object is converted to primitive via `ToPrimitive`.

### 3. Edge Cases: Arrays and Objects in Coercion

- An empty array `[]`:

    - `[].toString()` yields `""`.

    - `Number("")` yields `0`.

    - `Boolean([])` yields `true` (all objects are truthy).

- An empty object `{}`:

    - `{}.toString()` yields `"[object Object]"`.

## Code Snippets / Examples

### Explicit Conversion Patterns

```javascript
// To Number
Number("42");       // 42
Number("");         // 0
Number(null);       // 0
Number(undefined);  // NaN
parseInt("42px");   // 42 (stops at first non-digit)
+"42";              // 42 (Unary plus idiom)

// To String
String(123);        // "123"
String(null);       // "null"
(123).toString();   // "123"

// To Boolean
Boolean(0);         // false
Boolean("hello");   // true
!!"hello";          // true (Double-bang idiom)
```

### Classic "Predict the Output" Traps

```javascript
// Trap 1: Plus vs Minus
console.log("5" + 2);     // "52" (string concatenation)
console.log("5" - 2);     // 3    (numeric subtraction)
console.log("5" * "2");   // 10   (numeric multiplication)

// Trap 2: Booleans in Math
console.log(true + true); // 2    (1 + 1)
console.log(true - false);// 1    (1 - 0)
console.log(true + "2");  // "true2"

// Trap 3: Arrays & Objects
console.log([] + []);     // ""   (both coerce to "")
console.log([] + {});     // "[object Object]" ("" + "[object Object]")
console.log([1, 2] + [3]);// "1,23" ("1,2" + "3")
console.log([] == 0);     // true ([] -> "" -> 0)
console.log([] == ![]);   // true (![] is false -> [] == false -> 0 == 0)

// Trap 4: Null & Undefined Equality
console.log(null == undefined); // true
console.log(null === undefined);// false
console.log(null == 0);         // false (null only == null or undefined)
console.log(null >= 0);         // true  (relational >= coerces null to 0: 0 >= 0)

// Trap 5: The Infamous NaN
console.log(NaN == NaN);        // false (NaN is never equal to anything, including itself)
console.log(Object.is(NaN, NaN));// true
```

## Related Topics

- [[JavaScript Data Types, Objects & Prototypal Inheritance]]

- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]

- [[JavaScript Equality Comparisons & Internal Algorithms|JavaScript Equality Comparisons: == vs === vs Object.is]]

- [[JavaScript Type Casting Coercion vs. Conversion & Predict-the-Output|Symbol.toPrimitive and Object-to-Primitive Algorithms]]

## Tags

#fullstack #interview #javascript #type-coercion #type-casting #predict-the-output

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
