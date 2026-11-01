# MiniJS Playground: A JavaScript-Like Engine in C++ with a Live Web Visualizer

## 1. Introduction

MiniJS is a small JavaScript-like interpreter that you write yourself in C++,
line by line. It runs inside a web page, the **MiniJS Playground**:

- On the **left**, a visitor types JavaScript-like code.
- On the **right**, the page shows how the engine processes that code: tokens,
  the syntax tree, step-by-step execution, scopes, the call stack, and output.

MiniJS is **not** a copy of Google V8. V8 is a huge production engine with JIT
compilers and a garbage collector. MiniJS follows the same basic idea in a
small form, so you can understand every line of it.

```text
Source code -> Tokens -> AST -> Execution -> Output
```

The playground lets anyone watch each of these arrows happen.

---

## 2. Outline

1. Introduction
2. Outline
3. Problem Statement
4. Motivation
5. Purpose
6. Overview (architecture, V8 comparison, playground idea)
7. Structure (repository layout)
8. The language MiniJS will support
9. Engine roadmap with matching visual milestones
10. Playground design
11. C++ concepts you will practice
12. Testing
13. Scope and non-goals
14. Rules for building it yourself
15. Final Thoughts

---

## 3. Problem Statement

Most developers use JavaScript every day but treat the engine as a black box.
Reading about V8 is not enough to understand it, and copying a finished
interpreter does not teach C++.

You need a project where:

- you write every line yourself, so the C++ knowledge is truly yours
- the engine is small enough to finish
- anyone can type code and see the internal steps live, with no installation

---

## 4. Motivation

- **Learn C++ by building:** a lexer, parser, and interpreter use almost every
  important C++ feature: classes, enums, smart pointers, `std::variant`,
  containers, recursion, and error handling.
- **Understand JavaScript deeply:** scope, precedence, function calls, and
  `const` behavior become obvious once you implement them.
- **Make it explainable:** a live demo proves you understand the engine far
  better than a code listing does.
- **Make it shareable:** a link to a working playground is easy to show to
  anyone.

---

## 5. Purpose

By the end of this project you will have:

1. A working interpreter that runs `.mini` programs from the command line.
2. A public web page where anyone can write code and watch the engine process
   it, running in the browser through WebAssembly.
3. A clear mental model of how a JavaScript engine turns text into behavior.
4. Real C++ experience, written entirely by you.

---

## 6. Overview

### 6.1 Engine pipeline

```text
Source Code
    |
    v
Lexer ------------> Token list
    |
    v
Parser -----------> AST (tree)
    |
    v
Interpreter (tree-walk)
    |
    +--> Environment (variables and scopes)
    +--> Call stack (function calls)
    +--> Console output
```

### 6.2 How MiniJS maps to V8 (simplified)

| Idea | V8 | MiniJS |
|------|----|--------|
| Read text into tokens | Scanner | Lexer |
| Build program structure | Parser | Recursive-descent parser |
| Tree of the program | AST | AST |
| Run the program | Ignition bytecode interpreter | Tree-walk interpreter |
| Faster execution for hot code | JIT compilers (e.g. TurboFan) | Not included |
| Memory cleanup | Garbage collector | C++ smart pointers |

This table is a simplification. It shows where each MiniJS part fits in the
bigger picture.

### 6.3 How the playground works

A browser cannot run C++ directly, so the engine is compiled to
**WebAssembly (WASM)** with Emscripten. The same C++ code then runs in the
visitor's browser, with no server.

```text
Editor (left)
    |  source text
    v
Web Worker --> MiniJS engine (WASM) --> trace JSON
                                              |
                                              v
                                   Visualizer panels (right)
```

Key points:

- The engine records every important event as JSON (the **trace**).
- The React page only reads the trace and draws it. It never runs the code itself.
- The engine runs inside a **Web Worker**, so an infinite loop cannot freeze the page.
- Because the trace is saved, stepping backward is as easy as stepping forward.
- No server means no hosting cost, no security risk from running strangers' code
  on your machine, and a simple static deployment (for example GitHub Pages).

### 6.4 Build order

1. Build the engine for the **command line** first. It can write `trace.json`.
2. Test the visualizer by loading that file.
3. Compile the engine to WASM and connect it to the live editor.

This way you debug C++ problems with normal tools, and browser problems only
appear once the engine already works.

---

## 7. Structure

```text
minijs/
├── CMakeLists.txt
├── README.md
├── examples/
│   ├── hello.mini
│   ├── arithmetic.mini
│   ├── scope.mini
│   ├── loops.mini
│   └── functions.mini
├── include/minijs/
│   ├── token.hpp
│   ├── lexer.hpp
│   ├── ast.hpp
│   ├── parser.hpp
│   ├── value.hpp
│   ├── environment.hpp
│   ├── interpreter.hpp
│   ├── trace.hpp
│   └── engine.hpp
├── src/
│   ├── lexer.cpp
│   ├── parser.cpp
│   ├── interpreter.cpp
│   ├── environment.cpp
│   ├── value.cpp
│   ├── trace.cpp
│   ├── engine.cpp
│   ├── main.cpp
│   └── wasm_api.cpp
├── tests/
│   ├── lexer_tests.cpp
│   ├── parser_tests.cpp
│   ├── interpreter_tests.cpp
│   └── trace_tests.cpp
└── playground/
    ├── package.json
    ├── public/
    │   └── (compiled WASM files appear here)
    └── src/
        ├── App.jsx
        ├── engineWorker.js
        └── panels/
            ├── EditorPanel.jsx
            ├── TokensPanel.jsx
            ├── AstPanel.jsx
            ├── ScopePanel.jsx
            ├── CallStackPanel.jsx
            ├── OutputPanel.jsx
            └── Controls.jsx
```

Important design rule from day one: split the C++ code into two parts.

- **Core library** (`engine.cpp` and everything it uses): takes source text,
  returns a result object with the output, the trace, and any errors. It never
  prints, never reads files, and never calls `exit`.
- **Thin wrappers**: `main.cpp` (command line) and `wasm_api.cpp` (browser) only
  call the core library.

If you follow this rule, adding WebAssembly later is a small step.

This structure is a suggestion. Change it when your code needs it.

---

## 8. The language MiniJS will support

Keep version 1 small. Anything not listed here is out of scope.

| Area | Supported |
|------|-----------|
| Values | numbers, strings, booleans, `null` |
| Declarations | `let`, `const` |
| Operators | `+ - * / %`, `== !=`, `< > <= >=`, `&& \|\| !`, `=` |
| Statements | expression, block `{ }`, `if / else`, `while`, `return` |
| Functions | declaration, parameters, return values, recursion |
| Built-in | `console.log(...)` |
| Comments | `// single line` |

Target programs:

```javascript
let price = 100;
let quantity = 3;
console.log(price * quantity);
```

```javascript
let score = 82;
if (score >= 40) {
    console.log("Pass");
} else {
    console.log("Fail");
}
```

```javascript
function factorial(n) {
    if (n <= 1) {
        return 1;
    }
    return n * factorial(n - 1);
}
console.log(factorial(5));
```

These programs define the goal. They are not a claim that they already work.
The playground should show this supported list next to the editor, so visitors
know what to try.

---

## 9. Engine roadmap with matching visual milestones

For every milestone: **build the engine part, make it write trace events, then
show those events in the visualizer.** Do not move on until all three work.

### Milestone 1: Lexer + Tokens view

**Engine**

- Set up CMake with two targets: the core library and the command-line program.
- Define `TokenType` and `Token` (type, text, line, column).
- Scan numbers, strings, identifiers, keywords, operators, and punctuation.
- Skip whitespace and `//` comments.
- Report an error for unknown characters and unterminated strings.

**Visualizer**

- Create the React app (Vite).
- Load `trace.json` and show the source next to a token table.
- Clicking a token highlights its position in the source.

**Done when:** `let total = 10 + 20;` shows the tokens
`LET, IDENTIFIER, EQUAL, NUMBER, PLUS, NUMBER, SEMICOLON, EOF`.

### Milestone 2: Parser + AST view

**Engine**

- Define AST nodes for expressions and statements.
- Write a recursive-descent parser with one function per grammar rule.
- Handle precedence and parentheses: `10 + 20 * 2` must multiply first.
- Report syntax errors with line and column. Never crash.
- Write the AST into the trace as a tree.
- If parsing fails, keep the tokens already in the trace, so the visualizer can
  still show them.

**Visualizer**

- Draw the AST as an expandable tree.
- Hovering a node highlights the matching text in the source.

**Done when:** `let result = 10 + 20 * 2;` shows a tree where `*` is below `+`.

### Milestone 3: Expression evaluation + Step view

**Engine**

- Define the runtime `Value` type (number, string, boolean, null).
- Evaluate literals, grouping, arithmetic, comparisons, and logic.
- Implement `console.log`.
- Emit an event when a node starts evaluating and when it finishes with a result.

**Visualizer**

- Add Play, Pause, Step forward, Step back, and a speed control.
- Highlight the AST node being evaluated and show its result.
- Show console output in the output panel.

**Done when:** you can step through `10 + 20 * 2` and see `20 * 2 = 40`
first, then `10 + 40 = 50`.

### Milestone 4: Variables and scope + Scope view

**Engine**

- Build `Environment` with a map of names to values and an optional parent.
- Support declaration, lookup, assignment, and block scopes.
- Make `const` reassignment a runtime error.
- Report undefined variables clearly.
- Emit events: scope pushed, scope popped, variable declared, variable assigned.

**Visualizer**

- Show the scope chain as stacked boxes, with the current scope on top.
- Flash a variable when it changes.

**Done when:** a block can read an outer variable, the outer code cannot read a
variable declared inside the block, and the visualizer shows this.

### Milestone 5: Control flow and functions + Call stack view

**Engine**

- Implement `if / else` and `while`.
- Implement function declarations, calls, parameters, and `return`.
- Support nested calls and recursion.
- Emit events: function called, function returned.

**Visualizer**

- Show a call-stack panel that grows and shrinks.
- Show each call's arguments and return value.

**Done when:** `factorial(5)` runs, and you can watch the stack grow to five
frames and unwind with the result.

### Milestone 6: Safety and polish (engine)

Visitors will type anything, including infinite loops and broken code. Prepare
the engine for that.

- Add a **step limit**: stop with a clear error after a fixed number of
  evaluation steps (for example, `while (true) {}` must end with an error).
- Add a **call depth limit**: stop runaway recursion before the C++ stack overflows.
- Add a **trace size limit**: stop recording after a fixed number of events and
  tell the user that the trace was cut.
- Improve error messages for lexical, syntax, and runtime errors.
- Test a clean build on a fresh clone, with warnings enabled and sanitizers on
  (`-fsanitize=address,undefined`).

**Done when:** no program a visitor can type crashes the engine or runs forever.

### Milestone 7: WebAssembly bridge

**Engine**

- Install Emscripten.
- In `wasm_api.cpp`, export one function that takes source text and returns the
  result (trace JSON) as a string.
- Compile the core library to `.wasm` and a `.js` loader into `playground/public/`.
- Build the whole thing from one script or CMake preset, so you can repeat it easily.

**Visualizer**

- Load the WASM module inside a Web Worker (`engineWorker.js`).
- Send source text to the worker and receive the trace back.
- Terminate and restart the worker if it runs longer than a time limit.

**Done when:** a small button in the page runs a hard-coded program in the
browser and shows its trace.

### Milestone 8: Live playground

**Page**

- Split-screen layout: editor on the left, visualizer panels on the right.
- Re-run the engine automatically a short time after the user stops typing
  (debounce, for example about half a second), or with a Run button.
- Show errors next to the right source line, and keep partial results (tokens,
  AST) visible when possible.
- Add a dropdown with the example programs.
- Show a list of supported syntax.
- Make the layout work on a narrow screen (panels stack vertically).
- Deploy as a static site.

**Done when:** a stranger opens the link, types a small program, and understands
how the engine processed it without any explanation from you.

### Optional extensions (only after Milestone 8)

- **Bytecode compiler and stack VM:** the closest idea to V8's Ignition.
  The playground can show the instruction list and the operand stack.
- **Closures:** let a function keep access to its creation scope.
- **Objects and arrays:** the natural next language feature.
- **Syntax highlighting** in the editor (a CodeMirror wrapper is the easiest
  React-friendly option).
- **Benchmarks:** compare the tree-walk interpreter with the VM. Report real
  numbers only.

---

## 10. Playground design

### 10.1 Layout

```text
+--------------------------+-----------------------------------+
|                          |  Tokens | AST | Scope | Stack     |
|   Code editor            |-----------------------------------|
|   (user types here)      |   Active panel with highlights    |
|                          |-----------------------------------|
|                          |   Output / Errors                 |
+--------------------------+-----------------------------------+
|  [Play] [Pause] [Step back] [Step forward]   Speed   Examples |
+--------------------------------------------------------------+
```

### 10.2 Panels

| Panel | Shows | Appears in |
|-------|-------|------------|
| Editor / Source | code with the current line highlighted | Milestone 1 |
| Tokens | token list | Milestone 1 |
| AST | tree with the active node highlighted | Milestone 2 |
| Output | console output so far | Milestone 3 |
| Scope | scope chain and variable values | Milestone 4 |
| Call stack | active function calls | Milestone 5 |
| Errors | lexical, syntax, and runtime errors | Milestone 6 |

### 10.3 Trace event format

Each event is one JSON object in a list. Keep the shapes simple and similar.

```json
{ "step": 0, "phase": "lex", "event": "token", "type": "LET", "text": "let", "line": 1, "col": 1 }
{ "step": 7, "phase": "parse", "event": "node", "id": 3, "kind": "BinaryExpression", "parent": 2, "line": 1 }
{ "step": 12, "phase": "exec", "event": "eval_start", "node": 3 }
{ "step": 13, "phase": "exec", "event": "eval_end", "node": 3, "value": 40 }
{ "step": 14, "phase": "exec", "event": "declare", "name": "total", "value": 50, "scope": 1 }
{ "step": 15, "phase": "exec", "event": "call", "name": "factorial", "args": [5], "depth": 1 }
{ "step": 20, "phase": "exec", "event": "output", "text": "120" }
{ "step": 21, "phase": "exec", "event": "error", "kind": "runtime", "message": "Undefined variable x", "line": 4, "col": 9 }
```

### 10.4 How the engine writes the trace

Use a `TraceSink` interface that the lexer, parser, and interpreter call at
important moments.

- In the playground and with `--trace`, the sink collects events and returns JSON.
- When tracing is off, the sink does nothing, so normal runs stay fast.

Write the JSON by hand with `std::string` or `std::ostringstream`. It is a good
C++ exercise, and the format is small. Remember to escape quotes and newlines in
strings.

### 10.5 Stepping logic in the UI

- Keep one array of events and one number: the current step.
- Everything on screen is calculated from events `0..currentStep`.
- Forward, back, and jump-to-step only change that one number.
- After each run, show the finished result first. Play replays from step 0.

### 10.6 Frontend approach

Use React with simple state (`useState`, `useEffect`) and plain components. Avoid
direct DOM manipulation. Start the editor as a plain `<textarea>`, and upgrade
to CodeMirror only after everything else works.

---

## 11. C++ concepts you will practice

| Milestone | C++ concepts |
|-----------|--------------|
| 1 | `enum class`, `struct`, `std::string`, `std::string_view`, `std::vector`, `std::ifstream`, `switch`, CMake targets (library plus executable) |
| 2 | `std::unique_ptr`, class inheritance with `virtual` (or `std::variant`), recursion, exceptions for syntax errors, `const` correctness |
| 3 | `std::variant`, `std::visit`, interface classes (`TraceSink`), string building |
| 4 | `std::unordered_map`, `std::shared_ptr` for parent scopes, RAII for scope push/pop |
| 5 | call stack design, unwinding on `return`, references vs copies |
| 6 | counters and limits, `std::optional`, move semantics, sanitizers, warnings |
| 7 | `extern "C"` functions, memory ownership across the C++/JavaScript boundary, Emscripten build flags |
| 8 | designing a stable result type (`output`, `trace`, `errors`) that the page can rely on |

Pick **one** AST style (class hierarchy or `std::variant`) and keep it for the
whole project.

---

## 12. Testing

Test behavior, not just compilation. Keep examples in `examples/` and tests in
`tests/`.

| Area | Example test |
|------|--------------|
| Lexer | keywords, identifiers, strings, operators, bad characters |
| Parser | `10 + 20 * 2`, parentheses, missing expression |
| Arithmetic | correct results and division behavior |
| Variables | declare, read, reassign, `const` error |
| Scope | inner access, outer isolation |
| Conditions | true and false branches |
| Loops | repeats and ends correctly |
| Functions | parameters, return values, recursion |
| Errors | undefined variable, invalid operation |
| Limits | infinite loop, deep recursion, huge output |
| Trace | the event list for a tiny program matches the expected output |
| Playground | the page shows the same output as the command-line program for each example |

Start with a simple custom test runner. Move to GoogleTest only if you want to.

---

## 13. Scope and non-goals

**In scope**

- A small JavaScript-like language
- Handwritten lexer, parser, and tree-walk interpreter in C++
- Variables, scope, control flow, functions
- Clear errors, limits, tests, and documentation
- A WebAssembly build and a live React playground

**Not in scope for version 1**

- Full ECMAScript compatibility
- DOM, browser APIs, Node.js APIs
- Objects, arrays, classes, modules
- Promises, async/await, event loop
- JIT compilation and V8-level optimization
- A real garbage collector
- A backend server

---

## 14. Rules for building it yourself

1. Write every line yourself, including the C++ engine, tests, and the React page.
   Use help only for explanations, ideas, and code review of what you wrote.
2. Do not copy a full interpreter from a tutorial.
3. Finish each milestone's tests before starting the next one.
4. Keep a short notes file with decisions you made and bugs you fixed.
5. Compile often with warnings on (`-Wall -Wextra`) and fix them.
6. When you are stuck, make the smallest possible example and print the tokens,
   the AST, or the trace.
7. Keep the core library clean: no printing, no file access, no `exit`.

Learning loop for every feature:

```text
Problem -> Limitation -> Idea -> Implementation -> Trade-off -> Next step
```

---

## 15. Final Thoughts

**What this project really is.** MiniJS is a way to learn how languages work
and to learn C++ at the same time. The playground makes it special: a stranger
can type code and see the engine work, without reading any C++.

**What you will be able to explain at the end.**

- Why source text must become tokens, and why tokens are not enough.
- How operator precedence turns into the shape of a tree.
- How an interpreter walks that tree to produce a result.
- Where variables live and how a function call creates a new scope.
- How C++ code is compiled to WebAssembly and used from a web page.
- How the same ideas appear, in a much larger form, inside V8.

**Risks and how to avoid them.**

| Risk | How to avoid it |
|------|-----------------|
| Project becomes too big | Follow the milestones in order; do not add features early |
| Playground work slows down the engine | Build the engine for the command line first; add WASM only at Milestone 7 |
| User code freezes the page | Step, depth, and trace limits, plus a Web Worker with a timeout |
| WASM build problems | Keep the core library free of files and printing from the start |
| Getting lost in C++ details | Use simple code first, then improve it |
| Losing motivation | Every milestone ends with something you can see and demo |

**Your first week.**

1. Create the repository with a working CMake build (core library plus command-line program).
2. Write `Token` and `TokenType`.
3. Make the lexer print tokens for `let total = 10 + 20;`.
4. Write the first trace JSON by hand, and show it in a basic React page.

When those four things work, you already have the full idea in miniature:
engine, trace, and visualizer. The rest is repeating the same pattern for the
parser, interpreter, scopes, and functions, and then moving it into the browser.

**After the project.** Add objects and arrays, then try the bytecode VM. At that
point you can compare your design with the real V8 pipeline and see clearly what
it adds: bytecode, optimization, and memory management.

A project whose internals you can explain and demonstrate is stronger than a
project with many features. Build the foundation first.