# JavaScript Exception Handling: Try-Catch-Finally, Error Objects & Global Error Boundaries

## Key Concepts

> [!summary] Structured Exception Handling
> 
> Synchronous and `async/await` runtime errors are trapped using `try...catch...finally`:
> 
>   
> 
> - `try`: Wraps code that might throw an exception.
>     
>       
>     
> - `catch (err)`: Executes only if an exception is thrown inside `try`. Optional catch binding (`catch { ... }`) allows omitting `err` if unused (ES2019).
>     
>       
>     
> - `finally`: Unconditionally executes after `try` and `catch` finish, even if an unhandled error is thrown or a `return` statement is encountered.
>     
>       
>     

> [!abstract] Standard Error Objects
> 
> Built-in error constructors inherit from `Error.prototype`. They capture a `.message`, a `.name`, and an engine-generated `.stack` trace. Standard subclasses include `TypeError`, `ReferenceError`, `SyntaxError`, `RangeError`, `URIError`, and `EvalError`.
> 
>   

> [!danger] The Uncaught Exception Hazard
> 
> In single-threaded JavaScript:
> 
>   
> 
> - In the browser: An uncaught exception halts execution of the current task/script, logs to devtools, but leaves the overall page session alive.
>     
>       
>     
> - In Node.js: An uncaught exception leaves the process in an indeterminate state; by default, Node.js terminates the process (`process.exit(1)`).
>     
>       
>     

> [!tip] Global Unhandled Exception Handling
> 
> When code runs outside a local `try...catch` (e.g., forgotten catch blocks, background event handlers):
> 
>   
> 
> - **Browser**: Trapped globally via `window.onerror` / `window.addEventListener('error', ...)` for synchronous runtime errors, and `window.addEventListener('unhandledrejection', ...)` for unhandled promise rejections.
>     
>       
>     
> - **Node.js**: Trapped globally via `process.on('uncaughtException', ...)` and `process.on('unhandledRejection', ...)`.
>     
>       
>     

## Common Interview Questions

- "What happens if both the `try` block and the `finally` block have a `return` statement?"
    
      
    
- "Can a synchronous `try...catch` block catch an error thrown inside a `setTimeout` callback? Why or why not?"
    
      
    
- "How do you catch unhandled asynchronous Promise rejections across an entire application?"
    
      
    
- "What is the recommended operational practice when `process.on('uncaughtException')` is triggered in Node.js?"
    
      
    
- "What is the difference between `Error.captureStackTrace` and standard custom error subclassing?"
    
      
    
- "How does optional catch binding work, and when should you use it?"
    
      
    

## Strong Answers / Talking Points

### 1. The `finally` Block Override Rule

- If the `finally` block contains a `return` or `throw` statement, it **overrides and discards** any `return` value or uncaught `throw` from within the preceding `try` or `catch` blocks.
    
      
    
- Cleanup code (closing file handles, clearing connection pools, stopping timers) belongs in `finally`.
    
      
    

### 2. The Asynchronous Trap with Synchronous `try...catch`

- A standard `try...catch` cannot trap errors inside asynchronous callbacks like `setTimeout` or raw event listeners.
    
      
    
- **Why?** The `try` block finishes executing and is popped off the Call Stack immediately. When the timer callback fires later from the Task Queue, it runs in a brand-new call stack with no surrounding `try` block.
    
      
    
- **Solution**: Wrap logic inside the callback itself in `try...catch`, or use Promises with `async/await`.
    
      
    

### 3. Handling Global Unhandled Errors in the Browser

- **`window.addEventListener('error', callback)`**: Intercepts unhandled synchronous runtime errors and resource loading failures (e.g., broken `<img>` or `<script>` tags, using capture phase `{ capture: true }`).
    
      
    
- **`window.addEventListener('unhandledrejection', callback)`**: Catches rejected Promises that do not have a `.catch()` attached. Commonly used to pipe telemetry to tools like Sentry or Datadog.
    
      
    

### 4. Handling Global Unhandled Errors in Node.js

- **`process.on('uncaughtException', (err) => { ... })`**:
    
      
    - The process is now in an undefined, corrupted state (file descriptors might be half-written, memory buffers leaked).
        
          
        
    - Best practice: Log the stack trace, flush telemetry logs, and exit the process (`process.exit(1)`). Let an external process supervisor (PM2, Kubernetes, Docker) restart a clean instance.
        
          
        
