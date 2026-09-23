# React Fundamentals and Core Concepts

> [!note] React Core Definition
> 
> A declarative, component-based JavaScript library designed exclusively for building responsive user interfaces by synchronizing view states via a virtual representation of the DOM.
> 
>   

> [!abstract] Library vs Framework
> 
> A library is unopinionated and gives you control over architectural decisions (Inversion of Control remains with the developer); a framework controls the architecture and lifecycle, calling your code into its prescribed skeleton.
> 
>   

## Key Concepts

- **Declarative UI**: You declare _what_ the UI should look like for a given state (`UI = f(state)`), and React handles _how_ to mutate the DOM to match it.
    
      
    
- **Component-Driven Architecture**: Breaks monolithic web pages into reusable, encapsulated, and testable units of UI that manage their own state.
    
      
    
- **Library vs. Framework**: React only solves the "View" layer of MVC. Routing, state management, build tooling, and data fetching are left to developer choice.
    
      
    
- **Virtual DOM (VDOM) and Reconciliation**: Maintains an in-memory tree representing the real DOM; computes minimal batch updates (diffing) to avoid costly direct DOM reflows and repaints.
    
      
    
- **Unidirectional Data Flow**: Data flows strictly downward from parent to child via props, making state changes predictable and easier to debug than two-way data-binding.
    
      
    

## Common Interview Questions

- What is React, and why is it categorized as a library rather than a framework?
    
      
    
- What core problems does React solve that plain HTML, CSS, and Vanilla JavaScript struggle with at scale?
    
      
    
- What is the difference between Imperative programming and Declarative programming in frontend development?
    
      
    
- What is the Virtual DOM, and does it make React faster than Vanilla JavaScript?
    
      
    
- What are the trade-offs of using an unopinionated library like React over an opinionated framework like Angular or Next.js?
    
      
    
- How does unidirectional data flow improve maintainability in large-scale applications?
    
      
    

## Strong Answers / Talking Points

- **Why React is a Library, Not a Framework**:
    
      
    - _Core responsibility_: React only provides primitives for rendering UI and managing component state (`useState`, `useEffect`, JSX).
        
          
        
    - _Inversion of Control_: In Angular or NestJS, the framework dictates folder structure, routing, HTTP services, and dependency injection. With React, you architect the app and call React as a tool.
        
          
        
    - _Trade-off_: Maximum flexibility to pair with tools like React Query, Zustand, or React Router, at the cost of decision fatigue and inconsistent project structures across teams.
        
          
        
- **React vs Plain HTML, CSS, and Vanilla JavaScript**:
    
      
    - _The State Synchronization Problem_: In vanilla JS, changing data requires manually finding DOM nodes (`document.querySelector`) and mutating them (`element.innerHTML = ...`). As apps grow, UI and state fall out of sync easily (spaghetti code).
        
          
        
    - _Reusability_: HTML is static. Reusing an HTML component in vanilla code requires string concatenation or template tags. React encapsulates logic, markup, and styling into cohesive, testable components.
        
          
        
    - _Performance predictability_: Direct DOM operations trigger expensive browser recalculations (Layout/Reflow and Paint). React batches updates and computes minimal DOM diffs via its reconciliation engine.
        
          
        
- **Is the Virtual DOM faster than direct DOM manipulation?**:
    
      
    - _Nuance point_: No, direct, handwritten DOM updates tailored to a specific use case are theoretically always faster.
        
          
        
    - _Real benefit_: React gives you _sufficiently fast_ performance by default across complex, dynamic trees without requiring tedious manual DOM optimizations.
        
          
        

## Code Snippets / Examples



```JavaScript
// Imperative (Vanilla JS): Developer dictates HOW to update the DOM
const button = document.createElement('button');
let count = 0;
button.textContent = `Clicks: ${count}`;
button.addEventListener('click', () => {
  count += 1;
  button.textContent = `Clicks: ${count}`; // Manual DOM mutation
});
document.body.appendChild(button);
```



```JavaScript
// Declarative (React): Developer declares WHAT the UI looks like for a given state
import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);

  // React manages DOM updates automatically when state changes
  return (
    <button onClick={() => setCount(prev => prev + 1)}>
      Clicks: {count}
    </button>
  );
}
```

## Related Topics

- [[React Reconciliation and Diffing Algorithm|Virtual DOM and Reconciliation]]
    
      
    
- [[React Lifecycle and Execution Flow]]
    
      
    
- [[JSX and ReactDOM Execution Pipeline|JSX and Babel Compilation]]
    
      
    
- [[React Fundamentals and Core Concepts|Imperative vs Declarative UI]]
    
      
    
- [[React State and Props Architecture|React State and Props]]
    
      
    

## Tags

#fullstack #interview #react-fundamentals

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups