# JavaScript Control Flow: If-Else, Ternary & Switch Statements

## Key Concepts

> [!summary] Control Flow Overview
> 
> Control flow defines the order in which statements are evaluated at runtime. Branching constructs allow programs to take divergent paths based on boolean evaluation (`if...else`, ternary `?:`) or strict pattern matching (`switch`).
> 
>   

> [!abstract] `if...else` Evaluation & Truthiness
> 
> An `if (expression)` statement coerces its condition via the `ToBoolean` abstract operation. If the result is truthy, the `if` block executes; otherwise, execution falls through to subsequent `else if` conditions or the optional terminating `else` block.
> 
>   

> [!info] The Conditional (Ternary) Operator (`condition ? expr1 : expr2`)
> 
> The only operator in JavaScript taking three operands. Unlike `if...else` (which is a **statement**), the ternary operator is an **expression** that evaluates and returns a value, making it valid inside JSX, template literals, and variable assignments.
> 
>   

> [!danger] `switch` Statement: Fall-Through & Strict Matching
> 
>   
> 
> - The `switch (expression)` statement compares candidate cases using **strict equality (`===`)**—no type coercion occurs.
>     
>       
>     
> - Execution jumps to the matching `case` and continues sequentially across subsequent cases (**fall-through**) until a `break` or `return` statement is encountered, or the block ends.
>     
>       
>     
> - The `default` clause handles unmatched expressions regardless of its position in the block, though it is conventionally placed at the end.
>     
>       
>     

## Common Interview Questions

- "How does a `switch` statement compare values under the hood? Does `'5'` match `case 5:`?"
    
      
    
- "What is case fall-through in a `switch` statement, and when is it intentionally useful?"
    
      
    
- "What is the structural difference between an `if...else` block and a ternary expression?"
    
      
    
- "Why can lexical declarations (`let`, `const`) cause syntax errors inside a `switch` statement without block braces?"
    
      
    
- "When would you prefer a lookup object/Map over a complex `switch` or long `if...else if` chain?"
    
      
    

## Strong Answers / Talking Points

### 1. `switch` Uses Strict Equality (`===`)

- Interviewers frequently test coercion in `switch`. A string expression `"10"` will **not** match `case 10:`, because the engine evaluates `expression === caseValue`.
    
      
    
- If you need range evaluations or dynamic conditions in a `switch`, the common pattern is `switch (true)`, where each case evaluates a boolean condition (`case x > 10:`).
    
      
    

### 2. Lexical Scoping Inside `switch` Blocks

- A `switch` statement forms a **single overarching block scope** across all its `case` clauses.
    
      
    
- Declaring `let x = 1` inside `case 'A':` and another `let x = 2` inside `case 'B':` throws a `SyntaxError: Identifier 'x' has already been declared`.
    
      
    
- **Solution**: Wrap individual cases in explicit curly braces (`case 'A': { let x = 1; break; }`) to create isolated lexical environments per branch.
    
      
    

### 3. Ternary Operator (`?:`) Best Practices & Trade-offs

- **Pros**: Returns a value inline; useful in functional paradigms, React conditional rendering (`{isLoggedIn ? <Dashboard/> : <Login/>}`), and immutable variable assignments (`const status = active ? 'ON' : 'OFF'`).
    
      
    
- **Cons**: Deeply nested ternaries hurt readability and maintainability. For more than two branches, prefer `if...else`, a `switch`, or an early `return` pattern.
    
      
    

### 4. Branching Alternatives: Lookup Tables (Objects / Maps)

- Long `if...else if` or `switch` blocks evaluate sequentially in $O(N)$ worst-case time.
    
      
    
- Replacing them with a plain object dictionary or `Map` gives $O(1)$ constant-time lookups, cleaner code separation, and easier unit testing (open-closed principle).
    
      
    

## Code Snippets / Examples

### `switch` Lexical Scope Pitfall & Fix

```javascript
const action = "LOGIN";

// Broken: SyntaxError due to shared switch block scope
// switch (action) {
//   case "LOGIN":
//     let message = "Logging in...";
//     break;
//   case "LOGOUT":
//     let message = "Logging out..."; // SyntaxError: Identifier 'message' has already been declared
//     break;
// }

// Fixed: Isolated block scopes per case
switch (action) {
  case "LOGIN": {
    const message = "Logging in...";
    console.log(message);
    break;
  }
  case "LOGOUT": {
    const message = "Logging out...";
    console.log(message);
    break;
  }
  default: {
    console.log("Unknown action");
    break;
  }
}
```

### Intentional Fall-Through & `switch (true)` Pattern

```javascript
// 1. Grouping multiple cases via intentional fall-through
function getDayCategory(dayNumber) {
  switch (dayNumber) {
    case 1:
    case 2:
    case 3:
    case 4:
    case 5:
      return "Weekday";
    case 6:
    case 7:
      return "Weekend";
    default:
      return "Invalid Day";
  }
}

// 2. Dynamic range evaluation with switch (true)
function getGrade(score) {
  switch (true) {
    case score >= 90:
      return "A";
    case score >= 80:
      return "B";
    case score >= 70:
      return "C";
    default:
      return "F";
  }
}
```

### Refactoring Branching Logic: Dictionary / Map Pattern

```javascript
// Before: Verbose switch statement
function getDiscountByRole(role) {
  switch (role) {
    case "VIP": return 0.20;
    case "PREMIUM": return 0.10;
    case "REGULAR": return 0.05;
    default: return 0.0;
  }
}

// After: Clean O(1) Object Lookup Table
const ROLE_DISCOUNTS = {
  VIP: 0.20,
  PREMIUM: 0.10,
  REGULAR: 0.05
};

function getDiscount(role) {
  return ROLE_DISCOUNTS[role] ?? 0.0;
}
```

## Comparison Matrix

|**Feature**|**if...else**|**Ternary (? :)**|**switch**|**Object/Map Lookup**|
|---|---|---|---|---|
|**Type**|Statement|**Expression** (evaluates to value)|Statement|Expression / Data structure|
|**Condition Flexibility**|Any truthy/falsy evaluation|Any truthy/falsy evaluation|Discrete values via `===`|Exact key matching ($O(1)$)|
|**Early Termination**|Via `return` / `throw`|Not supported|Via `break` / `return`|N/A (direct access)|
|**Best Used For**|Complex logic, range checks|Simple inline conditionals (JSX)|Enum matching, grouping cases|Fixed dictionary dispatching|

## Related Topics

- [[JavaScript Expressions, Operators & Output Prediction]]
    
      
    
- [[JavaScript Equality Comparisons & Internal Algorithms]]
    
      
    
- [[JavaScript Scope, Lexical Environment, and Shadowing]]
    
      
    
- [[Clean Architecture, Directory Structure & DTOs|Clean Code & Refactoring: Guard Clauses and Early Returns]]
    
      
    

## Tags

#fullstack #interview #javascript #control-flow #conditional #switch-case #ternary

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups