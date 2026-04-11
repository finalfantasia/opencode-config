---
name: clojure-eval
description: Evaluate Clojure code via nREPL using clj-nrepl-eval. Use this when you need to test code, check if edited files compile, verify function behavior, or interact with a running REPL session.
---

# Clojure REPL Evaluation

## MANDATORY USAGE

**This skill should be your FIRST tool for all coding tasks.**

## When to Use This Skill

Use this skill when you need to:
- **Verify that edited Clojure files compile and load correctly**
- Test function behavior interactively
- Check the current state of the REPL
- Debug code by evaluating expressions
- Test edge cases to find bugs instantly
- Require or load namespaces for testing
- Validate that code changes work before committing

## How It Works

The `clj-nrepl-eval` command evaluates Clojure code against an nREPL server. **Session state persists between evaluations**, so you can require a namespace in one evaluation and use it in subsequent calls. Each host:port combination maintains its own session file.

## Instructions

### 0. Discover and select nREPL server

#### Non-sandboxed environments

First, discover what nREPL servers are running in the current directory:

```bash
clj-nrepl-eval --discover-ports
```

This will show all nREPL servers (Clojure, Babashka, shadow-cljs, etc.) running in the current project directory.

#### Sandboxed environments

In sandboxed environments, connecting to an already running local nREPL server may fail with `java.net.SocketException: Operation not permitted`, even when the server is healthy and `.nrepl-port` is correct. Treat this as a sandbox socket restriction, not as evidence that the REPL is missing.

When discovery cannot work because you are inside a sandbox, or when discovery returns nothing but the project clearly has a running REPL, fall back to the manual nREPL port-file checks.

Check whether the current directory has a conventional nREPL port file:

```bash
sed -n '1p' .nrepl-port
```

If `.nrepl-port` exists, try that port. This is the standard signal that a project already has a running nREPL server.

For shadow-cljs projects, also check the port files under `.shadow-cljs/`:

```bash
sed -n '1p' .shadow-cljs/nrepl.port
sed -n '1p' .shadow-cljs/socket-repl.port
```

Use `.shadow-cljs/nrepl.port` with `clj-nrepl-eval -p <PORT>`.

If connecting to a port fails with a sandbox socket restriction, rerun the same `clj-nrepl-eval -p <PORT> ...` command with escalated permissions and use the prefix rule `["clj-nrepl-eval", "-p"]` so future nREPL evaluations can connect without repeated approval prompts.

### 1. Evaluate Clojure Code

> Evaluation automatically connects to the given port

Use the `-p` flag to specify the port and pass your Clojure code.

**Recommended: Pass code as a command-line argument:**
```bash
clj-nrepl-eval -p <PORT> "(+ 1 2 3)"
```

**For multiple expressions (single line):**
```bash
clj-nrepl-eval -p <PORT> "(def x 10) (+ x 20)"
```

**Alternative: Using heredoc (may require permission approval for multiline commands):**
```bash
clj-nrepl-eval -p <PORT> <<'EOF'
(def x 10)
(+ x 20)
EOF
```

**Alternative: Via stdin pipe:**
```bash
echo "(+ 1 2 3)" | clj-nrepl-eval -p <PORT>
```

### 2. Display nREPL Sessions

**Discover all nREPL servers in current directory:**
```bash
clj-nrepl-eval --discover-ports
```
Shows all running nREPL servers in the current project directory, including their type (clj/bb/basilisp) and whether they match the current working directory.

**Check previously connected sessions:**
```bash
clj-nrepl-eval --connected-ports
```
Shows only connections you have made before (appears after first evaluation on a port).

### 3. Common Patterns

**Require a namespace (always use :reload to pick up changes):**
```bash
clj-nrepl-eval -p <PORT> "(require '[my.namespace :as ns] :reload)"
```

**Test a function after requiring:**
```bash
clj-nrepl-eval -p <PORT> "(ns/my-function arg1 arg2)"
```

**Check if a file compiles:**
```bash
clj-nrepl-eval -p <PORT> "(require 'my.namespace :reload)"
```

**Multiple expressions:**
```bash
clj-nrepl-eval -p <PORT> "(def x 10) (* x 2) (+ x 5)"
```

**Complex multiline code (using heredoc):**
```bash
clj-nrepl-eval -p <PORT> <<'EOF'
(def x 10)
(* x 2)
(+ x 5)
EOF
```
*Note: Heredoc syntax may require permission approval.*

**With custom timeout (in milliseconds):**
```bash
clj-nrepl-eval -p <PORT> --timeout 5000 "(long-running-fn)"
```

**Reset the session (clears all state):**
```bash
clj-nrepl-eval -p <PORT> --reset-session
clj-nrepl-eval -p <PORT> --reset-session "(def x 1)"
```

## Available Options

- `-p, --port PORT` - nREPL port (required)
- `-H, --host HOST` - nREPL host (default: 127.0.0.1)
- `-t, --timeout MILLISECONDS` - Timeout (default: 120000 = 2 minutes)
- `-r, --reset-session` - Reset the persistent nREPL session
- `-c, --connected-ports` - List previously connected nREPL sessions
- `-d, --discover-ports` - Discover nREPL servers in current directory
- `-h, --help` - Show help message

## Important Notes

- **Prefer command-line arguments:** Pass code as quoted strings: `clj-nrepl-eval -p <PORT> "(+ 1 2 3)"`. This should work with existing permissions; if not (e.g., inside a sandbox), run with escalated permissions as described above.
- **Heredoc for complex code:** Use heredoc (`<<'EOF' ... EOF`) for truly multiline code, but note it may require permission approval
- **Sessions persist:** State (vars, namespaces, loaded libraries) persists across invocations until the nREPL server restarts or `--reset-session` is used
- **Automatic delimiter repair:** The tool automatically repairs missing or mismatched parentheses
- **Always use :reload:** When requiring namespaces, use `:reload` to pick up recent changes
- **Default timeout:** 2 minutes (120000ms) - increase for long-running operations
- **Input precedence:** Command-line arguments take precedence over stdin

## Typical Workflow

1. Discover nREPL servers: `clj-nrepl-eval --discover-ports`
2. Prompt user to select a port
3. Require namespace:
   ```bash
   clj-nrepl-eval -p <PORT> "(require '[my.ns :as ns] :reload)"
   ```
4. Test function:
   ```bash
   clj-nrepl-eval -p <PORT> "(ns/my-fn ...)"
   ```
5. Iterate: Make changes, re-require with `:reload`, test again
