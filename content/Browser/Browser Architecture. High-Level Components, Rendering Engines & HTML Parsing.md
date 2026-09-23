# Browser Architecture: High-Level Components, Rendering Engines & HTML Parsing

## Key Concepts

> [!summary] High-Level Browser Component Architecture
> A modern web browser is a distributed modular operating system composed of seven core subsystems:
> 1. **User Interface (UI)**: The browser chrome (address bar, back/forward buttons, bookmarks, tabs, download manager). Everything except the window displaying the requested page.
> 2. **Browser Engine**: Marshals actions between the UI and the Rendering Engine (handles tab management, high-level navigation, and session restoration).
> 3. **Rendering Engine (Layout Engine)**: Parses HTML/CSS, calculates layout geometry, and paints visual pixels to the screen (e.g., Blink in Chrome/Edge, Gecko in Firefox, WebKit in Safari).
> 4. **JavaScript Engine**: Parses, JIT-compiles, and executes JavaScript code and manages heap memory/garbage collection (e.g., V8 in Chrome, SpiderMonkey in Firefox, JavaScriptCore in Safari).
> 5. **Networking Layer**: Implements protocol stacks (HTTP/1.1, HTTP/2, HTTP/3, TLS, DNS, TCP/UDP), connection pooling, socket caching, and proxy resolution.
> 6. **UI Backend**: Platform-agnostic drawing primitives (windows, combo boxes, fonts) delegating to host OS graphics APIs (DirectX, Skia, CoreGraphics).
> 7. **Data Storage Layer**: Persistence layer managing disk and memory caches (`localStorage`, `sessionStorage`, `IndexedDB`, `Cache API`, Cookies, OPFS).


> [!abstract] Multi-Process vs. Single-Process Architecture
> Modern browsers (Chromium architecture) isolate responsibilities into sandboxed operating system processes to guarantee security, fault isolation, and stability:
> * **Browser Process**: Privileged coordinator managing UI, tab orchestration, disk I/O, and network initialization.
> * **Renderer Process**: Sandboxed per-tab (or per-site-instance via **Site Isolation**) running Blink + V8. If a malicious script crashes or exploits a page, it cannot compromise the host OS or read data from adjacent tabs.
> * **GPU Process**: Handles isolated GPU rasterization and compositor frame rendering.
> * **Network Process**: Handles all network I/O, TLS termination, and disk cache management in modern Chrome (separated from the main browser process).


> [!danger] Why Standard Parsers (LL/LR) Fail for HTML
> Traditional programming languages use deterministic **Context-Free Grammars (CFG)** parsed by standard **LL(k)** or **LR(k)** parsers. HTML **cannot** be parsed by traditional parsers because:
> 1. **Fault Tolerance (Tag Soup)**: The web cannot break when authors omit closing tags (`<p>Hello <p>World`).
> 2. **Dynamic Re-entrancy (`document.write`)**: JavaScript can execute during parsing and inject new markup directly into the unparsed input stream, mutating the tokenizer's current position.
> 3. **Context Sensitivity**: Certain tags change how following characters are interpreted (e.g., `<script>`, `<style>`, `<iframe>` switch the tokenizer into CDATA or raw text modes).


> [!tip] HTML5 Parsing Specification (Tokenization $\to$ Tree Construction)
> The HTML5 specification standardizes parsing via a deterministic, two-stage state machine:
> 4. **Tokenization (Lexical Analysis)**: Converts character streams into structured tokens: `StartTag`, `EndTag`, `Character`, `Comment`, or `DOCTYPE`.
> 5. **Tree Construction**: Consumes tokens using a **State Machine** paired with an **Open Elements Stack** and an **Active Formatting Elements List** to enforce HTML semantics and build the DOM tree.


---

## Common Interview Questions

* "Name the main structural components of a web browser and describe the responsibility of each."
* "Why does Chrome use a multi-process architecture instead of running tabs in lightweight threads?"
* "Why can't standard LL(k) or LR(k) compiler tools (like Lex/Yacc or Bison) parse HTML?"
* "Walk me through the two stages of the HTML5 parsing algorithm: Tokenization and Tree Construction."
* "What is the 'Active Formatting Elements List', and how does the adoption agency algorithm handle misnested tags like `<b>1<i>2</b>3</i>`?"
* "How does the Speculative Pre-parser (Preload Scanner) prevent network deadlocks during HTML parsing?"
* "How do the Rendering Engine and the JavaScript Engine coordinate across the browser boundary?"

---

## Deep Dive & Talking Points

