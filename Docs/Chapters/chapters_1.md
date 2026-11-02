# Chapter 1: Foundation — build guide

This chapter is your **hands-on worksheet** for starting MiniJS. Read
`Docs/buildBlueprint.md` (Chapter 1) and `Docs/initialDoc.md` for the big picture;
**this file tells you what to create, in what order, and how to write each piece
yourself.**

You are **not** building a lexer, parser, or playground yet. You are laying the
scaffold: CMake, a core library, a CLI wrapper, and a tiny test runner.

---

## 1. What you will have when Chapter 1 is done

| Piece | Name | Role |
|-------|------|------|
| Library | `minijs_core` | Contains `minijs::run(source, options)` → `EngineResult`. No printing, no files, no `exit`. |
| Program | `minijs` | Reads a `.mini` file, calls `run`, prints output/errors, returns a Unix exit code. |
| Tests | `minijs_tests` | Runs your `TEST(...)` cases; `ctest` invokes it from the project root. |

**Design rule (keep for the whole project):**

```text
main.cpp  ──reads file, prints──>  minijs::run()  in engine.cpp
wasm_api.cpp (Chapter 9) ─────────>  same run()
```

The engine **returns data** (`EngineResult`). Wrappers **do I/O**.

---

## 2. C++ ideas you practice in this chapter

- **CMake**: one library target + two executables, include paths, C++20, warnings.
- **Headers vs sources**: public API in `include/minijs/engine.hpp`, implementation in `src/engine.cpp`.
- **`#pragma once`**, **`namespace minijs`**, **`std::string_view`** for source text without copying.
- **CLI**: `argc`/`argv`, reading a whole file with `std::ifstream`.
- **Exit codes**: `0` success, `64` usage, `66` missing file, `65` / `70` for errors (used more from Chapter 2 onward).
- **Test macros**: static registration so each test file registers itself at startup.

---

## 3. Folder layout to create

Create these paths under your **repository root** (the folder that contains
`CMakeLists.txt` — for you, that is the `miniJs` project root, not inside `Docs/`):

```text
CMakeLists.txt
.gitignore
README.md
NOTES.md
examples/
  hello.mini
  hello.expected
include/minijs/
  engine.hpp
src/
  engine.cpp
  main.cpp
tests/
  test.hpp
  test_main.cpp
  engine_tests.cpp
```

`Docs/` stays documentation only; **all code lives at the repo root** as above.

---

## 4. Recommended order (write code in this sequence)

1. `CMakeLists.txt` — so you can compile as soon as you have one `.cpp`.
2. `include/minijs/engine.hpp` — types and `run` declaration.
3. `src/engine.cpp` — stub `run` + `errorKindName`.
4. `src/main.cpp` — CLI (needs `engine.hpp`).
5. `tests/test.hpp` — macros + registry declarations.
6. `tests/test_main.cpp` — registry implementation + runner.
7. `tests/engine_tests.cpp` — first real tests.
8. `examples/hello.mini`, `examples/hello.expected`.
9. `.gitignore`, short `README.md`, first entry in `NOTES.md`.
10. Configure, build, fix warnings, run checks in section 10.

---

## 5. File-by-file: what to write

### 5.1 `CMakeLists.txt`

**What it is:** The build script. For Chapter 1 you can copy the **exact content**
from `buildBlueprint.md` section **1.1** (infrastructure is given in full there).

**What you must understand:**

- `minijs_core` is a **static library** built from `src/engine.cpp`.
- `target_include_directories(... PUBLIC include)` lets any target that links
  `minijs_core` write `#include "minijs/engine.hpp"`.
- `-Wall -Wextra -Wpedantic` on the core catches real bugs early.
- The block `if(MINIJS_SANITIZE)` must appear **before** `add_library(minijs_core ...)`.
- `if(NOT EMSCRIPTEN)` skips the CLI and tests when you compile for WASM later.
- `add_test(... WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})` runs tests with the repo
  root as cwd (needed when tests read `examples/` in later chapters).

**Your job:** Create the file, then run configure + build (section 10).

---

