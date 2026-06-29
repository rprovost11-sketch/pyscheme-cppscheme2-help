# Evaluator 2 - Lexical Scopes, Let, and Closures

*Continues from `EVALUATOR1-DOC`: the minimal recursive evaluator.*

The flat dictionary in Part 1 supports exactly one scope: the global
environment.  Every variable lives there and every assignment affects it.
There is no way for two function calls to maintain separate copies of a local
variable, or for one name to mean something different inside a function than
it does outside.

The solution is a **stack of scopes**.  Instead of one flat dictionary,
each scope is a small `Environment` node that holds its own bindings and points to a
parent.  Looking up a name walks the chain outward until it is found; inner
bindings shadow outer ones naturally.

Once you have this mechanism, three things follow almost for free.

**`let`** is the simplest: it opens a fresh `Environment`, evaluates each init-form in
the *current* scope, binds the results in the new scope, and evaluates its
body there.  No concept beyond `Environment` itself is required.

**User-defined functions** are the next step.  A call binds argument values in
a fresh `Environment` and evaluates the function body there.  The function object just
records `(params, body, captured-env)`.

**Closures** are what you get when you choose *lexical* scoping over *dynamic*
scoping.  The question is: which `Environment` does the fresh call scope extend - the
*caller's*, or the one active when the function was *defined*?  Extending the
definition-time env (lexical scoping) means free variables always refer to
what the programmer saw when writing the function.  The `Function.env` field
captures that env at `lambda`-evaluation time; calls open their new scope off
the captured env, not off the caller's.

A useful side-effect: because each call opens its own fresh scope, recursive
calls do not interfere with each other.  Each level of recursion has its own
independent copy of the local variables.

## Environment and Function

```python
# ---------------------------------------------------------------------------
# Environment: a linked chain of scopes
# ---------------------------------------------------------------------------

class Environment:
    def __init__( self, parent=None, bindings=None ):
        self.vars    = dict(bindings or {})
        self.parent  = parent
        self._global = parent._global if parent else self   # direct handle to the root

    def lookup( self, name ):
        scope = self
        while scope:
            if name in scope.vars:
                return scope.vars[name]
            scope = scope.parent
        raise NameError( f'Unbound variable: {name}' )

    def set( self, name, val ):
        # Walk to the innermost scope that already owns the name.
        scope = self
        while scope:
            if name in scope.vars:
                scope.vars[name] = val
                return val
            scope = scope.parent
        # Name not found anywhere -- create it in the global scope.  The _global
        # handle goes straight there, with no second walk down the chain.
        self._global.vars[name] = val
        return val

# ---------------------------------------------------------------------------
# Function: a closure capturing its lexical environment
# ---------------------------------------------------------------------------

class Function:
    def __init__( self, params, body, env ):
        self.params = params   # list of parameter name strings
        self.body   = body     # list of body expressions; last is the tail
        self.env    = env      # lexical environment at definition time
```

`Environment.lookup` walks the stack outward, so inner bindings shadow outer ones.
`Environment.set` does the same walk but mutates: if the name already exists anywhere
in the stack it updates it there; only when the name is absent everywhere does
it create a new binding in the global (root) scope.  That is the assignment
semantics of `set!` -- update the nearest existing binding -- with a lenient
twist for this small interpreter: a brand-new name is created at the global
scope rather than raising an error.

`Function.env` is the key to lexical scoping: it records *which* `Environment` was
active at the point `lambda` was evaluated.  When the function is called, the
new argument scope is chained off that captured env, not the caller's.

## Updated Global Environment

The global environment becomes the root `Environment` instead of a plain dict.  The
`print` primitive is added so demos can produce output.

```python
# ---------------------------------------------------------------------------
# Primitives and global environment
# ---------------------------------------------------------------------------

global_env = Environment( bindings={
    '+':     lambda args: args[0] + args[1],
    '-':     lambda args: args[0] - args[1],
    '*':     lambda args: args[0] * args[1],
    '=':     lambda args: '#t' if args[0] == args[1] else '#f',
    '<':     lambda args: '#t' if args[0] <  args[1] else '#f',
    'print': lambda args: (print( args[0] ), args[0])[1],
} )
```

## Updated Evaluator

The evaluator gains three new special-form cases (`let`, `lambda`, and the
user-defined function call path) and switches from plain dict operations to
`Environment` methods.

```python
def lEval( expr, env ):
    if expr in ( '#t', '#f' ):         # boolean literals self-evaluate (not identifiers)
        return expr
    elif isinstance( expr, str ):      # symbol -- variable lookup
        return env.lookup( expr )
    elif not isinstance( expr, list ): # number, etc. -- self-evaluate
        return expr

    # expr is a non-empty list -- a special form or procedure call.  (A bare ()
    # is not a valid Scheme expression.)  Every special form is one more arm of
    # this single dispatch chain; the final else handles procedure calls.

    elif expr[0] == 'if':
        cond = lEval( expr[1], env )
        return lEval( expr[2] if cond != '#f' else expr[3], env )  # #f is the only false

    elif expr[0] == 'begin':
        for sub in expr[1:-1]:
            lEval( sub, env )
        return lEval( expr[-1], env )

    elif expr[0] == 'let':
        vardefs = expr[1]               # list of [name, init-expr] pairs
        body    = expr[2:]
        new_env = Environment( parent=env )
        for vardef in vardefs:
            new_env.vars[vardef[0]] = lEval( vardef[1], env )
        for sub in body[:-1]:
            lEval( sub, new_env )
        return lEval( body[-1], new_env )

    elif expr[0] == 'set!':
        val = lEval( expr[2], env )
        env.set( expr[1], val )
        return val

    elif expr[0] == 'lambda':
        return Function( expr[1], expr[2:], env )

    elif expr[0] == 'quote':
        return expr[1]

    # Otherwise it's a function call: evaluate the operator and all arguments.
    else:
        fn, *args = [lEval( subexpr, env ) for subexpr in expr]
        if callable( fn ):               # Python primitive
            return fn( args )

        # User-defined function: bind arguments in a new scope on the captured env.
        new_env = Environment( parent=fn.env, bindings=dict( zip( fn.params, args ) ) )
        for sub in fn.body[:-1]:
            lEval( sub, new_env )
        return lEval( fn.body[-1], new_env )
```

What changed from Part 1:

- **Symbol lookup**: `env[expr]` -> `env.lookup( expr )` (walks the scope chain)
- **`set!`**: `env[var] = val` -> `env.set( var, val )` (walks chain to find existing binding)
- **`let`**: opens a new inner `Environment`, evaluates each init-form in the *outer*
  scope, binds results in the inner scope, then evaluates the body there
- **`lambda`**: constructs a `Function` capturing the current `env` -- this is
  what makes it a closure
- **Primitive call**: `fn( args )` -- unchanged in structure
- **User-defined call**: binds arguments in a new scope chained off `fn.env`
  (the *captured* env, not the caller's), then evaluates the body recursively

## Running the Example

The complete working code is in `examples/IttyBittyLisp2.py`.

```
python examples/IttyBittyLisp2.py
```

It demonstrates closures, higher-order functions, local bindings, and
recursion.  Note that deep recursion will overflow Python's call stack --
there is no tail-call optimization here.  That is what `EVALUATOR3-DOC` and
`IttyBittyLisp3.py` address.

## Challenges

- **Add `let*`.** `let*` is like `let` but each init-form is evaluated in a
  scope that already includes the previous bindings, so `(let* ((x 1) (y (+ x 1))) y)`
  works where `let` would fail.  The implementation differs from `let` by
  exactly one line.  Try to identify it before writing any code.

- **Add `define`.** The procedure-shorthand form `(define (square x) (* x x))`
  is sugar for `(set! square (lambda (x) (* x x)))`.  Add it as a special form.
  Then ask: could this be implemented as a *macro* instead of a special form?
  What would that require, and why can't you do it yet?

- **Add variadic functions.** Allow a rest parameter: `(lambda (x . rest) ...)`,
  where `rest` is bound to the list of any extra arguments.  The parser
  already tokenizes `.` -- you just need `lEval` to detect it at binding
  time and collect the remaining args into a list.

- **Implement `apply`.** `(apply + '(1 2 3))` calls `+` with the elements
  of the list as arguments.  It needs to be a special form rather than a
  primitive because it calls a user-defined function -- and user-defined
  calls need to go through `lEval`'s function-call path, not through a
  Python lambda.

## What the Full Interpreter Adds

The `lEval` above is the complete conceptual core.  The full interpreter's
`cek_eval` in `Evaluator.py` extends it with:

- **More special forms** inlined for performance: `let*`, `cond`, `case`,
  and `apply`
- **Tail-call optimization (TCO)**: the evaluator loops instead of recursing
  for tail positions, so deeply recursive Lisp code does not overflow
  Python's call stack
- **Macro expansion**: before function dispatch, macro calls are expanded
  inline and re-evaluated in the same loop iteration
- **Continuations, tracing, and full argument binding** for the complete
  feature set
```