### 1. Multi-Process Architecture & Site Isolation

* **Process Isolation**: In early browsers, a single unhandled exception or memory leak in one tab crashed the entire browser window.
* **Chromium Process Model**:
* Each tab typically runs in its own sandboxed **Renderer Process**.
* **Site Isolation (Out-of-Process Iframes / OOPIF)**: Cross-origin iframes (`<iframe src="[https://bank.com](https://bank.com)">`) run in a *separate OS process* from the parent page (`[https://evil.com](https://evil.com)`), preventing microarchitectural side-channel attacks like **Spectre** from reading cross-origin process memory.

### 2. The HTML5 Parsing Stages

The parsing pipeline operates as a streaming state machine:

```text
Raw Bytes ──(Encoding Sniffing)──> Unicode Characters
                                           │
                                    [ Tokenizer ]  <── (document.write re-entry)
                                           │
                                      HTML Tokens (StartTag, EndTag, Character, etc.)
                                           │
                                [ Tree Construction ]
                                    ┌──────┴──────┐
                        [Open Elements Stack]  [Active Formatting List]
                                           │
                                       DOM Tree
```

#### Stage A: Tokenization

* The tokenizer begins in the **Data State**.
* Consuming a `<` transitions it to the **Tag Open State**.
* Consuming an alphanumeric character creates a `StartTag` token and transitions to the **Tag Name State**.
* Consuming `/` transitions to the **End Tag Open State**.
* When reaching `>`, the current token is emitted and the state returns to **Data State**.

#### Stage B: Tree Construction

* As tokens are emitted, the tree constructor inserts matching `Node` objects into the **DOM Tree**.
* It maintains the **Stack of Open Elements** to track hierarchical nesting (e.g., `html` $\to$ `body` $\to$ `div`).
* It applies **HTML Correction Rules**:
* If a `<tr>` token appears outside a `<table>`, the parser automatically injects `<table>` and `<tbody>` ancestor nodes.
* If a `<p>` token arrives while another `<p>` is on the open stack, the open `<p>` is implicitly closed before opening the new one.

### 3. Misnested Markup & The Adoption Agency Algorithm

* Consider invalid HTML: `<b>1<i>2</b>3</i>`.
* Strict XML parsers fail with fatal syntax errors.
* The HTML5 parser uses the **Active Formatting Elements List** to repair the hierarchy using the **Adoption Agency Algorithm**:
1. `<b>` enters formatting list and open stack.
2. `<i>` enters formatting list and open stack.
3. `</b>` arrives: The parser notices `<b>` is *not* at the top of the stack (`<i>` is).
4. The algorithm "adopts" the nodes, splitting and restructuring the tree into:
`<b>1<i>2</i></b><i>3</i>`.
5. The resulting DOM is semantically valid without halting execution.

### 4. Speculative Parsing (Preload Scanner)

* **The Bottleneck**: When the HTML parser hits a synchronous `<script src="app.js"></script>`, it must halt tokenization and DOM construction completely until `app.js` is downloaded and executed.
* **The Optimization**: Modern engines run a secondary background thread called the **Preload Scanner** (or Speculative Parser).
* While the main parser is blocked on JS execution, the Preload Scanner scans ahead through the remaining unparsed HTML stream, discovers referenced external resources (`<link rel="stylesheet">`, `<script>`, `<img src="...">`), and dispatches network requests immediately in parallel.

### 5. Bridging the Rendering Engine & JS Engine (DOM Binding Overhead)

* The Rendering Engine (C++) and the JS Engine (V8) maintain separate memory heaps.
* Accessing the DOM from JavaScript (`document.getElementById()`, `element.innerHTML`) crosses the **language boundary** using C++ wrapper objects.
* Excessive DOM traversal or reading layout properties introduces overhead due to context switches between V8 and Blink and subsequent layout invalidation.

---

## Code Snippets / Architectural Flow

### 1. Conceptual State Machine of the HTML5 Tokenizer

```javascript
// Simplified mental model of the Tokenizer state machine
function tokenizeHTML(stream) {
  let state = "DATA";
  let currentToken = null;
  const tokens = [];

  for (let i = 0; i < stream.length; i++) {
    const char = stream[i];

    switch (state) {
      case "DATA":
        if (char === "<") {
          state = "TAG_OPEN";
        } else {
          tokens.push({ type: "CHARACTER", data: char });
        }
        break;

      case "TAG_OPEN":
        if (char === "/") {
          state = "END_TAG_OPEN";
        } else if (/[a-zA-Z]/.test(char)) {
          currentToken = { type: "START_TAG", tagName: char.toLowerCase() };
          state = "TAG_NAME";
        }
        break;

      case "TAG_NAME":
        if (char === ">") {
          tokens.push(currentToken);
          state = "DATA";
        } else {
          currentToken.tagName += char.toLowerCase();
        }
        break;

      case "END_TAG_OPEN":
        if (/[a-zA-Z]/.test(char)) {
          currentToken = { type: "END_TAG", tagName: char.toLowerCase() };
          state = "TAG_NAME";
        }
        break;
    }
  }

  return tokens;
}

console.log(tokenizeHTML("<p>Hello</p>"));
// [
//   { type: 'START_TAG', tagName: 'p' },
//   { type: 'CHARACTER', data: 'H' },
//   { type: 'CHARACTER', data: 'e' }, ...
//   { type: 'END_TAG', tagName: 'p' }
// ]
```

---

### 2. Demonstrating Parser Interruption via `document.write`

```html
<!DOCTYPE html>
<html>
<body>
  <div>Before script</div

  <script>
    // document.write re-enters the tokenizer, injecting characters directly
    // into the remaining input byte stream during tree construction!
    document.write("<span>Injected inline during parsing</span>");
  </script

  <div>After script</div>
</body>
</html>
```

* When `document.write()` executes, the parser immediately parses the injected text before reading the next byte of the original HTML document.
* Modern browsers heavily discourage or block `document.write` on slow 2G/3G connections because it interferes with the Preload Scanner and degrades page load speed.

---

## Comparison Matrix: Major Browser Engines

| Browser | Browser Shell / UI | Rendering / Layout Engine | JavaScript Engine | Open Source Base |
| --- | --- | --- | --- | --- |
| **Google Chrome** | Chromium UI | **Blink** (forked from WebKit) | **V8** | Chromium |
| **Mozilla Firefox** | Firefox UI (Gecko) | **Gecko** | **SpiderMonkey** | Mozilla Public |
| **Apple Safari** | Safari Cocoa UI | **WebKit** | **JavaScriptCore (Nitro)** | WebKit Open Source |
| **Microsoft Edge** | Chromium UI | **Blink** | **V8** | Chromium |
| **Node.js / Deno** | Headless CLI / Server | *None* (No DOM / Layout) | **V8** | V8 Engine Core |

---

## Comparison Matrix: Browser Subsystems

| Subsystem | Thread / Process Affinity | Core Input | Core Output | Primary Bottleneck |
| --- | --- | --- | --- | --- |
| **Networking** | Network Process | URLs, headers, sockets | Raw byte streams, cached files | Round-Trip Time (RTT), bandwidth, DNS lag |
| **HTML Parser** | Main Renderer Thread | Raw character stream | **Document Object Model (DOM)** | Synchronous blocking `<script>` tags |
| **CSS Parser** | Main Renderer Thread | CSS rules & stylesheets | **CSS Object Model (CSSOM)** | Deep specificity, excessive `@import` chains |
| **Layout Engine** | Main Renderer Thread | Render Tree (DOM + CSSOM) | Bounding boxes & coordinates | Forced synchronous layouts (reflow thrashing) |
| **JS Engine (V8)** | Main Renderer Thread | JS Source / AST | Native machine code / Bytecode | Heavy main-thread execution, long tasks (>50ms) |
| **Compositor** | GPU / Compositor Threads | Layer textures | Screen pixels | VRAM limits, excessive layer promotion |

---

## Related Topics

* [[What Happens When You Enter a URL in the Browser. The End-to-End Lifecycle|What Happens When You Enter a URL in the Browser: The End-to-End Lifecycle]]
* [[V8 Engine Architecture. Parsing, JIT Compilation & Execution Pipeline|V8 Engine Architecture: Parsing, JIT Compilation & Execution Pipeline]]
* [[DOM Event Propagation. Bubbling, Capturing & Event Delegation|DOM Event Propagation: Bubbling, Capturing & Event Delegation]]
* [[The Browser Rendering Pipeline. Reflow, Repaint, and Composite|Web Performance: DOM Manipulation, Reflow & Repaint]]

---

## Tags

#fullstack #interview #browser-internals #html-parsing #blink #v8 #renderer-process #multi-process-architecture #dom

---

## Revision Checklist

* [ ] Can explain in 60 seconds
* [ ] Can explain trade-offs
* [ ] Can give a real project example
* [ ] Can answer common follow-ups
