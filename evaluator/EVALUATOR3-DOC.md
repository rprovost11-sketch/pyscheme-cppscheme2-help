# Evaluator 3 - The Looping Evaluator (Tail-Call Optimization)

*Continues from `EVALUATOR2-DOC`: lexical scopes, let, and closures.*

The recursive evaluator of Parts 1 and 2 has a hard limit: Python allows
only about 1000 nested calls.  Because `lEval` recurses even in tail position,
any deeply tail-recursive Lisp program overflows the Python call stack.

The looping evaluator fixes this.  It keeps the current expression and
environment in two **registers** - C and E - and wraps the whole evaluator in
a `while True:` loop.  A tail position no longer recurses; it overwrites C
(and sometimes E) and loops back to the top.

  C - the Control:     the expression currently being evaluated  (was `expr`)
  E - the Environment: the bindings in scope                     (was `env`)

These two registers, plus a third introduced in Part 4, are the C, E, and K of
the CEK machine.  Part 3 makes C and E explicit; Part 4 adds K.

## The Problem with Naive Recursion

The recursive evaluator from `EVALUATOR1-DOC` handles `if` like this:

```python
elif expr[0] == 'if':
    cond = lEval( expr[1], env )
    return lEval( expr[2] if cond != '#f' else expr[3], env )   # recursive call
```

A tail-recursive Lisp countdown:

```lisp
(set! countdown (lambda (n) (if (= n 0) 0 (countdown (- n 1)))))
(countdown 100000)
```

generates 100,000 nested Python calls and crashes with `RecursionError`.
The recursive call to the tail branch of `if` is the culprit - each call
waits for the next one to return, so the stack grows one frame per
iteration.

## The Fix: Loop Instead of Recurse at Tail Positions

The looping evaluator wraps everything in `while True:`.  Instead of
recursing at the tail branch of `if`, it overwrites the C register and
`continue`s:

```python
def lEval( expr, env ):
    C = expr   # Control register
    E = env    # Environment register
    while True:
        ...
        elif C[0] == 'if':
            cond = lEval( C[1], E )   # condition: NOT tail - recurse
            C = C[2] if cond != '#f' else C[3]   # every value except #f is true
            continue                   # tail branch: loop, no new frame
```

`continue` jumps back to the top of the `while True:` loop.  No new Python
stack frame is created.  The same pattern applies to every other tail
position: the last form of a `begin`, the last form of a `let` body, and
- most importantly - any user-defined function call in tail position.

## Two Kinds of Sub-expression

Every sub-expression falls into one of two categories:

| Position | Action | Stack grows? |
|---|---|---|
| Tail position | `C = sub; continue` | No |
| Non-tail position | `lEval( sub, E )` | Yes |

The condition in `if`, all arguments to a function call, and all but the
last form in a `begin` or `let` body are **non-tail** - they recurse
normally.  Only the final result-producing expression gets the loop.

This is why C and E are only *almost* a full machine: the non-tail recursive
calls still ride the Python call stack, which is implicitly acting as the
continuation.  Part 4 makes that continuation an explicit register, K, and
removes the recursion entirely.

## Closures, Unchanged

`let`, `lambda`, and closures behave exactly as they did in Part 2 - the
`Environment` chain and the `Function` object are identical.  Tail-call optimization
changes only *how* the evaluator drives itself (loop vs. recurse), not *what*
a closure is.  The one place it shows up is the user-defined call path:
instead of recursing into the body, the evaluator reassigns E to the new
scope, sets C to the body's tail form, and loops.

```python
# Bind arguments in a new scope on the *captured* environment, then loop.
new_env = Environment( parent=fn.env, bindings=dict( zip( fn.params, args ) ) )
E = new_env
for sub in fn.body[:-1]:
    lEval( sub, E )        # non-tail body forms: recurse normally
C = fn.body[-1]
continue                   # tail call: loop - no stack growth
```

Tail-calling a function does not grow the Python call stack, regardless of
how deeply the Lisp recursion goes.

## The Complete Evaluator

