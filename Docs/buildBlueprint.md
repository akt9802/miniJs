# MiniJS Build Blueprint: Ten Chapters from Empty Folder to Live Playground

This document is the step-by-step build plan for MiniJS. `initialDoc.md` explains
**what** MiniJS is and **why** it exists. This document explains **how** to build
it: which files to create, which types and functions to write, which algorithm
each function follows, what to test, and how you know a chapter is finished.

---

## 0. How to use this blueprint

### 0.1 The rule about code in this document

`initialDoc.md` says: *write every line yourself*. This blueprint respects that.

- **Given in full:** build files, file layout, type and function *declarations*
  (the shape of the code), grammar rules, JSON formats, and test cases.
- **Given as steps or pseudocode:** function *bodies*. You turn each step into
  C++ or JavaScript yourself.
- **Never given:** a finished, copy-paste implementation of the lexer, parser, or
  interpreter.

When a declaration below says `// TODO body`, that body is your work.

### 0.2 The ten chapters at a glance

| Ch | Title | Engine work | Visualizer work | `initialDoc` milestone |
|----|-------|-------------|-----------------|------------------------|
| 1 | Foundation | CMake, core/wrapper split, `EngineResult`, test runner | none | start of M1 |
| 2 | Lexer | `Token`, `TokenType`, `Lexer` | none | M1 |
| 3 | Trace system + first page | `TraceSink`, JSON writer, `--trace` flag | Vite app, Source view, Tokens panel | M1 |
| 4 | AST + Parser | AST nodes, recursive-descent parser, syntax errors | AST panel | M2 |
| 5 | Values + Expressions | `Value`, expression evaluation, `console.log` | Step controls, Output panel | M3 |
| 6 | Variables + Scope | `Environment`, `let`/`const`, blocks | Scope panel | M4 |
| 7 | Control flow + Functions | `if`, `while`, functions, `return`, recursion | Call stack panel | M5 |
| 8 | Safety + Errors | step/depth/trace/output limits, error polish | Errors panel | M6 |
| 9 | WebAssembly bridge | Emscripten build, `wasm_api.cpp` | Web Worker, run in browser | M7 |
| 10 | Live playground | none (engine is done) | Editor, live re-run, examples, deploy | M8 |

### 0.3 How every chapter is written

Each chapter has the same parts:

1. **Goal**: one sentence.
2. **Concepts**: the C++ (or web) ideas you practice.
3. **Files**: what you create or change.
4. **Build steps**: numbered steps with declarations and algorithms.
5. **Trace events**: new JSON events the engine emits (from Chapter 3 on).
6. **Visualizer work**: what to add to the page (from Chapter 3 on).
7. **Tests**: concrete inputs and expected results.
8. **Done when**: the checklist that must pass before you start the next chapter.
9. **Pitfalls**: mistakes that commonly cost hours.

### 0.4 Dependency picture

```text
Ch1 Foundation
  └─> Ch2 Lexer
        └─> Ch3 Trace + page ──────────────────────────────┐
              └─> Ch4 Parser (+ AST panel)                  │
                    └─> Ch5 Expressions (+ steps, output)   │  every chapter from
                          └─> Ch6 Scope (+ scope panel)     │  here on adds trace
                                └─> Ch7 Functions (+ stack) │  events AND a panel
                                      └─> Ch8 Safety        │
                                            └─> Ch9 WASM ───┘
                                                  └─> Ch10 Live playground
```

### 0.5 Global decisions (made once, used everywhere)

These choices are fixed for the whole project, so later chapters never have to
revisit them.

| Decision | Choice | Reason |
|----------|--------|--------|
| C++ standard | C++20 | `std::string_view`, `std::variant`, designated initializers |
| Namespace | `minijs` | avoids name clashes |
| AST style | class hierarchy + `NodeKind` enum + `std::unique_ptr` | easiest to read and debug |
| Runtime value | `std::variant` inside a `Value` class | type-safe, no manual tagging |
| Errors inside the engine | C++ exceptions, caught in `engine.cpp` | simple unwinding through recursion |
| Errors leaving the engine | `std::vector<EngineError>` inside `EngineResult` | never let an exception escape the core |
| `return` handling | an `ExecSignal` return value, not an exception | cheaper, clearer control flow |
| Source positions | `line`, `col` (1-based) **and** byte `offset` | line/col for messages, offset for UI highlights |
| Semicolons | required | simpler parser |
| Equality | `==` is strict (no type coercion), `===` is rejected with a hint | avoids JS coercion rules |
| Division by zero | runtime error | avoids printing `inf` and `nan` |
| Missing return value | `null` (MiniJS has no `undefined`) | one "empty" value only |
| Function hoisting | none: declare a function before calling it | simpler interpreter |
| Trace format | one JSON array of flat event objects | the UI only replays it |

---

## Chapter 1: Foundation

**Goal.** A repository that builds a core library, a command-line program, and a
test program, with the core/wrapper split already in place.

**Concepts.** CMake targets, headers vs sources, `#pragma once`, namespaces,
`std::ifstream`, exit codes, a tiny test framework with macros and static
registration.

### Files

```text
minijs/
├── CMakeLists.txt
├── .gitignore
├── README.md
├── NOTES.md                    (your decision and bug log)
├── examples/
│   ├── hello.mini
│   └── hello.expected
├── include/minijs/engine.hpp
├── src/engine.cpp
├── src/main.cpp
└── tests/
    ├── test.hpp
    ├── test_main.cpp
    └── engine_tests.cpp
```

### Build steps

**1.1 Create `CMakeLists.txt`.** This is infrastructure, so the full shape is
given. You will add source files to it in each chapter.

```cmake
cmake_minimum_required(VERSION 3.20)
project(minijs LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(MINIJS_SANITIZE "Build with AddressSanitizer and UBSan" OFF)
if(MINIJS_SANITIZE)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
endif()

add_library(minijs_core
    src/engine.cpp
)
target_include_directories(minijs_core PUBLIC include)
target_compile_options(minijs_core PRIVATE -Wall -Wextra -Wpedantic)

if(NOT EMSCRIPTEN)
    add_executable(minijs src/main.cpp)
    target_link_libraries(minijs PRIVATE minijs_core)

    enable_testing()
    add_executable(minijs_tests
        tests/test_main.cpp
        tests/engine_tests.cpp
    )
    target_link_libraries(minijs_tests PRIVATE minijs_core)
    add_test(NAME minijs_tests COMMAND minijs_tests
             WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
endif()
```

Build commands you will use every day:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/minijs examples/hello.mini
ctest --test-dir build --output-on-failure

# sanitizer build, run it at the end of every chapter
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug -DMINIJS_SANITIZE=ON
cmake --build build-asan && ./build-asan/minijs_tests
```

**1.2 Write `include/minijs/engine.hpp`.** This is the **only** header the
wrappers (`main.cpp`, later `wasm_api.cpp`) really depend on. Design it now and
keep it stable.

```cpp
#pragma once

#include <cstddef>
#include <string>
#include <string_view>
#include <vector>

namespace minijs {

enum class ErrorKind { Lexical, Syntax, Runtime, Limit, Internal };

const char* errorKindName(ErrorKind kind);   // "lexical", "syntax", ...

struct EngineError {
    ErrorKind kind = ErrorKind::Internal;
    std::string message;
    int line = 0;
    int col = 0;
};

struct EngineOptions {
    bool trace = false;
    std::size_t maxSteps = 100'000;        // used from Chapter 8
    std::size_t maxCallDepth = 200;        // used from Chapter 8
    std::size_t maxTraceEvents = 50'000;   // used from Chapter 8
    std::size_t maxOutputLines = 1'000;    // used from Chapter 8
};

struct EngineResult {
    bool ok = true;
    std::vector<std::string> output;   // one entry per console.log call
    std::vector<EngineError> errors;
    std::string traceJson = "[]";      // filled from Chapter 3
    bool traceTruncated = false;       // filled from Chapter 8
};

EngineResult run(std::string_view source, const EngineOptions& options);

}  // namespace minijs
```

**1.3 Write `src/engine.cpp`.** For now, `run` is a stub:

1. Create an `EngineResult`.
2. If `source` is empty or only whitespace, return it with `ok = true`.
3. Otherwise also return `ok = true` with no output. (Chapter 2 replaces this.)
4. Implement `errorKindName` with a `switch` over all enum values.

**1.4 Write `src/main.cpp`** (the command-line wrapper). Algorithm:

1. Parse arguments: `minijs <file> [--tokens] [--ast] [--trace <out.json>]`.
   Only `<file>` works now; unknown flags print usage and exit with code `64`.
2. Read the whole file into a `std::string` with `std::ifstream` and
   `std::ostringstream` (`buffer << file.rdbuf()`). If the file cannot be opened,
   print an error and exit `66`.
3. Call `minijs::run(source, options)`.
4. Print each `output` line to `std::cout`.
5. Print each error to `std::cerr` as `file:line:col: <kind> error: <message>`.
6. Exit codes: `0` success, `65` lexical or syntax error, `70` runtime or limit
   error.

`main.cpp` is allowed to print and read files. `engine.cpp` never is.

**1.5 Write the test framework (`tests/test.hpp`, `tests/test_main.cpp`).**
Declarations:

```cpp
// tests/test.hpp
#pragma once
#include <string>
#include <vector>

struct TestCase {
    const char* name;
    void (*fn)();
};

std::vector<TestCase>& testRegistry();

struct TestRegistrar {
    TestRegistrar(const char* name, void (*fn)());   // pushes into registry
};

struct TestFailure {
    std::string message;   // "file:line: CHECK failed: <expr>"
};

#define TEST(name)                                         \
    static void name();                                    \
    static TestRegistrar name##_registrar(#name, name);    \
    static void name()

// CHECK(cond): throw TestFailure if cond is false.
// CHECK_EQ(a, b): throw TestFailure showing both values if a != b.
// TODO: write both macros using __FILE__, __LINE__, and #cond.
```

`test_main.cpp` algorithm:

1. Loop over `testRegistry()`.
2. Call each test inside `try`. Catch `TestFailure` (print `FAIL name: message`)
   and `std::exception` (print `ERROR name: what()`).
3. Print `PASS name` for successes and a summary `N passed, M failed`.
4. Return `1` if anything failed, so `ctest` reports failure.

**1.6 Golden example files.** Every `examples/<name>.mini` gets a
`examples/<name>.expected` file holding the exact expected console output. In
Chapter 5 you will add a test that runs every example and compares. Start with:

```javascript
// examples/hello.mini
console.log("Hello, MiniJS");
```

```text
Hello, MiniJS
```

(It will not pass until Chapter 5. That is fine.)

### Tests

| Test | Expectation |
|------|-------------|
| `run("", {})` | `ok == true`, no output, no errors |
| `run("   \n  ", {})` | `ok == true` |
| `errorKindName(ErrorKind::Syntax)` | `"syntax"` |
| A deliberately failing `CHECK(1 == 2)` | the runner prints `FAIL` (then delete this test) |

### Done when

- [ ] `cmake --build build` has zero warnings.
- [ ] `./build/minijs examples/hello.mini` runs and exits `0`.
- [ ] `./build/minijs missing.mini` exits `66` with a clear message.
- [ ] `ctest` runs and passes.
- [ ] The sanitizer build compiles and the tests pass.
- [ ] `NOTES.md` has its first entry.

### Pitfalls

- Putting `add_compile_options(-fsanitize=...)` **after** `add_library`. It only
  affects targets created after it.
- Forgetting `WORKING_DIRECTORY` in `add_test`, so tests cannot find `examples/`.
- Letting `engine.cpp` include `<iostream>`. If it needs to print, the design is
  wrong.

---

## Chapter 2: Lexer

**Goal.** Turn source text into a list of tokens with exact positions, and report
bad characters and broken strings without crashing.

**Concepts.** `enum class`, `struct`, `std::string_view`, `switch`, character
classification, `std::unordered_map`, member functions with state.

### Files

```text
include/minijs/token.hpp      (new)
include/minijs/lexer.hpp      (new)
src/token.cpp                 (new: tokenTypeName)
src/lexer.cpp                 (new)
src/engine.cpp                (call the lexer)
src/main.cpp                  (--tokens flag)
tests/lexer_tests.cpp         (new)
CMakeLists.txt                (add the new .cpp files)
```

### Build steps

**2.1 Write `token.hpp`.**

```cpp
#pragma once
#include <cstddef>
#include <string>

