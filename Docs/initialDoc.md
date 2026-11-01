# MiniJS: A JavaScript-Like Engine in C++ with a Live Visualizer

## 1. Introduction

MiniJS is a small JavaScript-like interpreter that you write yourself in C++.
It comes with a visualizer that shows, step by step, what the engine is doing
while it runs a program.

MiniJS is **not** a copy of Google V8. V8 is a huge production engine with
JIT compilers and a garbage collector. MiniJS follows the same basic idea in a
small form, so you can understand every line of it.

```text
Source code -> Tokens -> AST -> Execution -> Output
```

The visualizer lets you watch each of these arrows happen.

---

## 2. Outline

1. Introduction
2. Outline
3. Problem Statement
4. Motivation
5. Purpose
6. Overview (architecture and V8 comparison)
7. Structure (repository layout)
8. The language MiniJS will support
9. Engine roadmap with matching visual milestones
10. Visualizer design
11. C++ concepts you will practice
12. Testing
13. Scope and non-goals
14. Rules for building it yourself
15. Final Thoughts

---

## 3. Problem Statement

Most developers use JavaScript every day, but treat the engine as a black box.
Reading about V8 is not enough to understand it. Reading a finished tutorial
interpreter does not teach C++ either.

You need a project where:

- you write every part yourself, so the C++ knowledge is truly yours
- the engine is small enough to finish
- the internal steps are visible, so you can see and explain what happens

---

## 4. Motivation

- **Learn C++ by building:** a lexer, parser, and interpreter use almost every
  important C++ feature: classes, enums, smart pointers, `std::variant`,
  containers, recursion, and error handling.
- **Understand JavaScript deeply:** scope, precedence, function calls, and
  `const` behavior become obvious once you implement them.
- **Make it explainable:** a visual demo is easier to understand than a code
  listing. It proves that you know how the engine works.

---

## 5. Purpose

By the end of this project you will have:

1. A working interpreter that runs `.mini` programs from the command line.
2. A browser visualizer that replays any program step by step.
3. A clear mental model of how a JavaScript engine turns text into behavior.
4. Real, practical C++ experience written by you.

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

This table is a simplification. Its purpose is to show where each MiniJS part
fits in the bigger picture.

### 6.3 Visualizer idea (record, then replay)

The engine does not talk to the browser directly. Instead:

1. You run `minijs program.mini --trace trace.json`.
2. The engine runs the program and writes every important event into a JSON file.
3. The React visualizer opens that file and replays the events one by one.

```text
minijs (C++)  --->  trace.json  --->  React visualizer
```

Why this design is good:

- No networking and no extra libraries, so it stays local-first.
- You can step **backward** as easily as forward, because all events are saved.
- The engine and the UI are separate, so you can build them independently.

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
│   └── trace.hpp
├── src/
│   ├── main.cpp
│   ├── lexer.cpp
│   ├── parser.cpp
│   ├── interpreter.cpp
│   ├── environment.cpp
│   ├── value.cpp
│   └── trace.cpp
├── tests/
│   ├── lexer_tests.cpp
│   ├── parser_tests.cpp
│   └── interpreter_tests.cpp
└── visualizer/
    ├── package.json
    └── src/
        ├── App.jsx
        └── panels/
            ├── SourcePanel.jsx
            ├── TokensPanel.jsx
            ├── AstPanel.jsx
            ├── ScopePanel.jsx
            ├── CallStackPanel.jsx
            └── OutputPanel.jsx
```

The structure is a suggestion. Change it when your code needs it.

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

---

## 9. Engine roadmap with matching visual milestones

The rule for every milestone: **build the engine part, make it write trace
events, then show those events in the visualizer.** Do not move on until all
three are working.

### Milestone 1: Lexer + Tokens view

**Engine**

- Set up CMake and a `main.cpp` that reads a file path from the command line.
- Define `TokenType` and `Token` (type, text, line, column).
- Scan numbers, strings, identifiers, keywords, operators, and punctuation.
- Skip whitespace and `//` comments.
- Report an error for unknown characters and unterminated strings.

**Visualizer**

- Create the React app (Vite).
- Load `trace.json` and show the source code next to a token table.
- Clicking a token highlights its position in the source.

**Done when:** `let total = 10 + 20;` prints and shows the tokens
`LET, IDENTIFIER, EQUAL, NUMBER, PLUS, NUMBER, SEMICOLON, EOF`.

### Milestone 2: Parser + AST view

**Engine**

- Define AST nodes for expressions and statements.
- Write a recursive-descent parser with one function per grammar rule.
- Handle precedence and parentheses: `10 + 20 * 2` must multiply first.
- Report syntax errors with line and column. Never crash.
- Write the AST into the trace as a tree.

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

**Done when:** a block can read an outer variable, and the outer code
cannot read a variable declared inside the block. The visualizer shows this.

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

### Milestone 6: Polish

- Better error messages for all three error types (lexical, syntax, runtime).
- Show errors in the visualizer at the correct source line.
- Add a dropdown of example programs.
- Write a README with build steps, supported syntax, and known limits.
- Test a clean build on a fresh clone.

### Optional extensions (only after Milestone 6)

- **Bytecode compiler + stack VM:** compile the AST to instructions, then run
  them. The visualizer can show the instruction list and the operand stack.
  This is the closest idea to V8's Ignition.
- **Closures:** let a function keep access to its creation scope.
- **WebAssembly build:** compile the engine with Emscripten so the browser
  runs it directly with no trace file.
- **Benchmarks:** measure the tree-walk interpreter against the VM. Report real
  numbers only.

---

## 10. Visualizer design

### 10.1 Panels

| Panel | Shows | Appears in |
|-------|-------|------------|
| Source | code with the current line highlighted | Milestone 1 |
| Tokens | token list | Milestone 1 |
| AST | tree with the active node highlighted | Milestone 2 |
| Output | console output so far | Milestone 3 |
| Scope | scope chain and variable values | Milestone 4 |
| Call stack | active function calls | Milestone 5 |

### 10.2 Trace event format

Each event is one JSON object in a list. Keep the shape simple and similar
across events.

```json
{ "step": 0, "phase": "lex", "event": "token", "type": "LET", "text": "let", "line": 1, "col": 1 }
{ "step": 7, "phase": "parse", "event": "node", "id": 3, "kind": "BinaryExpression", "parent": 2, "line": 1 }
{ "step": 12, "phase": "exec", "event": "eval_start", "node": 3 }
{ "step": 13, "phase": "exec", "event": "eval_end", "node": 3, "value": 40 }
{ "step": 14, "phase": "exec", "event": "declare", "name": "total", "value": 50, "scope": 1 }
{ "step": 15, "phase": "exec", "event": "call", "name": "factorial", "args": [5], "depth": 1 }
{ "step": 20, "phase": "exec", "event": "output", "text": "120" }
```

### 10.3 How the engine writes the trace

Use a `TraceSink` interface that the lexer, parser, and interpreter call at
important moments.

- With `--trace`, the sink collects events and writes JSON at the end.
- Without `--trace`, the sink does nothing, so normal runs stay clean and fast.

Write the JSON output by hand with `std::string` or `std::ostringstream`. It is
a good C++ exercise, and the format is small.

### 10.4 Stepping logic in the UI

- Keep one array of events and one number: the current step.
- Everything on screen is calculated from events `0..currentStep`.
- Forward, back, and jump-to-step are then just changes to that one number.

---

## 11. C++ concepts you will practice

| Milestone | C++ concepts |
|-----------|--------------|
| 1 | `enum class`, `struct`, `std::string`, `std::string_view`, `std::vector`, `std::ifstream`, `switch`, CMake basics |
| 2 | `std::unique_ptr`, class inheritance with `virtual` (or `std::variant`), recursion, exceptions for syntax errors, `const` correctness |
| 3 | `std::variant`, `std::visit`, operator overloading (optional), interface classes (`TraceSink`) |
| 4 | `std::unordered_map`, `std::shared_ptr` for parent scopes, RAII for scope push/pop |
| 5 | call stack design, using an exception or return signal to unwind on `return`, references vs copies |
| 6 | move semantics, `std::optional`, unit tests, sanitizers (`-fsanitize=address,undefined`) |

Pick **one** AST style (class hierarchy or `std::variant`) and keep it for the
whole project.

---

## 12. Testing

Test behavior, not just compilation. Keep examples in `examples/` and
tests in `tests/`.

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
| Trace | the event list for a tiny program matches the expected output |

Start with a simple custom test runner. Move to GoogleTest only if you want to.

---

## 13. Scope and non-goals

**In scope**

- A small JavaScript-like language
- Handwritten lexer, parser, and tree-walk interpreter in C++
- Variables, scope, control flow, functions
- Clear errors, tests, and documentation
- A React visualizer that replays recorded traces

**Not in scope for version 1**

- Full ECMAScript compatibility
- DOM, browser APIs, Node.js APIs
- Objects, arrays, classes, modules
- Promises, async/await, event loop
- JIT compilation and V8-level optimization
- A real garbage collector

Objects and arrays are the natural next step after Milestone 6.

---

## 14. Rules for building it yourself

1. Write every line of the engine yourself. Ask for explanations, not finished code.
2. Do not copy a full interpreter from a tutorial.
3. Finish each milestone's tests before starting the next one.
4. Keep a short notes file: decisions you made and bugs you fixed.
5. Compile often with warnings on (`-Wall -Wextra`) and fix them.
6. When you are stuck, make the smallest possible example and print the tokens, the AST, or the trace.

Learning loop for every feature:

```text
Problem -> Limitation -> Idea -> Implementation -> Trade-off -> Next step
```

---

## 15. Final Thoughts

**What this project really is.** MiniJS is a way to learn how languages work
and to learn C++ at the same time. The visualizer is what makes it special: any
person can open it and understand the engine without reading code.

**What you will be able to explain at the end.**

- Why source text must become tokens, and why tokens are not enough.
- How operator precedence turns into the shape of a tree.
- How an interpreter walks that tree to produce a result.
- Where variables live and how a function call creates a new scope.
- How the same ideas appear, in a much larger form, inside V8.

**Risks and how to avoid them.**

| Risk | How to avoid it |
|------|-----------------|
| Project becomes too big | Follow the milestones in order; do not add features early |
| Visualizer slows down the engine work | Build only the panel for the current milestone |
| Getting lost in C++ details | Use simple code first, then improve it |
| Losing motivation | Every milestone ends with something you can see and demo |

**Your first week.**

1. Create the repository and a working CMake build.
2. Write `Token` and `TokenType`.
3. Make the lexer print tokens for `let total = 10 + 20;`.
4. Write the first trace JSON by hand, and show it in a basic React page.

When those four things work, you already have the full idea of the project in
miniature: engine, trace, and visualizer. The rest is repeating the same
pattern for the parser, the interpreter, scopes, and functions.

**After the project.** Add objects and arrays, then try the bytecode VM. At
that point you can compare your design with the real V8 pipeline and see
clearly what it adds: bytecode, optimization, and memory management.

A project whose internals you can explain and demonstrate is stronger than a
project with many features. Build the foundation first.