```python
def lEval( expr, env ):
    # C and E are machine registers.  A tail position overwrites them and
    # loops (TCO) instead of recursing -- no new Python frame is pushed.
    C = expr   # Control:     the expression currently being evaluated
    E = env    # Environment: the bindings in scope
    while True:

        # --- Atoms ---

        if C in ( '#t', '#f' ):            # boolean literals self-evaluate (not identifiers)
            return C
        elif isinstance( C, str ):         # symbol - variable lookup
            return E.lookup( C )
        elif not isinstance( C, list ):    # number, etc. - self-evaluate
            return C

        # C is a non-empty list -- a special form or procedure call.  (A bare ()
        # is not a valid Scheme expression.)  Each special form is one more arm of
        # this single dispatch chain; the final else handles procedure calls.
        #
        # --- Special forms ---
        # Tail positions reassign C/E and `continue` - no new Python frame.
        # Non-tail positions call lEval() recursively, riding the Python call
        # stack (which here still plays the role of the continuation K).

        elif C[0] == 'if':
            cond = lEval( C[1], E )          # condition: not tail, recurse
            C = C[2] if cond != '#f' else C[3]  # every value except #f is true
            continue                          # tail branch: loop

        elif C[0] == 'begin':
            for sub in C[1:-1]:
                lEval( sub, E )               # non-tail forms: recurse
            C = C[-1]
            continue                          # tail: last form

        elif C[0] == 'let':
            vardefs = C[1]                    # list of [name, init-expr] pairs
            body    = C[2:]
            # Eval every init in the OUTER E (parallel let, not let*), then open
            # one new scope holding them all.
            E = Environment( parent=E,
                     bindings={ name: lEval( init, E ) for name, init in vardefs } )
            for sub in body[:-1]:
                lEval( sub, E )               # non-tail body forms: recurse
            C = body[-1]
            continue                          # tail: last body form

        elif C[0] == 'set!':
            val = lEval( C[2], E )
            E.set( C[1], val )
            return val

        elif C[0] == 'lambda':
            return Function( C[1], C[2:], E )

        elif C[0] == 'quote':
            return C[1]

        # --- Function call ---
        # All sub-expressions (operator + arguments) are non-tail.
        else:
            fn, *args = [lEval( subexpr, E ) for subexpr in C]
            if callable( fn ):                   # Python primitive: return directly
                return fn( args )

            # User-defined function: TCO - reassign the registers and loop.
            # The new scope is opened on the *captured* (lexical) environment,
            # not the caller's environment.  This is what makes closures work.
            new_env = Environment( parent=fn.env, bindings=dict( zip( fn.params, args ) ) )
            E = new_env
            for sub in fn.body[:-1]:
                lEval( sub, E )                  # non-tail body forms: recurse
            C = fn.body[-1]
            continue                             # tail call: loop, no stack growth
```

## Challenges

- **Add `while`.** Implement `(while condition body...)` as a special form
  using the looping pattern: evaluate the condition, and if true evaluate
  the body then `continue` back to the top.  Verify that it can iterate a
  million times without growing the Python call stack at all.

- **Add `cond` and `case` with correct TCO.** If you implemented `cond` in
  the Part 1 evaluator, port it here -- but make sure the chosen branch is in
  tail position (`C = branch; continue`).  Test it: a tail-recursive
  function dispatched through `cond` should handle 100,000 iterations just
  as well as one dispatched through `if`.

- **Named `let`.** `(let loop ((n 100000)) (if (= n 0) 0 (loop (- n 1))))`
  binds a local name `loop` to a function and immediately calls it.  It is
  the standard TCO-friendly looping idiom in Scheme.  Add it as a special
  form and observe that the recursive call in tail position gets TCO for
  free -- you do not need to do anything special to make it work.

- **Implement a macro expander.** A macro transforms code *before* it is
  evaluated.  Before the function-call path, check whether `C[0]` names a
  macro; if so, call its transformer on the *unevaluated* argument forms, set
  `C` to the result, and `continue` so the expansion is itself evaluated.
  `code is data` is what makes this work -- the transformer takes and returns
  plain lists, and the rest of the evaluator never knows.  Start with a
  procedural `define-macro`, e.g.
  `(define-macro (when test . body) \`(if ,test (begin ,@body)))`.  (Scheme's
  standard *hygienic* `syntax-rules` is a much larger, separate undertaking --
  see the Macros project in PROJECTS-DOC.)

- **Add a tracing mode.** When a trace flag is set, print each function call's
  name and arguments as you enter it and its result as you leave, indented by
  a depth counter you bump around the call.  (The looping evaluator has no `K`
  to read the depth from -- that trick is unique to the CEK machine in
  EVALUATOR4-DOC -- so track it yourself.)  Watch a *tail* call: it loops
  instead of returning, so it never prints a matching "exit" line -- a vivid
  picture of exactly what TCO eliminates.

## Running the Example

The complete working code is in `examples/IttyBittyLisp3.py`.
It demonstrates a tail-recursive countdown from 100,000 - which crashes
the recursive evaluator from `EVALUATOR1-DOC` but completes here in one
flat Python stack frame.

```
python examples/IttyBittyLisp3.py
```
```