namespace minijs {

enum class TokenType {
    // single character
    LeftParen, RightParen, LeftBrace, RightBrace,
    Comma, Dot, Semicolon,
    Plus, Minus, Star, Slash, Percent,
    // one or two characters
    Bang, BangEqual, Equal, EqualEqual,
    Less, LessEqual, Greater, GreaterEqual,
    AndAnd, OrOr,
    // literals
    Identifier, Number, String,
    // keywords
    Let, Const, Function, Return, If, Else, While, True, False, Null,
    // end
    EndOfFile,
};

// Names used in trace JSON and in --tokens output:
// "LEFT_PAREN", "EQUAL_EQUAL", "IDENTIFIER", "LET", "EOF", ...
const char* tokenTypeName(TokenType type);

struct Token {
    TokenType type = TokenType::EndOfFile;
    std::string lexeme;    // exact source text: "let", "\"hi\\n\"", "3.5"
    std::string literal;   // processed value: string without quotes and with
                           // escapes applied; empty for other tokens
    int line = 1;          // 1-based
    int col = 1;           // 1-based
    std::size_t offset = 0;  // byte index of the first character
    std::size_t length = 0;  // number of bytes in lexeme
};

}  // namespace minijs
```

**2.2 Write `lexer.hpp`.**

```cpp
#pragma once
#include <string_view>
#include <vector>
#include "minijs/engine.hpp"
#include "minijs/token.hpp"

namespace minijs {

class Lexer {
public:
    Lexer(std::string_view source, std::vector<EngineError>& errors);
    std::vector<Token> scanAll();   // always ends with one EndOfFile token

private:
    void scanToken();
    void number();
    void string(char quote);
    void identifierOrKeyword();

    char advance();                 // consume one char, update line/col
    char peek() const;              // current char or '\0' at end
    char peekNext() const;          // char after current or '\0'
    bool match(char expected);      // consume only if it matches
    bool isAtEnd() const;

    void addToken(TokenType type, std::string literal = {});
    void error(const std::string& message);   // uses the token start position

    std::string_view source_;
    std::vector<EngineError>& errors_;
    std::vector<Token> tokens_;
    std::size_t start_ = 0;      // offset where the current token began
    std::size_t current_ = 0;    // offset of the next char to read
    int line_ = 1, col_ = 1;             // position of current_
    int startLine_ = 1, startCol_ = 1;   // position of start_
};

}  // namespace minijs
```

**2.3 Implement the helpers** (`advance`, `peek`, `peekNext`, `match`,
`isAtEnd`). The only subtle one is `advance`:

1. Read `source_[current_]`, then increment `current_`.
2. If the char was `'\n'`: `line_ += 1`, `col_ = 1`. Otherwise `col_ += 1`.
3. Return the char.

**2.4 Implement `scanAll`.**

1. While not at end: set `start_ = current_`, `startLine_ = line_`,
   `startCol_ = col_`, then call `scanToken()`.
2. Push a final `EndOfFile` token with an empty lexeme at the current position.
3. Return `tokens_` (move it out).

**2.5 Implement `scanToken`** as one `switch` on `char c = advance()`:

| Input | Action |
|-------|--------|
| `' '`, `'\t'`, `'\r'`, `'\n'` | ignore |
| `( ) { } , . ; + - * %` | add the single-char token |
| `/` | if `match('/')`: skip until `'\n'` or end (comment); else `Slash` |
| `!` | `BangEqual` if `match('=')`, else `Bang` |
| `=` | if `match('=')`: if `peek() == '='` report error *"'===' is not supported; MiniJS '==' is already strict"* and consume it; add `EqualEqual`. Else `Equal` |
| `<` / `>` | `LessEqual` / `GreaterEqual` if `match('=')`, else `Less` / `Greater` |
| `&` | `AndAnd` if `match('&')`, else error *"Expected '&&'"* |
| `\|` | `OrOr` if `match('\|')`, else error *"Expected '\|\|'"* |
| `"` or `'` | `string(c)` |
| digit | `number()` |
| letter or `_` or `$` | `identifierOrKeyword()` |
| anything else | error *"Unexpected character 'X'"* and continue |

**2.6 Implement `number`.**

1. Consume digits while `peek()` is a digit.
2. If `peek() == '.'` **and** `peekNext()` is a digit: consume the `.` and the
   following digits. (`1.` and `.5` are not numbers in MiniJS.)
3. `addToken(Number)`. The parser converts the lexeme with `std::stod` later.

**2.7 Implement `string(char quote)`.**

1. Build `std::string value`.
2. Loop: if at end or `peek() == '\n'`, report *"Unterminated string"* and return
   without adding a token.
3. If `peek() == quote`, consume it and stop.
4. If `peek() == '\\'`, consume it, then map the next char: `n`→newline,
   `t`→tab, `\\`, `"`, `'`. Any other escape: error *"Unknown escape '\\X'"*.
5. Otherwise append the char to `value`.
6. `addToken(String, value)`.

**2.8 Implement `identifierOrKeyword`.**

1. Consume while `peek()` is a letter, digit, `_`, or `$`.
2. Look the text up in a keyword table:
   `static const std::unordered_map<std::string_view, TokenType> keywords`
   with `let const function return if else while true false null`.
3. Add the keyword type if found, otherwise `Identifier`.

**2.9 Implement `addToken`.** Build a `Token` with
`lexeme = source_.substr(start_, current_ - start_)`, `offset = start_`,
`length = current_ - start_`, `line = startLine_`, `col = startCol_`.

**2.10 Connect it.** In `engine.cpp`, `run` now creates a `Lexer`, calls
`scanAll`, and sets `ok = false` if `errors` is not empty. In `main.cpp`,
`--tokens` prints one token per line:

```text
1:1   LET          let
1:5   IDENTIFIER   total
1:11  EQUAL        =
1:13  NUMBER       10
1:16  PLUS         +
1:18  NUMBER       20
1:20  SEMICOLON    ;
1:21  EOF
```

(The wrapper calls the `Lexer` directly for this debug view. That is allowed: a
wrapper may use any part of the core library.)

### Tests (`tests/lexer_tests.cpp`)

Write a helper `std::vector<TokenType> types(std::string_view src)` first.

| Input | Expected |
|-------|----------|
| `let total = 10 + 20;` | `LET IDENTIFIER EQUAL NUMBER PLUS NUMBER SEMICOLON EOF` |
| `a <= b == c != d` | `IDENTIFIER LESS_EQUAL IDENTIFIER EQUAL_EQUAL IDENTIFIER BANG_EQUAL IDENTIFIER EOF` |
| `3.14` | one `NUMBER` with lexeme `3.14` |
| `1.` | `NUMBER(1) DOT EOF` |
| `"a\nb"` (escaped) | `STRING`, literal has a real newline, length 3 |
| `'single'` | `STRING` with literal `single` |
| `"open` | one lexical error *Unterminated string* |
| `let @ x` | one error at `1:5`, tokens `LET IDENTIFIER EOF` |
| `x // comment\ny` | `IDENTIFIER IDENTIFIER EOF`, second one on line 2 col 1 |
| `letter` | `IDENTIFIER` (not `LET`) |
| `a === b` | one error mentioning `===` |
| `""` | `STRING` with empty literal |
| empty input | just `EOF` at `1:1` |

Also check `offset` and `length` for at least one token in each test.

### Done when

- [ ] All lexer tests pass, also under sanitizers.
- [ ] `./build/minijs --tokens examples/hello.mini` prints the token list.
- [ ] Unknown characters produce errors with correct line and column, and the
      program exits `65`.
- [ ] `tokenTypeName` has a case for every `TokenType` (turn on `-Wswitch`; it is
      part of `-Wall`).

### Pitfalls

- Naming an enum value `EOF` or `NULL`. Both are macros in C/C++ headers. That is
  why the enum uses `EndOfFile` and `Null`, and only the *printed* name is `"EOF"`.
- Calling `std::isdigit(c)` with a negative `char` (any non-ASCII byte) is
  undefined behavior. Cast first: `std::isdigit(static_cast<unsigned char>(c))`.
- Storing `std::string_view` in `Token` that points into a temporary string. That
  is why `Token` uses `std::string`.
- Forgetting to update `col_` in `advance`, so every column after the first token
  is wrong.

---

## Chapter 3: Trace system and the first page

**Goal.** The engine records what it does as JSON, the CLI can save it, and a
React page can load that file and show the source next to its tokens.

**Concepts.** Interface classes (pure `virtual`), `final`, overload resolution,
string escaping, `std::ostringstream`; on the web side Vite, React components,
`useState`, file loading.

### Files

```text
include/minijs/trace.hpp      (new)
src/trace.cpp                 (new)
src/lexer.cpp                 (emit token events)
src/engine.cpp                (create the sink, add toJson)
include/minijs/engine.hpp     (declare toJson)
src/main.cpp                  (--trace <out.json>)
tests/trace_tests.cpp         (new)
playground/                   (new Vite + React app)
```

### Build steps

**3.1 Write `trace.hpp`.**

```cpp
#pragma once
#include <cstddef>
#include <string>
#include <string_view>
#include <utility>
#include <vector>

namespace minijs {

std::string jsonEscape(std::string_view text);   // returns "\"...\"" with escapes

class TraceEvent {
public:
    TraceEvent(std::string_view phase, std::string_view event);

    TraceEvent& add(std::string_view key, int value);
    TraceEvent& add(std::string_view key, std::size_t value);
    TraceEvent& add(std::string_view key, double value);
    TraceEvent& add(std::string_view key, bool value);
    TraceEvent& add(std::string_view key, const char* value);
    TraceEvent& add(std::string_view key, std::string_view value);
    TraceEvent& addRaw(std::string_view key, std::string json);   // already JSON

    std::string toJson(std::size_t step) const;

private:
    std::string phase_;
    std::string event_;
    std::vector<std::pair<std::string, std::string>> fields_;   // key, JSON text
};

class TraceSink {
public:
    virtual ~TraceSink() = default;
    virtual bool enabled() const = 0;
    virtual void emit(const TraceEvent& event) = 0;
};

class NullTraceSink final : public TraceSink {
public:
    bool enabled() const override { return false; }
    void emit(const TraceEvent&) override {}
};

class CollectingTraceSink final : public TraceSink {
public:
    explicit CollectingTraceSink(std::size_t maxEvents);
    bool enabled() const override;
    void emit(const TraceEvent& event) override;
    std::string json() const;          // "[{...},{...}]"
    bool truncated() const;

private:
    std::vector<std::string> events_;  // each already serialized
    std::size_t maxEvents_;
    bool truncated_ = false;
};

}  // namespace minijs
```

**3.2 Implement `jsonEscape`.** Loop over the bytes and append to a result that
starts and ends with `"`:

