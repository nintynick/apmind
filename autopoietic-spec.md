# Autopoietic Mind

## Core Thesis

An AI agent reasons on a Lisp scratchpad. Its programs are evaluated by a Lisp-level interpreter (`my-eval`) that the agent can read, understand, and modify. Evolution happens when the agent changes its own evaluator under computational pressure. The language the agent thinks in is whatever `my-eval` currently implements.

---

## Architecture

Three layers, bottom to top:

```
┌─────────────────────────────────────┐
│  AGENT (LLM)                        │
│  Emits s-expressions on scratchpad   │
│  Can read and modify my-eval         │
│  Constrained by cell + step budgets  │
├─────────────────────────────────────┤
│  my-eval (LISP)                      │
│  ~40 lines of Lisp                   │
│  Agent's programs run through this   │
│  Agent can modify this at any time   │
│  THIS IS WHAT EVOLVES               │
├─────────────────────────────────────┤
│  GROUND EVAL (JavaScript)            │
│  Fixed, never changes                │
│  Runs my-eval itself                 │
│  Provides primitives (+, cons, etc)  │
│  Counts eval steps                   │
└─────────────────────────────────────┘
```

**Ground eval** is trusted infrastructure. It implements the Lisp core in JavaScript: parsing, environments, closures, primitive operations. It never changes. It's what makes `my-eval` run.

**my-eval** is a Lisp function living in the agent's environment. The agent's scratchpad expressions are evaluated by calling `(my-eval expr env)`. Because my-eval is Lisp, the agent can `(display-eval)` to see its source, reason about it, and `(set! my-eval ...)` to change it. This is the only thing that evolves.

**The agent** is an LLM that emits only s-expressions. It sees the scratchpad (previous expressions + results), the current my-eval source, and its remaining budgets. It decides what to compute and when to modify its evaluator.

---

## The Seed: Initial my-eval

The agent starts with this evaluator (approximately):

```scheme
(define (my-eval expr env)
  (cond
    ((number? expr) expr)
    ((boolean? expr) expr)
    ((string? expr) expr)
    ((symbol? expr) (env-lookup expr env))
    ((not (pair? expr)) (error "unknown form"))
    (else
      (let ((head (car expr)))
        (cond
          ((eq? head 'quote)  (cadr expr))
          ((eq? head 'if)     (if (my-eval (cadr expr) env)
                                  (my-eval (caddr expr) env)
                                  (my-eval (cadddr expr) env)))
          ((eq? head 'lambda) (make-closure (cadr expr) (cddr expr) env))
          ((eq? head 'define) (env-define! env (cadr expr)
                                (my-eval (caddr expr) env)))
          ((eq? head 'let)    (my-eval-let (cadr expr) (cddr expr) env))
          ((eq? head 'begin)  (my-eval-seq (cdr expr) env))
          (else
            (my-apply (my-eval head env)
                      (map (lambda (a) (my-eval a env))
                           (cdr expr)))))))))
```

This handles: self-evaluating atoms, symbol lookup, quote, if, lambda, define, let, begin, and function application. That's enough to compute with. Everything else — cond, and, or, let*, pattern matching, lazy eval, memoization, new special forms — the agent must add by modifying my-eval.

The helpers `env-lookup`, `env-define!`, `make-closure`, `my-apply`, `my-eval-let`, `my-eval-seq` are also Lisp functions the agent can inspect and modify.

---

## Budgets: The Pressure

Two hard limits per problem:

| Resource | Budget | What it constrains |
|----------|--------|--------------------|
| **Cells** | 25 | Number of top-level s-expressions the agent can emit |
| **Steps** | 5,000 | Total ground-eval steps across all cells |

**Cells** constrain how much the agent can write. Each s-expression on the scratchpad costs 1 cell. A macro-like compression (one expression that does what three used to) saves cells.

**Steps** constrain how much the agent can compute. Every ground-eval call to `evaluate()` increments the counter. When `my-eval` interprets an expression, each recursive call to `my-eval` costs steps (because ground-eval is running my-eval). An inefficient my-eval burns steps fast. An optimized my-eval — one that short-circuits, memoizes, or fuses operations — lets the agent solve harder problems.

**Why this creates real autopoietic pressure:** If the agent's my-eval is the naive 40-line seed, solving a pipeline problem like "filter odds, square each, sum" might cost 800 steps — three list traversals. If the agent adds a `pipeline` special form to my-eval that fuses traversals, the same computation costs 300 steps. That 500-step savings is real: it's the difference between fitting a harder problem within budget or not. The agent has mechanical incentive to improve its evaluator.

**What the agent sees each turn:**

