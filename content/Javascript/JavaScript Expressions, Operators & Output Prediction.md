# JavaScript Expressions, Operators & Output Prediction

## Key Concepts

> [!summary] Expressions vs. Statements
> 
>   
> 
> - **Expression**: Any unit of code that resolves to a value (e.g., `5 + 2`, `x = 10`, `fn()`, `a ? b : c`). Can be passed as function arguments or assigned to variables.
>     
>       
>     
> - **Statement**: An instruction that performs an action (e.g., `if (...) {}`, `for (...) {}`, `return;`). Statements cannot be used where values are expected.
>     
>       
>     

> [!abstract] Logical Short-Circuiting & Modern Operators
> 
>   
> 
> - `&&` (Logical AND): Returns the **first falsy operand**, or the last operand if all are truthy.
>     
>       
>     
> - `||` (Logical OR): Returns the **first truthy operand**, or the last operand if all are falsy.
>     
>       
>     
> - `??` (Nullish Coalescing): Returns right-hand side **only if left-hand side is `null` or `undefined`**. Treats `0`, `""`, and `false` as valid values.
>     
>       
>     
> - `?.` (Optional Chaining): Short-circuits and evaluates to `undefined` instead of throwing a `TypeError` if target is nullish.
>     
>       
>     

> [!info] Unary Operators (`+`, `-`, `++`, `--`, `typeof`, `delete`, `void`)
> 
>   
> 
> - Unary `+` / `-`: Converts operand to number via `ToNumber`.
>     
>       
>     
> - Prefix (`++x`) vs. Postfix (`x++`): Prefix increments and returns the new value; postfix increments but returns the value _before_ incrementing.
>     
>       
>     
> - `delete`: Removes own properties from an object; returns `true` on deletion or if property doesn't exist, but **cannot delete variables declared with `var`, `let`, or `const`**.
>     
>       
>     
> - `void`: Evaluates an expression and unconditionally returns `undefined` (e.g., `void 0`).
>     
>       
>     

> [!danger] The Comma Operator (`,`)
> 
> Evaluates each of its operands from left to right and **returns the value of the last operand**. Commonly used in minified code or concise expressions, but often appears as a trap in interviews.
> 
>   

## Common Interview Questions

- "What is the difference between `||` and `??` (Nullish Coalescing)?"
    
      
    
- "Explain the difference between prefix increment (`++i`) and postfix increment (`i++`)."
    
      
    
- "What does the comma operator do, and what does `let a = (1, 2, 3);` evaluate to?"
    
      
    
- "What does `delete` actually do? Can you delete a variable or a prototype property?"
    
      
    
- "Why does `typeof null` return `'object'`, while `typeof undefined` returns `'undefined'`?"
    
      
    
- "Can you chain optional chaining with function calls or dynamic properties (`obj?.[key]?.()` )?"
    
      
    
- "Explain operator precedence and associativity: why does `1 < 2 < 3` return `true`, but `3 > 2 > 1` return `false`?"
    
      
    

## Strong Answers / Talking Points

### 1. `||` vs. `??` (Logical OR vs. Nullish Coalescing)

- **`||` Trap**: Checks for _falsy_ values (`false`, `0`, `""`, `NaN`, `null`, `undefined`). If a valid input is `0` (e.g., `score || 10`) or an empty string, `||` accidentally falls back to the default.
    
      
    
- **`??` Solution**: Specifically checks for _nullish_ values (`null` or `undefined`). Valid values like `0`, `""`, and `false` are retained.
    
      
    
- _Restriction_: You cannot combine `??` directly with `&&` or `||` without explicit parentheses (`(a || b) ?? c`), throwing a `SyntaxError`.
    
      
    

### 2. Postfix vs. Prefix Evaluation Flow

- `let y = x++`:
    
      
    1. Temporary variable copies current `x`.
        
          
        
    2. `x` is incremented in memory.
        
          
        
    3. Returns the temporary copy (original value).
        
          
        
- `let y = ++x`:
    
      
    1. `x` is incremented in memory.
        
          
        
    2. Returns the newly updated value of `x`.
        
          
        

### 3. Relational Operator Chaining Trap (`1 < 2 < 3` vs. `3 > 2 > 1`)

- JavaScript evaluates comparison operators from **left to right** (left-associative).
    
      
    
- `1 < 2 < 3` $\to$ `(1 < 2) < 3` $\to$ `true < 3` $\to$ `1 < 3` $\to$ `true`.
    
      
    
- `3 > 2 > 1` $\to$ `(3 > 2) > 1` $\to$ `true > 1` $\to$ `1 > 1` $\to$ `false`.
    
      
    

