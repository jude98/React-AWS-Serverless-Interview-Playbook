## Key Concepts

> [!summary] The Four Equality Operations
> 
> JavaScript specifies four distinct comparison operations under the ECMAScript standard:
> 
>   
> 
> 1. **Loose Equality / Abstract Equality (`==`)**: Compares values with implicit type coercion.
>     
>       
>     
> 2. **Strict Equality (`===`)**: Compares type and value without type coercion.
>     
>       
>     
> 3. **SameValue (`Object.is`)**: Exact identity comparison; distinguishes `+0` from `-0`, and evaluates `NaN === NaN` as `true`.
>     
>       
>     
> 4. **SameValueZero**: Internal equality algorithm used by modern collections (`Set`, `Map`, `Array.prototype.includes`). Like `SameValue`, it considers `NaN` equal to `NaN`, but treats `+0` and `-0` as identical.
>     
>       
>     

> [!abstract] Loose Equality Edge Cases (`==`)
> 
>   
> 
> - `null == undefined` is hard-coded to evaluate to `true` (and neither loosely equals any other falsy value like `0`, `""`, or `false`).
>     
>       
>     
> - Comparing a `boolean` to anything coerces the boolean to a number first (`true` $\to$ `1`, `false` $\to$ `0`).
>     
>       
>     
> - Comparing a `string` to a `number` coerces the string to a number via the `ToNumber` abstract operation.
>     
>       
>     
> - Comparing an `object` to a `primitive` runs `ToPrimitive(object)`.
>     
>       
>     

> [!danger] The Anomalies of `===`
> 
> Strict equality has two major IEEE 754 floating-point edge cases:
> 
>   
> 
> 1. `NaN === NaN` is `false` (by specification, `NaN` is not equal to any value, including itself).
>     
>       
>     
> 2. `+0 === -0` is `true` (even though they have distinct sign bits in memory and behave differently in division: `1 / +0 === Infinity`, while `1 / -0 === -Infinity`).
>     
>       
>     

## Common Interview Questions

- "What is the difference between `==`, `===`, and `Object.is()`?"
    
      
    
- "Explain step-by-step why `[] == ![]` evaluates to `true`."
    
      
    
- "Why does `null == undefined` evaluate to `true`, but `null >= 0` evaluates to `true` while `null == 0` is `false`?"
    
      
    
- "How do `Set` and `Array.prototype.includes` treat `NaN`, and why does that differ from `===`?"
    
      
    
- "What equality algorithm does `Object.is` implement, and what problem does it solve that `===` doesn't?"
    
      
    
- "How does React utilize `Object.is` under the hood in `React.memo` and `useEffect` dependency arrays?"
    
      
    

## Strong Answers / Talking Points

### 1. The Four Internal Algorithms

- **IsLooselyEqual ($x == y$)**:
    
      
    - If types match, delegates directly to `IsStrictlyEqual`.
        
          
        
    - If `null` and `undefined`, returns `true`.
        
          
        
    - If `number` and `string`, runs `ToNumber(string) == number`.
        
          
        
    - If one operand is `boolean`, runs `ToNumber(boolean) == other`.
        
          
        
    - If comparing `object` to `string | number | symbol | bigint`, runs `ToPrimitive(object) == primitive`.
        
          
        
    - Otherwise, returns `false`.
        
          
        
- **IsStrictlyEqual ($x === y$)**:
    
      
    - If types differ, returns `false`.
        
          
        
    - If both are numbers: if either is `NaN`, returns `false`; if `+0` and `-0`, returns `true`.
        
          
        
    - Otherwise, checks value equality or object reference identity.
        
          
        
- **SameValue ($Object.is(x, y)$)**:
    
      
    - Identical to `IsStrictlyEqual`, except:
        
          
        - `Object.is(NaN, NaN)` is `true`.
            
              
            
        - `Object.is(+0, -0)` is `false`.
            
              
            
- **SameValueZero**:
    
      
    - Used in `Map.prototype.set`, `Set.prototype.add`, `Array.prototype.indexOf` vs `includes`.
        
          
        
    - Identical to `SameValue`, except it treats `+0` and `-0` as equal (`SameValueZero(+0, -0)` is `true`).
        
          
        
    - Solves the classic bug where `[NaN].indexOf(NaN)` was `-1` (because `indexOf` uses `===`), whereas `[NaN].includes(NaN)` is `true` (uses `SameValueZero`).
        
          
        

