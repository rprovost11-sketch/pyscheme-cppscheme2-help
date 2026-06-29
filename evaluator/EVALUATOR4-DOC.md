# Evaluator 4 - The CEK Machine

*Continues from `EVALUATOR3-DOC`: the looping evaluator.  Continues in
`EVALUATOR5-DOC`: the same machine with the full language.*

This is the hardest step in the series, and the only one that changes the
*shape* of evaluation rather than adding a feature.  It is also the turning
point: once you are through it, the rest of the series is downhill, because #5
and #6 only build on the one idea introduced here.

> **Should you be here?**  If you only want to *extend* the interpreter -- new
> special forms, macros, more primitives -- stop at `EVALUATOR3-DOC`.  The
> looping evaluator covers about everything you can build in a language *except*
> one thing: code that captures and manipulates its own control flow
> (`call/cc`, generators, coroutines).  Those need the continuation to be a
> value you can grab -- and making the continuation an explicit value is exactly
> what this chapter does.  Come here when you want that power; otherwise #3 is
> the friendlier place to work.

## The one idea: make the continuation explicit

Look back at every evaluator so far.  When `IttyBittyLisp1` evaluates the
condition of an `if`, it calls `lEval` and *waits* for the answer:

```python
conditionVal = lEval(condExpr, env)   # what happens AFTER this is on the Python stack
return lEval(thenBranch if ... else elseBranch, env)
```

The "what to do once the condition's value arrives" -- pick a branch and
evaluate it -- is not written down anywhere.  It lives *implicitly* in the
Python call stack: the half-finished `lEval` frame is sitting there, waiting to
resume.  Even the looping evaluator (#3) still does this for non-tail positions.

That implicit "what to do next" has a name: the **continuation**.  The CEK
machine's whole move is to stop leaning on Python's stack for it and instead
write the continuation down as **data** -- a stack of small records called
*continuation frames*, each one saying "when the next value arrives, here is
what to do with it."  With the continuation reified as data, the evaluator
never recurses; it just loops, pushing and popping frames.

## The machine state: C, E, K (and V)

A CEK machine is named for its three-register state (Felleisen & Friedman,
1987):

- **C - Control**: the expression currently being evaluated.
- **E - Environment**: the current lexical scope.
- **K - Kontinuation**: an explicit stack of continuation frames -- "what to do
  with the next value."

This toy adds a fourth register for clarity:

- **V - Value**: the value flowing *back* once a sub-expression finishes.

Keeping the value in its own register `V` means `C` is *always* code, never a
finished value -- so the machine never has to ask "is this thing code or a
result?"  (Textbook CEK machines reuse `C` for both and need a discriminator;
splitting out `V` avoids it.)

## Two states: EVAL and APPLY

The machine has exactly two states, and we write each as its own inner loop:

| State | Job |
|---|---|
| **EVAL**  | Walk *down* into `C`.  A leaf (number, variable, lambda) produces a value into `V`.  A compound (`if`, an application) needs a sub-expression evaluated first, so it **pushes a frame** and keeps descending. |
| **APPLY** | A value `V` has arrived.  Pop the top frame of `K` and feed `V` to it.  That either finishes the program (`K` empty) or sets up the next `C`/`E` to evaluate. |

Compare the three generations on the one question that matters -- what happens
in a non-tail position:

| Situation | Recursive (#1) | Looping (#3) | CEK (#4) |
|---|---|---|---|
| Tail position | recurse | `C = sub; continue` | set `C`, loop |
| Non-tail position | recurse | **recurse** | **push a frame**, set `C`, loop |

#3 made *tail* positions stop recursing.  #4 makes *non-tail* positions stop
too -- the frame on `K` is the heap-allocated stand-in for the Python frame #3
still used.  Nothing recurses; the Python call stack stays flat forever.

## The frames

This minimal machine is a pure lambda calculus with `if` (a number is true
unless it is `0`).  That tiny language needs just three frame kinds:

```python
FRAME_IF   = 0   # waiting on a test value      (FRAME_IF, then, else, env)
FRAME_ARG  = 1   # waiting on a function value   (FRAME_ARG, arg, env)
FRAME_CALL = 2   # waiting on an argument value  (FRAME_CALL, closure)
```

A frame is a plain **tagged tuple** -- the first slot is its kind, the rest is
the data it remembers.  There are no frame *classes* and no `step` methods: a
frame is inert data, and all the behavior lives in the machine loop.  That is
how the real interpreters (pyScheme / cppScheme2) do it, and it is the main
reason this machine reads cleanly.

## The machine

```python
def lEval( expr, env=None ):
    C = expr                                       # Control:      expression
    V = None                                       # Value:        result in APPLY
    E = Environment() if env is None else env      # Environment
    K = []                                         # Kontinuation: a stack of frames

    while True:

        # ----- state EVAL: descend into C, pushing frames, until a leaf -> V -----
        while True:
            if isinstance( C, int ):          # a number literal -> itself
                V = C
                break
            elif isinstance( C, str ):        # a variable -> look it up
                V = E.lookup( C )
                break
            elif C[0] == 'lambda':            # ['lambda', param, body] -> a closure
                V = ( VAL_CLOSURE, C[1], C[2], E )
                break
            elif C[0] == 'if':                # ['if', test, then, else]
                K.append( (FRAME_IF, C[2], C[3], E) )
                C = C[1]                      # evaluate the test first (keep descending)
            else:                             # [fn, arg] -- an application
                K.append( (FRAME_ARG, C[1], E) )
                C = C[0]                      # evaluate fn first (keep descending)

        # ----- state APPLY: feed V to the top frame -----
        while True:
            if not K:
                return V

            frame = K.pop()
            ftag  = frame[0]

            if ftag == FRAME_IF:              # (FRAME_IF, then, else, env)
                C = frame[2] if V == 0 else frame[1]   # 0 is false, all else true
                E = frame[3]
                break

            elif ftag == FRAME_ARG:           # (FRAME_ARG, arg, env)
                K.append( (FRAME_CALL, V) )   # remember the function value
                C = frame[1]                  # evaluate the argument next
                E = frame[2]
                break

            elif ftag == FRAME_CALL:          # (FRAME_CALL, closure)
                _, param, body, clo_env = frame[1]
                E = Environment( parent=clo_env, bindings={ param: V } )
                C = body                      # evaluate the body (no frame pushed!)
                break

        # fall through to the outer loop -- re-enter EVAL with the new C/E
```

A few things to notice in EVAL: the leaf cases set `V` and `break` (down to
APPLY); the compound cases push a frame, reassign `C`, and *do not* break, so
the EVAL loop keeps descending.  In APPLY, every frame either returns (`K`
empty) or sets up a fresh `C`/`E` and `break`s back up to EVAL.

## Step-by-Step: Watching the Machine Run

The best way to understand the machine is to watch it step.  Each table shows
the registers at the *start* of a loop iteration; `E` is shown only when it
changes.

### Constant: `42`

| step | state | C | V | K |
|---|---|---|---|---|
| 1 | EVAL  | `42` | -    | `[]` |
| 2 | APPLY | -    | `42` | `[]` -> **return 42** |

A number is a leaf: EVAL sets `V` and breaks to APPLY, which finds `K` empty and
returns.

### Variable: `x`, where `x = 7`

| step | state | C | V | K |
|---|---|---|---|---|
| 1 | EVAL  | `x` | -   | `[]` |
| 2 | APPLY | -   | `7` | `[]` -> **return 7** |

EVAL looks `x` up in `E` and sets `V`.  Same shape as a constant -- a variable is
just a leaf that takes one lookup.

### A function call: `((lambda (x) x) 7)`

Written `[['lambda', 'x', 'x'], 7]`.  Here `K` does some work:

| step | state | C | V | E | K |
|---|---|---|---|---|---|
| 1 | EVAL  | `[[lambda x x], 7]` | -       | `{}`    | `[]` |
| 2 | EVAL  | `[lambda x x]`      | -       | `{}`    | `[ARG 7]` |
| 3 | APPLY | -                   | closure | `{}`    | `[ARG 7]` |
| 4 | EVAL  | `7`                 | -       | `{}`    | `[CALL clo]` |
| 5 | APPLY | -                   | `7`     | `{}`    | `[CALL clo]` |
| 6 | EVAL  | `x`                 | -       | `{x:7}` | `[]` |
| 7 | APPLY | -                   | `7`     | `{x:7}` | `[]` -> **return 7** |

- **Step 1->2** -- an application: push `FRAME_ARG` to remember the argument `7`,
  then descend into the operator.
- **Step 3** -- the operator evaluated to a closure; APPLY pops `FRAME_ARG`,
  swaps it for a `FRAME_CALL` holding that closure, and goes off to evaluate the
  argument.
- **Step 5->6** -- the argument `7` has arrived; `FRAME_CALL` binds `x` to `7`
  in a fresh scope, sets `C` to the body, and **pushes nothing**.
- **Step 7** -- the body `x` looks up to `7`; `K` is empty; done.

`K` rose to depth 1 (first the argument, then the function) and fell back to
empty.  Every "what to do next" the recursive evaluator kept on the Python stack
is here a frame on `K` instead.

### Nested closures: `(((lambda (x) (lambda (y) x)) 3) 9)`

The curried constant function -- the inner closure captures `x = 3`, then ignores
its own argument `9`.  Watch `K` reach depth 2:

| step | state | C | V | E | K |
|---|---|---|---|---|---|
| 1 | EVAL  | `[[[lam x [lam y x]] 3] 9]` | -     | `{}`           | `[]` |
| 2 | EVAL  | `[[lam x [lam y x]] 3]`     | -     | `{}`           | `[ARG 9]` |
| 3 | EVAL  | `[lam x [lam y x]]`         | -     | `{}`           | `[ARG 9, ARG 3]` |
| 4 | APPLY | -                           | clo-x | `{}`           | `[ARG 9, ARG 3]` |
| 5 | EVAL  | `3`                         | -     | `{}`           | `[ARG 9, CALL clo-x]` |
| 6 | APPLY | -                           | `3`   | `{}`           | `[ARG 9, CALL clo-x]` |
| 7 | EVAL  | `[lam y x]`                 | -     | `{x:3}`        | `[ARG 9]` |
| 8 | APPLY | -                           | clo-y | `{x:3}`        | `[ARG 9]` |
| 9 | EVAL  | `9`                         | -     | `{}`           | `[CALL clo-y]` |
| 10| APPLY | -                           | `9`   | `{}`           | `[CALL clo-y]` |
| 11| EVAL  | `x`                         | -     | `{y:9}->{x:3}` | `[]` |
| 12| APPLY | -                           | `3`   | `{y:9}->{x:3}` | `[]` -> **return 3** |

The key moment is **step 7**: evaluating the inner `(lambda (y) x)` builds a
closure `clo-y` whose captured environment is `{x:3}`.  When `clo-y` is finally
called (step 10) its body `x` is looked up through that captured scope -- step
11, the new `{y:9}` chained onto `{x:3}` -- and finds `3`, regardless that the
argument was `9`.  That is lexical scope, on the explicit machine.

## Tail-call optimization, for free

Look again at `FRAME_CALL`: it binds the parameter, sets `C` to the body, and
**pushes nothing**.  The call does not leave a frame behind.  So if the body
ends in another call (a tail call), that call runs at the *same* `K` depth as
this one -- the stack does not grow.  TCO is not a special case here; it falls
out of "applying a function pushes no frame."

(In this pure-λ toy there is no way to write a deep loop -- that needs
recursion by name, which needs `set!` or the Y-combinator -- so we can only
*describe* the TCO here.  `EVALUATOR5-DOC` adds the language back and proves it
with `(countdown 100000)`.)

## Why so small a language?

Making the continuation explicit is the single hardest idea in the series.  So
this chapter strips the language down to the smallest thing that still has
closures and control flow -- pure lambda calculus plus `if` -- to keep the
*machine* in the foreground.  The fuller language of toys 1-3 (`let`, `set!`,
`begin`, primitives, multi-argument calls, `#t`/`#f`) does not change the
machine's shape at all; it only adds more frame kinds.  Showing exactly that is
the job of `EVALUATOR5-DOC`.

## Running the Example

The complete working code is in `examples/IttyBittyLisp4.py`.

```
python examples/IttyBittyLisp4.py
```

*Next: `EVALUATOR5-DOC` puts the full language back on this same machine.*

## Challenges

- **Add booleans and Scheme truthiness.** Give the machine real `#t`/`#f`
  literals (self-evaluating, like numbers) and change `FRAME_IF` to treat `#f`
  as the only false value.  This is the first step `EVALUATOR5-DOC` takes --
  try it before reading ahead.

- **Trace a nested program.** Hand-trace `(((lambda (x) (lambda (y) x)) 3) 9)`
  as a `C / V / E / K` table like the one above.  Watch the curried closures
  capture their environments and the `K` stack stay shallow.

- **Add a one-argument primitive.** Put a Python function (say a `double`) in
  the starting environment and make the application path apply it when the
  function value is callable rather than a closure.  Notice that a primitive
  *produces a value* -- so, unlike a closure, it keeps the machine in the APPLY
  state instead of dropping back to EVAL.  (This is the change that makes the
  APPLY loop genuinely iterate; `EVALUATOR5-DOC` leans on it.)