| Byte | Output |
|------|--------|
| `"` | `\"` |
| `\` | `\\` |
| newline | `\n` |
| carriage return | `\r` |
| tab | `\t` |
| other byte below `0x20` | `\u00XX` (two hex digits) |
| anything else | the byte itself (UTF-8 passes through) |

**3.3 Implement `TraceEvent`.**

- Each `add` converts the value to JSON text and pushes `{key, json}`.
  Integers: `std::to_string`. Booleans: `"true"` / `"false"`. Strings:
  `jsonEscape`. Doubles: if not finite (`std::isfinite`), write
  `"\"Infinity\""`-style strings; otherwise write the number (Chapter 5 gives you
  `formatNumber` for this).
- `toJson(step)` writes `{"step":N,"phase":"...","event":"...",<fields>}` in the
  order fields were added.

**3.4 Implement `CollectingTraceSink`.**

1. `enabled()` returns `!truncated_`.
2. `emit`: if truncated, return. If `events_.size() == maxEvents_`, push one
   final event `{"phase":"meta","event":"truncated","limit":N}`, set
   `truncated_ = true`, and return. Otherwise push `event.toJson(events_.size())`.
3. `json()` joins the strings with commas inside `[` `]`.

`emit` must **never throw**. Later, `emit` is called from destructors.

**3.5 Emit token events.** Give `Lexer` a `TraceSink&` constructor parameter. In
`scanAll`, after scanning is finished, emit one event per token (including EOF):

```cpp
// Pattern used everywhere in the engine:
if (trace_.enabled()) {
    trace_.emit(TraceEvent("lex", "token")
                    .add("type", tokenTypeName(t.type))
                    .add("text", t.lexeme)
                    .add("line", t.line)
                    .add("col", t.col)
                    .add("offset", t.offset)
                    .add("length", t.length));
}
```

The `enabled()` check means normal runs never build these strings.

**3.6 Define the result JSON.** Add to `engine.hpp`:

```cpp
std::string toJson(const EngineResult& result, std::string_view source);
```

It produces this exact shape. The CLI `--trace` file, the tests, and later the
WebAssembly build all return **this same shape**, so the page never cares where
the JSON came from.

```json
{
  "version": 1,
  "ok": true,
  "source": "let total = 10 + 20;",
  "output": [],
  "errors": [],
  "truncated": false,
  "trace": [
    { "step": 0, "phase": "lex", "event": "token", "type": "LET", "text": "let",
      "line": 1, "col": 1, "offset": 0, "length": 3 }
  ]
}
```

Each error is `{"kind":"lexical","message":"...","line":1,"col":5}`.

**3.7 Update `run` in `engine.cpp`.**

1. Create `NullTraceSink` or `CollectingTraceSink(options.maxTraceEvents)`
   depending on `options.trace`. Hold it through a `TraceSink&`.
2. Lex with that sink.
3. At the end, if collecting, copy `json()` and `truncated()` into the result.

**3.8 CLI.** `--trace out.json` sets `options.trace = true`, runs, and writes
`toJson(result, source)` to `out.json`.

**3.9 Create the playground app.**

```bash
cd minijs
npm create vite@latest playground -- --template react
cd playground && npm install && npm run dev
```

Generate a sample trace and put it where the page can fetch it:

```bash
./build/minijs examples/hello.mini --trace playground/public/sample-trace.json
```

**3.10 Build three components.**

| Component | Props | What it does |
|-----------|-------|--------------|
| `App.jsx` | none | holds `result` and `highlight` (`{start, end}` or `null`) in `useState`; fetches `sample-trace.json` on mount; also has an `<input type="file">` to load any trace |
| `panels/SourceView.jsx` | `source`, `highlight` | renders source in a `<pre>`; when `highlight` is set, splits into `before`, `<mark>middle</mark>`, `after` using offsets |
| `panels/TokensPanel.jsx` | `events`, `onSelect` | filters `event === "token"`; shows a table of type, text, line:col; clicking a row calls `onSelect({start: offset, end: offset + length})` |

### Trace events added

| phase | event | fields |
|-------|-------|--------|
| `lex` | `token` | `type`, `text`, `line`, `col`, `offset`, `length` |
| `meta` | `truncated` | `limit` |

### Tests (`tests/trace_tests.cpp`)

| Test | Expected |
|------|----------|
| `jsonEscape("a\"b")` | `"a\"b"` with the backslash |
| `jsonEscape("line\nnext")` | contains `\n`, no raw newline |
| `jsonEscape(std::string(1, '\x01'))` | `"\u0001"` |
| `TraceEvent("lex","token").add("type","LET").toJson(0)` | `{"step":0,"phase":"lex","event":"token","type":"LET"}` |
| `CollectingTraceSink(2)` with 5 emits | 3 entries, last is `truncated`, `truncated() == true` |
| `run("let x;", {.trace = true})` | trace has 4 token events (`LET IDENTIFIER SEMICOLON EOF`) |
| `run("let x;", {.trace = false})` | `traceJson == "[]"` |

### Done when

- [ ] The CLI writes a valid JSON file (check it with `python3 -m json.tool out.json`).
- [ ] The page shows the source and a token table.
- [ ] Clicking a token highlights exactly that text in the source.
- [ ] Loading a different trace file through the file input works.

### Pitfalls

- **`add("type", "LET")` silently calls the `bool` overload** if there is no
  `const char*` overload. A string literal converts to `bool` (a standard
  conversion) before `std::string_view` (a user-defined conversion). That is why
  the `const char*` overload exists.
- **`add("line", 5)` is ambiguous** if only `std::size_t` and `double` overloads
  exist, because `int → size_t` and `int → double` are conversions of the same
  rank. That is why there is an exact `int` overload.
- Building trace strings when tracing is off. Always check `enabled()` first.
- Highlighting by line/column in JavaScript. Use `offset` and `length`; it is
  much simpler. (Offsets are bytes; this is exact for ASCII source, which is what
  MiniJS examples use. Note it in `NOTES.md`.)

---

## Chapter 4: AST and Parser

**Goal.** Turn tokens into a tree that respects precedence, report every syntax
error with its position, and show the tree in the page.

**Concepts.** `std::unique_ptr`, inheritance, `virtual` destructors,
`static_cast`, recursion, exceptions, `std::initializer_list`, templates for
small helpers, `const` correctness.

### Files

```text
include/minijs/ast.hpp        (new)
src/ast.cpp                   (new: nodeKindName, astToString, emitAst)
include/minijs/parser.hpp     (new)
src/parser.cpp                (new)
src/engine.cpp                (parse after lexing)
src/main.cpp                  (--ast)
tests/parser_tests.cpp        (new)
playground/src/panels/AstPanel.jsx   (new)
```

### 4.1 The grammar (your parser is a direct translation of this)

Each rule becomes one function. A rule that appears to its own left with a lower
precedence calls the next rule down.

```text
program        := declaration* EOF

declaration    := functionDecl | varDecl | statement
functionDecl   := "function" IDENTIFIER "(" parameters? ")" block
parameters     := IDENTIFIER ( "," IDENTIFIER )*
varDecl        := ( "let" | "const" ) IDENTIFIER ( "=" expression )? ";"

statement      := ifStmt | whileStmt | returnStmt | block | exprStmt
ifStmt         := "if" "(" expression ")" statement ( "else" statement )?
whileStmt      := "while" "(" expression ")" statement
returnStmt     := "return" expression? ";"
block          := "{" declaration* "}"
exprStmt       := expression ";"

expression     := assignment
assignment     := IDENTIFIER "=" assignment | logicOr          (right-associative)
logicOr        := logicAnd ( "||" logicAnd )*
logicAnd       := equality ( "&&" equality )*
equality       := comparison ( ( "==" | "!=" ) comparison )*
comparison     := term ( ( "<" | ">" | "<=" | ">=" ) term )*
term           := factor ( ( "+" | "-" ) factor )*
factor         := unary ( ( "*" | "/" | "%" ) unary )*
unary          := ( "!" | "-" ) unary | call
call           := primary ( "(" arguments? ")" )*
arguments      := expression ( "," expression )*
primary        := NUMBER | STRING | "true" | "false" | "null"
                | IDENTIFIER ( "." IDENTIFIER )?
                | "(" expression ")"
```

Extra rules the grammar cannot express:

- `const` **must** have an initializer: *"const 'x' must be initialized"*.
- `return` is only allowed inside a function: *"'return' outside a function"*.
- `IDENTIFIER "." IDENTIFIER` is only used for `console.log`. The parser joins it
  into one `Identifier` named `"console.log"`. Any other dotted name is a syntax
  error: *"Property access is not supported (only console.log)"*.

### 4.2 Write `ast.hpp`

```cpp
#pragma once
#include <cstddef>
#include <memory>
#include <string>
#include <vector>
#include "minijs/token.hpp"

namespace minijs {

struct Span {
    std::size_t start = 0;   // byte offset of the first token
    std::size_t end = 0;     // byte offset just after the last token
    int line = 1;
    int col = 1;
};

enum class NodeKind {
    // expressions
    NumberLiteral, StringLiteral, BooleanLiteral, NullLiteral,
    Identifier, Grouping, Unary, Binary, Logical, Assign, Call,
    // statements
    ExpressionStmt, VarDecl, Block, If, While, Return, FunctionDecl,
    // root
    Program,
};

// "NumberLiteral", "BinaryExpression", "VariableDeclaration", ...
const char* nodeKindName(NodeKind kind);

struct Node {
    NodeKind kind;
    int id = 0;
    Span span;
    virtual ~Node() = default;
protected:
    explicit Node(NodeKind k) : kind(k) {}
};

struct Expr : Node { using Node::Node; };
struct Stmt : Node { using Node::Node; };
using ExprPtr = std::unique_ptr<Expr>;
using StmtPtr = std::unique_ptr<Stmt>;

struct NumberLiteral  : Expr { NumberLiteral()  : Expr(NodeKind::NumberLiteral) {}  double value = 0; };
struct StringLiteral  : Expr { StringLiteral()  : Expr(NodeKind::StringLiteral) {}  std::string value; };
struct BooleanLiteral : Expr { BooleanLiteral() : Expr(NodeKind::BooleanLiteral) {} bool value = false; };
struct NullLiteral    : Expr { NullLiteral()    : Expr(NodeKind::NullLiteral) {} };
struct Identifier     : Expr { Identifier()     : Expr(NodeKind::Identifier) {}     std::string name; };
struct Grouping       : Expr { Grouping()       : Expr(NodeKind::Grouping) {}       ExprPtr inner; };
struct UnaryExpr      : Expr { UnaryExpr()      : Expr(NodeKind::Unary) {}          TokenType op{}; ExprPtr operand; };
struct BinaryExpr     : Expr { BinaryExpr()     : Expr(NodeKind::Binary) {}         TokenType op{}; ExprPtr left, right; };
struct LogicalExpr    : Expr { LogicalExpr()    : Expr(NodeKind::Logical) {}        TokenType op{}; ExprPtr left, right; };
struct AssignExpr     : Expr { AssignExpr()     : Expr(NodeKind::Assign) {}         std::string name; ExprPtr value; };
struct CallExpr       : Expr { CallExpr()       : Expr(NodeKind::Call) {}           ExprPtr callee; std::vector<ExprPtr> args; };

struct ExpressionStmt : Stmt { ExpressionStmt() : Stmt(NodeKind::ExpressionStmt) {} ExprPtr expr; };
struct VarDecl        : Stmt { VarDecl()        : Stmt(NodeKind::VarDecl) {}        bool isConst = false; std::string name; ExprPtr init; };
struct BlockStmt      : Stmt { BlockStmt()      : Stmt(NodeKind::Block) {}          std::vector<StmtPtr> body; };
struct IfStmt         : Stmt { IfStmt()         : Stmt(NodeKind::If) {}             ExprPtr condition; StmtPtr thenBranch; StmtPtr elseBranch; };
struct WhileStmt      : Stmt { WhileStmt()      : Stmt(NodeKind::While) {}          ExprPtr condition; StmtPtr body; };
struct ReturnStmt     : Stmt { ReturnStmt()     : Stmt(NodeKind::Return) {}         ExprPtr value; };  // may be null
struct FunctionDecl   : Stmt { FunctionDecl()   : Stmt(NodeKind::FunctionDecl) {}   std::string name; std::vector<std::string> params; std::unique_ptr<BlockStmt> body; };

struct Program : Node {
    Program() : Node(NodeKind::Program) {}
    std::vector<StmtPtr> body;
};

std::string astToString(const Program& program);    // indented text tree
class TraceSink;
void emitAst(const Program& program, TraceSink& trace);

}  // namespace minijs
```

Why `BinaryExpr` and `LogicalExpr` are separate: `&&` and `||` must
**short-circuit** (the right side may not run), so the interpreter treats them
differently from `+`.

### 4.3 Write `parser.hpp`

```cpp
#pragma once
#include <initializer_list>
#include <stdexcept>
#include <string_view>
#include <vector>
#include "minijs/ast.hpp"
#include "minijs/engine.hpp"

namespace minijs {

struct ParseError : std::runtime_error {
    using std::runtime_error::runtime_error;
};

class Parser {
public:
    Parser(const std::vector<Token>& tokens, std::vector<EngineError>& errors);
    std::unique_ptr<Program> parse();

private:
    // statements
    StmtPtr declaration();
    StmtPtr functionDeclaration();
    StmtPtr varDeclaration();
    StmtPtr statement();
    StmtPtr ifStatement();
    StmtPtr whileStatement();
    StmtPtr returnStatement();
    std::unique_ptr<BlockStmt> block();
    StmtPtr expressionStatement();

    // expressions, lowest to highest precedence
    ExprPtr expression();
    ExprPtr assignment();
    ExprPtr logicOr();
    ExprPtr logicAnd();
    ExprPtr equality();
    ExprPtr comparison();
    ExprPtr term();
    ExprPtr factor();
    ExprPtr unary();
    ExprPtr call();
    ExprPtr primary();

    // token helpers
    const Token& peek() const;
    const Token& previous() const;
    bool isAtEnd() const;
    bool check(TokenType type) const;
    bool match(std::initializer_list<TokenType> types);
    const Token& advance();
    const Token& consume(TokenType type, std::string_view message);

    // node helpers
    template <class T> std::unique_ptr<T> make(const Token& first);   // sets id + span start
    void finish(Node& node);                                          // sets span end from previous()

    // errors
    ParseError error(const Token& at, std::string_view message);      // records + returns
    void synchronize();

    const std::vector<Token>& tokens_;
    std::vector<EngineError>& errors_;
    std::size_t current_ = 0;
    int nextId_ = 1;            // Program gets id 0
    int functionDepth_ = 0;     // for the return check
};

}  // namespace minijs
```

### 4.4 Implement the helpers

- `peek()` returns `tokens_[current_]`. `previous()` returns `tokens_[current_ - 1]`.
- `isAtEnd()` is `peek().type == EndOfFile`.
- `advance()`: if not at end, `++current_`; return `previous()`.
- `match({...})`: if `check` succeeds for any type, `advance()` and return `true`.
- `consume(type, msg)`: `advance()` if it matches, else `throw error(peek(), msg)`.
- `error(at, msg)`: push `EngineError{Syntax, msg, at.line, at.col}`; if `at` is
  EOF, make the message *"... at end of input"*; return a `ParseError` (the
  caller throws it).
- `make<T>(first)`: `auto n = std::make_unique<T>(); n->id = nextId_++;` and set
  `span.start = first.offset`, `span.line = first.line`, `span.col = first.col`.
- `finish(node)`: `span.end = previous().offset + previous().length`.

### 4.5 Implement the expression functions

All binary levels follow **one pattern**. Write `term()` first, then copy the
pattern for the others:

```text
term():
    first = peek()
    left = factor()
    while match({Plus, Minus}):
        op = previous().type
        right = factor()
        node = make<BinaryExpr>(first); node.op = op
        node.left = move(left); node.right = move(right)
        finish(node)
        left = move(node)
    return left
```

The loop makes `1 - 2 - 3` left-associative: `(1 - 2) - 3`.

The other functions:

- **`unary()`**: if `match({Bang, Minus})`, make a `UnaryExpr` whose operand is a
  recursive `unary()` call; else return `call()`.
- **`call()`**: `expr = primary()`; while `match({LeftParen})`: parse comma
  separated `expression()`s until `RightParen`, wrap into `CallExpr` with
  `callee = expr`. The loop allows `f()()` (it will fail at runtime, not parse time).
- **`primary()`**: one `if (match(...))` per alternative. `NUMBER` →
  `std::stod(lexeme)`. `STRING` → use `literal`. `IDENTIFIER` → if followed by
  `Dot`, apply the `console.log` rule from 4.1. `LeftParen` → `Grouping` with
  `expression()` and `consume(RightParen, "Expected ')' after expression")`.
  Otherwise `throw error(peek(), "Expected an expression")`.
- **`assignment()`**:
  1. `first = peek(); expr = logicOr()`.
  2. If `match({Equal})`: `value = assignment()` (recursion gives right
     associativity: `a = b = 3`).
  3. If `expr` is an `Identifier` (`expr->kind == NodeKind::Identifier`), build an
     `AssignExpr` with that name.
  4. Otherwise record *"Invalid assignment target"* at the `=` token (no throw:
     the parser is not confused) and return `expr`.

### 4.6 Implement the statement functions

- **`parse()`**: create `Program` with id `0`. While not at end, push
  `declaration()` if it is not null. Set the program span to the whole source.
- **`declaration()`**: wrap the body in `try`:
  `if match(Function)` → `functionDeclaration()`,
  `if match({Let, Const})` → `varDeclaration()`, else `statement()`.
  `catch (const ParseError&)` → `synchronize()` and return `nullptr`.
  This is **error recovery**: one bad line does not hide errors further down.
- **`synchronize()`**: `advance()`; then loop until at end: if `previous()` is
  `Semicolon`, return; if `peek()` is `function let const if while return` or
  `RightBrace`, return; else `advance()`.
- **`varDeclaration()`**: keyword is `previous()`. Consume the name. If
  `match({Equal})`, parse `expression()`. Check the `const` rule. Consume `;`.
- **`functionDeclaration()`**: consume name, `(`, parameters, `)`. Check for
  duplicate parameter names (*"Duplicate parameter 'n'"*). Then
  `++functionDepth_`, `body = block()` after consuming `{`, `--functionDepth_`.
  (Use a small RAII guard so the counter is restored even when `block()` throws.)
- **`block()`**: called after `{` was consumed. Collect `declaration()`s until
  `RightBrace` or end; `consume(RightBrace, "Expected '}' after block")`.
- **`ifStatement()`**, **`whileStatement()`**: consume `(`, `expression()`, `)`,
  then `statement()` for the bodies, plus `else` for `if`.
- **`returnStatement()`**: check `functionDepth_ > 0`. If the next token is not
  `;`, parse the value. Consume `;`.
- **`expressionStatement()`**: `expression()`, then
  `consume(Semicolon, "Expected ';' after expression")`.

### 4.7 `astToString` (for `--ast`) and `emitAst` (for the trace)

Both are recursive walks with a `switch (node.kind)` and `static_cast` to the
concrete type.

`astToString` output for `let result = 10 + 20 * 2;`:

```text
Program
  VariableDeclaration let result
    BinaryExpression +
      NumberLiteral 10
      BinaryExpression *
        NumberLiteral 20
        NumberLiteral 2
```

`emitAst` emits one event per node in **pre-order** (parent before children), so
the page can build the tree in a single pass. Emit after parsing has finished,
not during parsing, because during parsing a node's parent does not exist yet.

The `label` field is the short text after the kind in the tree above: the
operator, the name, or the literal value.

### 4.8 Connect it

`run` in `engine.cpp`:

1. Lex. If there are errors, stop (tokens are already in the trace).
2. Parse. If there are errors, stop.
3. `emitAst(*program, sink)`.

`main.cpp`: `--ast` prints `astToString`.

### 4.9 Visualizer: `AstPanel.jsx`

1. Filter `event === "node"` events. Build a map `id → {...event, children: []}`.
2. Push every node into its parent's `children`. The node with `parent === null`
   is the root.
3. Render recursively: each node shows `kind label`, has an expand/collapse
   toggle (start expanded), and on hover calls `onHover({start, end})` so
   `SourceView` highlights the matching text.

### Trace events added

| phase | event | fields |
|-------|-------|--------|
| `parse` | `node` | `id`, `kind`, `label`, `parent` (`null` for the root), `start`, `end`, `line`, `col` |
| `parse` | `error` | `kind: "syntax"`, `message`, `line`, `col` (also stored in `errors`) |

### Tests (`tests/parser_tests.cpp`)

Compare `astToString` output with expected text. It is easier to read than
walking pointers in tests.

| Input | Expected shape |
|-------|----------------|
| `10 + 20 * 2;` | `+` at the top, `*` as its right child |
| `(10 + 20) * 2;` | `*` at the top, `Grouping` on the left |
| `1 - 2 - 3;` | outer `-` has the inner `-` as its **left** child |
| `a = b = 3;` | outer assignment's value is the inner assignment |
| `!-x;` | `Unary !` → `Unary -` → `Identifier x` |
| `a || b && c;` | `||` at the top, `&&` on the right |
| `1 < 2 == true;` | `==` at the top |
| `console.log(1, "a");` | `Call` with callee `Identifier console.log`, 2 args |
| `if (x) { } else y;` | `IfStatement` with both branches |
| `function f(a, b) { return a; }` | `FunctionDeclaration f(a, b)` with a block |
| `let x = ;` | 1 syntax error *Expected an expression* at col 9 |
| `1 + 2` | error *Expected ';'* at end of input |
| `let a = 1 let b = 2; let c = ;` | **2** errors (recovery worked) |
| `const x;` | error *must be initialized* |
| `return 1;` (top level) | error *outside a function* |
| `1 = 2;` | error *Invalid assignment target* |
| `a.b;` | error *Property access is not supported* |
| `function f(a, a) {}` | error *Duplicate parameter* |

Also check that node `span.start`/`span.end` covers `20 * 2` exactly for the `*`
node in the first test.

### Done when

- [ ] All parser tests pass, also under sanitizers.
- [ ] `./build/minijs --ast` prints the tree for every example file.
- [ ] A file with three separate syntax errors reports all three.
- [ ] The page draws `let result = 10 + 20 * 2;` with `*` below `+`, and hovering
      a node highlights its text.
- [ ] When parsing fails, the page still shows the tokens.

### Pitfalls

- **Calling `peek()` past the end.** The last token is always EOF and `advance`
  never moves past it, so `peek()` is always safe. Keep that invariant.
- Writing `term` with recursion on the left (`term := term "+" factor`). That
  recurses forever. Use the `while` loop.
- Using `dynamic_cast` everywhere. You already have `kind`; check it and use
  `static_cast`.
- Forgetting `finish(node)`, so spans have `end == 0` and the UI highlights
  nothing.
- Deeply nested input like `((((((...))))))` overflowing the C++ stack. That is
  handled in Chapter 8; note it in `NOTES.md` now.

---

## Chapter 5: Values and expression evaluation

**Goal.** Run expressions and `console.log`, and let the page step through
evaluation one node at a time.

**Concepts.** `std::variant`, `std::visit`, `std::holds_alternative`,
`std::get`, `std::fmod`, number formatting, `std::function`, exception classes
with extra data.

### Files

```text
include/minijs/value.hpp        (new)
src/value.cpp                   (new)
include/minijs/interpreter.hpp  (new)
src/interpreter.cpp             (new)
src/engine.cpp                  (run the interpreter)
tests/interpreter_tests.cpp     (new)
tests/examples_tests.cpp        (new: golden files)
examples/arithmetic.mini + .expected
playground/src/state/deriveState.js   (new)
playground/src/panels/Controls.jsx    (new)
playground/src/panels/OutputPanel.jsx (new)
```

### 5.1 Language rules for values (implement exactly these)

| Rule | MiniJS behavior |
|------|-----------------|
| Types | `null`, boolean, number (`double`), string, function (Chapter 7) |
| Falsy values | `false`, `null`, `0`, `""` (`NaN` cannot occur, see division) |
| `+` | number + number adds; if **either** side is a string, both become display strings and are joined |
| `- * / %` | both sides must be numbers, else *"Operator '-' needs numbers, got string and number"* |
| `/` and `%` by zero | runtime error *"Division by zero"* |
| `%` | `std::fmod` (sign follows the left side, like JS) |
| `< > <= >=` | numbers compare numerically, strings compare lexicographically, mixed types are an error |
| `== !=` | true only if same type **and** same value. Functions compare by identity |
| `!` | `!isTruthy(x)` |
| unary `-` | number only |
| `&&` | evaluate left; if falsy return left, else evaluate and return right |
| `\|\|` | evaluate left; if truthy return left, else evaluate and return right |
| Display of numbers | integers print without `.0` (`40`, `-3`); others use the shortest form that round-trips (`0.30000000000000004`) |
| `console.log(a, b)` | display strings joined by one space; no arguments prints an empty line |

### 5.2 Write `value.hpp`

```cpp
#pragma once
#include <memory>
#include <string>
#include <variant>

namespace minijs {

struct Callable;   // defined in Chapter 7's callable.hpp; NativeFunction is in this chapter

class Value {
public:
    using Data = std::variant<std::monostate, bool, double, std::string,
                              std::shared_ptr<Callable>>;

    Value() = default;                         // null
    Value(bool b) : data_(b) {}
    Value(double d) : data_(d) {}
    Value(std::string s) : data_(std::move(s)) {}
    Value(const char* s) : data_(std::string(s)) {}
    Value(std::shared_ptr<Callable> c) : data_(std::move(c)) {}

    bool isNull() const;
    bool isBool() const;
    bool isNumber() const;
    bool isString() const;
    bool isCallable() const;

    bool asBool() const;
    double asNumber() const;
    const std::string& asString() const;
    const std::shared_ptr<Callable>& asCallable() const;

    const Data& data() const { return data_; }

private:
    Data data_;
};

const char* typeName(const Value& v);              // "null", "boolean", "number", "string", "function"
bool isTruthy(const Value& v);
bool strictEquals(const Value& a, const Value& b);
std::string formatNumber(double d);
std::string toDisplayString(const Value& v);       // what console.log prints
std::string toTraceJson(const Value& v);           // what the trace stores

}  // namespace minijs
```

Write a small `callable.hpp` now with just the base class and `NativeFunction`.
Chapter 7 adds `FunctionValue` to it.

```cpp
struct Callable {
    virtual ~Callable() = default;
    virtual std::string name() const = 0;
    virtual int arity() const = 0;   // -1 means any number of arguments
};

struct NativeFunction final : Callable {
    std::string fnName;
    int fnArity = -1;
    std::function<Value(std::vector<Value>& args)> fn;
    std::string name() const override { return fnName; }
    int arity() const override { return fnArity; }
};
```

### 5.3 Implement `value.cpp`

- `typeName`, `isTruthy`: use `std::visit` with an overloaded lambda set, or a
  chain of `std::holds_alternative` checks. Try `std::visit` at least once; it is
  the main reason to use `std::variant`.
- `strictEquals`: if `a.data().index() != b.data().index()` return `false`; then
  compare the values. For callables, compare the `shared_ptr`s.
- `formatNumber(d)`:
  1. If `d` is a whole number and `std::fabs(d) < 1e15`, print it as
     `static_cast<long long>(d)`. Handle `-0` by printing `0`.
  2. Otherwise try precision 1 to 17 with `std::ostringstream` and
     `std::setprecision(p)`; return the first string that `std::stod` turns back
     into exactly `d`.
- `toDisplayString`: null → `null`, booleans → `true`/`false`, numbers →
  `formatNumber`, strings → raw text, functions → `[function name]`.
- `toTraceJson`: null → `null`, booleans → `true`/`false`, finite numbers → the
  number, strings → `jsonEscape`, functions → `{"fn":"name"}`, non-finite numbers
  (possible through overflow, for example `1e308 * 10`) → `{"num":"Infinity"}`.

### 5.4 Write `interpreter.hpp`

```cpp
#pragma once
#include <stdexcept>
#include <string>
#include <unordered_map>
#include <vector>
#include "minijs/ast.hpp"
#include "minijs/engine.hpp"
#include "minijs/trace.hpp"
#include "minijs/value.hpp"

namespace minijs {

struct RuntimeError : std::runtime_error {
    RuntimeError(ErrorKind kind, const std::string& message, int line, int col);
    ErrorKind kind;
    int line;
    int col;
};

class Interpreter {
public:
    Interpreter(TraceSink& trace, const EngineOptions& options,
                std::vector<std::string>& output);
    void run(const Program& program);     // throws RuntimeError

private:
    Value evaluate(const Expr& expr);
    Value evalUnary(const UnaryExpr& e);
    Value evalBinary(const BinaryExpr& e);
    Value evalLogical(const LogicalExpr& e);
    Value evalIdentifier(const Identifier& e);
    Value evalCall(const CallExpr& e);

    void execute(const Stmt& stmt);   // becomes ExecSignal in Chapter 7

    [[noreturn]] void fail(const Node& at, const std::string& message);

    TraceSink& trace_;
    const EngineOptions& options_;
    std::vector<std::string>& output_;
    std::unordered_map<std::string, Value> builtins_;   // replaced in Chapter 6
};

}  // namespace minijs
```

### 5.5 Implement the interpreter

**Constructor.** Put `console.log` into `builtins_` as a `NativeFunction` whose
`fn` joins `toDisplayString` of all arguments with spaces, pushes the line into
`output_`, emits an `output` trace event, and returns `null`.

**`evaluate(expr)`**, the heart of the interpreter:

1. Emit `eval_start` with `node = expr.id`.
2. `switch (expr.kind)`:
   - literals: return the value stored in the node;
   - `Grouping`: return `evaluate(*inner)`;
   - `Unary`, `Binary`, `Logical`, `Identifier`, `Call`: call the matching
     `evalX(static_cast<const X&>(expr))`;
   - `Assign`: `fail(expr, "Assignment comes in Chapter 6")` for now.
3. Emit `eval_end` with `node` and `value` (`addRaw("value", toTraceJson(v))`).
4. Return the value.

**`evalBinary(e)`**: evaluate left, then right (**left first, always**), then
apply the table in 5.1 with a `switch (e.op)`. Use `fail(e, ...)` for type errors.

**`evalLogical(e)`**: follow the `&&` / `||` rows in 5.1. The right side must not
be evaluated when the left side decides the result. Test this with a
`console.log` on the right side.

**`evalIdentifier(e)`**: look `e.name` up in `builtins_`; if missing,
`fail(e, "Undefined variable 'x'")`.

**`evalCall(e)`**:

1. `callee = evaluate(*e.callee)`.
2. If it is not callable: `fail(e, "'x' is not a function")`.
3. Evaluate the arguments left to right into a `std::vector<Value>`.
4. If it is a `NativeFunction` (use `dynamic_cast`), check arity (unless `-1`)
   and call `fn(args)`.

**`execute(stmt)`**: for now only `ExpressionStmt` (evaluate and discard). Any
other kind: `fail(stmt, "Not supported yet")`.

**`run(program)`**: `execute` each statement in order.

**`fail`** throws `RuntimeError(ErrorKind::Runtime, message, at.span.line, at.span.col)`.

### 5.6 Connect it in `engine.cpp`

After a successful parse:

1. Create the `Interpreter` with the sink, options, and `result.output`.
2. `try { interp.run(*program); }`
   `catch (const RuntimeError& e) { push the error; emit an "error" event; }`
3. Set `result.ok = result.errors.empty()`.

### 5.7 Golden example test (`tests/examples_tests.cpp`)

1. Iterate `examples/` with `std::filesystem::directory_iterator`.
2. For each `.mini` file with a `.expected` file next to it, run the engine and
   join `output` with `"\n"`.
3. `CHECK_EQ` against the `.expected` contents (ignore one trailing newline).

From now on, every new feature gets an example file. This test runs all of them.

### 5.8 Visualizer: stepping

Everything on screen is computed from `trace[0..step]`. Write one pure function:

```text
deriveState(trace, step) -> {
    activeNode,      // id of the node being evaluated right now
    values,          // map nodeId -> last result (for the AST panel)
    output,          // console lines printed so far
    error,           // error event if reached
    // Chapter 6 adds scopes, Chapter 7 adds callStack
}

algorithm:
    state = empty
    for i in 0..min(step, trace.length - 1):
        e = trace[i]
        switch e.event:
            "eval_start": activeNode = e.node
            "eval_end":   activeNode = e.node; values[e.node] = e.value
            "output":     output.push(e.text)
            "error":      error = e
    return state
```

Components:

| Component | What it does |
|-----------|--------------|
| `Controls.jsx` | buttons Play, Pause, Step back, Step forward, Reset, a slider for the step number, a speed select (1, 5, 20, 100 steps per second). Play uses `setInterval` inside `useEffect` and stops at the last step |
| `OutputPanel.jsx` | lists `state.output`; shows `state.error` in red |
| `AstPanel.jsx` (update) | highlights `state.activeNode`; shows `= value` next to nodes that have a value in `state.values` |
| `SourceView.jsx` (update) | highlights the span of `state.activeNode` |

After loading a trace, set `step` to the **last** event (show the finished
result). Play starts again from step 0.

### Trace events added

| phase | event | fields |
|-------|-------|--------|
| `exec` | `eval_start` | `node` |
| `exec` | `eval_end` | `node`, `value` |
| `exec` | `output` | `text` |
| `exec` | `error` | `kind`, `message`, `line`, `col` |

### Tests (`tests/interpreter_tests.cpp`)

Write a helper `std::string outputOf(std::string_view src)` (joins output lines)
and `std::string errorOf(std::string_view src)` (first error message).

| Program | Output or error |
|---------|-----------------|
| `console.log(10 + 20 * 2);` | `50` |
| `console.log((10 + 20) * 2);` | `60` |
| `console.log(7 / 2, 7 % 3, -7 % 3);` | `3.5 1 -1` |
| `console.log(0.1 + 0.2);` | `0.30000000000000004` |
| `console.log("a" + 1 + 2);` | `a12` |
| `console.log(1 + 2 + "a");` | `3a` |
| `console.log(1 == 1, 1 == "1", null == null);` | `true false true` |
| `console.log("b" > "a", 2 <= 1);` | `true false` |
| `console.log(!0, !"x", -(3));` | `true false -3` |
| `console.log(0 || "x", 1 && 2, null && boom);` | `x 2 null` (no error: short-circuit) |
| `console.log();` | one empty line |
| `console.log(1 / 0);` | error *Division by zero* |
| `console.log("a" - 1);` | error mentions `string` and `number` |
| `console.log(x);` | error *Undefined variable 'x'* at `1:13` |
| `5();` | error *is not a function* |

Trace test: for `1 + 2;`, the exec events are, in order:
`eval_start(+) eval_start(1) eval_end(1) eval_start(2) eval_end(2) eval_end(+ = 3)`.

### Done when

- [ ] All tests pass, including `examples/hello.mini` and `arithmetic.mini`.
- [ ] In the page you can step through `10 + 20 * 2` and see `20 * 2 = 40` before
      `10 + 40 = 50`.
- [ ] Output appears step by step; an error shows in red with its position.
- [ ] Step back works exactly like step forward.

### Pitfalls

- **`Value v = "hi";` becomes `true`** without the `const char*` constructor (the
  same pointer-to-`bool` trap as in Chapter 3).
- Evaluating the right operand before the left. JS order is left to right and
  your tests (and trace) depend on it.
- Using `==` on `double` results in tests. Prefer checking printed output, which
  also tests `formatNumber`.
- `std::get<double>` on the wrong alternative throws `std::bad_variant_access`.
  Always check the type first and produce a MiniJS error instead.

---

## Chapter 6: Variables and scope

**Goal.** `let`, `const`, assignment, and block scopes work, and the page shows
the scope chain changing.

**Concepts.** `std::unordered_map`, `std::shared_ptr`, parent pointers, RAII
guards, deleted copy operations, returning pointers for "maybe found".

### Files

```text
include/minijs/environment.hpp   (new)
src/environment.cpp              (new)
src/interpreter.cpp              (variables, blocks, assignment)
include/minijs/interpreter.hpp
tests/interpreter_tests.cpp      (scope tests)
examples/scope.mini + .expected
playground/src/panels/ScopePanel.jsx  (new)
```

### 6.1 Rules

| Rule | Behavior |
|------|----------|
| `let x;` | declares `x` with value `null` |
| `let x = 1;` twice in the **same** scope | error *"'x' is already declared in this scope"* |
| `let x` inside a block when an outer `x` exists | allowed (shadowing) |
| `x = 2;` | updates the **nearest** scope that has `x`; error *"Undefined variable 'x'"* if none |
| `x = 2;` when `x` is `const` | error *"Cannot assign to const 'x'"* |
| assignment value | the assigned value (so `a = b = 3` works) |
| `{ ... }` | creates a new scope whose parent is the current scope |
| `let x = x;` | the right side runs **before** `x` exists in the new scope, so it refers to an outer `x` or fails |
| global scope | holds `console.log` (it is `const`: `console.log = 1;` is an error) |

### 6.2 Write `environment.hpp`

The environment does not throw. It reports problems through return values, and
the interpreter turns them into errors with the right source position.

```cpp
#pragma once
#include <memory>
#include <string>
#include <unordered_map>
#include <vector>
#include "minijs/value.hpp"

namespace minijs {

struct Binding {
    Value value;
    bool isConst = false;
};

class Environment {
public:
    Environment(int id, std::string kind, std::shared_ptr<Environment> parent);

    bool declare(const std::string& name, Value value, bool isConst);   // false if already here
    Binding* findLocal(const std::string& name);                        // this scope only
    Environment* resolve(const std::string& name);                      // nearest owner or nullptr

    int id() const;
    const std::string& kind() const;           // "global", "block", "function"
    Environment* parent() const;
    std::vector<std::string> namesInOrder() const;   // declaration order, for traces
    void clear();                                    // drops all bindings (Chapter 7 uses it)

private:
    int id_;
    std::string kind_;
    std::shared_ptr<Environment> parent_;
    std::unordered_map<std::string, Binding> bindings_;
    std::vector<std::string> order_;
};

}  // namespace minijs
```

`resolve` algorithm: start at `this`; while not null: if `findLocal(name)`
succeeds return the current environment; move to `parent`. Return `nullptr`.

### 6.3 Update the interpreter

New members:

```cpp
std::shared_ptr<Environment> globals_;
std::shared_ptr<Environment> current_;
int nextScopeId_ = 0;

class ScopeGuard {   // nested inside Interpreter so it can touch current_
public:
    ScopeGuard(Interpreter& interp, std::shared_ptr<Environment> env);
    ~ScopeGuard();
    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
private:
    Interpreter& interp_;
    std::shared_ptr<Environment> previous_;
};
```

- **Constructor of `Interpreter`**: create `globals_` (id `0`, kind `"global"`),
  declare `console.log` in it as `const`, set `current_ = globals_`, emit
  `scope_push` for the global scope. Delete `builtins_`.
- **`ScopeGuard` constructor**: save `current_` in `previous_`, set
  `current_ = env`, emit `scope_push`.
- **`ScopeGuard` destructor**: emit `scope_pop` for the current id, restore
  `current_ = previous_`. Because it is a destructor, this also runs when a
  `RuntimeError` unwinds through the block. That is the point of RAII.
- **`VarDecl`**: evaluate `init` (or `null`), then `current_->declare(...)`. If it
  returns `false`, `fail(stmt, "'x' is already declared in this scope")`. Emit
  `declare`.
- **`Block`**: create `std::make_shared<Environment>(++nextScopeId_, "block", current_)`,
  create a `ScopeGuard` on the stack, execute the statements.
- **`Identifier`**: `resolve`, then read the binding; `fail` with
  *Undefined variable* if not found.
- **`Assign`**: evaluate the value first, then `resolve`; check missing and
  `const`; store; emit `assign`; return the value.

### 6.4 Visualizer: `ScopePanel.jsx`

Extend `deriveState`:

```text
scopes: map scopeId -> { id, kind, parent, vars: map name -> {value, isConst} }
active: array of scope ids (push on scope_push, pop on scope_pop)
changed: name + scope id of the last declare/assign (for the flash effect)

"scope_push": scopes[e.scope] = {kind: e.kind, parent: e.parent, vars: {}}; active.push(e.scope)
"scope_pop":  active.pop()
"declare":    scopes[e.scope].vars[e.name] = {value: e.value, isConst: e.const}; changed = ...
"assign":     scopes[e.scope].vars[e.name].value = e.value; changed = ...
```

Rendering: take the top of `active` as the current scope and follow `parent`
links to build the **scope chain**. Draw it as stacked boxes, current scope on
top, global at the bottom. Each box lists its variables (`const` ones with a lock
icon or label). The variable in `changed` gets a CSS flash animation for one step.

### Trace events added

| phase | event | fields |
|-------|-------|--------|
| `exec` | `scope_push` | `scope`, `parent` (`null` for global), `kind` |
| `exec` | `scope_pop` | `scope` |
| `exec` | `declare` | `name`, `value`, `scope`, `const` |
| `exec` | `assign` | `name`, `value`, `scope` |
| `exec` | `stmt` | `node` (emit at the start of every statement, so the page can highlight the current line) |

### Tests

| Program | Output or error |
|---------|-----------------|
| `let a = 1; a = a + 1; console.log(a);` | `2` |
| `let a; console.log(a);` | `null` |
| `const c = 1; c = 2;` | error *Cannot assign to const 'c'* |
| `let a = 1; let a = 2;` | error *already declared* |
| `let a = 1; { let a = 2; console.log(a); } console.log(a);` | `2` then `1` |
| `let a = 1; { a = 5; } console.log(a);` | `5` |
| `{ let inner = 1; } console.log(inner);` | error *Undefined variable 'inner'* |
| `let a; let b; a = b = 3; console.log(a, b);` | `3 3` |
| `y = 1;` | error *Undefined variable 'y'* |
| `console.log = 1;` | error *Cannot assign to const* |
| `let x = 1; { let x = x + 1; console.log(x); }` | `2` |

Trace test: for `{ let a = 1; }` the scope events are
`scope_push(global) scope_push(1, parent 0) declare(a) scope_pop(1)`.

Error trace test: for `{ let a = 1; boom; }` a `scope_pop` still appears before
the trace ends (the guard destructor ran).

### Done when

- [ ] All tests pass, also under sanitizers.
- [ ] `examples/scope.mini` matches its `.expected` file.
- [ ] The page shows a block scope appearing on top of global and disappearing at
      `}`; outer variables stay visible in the chain; changed variables flash.

### Pitfalls

- Keeping a `Binding*` or `Environment*` after the scope may be destroyed. Look
  things up, use them immediately, do not store raw pointers.
- Throwing (or emitting something that can throw) inside `~ScopeGuard`. A
  destructor that throws during unwinding calls `std::terminate`. `emit` is
  `noexcept` in spirit; mark the destructor `noexcept` (it is by default) and keep
  it simple.
- Using `std::map` and expecting declaration order in the UI. That is what
  `order_` is for.

---

## Chapter 7: Control flow and functions

**Goal.** `if`, `while`, function declarations, calls, `return`, and recursion
work, and the page shows the call stack growing and shrinking.

**Concepts.** Designing a small control-flow signal type, call stack frames,
`shared_ptr` cycles, `dynamic_cast`, references vs copies, AST lifetime.

### Files

```text
include/minijs/callable.hpp     (add FunctionValue)
include/minijs/interpreter.hpp  (ExecSignal, call stack)
src/interpreter.cpp
src/engine.cpp                  (break shared_ptr cycles at the end)
tests/interpreter_tests.cpp
examples/loops.mini, functions.mini + .expected
playground/src/panels/CallStackPanel.jsx  (new)
```

### 7.1 Rules

| Rule | Behavior |
|------|----------|
| `if (c) A else B` | runs `A` if `isTruthy(c)`, else `B` (if present) |
| `while (c) S` | checks `c` before every iteration |
| `function f(a, b) {...}` | declares `f` as a **`const`** binding in the current scope, holding a function value |
| call arity | must match exactly: *"f expects 2 arguments, got 1"* |
| parameters | new `function` scope whose parent is the scope where `f` was **declared** |
| `return x;` | stops the function, result is `x` |
| `return;` or no return | result is `null` |
| recursion | works because `f` is in scope when its body runs |
| hoisting | none: calling `f` before its declaration is *Undefined variable 'f'* |

Using the declaration scope as the parent is called **lexical scoping**. It also
means nested functions can already see outer variables (simple closures) for
free. `initialDoc.md` lists closures as optional; you get the basic form without
extra work.

### 7.2 Add `FunctionValue` to `callable.hpp`

```cpp
class Environment;
struct FunctionDecl;

struct FunctionValue final : Callable {
    const FunctionDecl* decl;                 // owned by the Program, not by this
    std::shared_ptr<Environment> closure;     // scope where it was declared
    std::string name() const override;        // decl->name
    int arity() const override;               // decl->params.size()
};
```

### 7.3 Interpreter changes

```cpp
struct ExecSignal {
    enum class Kind { Normal, Return } kind = Kind::Normal;
    Value value;   // only meaningful for Return
};

struct CallFrame {
    std::string name;
    std::vector<Value> args;
    int line;
};

ExecSignal execute(const Stmt& stmt);
ExecSignal executeStatements(const std::vector<StmtPtr>& body);
Value callFunction(const FunctionValue& fn, std::vector<Value>& args, const CallExpr& site);

std::vector<CallFrame> callStack_;
```

Algorithms:

- **`executeStatements(body)`**: for each statement: `sig = execute(*s)`; if
  `sig.kind == Return`, return `sig` immediately. At the end return `Normal`.
- **`execute(Block)`**: make the scope + guard (Chapter 6), then
  `return executeStatements(block.body)`. The `Return` signal passes upward
  through every nested block until a function call catches it.
- **`execute(If)`**: evaluate the condition; execute and **return the signal of**
  the chosen branch.
- **`execute(While)`**: loop: evaluate the condition; stop if falsy; execute the
  body; if the body returned `Return`, pass it up.
- **`execute(Return)`**: evaluate the value (or `null`) and return
  `{Return, value}`.
- **`execute(FunctionDecl)`**: create
  `std::make_shared<FunctionValue>(&decl, current_)`, `declare` it as `const`.
- **`evalCall`** (update): if the callee is a `FunctionValue`, check arity and
  call `callFunction`.
- **`callFunction(fn, args, site)`**:
  1. Create `env = make_shared<Environment>(++nextScopeId_, "function", fn.closure)`.
  2. Push a `CallFrame` and emit `call` (`name`, `args` as a JSON array,
     `depth = callStack_.size()`, `line`).
  3. `ScopeGuard guard(*this, env);` then declare every parameter in `env`.
  4. `sig = executeStatements(fn.decl->body->body)`. (Run the body's statements
     directly in `env`; do not call `execute(Block)`, which would create a second,
     useless scope.)
  5. `result = sig.kind == Return ? sig.value : Value{}`.
  6. Emit `return` (`name`, `value`, `depth`), pop the frame, return `result`.

  Use a small `CallFrameGuard` (RAII) for the push/pop so that a runtime error
  inside the function still pops the frame.

### 7.4 Breaking the `shared_ptr` cycle

A function value lives in an environment, and it holds a `shared_ptr` to that
same environment. That is a cycle: neither is ever freed. At the end of `run` in
`engine.cpp`, after the interpreter finishes (success or error), call
`globals->clear()` (expose it through a method like `Interpreter::shutdown()`).
Clearing the global bindings drops the function values and breaks the cycles.
Write this down in `NOTES.md`; it is the small version of why real engines need a
garbage collector.

### 7.5 AST lifetime

`FunctionValue::decl` is a raw pointer into the AST. This is safe only because
`engine.cpp` keeps the `Program` alive until the interpreter is destroyed. Make
that order explicit in `run`: declare `program` **before** `interp`, so `interp`
is destroyed first.

### 7.6 Visualizer: `CallStackPanel.jsx`

Extend `deriveState`:

```text
callStack: array of { name, args, depth }
lastReturn: { name, value } or null

"call":   callStack.push({name: e.name, args: e.args, depth: e.depth})
"return": callStack.pop(); lastReturn = {name: e.name, value: e.value}
```

Render frames as a vertical stack, newest on top: `factorial(3)`. Show
`lastReturn` as a small "returned 6" badge for one step. When the trace ends in
an `error` event, the stack stays as it was, which shows where the error
happened.

Also mark native calls (`console.log`) differently, or hide them with a checkbox.

### Trace events added

| phase | event | fields |
|-------|-------|--------|
| `exec` | `call` | `name`, `args` (array), `depth`, `line`, `native` |
| `exec` | `return` | `name`, `value`, `depth` |
| `exec` | `branch` | `node` (the `if`), `taken` (`"then"`, `"else"`, `"none"`) |
| `exec` | `loop` | `node` (the `while`), `iteration` |

### Tests

| Program | Output or error |
|---------|-----------------|
| `if (1 < 2) console.log("yes"); else console.log("no");` | `yes` |
| `if (0) console.log("no");` | no output |
| `let i = 0; while (i < 3) { console.log(i); i = i + 1; }` | `0`, `1`, `2` |
| `function add(a, b) { return a + b; } console.log(add(2, 3));` | `5` |
| `function f() {} console.log(f());` | `null` |
| `function f() { return; } console.log(f());` | `null` |
| factorial from `initialDoc.md` with `5` | `120` |
| `function fib(n) { if (n < 2) return n; return fib(n-1) + fib(n-2); } console.log(fib(15));` | `610` |
| `function f(a) {} f();` | error *expects 1 argument* |
| `f(); function f() {}` | error *Undefined variable 'f'* |
| return inside a `while` inside an `if` | stops the function immediately |
| `let x = 1; function show() { console.log(x); } x = 2; show();` | `2` |
| `function outer() { let v = "in"; function inner() { return v; } return inner(); } console.log(outer());` | `in` |
| `function f() {} f = 1;` | error *Cannot assign to const* |

Trace test: `factorial(3)` produces `call` events with depth 1, 2, 3 and then
`return` events with values `1`, `2`, `6`.

### Done when

- [ ] All tests pass, also under sanitizers (no leaks reported thanks to 7.4).
- [ ] `loops.mini` and `functions.mini` match their `.expected` files.
- [ ] In the page, `factorial(5)` grows the stack to five frames and unwinds,
      showing each return value.

### Pitfalls

- Using an exception for `return`. It works, but it is slow and makes the control
  flow hard to follow. The `ExecSignal` is explicit.
- Using the **caller's** scope as the function's parent (dynamic scoping). The
  `x = 2; show();` test catches this, and so does the `inner` test.
- Copying `CallFrame`s or `Value`s by value in hot loops when a `const&` would do.
- Forgetting to propagate the `Return` signal out of `If`, `While`, or `Block`.
  The function then keeps running after `return`.

---

## Chapter 8: Safety, limits, and error polish

**Goal.** Nothing a visitor types can crash the engine, hang it, or produce an
endless trace, and every error message is clear.

**Concepts.** Counters and limits, `try`/`catch (...)`, `std::bad_alloc`,
sanitizers, stack depth, fuzz-style testing.

### Files

```text
src/interpreter.cpp     (step, depth, output limits)
src/parser.cpp          (nesting limit)
src/engine.cpp          (catch-all, final error handling)
src/main.cpp            (pretty errors with a caret)
tests/limits_tests.cpp  (new)
tests/robustness_tests.cpp (new)
playground/src/panels/ErrorsPanel.jsx (new)
```

### 8.1 The limits

| Limit | Where | Check | Error |
|-------|-------|-------|-------|
| `maxSteps` | `Interpreter::tick()`, called at the start of every `evaluate` and `execute` | `++steps_ > options_.maxSteps` | `Limit`: *"Step limit of 100000 reached (infinite loop?)"* |
| `maxCallDepth` | `callFunction` before pushing a frame | `callStack_.size() >= maxCallDepth` | `Limit`: *"Call depth limit of 200 reached (infinite recursion?)"* |
| parser nesting | `Parser::expression()` and `statement()`: a depth counter with an RAII guard | `depth_ > 200` | `Syntax`: *"Code is nested too deeply"* |
| `maxTraceEvents` | `CollectingTraceSink` (Chapter 3) | already done | sets `truncated`, **no error**, program keeps running |
| `maxOutputLines` | `console.log` | `output.size() >= maxOutputLines` | `Limit`: *"Output limit of 1000 lines reached"* |

Why the parser needs a limit too: the parser is recursive, so `((((((1))))))`
with 100,000 parentheses overflows the C++ stack before the interpreter even
starts.

### 8.2 The catch-all in `engine.cpp`

`run` must never let an exception escape (this matters a lot in WebAssembly in
Chapter 9). Structure:

```text
run(source, options):
    result
    try:
        lex, parse, emitAst, interpret     (as before)
    catch RuntimeError e:   push {e.kind, e.what(), e.line, e.col}
    catch std::bad_alloc:   push {Internal, "Out of memory"}
    catch std::exception e: push {Internal, "Internal error: " + e.what()}
    catch (...):            push {Internal, "Unknown internal error"}
    always:
        interpreter shutdown (break cycles, Chapter 7)
        copy trace json + truncated flag
        ok = errors.empty()
    return result
```

### 8.3 Error message quality

Every message follows the same style: what is wrong, the name involved, and a
hint when there is an obvious one. Collect them all into Appendix D as you write
them.

`main.cpp` prints errors with the source line and a caret:

```text
examples/bad.mini:3:17: runtime error: Undefined variable 'totl'
    console.log(totl);
                ^
```

Algorithm: split the source by `\n`, take line `line - 1`, print it, then
`col - 1` spaces and `^`. (Write this in `main.cpp`, not in the core: the page
renders errors its own way.)

### 8.4 Robustness tests

These are the most valuable tests in the project.

1. **Prefix test**: for each example file, run the engine on every prefix
   (`source.substr(0, n)` for every `n`). Each run must return normally with
   either `ok` or errors. This feeds the parser every possible half-written
   program, which is exactly what the live editor will do in Chapter 10.
2. **Random bytes test**: run 1,000 random strings built from the characters
   `letfunctionwhileif(){};=+-*/%<>!&|"' 0123456789\n`. Use a fixed seed
   (`std::mt19937 rng(42)`) so failures can be reproduced.
3. Run both under the sanitizer build. A sanitizer report is a failure even if
   the test "passes".

### 8.5 Visualizer: `ErrorsPanel.jsx`

- List `result.errors` with kind, message, and `line:col`. Clicking one
  highlights that position in the source.
- Color by kind: lexical/syntax (before running), runtime, limit.
- If `result.truncated`, show a notice: *"Trace stopped after 50,000 events; the
  program kept running and its output is complete."*

### Tests (`tests/limits_tests.cpp`)

| Program | Result |
|---------|--------|
| `while (true) {}` | one `Limit` error, returns quickly |
| `function f() { return f(); } f();` | one `Limit` error mentioning call depth |
| 10,000 nested `(` | one `Syntax` error *nested too deeply*, no crash |
| `let i = 0; while (i < 5000) { console.log(i); i = i + 1; }` | `Limit` error after 1,000 lines; `output.size() == 1000` |
| a loop that emits more than `maxTraceEvents` events with `trace = true` | `truncated == true`, `ok == true`, output complete |
| same loop with `trace = false` | same output, `traceJson == "[]"` |
| `EngineOptions{.maxSteps = 10}` with `console.log(1+1+1+1+1+1);` | `Limit` error (the limit is really configurable) |

### Done when

- [ ] Prefix and random tests pass under sanitizers.
- [ ] `while (true) {}` and runaway recursion stop with a clear message in well
      under a second.
- [ ] A fresh clone builds with zero warnings in both normal and sanitizer modes.
- [ ] The page shows errors clearly and highlights their position.

### Pitfalls

- Throwing the step-limit error from inside `emit` or a destructor. Limits are
  checked in `tick()` and `callFunction` only.
- Setting `maxCallDepth` without measuring. Run the deep-recursion test in the
  **sanitizer** build (which uses much bigger stack frames). If it crashes at
  200, lower the default and note the measurement in `NOTES.md`.
- Treating trace truncation as an error. The program is fine; only the
  recording stopped.

---

## Chapter 9: WebAssembly bridge

**Goal.** The same C++ engine runs inside the browser in a Web Worker and returns
the same JSON as the CLI.

**Concepts.** `extern "C"`, `EMSCRIPTEN_KEEPALIVE`, memory ownership across the
C++/JavaScript boundary, Emscripten flags, ES modules, Web Workers,
`postMessage`, timeouts.

### Files

```text
src/wasm_api.cpp                     (new)
CMakeLists.txt                       (EMSCRIPTEN branch)
scripts/build-wasm.sh                (new)
scripts/parity.mjs                   (new: Node test)
playground/public/minijs.js          (generated)
playground/public/minijs.wasm        (generated)
playground/public/examples/*.mini    (copied by the script)
playground/src/engineWorker.js       (new)
playground/src/engineClient.js       (new)
```

### Build steps

**9.1 Install Emscripten** (once):

```bash
git clone https://github.com/emscripten-core/emsdk.git ~/emsdk
cd ~/emsdk && ./emsdk install latest && ./emsdk activate latest
source ~/emsdk/emsdk_env.sh     # in every new terminal, or add to your shell profile
emcc --version
```

**9.2 Write `wasm_api.cpp`.** This is the whole browser API: two functions.

```cpp
#include <emscripten/emscripten.h>
#include "minijs/engine.hpp"

extern "C" {

// Runs the program and returns a malloc'ed, NUL-terminated UTF-8 JSON string
// in the shape of minijs::toJson. JavaScript must pass it to minijs_free.
EMSCRIPTEN_KEEPALIVE char* minijs_run(const char* source, int traceOn);

EMSCRIPTEN_KEEPALIVE void minijs_free(char* json);

}
```

`minijs_run` body steps:

1. Build `EngineOptions` with `trace = traceOn != 0`.
2. `result = minijs::run(source, options)`; `json = minijs::toJson(result, source)`.
3. `std::malloc(json.size() + 1)`, `std::memcpy` the bytes including the final
   `'\0'`, return the pointer.

`minijs_free` calls `std::free`.

Why a copy with `malloc`: the `std::string` is destroyed when `minijs_run`
returns, so its buffer cannot be handed to JavaScript. The rule is: **whoever
allocates, says who frees**, and you write that rule in the comment above the
function.

**9.3 Add the Emscripten branch to `CMakeLists.txt`.**

```cmake
if(EMSCRIPTEN)
    target_compile_options(minijs_core PRIVATE -fexceptions)

    add_executable(minijs_wasm src/wasm_api.cpp)
    target_link_libraries(minijs_wasm PRIVATE minijs_core)
    target_compile_options(minijs_wasm PRIVATE -fexceptions)
    set_target_properties(minijs_wasm PROPERTIES
        OUTPUT_NAME "minijs"
        SUFFIX ".js"
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/playground/public")
    target_link_options(minijs_wasm PRIVATE
        -fexceptions
        -sMODULARIZE=1
        -sEXPORT_ES6=1
        -sENVIRONMENT=web,worker,node
        -sALLOW_MEMORY_GROWTH=1
        -sSTACK_SIZE=5MB
        "-sEXPORTED_FUNCTIONS=['_minijs_run','_minijs_free','_malloc','_free']"
        "-sEXPORTED_RUNTIME_METHODS=['UTF8ToString','stringToNewUTF8']")
endif()
```

**9.4 Write `scripts/build-wasm.sh`.**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
emcmake cmake -S . -B build-wasm -DCMAKE_BUILD_TYPE=Release
cmake --build build-wasm
mkdir -p playground/public/examples
cp examples/*.mini playground/public/examples/
echo "WASM build written to playground/public/"
```

**9.5 Parity test (`scripts/parity.mjs`).** Run it with Node:

1. `import createMiniJs from '../playground/public/minijs.js'`, then
   `const mod = await createMiniJs()`.
2. For each `examples/*.mini` with a `.expected` file: call the engine (same steps
   as the worker below), join `output` with `\n`, compare.
3. Exit with code `1` on any difference.

This proves the browser engine and the CLI engine behave the same.

**9.6 Write `engineWorker.js`** (runs inside the worker). Steps:

1. On the first message, load the module once:
   `import(/* @vite-ignore */ new URL('minijs.js', self.location.origin + import.meta.env.BASE_URL).href)`,
   then call its default export (`createMiniJs()`), and keep the promise.
2. For each message `{ id, source }`:
   - `srcPtr = mod.stringToNewUTF8(source)`
   - `outPtr = mod._minijs_run(srcPtr, 1)`
   - `json = mod.UTF8ToString(outPtr)`
   - `mod._minijs_free(outPtr)` and `mod._free(srcPtr)` (both, every time)
   - `postMessage({ id, result: JSON.parse(json) })`
3. Wrap it in `try/catch`; on failure post `{ id, crash: String(err) }`.

**9.7 Write `engineClient.js`** (runs on the page). It hides the worker from
React:

```text
createEngineClient({ timeoutMs = 3000 }) -> { run(source) -> Promise<result>, dispose() }