### 2. Dissecting the Classic Trap: `[] == ![]`

1. Evaluate right side: `![]` coerces `[]` to a boolean. All objects are truthy, so `!true` evaluates to `false`.
    
      
    
2. Expression is now: `[] == false`.
    
      
    
3. Rule (boolean to number): `false` becomes `0`.
    
      
    
4. Expression is now: `[] == 0`.
    
      
    
5. Rule (object to primitive): `ToPrimitive([])` calls `[].toString()`, returning `""`.
    
      
    
6. Expression is now: `"" == 0`.
    
      
    
7. Rule (string to number): `ToNumber("")` converts empty string to `0`.
    
      
    
8. Expression is now: `0 == 0`, which evaluates to `true`.
    
      
    

### 3. The `null` Relational Comparison Paradox

- `null == 0` evaluates to `false` because `null` only loosely equals `null` or `undefined`.
    
      
    
- `null > 0` evaluates to `false` because relational comparison converts `null` to `0` via `ToNumber(null)`: `0 > 0` is `false`.
    
      
    
- `null >= 0` evaluates to `true` because the `>=` operator is evaluated as the logical negation of `<`: `!(null < 0)` $\to$ `!(0 < 0)` $\to$ `!false` $\to$ `true`.
    
      
    

## Code Snippets / Examples

### Algorithm Comparison Matrix in Code

JavaScript

```
// 1. NaN Comparison
console.log(NaN === NaN);           // false
console.log(Object.is(NaN, NaN));   // true
console.log([NaN].includes(NaN));   // true  (uses SameValueZero)
console.log([NaN].indexOf(NaN));    // -1    (uses IsStrictlyEqual / ===)

// 2. Positive & Negative Zero
console.log(+0 === -0);             // true
console.log(Object.is(+0, -0));     // false
console.log(1 / +0);                // Infinity
console.log(1 / -0);                // -Infinity

// 3. React-Style Shallow Equality Check (uses SameValue / Object.is)
function shallowEqual(objA, objB) {
  if (Object.is(objA, objB)) return true;
  if (typeof objA !== 'object' || objA === null ||
      typeof objB !== 'object' || objB === null) return false;

  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) return false;

  for (let key of keysA) {
    if (!Object.prototype.hasOwnProperty.call(objB, key) ||
        !Object.is(objA[key], objB[key])) {
      return false;
    }
  }
  return true;
}
```

### Predict the Output: Equality Corner Cases

JavaScript

```
console.log(null == undefined); // true
console.log(null === undefined);// false

console.log(0 == "");           // true  ("" -> 0)
console.log(0 == false);        // true  (false -> 0)
console.log("" == false);       // true  (both -> 0)

console.log("0" == false);      // true  (false -> 0, "0" -> 0)
console.log(false == []);       // true  (false -> 0, [] -> "" -> 0)
console.log(false == {});       // false ({} -> "[object Object]" -> NaN)

console.log([1, 2] == "1,2");   // true  ([1,2].toString() -> "1,2")
console.log({} == {});          // false (distinct object reference addresses in memory)
```

## Comparison Matrix

|**Equality Operation**|**IEEE NaN equal to NaN?**|**+0 equal to -0?**|**Type Coercion?**|**Where It Is Used**|
|---|---|---|---|---|
|**`==` (IsLooselyEqual)**|No (`false`)|Yes (`true`)|Yes|Legacy code, checking `x == null` (null or undefined)|
|**`===` (IsStrictlyEqual)**|No (`false`)|Yes (`true`)|No|Default comparison in general application code|
|**`Object.is()` (SameValue)**|Yes (`true`)|No (`false`)|No|`React.memo`, `useEffect` dependency diffing|
|**SameValueZero**|Yes (`true`)|Yes (`true`)|No|`Set`, `Map` keys, `Array.prototype.includes`|

## Related Topics

- [[JavaScript Type Casting: Coercion vs. Conversion & Predict-the-Output]]
    
      
    
- [[JavaScript Data Types, Objects & Prototypal Inheritance]]
    
      
    
- [[JavaScript Data Structures: Structured Data, Keyed & Indexed Collections]]
    
      
    
- [[React State Reconciliation and Shallow Comparison]]
    
      
    

## Tags

#fullstack #interview #javascript #equality #object-is #same-value #type-coercion

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups