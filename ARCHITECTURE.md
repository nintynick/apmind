# Architecture

Detailed diagrams for the Autopoietic Mind system. See [README](README.md) for overview.

## Tournament selection

```
┌────────────────────────────────────────────────────────────────────┐
│  TOURNAMENT                                                        │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Lineage 1│  │ Lineage 2│  │ Lineage 3│  │ Lineage 4│          │
│  │ R1→R2→R3 │  │ R1→R2→R3 │  │ R1→R2→R3 │  │ R1→R2→R3 │          │
│  │ fit: .60  │  │ fit: .71  │  │ fit: .55  │  │ fit: .48  │          │
│  └─────┬────┘  └─────┬────┘  └──────────┘  └──────────┘          │
│        │         ┌────┘                                            │
│        │         ▼  WINNER                                         │
│        │    ┌──────────┐                                           │
│        │    │ Hall of   │── eval source ──► Seeds ALL lineages     │
│        │    │ Fame      │                   in next generation     │
│        │    └──────────┘                                           │
│        ▼         ▼                                                 │
│   Gen 1 ────► Gen 2 ────► Gen 3 ────► ...                         │
│                                                                    │
│  Fitness:                                                          │
│    0.45 solve rate + 0.15 novelty + 0.10 diversity                │
│  + 0.10 eval richness + 0.10 adversarial + 0.10 evolution        │
└────────────────────────────────────────────────────────────────────┘
```

## One lineage

```
┌────────────────────────────────────────────────────────────────────┐
│  LINEAGE RUN                                                       │
│                                                                    │
│  Round 1 (15 problems)       Round 2 (15 problems)     Round 3    │
│  Budget: 1500 steps          Budget: 1200 steps        Bgt: 1000  │
│  Native cap: unlimited       Native cap: 10            Cap: 5     │
│                                                                    │
│  ┌────┐┌────┐    ┌────┐    ┌────┐┌────┐    ┌────┐                │
│  │ P1 ││ P2 │ .. │P15 │    │ P1 ││ P2 │ .. │P15 │    ...        │
│  └──┬─┘└──┬─┘    └──┬─┘    └──┬─┘└──┬─┘    └──┬─┘                │
│     ▼     ▼         ▼        ▼     ▼         ▼                    │
│  ┌────────────────────────┐ ┌────────────────────────┐            │
│  │    REFLECTION PHASE    │ │    REFLECTION PHASE    │            │
│  │    Evolve my-eval for  │ │    Evolve my-eval for  │            │
│  │    next round          │ │    next round          │            │
│  └───────────┬────────────┘ └────────────────────────┘            │
│              │                                                     │
│   helpers + my-eval carry forward ───────────────►                │
│                                                                    │
│  Accumulates within a round:                                      │
│    - my-eval modifications (extend-eval!, edit-eval!)             │
│    - Persistent helper functions                                  │
│    - Novel construct discoveries                                  │
└────────────────────────────────────────────────────────────────────┘
```

## Problem-solving loop

```
┌────────────────────────────────────────────────────────────────────┐
│  PER-PROBLEM LOOP                                                  │
│                                                                    │
│  ┌──────────────────┐      ┌─────────────────────────────────┐    │
│  │  buildPrompt()   │      │  LLM (Claude)                   │    │
│  │                  │      │                                  │    │
│  │  - Problem desc  │─────►│  Sees: budget, my-eval source,  │    │
│  │  - Budget left   │      │  scratchpad, helpers, failures   │    │
│  │  - my-eval src   │◄─────│                                  │    │
│  │  - Scratchpad    │      │  Returns: 1-5 s-expressions     │    │
│  │  - Helpers       │      │  using metacognitive forms       │    │
│  └──────────────────┘      └─────────────────────────────────┘    │
│           │                                                        │
│           ▼                                                        │
│  ┌────────────────────────────────────────────┐                   │
│  │         METACOGNITIVE SCRATCHPAD           │                   │
│  │                                            │                   │
│  │  1 GOAL  "Define factorial"          +2    │                   │
│  │  2 CODE  (define (factorial n) ...)  +45   │                   │
│  │  3 TRY   (factorial 5)              +30    │                   │
│  │  4 CHECK '120 (factorial 5)         +31    │                   │
│  │  5 DONE  (factorial 5)              +30    │                   │
│  │     => 120 SOLVED                          │                   │
│  │                                            │                   │
│  │  Max 25 cells per problem                  │                   │
│  └────────────────────────────────────────────┘                   │
│                                                                    │
│  Loop until: (done ...) succeeds, budget exhausted, or error      │
└────────────────────────────────────────────────────────────────────┘
```

## Dual evaluator

```
┌────────────────────────────────────────────────────────────────────┐
│  TWO EVALUATORS                                                    │
│                                                                    │
│  ┌──────────────────────────┐  ┌──────────────────────────┐      │
│  │ Ground Eval (JavaScript) │  │ my-eval (Lisp)           │      │
│  │ Immutable, trusted       │  │ Self-modifying            │      │
│  │                          │  │                           │      │
│  │ Handles:                 │  │ Starts as 49-line seed    │      │
│  │  - All special forms     │  │ Agent evolves via:        │      │
│  │  - Metacognitive forms   │  │   extend-eval! (add)      │      │
│  │  - Step counting         │  │   edit-eval!   (replace)  │      │
│  │  - Error handling        │  │                           │      │
│  │                          │  │ Handles:                  │      │
│  │ Entry: evaluate()        │  │  - Standard Lisp forms    │      │
│  │                          │  │  - Agent-invented forms   │      │
│  │ Routes to my-eval via    │  │  - All problem code       │      │
│  │ evaluateViaMyEval()      │  │                           │      │
│  └────────────┬─────────────┘  │ Entry: (my-eval expr env) │      │
│               │                └─────────────┬────────────┘      │
│               │                               │                   │
│               │  Routing:                     │                   │
│               │  (goal ...) ──► ground only    │                   │
│               │  (try ...)  ──► ground+my-eval │                   │
│               │  (done ...) ──► ground+my-eval │                   │
│               │  (extend-eval!) ► ground only  │                   │
│               │  plain code ──► my-eval only   │                   │
│               └───────────────────────────────┘                   │
└────────────────────────────────────────────────────────────────────┘
```

## Self-modification flow

```
┌────────────────────────────────────────────────────────────────────┐
│  extend-eval! FLOW                                                 │
│                                                                    │
│                   ┌────────────────────┐                          │
│                   │  extend-eval!      │                          │
│                   │  (add one form)    │                          │
│                   └─────────┬──────────┘                          │
│                             ▼                                      │
│                                                                    │
│  my-eval source:       Find insertion point                       │
│  (define (my-eval ...) (before final else)                        │
│    (cond                     │                                    │
│      ((number?) expr)        ▼                                    │
│      ((symbol?) lookup)  Insert new clause:                       │
│      ...                 ((eq? head 'NEW) body)                   │
│      ((eq? head 'if) ..)     │                                    │
│      ((eq? head 'let) ..)    ▼                                    │
│   ┌► ((eq? head 'NEW) ..)  Sanity tests                          │
│   │  (else (my-apply ..))  42, (+ 1 2), (if #t 1 2),             │
│   │                        ((lambda (x) x) 5)                    │
│   │                          │                                    │
│   │                   pass ──┤                                    │
│   │                          ▼                                    │
│   │                   Commit new source                           │
│   └── NEW FORM ──────Detect novel construct                       │
│                       Grant +150 bonus steps                      │
│                                                                    │
│  Observed inventions:                                             │
│    match, ->, fold, foldr, pipe, collect, do, memo-lambda         │
└────────────────────────────────────────────────────────────────────┘
```

## Persistence

```
┌────────────────────────────────────────────────────────────────────┐
│  ACCUMULATION ACROSS PROBLEMS                                      │
│                                                                    │
│  Problem 1       Problem 2        Problem 3        Problem 4      │
│  ┌──────┐        ┌──────┐         ┌──────┐         ┌──────┐      │
│  │Define │        │Reuse │         │Reuse │         │Reuse │      │
│  │helper1│──save─►│helper1│──save─►│helper1│──save─►│helper1│     │
│  │       │        │Define │        │helper2│        │helper2│     │
│  │       │        │helper2│──save─►│Define │──save─►│helper3│     │
│  └──────┘        └──────┘         │helper3│        │Define │     │
│                                    └──────┘         │helper4│     │
│                                                      └──────┘      │
│                                                                    │
│  After solved:  savePersistentDefs() scans env                    │
│  Before next:   replayPersistentDefs() reinstalls all             │
│  my-eval:       source string carried automatically               │
└────────────────────────────────────────────────────────────────────┘
```

## Evolutionary pressure

```
  Round │ Steps │ Native cap │ Effect
  ──────┼───────┼────────────┼──────────────────────────
    1   │ 1500  │  unlimited │ Learn basics, define helpers
    2   │ 1200  │     10     │ Build on helpers, start evolving
    3   │ 1000  │      5     │ Lean on evolved language
   4+   │  600  │      3     │ Pure language fitness

  Bonuses:
    +150 steps per successful extend-eval! / edit-eval!
    +75  steps per successful edit during reflection
```
