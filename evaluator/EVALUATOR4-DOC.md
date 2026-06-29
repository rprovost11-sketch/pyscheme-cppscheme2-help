# Evaluator 4 - The CEK Machine

*Continues from `EVALUATOR3-DOC`: the looping evaluator.*

This interpreter uses a **CEK machine** as its evaluator.
(See `EVALUATOR1-DOC` for the simplest recursive baseline, and
`EVALUATOR3-DOC` for the looping evaluator it builds on.)

> **Heads up:** the CEK machine is the most advanced evaluator in the series.
> If you just want to *extend* the interpreter -- new special forms, a macro
> expander, a tracing mode -- do it on `EVALUATOR3-DOC` instead; it is far
> easier to modify.  Come here when you need **first-class continuations**
> (`call/cc`, `dynamic-wind`): the explicit `K` stack is what makes them
> possible, and the looping evaluator cannot easily support them.

## What Is a CEK Machine?

A CEK machine is a formal model of computation introduced by Matthias
Felleisen and Dan Friedman in 1987 as a rigorous operational semantics for
call-by-value languages.  The name stands for its three-component state:

- **C - Control**: the current expression to evaluate, *or* a computed value
  ready to be delivered to the next waiting computation
- **E - Environment**: the current lexical scope - a mapping from variable names
  to values
- **K - Kontinuation**: an explicit stack of suspended computation frames, each
  representing "what to do with the next value that arrives"

The machine runs as a pure loop.  Each iteration is one reduction step.
The evaluator itself never calls itself recursively - not even for
sub-expressions in non-tail positions.  Instead of recursing, it pushes a
**continuation frame** onto K that will resume the suspended computation
when a value arrives, then loops.

This is the fundamental contrast between the three evaluator generations:

| Situation | Recursive (EVALUATOR1-DOC) | Looping (EVALUATOR3-DOC) | CEK (this doc) |
|---|---|---|---|
| Tail position | recurse | `expr = sub; continue` | set C and loop |
| Non-tail position | recurse | recurse | push frame; set C and loop |

The CEK machine eliminates the Python call-stack growth that non-tail
positions caused in both earlier designs.  Sub-expression depth is absorbed
by K - a heap-allocated Python list - rather than the Python call stack.

The deeper point: the CEK machine makes the **call stack a first-class data
structure**.  The K list at any moment is exactly the information that was
previously spread across implicit Python stack frames - made visible,
inspectable, and copyable.  This is what enables `call/cc`: capturing the
current continuation means copying K.

## Where Do the Frames Come From? (CPS and Defunctionalization)

The jump from the looping evaluator (Part 3) to this machine looks large: the
loop suddenly grows a stack of frame objects -- `IfFrame`, `ArgFrame`, and the
rest -- that did not exist before.  Where do they come from?  They are not
arbitrary.  Each one is a mechanical transformation of the recursive evaluator,
in two steps.  Following the derivation makes the machine feel inevitable
rather than invented.

Recall where Part 3 left off: tail positions loop, but **non-tail**
sub-expressions still recurse, and that recursion rides the Python call stack.
The Python stack, in other words, was already acting as K -- it just was not
something we could name or hold.  These two steps make it explicit.

### Step 1: Make the continuation an explicit function (CPS)

In the recursive evaluator, "what to do next" is implicit in the Python call
stack: when `lEval` returns, control resumes wherever it was called from.
*Continuation-passing style* (CPS) makes that explicit by passing an extra
argument `k` -- a function representing "what to do with the value once I have
it."  Instead of *returning* a value, `lEval` *calls* `k` with it:

```python
def lEval( expr, env, k ):           # k = the continuation: what to do with the value
    if isinstance( expr, str ):
        return k( env.lookup( expr ) )            # deliver the value to k
    if not isinstance( expr, list ):
        return k( expr )                          # atoms deliver themselves

    head = expr[0]
    if head == 'if':
        # Evaluate the test; its continuation picks a branch and continues with k.
        return lEval( expr[1], env,
                      lambda t: lEval( expr[2] if t != '#f' else expr[3], env, k ) )
    ...
```