state: worker, nextId, pending (map id -> {resolve, reject, timer})

startWorker(): worker = new Worker(new URL('./engineWorker.js', import.meta.url), { type: 'module' })
               worker.onmessage = resolve the matching pending entry, clear its timer

run(source):
    id = nextId++
    post {id, source}
    timer = setTimeout(timeoutMs):
        worker.terminate(); startWorker()
        reject every pending entry with "Engine timed out"
    return a promise stored in pending[id]
```

The step limit (Chapter 8) normally stops runaway programs first. The timeout is
the second safety net, for anything the engine did not anticipate.

**9.8 Prove it in the page.** Add a temporary "Run in browser" button that sends
a hard-coded program through `engineClient.run` and feeds the result into the
existing panels.

### Tests

| Check | Expected |
|-------|----------|
| `./scripts/build-wasm.sh` | produces `minijs.js` and `minijs.wasm` in `playground/public/` |
| `node scripts/parity.mjs` | every example matches |
| `while (true) {}` in the browser | `Limit` error from the engine (not a timeout) |
| syntax error in the browser | same error, line, and column as the CLI |
| a runtime error | shows as a MiniJS error, not as a worker crash (proves exceptions work in WASM) |
| run 100 times in a row | memory in DevTools stays flat (proves both `free` calls happen) |

### Done when

- [ ] The button runs a program in the browser and every panel shows its trace.
- [ ] `node scripts/parity.mjs` passes.
- [ ] Rebuilding is one command.

### Pitfalls

- **Exceptions are disabled by default in Emscripten.** Without `-fexceptions` on
  both compile and link, the first `throw` aborts the whole module. Every syntax
  error would look like a crash.
- **Small default stack.** Emscripten's default stack is small (64 KB in recent
  versions). A tree-walk interpreter uses a lot of stack per MiniJS call, so set
  `-sSTACK_SIZE` and re-test the deep recursion limit in the browser.
- Forgetting `_malloc`/`_free` in `EXPORTED_FUNCTIONS`; `stringToNewUTF8` needs
  `_malloc`.
- Importing `public/` files through normal `import` statements: Vite will
  complain. Load the engine with a runtime `import()` and `@vite-ignore` as above.
- Hard-coding `/minijs.js`: it breaks when the site is deployed under a sub-path.
  Always use `import.meta.env.BASE_URL`.

---

## Chapter 10: Live playground and deployment

**Goal.** A stranger opens a link, types code, and watches the engine process it,
with no explanation from you.

**Concepts.** React layout and state, debouncing, stale-request handling,
responsive CSS, static deployment, CI.

### Files

```text
playground/src/App.jsx                (final layout)
playground/src/panels/EditorPanel.jsx (new)
playground/src/panels/ExamplesMenu.jsx(new)
playground/src/panels/SyntaxHelp.jsx  (new)
playground/src/hooks/useEngine.js     (new)
playground/src/App.css
playground/vite.config.js             (base path)
.github/workflows/deploy.yml          (new, optional)
README.md                             (final)
```

### Build steps

**10.1 Final layout** (from `initialDoc.md`, section 10.1): a CSS grid with the
editor on the left, a tab bar (Tokens, AST, Scope, Stack) over the active panel on
the right, Output and Errors under it, and the step controls in a bottom bar.
Below 900 px wide, switch to a single column (editor first, then panels).

**10.2 `EditorPanel.jsx`.** Start with a `<textarea>`:

- monospace font, `spellCheck={false}`, Tab key inserts two spaces (handle
  `onKeyDown`, call `preventDefault`)
- a line-number gutter next to it (a `<pre>` with numbers, scroll-synced)
- error lines marked in the gutter (from `result.errors`)

Upgrade to CodeMirror only when everything else is done (see `initialDoc.md`).

**10.3 `useEngine.js` hook.** It connects the editor to the engine:

```text
useEngine(source, { debounceMs = 500, auto = true }) -> { result, running, crash, runNow }

- keep one engineClient in a useRef (create on mount, dispose on unmount)
- latestRequest = useRef(0)
- useEffect on [source, auto]:
      if !auto: return
      timer = setTimeout(debounceMs): runNow()
      cleanup: clearTimeout(timer)
- runNow():
      myId = ++latestRequest.current
      running = true
      result = await client.run(source)
      if myId !== latestRequest.current: ignore it (a newer run started)
      else setResult(result), running = false
```

Ignoring stale results is essential: when you type fast, an older, slower run can
finish **after** a newer one.

**10.4 Keep partial results.** If the new result has only lex errors, still show
its tokens. If it has syntax errors, still show tokens and keep the previous AST
dimmed with a "showing last valid tree" label. Users are always halfway through
typing; the page should not flash empty.

**10.5 `ExamplesMenu.jsx`.** A `<select>` listing the example files. Picking one
fetches `${import.meta.env.BASE_URL}examples/<name>.mini` and replaces the
editor text (ask for confirmation if the editor has unsaved edits).

**10.6 `SyntaxHelp.jsx`.** A collapsible box with the supported syntax table from
`initialDoc.md` section 8, so visitors know what they can try.

**10.7 Step behavior after a run.** Jump to the last step (finished state). The
Play button replays from step 0. Changing the code resets the step to the end of
the new result.

**10.8 Share links (optional, small).** Store the source in the URL hash
(`encodeURIComponent`), and on load read it back. Then anyone can share a program
as a link.

**10.9 Deploy.**

1. In `vite.config.js` set `base: '/<repo-name>/'` for GitHub Pages.
2. `npm run build` produces `playground/dist/`.
3. Either deploy `dist/` by hand, or add a GitHub Actions workflow that installs
   Emscripten (for example with the `mymindstorm/setup-emsdk` action), runs
   `scripts/build-wasm.sh`, runs the C++ tests and `parity.mjs`, builds the page,
   and publishes it with the official Pages actions.
4. Open the deployed link in a private window, on a phone, and in a second browser.

**10.10 Final README.** One paragraph about the project, the playground link, a
screenshot or GIF, build commands for CLI and WASM, the supported syntax, and a
link to these two docs.

### Tests

| Check | Expected |
|-------|----------|
| type `let x = 1` slowly | tokens update; syntax error *Expected ';'* appears; no crash, no flashing |
| type quickly for 5 seconds | only the latest result is shown |
| `while (true) {}` | limit error appears; the page stays responsive |
| every example from the menu | same output as `./build/minijs` |
| narrow window / phone | panels stack vertically, everything usable |
| reload a share link | same program appears |
| deployed site in a private window | loads, runs, steps |

### Done when

- [ ] A friend who has never seen the project opens the link, types a small
      program, and can explain what the Tokens, AST, Scope, and Stack panels showed.
- [ ] The deployed site works with no server and no console errors.
- [ ] The README explains how to build and run everything from a fresh clone.

### Pitfalls

- Running the engine on every keystroke without a debounce.
- Storing derived state (`deriveState` output) in React state. Compute it from
  `result` and `step` with `useMemo`; keep only the inputs in state.
- Calling `deriveState` from step 0 on every Play tick for huge traces. If it gets
  slow, cache a snapshot every 1,000 steps and start from the nearest one.
- Forgetting the `base` path, so the deployed site loads a blank page.

---

## Appendix A: Complete grammar

See Chapter 4, section 4.1. It is the single source of truth for the parser.

## Appendix B: Complete trace event reference

Every event also has `step` (0-based index) and `phase`.

| phase | event | fields | Chapter |
|-------|-------|--------|---------|
| `lex` | `token` | `type`, `text`, `line`, `col`, `offset`, `length` | 3 |
| `parse` | `node` | `id`, `kind`, `label`, `parent`, `start`, `end`, `line`, `col` | 4 |
| `parse` | `error` | `kind`, `message`, `line`, `col` | 4 |
| `exec` | `eval_start` | `node` | 5 |
| `exec` | `eval_end` | `node`, `value` | 5 |
| `exec` | `output` | `text` | 5 |
| `exec` | `error` | `kind`, `message`, `line`, `col` | 5 |
| `exec` | `stmt` | `node` | 6 |
| `exec` | `scope_push` | `scope`, `parent`, `kind` | 6 |
| `exec` | `scope_pop` | `scope` | 6 |
| `exec` | `declare` | `name`, `value`, `scope`, `const` | 6 |
| `exec` | `assign` | `name`, `value`, `scope` | 6 |
| `exec` | `call` | `name`, `args`, `depth`, `line`, `native` | 7 |
| `exec` | `return` | `name`, `value`, `depth` | 7 |
| `exec` | `branch` | `node`, `taken` | 7 |
| `exec` | `loop` | `node`, `iteration` | 7 |
| `meta` | `truncated` | `limit` | 3 / 8 |

Value encoding (`value`, `args`): JSON `null`, booleans, finite numbers, and
strings as themselves; functions as `{"fn":"name"}`; non-finite numbers as
`{"num":"Infinity"}` or `{"num":"-Infinity"}`.

## Appendix C: Language semantics summary

| Topic | Rule |
|-------|------|
| Types | null, boolean, number, string, function |
| Falsy | `false`, `null`, `0`, `""` |
| `+` | numeric add, or string join if either side is a string |
| Arithmetic | numbers only; `/` and `%` by zero are errors |
| Comparison | numbers with numbers, strings with strings |
| Equality | strict; `===` is rejected with a hint |
| Logic | `&&` and `\|\|` short-circuit and return an operand |
| Variables | `let` (mutable), `const` (must be initialized, immutable) |
| Scope | block-scoped, lexical; shadowing allowed, redeclaring in the same scope is not |
| Functions | declared with `function`, stored as `const`, exact arity, return `null` by default, no hoisting |
| Output | `console.log(a, b, ...)` joins display strings with spaces |
| Statements | semicolons required |

## Appendix D: Error message catalog

Fill in the last column as you implement each message.

| Kind | Message | Chapter | Test name |
|------|---------|---------|-----------|
| lexical | Unexpected character 'X' | 2 | |
| lexical | Unterminated string | 2 | |
| lexical | Unknown escape '\X' | 2 | |
| lexical | '===' is not supported; MiniJS '==' is already strict | 2 | |
| lexical | Expected '&&' / Expected '\|\|' | 2 | |
| syntax | Expected an expression | 4 | |
| syntax | Expected ';' after expression | 4 | |
| syntax | Expected ')' after expression | 4 | |
| syntax | Expected '}' after block | 4 | |
| syntax | Invalid assignment target | 4 | |
| syntax | const 'x' must be initialized | 4 | |
| syntax | 'return' outside a function | 4 | |
| syntax | Property access is not supported (only console.log) | 4 | |
| syntax | Duplicate parameter 'x' | 4 | |
| syntax | Code is nested too deeply | 8 | |
| runtime | Undefined variable 'x' | 5 | |
| runtime | Operator 'op' needs numbers, got A and B | 5 | |
| runtime | Cannot compare A and B | 5 | |
| runtime | Division by zero | 5 | |
| runtime | 'x' is not a function | 5 | |
| runtime | 'x' is already declared in this scope | 6 | |
| runtime | Cannot assign to const 'x' | 6 | |
| runtime | f expects N arguments, got M | 7 | |
| limit | Step limit of N reached (infinite loop?) | 8 | |
| limit | Call depth limit of N reached (infinite recursion?) | 8 | |
| limit | Output limit of N lines reached | 8 | |
| internal | Out of memory / Internal error: ... | 8 | |

## Appendix E: Master progress checklist

Copy this into `NOTES.md` and tick it off.

- [ ] **Ch 1** Builds, CLI runs, test runner works, sanitizer build works
- [ ] **Ch 2** Lexer complete with positions and errors; `--tokens`
- [ ] **Ch 3** Trace JSON; `--trace`; page shows source + tokens with highlights
- [ ] **Ch 4** Parser with precedence and error recovery; `--ast`; AST panel
- [ ] **Ch 5** Expressions + `console.log`; golden example test; step controls + output
- [ ] **Ch 6** `let`/`const`/blocks/assignment; scope panel
- [ ] **Ch 7** `if`/`while`/functions/recursion; call stack panel
- [ ] **Ch 8** All limits; robustness tests clean under sanitizers; errors panel
- [ ] **Ch 9** WASM build + worker + parity test
- [ ] **Ch 10** Live editor, examples, responsive layout, deployed link

When every box is ticked, go back to the optional extensions in `initialDoc.md`
(bytecode VM, objects and arrays, closures in depth). The trace design you built
here is how the playground will show those too: new engine part, new trace
events, new panel.