- **`process.on('unhandledRejection', (reason, promise) => { ... })`**: Traps unhandled Promise rejections. In modern Node.js versions, unhandled rejections terminate the process with exit code 1 if not intercepted.
    
      
    

## Code Snippets / Examples

### `finally` Block Return Overwrite Trap

```javascript
function trapDemo() {
  try {
    throw new Error("Something broke!");
  } catch (err) {
    return "Handled in catch";
  } finally {
    return "Overridden by finally!"; // Discards the catch return value
  }
}

console.log(trapDemo()); // "Overridden by finally!"
```

### The Asynchronous Callback Pitfall

```javascript
// BROKEN: try...catch cannot trap callback errors
try {
  setTimeout(() => {
    throw new Error("Timer failed!"); // Uncaught Exception!
  }, 100);
} catch (err) {
  console.log("This will NEVER run!");
}

// FIXED: Handle errors inside the asynchronous scope
setTimeout(() => {
  try {
    throw new Error("Timer failed safely!");
  } catch (err) {
    console.error("Caught inside callback:", err.message);
  }
}, 100);
```

### Creating Custom Typed Errors

```javascript
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, ValidationError);
    }
  }
}

try {
  throw new ValidationError("Invalid email address", "email");
} catch (err) {
  if (err instanceof ValidationError) {
    console.error(`Validation failed on [${err.field}]: ${err.message}`);
  } else {
    throw err; // Re-throw unrecognized exceptions
  }
}
```

### Global Unhandled Error Handlers (Browser & Node.js)

```javascript
// ==========================================
// 1. CLIENT-SIDE (Browser Environment)
// ==========================================
window.addEventListener("error", (event) => {
  console.error("Global Error Caught:", event.message, event.filename, event.lineno);
  // Send error details to monitoring service (e.g., Sentry)
});

window.addEventListener("unhandledrejection", (event) => {
  console.error("Unhandled Promise Rejection:", event.reason);
  event.preventDefault(); // Prevents default browser console error output
});

// ==========================================
// 2. SERVER-SIDE (Node.js Environment)
// ==========================================
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled Rejection at:", promise, "reason:", reason);
  // Log telemetry; optionally exit
});

process.on("uncaughtException", (error) => {
  console.error("CRITICAL: Uncaught Exception! Shutting down...", error);
  // Close active servers, flush database connections
  // server.close(() => process.exit(1));
  process.exit(1); // Never attempt to keep running in an unhandled state
});
```

## Comparison Matrix: Error Trapping Mechanisms

|**Mechanism**|**Target Runtime**|**Trapped Errors**|**Primary Use Case**|
|---|---|---|---|
|**`try...catch...finally`**|All|Synchronous & `async/await`|Local, targeted error handling and recovery|
|**`window.onerror` / `window.addEventListener('error')`**|Browser|Uncaught synchronous runtime errors|Global telemetry, crash reporting|
|**`window.addEventListener('unhandledrejection')`**|Browser|Unhandled Promise rejections|Catching missing `.catch()` or `await` errors|
|**`process.on('uncaughtException')`**|Node.js|Uncaught synchronous/top-level errors|Graceful process exit and logging|
|**`process.on('unhandledRejection')`**|Node.js|Unhandled Promise rejections|Preventing silent failures, forced crash auditing|

## Related Topics

- [[JavaScript Promises & Async, Await. Architecture, Mechanics & Patterns|JavaScript Asynchronous Programming: Promises, Async/Await and Event Loop]]
    
      
    
- [[JavaScript Execution Context, Memory Creation & Hoisting Mechanics]]
    
      
    
- [[JavaScript Exception Handling. Try-Catch-Finally, Error Objects & Global Error Boundaries|Node.js Process Lifecycle and Exit Codes]]
    
      
    
- [[Debugging and Fixing Slow Dashboard Performance|Frontend Error Logging and Observability: Sentry and Datadog]]
    
      
    

## Tags

#fullstack #interview #javascript #error-handling #try-catch #exceptions #nodejs #browser

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups