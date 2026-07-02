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

## Closures from the programmer's seat

Before implementing closures it is worth knowing what one *is* from the outside,
because you have almost certainly written them already - just without the name.
In Python:

```python
def make_adder( n ):
    def add( x ):
        return n + x      # add refers to n, which belongs to make_adder
    return add

add5 = make_adder( 5 )    # make_adder has returned...
add5( 3 )                 # ...yet add5 still knows n = 5  ->  8
```

`make_adder` has returned, its call frame is gone, and still `add5` remembers
`n = 5`.  A function bundled with the variables it referred to from its
enclosing scope is called a **closure**, and `n` is a *captured* variable.  You
reach `n` only by calling `add5`; there is no other handle to it.

The language we are building has the same behavior.  Here is `make_adder`
transliterated, and then a second closure that shows the two properties that
make closures more than a curiosity - captured state can be *mutated*, and each
construction is *independent*:

```scheme
(set! make-adder
  (lambda (n)
    (lambda (x) (+ n x))))     ; the inner lambda captures n

(set! add5 (make-adder 5))
(add5 3)                       ; => 8   -- n = 5 lived on inside add5

(set! make-counter
  (lambda ()
    (let ((count 0))                                  ; private state...
      (lambda () (begin (set! count (+ count 1))      ; ...mutated by set!...
                        count)))))

(set! c1 (make-counter))
(set! c2 (make-counter))
(c1)   ; => 1
(c1)   ; => 2    -- the same captured count, incremented and remembered
(c2)   ; => 1    -- ...but c2 captured its OWN count, untouched by c1
```

So a closure gives a programmer three things, and they are worth naming because
the rest of this series - and the optional OO fork - leans on all three:

- **capture** - the inner function keeps the variables it saw when defined
  (`n`, `count`)
- **persistence and mutation** - that captured state outlives the call that
  created it, and `set!` can change it in place
- **independence** - each call to the maker (`make-counter`) captures a *fresh*
  set of variables, so `c1` and `c2` never collide

Everything below is the machinery that delivers exactly this.  The one idea that
makes it work is that a function must remember the environment it was *defined*
in - so watch for the `Function.env` field, and the moment a call chains its new
scope off that captured env rather than the caller's.  That single choice is the
whole of what you just saw.

## Environment and Function

```python
# ---------------------------------------------------------------------------
# Environment: a linked chain of scopes
# ---------------------------------------------------------------------------

class Environment:
    def __init__( self, parent=None, bindings=None ):
        self._bindings = dict(bindings or {})
        self._parent   = parent
        self._global   = parent._global if parent else self   # direct handle to the root

    def lookup( self, name ):
        scope = self
        while scope:
            if name in scope._bindings:
                return scope._bindings[name]
            scope = scope._parent
        raise NameError( f'Unbound variable: {name}' )

    def set( self, name, value ):
        # Walk to the innermost scope that already owns the name.
        scope = self
        while scope:
            if name in scope._bindings:
                scope._bindings[name] = value
                return value
            scope = scope._parent
        # Name not found anywhere -- create it in the global scope.  The _global
        # handle goes straight there, with no second walk down the chain.
        self._global._bindings[name] = value
        return value

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

These two classes are built on opposite principles, on purpose.  `Environment`
**hides** its fields -- `_bindings`, `_parent`, and `_global` all carry a leading
underscore, and only `lookup`, `set`, and the constructor ever touch them.  That
encapsulation means you could change *how* a scope stores its bindings without
editing a single line of the evaluator.  `Function` is the reverse: a plain
record (a C-style struct) -- public fields, just a constructor -- whose
*behavior* lives in the evaluator beside the rest of the dispatch rather than as
a `__call__` method on the class.  A useful rule of thumb: encapsulate what has
invariants to protect; leave open what is merely data.

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

    # ---- State = EVAL (dispatch on expression syntax) ----
    if expr in ('#t', '#f'):           # boolean literals self-evaluate (they are
        return expr                    # data, not identifiers -- never looked up)
    elif isinstance(expr, str):        # a symbol -- look it up in the environment
        return env.lookup(expr)
    elif not isinstance(expr, list):   # everything else evaluates to itself
        return expr

    # expr is a non-empty list -- a special form or a procedure call.  (A bare
    # () is not a valid Scheme expression.)

    # Handle Special operators inline
    elif expr[0] == 'set!':
        name, valExpr = expr[1:]
        val = lEval(valExpr, env)
        return env.set(name, val)

    elif expr[0] == 'if':
        condExpr, thenExpr, elseExpr = expr[1:]
        condVal = lEval(condExpr, env)
        return lEval(elseExpr if condVal == '#f' else thenExpr, env)

    elif expr[0] == 'begin':
        for subExpr in expr[1:-1]:     # non-tail forms: evaluated for effect
            lEval(subExpr, env)
        return lEval(expr[-1], env)    # tail form: its value is the result

    elif expr[0] == 'lambda':
        params, *body = expr[1:]
        return Function(params, body, env)

    elif expr[0] == 'quote':
        return expr[1]

    elif expr[0] == 'let':
        bindingPairs, *body = expr[1:]
        # Eval each init in the OUTER env, then open one new scope holding them all,
        # passed through the constructor so the bindings stay private to Environment.
        initialBindings = { name: lEval(initExpr, env) for name, initExpr in bindingPairs }
        new_env = Environment( parent=env, bindings=initialBindings )

        # Execute body in new_env
        for subExpr in body[:-1]:             # non-tail body forms
            lEval(subExpr, new_env)
        return lEval(body[-1], new_env)       # tail body form

    else:
        fn, *args = [ lEval(elt, env) for elt in expr ]   # eval operator + operands

        # ---- State = APPLY (invoke a procedure on evaluated args) ----
        if callable(fn):                   # primitive implemented in Python
            return fn(args)
        else:
            # user-defined function: evaluate its body in a fresh local scope chained
            # off the *captured* (lexical) environment, not the caller's.
            initialBindings = dict(zip(fn.params, args))
            new_env = Environment(parent=fn.env, bindings=initialBindings)

            # Execute the body in new_env
            for subExpr in fn.body[:-1]:       # non-tail body forms
                lEval(subExpr, new_env)
            return lEval(fn.body[-1], new_env)   # tail body form
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