### 4. The `delete` Operator Mechanics

- Deletes only **own configurable properties** from an object.
    
      
    
- Returns `false` in strict mode when attempting to delete non-configurable properties (or throws a `TypeError`).
    
      
    
- Does **not** affect prototype properties; deleting an inherited property on an instance does nothing and leaves the prototype intact.
    
      
    
- Cannot delete direct variable bindings declared with `var`, `let`, `const`, or function declarations.
    
      
    

## Code Snippets / Examples

### Logical Operators, Short-Circuiting & Nullish Coalescing

```javascript
// Short-circuiting evaluation
console.log("hello" && 0 && "world"); // 0 (stops at first falsy)
console.log(null || false || "fallback"); // "fallback" (stops at first truthy)

// || vs ?? comparison
const config = {
  timeout: 0,
  title: "",
  debug: false
};

// || falls back incorrectly on 0, "", false
console.log(config.timeout || 3000); // 3000 (0 is falsy)
console.log(config.title || "Untitled"); // "Untitled" ("" is falsy)

// ?? handles 0, "", false correctly
console.log(config.timeout ?? 3000); // 0
console.log(config.title ?? "Untitled"); // ""
console.log(config.debug ?? true); // false
```

### Unary & Comma Operator Traps

```javascript
// Comma operator evaluation
let result = (1 + 1, 2 * 3, 4 + 5);
console.log(result); // 9 (last expression is returned)

// Postfix vs Prefix in complex expressions
let a = 1;
let b = a++ + ++a; 
// Step 1: a++ yields 1 (a becomes 2)
// Step 2: ++a increments a to 3, yields 3
// Result: 1 + 3 = 4
console.log(b); // 4
console.log(a); // 3

// Unary plus coercion
console.log(+true);      // 1
console.log(+false);     // 0
console.log(+null);      // 0
console.log(+undefined); // NaN
```

### Predict the Output: Common Interview Operator Puzzles

```javascript
// Puzzle 1: Relational chaining
console.log(1 < 2 < 3); // true  ((1 < 2) -> true -> 1 < 3)
console.log(3 > 2 > 1); // false ((3 > 2) -> true -> 1 > 1)

// Puzzle 2: delete operator
const person = { name: "Alice" };
let city = "Seattle";

console.log(delete person.name); // true
console.log(person.name);        // undefined
console.log(delete person.age);  // true (property doesn't exist, still returns true)
console.log(delete city);        // false (cannot delete let/const/var declarations)

// Puzzle 3: Optional chaining with methods
const user = {
  getName: null
};

// Safe invocation: prevents "user.getName is not a function"
console.log(user.getName?.()); // undefined

// Puzzle 4: void operator
console.log(void (1 + 1)); // undefined
console.log(void 0);       // undefined (idiomatic safe replacement for global undefined)
```

## Operator Precedence Summary (Highest to Lowest)

|**Precedence**|**Category**|**Operators**|**Associativity**|
|---|---|---|---|
|**19**|Member Access / Calls|`.` `?.` `[]` `()`|Left-to-right|
|**15**|Postfix Increment/Decrement|`x++` `x--`|N/A|
|**14**|Prefix / Unary|`++x` `--x` `+` `-` `!` `typeof` `void` `delete`|Right-to-left|
|**13**|Exponentiation|`**`|**Right-to-left**|
|**12**|Multiplicative|`*` `/` `%`|Left-to-right|
|**11**|Additive|`+` `-`|Left-to-right|
|**9**|Relational|`<` `<=` `>` `>=` `in` `instanceof`|Left-to-right|
|**8**|Equality|`==` `!=` `===` `!==`|Left-to-right|
|**4**|Logical AND|`&&`|Left-to-right|
|**3**|Logical OR / Nullish|`\|` `??`|Left-to-right|
|**2**|Conditional / Ternary|`? :`|**Right-to-left**|
|**2**|Assignment|`=` `+=` `-=` `??=`|**Right-to-left**|
|**1**|Comma|`,`|Left-to-right|

## Related Topics

- [[JavaScript Type Casting Coercion vs. Conversion & Predict-the-Output|JavaScript Type Casting: Coercion vs. Conversion & Predict-the-Output]]
    
      
    
- [[JavaScript Equality Comparisons & Internal Algorithms]]
    
      
    
- [[JavaScript Data Types, Objects & Prototypal Inheritance]]
    
      
    
- [[JavaScript Control Flow. If-Else, Ternary & Switch Statements|Control Flow and Conditional Statements]]
    
      
    

## Tags

#fullstack #interview #javascript #operators #expressions #precedence #short-circuit

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups