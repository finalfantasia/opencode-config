---
name: clojure-repl
description: |
  Guide for REPL-driven development in Clojure. Use when working interactively, testing code, exploring libraries, looking up documentation, debugging exceptions, or developing iteratively. Covers clojure.repl utilities for exploration, debugging, and iterative development. Essential for the Clojure development workflow.
---

# Clojure REPL

## REPL-Driven Development

**The REPL is your primary development environment, not a secondary tool.**

Before reading or editing any file, ask: "Can I test this in the REPL first?" The answer is almost always YES.

### When to Use REPL

Use REPL via [clojure-eval](../clojure-eval/SKILL.md) for:
- Debugging errors or failures
- Understanding existing code
- Testing hypotheses and edge cases
- Experimenting with solutions
- Checking function signatures and documentation

### Core Workflow

1. Use [clojure-eval](../clojure-eval/SKILL.md) to connect to REPL
2. Load the relevant namespace
3. Test with real data
4. Verify edge cases (nil, empty, zero, etc.)
5. Experiment with fixes
6. Edit files (use [clojure-delimiter-repair](../clojure-delimiter-repair/SKILL.md) for syntax errors)
7. Reload and verify

**Note**: If you encounter unbalanced delimiters when editing files, use the [clojure-delimiter-repair](../clojure-delimiter-repair/SKILL.md) skill instead of trying to fix them manually.

### Common Patterns

**Test edge cases immediately:**
```clojure
(my-fn nil)    ; nil handling
(my-fn {})     ; empty collections
(my-fn "")     ; empty strings
(my-fn 0)      ; zero
```

**Build incrementally:**
```clojure
;; Step 1: Test the transformation
(map inc [1 2 3])

;; Step 2: Make it a function
(defn inc-all [xs] (map inc xs))

;; Step 3: Test edge cases
(inc-all [])   ; => ()
(inc-all nil)  ; => NullPointerException! Fix it.
```

**Understand before changing:**
```clojure
;; Load the namespace
(require '[my.namespace :as ns])

;; Test current behavior
(ns/the-function test-data)

;; Check edge cases
(ns/the-function nil)
(ns/the-function {})

;; Read the implementation
(source ns/the-function)

;; Now edit with confidence
```

## REPL Basics

The REPL (Read-Eval-Print Loop) reads expressions, evaluates them, prints results, and loops.

```clojure
user=> (+ 2 3)
5
user=> (defn greet [name] (str "Hello, " name))
#'user/greet
user=> (greet "World")
"Hello, World"
```

### Side Effects vs Return Values

```clojure
user=> (println "Hello World")
Hello World    ; <- Side effect: printed by your code
nil            ; <- Return value: printed by the REPL
```

### Namespace Management

```clojure
;; Basic require
(require '[clojure.string])
(clojure.string/upper-case "hello")  ; => "HELLO"

;; With alias (recommended)
(require '[clojure.string :as str])
(str/upper-case "hello")  ; => "HELLO"

;; Reload after file changes
(require 'my.namespace :reload)
```

## clojure.repl Utilities

Load utilities first:
```clojure
(require '[clojure.repl :refer :all])
```

### Exploration Functions

**`(all-ns)` - List all namespaces**
```clojure
(map ns-name (all-ns))
; => (clojure.core clojure.string user ...)
```

**`(dir namespace)` - List functions in a namespace**
```clojure
(dir clojure.string)
; blank?
; capitalize
; upper-case
; ...
```

**`(doc symbol)` - View documentation**
```clojure
(doc map)
(doc clojure.string/upper-case)
```

**`(source symbol)` - View source code**
```clojure
(source some?)
; Shows implementation
```

**`(apropos "pattern")` - Search for symbols**
```clojure
(apropos "map")
; (clojure.core/map clojure.core/map-indexed ...)
```

**`(find-doc "pattern")` - Search documentation**
```clojure
(find-doc "indexed")
; Shows all docs containing "indexed"
```

### Debugging Functions

**`(pst)` - Print stack trace**
```clojure
user=> (/ 1 0)
; ArithmeticException: Divide by zero

user=> (pst)
; Shows full stack trace
```

**Special REPL vars:**
- `*e` - Last exception thrown
- `*1` - Result of last expression
- `*2` - Result of second-to-last expression
- `*3` - Result of third-to-last expression

**`(root-cause *e)` - Find original exception**

**`(demunge "class$name")` - Readable class names**
```clojure
(demunge "clojure.core$map")
; => "clojure.core/map"
```

## Quick Reference

| Task | Function | Example |
|------|----------|---------|
| List namespaces | `all-ns` | `(map ns-name (all-ns))` |
| List vars in namespace | `dir` | `(dir clojure.string)` |
| Show documentation | `doc` | `(doc map)` |
| Show source code | `source` | `(source some?)` |
| Search symbols by name | `apropos` | `(apropos "index")` |
| Search documentation | `find-doc` | `(find-doc "sequence")` |
| Print stack trace | `pst` | `(pst)` |
| Get root cause | `root-cause` | `(root-cause *e)` |
| Add library at REPL (1.12+) | `add-lib` | `(add-lib 'org.clojure/data.json)` |
| Add multiple libraries (1.12+) | `add-libs` | `(add-libs '{org.clojure/data.json {:mvn/version "2.4.0"}})` |
| Sync deps.edn libraries (1.12+) | `sync-deps` | `(sync-deps)` |

### Dynamic Library Loading (Clojure 1.12+)

Add dependencies at the REPL without restarting:

```clojure
(require '[clojure.repl.deps :refer [add-lib add-libs sync-deps]])

;; Add a single library
(add-lib 'org.clojure/data.json)
(require '[clojure.data.json :as json])
(json/write-str {:foo "bar"})

;; Add multiple libraries with coordinates
(add-libs '{org.clojure/data.json {:mvn/version "2.4.0"}
            org.clojure/data.csv {:mvn/version "1.0.1"}})

;; Sync with deps.edn
(sync-deps)  ; Loads any libs in deps.edn not yet on classpath
```

**Note**: Requires Clojure 1.12+ and a valid parent `DynamicClassLoader`.

## Common Issues

### "Unable to resolve symbol"
```clojure
user=> (str/upper-case "hello")
; CompilerException: Unable to resolve symbol: str/upper-case
```

**Solution**: Require the namespace first:
```clojure
(require '[clojure.string :as str])
(str/upper-case "hello")  ; => "HELLO"
```

### "No documentation found"
**Solution**: Require namespace before using `doc`:
```clojure
(require '[clojure.set])
(doc clojure.set/union)  ; Now works
```

### Stale definitions after file changes
```clojure
;; Wrong - might keep old definitions
(require 'my.namespace)

;; Right - forces reload
(require 'my.namespace :reload)
```

### Unbalanced delimiters in files
**Solution**: Use the [clojure-delimiter-repair](../clojure-delimiter-repair/SKILL.md) skill instead of trying to fix parentheses manually:
```bash
clj-paren-repair src/my_app/core.clj
```

## Example: Exploring an Unknown Namespace

```clojure
;; 1. Discover available namespaces
(map ns-name (all-ns))

;; 2. Require the namespace
(require '[clojure.string :as str])

;; 3. Explore contents
(dir clojure.string)

;; 4. Find relevant functions
(apropos "upper")

;; 5. Get documentation
(doc str/upper-case)

;; 6. View implementation
(source str/upper-case)

;; 7. Test it
(str/upper-case "hello")  ; => "HELLO"
```