You start the whole computation with the identity continuation,
`lEval(program, global_env, lambda v: v)`, and the final answer is whatever
gets delivered to it.  Notice the `if` case: the inner `lambda t: ...` is the
continuation *of the test*.  It captures exactly four things -- `expr[2]` (the
then-branch), `expr[3]` (the else-branch), `env`, and the outer `k`.

### Step 2: Turn those functions into data (defunctionalization)

The evaluator only ever builds a **finite number of continuation shapes**:
the if-continuation, the continuation waiting for a function value, the one
waiting for each argument, the `set!`-continuation, the `begin`-continuation,
and so on.  *Defunctionalization* replaces each shape with a small data object
that stores exactly the variables that shape's lambda captured, plus a `step`
method holding the lambda's body.  The if-continuation becomes:

```python
class IfFrame:                       # the defunctionalized if-continuation
    def __init__( self, then_expr, else_expr, env ):
        self.then_expr = then_expr   # ...the variables the lambda captured...
        self.else_expr = else_expr
        self.env       = env
    def step( self, t, K ):          # ...and the lambda's body, as a method.
        branch = self.then_expr if t != '#f' else self.else_expr
        return branch, self.env      # "evaluate branch under env, with k = the rest of K"
```

One thing is conspicuously missing: the captured outer `k`.  In CPS, `k` is
the *next* continuation, reached by nesting one lambda inside another.
Defunctionalized, that nesting becomes **the rest of the list K below this
frame**.  So the correspondence is exact:

| CPS (continuations as functions) | CEK (continuations as data) |
|---|---|
| wrap `k` in a new `lambda` | push a frame onto `K` |
| call the continuation with a value | pop the top frame, call `step(value, K)` |
| the identity continuation `lambda v: v` | the empty `K` (value is the final answer) |

That is the whole secret.  Every frame class in the sections below --
`IfFrame`, `BeginFrame`, `LetFrame`, `SetFrame`, `ArgFrame` -- is the
defunctionalized form of one continuation shape, and **K is the chain of
pending continuations made into a list**.  Because K is now ordinary data,
capturing the continuation (`call/cc`) is nothing more than copying that list.

(This direct-style -> CPS -> defunctionalized-machine derivation is general:
it is the standard way every abstract machine of this family -- CEK, SECD --
is obtained from an interpreter.  The frames are not a design you must invent;
they are a transformation you can turn a crank to produce.)

## The Value Problem

Before the machine loop can be described, one subtlety must be addressed:
in a Lisp that represents code as lists, both expressions and values can be
Python lists.  A computed result `[1, 2, 3]` is visually identical to a
function call expression `[fn, arg, arg]`.  The machine must tell them apart.

The solution is a `Val` wrapper:

```python
class Val:
    def __init__( self, v ):
        self.v = v
```

Any computed value is wrapped in `Val` before being placed in C.  A predicate
`is_value(C)` treats `Val` instances - and self-evaluating atoms like numbers
and booleans - as values ready for delivery.  Any unwrapped list is treated as
code to evaluate (an empty-list *value* reaches here already `Val`-wrapped):

```python
def is_value( c ):
    if isinstance( c, Val ):
        return True
    if c in ( '#t', '#f' ):                     # boolean literals are self-evaluating data
        return True
    if isinstance( c, str ):                    # symbol - needs lookup
        return False
    if isinstance( c, list ):                   # any unwrapped list is code
        return False
    return True                                  # number, etc.
```

Every path in the machine that produces a value wraps it in `Val`.  Every
path that produces an expression to evaluate leaves it unwrapped.  This
discipline is what enables TCO: a user-function body returned without `Val`
is treated as code in the next iteration, not as a value to deliver.

The looping evaluator in EVALUATOR3-DOC did not need this distinction because
its C register always held code and values lived in local variables.  In the
CEK machine, C holds both - the wrapper is the only way to tell which.

## The Machine Loop

The main loop has three cases for C:

```python
def lEval( expr, env ):
    C = expr
    E = env
    K = []   # continuation stack

    while True:

        # Case 1: C is a value - deliver it to the top continuation frame.
        if is_value( C ):
            v = C.v if isinstance( C, Val ) else C
            if not K:
                return v                     # K empty: computation finished
            frame = K.pop()
            C, E  = frame.step( v, K )
            continue

        # Case 2: C is a symbol - look it up, Val-wrap the result.
        if isinstance( C, str ):
            C = Val( E.lookup( C ) )
            continue

        # Case 3: C is a non-empty list - an expression to reduce.  (A bare ()
        # is invalid Scheme; an empty-list value arrives Val-wrapped via Case 1.)
        head = C[0]
        ...
```

When K is empty and a value arrives in Case 1, the entire computation is
finished and the value is returned.  Otherwise, the top frame is popped and
handed the value; it returns the next `(C, E)` pair for the loop.

Notice that Case 1 both consumes values **and** resumes the next computation.
There is no separate "return" path - every value flows through the same
delivery mechanism.  When K empties, delivery becomes the final return.

## Step-by-Step: Six Evaluation Paths

The best way to understand the machine is to watch it run.  Each block shows
the complete machine state at the start of that loop iteration.  E is shown
only when it changes.

---

### Example 1 - Constant: `42`

```
C: 42
E: {}
K: []
```

**Step 1** - C is a number.  `is_value` returns True.  K is empty: return `42`.

**Result: 42**

---

### Example 2 - Variable lookup: `x` where `x = 7`

```
C: x
E: {x: 7}
K: []
```

**Step 1** - Case 2: C is a symbol.  Look up `x` in E, get `7`, wrap in Val.

```
C: Val(7)
K: []
```

**Step 2** - Case 1: C is a value.  K is empty: return `7`.

**Result: 7**

---

### Example 3 - Primitive call: `(+ x 1)` where `x = 2`

```
C: (+ x 1)
E: {x: 2}
K: []
```

**Step 1** - Case 3: non-empty list, head is not a special form.  Push `ArgFrame`,
set `C = +`.

```
C: +
K: [ArgFrame(fn=None, pending=[x, 1], done=[])]
```

**Step 2** - Case 2: symbol.  Look up `+`, wrap: `C = Val(fn_+)`.

```
C: Val(fn_+)
K: [ArgFrame(fn=None, pending=[x, 1], done=[])]
```

**Step 3** - Case 1: value.  Pop `ArgFrame`.  **Phase 1**: record `fn = fn_+`,
take `x` from pending, push self back, return `C = x`.

```
C: x
K: [ArgFrame(fn=fn_+, pending=[1], done=[])]
```

**Step 4** - Case 2: symbol.  Look up `x`, get `2`, wrap: `C = Val(2)`.

```
C: Val(2)
K: [ArgFrame(fn=fn_+, pending=[1], done=[])]
```

**Step 5** - Case 1: value.  Pop `ArgFrame`.  **Phase 2**: append `2` to done,
take `1` from pending, push self back, return `C = 1`.

```
C: 1
K: [ArgFrame(fn=fn_+, pending=[], done=[2])]
```

**Step 6** - Case 1: `1` is a self-evaluating number.  Pop `ArgFrame`.
Phase 2: `done = [2, 1]`, pending empty.  Call `do_apply(fn_+, [2, 1])`,
return `Val(3)`.

```
C: Val(3)
K: []
```

**Step 7** - Case 1: value.  K is empty: return `3`.

**Result: 3**

---

### Example 4 - Nested calls: `(+ (* 2 3) 1)` - K grows to depth 2

The outer `+` call cannot proceed until `(* 2 3)` is resolved.  Both calls
push `ArgFrame` instances, stacking K to depth 2 before any unwinding begins.

```
C: (+ (* 2 3) 1)
E: global
K: []
```

**Step 1** - Case 3: push `ArgFrame_outer`, set `C = +`.

```
C: +
K: [ArgFrame_outer(fn=None, pending=[(* 2 3), 1], done=[])]
```

**Step 2** - Symbol `+` -> `Val(fn_+)`.  Pop `ArgFrame_outer`.  Phase 1:
`fn = fn_+`, next pending is `(* 2 3)`, push self back, return `C = (* 2 3)`.

```
C: (* 2 3)
K: [ArgFrame_outer(fn=fn_+, pending=[1], done=[])]
```

**Step 3** - Case 3: `(* 2 3)` is a function call.  Push `ArgFrame_inner`.
**K is now at depth 2.**

```
C: *
K: [ArgFrame_outer(fn=fn_+,  pending=[1],    done=[]),
    ArgFrame_inner(fn=None,   pending=[2, 3], done=[])]
```

**Steps 4-7** - Evaluate `*`, `2`, and `3` the same way example 3 showed.
After `3` is delivered: `done = [2, 3]`, pending empty.
`do_apply(fn_*, [2, 3])` returns `Val(6)`.  `ArgFrame_inner` is consumed.
**K shrinks back to depth 1.**

```
C: Val(6)
K: [ArgFrame_outer(fn=fn_+, pending=[1], done=[])]
```

**Step 8** - Pop `ArgFrame_outer`.  Phase 2: `done = [6]`, take `1` from
pending, push self back, return `C = 1`.

```
C: 1
K: [ArgFrame_outer(fn=fn_+, pending=[], done=[6])]
```

**Step 9** - `1` is self-evaluating.  Pop `ArgFrame_outer`.  `done = [6, 1]`,
pending empty.  `do_apply(fn_+, [6, 1])` -> `Val(7)`.

```
C: Val(7)
K: []
```

**Step 10** - K empty: return `7`.

**Result: 7**

Nested calls require no special handling - they fall out naturally.  Each
pending argument that is itself a call pushes its own `ArgFrame` on top of
the existing stack.  K depth equals the current expression nesting depth.

---

### Example 5 - Branching: `(if (= x 0) 42 99)` where `x = 0`

An `if` pushes an `IfFrame` to remember the two branches, then sets C to the
condition expression.  The condition is itself a function call, so K holds
two different frame types at the same time.

```
C: (if (= x 0) 42 99)
E: {x: 0}
K: []
```

**Step 1** - Case 3, `head = if`.  Push `IfFrame(then=42, else=99)`, set `C`
to the condition.

```
C: (= x 0)
K: [IfFrame(then=42, else=99)]
```

**Step 2** - Case 3: `(= x 0)` is a function call.  Push `ArgFrame`.
K now holds **two different frame types**.

```
C: =
K: [IfFrame(then=42, else=99),
    ArgFrame(fn=None, pending=[x, 0], done=[])]
```

**Steps 3-6** - Evaluate `=`, `x`, and `0` exactly as in example 3.
`do_apply(fn_=, [0, 0])` returns `Val(#t)`.  `ArgFrame` is consumed.

```
C: Val(#t)
K: [IfFrame(then=42, else=99)]
```

**Step 7** - Case 1: pop `IfFrame`.  `#t` is not `#f`, so the then-branch is
taken: return `(42, E)` as an
unwrapped expression.  **No frame is pushed** - both branches are in tail
position inside `if`.

```
C: 42
K: []
```

**Step 8** - `42` is self-evaluating.  K empty: return `42`.

**Result: 42**

`IfFrame.step` never pushes anything onto K.  It simply returns one branch
as the next expression.  Whatever computation surrounds the `if` - or nothing,
if K was already empty - is unaffected.

---

### Example 6 - User-defined function and TCO: `((lambda (x) (* x 2)) 5)`

A user-defined function call does not execute immediately.  `do_apply` opens
a new environment and returns the body expression **without wrapping it in
`Val`**.  The loop sees an unwrapped expression and evaluates it as code.
No frame was pushed for "what to do with the return value."  This is TCO.

```
C: ((lambda (x) (* x 2)) 5)
E: global
K: []
```

**Step 1** - Case 3: function call.  Push `ArgFrame`, set `C` to the head of
the list - the lambda form itself.

```
C: (lambda (x) (* x 2))
K: [ArgFrame(fn=None, pending=[5], done=[])]
```

**Step 2** - Case 3, `head = lambda`.  Construct
`Function(params=[x], body=[(* x 2)], closure_env=global)`,
wrap: `C = Val(fn_user)`.  No frame pushed.

```
C: Val(fn_user)
K: [ArgFrame(fn=None, pending=[5], done=[])]
```

**Step 3** - Case 1: pop `ArgFrame`.  Phase 1: `fn = fn_user`, take `5` from
pending, push self back, return `C = 5`.

```
C: 5
K: [ArgFrame(fn=fn_user, pending=[], done=[])]
```

**Step 4** - `5` is self-evaluating.  Pop `ArgFrame`.  `done = [5]`, pending
empty.  Call `do_apply(fn_user, [5])`.

`do_apply` sees a user-defined function: open `new_env = {x: 5} -> global`,
call `_begin_body([(* x 2)], new_env, K)`.

`_begin_body` returns `(* x 2)` **without a `Val` wrapper**.

```
C: (* x 2)           <- unwrapped body; next iteration treats it as code
E: {x: 5} -> global
K: []                <- ArgFrame was popped and NOT replaced
```

This is the TCO moment.  The frame that triggered the call has been consumed.
K is empty.  The Python call stack has not grown.  The body evaluates exactly
as if it had been written at the top level.

**Steps 5-9** - `(* x 2)` evaluates exactly as in example 3:
`*` -> `fn_*`, `x` -> `Val(5)`, `2` is self-eval.
`do_apply(fn_*, [5, 2])` -> `Val(10)`.

```
C: Val(10)
K: []
```

**Step 10** - K empty: return `10`.

**Result: 10**

If this function called itself in tail position - `(f (- n 1))` as the last
form in a body - the same thing would happen on every recursive call: the
caller's `ArgFrame` is consumed by `do_apply`, the recursive call's `ArgFrame`
is the only frame ever on K, and K never grows past depth 1 regardless of
how many iterations the recursion runs.

## K Is the Explicit Call Stack

The K list at any moment is exactly the information that a recursive evaluator
would have spread across implicit Python stack frames.

Consider how the recursive evaluator from EVALUATOR1-DOC would handle `(+ x 1)`:

```
Python frame 1: evaluating '+' - "I need the function value for this call"
Python frame 2: evaluating 'x' - "I need argument 0 for this call"
Python frame 3: evaluating 1   - "I need argument 1 for this call"
```

In the CEK machine, that same information lives in a single `ArgFrame` object,
mutated across three iterations:

| Loop iteration | ArgFrame state |
|---|---|
| Evaluating `'+'` | `fn=None, pending=['x', 1], done=[]` |
| Evaluating `'x'` | `fn=fn_+, pending=[1], done=[]` |
| Evaluating `1` | `fn=fn_+, pending=[], done=[2]` |

Each iteration that would have pushed a Python stack frame instead updates the
ArgFrame in place and loops.  The frame is not popped until all its work is
done and `do_apply` is called.

For nested calls like `(f (g (h 1)))`, three separate `ArgFrame` instances
accumulate on K - one per call - just as three levels of Python recursion
would accumulate in the other evaluators.  The difference is that K is a
heap list: it can grow to any depth without a `RecursionError`.

**Tail calls** are the special case where K does not grow.  When a
user-defined function's body is set as C and `_begin_body` returns, the
`ArgFrame` that triggered the call has already been popped and discarded.
Nothing new is pushed.  K stays the same size.

## Continuation Frames in Depth

### IfFrame

The simplest frame.  Pushed when `(if cond then else)` is seen; receives
the condition value and picks a branch:

```python
class IfFrame:
    def __init__( self, then_expr, else_expr, env ):
        self.then_expr = then_expr
        self.else_expr = else_expr
        self.env       = env

    def step( self, value, K ):
        branch = self.then_expr if value != '#f' else self.else_expr
        return branch, self.env   # expression - NOT Val-wrapped
```

The chosen branch is returned unwrapped.  Neither branch pushes a frame -
both are in tail position.

### BeginFrame

Sequences a list of forms, discarding all results except the last.  The
last form is delivered in tail position by *not* pushing the frame back:

```python
class BeginFrame:
    def __init__( self, remaining, env ):
        self.remaining = remaining
        self.env       = env

    def step( self, value, K ):
        if len( self.remaining ) == 1:
            return self.remaining[0], self.env   # tail - no re-push
        nxt            = self.remaining[0]
        self.remaining = self.remaining[1:]
        K.append( self )                          # more forms remain
        return nxt, self.env
```

For `(begin a b c)`, the machine pushes `BeginFrame([b, c])` and evaluates
`a`.  The frame receives `a`'s value (discards it), shrinks to `[c]`, pushes
itself, evaluates `b`.  The frame receives `b`'s value, sees `remaining = [c]`
has length 1, returns `c` without re-pushing - `c` is the tail.

### LetFrame

The `let` form binds multiple variables, but all init expressions must be
evaluated in the **outer** environment before any binding takes effect.
`LetFrame` enforces this by holding a reference to `outer_env` and using it
for every init evaluation, only opening the new scope once all values are
collected:

```python
class LetFrame:
    def step( self, value, K ):
        self.bound[self.current_name] = value
        if self.pending:
            name, form        = self.pending[0]
            self.current_name = name
            self.pending      = self.pending[1:]
            K.append( self )
            return form, self.outer_env      # always the outer env
        new_env = Env( parent=self.outer_env, bindings=self.bound )
        return _begin_body( self.body, new_env, K )
```

This is exactly what distinguishes `let` from `let*`: a `LetStarFrame` would
bind each name immediately and pass the growing inner environment to the next
init expression instead of `outer_env`.

### ArgFrame

The most complex frame.  A single `ArgFrame` manages all phases of one
function call - evaluating the function, evaluating each argument in order,
and finally dispatching:

```python
class ArgFrame:
    def step( self, value, K ):
        if self.fn is None:
            # Phase 1: received the function.
            self.fn = value
            if not self.pending:
                return do_apply( self.fn, self.done, self.env, K )
            nxt          = self.pending[0]
            self.pending = self.pending[1:]
            K.append( self )
            return nxt, self.env

        # Phase 2: received an argument value.
        self.done.append( value )
        if self.pending:
            nxt          = self.pending[0]
            self.pending = self.pending[1:]
            K.append( self )
            return nxt, self.env

        return do_apply( self.fn, self.done, self.env, K )
```

The frame is pushed once per call and re-pushed up to `1 + len(args)` times
before finally calling `do_apply`.  Each re-push corresponds to one sub-
expression that would have been a recursive `lEval` call in the earlier
evaluators.

## Function Calls and TCO

When all argument values are in hand, `ArgFrame` calls `do_apply`:

```python
def do_apply( fn, args, env, K ):
    if callable( fn ):
        return Val( fn( args ) ), env    # primitive: Val-wrap the result

    # User-defined function: open a new scope and begin the body.
    new_env = Env( parent=fn.env, bindings=dict( zip( fn.params, args ) ) )
    return _begin_body( fn.body, new_env, K )

def _begin_body( body, env, K ):
    if not body:
        return Val( [] ), env
    if len( body ) > 1:
        K.append( BeginFrame( list( body[1:] ), env ) )
    return body[0], env              # first body form - NOT Val-wrapped
```

The critical line is the last one.  The first body form is returned
**without** a `Val` wrapper.  On the next loop iteration `is_value` is
False, so the machine evaluates it as code.  No new Python stack frame was
created anywhere in this chain - `do_apply` and `_begin_body` both return
immediately to the main loop.

For a **tail call** - a function call in the final position of a body:

1. `ArgFrame` was the only frame pushed for this call.
2. It delivers args to `do_apply`, then is discarded.
3. `do_apply` returns an unwrapped expression.
4. The main loop evaluates that expression with K unchanged (possibly empty).

K does not grow.  Python call stack depth stays constant.

For a **non-tail call** - a function call whose result feeds into further
computation (e.g. an argument to an outer call):

1. The outer `ArgFrame` is on K, waiting for this sub-result.
2. `do_apply` sets C to the callee's body; K still has the outer frame on it.
3. K has grown by whatever frames the callee's body evaluation requires.

K depth grows with nesting depth, just as the Python call stack would in the
earlier evaluators.  The difference is that K lives on the heap - there is
no `RecursionError` limit.

