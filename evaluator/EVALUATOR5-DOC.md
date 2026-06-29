# Evaluator 5 - The CEK Machine, Complete

*Continues from `EVALUATOR4-DOC`: the CEK machine on pure lambda calculus.
Continues in `EVALUATOR6-DOC`: compiling to bytecode.*

`EVALUATOR4-DOC` built the whole machine -- the `C`/`E`/`K` (and `V`) registers,
the two-state EVAL/APPLY loop, continuation frames as tagged tuples -- on the
smallest language that could show it off: pure lambda calculus plus `if`.  This
chapter puts the full `IttyBittyLisp3` language back: `#t`/`#f`, `quote`, `set!`,
`begin`, `let`, multi-argument functions, and primitives.

The point of the chapter is what *doesn't* change.  The two loops, the explicit
`K` stack, the frames-as-data idea -- all identical.  Growing the language does
not change the machine's *shape*; it only adds **more kinds of continuation
frame**.  That is the real lesson, and it is how production interpreters grow:
you don't redesign the engine for each feature, you add a frame.

## Three frames became four

The minimal machine had `FRAME_IF`, `FRAME_ARG`, and `FRAME_CALL`.  The full
language has four:

```python
FRAME_IF  = 0   # waiting on a test value          -> pick a branch
FRAME_SET = 1   # waiting on a value to assign      -> set! it
FRAME_SEQ = 2   # a begin / body, forms still to run
FRAME_ARG = 3   # an application accumulating operator + operands
```

- **`FRAME_IF`** is unchanged in shape, but truthiness changes: now that we have
  real `#t`/`#f`, `if` uses Scheme's rule -- **`#f` is the only false value** --
  so `0`, `''`, and the empty list are all true.  (In #4, with no booleans,
  `0` was the false value; that was a stand-in, and it goes away here.)
- **`FRAME_ARG`** absorbs the old `FRAME_CALL`.  In #4 a call had exactly one
  argument, so two small frames sufficed.  Real calls take any number of
  arguments, so `FRAME_ARG` now carries an **accumulator**: `(done, todo, env)`,
  where `done` is the values gathered so far (operator first) and `todo` is the
  expressions still to evaluate.
- **`FRAME_SET`** and **`FRAME_SEQ`** are genuinely new, for `set!` and for
  sequencing (`begin` and multi-form bodies).

And one form needs *no* frame at all.

## `let` is just sugar

A `let` evaluates its initialisers in the outer scope, then runs its body with
those values bound -- which is exactly what applying a `lambda` does.  So `let`
desugars to a lambda application, right in the EVAL loop, and the existing
application machinery handles it:

```python
elif C[0] == 'let':            # ['let', ((name init)...), *body]
    names = [ pair[0] for pair in C[1] ]
    inits = [ pair[1] for pair in C[1] ]
    C = [ ['lambda', names] + list(C[2:]) ] + inits   # ((lambda (names) body...) inits...)
```

`(let ((a 3) (b 4)) (+ a b))` becomes `((lambda (a b) (+ a b)) 3 4)` and is never
seen as `let` again.  No `FRAME_LET`, no special apply logic -- the cheapest way
to add a feature is to express it in terms of one you already have.

## The new trick: primitives make APPLY iterate

`EVALUATOR4-DOC` ended on a foreshadowing: a primitive *produces a value*, while
a closure *installs a body*.  That difference is where the two-loop shape earns
its keep.

When `FRAME_ARG` has gathered the operator and all operands, it applies them:

```python
fn, args = done[0], done[1:]
if callable( fn ):             # a primitive: compute the value and flow it on
    V = fn( args )
    continue                   # STAY in APPLY -- V goes to the next frame
_, params, body, clo_env = fn  # a closure: bind params, evaluate the body
E = Environment( parent=clo_env, bindings=dict( zip(params, args) ) )
if len(body) > 1:
    K.append( (FRAME_SEQ, body[1:], E) )
C = body[0]
break                          # drop to EVAL to run the body
```

A closure `break`s back to EVAL (there is new code to run).  A primitive does
**not** -- it has a finished value, so it `continue`s the APPLY loop, handing
`V` straight to the next waiting frame.  In #4's pure-λ language nothing
produced a value mid-APPLY, so the APPLY loop always ran exactly once per step.
Add `+` and the APPLY loop genuinely *loops*.  That symmetry -- EVAL descends,
APPLY climbs -- is why the machine was written as two loops in the first place.

## Step-by-Step: Watching the Machine Run

Each table shows the registers at the start of a loop iteration.

### A primitive call: `(+ 2 3)`

Watch `FRAME_ARG` accumulate the operator and operands, then the primitive
deliver its value *inside* APPLY:

| step | state | C | V | K |
|---|---|---|---|---|
| 1 | EVAL  | `[+ 2 3]` | -   | `[]` |
| 2 | EVAL  | `+`       | -   | `[ARG done=[] todo=[2,3]]` |
| 3 | APPLY | -         | `+` | `[ARG done=[+] todo=[3]]` |
| 4 | EVAL  | `2`       | -   | `[ARG done=[+] todo=[3]]` |
| 5 | APPLY | -         | `2` | `[ARG done=[+,2] todo=[]]` |
| 6 | EVAL  | `3`       | -   | `[ARG done=[+,2] todo=[]]` |
| 7 | APPLY | -         | `3` | `[]`  -> `+([2,3])` = `5`, stay in APPLY |
| 8 | APPLY | -         | `5` | `[]`  -> **return 5** |

A single `FRAME_ARG` carries the work: each time a value arrives it is appended
to `done`, and the frame either evaluates the next operand (steps 3, 5) or --
when `todo` is empty -- applies.  Steps 7-8 are the behaviour #4 could not show:
`+` produces a value, so the machine `continue`s in APPLY and hands `5` straight
on, never dropping back to EVAL.

### Nested calls: `(+ (* 2 3) 1)` -- K grows to depth 2

The outer `+` cannot finish until `(* 2 3)` resolves, so a second `FRAME_ARG`
stacks on top of the first:

| step | state | C | V | K |
|---|---|---|---|---|
| 1 | EVAL  | `[+ [* 2 3] 1]` | -   | `[]` |
| 2 | EVAL  | `+`             | -   | `[ARG done=[] todo=[[* 2 3],1]]` |
| 3 | APPLY | -               | `+` | `[ARG done=[+] todo=[1]]` |
| 4 | EVAL  | `[* 2 3]`       | -   | `[ARG done=[+] todo=[1]]` |
| 5 | EVAL  | `*`             | -   | `[ARG done=[+] todo=[1], ARG done=[] todo=[2,3]]` |
| . | ...   | evaluate `(* 2 3)` -> `6`; the inner frame applies and pops | | |
| 6 | APPLY | -               | `6` | `[ARG done=[+] todo=[1]]` |
| . | ...   | evaluate `1`; the outer frame applies | | |
| 7 | APPLY | -               | `7` | `[]` -> **return 7** |

At step 5 the stack is two frames deep: the outer `+` parked with its first
operand still pending, the inner `*` just starting.  The inner call resolves to
`6` and its frame pops, the outer call carries on, and `K` falls back to empty.
Depth rises and falls exactly with how many calls are partway through -- which is
all a call stack ever was.

### Branching: `(if (= n 0) 0 99)` where `n = 0`

`if` evaluates its test under a `FRAME_IF` that remembers both branches:

| step | state | C | V | K |
|---|---|---|---|---|
| 1 | EVAL  | `[if [= n 0] 0 99]` | -    | `[]` |
| 2 | EVAL  | `[= n 0]`           | -    | `[IF 0/99]` |
| . | ...   | evaluate `(= n 0)` -> `#t` under the IF frame | | |
| 8 | APPLY | -                   | `#t` | `[IF 0/99]` |
| 9 | APPLY | -                   | `#t` | `[]` -> not `#f`, so pick `then`: C = `0` |
| 10| EVAL  | `0`                 | -    | `[]` |
| 11| APPLY | -                   | `0`  | `[]` -> **return 0** |

At step 9 the `FRAME_IF` pops and inspects the test value: `#t` is not `#f`, so
it picks the *then* branch.  Note that `0` would have been *false* back in #4 --
here, with real booleans, `0` is just a number and **only `#f` is false**.

### Tail recursion: `(countdown 100000)`

We cannot print 100,000 rows, but the number that matters is the depth of `K` at
the *start* of each `countdown` call:

| call | K at the body's first step |
|---|---|
| `countdown(100000)` | `[]` |
| `countdown(99999)`  | `[]` |
| `countdown(99998)`  | `[]` |
| ...                 | ... |
| `countdown(0)`      | `[]` -> returns `0` |

Inside one call, the body `(if (= n 0) 0 (countdown (- n 1)))` pushes frames for
the `if`, the `(= n 0)` test, and the `(- n 1)` argument -- and pops every one of
them before the tail call.  The tail call `(countdown (- n 1))` then applies
`countdown` and **installs the body with no frame of its own**.  So each call
begins at the same `K` depth the last one did.  `K` never grows; 100,000
iterations run in constant space -- exactly `IttyBittyLisp3`'s tail-call
optimization, now plainly visible as "the stack didn't get taller."

## The complete machine

```python
def lEval( expr, env ):
    C = expr
    V = None
    E = env
    K = []

    while True:

        # ----- state EVAL: descend into C (pushing frames) until a leaf -> V -----
        while True:
            if C in ('#t', '#f'):              # boolean literal -> itself
                V = C
                break
            elif isinstance( C, (int, float) ):  # number -> itself
                V = C
                break
            elif isinstance( C, str ):         # variable -> look it up
                V = E.lookup( C )
                break
            elif C[0] == 'quote':              # ['quote', datum] -> the datum, unevaluated
                V = C[1]
                break
            elif C[0] == 'lambda':             # ['lambda', params, *body] -> a closure
                V = ( VAL_CLOSURE, C[1], list(C[2:]), E )
                break
            elif C[0] == 'if':                 # ['if', test, then, else]
                K.append( (FRAME_IF, C[2], C[3], E) )
                C = C[1]
            elif C[0] == 'set!':               # ['set!', name, valueExpr]
                K.append( (FRAME_SET, C[1], E) )
                C = C[2]
            elif C[0] == 'begin':              # ['begin', *forms]
                forms = list( C[1:] )
                if len(forms) > 1:
                    K.append( (FRAME_SEQ, forms[1:], E) )
                C = forms[0]
            elif C[0] == 'let':                # ['let', ((name init)...), *body] -> sugar
                names = [ pair[0] for pair in C[1] ]
                inits = [ pair[1] for pair in C[1] ]
                C = [ ['lambda', names] + list(C[2:]) ] + inits
            else:                              # [fn, *args] -- an application
                K.append( (FRAME_ARG, [], list(C[1:]), E) )
                C = C[0]

        # ----- state APPLY: feed V to the top frame -----
        while True:
            if not K:
                return V

            frame = K.pop()
            ftag  = frame[0]

            if ftag == FRAME_IF:               # (FRAME_IF, then, else, env)
                C = frame[1] if V != '#f' else frame[2]   # #f is the only false
                E = frame[3]
                break

            elif ftag == FRAME_SET:            # (FRAME_SET, name, env)
                frame[2].set( frame[1], V )    # V is set!'s result; it flows on
                continue

            elif ftag == FRAME_SEQ:            # (FRAME_SEQ, remaining_forms, env)
                forms = frame[1]               # the previous form's value is discarded
                E = frame[2]
                if len(forms) > 1:
                    K.append( (FRAME_SEQ, forms[1:], E) )
                C = forms[0]
                break

            elif ftag == FRAME_ARG:            # (FRAME_ARG, done, todo, env)
                done = frame[1] + [V]
                todo = frame[2]
                if todo:                       # more operands to evaluate
                    K.append( (FRAME_ARG, done, todo[1:], frame[3]) )
                    C = todo[0]
                    E = frame[3]
                    break
                fn, args = done[0], done[1:]
                if callable( fn ):             # primitive: value flows on
                    V = fn( args )
                    continue
                _, params, body, clo_env = fn  # closure: bind params, run the body
                E = Environment( parent=clo_env, bindings=dict( zip(params, args) ) )
                if len(body) > 1:
                    K.append( (FRAME_SEQ, body[1:], E) )
                C = body[0]
                break
```

The `Environment` class is unchanged from `EVALUATOR2-DOC`, and the primitives
are the same plain Python functions as in `EVALUATOR3-DOC`:

```python
global_env = Environment( bindings={
    '+': lambda args: args[0] + args[1],
    '-': lambda args: args[0] - args[1],
    '*': lambda args: args[0] * args[1],
    '=': lambda args: '#t' if args[0] == args[1] else '#f',
    '<': lambda args: '#t' if args[0] <  args[1] else '#f',
} )
```

## TCO, now provable

In #4 the tail-call optimization was real but undemonstrated -- pure lambda
calculus has no way to write a deep loop.  With `set!` and primitives back, it
does:

```scheme
(set! countdown
  (lambda (n) (if (= n 0) 0 (countdown (- n 1)))))
(countdown 100000)   ; => 0
```

The recursive call to `countdown` is in tail position, so when `FRAME_ARG`
applies it, it installs the body and **pushes no frame**.  Across all 100,000
iterations the intermediate frames (`FRAME_IF`, the `FRAME_ARG`s for `=` and
`-`) are pushed and popped *within* each step, so `K` returns to the same depth
every time.  The continuation stack stays flat; nothing overflows.  This is the
same guarantee `IttyBittyLisp3` had -- only now it lives on an explicit,
inspectable stack instead of Python's.

## Running the Example

The complete working code is in `examples/IttyBittyLisp5.py`.

```
python examples/IttyBittyLisp5.py
```

*Next: `EVALUATOR6-DOC` compiles the AST to a flat bytecode and runs it on a
stack VM -- the same control story, with the dispatch decided once at compile
time instead of re-walked on every step.*

## Challenges

- **Add `and` / `or` with short-circuiting.** `(and a b)` must not evaluate `b`
  when `a` is `#f`.  That makes them special forms, not primitives -- so they
  need frames, like `if`.  Which existing frame is `and` almost identical to?

- **Add `let*`.** Sequential binding, where each initialiser sees the previous
  ones.  You can desugar it (to nested `let`s) the way `let` desugars to a
  lambda application -- no new frame required.

- **First-class continuations.** This is the payoff the whole CEK detour was
  for.  `K` is an ordinary Python list -- a *value*.  Add a `call/cc` primitive
  that hands the current `K` (a copy) to its argument as a reified continuation;
  invoking that continuation restores `K`.  Nothing in toys 1-3 could do this,
  because their continuation was the Python stack, which you cannot copy or
  resume.  Here it is just data.
