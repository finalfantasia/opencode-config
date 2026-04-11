---
name: clojure-delimiter-repair
description: A standalone CLI tool for automatically detecting and fixing errors with delimiters (parentheses, brackets, and braces) in Clojure sources (.clj, .cljs, .cljc, and .edn). Use this when you encounter delimiter errors (unbalanced/mismatched delimiters).
---

# Clojure Delimiters Repair

The command `clj-paren-repair` is installed on your path.

**IMPORTANT: Always run `clj-paren-repair` on the file immediately after any edit that produces LSP or compiler delimiter diagnostics, and before making any further edits.** This tool is the reliable way to fix mismatched delimiters. If the tool doesn't work, report to the user that they need to fix the delimiter error manually.

The tool automatically formats files with cljfmt when it processes them.

```bash
# Fix delimiter errors in a single file
clj-paren-repair src/my_app/core.clj

# Fix multiple files at once
clj-paren-repair src/my_app/core.clj src/my_app/utils.clj

# Fix all Clojure files in a directory
clj-paren-repair src/**/*.clj

# Show help
clj-paren-repair --help
```

**Key benefits:**
- Automatic delimiter error detection and repair
- Batch processing of multiple files
- Intelligent backend selection (parinfer-rust or parinferish)
- No configuration needed
- Works on any Clojure file (.clj, .cljs, .cljc, .edn)

## Core Concepts

### Automatic Delimiter Repair

The tool detects and fixes common delimiter errors in Clojure code:

```clojure
;; Before repair (missing closing delimiter)
(defn broken [x]
  (let [result (* x 2]
    result))

;; After clj-paren-repair
(defn broken [x]
  (let [result (* x 2)]
    result))
```

**How it works:**
1. Parses file with edamame to detect delimiter errors
2. If errors found, applies parinfer-rust (or parinferish fallback) to repair
3. Writes repaired code back to file
4. Reports what was fixed

### Intelligent Backend Selection

The tool automatically chooses the best available delimiter repair backend:

- **parinfer-rust** - Preferred when available (faster, battle-tested)
- **parinferish** - Pure Clojure fallback (no external dependencies)

Both backends provide equivalent delimiter fixing functionality.

### Batch Processing

Process multiple files in a single command:

```bash
# Fix all files in src/ directory
clj-paren-repair src/my_app/*.clj

# Fix specific files
clj-paren-repair file1.clj file2.clj file3.clj

# Use shell globbing
clj-paren-repair **/*.clj
```

Each file is processed independently with individual success/failure reporting.

## Common Workflows

### Workflow 1: Fix LLM-Generated Code

LLMs often generate Clojure code with delimiter errors. Fix them before using:

```bash
# After LLM writes code to file
clj-paren-repair src/generated_code.clj

# Output shows what was fixed:
# clj-paren-repair Results
# ========================
#
#   src/generated_code.clj: Delimiter errors fixed and formatted [delimiter-fixed]
#
# Summary:
#   Success: 1
#   Failed:  0
```

**Use case:**
- Post-processing LLM-generated code
- Fixing Claude Code output
- Repairing code from other AI tools

### Workflow 2: Clean Up Manually Edited Code

Fix delimiter errors introduced during manual editing:

```bash
# After manual edits with errors
clj-paren-repair src/my_app/core.clj

# File is fixed in place
# Continue editing with corrected code
```

**Use case:**
- Quick fixes during development
- Repairing broken files
- Cleaning up after incomplete edits

### Workflow 3: Batch Fix Multiple Files

Process many files at once:

```bash
# Fix all Clojure files in project
clj-paren-repair src/**/*.clj test/**/*.clj

# Output shows results for each file:
# clj-paren-repair Results
# ========================
#
#   src/app/core.clj: Delimiter errors fixed and formatted [delimiter-fixed]
#   src/app/utils.clj: No changes needed
#   test/app/core_test.clj: Delimiter errors fixed and formatted [delimiter-fixed]
#
# Summary:
#   Success: 3
#   Failed:  0
```

**Use case:**
- Cleaning up project-wide issues
- Post-processing bulk changes
- Preparing code for commit

### Workflow 4: Verify Files Are Valid

Check if files have delimiter errors without making changes:

```bash
# Run clj-paren-repair
clj-paren-repair src/my_app/core.clj

# If output shows "No changes needed", file is valid
# If output shows "Delimiter errors fixed", file had issues (now fixed)
```