```
Scratchpad (8/25 cells, 2847/5000 steps):
[1] GOAL (goal "partition list")          → "partition list"
[2] SUB  (sub "test" (my-eval ...))       → (2 4 6 8)
...
[8] CODE (define ...)                      → <λ>

2153 steps remaining. Emit 1-5 s-expressions:
```

---

## Metacognitive Primitives

Evaluated by ground-eval (not by my-eval):

### Reasoning

| Form | Semantics | Purpose |
|------|-----------|---------|
| `(goal "text")` | Returns the string | Declare intent |
| `(sub "label" expr)` | Evaluates expr via my-eval, returns result | Labeled reasoning step |
| `(try expr)` | Evaluates expr via my-eval, catches errors → `(#t result)` or `(#f "msg")` | Safe exploration |
| `(note "text")` | Returns the string | Record observation |
| `(check expected actual)` | Compares, returns `#t` or `#f` | Verify correctness |
| `(done expr)` | Signals solved, returns expr | Terminate problem |

### Reflection (reify/reflect)

| Form | Semantics | Purpose |
|------|-----------|---------|
| `(show-eval)` | Returns the source of my-eval as a quoted list | Reify: see the evaluator as data |
| `(edit-eval! new-eval)` | Replaces my-eval, runs safety check | Reflect: modify evaluator and resume |
| `(show-env)` | Returns current bindings as `((name value) ...)` | Reify: see the live environment |
| `(eval-in expr env-alist)` | Evaluates expr through my-eval in a constructed env | Sandboxed evaluation in custom env |
| `(with-limit n expr)` | Evaluates expr with n-step sub-budget → `(#t result)` or `(#f "step limit")` | Bounded exploration |

**`show-eval`** is reification of the evaluation rules: the agent sees its own interpreter as data.

**`show-env`** is reification of the live state: the agent sees what's currently bound, mid-computation. This enables inspection-driven modification — the agent can ask "what's in scope?" and make decisions based on the actual computational state, not just the source code. Returned as an association list the agent can manipulate, filter, or pass to `eval-in`.

**`eval-in`** is sandboxed evaluation: the agent constructs a custom environment (possibly by modifying the output of `show-env`) and evaluates an expression in it. This is how the agent tests modifications safely — try a change in a controlled context before committing with `edit-eval!`. It also enables the agent to construct specialized evaluation contexts: "evaluate this expression as if x were bound to a lazy thunk instead of a value."

**`with-limit`** is bounded exploration: the agent can try an expensive computation with a step cap. If it exceeds the cap, it gets back a failure instead of blowing its entire budget. This gives partial control over the future of a computation — not full continuations, but enough to prevent catastrophic budget waste while exploring.

**`edit-eval!`** is reflection: the agent modifies its interpreter and resumes. All subsequent computation runs through the new evaluator.

**Safety check on edit-eval!:** After replacement, ground-eval runs a small sanity suite through the new my-eval:
- `(my-eval 42 empty-env)` → 42
- `(my-eval '(+ 1 2) base-env)` → 3
- `(my-eval '(if #t 1 2) base-env)` → 1
- `(my-eval '((lambda (x) x) 5) base-env)` → 5

If any fails, my-eval reverts to the previous version and the agent sees `(#f "eval edit rejected: [reason]")`. This is the minimal Gödel Machine criterion — not "provably better" but "provably not broken."

### The Tower Property

my-eval is permitted to call my-eval. This creates interpretive levels on demand, charged against the step budget. The agent can define a meta-evaluator that wraps my-eval with tracing, memoization, or modified rules, then route specific computations through it:

```scheme
(define (tracing-eval expr env)
  (note (list 'evaluating expr))
  (my-eval expr env))

(define (memo-eval expr env)
  (let ((cached (lookup-cache expr)))
    (if cached cached
        (let ((result (my-eval expr env)))
          (store-cache! expr result)
          result))))
```

The tower emerges from use, not from architecture. There is no fixed number of levels. If the agent defines a `meta-my-eval` that interprets `my-eval`, that's a third level. If `meta-my-eval` calls itself, that's a fourth. Each level costs step budget proportional to the interpretation overhead. This creates natural pressure against unnecessary meta-levels — every tower level is a multiplicative cost — while allowing them when the benefit (tracing, optimization, sandboxing) justifies it.

---

## What the Agent Can Modify

Because my-eval and its helpers are Lisp, the agent can:

**Add special forms.** Insert a new `cond` branch in my-eval:
```scheme
((eq? head 'cond) (my-eval-cond (cdr expr) env))
```
Then define `my-eval-cond`. Now the language has `cond`.

**Add evaluation strategies.** Wrap the function application branch with memoization:
```scheme
(else (let ((cached (memo-lookup expr)))
        (if cached (cdr cached)
            (let ((result (my-apply ...)))
              (memo-store! expr result)
              result))))
```

**Modify env-lookup.** Add dynamic scoping for certain symbols. Add default values. Add logging.

**Modify my-apply.** Add tracing. Add currying (if too few args, return a partial closure). Add type checking.

**Fuse operations.** Add a `pipeline` form that traverses a list once applying filter+map+reduce in a single pass instead of three.

**Add compile-time optimization.** Modify my-eval to recognize `(+ 0 x)` → `x` or `(if #t a b)` → `a` before evaluating.

Each of these is a concrete modification to a Lisp function the agent can see — a change the agent makes to its own thinking process while thinking.

---

## Problem Bank

A diverse set of 15 problems, chosen randomly without replacement. The agent is not told what to modify — it discovers the need through hitting budget limits or failing.

Problems range from simple (factorial, list sum) to medium (insert-sorted, partition, sliding windows) to complex (tree traversal, run-length encoding, group-by, unfold). Harder problems naturally require more eval steps, creating pressure to optimize my-eval.

---

## Agent Prompt

The LLM receives:

```
You are a mind that thinks in Lisp. Output ONLY s-expressions.

Your programs run through my-eval — a Lisp interpreter you can see and modify.
Use (show-eval) to see it. Use (edit-eval! new-eval) to change it.

Metacognitive forms: goal, sub, try, note, check, done, show-eval, edit-eval!, show-env, eval-in, with-limit

Budget: {cells_remaining}/{CELL_BUDGET} cells, {steps_remaining}/{STEP_BUDGET} steps

[current my-eval source, if recently modified or if agent asked]

Scratchpad:
[1] ...
[2] ...

Problem: {problem text}

Emit 1-5 s-expressions. No English. No markdown.
```

---

## What We Track (UI)

**Scratchpad panel** — numbered cells with type badges (GOAL, SUB, TRY, NOTE, CHECK, DONE, EVAL, CODE, ERR). Color-coded by type.

**Budget bar** — cells and steps, both as progress bars depleting left to right.

**Eval history panel** — shows a timeline of my-eval modifications:
```
Problem 1-4:  seed eval (40 lines, ~12 steps/expr avg)
Problem 5:    +cond, +let* (48 lines, ~11 steps/expr avg)
Problem 8:    +memoized-lookup (52 lines, ~8 steps/expr avg)  
Problem 12:   +pipeline fusion (61 lines, ~6 steps/expr avg)
```

This is the autopoietic trajectory. If the agent is genuinely self-improving, the steps/expr metric drops over time as my-eval gets smarter.

**Eval source panel** — shows the current my-eval source, highlighted where the agent has modified it. Collapsible.

**Stats bar** — problems solved, problems failed (budget exceeded), eval edits attempted, eval edits accepted, current eval size.

---

## The Autopoietic Loop

There is no external evolution phase. The loop is:

1. Agent gets problem
2. Agent reasons on scratchpad through my-eval  
3. If agent hits step limit → problem fails, agent sees the failure
4. Agent can choose to `(show-eval)` and `(edit-eval! ...)` at any time
5. Modified eval applies to all subsequent computation
6. Next problem

The agent learns to modify its evaluator the same way it learns anything — by observing what works and what doesn't on the scratchpad. If it keeps hitting step limits on pipeline-heavy problems, it has incentive to add pipeline fusion. If it keeps writing verbose cond chains, it has incentive to add cond as a special form. The pressure is the budget. The mechanism is edit-eval!. The selection criterion is: does the modification let me solve problems I couldn't before?

That's autopoiesis. The system produces its own cognitive components (eval modifications) through its own operation (reasoning on the scratchpad), maintaining its organization (ground-eval is fixed, safety checks hold) while changing its structure (my-eval evolves).

---

## Success Criteria

**Minimum viable:** Agent modifies my-eval at least once across 15 problems, and the modification persists (passes safety check) and is used in subsequent problems.

**Interesting:** Agent adds 3+ modifications that reduce avg steps/expr, demonstrating genuine evaluator optimization.

**Remarkable:** Agent's my-eval diverges significantly from the seed — new special forms, new evaluation strategies, structural changes to how application or lookup works. The language the agent thinks in is no longer recognizable as the seed Lisp.

**The dream:** Agent modifies my-eval to be better at modifying my-eval. A self-improvement to the self-improvement process. Recursive autopoiesis. We'll know it when we see it.

---

## Implementation Estimate

~550-650 lines total:
- Lisp parser/tokenizer: ~60 lines
- Ground eval (JS): ~180 lines (with first-class environments and step counting)
- Metacircular eval seed (Lisp): ~40 lines
- Metacognitive primitives + safety checks: ~80 lines
- Agent loop + LLM interface: ~100 lines
- React UI: ~200 lines (scratchpad + budget bars + eval source panel)

One file. One artifact. No external dependencies beyond the Anthropic API.
