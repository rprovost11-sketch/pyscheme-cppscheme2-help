# Chapter 2: Functions, Scopes, and Closures

Chapter 1's interpreter had exactly one environment: a single global dictionary,
where every name lived.  That was enough because the language had no way to make
functions of its own, and closing that gap is the whole of this chapter.  Adding
functions sounds like a small step, but it forces a genuinely new idea, and
everything here grows out of it.

Here is the problem.  Suppose we can write a function and call it:

```scheme
(set! square (lambda (n) (* n n)))
(square 5)                            ; => 25
```

Here `lambda` makes a function with no name (think of a Python `def` stripped of
its name) and `set!` gives this one the name `square`.  (We build `lambda`
properly in §2.1; for now, take `square` as a one-argument function that returns
`n * n`.)

To run `(square 5)`, the interpreter has to bind the parameter `n` to `5` and
then evaluate the body `(* n n)`.  But *where* does `n` live?  If we drop it into
the one global dictionary, things go wrong quickly.  It leaks, for a start:
after the call, `n` is still sitting in the globals, set to `5`, for no reason.
And it breaks outright as soon as a function calls itself:

```scheme
(set! factorial
  (lambda (n)
    (if (= n 0) 1 (* n (factorial (- n 1))))))
(factorial 3)
```

Computing `(factorial 3)` calls `(factorial 2)`, which calls `(factorial 1)`, and
so on, several calls all alive at once, each needing its *own* `n` (3, then 2,
then 1).  A single global `n` cannot be all of those at the same moment; the
inner calls would trample the outer ones, and the answer would be nonsense.

So each function call needs its **own** private place for its parameters, a
place that exists only while that call is running, and that falls back to the
globals for any name it does not define itself.  That place is called a
**scope**.  Chapter 1 had exactly one scope, the global one; this chapter gives
every call its own.

That single change, many scopes instead of one, is what makes user-defined
functions possible.  It is also, as we will see, what makes **closures**
possible: functions that remember the scope they were born in.  Along the way we
add two new forms to the language: `lambda`, which makes a function, and `let`,
which makes a local scope directly.

This is also where the `env` parameter we threaded through `lEval` back in
Chapter 1 starts to *vary*.  There, evaluation was always relative to the one
global environment: the parameter told the truth about what evaluation depends
on, but there was only ever one thing to pass.  Now `lEval( expr, env )` will be
handed a *different* `env` for almost every call (that call's own scope) and
the parameter we set up is exactly what carries it.


## 2.1 The language we're building in this chapter

Everything from Chapter 1 stays: data, names, `set!`, `if`, `begin`, `quote`, and
function calls all behave as before.  This chapter adds just two new special
forms.

**`lambda` makes a function.**  A `lambda` is a function with no name of its own:
the closest Python analogy is a `def` with the name taken off.  (Python has a
`lambda` too, but it is limited to a single expression; Scheme's holds a full
body, so `def` is the better comparison.)  `(lambda (params...) body...)`
evaluates to a function *value*: something you can store under a name, pass
around, and call later.  The parameter list names the inputs; the body runs when
the function is called, with the parameters bound to the argument values.  A
`lambda` starts out nameless, and a `set!` is what gives it a name:

```scheme
(set! square (lambda (n) (* n n)))   ; make a nameless function, then name it square
(square 5)                           ; call it: n = 5, body (* n n)   =>  25
```

Calling a user-defined function uses the same shape as calling a primitive,
`(square 5)`, head first, so from the caller's side there is no difference
between `(+ 1 2)` and `(square 5)`.  And a function is an ordinary value: you can
store it under a name, as we did, or pass it to another function, exactly as you
would a number.  A value you can pass around this freely is called **first-class**:
functions in our Lisp are first-class, and that fact is doing more work than it
looks.

**`let` makes a local scope.**  `(let ((name init) ...) body...)` evaluates each
`init`, binds the results to the names in a fresh local scope, and evaluates the
body in that scope.  The names exist only inside the `let`:

```scheme
(let ((a 3)
      (b 4))
  (print a)                          ; the body may hold several expressions...
  (+ (* a a) (* b b)))               ; ...run in order; the last value wins   =>  25
```

The body here has two expressions, and that is worth pausing on: a body (in a
`let` or a `lambda`) may hold as many expressions as you like.  They run in
order, exactly like a `begin` from Chapter 1, and the value of the *last* one is
the result; any earlier ones run for their side effects (here, the `print`, which
displays `3`).  Most examples you meet use single-expression bodies, but multiple
expressions are perfectly ordinary, correct code.

Outside the `let`, `a` and `b` are gone.  A `let` is the simplest way to carve
out a small private workspace, a scope you open by hand, without defining a
function to do it.

With `lambda`, the language can now express recursion, because a function can
refer to its own name and call itself:

```scheme
(set! factorial
  (lambda (n)
    (if (= n 0)
        1
        (* n (factorial (- n 1))))))
(factorial 5)                        ; => 120
```

And a function can build and return *another* function, which is where the
chapter is really headed:

```scheme
(set! make-adder
  (lambda (n)
    (lambda (x) (+ n x))))           ; the inner function uses n from the outer one
(set! add5 (make-adder 5))
(add5 3)                             ; => 8
```

That last example is the interesting one.  `make-adder` has already returned by
the time we call `add5`, its `n` should be long gone, yet `add5` still knows
that `n` is `5`.  Making sense of *how* is the subject of the next section.

The two new forms, one line each:

- **`lambda`**: `(lambda (params) body...)` makes a function value; calling it
  binds the parameters to the argument values and evaluates the body.
- **`let`**: `(let ((name init) ...) body...)` opens a fresh local scope with the
  given names bound, evaluates the body in it, and discards the scope afterward.


## 2.2 Closures, from the programmer's seat

Return to the puzzle from §2.1: `make-adder` builds and returns a function, and
the returned function still knows `n = 5` even though `make-adder` has already
finished.  Before we implement anything, it is worth seeing what is going on from
the outside, because if you have written much Python, you have almost certainly
done this already, just without a name for it.

Here it is in Python:

```python
def make_adder( n ):
    def add( x ):
        return n + x          # add uses n, which belongs to the enclosing make_adder
    return add

add5 = make_adder( 5 )        # make_adder runs, returns the inner function, and exits
add5( 3 )                     # yet add5 still knows n = 5   ->   8
```

Look at what has to be true for `add5( 3 )` to return `8`.  By the time we call
it, `make_adder` has returned; its call is over, its `n` should be gone.  And yet
`add` (now living under the name `add5`) still finds `n = 5`.  The inner
function did not merely *use* `n` while `make_adder` was running; it *kept* it.

A function bundled together with the variables it uses from its enclosing scope is
called a **closure**.  In `add`, the name `x` is a parameter: it is bound right
there, by the call.  The name `n` is not; `n` comes from outside `add`, from the
scope that enclosed it.  A name used in a function but not bound inside it is
called a **free variable**, and a closure is exactly a function plus the free
variables it *captured* from where it was defined.  Here `n` is the captured
variable, and the only way to reach it is to call `add5`: there is no other
handle to it.

One question is worth making explicit, because the whole idea turns on it: *which*
`n` does `add` use?  It uses the `n` of the `make_adder` call that created it
(the scope `add` was **written inside**) and it would do so no matter where
`add5` is later called from.  A function's free variables are resolved by where
the function *sits in the source*, not by who calls it.  That rule is called
**lexical scoping**, and it is the rule our interpreter will follow: a function
remembers the scope it was *defined* in, and looks up its free variables there.

The word *lexical* itself points at the source text.  A name's scope (the
stretch of program where that name is visible) is something you can literally
see in the code: it runs between the pair of delimiters that enclose it, the
parentheses of a `lambda` or `let` in Lisp (a pair of braces `{ }` in many other
languages).  Code written between those delimiters can refer to the name; code
outside them cannot.  (The *value* a captured name holds can outlive that textual
region, that is exactly what `add5` does with `n`, but the stretch of source
where the *name* is in scope stays fixed by where the delimiters sit.)

Our Lisp behaves exactly the same way.  Here is `make-adder` again:

```scheme
(set! make-adder
  (lambda (n)
    (lambda (x) (+ n x))))       ; the inner lambda's n is free -- captured from make-adder

(set! add5 (make-adder 5))
(add5 3)                         ; => 8   -- n = 5 lived on inside add5
```

`make-adder` is a function that returns a function, a **higher-order function**,
in the usual jargon.  Calling `(make-adder 5)` runs the outer `lambda` with
`n = 5`, and its body builds the inner `lambda`.  That inner function captures the
`n` it can see (`5`) and *that* captured function is what we store as `add5`.
When we later call `(add5 3)`, the `x` is `3`, the free `n` is still `5`, and the
sum is `8`.

Closures earn their keep when the captured state does more than sit there.  This
next one shows two further properties: the captured state can be *changed* and
made to *persist*, and each closure gets its *own* copy:

```scheme
(set! make-counter
  (lambda ()
    (let ((count 0))                       ; a private variable...
      (lambda ()
        (set! count (+ count 1))           ; ...bumped on each call (a two-expression body)...
        count))))                          ; ...and returned

(set! c1 (make-counter))
(set! c2 (make-counter))
(c1)     ; => 1
(c1)     ; => 2    -- the same captured count, incremented and remembered
(c2)     ; => 1    -- but c2 captured its OWN count, untouched by c1
```

Each call to `make-counter` opens a fresh `let` scope with its own `count`, and
returns a function that captures it.  `c1` and `c2` are built by two separate
calls, so they capture two separate `count`s: bumping `c1` never touches `c2`.
And the count survives *between* calls: it is not reset each time `c1` runs,
because the scope holding it was captured once, when `c1` was created, and lives
as long as `c1` does.

So a closure gives you three things, worth naming because the rest of the series
leans on all three:

- **capture**: the function keeps the free variables it saw where it was defined
  (`n`, `count`).
- **persistence and mutation**: that captured state outlives the call that
  created it, and `set!` can change it in place.
- **independence**: each construction (each `make-counter` call) captures a
  *fresh* set of variables, so `c1` and `c2` never collide.

Everything in the rest of this chapter is the machinery that delivers exactly
this.  And it all comes down to a single decision, worth holding in mind as we
build: **a function must remember the environment it was defined in.**  When we
implement `lambda`, it will capture the current environment; when we call a
function, we will run its body in a new scope chained off that *captured*
environment, not off the caller's.  That one choice is the whole of what you
just saw.


## 2.3 The environment chain

We have the behaviour we want; now for the structure that produces it.  The clue
is in the word *nest*.

Look again at the shape of `make-counter`:

```scheme
(lambda ()
  (let ((count 0))
    (lambda () (set! count (+ count 1)) count)))
```

The scopes here sit *inside* one another.  The innermost `lambda` lives inside
the `let`'s scope (where `count` is bound); the `let` lives inside the outer
`lambda`'s scope; and that, in the end, lives inside the global scope.  Scopes
nest, and nesting is a *stack*: entering a scope pushes a new one on top,
leaving it pops back to the one that encloses it.  Once you see that, the data
structure is forced.  Each scope has to be a small table of its own bindings plus
a link to the scope that encloses it, and a chain of these (local at the near
end, global at the far end) *is* the stack of scopes currently in effect.

That chain is the whole of the `Environment` class.  Each `Environment` object is
one scope:

```python
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
        scope = self
        while scope:
            if name in scope._bindings:            # name already bound here?
                scope._bindings[name] = value      # update it in place
                return value
            scope = scope._parent
        self._global._bindings[name] = value       # bound nowhere? create it in the globals
        return value
```

An `Environment` holds three things: `_bindings`, its own little dictionary of
names; `_parent`, the environment that encloses it (or `None`, for the global
one); and `_global`, a direct shortcut to the root of the chain, which we will
use in a moment.  (The leading underscores are Python's usual signal for
"internal": code outside the class goes through `lookup` and `set`, never the
dictionaries directly.)

**`lookup` walks the chain outward.**  To find a name, it checks the current
scope's bindings; if the name is not there, it steps to the parent and checks
again, and so on, until it either finds the name or runs off the end of the chain
(a `None` parent) and reports the name unbound.  This walk is exactly what gives
free variables their meaning: an inner function that does not define `n` finds it
by stepping outward to the scope that does.

The walk also produces **shadowing** for free.  If the same name is bound in two
scopes (say a parameter `x` inside a function and another `x` out in the global
scope) `lookup` finds the inner one first, because it is reached first.  The
inner binding *shadows* the outer: the outer is still there, just hidden for as
long as the inner scope is on the chain.  A local name quietly wins over a global
of the same name, which is what you want.

**`set` walks the same way, but to assign.**  It looks outward for a scope that
*already* has the name and, finding one, updates the binding right there.  This is
why `(set! count ...)` inside the counter reaches back to the `count` in the
captured `let` scope and mutates *that*: the very thing that let the counter
remember its value between calls.  If the name is bound nowhere on the chain,
`set` falls back to creating it in the global scope, the same lenient rule from
Chapter 1, now expressed over a stack.  This is the one place the `_global`
shortcut earns its keep: instead of walking to the end of the chain a second time,
`set` jumps straight to the root and drops the new binding there.

The global environment is now simply the root of this chain: an `Environment` with
no parent, holding the primitives.  Every other environment comes into being
*during* evaluation (one per `let`, one per function call) each chained onto the
scope it is meant to see.


## 2.4 Evaluating functions: the simple version first

With the `Environment` stack in hand, we can teach `lEval` to make and call
functions.  There is one genuinely subtle point in doing so (the closure hookup)
and it is far easier to see if we set it aside at first.  So this section builds
functions that work for everything *except* closures, and §2.5 adds the single
change that completes them.

The cases carried over from Chapter 1 barely change.  They just talk to the
`Environment` through its methods instead of poking a dictionary directly:

```python
elif isinstance(expr, str):        # a name: look it up along the chain
    return env.lookup(expr)

elif expr[0] == 'set!':
    name, valExpr = expr[1:]
    val = lEval(valExpr, env)
    return env.set(name, val)
```

`if`, `begin`, and `quote` are unchanged: they simply pass `env` along to their
sub-evaluations.  Everything new is about *scopes*, and there are three additions:
`let`, `lambda`, and the function-call path.

**`let` opens a scope.**  It evaluates each init expression in the *current*
environment, collects the results, and opens one new `Environment` (chained onto
the current one) that holds them.  The body runs in that new scope:

```python
elif expr[0] == 'let':
    bindingPairs, *body = expr[1:]
    initialBindings = { name: lEval(initExpr, env) for name, initExpr in bindingPairs }
    new_env = Environment( parent=env, bindings=initialBindings )
    for subExpr in body[:-1]:          # earlier body expressions, run for their effects
        lEval(subExpr, new_env)
    return lEval(body[-1], new_env)    # the last one's value is the result
```

Evaluating the inits in the *outer* environment, before the new scope exists, is
what makes this `let` rather than its sequential cousin `let*`, a distinction
left to the challenges.  And notice the body is an implicit `begin`, exactly as in
§2.1: run the earlier expressions, return the value of the last.

**`lambda` makes a function.**  For now, a function is just its parameter list and
its body, packaged into a small record:

```python
class Function:
    def __init__( self, params, body ):
        self.params = params   # list of parameter names
        self.body   = body     # list of body expressions; the last is the result

...

elif expr[0] == 'lambda':
    params, *body = expr[1:]
    return Function(params, body)
```

Evaluating a `lambda` runs nothing; it just packages the parameters and body into
a `Function` object and returns it, which is exactly why a function is a value
you can store and pass around.

**Calling a function.**  A call is still the `else` branch: evaluate the head and
the arguments.  The head may now evaluate to either a Python primitive or one of
our `Function` values, so we check which:

```python
else:
    fn, *args = [ lEval(elt, env) for elt in expr ]

    if callable(fn):                       # a primitive (a plain Python def or lambda)
        return fn(args)
    else:                                  # a user-defined function
        call_env = Environment(parent=global_env,
                               bindings=dict(zip(fn.params, args)))
        for subExpr in fn.body[:-1]:
            lEval(subExpr, call_env)
        return lEval(fn.body[-1], call_env)
```

The user-defined case is the intuitive core of function calling: pair each
parameter name with its argument value (`dict(zip(fn.params, args))`), open a
fresh scope holding those bindings, and run the body there.  The body sees its own
parameters first and (because the new scope is chained onto `global_env`) falls
back to the globals for anything else, like `+` or the name of a function it calls.

This already runs a great deal of the language.  `square` works: `(square 5)`
binds `n = 5` in a fresh scope over the globals and evaluates `(* n n)`: `n`
local, `*` global.  Recursion works too:

```scheme
(set! factorial (lambda (n) (if (= n 0) 1 (* n (factorial (- n 1))))))
(factorial 5)     ; => 120
```

Each nested call to `factorial` gets its *own* fresh scope with its *own* `n`, so
the levels no longer trample one another, the very problem we opened the chapter
with.  And `factorial` finds *itself* by falling back to the globals, where
`(set! factorial ...)` put it.

In fact this simple version is exactly right for every function defined at the top
level, which is nearly everything you write at first.  It has just one blind
spot, and it is precisely the one from §2.2: a function defined *inside* another
function.  Closing that blind spot is the whole of the next section.


## 2.5 Closures: the one hookup

The blind spot from §2.4 is a function defined *inside* another function.  Let us
walk `make-adder` straight through the simple machine and watch exactly where it
breaks:

```scheme
(set! make-adder (lambda (n) (lambda (x) (+ n x))))
(set! add5 (make-adder 5))
(add5 3)
```

Evaluating `(make-adder 5)` opens a fresh scope (call it **A**) holding
`n = 5`, chained off the globals, and runs the body.  The body is the inner
`(lambda (x) (+ n x))`; evaluating it builds a `Function(['x'], [(+ n x)])`.  But
in the §2.4 version, that `Function` records only its parameters and body: it
keeps *no* memory of scope **A**.  It is handed back and stored as `add5`, and
scope **A** is discarded.

Now evaluate `(add5 3)`:

```
call add5 with x = 3
  open scope B = { x: 3 }, chained off global_env       (parent = global_env, per §2.4)
  evaluate the body (+ n x) in B:
    lookup x  ->  3          (found in B)
    lookup n  ->  B? no. global? no.  -->  NameError: Unbound variable: n
```

There it is.  `n = 5` lived in scope **A**, but `add5`'s call was chained off the
*globals*, and **A** was thrown away the moment `make-adder` returned.  The value
is simply not reachable.

This is the rule from §2.2, now failing in the concrete: **a function must
remember the environment it was defined in.**  The §2.4 machine breaks that rule
twice over: the `Function` forgets its defining scope, and the call chains off
the globals instead of that scope.  Fixing both is three small edits, and they
are all one idea.

**1. Give `Function` somewhere to remember its defining scope:**

```python
class Function:
    def __init__( self, params, body, env ):
        self.params = params
        self.body   = body
        self.env    = env      # the environment this lambda was born in -- its home scope
```

**2. Have `lambda` capture the current `env` as it builds the function:**

```python
elif expr[0] == 'lambda':
    params, *body = expr[1:]
    return Function(params, body, env)      # capture the defining environment
```

**3. Chain each call's new scope off the function's captured `env`, not the
globals**, the single substantive change from §2.4:

```python
    else:                                            # a user-defined function
        new_env = Environment(parent=fn.env,         # <-- was parent=global_env
                              bindings=dict(zip(fn.params, args)))
        for subExpr in fn.body[:-1]:
            lEval(subExpr, new_env)
        return lEval(fn.body[-1], new_env)
```

That is the whole of closures.  Strip away the two supporting lines and the change
is literally `parent=global_env` becoming `parent=fn.env`, chaining a call off
the scope where the function was *defined* rather than the global one.

Run `make-adder` again on the fixed machine:

```
call make-adder with n = 5
  open scope A = { n: 5 }, chained off global_env
  evaluate (lambda (x) (+ n x)):  build Function(['x'], [(+ n x)], env = A)   <-- captures A
  return it  ->  add5

call add5 with x = 3
  open scope B = { x: 3 }, chained off fn.env = A       (not the globals this time)
  evaluate (+ n x) in B:
    lookup x  ->  3     (found in B)
    lookup n  ->  B? no.  A? yes: 5      (found by stepping into the captured scope)
  (+ 5 3)  ->  8
```

The captured scope **A** is exactly what carries `n = 5` into the later call.
That is the closure, and `make-counter` works for the same reason, with the twist
that its `(set! count ...)` reaches back into the captured `let` scope and *mutates*
the binding it finds there.

**And nothing regresses.**  A function defined at the top level is evaluated in the
global scope, so *its* captured `env` simply **is** the global scope.  For `square`
and `factorial`, `parent=fn.env` and `parent=global_env` are the very same thing,
the §2.4 behaviour, unchanged.  The new version only *adds* reach: functions
defined inside other scopes now remember those scopes too.  Closures are a strict
generalization of the simple version, not a replacement for it.

One design choice is worth pausing on before we read the whole thing.  `Function`,
unlike `Environment`, has no underscores and no methods: it is just a record of the
three things a closure is, its parameters, its body, and its captured environment,
read directly as `fn.params`, `fn.body`, `fn.env`.  `Environment` earns its methods
because a lookup is real work, walking the chain scope by scope, and `lookup` and
`set` package that behaviour up out of sight.  Calling a `Function` is real work too,
and we could have hidden it the same way, behind a `__call__` method on the class.
But calling a function *is* evaluation, so we keep it in `lEval` beside every other
form, out in the open with all the other eval code, rather than tucked inside
`Function`.