### 5.2 `include/minijs/engine.hpp`

**What it is:** The **public face** of the engine. `main.cpp` (and later WASM)
should only need this header for running programs.

**What to write:** Copy the declarations from `buildBlueprint.md` **1.2** into
your file. You should have:

| Type / function | Purpose |
|-----------------|--------|
| `ErrorKind` | Lexical, Syntax, Runtime, Limit, Internal |
| `errorKindName(ErrorKind)` | Lowercase string for CLI and JSON later |
| `EngineError` | kind + message + line + col |
| `EngineOptions` | trace flag + limits (limits used from Chapter 8) |
| `EngineResult` | ok, output lines, errors, traceJson, traceTruncated |
| `run(source, options)` | Main entry: source in, result out |

**How to write it:**

1. `#pragma once`
2. Includes: `<cstddef>`, `<string>`, `<string_view>`, `<vector>`
3. Everything inside `namespace minijs { ... }`
4. No function bodies in the header (only declarations).

**Do not** add lexer/parser headers here in Chapter 1.

---

### 5.3 `src/engine.cpp`

**What it is:** Implementation of `run` and `errorKindName`. This file must stay
**free of I/O** — no `<iostream>`, no reading paths, no `exit()`.

**Includes you will need (typical):**

- `"minijs/engine.hpp"`
- `<cctype>` or similar if you write a whitespace helper

#### Function: `errorKindName`

**Algorithm:**

1. `switch (kind)` on every `ErrorKind` enumerator.
2. Return a string literal for each:
   - `Lexical` → `"lexical"`
   - `Syntax` → `"syntax"`
   - `Runtime` → `"runtime"`
   - `Limit` → `"limit"`
   - `Internal` → `"internal"`
3. In `default`, return `"internal"` (covers unexpected values).

Use `-Wall`; if you add a new enum value later and forget a case, the compiler
may warn if you structure the switch carefully.

#### Helper: “is source only whitespace?” (your design)

You need this for the stub `run`. Options:

- Loop over `source` and check each char with `std::isspace(static_cast<unsigned char>(c))`.
- Or use a small loop: if every character is space, tab, newline, carriage return, treat as empty.

**Do not** use `isspace` on a negative `char` (undefined behavior).

#### Function: `run` (Chapter 1 stub)

**Algorithm:**

1. Create `EngineResult result` (defaults: `ok = true`, empty vectors, `traceJson = "[]"`).
2. If `source.empty()` **or** your whitespace helper says “only whitespace”:
   return `result` unchanged.
3. For **any other** source (including `console.log(...)` in `hello.mini`):
   still return `result` with `ok = true`, no output, no errors.
4. Chapter 2 will replace step 3 with lexing.

**Note:** `options` is unused in Chapter 1. Still take it by `const EngineOptions&`
so the signature never changes.

---

### 5.4 `src/main.cpp`

**What it is:** The **command-line wrapper**. All printing and file reading live here.

**Includes (typical):**

- `"minijs/engine.hpp"`
- `<iostream>`, `<fstream>`, `<sstream>`, `<string>`, `<vector>`

#### Step A — Usage string

Write a function or block that prints something like:

```text
Usage: minijs <file> [--tokens] [--ast] [--trace <out.json>]
```

Mention future flags even if they do nothing in Chapter 1.

#### Step B — Parse `argv`

**Expected shape:** `minijs <file> [optional flags...]`

**Rules for Chapter 1:**

| Situation | Action |
|-----------|--------|
| No arguments, or first arg is a flag (starts with `-`) | Print usage, `return 64` |
| First arg is the **file path** | Remember it |
| Later args: `--tokens`, `--ast` | You may ignore or print “not implemented yet” |
| `--trace` | Must be followed by a path; if missing, usage → `64` |
| Unknown flag | Usage → `64` |

Keep parsing simple (loop from `argv[1]`). You do not need a full getopt library.

#### Step C — Read the file

**Algorithm:**

