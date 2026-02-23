# Autopoietic Mind — Architecture

A self-modifying Lisp system where an LLM agent evolves its own language
by solving problems through a metacircular evaluator it can rewrite at runtime.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TOURNAMENT SELECTION                                │
│                                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│  │Lineage 1│  │Lineage 2│  │Lineage 3│  │Lineage 4│   M lineages/gen      │
│  │ R1→R2→R3│  │ R1→R2→R3│  │ R1→R2→R3│  │ R1→R2→R3│   N rounds each      │
│  │fitness:.6│  │fitness:.7│  │fitness:.5│  │fitness:.4│                      │
│  └────┬─────┘  └────┬─────┘  └─────────┘  └─────────┘                      │
│       │         ┌────┘                                                      │
│       │         ▼ WINNER (wholesale)                                        │
│       │    ┌─────────┐                                                      │
│       │    │ Hall of  │──── eval source ────► Seeds ALL lineages            │
│       │    │  Fame    │                       in next generation            │
│       │    └─────────┘                                                      │
│       │         │                                                           │
│       ▼         ▼                                                           │
│   Gen 1 ──► Gen 2 ──► Gen 3 ──► ...                                       │
│                                                                             │
│   Fitness = 0.50 × solve_rate                                              │
│           + 0.15 × novelty (unique constructs / 10)                        │
│           + 0.15 × efficiency (steps saved)                                │
│           + 0.10 × diversity (categories solved / 9)                       │
│           + 0.10 × eval_richness (eval lines / 120)                       │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                          ONE LINEAGE (Agent Run)                            │
│                                                                             │
│  Round 1 (15 problems)          Round 2 (15 problems)         Round 3      │
│  Budget: 1500 steps             Budget: 1200 steps            Budget: 1000 │
│  Native cap: ∞                  Native cap: 10                Native cap: 5│
│  ┌─────┐┌─────┐   ┌──────┐    ┌─────┐┌─────┐   ┌──────┐                  │
│  │Prob1││Prob2│...│Prob15│    │Prob1││Prob2│...│Prob15│    ...             │
│  └──┬──┘└──┬──┘   └──┬───┘    └──┬──┘└──┬──┘   └──┬───┘                  │
│     │      │         │           │      │         │                        │
│     ▼      ▼         ▼           ▼      ▼         ▼                        │
│  ┌──────────────────────┐     ┌──────────────────────┐                     │
│  │  REFLECTION PHASE    │     │  REFLECTION PHASE    │                     │
│  │  "Evolve my-eval for │     │  "Evolve my-eval for │                     │
│  │   next round"        │     │   next round"        │                     │
│  └──────────┬───────────┘     └──────────────────────┘                     │
│             │                                                               │
│    helpers + my-eval carry forward ──────────────►                          │
│                                                                             │
│  Accumulates across problems within a round:                               │
│    • my-eval modifications (extend-eval!, edit-eval!)                      │
│    • Persistent helper functions (defnative!, define)                      │
│    • Novel construct discoveries                                           │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                     PROBLEM-SOLVING LOOP (per problem)                      │
│                                                                             │
│  ┌────────────────────┐     ┌──────────────────────────────────────┐       │
│  │   buildPrompt()    │     │  LLM (Claude)                       │       │
│  │                    │     │                                      │       │
│  │ • Problem desc     │────►│  Sees: budget, my-eval source,      │       │
│  │ • Budget remaining │     │  scratchpad history, helpers,        │       │
│  │ • my-eval source   │◄────│  failure notes                      │       │
│  │ • Scratchpad cells │     │                                      │       │
│  │ • Persistent defs  │     │  Returns: up to 5 s-expressions     │       │
│  │ • Failure notes    │     │  using metacognitive forms           │       │
│  └────────────────────┘     └──────────────────────────────────────┘       │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────┐                           │
│  │          METACOGNITIVE SCRATCHPAD            │                           │
│  │                                              │                           │
│  │  1 GOAL  "Define factorial"            +2    │                           │
│  │  2 CODE  (define (factorial n) ...)    +45   │                           │
│  │  3 TRY   (factorial 5)                +30    │  Each cell: type,        │
│  │  4 CHECK '120 (factorial 5)           +31    │  expression, result,     │
│  │  5 DONE  (factorial 5)                +30    │  step cost               │
│  │     => 120 ✓ SOLVED                          │                           │
│  │                                              │                           │
│  │  Max 25 cells/problem                        │                           │
│  └──────────────────────────────────────────────┘                           │
│                                                                             │
│  Loop: buildPrompt → LLM → extract exprs → eval each → update scratchpad  │
│  Until: (done ...) succeeds, budget exhausted, or error                    │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                        DUAL EVALUATOR ARCHITECTURE                          │
│                                                                             │
│  ┌─────────────────────────────┐  ┌──────────────────────────────────┐     │
│  │    GROUND EVAL (JavaScript) │  │   MY-EVAL (Lisp, self-modifying) │     │
│  │    Immutable, trusted       │  │   Starts as 49-line seed         │     │
│  │                             │  │   Agent evolves via:             │     │
│  │  Handles:                   │  │    • extend-eval! (add form)     │     │
│  │   • All special forms       │  │    • edit-eval!   (replace all)  │     │
│  │   • Metacognitive forms     │  │                                  │     │
│  │   • Step counting           │  │  Handles:                        │     │
│  │   • Error handling          │  │   • All standard Lisp forms      │     │
│  │                             │  │   • Agent-invented forms         │     │
│  │  entry point: evaluate()    │  │   • Problem code execution       │     │
│  │                             │  │                                  │     │
│  │  Routes to my-eval via:     │  │  entry point: (my-eval expr env) │     │
│  │   evaluateViaMyEval()       │  │                                  │     │
│  └──────────┬──────────────────┘  └──────────────┬───────────────────┘     │
│             │                                     │                         │
│             │  Metacognitive forms use             │                         │
│             │  ground eval for control,            │                         │
│             │  my-eval for content:                │                         │
│             │                                     │                         │
│             │  (goal "text")     → ground eval     │                         │
│             │  (sub "label" body)→ ground + my-eval│                         │
│             │  (try expr)        → ground + my-eval│                         │
│             │  (check exp actual)→ ground + my-eval│                         │
│             │  (done expr)       → ground + my-eval│                         │
│             │  (extend-eval! ..) → ground eval     │                         │
│             │  (defnative! ..)   → ground eval     │                         │
│             │  plain code        → my-eval only    │                         │
│             └─────────────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                         SELF-MODIFICATION FLOW                              │
│                                                                             │
│                     ┌──────────────────────┐                               │
│                     │   extend-eval!       │                               │
│                     │   (add one form)     │                               │
│                     └──────────┬───────────┘                               │
│                                │                                            │
│                                ▼                                            │
│  my-eval source:          Find insertion point                             │
│  (define (my-eval ...)    (before final else clause)                       │
│    (cond                        │                                          │
│      ((number?) expr)           ▼                                          │
│      ((symbol?) lookup)   Insert: ((eq? head 'new-form) body)             │
│      ...                        │                                          │
│      ((eq? head 'quote) ..)     ▼                                          │
│      ((eq? head 'if) ..)  ┌──────────────┐                                │
│      ((eq? head 'let) ..) │  Sanity tests │  42, (+ 1 2),                 │
│   ┌► ((eq? head 'NEW) ..) │  (4 basic)    │  (if #t 1 2),                 │
│   │  (else (my-apply ..)) └──────┬───────┘  ((lambda (x) x) 5)           │
│   │                              │                                         │
│   │                    pass ─────┤                                         │
│   │                              ▼                                         │
│   │                     ┌──────────────────┐                               │
│   │                     │ Commit new source │                              │
│   └─── NEW FORM ADDED ─│ Detect novel form │                              │
│                         │ Grant +150 steps  │                              │
│                         └──────────────────┘                               │
│                                                                             │
│  Examples of novel constructs evolved in practice:                          │
│    collect, ->, fold, foldr, memo-lambda, pipe, match, do, for-each       │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                         PERSISTENCE & ACCUMULATION                          │
│                                                                             │
│  Problem 1        Problem 2         Problem 3         Problem 4            │
│  ┌──────┐         ┌──────┐          ┌──────┐          ┌──────┐            │
│  │Define │         │Reuse │          │Reuse │          │Reuse │            │
│  │helper1│────────►│helper1│────────►│helper1│────────►│helper1│           │
│  │       │  save   │Define │  save   │helper2│  save   │helper2│           │
│  │       │         │helper2│────────►│Define │────────►│helper3│           │
│  └──────┘         └──────┘          │helper3│         │Define │           │
│                                     └──────┘          │helper4│           │
│                                                        └──────┘            │
│                                                                             │
│  After each solved problem:                                                │
│    savePersistentDefs() → scans env for user-defined symbols               │
│                                                                             │
│  Before each new problem:                                                  │
│    replayPersistentDefs() → reinstalls all saved helpers                   │
│                                                                             │
│  my-eval modifications persist automatically (source string carried)       │
│  Native functions tracked separately (params + body stored for replay)     │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                          PROBLEM POOL (103 problems)                        │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│  │  STANDARD    │  │    HARD      │  │  EVOLUTION   │                     │
│  │  (IDs 1-15)  │  │ (IDs 101-120)│  │ (IDs 201-208)│                     │
│  │  factorial,  │  │  tree-map,   │  │  deriv,      │                     │
│  │  reverse,    │  │  topo-sort,  │  │  simplify,   │                     │
│  │  flatten,    │  │  CPS, church,│  │  free-vars,  │                     │
│  │  group-by    │  │  matrix-mul  │  │  rewrite     │                     │
│  └──────────────┘  └──────────────┘  └──────────────┘                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│  │   STRING     │  │ HIGHER-ORDER │  │ NUMBER THEORY│                     │
│  │ (IDs 300-309)│  │ (IDs 400-409)│  │ (IDs 500-509)│                     │
│  │  palindrome, │  │  curry,      │  │  GCD, prime?,│                     │
│  │  caesar,     │  │  pipe,       │  │  collatz,    │                     │
│  │  chunk,      │  │  complement, │  │  modpow,     │                     │
│  │  intersperse │  │  zip-with    │  │  totient     │                     │
│  └──────────────┘  └──────────────┘  └──────────────┘                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│  │  RECURSIVE   │  │    LOGIC     │  │     META     │                     │
│  │ (IDs 600-609)│  │ (IDs 700-709)│  │ (IDs 800-809)│                     │
│  │  bin-search, │  │  truth-table,│  │  accumulator,│                     │
│  │  merge-sort, │  │  de-morgan,  │  │  Y combinator│                     │
│  │  BST ops,    │  │  tautology?, │  │  church nums,│                     │
│  │  n-queens    │  │  eval-bool   │  │  make-stack  │                     │
│  └──────────────┘  └──────────────┘  └──────────────┘                     │
│                                                                             │
│  Tournament mode: 15 problems/round, ≥1 from each category                │
│  Single mode:     15 problems/round, weighted by round difficulty          │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                          EVOLUTIONARY PRESSURE                              │
│                                                                             │
│  Scarcity increases over rounds:                                           │
│                                                                             │
│     Round │ Step Budget │ Native Cap │ Problems From                       │
│    ───────┼─────────────┼────────────┼──────────────                       │
│       1   │    1500     │     ∞      │ Standard only                       │
│       2   │    1200     │     10     │ Standard + Hard + New pools         │
│       3   │    1000     │      5     │ Hard + Evolution + New pools        │
│      4+   │     600     │      3     │ Mostly Evolution + New pools        │
│                                                                             │
│  This creates pressure to:                                                 │
│   • Build reusable helpers early (persist across problems)                 │
│   • Evolve my-eval for efficiency (fewer steps = higher fitness)           │
│   • Invent novel constructs (extends language expressiveness)              │
│   • Prefer extend-eval! over defnative! (native slots scarce)             │
│                                                                             │
│  Bonuses reward self-modification:                                         │
│   • +150 steps per successful extend-eval! / edit-eval!                    │
│   • +75 steps per successful edit during reflection                        │
│   • Novel construct detection → tracked in stats                           │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                              TECH STACK                                     │
│                                                                             │
│  Single file: index.html (~2800 lines)                                     │
│                                                                             │
│  • React 18 (CDN, no JSX — createElement throughout)                       │
│  • No build step — runs directly in browser                                │
│  • LLM API: Anthropic Messages API (configurable endpoint/model)           │
│  • State: React useState/useRef, no external state management              │
│  • Persistence: JSON export/import (version 4 format)                      │
│  • CSS: Custom dark theme with CSS variables                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

## How It All Fits Together

The system is an **autopoietic loop** — a mind that makes itself:

1. **The LLM agent** receives a Lisp problem and its own evaluator source code
2. It solves the problem using **metacognitive forms** (goal → try → check → done)
3. Problem code runs through **my-eval** (the Lisp metacircular evaluator)
4. When the agent struggles, it can **modify my-eval** to add new language features
5. Successful modifications grant **bonus steps** and persist to future problems
6. After each round, a **reflection phase** encourages strategic language evolution
7. In **tournament mode**, multiple independent lineages compete — the best
   eval source becomes the seed for the next generation
8. Over generations, the language evolves toward greater expressiveness and
   efficiency, driven by problem-solving pressure across 103 diverse problems
