# DOM Event Propagation: Bubbling, Capturing & Event Delegation

## Key Concepts

> [!summary] The 3 Phases of DOM Event Propagation
> When an event occurs on a DOM element, the browser does not trigger the event on that element in isolation. The event travels along a bidirectional path through the DOM tree in three distinct phases:
> 1. **Capturing Phase (Trickling)**: The event starts from the `Window`, descends through `Document`, `<html>`, `<body>`, and down ancestor nodes toward the target element.
> 2. **Target Phase**: The event reaches the deepest element where the interaction physically occurred (`event.target`).
> 3. **Bubbling Phase**: The event reverses direction and ascends back up through parent elements all the way back to `Window`.
> 
> 

> [!abstract] Event Listener Phase Configuration
> `addEventListener(type, listener, useCapture)`:
> * By default, `useCapture` is `false` (or omitted). The listener fires **only during the Target and Bubbling phases**.
> * Passing `true` (or `{ capture: true }`) configures the listener to fire **during the Capturing phase**.
> 
> 

> [!info] `event.target` vs. `event.currentTarget`
> * `event.target`: The actual deepest DOM node where the user initiated the interaction (e.g., the specific `<span>` or `<button>` clicked).
> * `event.currentTarget`: The element to which the event listener is currently attached (equivalent to `this` inside standard non-arrow listener functions).
> 
> 

> [!danger] Halting Event Propagation
> * `event.stopPropagation()`: Prevents the event from traveling further along the propagation path (neither continuing down during capture nor ascending up during bubble). Other listeners attached to the *same current element* will still execute.
> * `event.stopImmediatePropagation()`: Prevents the event from bubbling/capturing **and** immediately blocks execution of any other listeners registered on that exact same element for that event type.
> * `event.preventDefault()`: Prevents the default browser action (e.g., following a link, submitting a form, checking a checkbox); does **not** stop propagation.
> 
> 

> [!tip] Event Delegation Defined
> A design pattern where instead of binding individual listeners to numerous child elements, you attach a **single event listener** to a common parent element. The parent leverages event bubbling to intercept events originating from any existing or dynamically appended children.

## Common Interview Questions

* "Walk me through the three phases of DOM event propagation."
* "What is the difference between `event.target` and `event.currentTarget`?"
* "What is the difference between `event.stopPropagation()`, `event.stopImmediatePropagation()`, and `event.preventDefault()`?"
* "What is Event Delegation, and how does it prevent memory leaks and optimize performance?"
* "Do all DOM events bubble? Name at least three events that do not bubble."
* "How do you handle clicks on dynamically injected elements using Event Delegation?"
* "Why does `element.closest()` play a critical role in robust Event Delegation implementations?"

## Strong Answers / Talking Points

### 1. Capturing vs. Bubbling Under the Hood

* Historical context: Netscape championed Event Capturing (top-down), while Microsoft Internet Explorer championed Event Bubbling (bottom-up).
* The W3C standardized both into a unified 3-phase model: events trickle down from root to target (capturing), hit the target, and bubble back up (bubbling).
* Capturing is rarely needed in standard business code, but is useful for global interception (e.g., analytics click-tracking, input masking, modal dismissal overlays, or blocking events before children receive them).

### 2. Events That Do NOT Bubble

* Not all events bubble up the DOM tree. Common examples:
* Form focus: `focus` and `blur` (use their capturing variants or bubbling equivalents `focusin` and `focusout`).
* Mouse movement: `mouseenter` and `mouseleave` (do not bubble; use `mouseover` and `mouseout` if bubbling is needed).
* Media/Resource loading: `load`, `unload`, `error` (on images/scripts), `scroll` (on elements, though `window` scroll can be listened to).

* Always verify if an event bubbles when attempting to apply Event Delegation.

### 3. The Power of Event Delegation

* **Memory Efficiency**: In a list of 10,000 items, adding 10,000 listeners creates 10,000 V8 function closures and 10,000 native browser C++ listener records. Delegation reduces this to **exactly 1 listener** on the parent container.
* **Dynamic Elements**: When new list items are added asynchronously via API responses or user input, zero additional listener bindings are required. The parent handles them automatically.
* **Garbage Collection**: Reduces the surface area for detached DOM node memory leaks when individual items are removed from the tree.

### 4. Handling Nested Child Elements with `.closest()`