1. Open `std::ifstream file(path)`.
2. If `!file`, print a clear error to `std::cerr` (include the path), `return 66`.
3. Read all bytes: `std::ostringstream buffer; buffer << file.rdbuf();`
4. `std::string source = buffer.str();`

#### Step D — Run the engine

```cpp
minijs::EngineOptions options{};
minijs::EngineResult result = minijs::run(source, options);
```

#### Step E — Print results

1. For each string in `result.output`, print to `std::cout` with a newline.
2. For each `EngineError e` in `result.errors`, print to `std::cerr`:

   ```text
   <filename>:<line>:<col>: <kind> error: <message>
   ```

   Use `minijs::errorKindName(e.kind)` for `<kind>`. If `line` or `col` is `0`,
   still print them (tests may use `0` before real parsing).

#### Step F — Exit code

**Algorithm:**

1. If `result.ok`, `return 0`.
2. If any error has kind `Lexical` or `Syntax`, `return 65`.
3. If any error has kind `Runtime`, `Limit`, or `Internal`, `return 70`.
4. If multiple errors exist, pick the code from the **first** error, or define a
   fixed priority (document your choice in `NOTES.md`).

With the Chapter 1 stub, running `hello.mini` should exit **0** and print **nothing**
yet (no `console.log` until Chapter 5).

---

### 5.5 `tests/test.hpp`

**What it is:** A minimal test framework (no GoogleTest).

**What to write:**

1. **`TestCase`** — `const char* name`, `void (*fn)()`.
2. **`testRegistry()`** — returns reference to a function-local static
   `std::vector<TestCase>` (declare in header; **define** in `test_main.cpp`).
3. **`TestRegistrar`** — constructor takes `name` and `fn`, pushes into registry
   (implement in `test_main.cpp`).
4. **`TestFailure`** — holds `std::string message`.
5. **`TEST(name)` macro** — expands to:
   - forward declare `static void name();`
   - static `TestRegistrar name##_registrar(#name, name);`
   - start definition `static void name()`
6. **`CHECK(cond)`** — if `!(cond)`, throw `TestFailure` with message containing
   `__FILE__`, `__LINE__`, and `#cond` (stringify the condition).
7. **`CHECK_EQ(a, b)`** — if `a != b`, throw `TestFailure` showing both values
   (use `std::ostringstream` or `std::to_string` where needed).

**Tip:** Wrap macro bodies in `do { ... } while (0)` if you use them after `if`
without braces.

---

### 5.6 `tests/test_main.cpp`

**What to write:**

1. **Define** `testRegistry()` and `TestRegistrar::TestRegistrar` (push back on the vector).
2. **`main()`**:
   - Counters `passed`, `failed`.
   - For each entry in `testRegistry()`:
     - `try { test.fn(); print "PASS" + name; ++passed; }`
     - `catch (const TestFailure& e) { print "FAIL" + name + e.message; ++failed; }`
     - `catch (const std::exception& e) { print "ERROR" + name + e.what(); ++failed; }`
   - Print summary: `N passed, M failed`.
   - `return failed > 0 ? 1 : 0`.

---

### 5.7 `tests/engine_tests.cpp`

**What to write:** Include `"test.hpp"` and `"minijs/engine.hpp"`. Use `TEST(...)`.

| Test | Code idea | Expect |
|------|-----------|--------|
| Empty source | `auto r = minijs::run("", {});` | `CHECK(r.ok); CHECK(r.output.empty()); CHECK(r.errors.empty());` |
| Whitespace only | `run("   \n  ", {})` | `CHECK(r.ok);` |
| Error kind name | `CHECK_EQ(minijs::errorKindName(minijs::ErrorKind::Syntax), std::string("syntax"));` | exact string `"syntax"` |

**Optional once:** add `CHECK(1 == 2)`, run tests, confirm you see `FAIL`, then
**delete** that test.

Do **not** expect `hello.mini` to produce output in Chapter 1 tests.

---

### 5.8 Example files

**`examples/hello.mini`** — one line (MiniJS-like syntax for later):

```javascript
console.log("Hello, MiniJS");
```

**`examples/hello.expected`** — exact expected stdout **when the engine works**
(Chapter 5):