### 2.5.1 The complete evaluator

Here is the whole of `lEval` for this chapter, the Chapter 1 cases and the new
ones together, the complete interpreter, and what actually runs in
`examples/IttyBittyLisp2.py`:

```python
def lEval( expr, env ):
    if expr in ('#t', '#f'):           # a boolean: return it unchanged
        return expr
    elif isinstance(expr, str):        # a name: look it up along the chain
        return env.lookup(expr)
    elif not isinstance(expr, list):   # a number (any non-list): return unchanged
        return expr

    elif expr[0] == 'set!':
        name, valExpr = expr[1:]
        val = lEval(valExpr, env)
        return env.set(name, val)

    elif expr[0] == 'if':
        condExpr, thenExpr, elseExpr = expr[1:]
        condVal = lEval(condExpr, env)
        return lEval(elseExpr if condVal == '#f' else thenExpr, env)

    elif expr[0] == 'begin':
        for subExpr in expr[1:-1]:
            lEval(subExpr, env)
        return lEval(expr[-1], env)

    elif expr[0] == 'quote':
        return expr[1]

    elif expr[0] == 'lambda':          # make a closure: capture the current env
        params, *body = expr[1:]
        return Function(params, body, env)

    elif expr[0] == 'let':             # open a fresh scope over the current env
        bindingPairs, *body = expr[1:]
        initialBindings = { name: lEval(initExpr, env) for name, initExpr in bindingPairs }
        new_env = Environment( parent=env, bindings=initialBindings )
        for subExpr in body[:-1]:
            lEval(subExpr, new_env)
        return lEval(body[-1], new_env)

    else:                              # a function call
        fn, *args = [ lEval(elt, env) for elt in expr ]
        if callable(fn):                                   # a primitive
            return fn(args)
        else:                                              # a user-defined function
            initialBindings = dict(zip(fn.params, args))
            new_env = Environment(parent=fn.env, bindings=initialBindings)
            for subExpr in fn.body[:-1]:
                lEval(subExpr, new_env)
            return lEval(fn.body[-1], new_env)
```

Read it straight down, as before, and notice how little the new power cost: three
carried-over atom cases, the four Chapter 1 special forms unchanged in spirit,
and just `lambda`, `let`, and the two-way function call.  Every hard idea in this
chapter (scopes, recursion, closures) is delivered by that handful of lines
plus the `Environment` stack behind them.


## 2.6 Running it

The pieces are in place: the `Environment` stack, the closure-aware `lEval`, and
the primitives.  Wiring them together is almost exactly Chapter 1's; only two
small things change.

**The global environment is now an `Environment`.**  In Chapter 1 it was a plain
dictionary; now it is the *base* of the stack (an `Environment` with no parent)
holding the same primitives as before:

```python
def lisp_print( args ):
    print( args[0] )
    return args[0]

def lisp_mul( args ):    # variadic product; (*) is 1
    result = 1
    for x in args:
        result *= x
    return result

globalBindings = {
    '+':     lambda args: sum( args ),
    '-':     lambda args: args[0] - args[1],
    '*':     lisp_mul,
    '=':     lambda args: '#t' if args[0] == args[1] else '#f',
    '<':     lambda args: '#t' if args[0] <  args[1] else '#f',
    'print': lisp_print,
}
global_env = Environment( bindings=globalBindings )
```

The primitives themselves are untouched, the same Python functions taking the
argument list.  Only their container changed: a root `Environment` instead of a
bare dict, so a `lookup` that walks all the way out ends at these.

**`lisp_str` learns to print a function.**  Chapter 1's version already rendered
lists and primitives; now a value can also be one of our `Function` closures,
which has no natural written form.  We show it as `#<procedure (...)>`, listing
its parameter names:

```python
def lisp_str( val ):
    if isinstance( val, list ):
        return '(' + ' '.join( lisp_str( x ) for x in val ) + ')'
    if isinstance( val, Function ):
        return '#<procedure (' + ' '.join( val.params ) + ')>'
    if callable( val ):
        return '#<primitive>'
    return str( val )
```

With the parser wired to this chapter's `lEval` (the §1.6 REPL, its import
pointed at `IttyBittyLisp2`) you can put the new power through its paces right at
the prompt, closures included:

```
lisp> (set! square (lambda (n) (* n n)))
#<procedure (n)>
lisp> (square 5)
25
lisp> (set! make-adder (lambda (n) (lambda (x) (+ n x))))
#<procedure (n)>
lisp> (set! add5 (make-adder 5))
#<procedure (x)>
lisp> (add5 3)
8
lisp> (let ((a 3) (b 4)) (+ (* a a) (* b b)))
25
lisp> (set! factorial (lambda (n) (if (= n 0) 1 (* n (factorial (- n 1))))))
#<procedure (n)>
lisp> (factorial 10)
3628800
```

The closure is right there in the middle: `(make-adder 5)` returns a
`#<procedure (x)>` (the inner function) and calling it, `(add5 3)`, returns `8`,
because that procedure (Scheme's name for functions) carried its captured `n = 5`
along with it.

The complete, runnable file is `examples/IttyBittyLisp2.py`:

```
python examples/IttyBittyLisp2.py
```

Two things are worth noticing about what we now have.  First, it is genuinely
expressive: first-class functions, recursion, and closures are the foundation of
most of what larger languages are built from, and they cost us only the
`Environment` chain plus three new `lEval` cases.  Second, it has a lurking limit.
Every call (`lEval` invoking itself) pushes a Python stack frame, and those
frames are not reclaimed until the call returns.  `(factorial 10)` is fine;
`(factorial 100000)` would run Python out of stack and crash.  Making deep
recursion safe, without a Python frame per step, is the problem the next chapter
takes up.


## 2.7 Challenges

Each of these builds on `lambda`, `let`, and the closure machinery from this
chapter.  Try them at the REPL against `IttyBittyLisp2.py`.

- **Add `let*`.**  Our `let` evaluates *all* of its init expressions in the outer
  scope, before the new scope exists, so the bindings cannot see one another.
  This fails:

  ```scheme
  (let ((x 1)
        (y (+ x 1)))     ; error: x is not visible here yet
    y)
  ```

  `let*` is the *sequential* version: each binding is evaluated in a scope that
  already holds the ones before it, so `y` can see `x`.  Implement it as a special
  form.  §2.4 pointed right at the difference (`let` behaves as it does *because*
  it evaluates the inits in the outer scope) so `let*` is what you get by
  evaluating each init in the *growing* scope instead: bind `x`, then evaluate
  `y`'s init with `x` already in place, and so on.  Try to name the one-line change
  before you write it.

- **Add a `define` shorthand.**  Writing `(set! square (lambda (n) (* n n)))`
  every time is a mouthful.  Most Lisps offer `(define (square n) (* n n))` as a
  shorthand that names a function in one step: it means *exactly* the
  `set!` + `lambda` above.  Add `define` as a special form that rewrites
  `(define (name params...) body...)` into
  `(set! name (lambda (params...) body...))` and evaluates that.  Then a question
  worth sitting with: could `define` be handled not by a new `elif` inside `lEval`,
  but by a rule that rewrites the code *before* evaluation, a **macro**?  It can,
  and that is a major idea a later chapter builds; work out why you cannot do it
  yet.

- **Add rest parameters (variadic functions).**  Sometimes a function should
  accept *any* number of arguments.  Lisp writes this with a dotted parameter list:
  `(lambda (first . rest) ...)` binds `first` to the first argument and `rest` to a
  *list* of all the others, the same idea as Python's `def f(first, *rest)`.  The
  parser already turns that `.` into its own token, so the work is all in the
  function-call path: when binding arguments, spot the `.` in the parameter list,
  bind the named parameters one-to-one, and gather whatever arguments remain into a
  list bound to the rest name.

- **Add `apply`.**  `(apply f args)` calls `f` with the *elements* of the list
  `args` as its arguments, so if `xs` is the list `(3 4)`, then `(apply + xs)` is
  `(+ 3 4)`, which is `7`.  It is the bridge between having arguments sitting *in a
  list* and *spreading them out* into a function call.  The catch: `apply` has to
  invoke `f` the way the evaluator does: for a user-defined function, that means
  opening a fresh scope off `f`'s captured environment and running its body (§2.5),
  which a plain Python primitive has no way to do.  So `apply` belongs where that
  machinery lives, as a special form (or a primitive given special access to the
  evaluator's call path), not an ordinary primitive.