* If a button contains an icon and text: `<button><svg>...</svg> <span>Submit</span></button>`.
* Clicking the icon sets `event.target` to `<svg>` or `<path>`, not the `<button>`.
* Relying on `event.target.tagName === 'BUTTON'` will fail.
* **Solution**: Use `event.target.closest('button')` to traverse up from the clicked target to find the intended interactive element within the delegated container.

## Code Snippets / Examples

### 1. Capturing vs. Bubbling Demonstration

```html
<div id="parent" style="padding: 20px; background: #eee;">
  <button id="child">Click Me</button>
</div>
```

```javascript
const parent = document.querySelector("#parent");
const child = document.querySelector("#child");

// Capturing Listener (3rd argument = true)
parent.addEventListener("click", () => {
  console.log("1. Parent Captured (Trickle Down)");
}, true);

// Bubbling Listeners (default / 3rd argument = false)
child.addEventListener("click", (event) => {
  console.log("2. Child Target Phase");
  console.log("target:", event.target.id);               // "child"
  console.log("currentTarget:", event.currentTarget.id); // "child"
});

parent.addEventListener("click", (event) => {
  console.log("3. Parent Bubbled (Bubble Up)");
  console.log("target:", event.target.id);               // "child"
  console.log("currentTarget:", event.currentTarget.id); // "parent"
});

// Output when clicking the button:
// 1. Parent Captured (Trickle Down)
// 2. Child Target Phase
// 3. Parent Bubbled (Bubble Up)
```

### 2. Halting Propagation vs. Immediate Propagation

```javascript
const btn = document.querySelector("#child");

btn.addEventListener("click", (e) => {
  console.log("Handler 1 executed");
  e.stopImmediatePropagation(); // Stops bubbling AND halts subsequent listeners on this element
});

btn.addEventListener("click", () => {
  console.log("Handler 2 executed"); // Will NOT execute due to stopImmediatePropagation!
});

document.body.addEventListener("click", () => {
  console.log("Body clicked"); // Will NOT execute due to propagation halt
});
```

### 3. Production Event Delegation Pattern (Using `.closest()`)

```html
<table id="user-table">
  <tbody>
    <tr data-user-id="101">
      <td>Alice</td>
      <td>
        <button class="btn-delete">
          <svg><path d="..." /></svg>
          <span>Delete</span>
        </button>
      </td>
    </tr>
  </tbody>
</table>
```

```javascript
const table = document.querySelector("#user-table");

// Single listener on parent container handles all actions, including dynamic rows
table.addEventListener("click", (event) => {
  // Find the closest action button regardless of whether user clicked the SVG, span, or button
  const deleteBtn = event.target.closest(".btn-delete");

  // Guard clause: click occurred outside the target interactive element
  if (!deleteBtn || !table.contains(deleteBtn)) return;

  const row = deleteBtn.closest("tr");
  const userId = row.dataset.userId;

  console.log(`Deleting user ID: ${userId}`);
  row.remove();
});
```

## Comparison Matrix: Propagation Methods & Properties

| API | Type | Purpose | Halts Propagation? | Cancels Browser Action? |
| --- | --- | --- | --- | --- |
| **`event.target`** | Property | References where the event originated | N/A | N/A |
| **`event.currentTarget`** | Property | References where the active listener is bound | N/A | N/A |
| **`e.stopPropagation()`** | Method | Stops event traveling to parent/child nodes | **Yes** | No |
| **`e.stopImmediatePropagation()`** | Method | Stops propagation AND sibling listeners on current node | **Yes (Full)** | No |
| **`e.preventDefault()`** | Method | Disables browser default behavior (forms, links) | No | **Yes** |
| **`{ capture: true }`** | Option | Runs listener during phase 1 (capture) | N/A | N/A |

## Related Topics

* [[DOM Event Listeners, Browser Memory Management & Teardown Mechanics]]
* [[The `this` Keyword & Execution Bindings|The this Keyword & Execution Bindings]]
* [[JavaScript Closures. Encapsulation, Currying & Output Puzzles|JavaScript Closures: Encapsulation, Currying & Output Puzzles]]
* [[The Browser Rendering Pipeline. Reflow, Repaint, and Composite|Web Performance: DOM Manipulation, Reflow & Repaint]]

## Tags

#fullstack #interview #javascript #dom #events #event-bubbling #event-capturing #event-delegation

## Revision Checklist

* [ ] Can explain in 60 seconds
* [ ] Can explain trade-offs
* [ ] Can give a real project example
* [ ] Can answer common follow-ups