## The Complete Evaluator

```python
def lEval( expr, env ):
    C = expr
    E = env
    K = []

    while True:

        if is_value( C ):
            v = C.v if isinstance( C, Val ) else C
            if not K:
                return v
            frame = K.pop()
            C, E  = frame.step( v, K )
            continue

        if isinstance( C, str ):
            C = Val( E.lookup( C ) )
            continue

        head = C[0]

        if head == 'if':
            then_ = C[2] if len( C ) > 2 else []
            else_ = C[3] if len( C ) > 3 else []
            K.append( IfFrame( then_, else_, E ) )
            C = C[1]
            continue

        if head == 'begin':
            if len( C ) <= 1:
                C = Val( [] )
                continue
            if len( C ) == 2:
                C = C[1]
                continue
            K.append( BeginFrame( list( C[2:] ), E ) )
            C = C[1]
            continue

        if head == 'let':
            vardefs = C[1]
            body    = list( C[2:] )
            if not vardefs:
                C, E = _begin_body( body, Env( parent=E ), K )
                continue
            pairs             = [(vd[0], vd[1]) for vd in vardefs]
            first_name, first = pairs[0]
            K.append( LetFrame( first_name, pairs[1:], {}, body, E ) )
            C = first
            continue

        if head == 'set!':
            K.append( SetFrame( C[1], E ) )
            C = C[2]
            continue

        if head == 'lambda':
            C = Val( Function( C[1], list( C[2:] ), E ) )
            continue

        if head == 'quote':
            C = Val( C[1] )
            continue

        # Function call: push ArgFrame and reduce the function position first.
        K.append( ArgFrame( None, list( C[1:] ), [], E ) )
        C = C[0]
        continue
```

Frame class definitions (`IfFrame`, `BeginFrame`, `LetFrame`, `SetFrame`,
`ArgFrame`) and the supporting `Env`, `Function`, `Val`, `is_value`,
`do_apply`, and `_begin_body` are shown in full in the example file.

## Running the Example

The complete working code is in `examples/IttyBittyLisp4.py`.
It runs the same countdown-from-100,000 as `IttyBittyLisp3.py`.  The
difference is invisible in the output but real in the machinery: in
IttyBittyLisp3 evaluating `(= n 0)` and `(- n 1)` each recurse into
`lEval`; here those evaluations push `ArgFrame` instances onto K and loop.
At no point during the countdown does the Python call stack grow beyond a
constant handful of frames.

```
python examples/IttyBittyLisp4.py
```

The real interpreter's `Evaluator.py` extends this design with tracing, macros,
multiple values, continuations, and the full lambda-list argument binding.
Because K is an explicit Python list, capturing the entire continuation
at any point - the basis of `call/cc` - requires nothing more than
copying K.

## Challenges

*(Extending the language -- macros, tracing, new special forms -- is easier on
the looping evaluator; see EVALUATOR3-DOC's challenges.  These two belong here
because they genuinely need the CEK machine's explicit `K` stack.)*

- **Implement `call/cc`.** `(call/cc (lambda (k) ...))` captures the current
  continuation -- the entire K stack -- and passes it as an argument.
  Calling `k` with a value discards the current K and reinstates the
  captured one.  Start with escape continuations (calling `k` only from
  within the dynamic extent of the `call/cc`): capturing is `list(K)` and
  reinstating is `K[:] = captured`.  For **full re-invocable continuations**,
  each mutable frame must also be copied so that re-invocation cannot corrupt
  the saved state; add a `copy()` method to every frame class (mutable frames
  return a new instance; immutable frames return `self`), then capture with
  `[f.copy() for f in K]`.  This interpreter uses this full approach.

- **Implement `dynamic-wind`.** `(dynamic-wind before thunk after)` guarantees
  that `after` runs whenever control leaves the thunk -- whether by normal
  return, exception, or continuation jump.  It requires tracking "winders"
  alongside K and running the appropriate before/after thunks as
  continuations cross their boundaries.  This is a significant challenge
  but it reveals exactly why continuations and `dynamic-wind` interact the
  way they do.
