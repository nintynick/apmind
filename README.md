# Autopoietic Mind

An LLM agent that evolves its own programming language. It starts with a 49-line Lisp evaluator it can read and rewrite at runtime — adding new syntax, new control flow, whatever it needs to solve increasingly hard problems. Multiple lineages compete in a tournament; the best-evolved language seeds the next generation.

Single file, no build step. Open `index.html` in a browser, paste an Anthropic API key, and watch it go.

## What happens

The agent gets a problem like "define `(merge-sort lst)`" and a step budget. It thinks in s-expressions on a scratchpad — planning, trying, testing — then submits an answer. If it struggles, it can crack open its own evaluator and bolt on a new language construct: pattern matching, pipelines, memoization, whatever it invents. That construct persists for all future problems.

Across rounds the budget shrinks, forcing the agent to rely on the language features it built. Across generations, tournament selection keeps the lineage whose evolved language solves the most problems.

Constructs agents have invented: `match`, `->`, `fold`, `pipe`, `memo-lambda`, `collect`, `do`, `for-each`

## Quick start

1. Open `index.html` in any modern browser
2. Enter your Anthropic API key
3. Pick **Single Lineage** to watch one agent work, or **Tournament** to run competitive evolution
4. Hit Start

State can be exported/imported as JSON at any time.

## Architecture

### The loop

```
                    ┌──────────────┐
                    │   Problem    │  "Define (merge-sort lst)"
                    └──────┬───────┘
                           │
                           ▼
  ┌─────────────────────────────────────────────┐
  │               buildPrompt()                 │
  │                                             │
  │  problem + budget + my-eval source          │
  │  + scratchpad history + helpers             │
  └──────────────────┬──────────────────────────┘
                     │
                     ▼
  ┌─────────────────────────────────────────────┐
  │               LLM (Claude)                  │
  │                                             │
  │  Returns 1-5 s-expressions:                 │
  │  (goal "sort the list")                     │
  │  (define (merge-sort lst) ...)              │
  │  (try (merge-sort '(5 3 1)))                │
  │  (check '(1 3 5) (merge-sort '(5 3 1)))    │
  │  (done (merge-sort '(5 3 8 1 9 2)))        │
  └──────────────────┬──────────────────────────┘
                     │
                     ▼
  ┌─────────────────────────────────────────────┐
  │          Metacognitive Scratchpad           │
  │                                             │
  │  1 GOAL  "sort the list"              +2    │
  │  2 CODE  (define (merge-sort ...) )   +45   │
  │  3 TRY   (merge-sort '(5 3 1))       +30   │
  │  4 CHECK (1 3 5) = (1 3 5)           +31   │
  │  5 DONE  (merge-sort '(5 3 8 1 9 2)) +30   │
  │     => (1 2 3 5 8 9) SOLVED                 │
  │                                             │
  │  25 cells max per problem                   │
  └─────────────────────────────────────────────┘
                     │
              solved? ──► next problem
              stuck?  ──► loop (LLM sees failure notes)
```

### Dual evaluator

The key architectural idea: two evaluators, one the agent controls.

```
  ┌──────────────────────────┐    ┌──────────────────────────┐
  │  Ground Eval (JavaScript)│    │  my-eval (Lisp)          │
  │  Immutable, trusted      │    │  Agent-modifiable        │
  │                          │    │                          │
  │  - Special forms         │    │  Starts as 49-line seed  │
  │  - Metacognitive forms   │    │  Agent extends via:      │
  │  - Step counting         │    │    extend-eval!          │
  │  - Error boundaries      │    │    edit-eval!            │
  │                          │    │                          │
  │  evaluate()              │    │  (my-eval expr env)      │
  └────────────┬─────────────┘    └────────────┬─────────────┘
               │                                │
               │    Control forms (goal, try,    │
               │    check, done) run in ground.  │
               │    User code runs in my-eval.   │
               └────────────────────────────────┘
```

### Self-modification

When the agent calls `(extend-eval! 'match ...)`:

```
  my-eval source (cond chain):

    ((number? expr) expr)
    ((symbol? expr) lookup)
    ((eq? head 'quote) ...)
    ((eq? head 'if) ...)
    ((eq? head 'let) ...)
    ((eq? head 'match) ...)    ◄── inserted here
    (else (my-apply ...))

                 │
                 ▼
        ┌─────────────────┐
        │  4 sanity tests  │  42, (+ 1 2), (if #t 1 2),
        │  must all pass   │  ((lambda (x) x) 5)
        └────────┬────────┘
                 │ pass
                 ▼
        Commit new source
        Detect novel construct
        Grant +150 bonus steps
```

### Tournament selection

```
  Gen 1                              Gen 2
  ┌─────────┐ ┌─────────┐           ┌─────────┐ ┌─────────┐
  │Lineage A│ │Lineage B│           │Lineage A│ │Lineage B│
  │ .62     │ │ .71     │  winner   │ .68     │ │ .74     │  winner
  └─────────┘ └────┬────┘  ──────►  └─────────┘ └────┬────┘  ──────► ...
  ┌─────────┐      │ seed   all     ┌─────────┐      │ seed
  │Lineage C│      ▼        get     │Lineage C│      ▼
  │ .55     │ ┌─────────┐  same     │ .59     │ ┌─────────┐
  └─────────┘ │Hall of  │  eval     └─────────┘ │Hall of  │
  ┌─────────┐ │Fame     │  source   ┌─────────┐ │Fame     │
  │Lineage D│ └─────────┘          │Lineage D│ └─────────┘
  │ .48     │                       │ .63     │
  └─────────┘                       └─────────┘

  Each lineage gets a different personality bias:
    recursion, higher-order, pattern-matching,
    compactness, state/accumulation, lazy eval,
    declarative, algebraic structure
```

Each lineage runs 3 rounds of 15 problems. Budgets tighten each round:

| Round | Steps | Native cap | Pressure |
|-------|-------|------------|----------|
| 1 | 1500 | unlimited | Learn the basics |
| 2 | 1200 | 10 | Build helpers |
| 3 | 1000 | 5 | Lean on evolved language |
| 4+ | 600 | 3 | Pure language fitness |

### Fitness

```
0.45  solve rate           How many problems solved
0.15  novelty              Unique constructs invented
0.10  diversity            Breadth across 9 problem categories
0.10  eval richness        Size/complexity of evolved evaluator
0.10  adversarial bonus    Solve rate on LLM-generated problems
0.10  evolution bonus      Any modification to my-eval at all
```

### Problem pool

103 hardcoded problems across 9 categories, plus LLM-generated adversarial problems each generation:

| Category | IDs | Examples |
|----------|-----|---------|
| Standard | 1-15 | factorial, reverse, flatten, group-by |
| Hard | 101-120 | tree-map, topo-sort, CPS, matrix-mul |
| Evolution | 201-208 | symbolic deriv, free-vars, rewrite rules |
| String | 300-309 | palindrome, caesar cipher, chunk |
| Higher-order | 400-409 | curry, pipe, complement, zip-with |
| Number theory | 500-509 | GCD, prime factors, modpow, totient |
| Recursive | 600-609 | binary search, merge-sort, n-queens |
| Logic | 700-709 | truth tables, De Morgan, tautology check |
| Meta | 800-809 | Y combinator, Church numerals, make-stack |
| **Adversarial** | 900+ | **LLM-generated each generation, targeting gaps** |

## Tech

- Single `index.html` (~2800 lines)
- React 18 via CDN, no JSX (`createElement` throughout)
- Anthropic Messages API (endpoint and model configurable)
- JSON export/import for full state persistence
- Dark theme, monospace everything

## License

MIT