```text
Hello, MiniJS
```

No extra blank line unless you decide that policy later and keep it consistent.

---

### 5.9 `.gitignore`

**What to write:** Ignore build trees and common noise, for example:

- `build/`, `build-asan/`, `build-wasm/`
- `.cache/`, `compile_commands.json` (optional)
- OS/IDE: `.DS_Store`, `.vscode/` (if you do not commit settings)

---

### 5.10 `README.md`

**What to write (short):**

- One paragraph: MiniJS is a small JS-like interpreter in C++ (link to `Docs/initialDoc.md`).
- Chapter 1 status: scaffold only.
- Commands:

  ```bash
  cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
  cmake --build build
  ./build/minijs examples/hello.mini
  ctest --test-dir build --output-on-failure
  ```

---

### 5.11 `NOTES.md`

**What to write:** A running log. Minimum for finishing Chapter 1:

- Date
- “Completed Chapter 1 scaffold”
- Any choice you made (e.g. how you parse argv, exit code when multiple errors)

---

## 6. What you deliberately do **not** build in Chapter 1

- No `token.hpp`, `lexer.hpp`, or lexer tests
- No `--tokens` / `--ast` / `--trace` behavior (flags may appear in usage only)
- No React playground
- No `toJson`, no trace events (Chapter 3)

---

## 7. Verify your work

Run from the **repository root** (where `CMakeLists.txt` lives):

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/minijs examples/hello.mini
echo $?    # expect 0

./build/minijs missing.mini
echo $?    # expect 66

ctest --test-dir build --output-on-failure

cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug -DMINIJS_SANITIZE=ON
cmake --build build-asan
./build-asan/minijs_tests
```

Fix every compiler **warning** before you call Chapter 1 done.

---

## 8. Done checklist

Copy this into `NOTES.md` and tick when true:

- [ ] `cmake --build build` — zero warnings
- [ ] `./build/minijs examples/hello.mini` — exits `0`
- [ ] `./build/minijs missing.mini` — exits `66`, clear message on stderr
- [ ] `ctest --test-dir build --output-on-failure` — all pass
- [ ] Sanitizer build + `minijs_tests` — pass
- [ ] `NOTES.md` has at least one entry
- [ ] `engine.cpp` does **not** include `<iostream>` or read files

---

## 9. Common mistakes

| Mistake | Why it hurts | Fix |
|---------|----------------|-----|
| Sanitizer flags after `add_library` | Core not sanitized | Move `if(MINIJS_SANITIZE)` block above `add_library` |
| No `WORKING_DIRECTORY` in `add_test` | Later tests cannot open `examples/` | Use `${CMAKE_SOURCE_DIR}` |
| Printing from `engine.cpp` | Breaks WASM and testing | Move all I/O to `main.cpp` |
| `isspace(char)` without `unsigned char` cast | UB on non-ASCII bytes | Cast before `isspace` |
| Putting code under `Docs/` only | CMake expects root layout | Code at repo root, docs in `Docs/` |

---

## 10. After Chapter 1

Open **`Docs/buildBlueprint.md` → Chapter 2: Lexer**. You will add `token.hpp`,
`lexer.hpp`, implement scanning, wire lexing into `run`, and add `--tokens` to
the CLI.

When you are stuck on a specific file, paste your code and the compiler or test
output in chat; use this doc as the checklist for what that file should do.

---

## Quick reference — public API (Chapter 1)

You implement these in `engine.hpp` / `engine.cpp`:

```cpp
namespace minijs {

enum class ErrorKind { Lexical, Syntax, Runtime, Limit, Internal };
const char* errorKindName(ErrorKind kind);

struct EngineError { /* kind, message, line, col */ };
struct EngineOptions { /* trace, maxSteps, maxCallDepth, maxTraceEvents, maxOutputLines */ };
struct EngineResult { /* ok, output, errors, traceJson, traceTruncated */ };

EngineResult run(std::string_view source, const EngineOptions& options);

}
```

Everything else in Chapter 1 is CMake, `main.cpp`, tests, and example files.
