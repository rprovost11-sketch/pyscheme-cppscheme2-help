# Chapter 4: The Continuation Becomes a Register

Chapter 3 left one honest gap.  The looping evaluator gave *tail* calls a home in
constant space, but non-tail sub-expressions still recurse (they still call
`lEval` and wait for the answer to come back) and each of those waits sits on
Python's stack.  We said what that stack is quietly holding for us: for every
unfinished sub-expression, *what to do with its value once it comes back*.  That
pending "what-to-do-next" is the **continuation**, and right now it lives on
Python's call stack where we can neither see it nor control it.

This chapter takes it away from Python and puts it in a register the machine owns
outright, a third register alongside `C` and `E`.  When we are done, the
evaluator will never call itself, not even for a non-tail sub-expression: *all* of
its pending work, tail and non-tail alike, will live in that register, on the
heap, where nothing Python does can overflow it.

That is a real change to the shape of the machine, and it is the one genuinely
hard step in this series.  So we take it slowly, and we take it on the smallest
language we can: small enough that the machine is the only new thing in the room.
The full language comes back in Chapter 5, running on this very machine; the deep
`factorial` that overflowed at the end of Chapter 3 will run there in flat space.
This chapter is about building the machine itself.


## 4.1 The language shrinks, and why

For this one chapter we put most of the language down.  Here is everything Chapter
4's Lisp has:

- **numbers**, like `42`, which evaluate to themselves;
- **variables**, like `x`, looked up in the environment;
- **`if`**, with the rule *a number is true unless it is `0`*, so `0` is false and
  every other number (and every function) is true;
- **`lambda`**, a function of exactly **one** parameter, written without parentheses
  around it (`(lambda x x)` is the identity function) with a body of exactly
  **one** expression;
- **function calls** of one argument, `(f 3)`.

That is the whole language.  No `let`, no `set!`, no `begin`, no `+`/`-`/`=`, no
`#t`/`#f`: just functions, variables, and `if`.  Two of these are deliberate
step-downs from Chapter 2's `lambda`, which took a *list* of parameters and a body
of several expressions:

- **one parameter, written bare.**  `(lambda x x)` instead of `(lambda (x) x)`.
  We lose nothing real by it: a function of two arguments is a function of the
  first that *returns* a function of the second, and you call it one argument at a
  time.  You already saw this in Chapter 2's `make-adder`.  Here it is directly:

  ```scheme
  (((lambda x (lambda y x)) 3) 9)
  ```

  The outer function takes `x` and returns `(lambda y x)`, a function that ignores
  its own argument and hands back `x`.  Apply it to `3` and you get "the function
  that always returns `3`"; apply *that* to `9` and you get `3`.  One argument at a
  time recovers as many arguments as you like, which is why a one-parameter
  `lambda` is not a real restriction.

- **a one-expression body.**  No implicit `begin`, because with no `set!` and no
  primitives there are no side effects to sequence: every expression here is
  computed purely for its value, so a body of one expression is all the language
  can use.

You might reasonably wonder whether a language this bare (no arithmetic, no data,
nothing but one-argument functions and a test) can compute anything worth
computing.  It can, and that is one of the quiet surprises at the foundation of
computing: functions and `if` are already enough to express *any* computation at
all.  Numbers, arithmetic, loops, data structures (everything the full language
hands back in Chapter 5) can be rebuilt from nothing but functions, given enough
ingenuity.  You have met the trick in miniature already: Chapter 2 conjured a
stateful counter out of a bare closure, and the curried functions above recovered
two arguments from one; the same kind of construction, pushed far enough, reaches an
entire language.  (This minimal core is older than the electronic computer and has a
name, the **lambda calculus**, whose central result is exactly this: that functions
alone are universal.)  So the smallness takes nothing away from what the language can
compute; it is a free choice, which leaves the question of why we make it.

Why shrink so far?  Because the machine we are about to build treats every kind of
sub-expression the same way ("evaluate this later, and here is what to do with its
value") and each *kind* of sub-expression needs its own little note describing
that "what to do."  A language with ten forms needs ten kinds of note; a language
with three needs three.  We want you watching the *mechanism*, not bookkeeping ten
near-identical cases, so we cut the language to the fewest forms that still have
everything interesting: functions, so there is something to call; `if`, so there
is a choice to make; and a notion of truth, so `if` has something to test.  That
is the smallest world in which the new machine still does something worth watching.

Everything you learn here scales straight back up.  Adding `let`, `begin`,
primitives, and the rest to this machine adds *more kinds of note* and nothing
else: no change to the machine's shape.  That is exactly the work of Chapter 5.


## 4.2 The continuation, made of data

You met the **continuation** at the end of Chapter 3 (§3.4): for every unfinished
sub-expression, the continuation is everything the program still has left to do once
that sub-expression produces a value: choose a branch with it, multiply it by `n`,
make it the next argument.  Every non-tail sub-expression has one, and in Chapters
1–3 it was never something you could point at.  It was *implied* by where the
evaluator happened to be in its own recursion, living on Python's stack instead of
in ours.

This chapter's one move is to make the continuation **data**.  We **reify** it.  To
*reify* something is to take an idea that is implicit (genuinely present and doing
work, but with no handle on it) and turn it into a concrete piece of data your
program can hold, inspect, and pass around.  Each pending "what-to-do-next" becomes
a small tuple, a **continuation frame**, and we keep a list of those frames in a
register of our own.  The list *is* the continuation, made of data instead of made
of Python's paused function calls.

Once the continuation is data, the recursion that used to hold it is unnecessary.
That is the whole idea of the chapter.  The rest is watching it happen, one form at
a time.


## 4.3 One recursion becomes a stack

Start where the recursion actually is.  Here is Chapter 3's `if`, unchanged:

```python
elif C[0] == 'if':
    condExpr, thenExpr, elseExpr = C[1:]
    condVal = lEval(condExpr, E)                     # condition: not tail -> recurse
    C = elseExpr if condVal == '#f' else thenExpr    # chosen branch: tail -> loop
    continue
```

Look at what that recursive call is doing.  It has two halves fused together:
*evaluate `condExpr`*, and, once that returns, *use the result to choose a
branch*.  The recursion fuses them because the second half is literally the code
sitting on the line below, waiting.

We are going to pull them apart.  Instead of calling `lEval` and waiting, we will:

1. write down the second half ("when a value comes back, choose `thenExpr` or
   `elseExpr`") as a piece of data, and set it aside;
2. then loop to evaluate `condExpr`, exactly the way the looping evaluator already
   loops, by pointing `C` at it;
3. and when `condExpr` finally produces a value, go find the note we set aside and
   carry it out.

Step 1 needs somewhere to put the note.  That is the new register: **`K`**, the
**Kontinuation** (spelled with a K so the three registers read `C`, `E`, `K`).  It
is a list used as a stack: we *push* a note when we set work aside, and *pop* the
most recent note when a value is ready for it.  A note is just a tuple, tagged so
we can tell one kind from another; the `if` note carries the two branches and the
environment to run them in:

```python
FRAME_IF = 0                          # tag: "a value is coming; choose a branch with it"

elif C[0] == 'if':
    K.append( (FRAME_IF, C[2], C[3], E) )   # set the second half aside...
    C = C[1]                                # ...and loop to evaluate the condition
```

No recursion, no waiting.  We pushed the pending work onto `K` and pointed `C` at
the condition, and the loop goes round again.

Step 3 needs somewhere for the value to *land* when a sub-expression finally
produces one, and it needs to happen when `C` reaches a leaf: a number or a
variable, something that yields a value without any further sub-expressions.  The
landing place is one more register, **`V`**, the current value:

```python
if isinstance( C, int ):        # a number is its own value
    V = C
elif isinstance( C, str ):      # a variable: look it up
    V = E.lookup( C )
```

The instant a leaf sets `V`, there is a value in hand and a stack of notes waiting
for one.  So we turn to `K`: pop the top note and carry it out.  For an `if` note,
carrying it out means choosing a branch by the truth of `V` and pointing `C` at it,
which is exactly the second half we set aside:

```python
frame = K.pop()
if frame[0] == FRAME_IF:                 # (FRAME_IF, thenExpr, elseExpr, env)
    C = frame[2] if V == 0 else frame[1] # 0 is false -> else, otherwise -> then
    E = frame[3]
    # loop again to evaluate the chosen branch
```

Put those three fragments in one loop and you have a complete little machine, for
a language of just numbers, variables, and `if`.  Walk `(if 0 100 200)` through it:

```
 step   C  (or action)            K (stack of notes)          V     what happens
 ----   ---------------------     ------------------------    ---   --------------------------
   1    (if 0 100 200)            []                           –    push note, C := 0
   2    0                         [(FRAME_IF, 100, 200, E)]    –    leaf: V := 0
   3    (consult K)               [(FRAME_IF, 100, 200, E)]    0    pop note: 0 is false, C := 200
   4    200                       []                           0    leaf: V := 200
   5    (consult K)               []                          200   K empty -> return 200
```

The note that used to be "the Python line waiting below the recursive call" is now
a tuple sitting on `K`.  When the condition's value arrived in `V`, we did not
*return* to a waiting caller (there is no waiting caller).  We *looked up* the
waiting work on `K` and did it.  That is the continuation, reified: a value on the
heap instead of a paused function on Python's stack.

We built this for one form, `if`.  Functions need two more notes, and we add them
in §4.5.  But the machine's whole shape is already here, and it is worth naming
before we go on.


## 4.4 The two states: EVAL and APPLY

Look back at the walk-through and you will see the machine was always doing one of
exactly two things:

- **going down**: taking the current expression `C` apart, pushing a note for each
  piece that has to be evaluated first, and pointing `C` at that piece.  Steps 1
  and 3→4 are this: descend until we reach a leaf that produces a value into `V`.
- **coming back up**: taking the value in `V`, popping the top note off `K`, and
  doing what the note says, which either finishes the program (when `K` is empty)
  or sets up the next `C` to go down into.  Steps 2→3 and 5 are this.

These two modes have names.  Going down is **EVAL**: given an expression, break it
toward a value.  Coming back up is **APPLY**: given a value and a stack of pending
notes, feed the value to the next note.  The machine simply alternates between
them (EVAL until a leaf hands a value to APPLY, APPLY until it hands a new
expression back to EVAL) and that back-and-forth *is* the evaluator now.  When
EVAL reaches a leaf it switches to APPLY; when APPLY sets up a fresh `C` it
switches back to EVAL; when APPLY finds `K` empty, the program is done and `V` is
the answer.

Because the value always travels in its own register `V`, the register `C` only
ever holds *code*: an expression on its way down through EVAL.  We never have to
ask "is `C` a value or an expression?"; the two never share a slot.  That is why
the machine reads cleanly as two loops, one per state, which is exactly how we will
write it.

So the machine has three registers of live state and one for the value in transit:

| register | holds | in the CPU-register sense |
|---|---|---|
| `C` | the expression being evaluated | Control |
| `E` | the current environment | Environment |
| `K` | the stack of pending notes | Kontinuation |
| `V` | the value coming back | the value in transit |

`C` and `E` you have had since Chapter 3.  `K` is the continuation, now a register
we own.  `V` is just the wire the value travels back on.  That is the CEK machine,
named for its three registers, and everything left in this chapter is filling in
the notes that functions need and then watching the whole thing run.


## 4.5 Functions: two more notes

`if` needed one kind of note because it has one sub-expression to evaluate before
it can act: the condition.  A function call has *two*: the function and the
argument.  Both must be evaluated, in order, before the call can happen, so a call
needs two notes, set aside one after the other.

First, though, the function itself.  A `lambda` produces a value the same way a
number does (immediately, with nothing to evaluate first), so in EVAL it is a
leaf.  Its value is a **closure**: the parameter, the body, and *the environment it
was written in* (this is Chapter 2's closure, captured exactly as before, now
carried as a tagged tuple instead of a `Function` object).  We tag it `VAL_CLOSURE`
so APPLY can tell a closure from a plain number:

```python
elif C[0] == 'lambda':               # (lambda param body) -> a closure value
    V = ( VAL_CLOSURE, C[1], C[2], E )   # (tag, param, body, captured-env)
```

Notice a closure is a tagged tuple of plain data, just like a continuation frame.
The whole machine is built from tagged tuples: some describe pending work, one
describes a captured function, and APPLY tells them apart by their tag.

Now the call, `(f 3)`.  We evaluate the function first.  So, exactly the move from
§4.3, we set aside a note saying "*when the function's value comes back, go
evaluate the argument*," and point `C` at the function.  The note has to carry the
argument expression (still unevaluated) and the environment to evaluate it in.  We
tag it `FRAME_ARG`, because the pending job it records is *evaluate the argument*:

```python
FRAME_ARG = 1                        # tag: "function value coming; then evaluate the arg"

else:                                # (fn arg) -- a function call
    K.append( (FRAME_ARG, C[1], E) )     # set the argument aside...
    C = C[0]                             # ...and loop to evaluate the function
```

The function descends through EVAL and eventually lands a closure in `V`.  Now APPLY
pops the `FRAME_ARG` note.  Its job is to launch the argument, but first it must
remember the function value we just got, or it will be lost when the argument
descends.  So it sets aside a *second* note carrying the closure, tagged
`FRAME_CALL` because its pending job is *make the call*, and points `C` at the
argument:

```python
FRAME_CALL = 2                       # tag: "argument value coming; then make the call"

elif ftag == FRAME_ARG:              # (FRAME_ARG, arg, env); V is the function value
    K.append( (FRAME_CALL, V) )          # remember the function...
    C = frame[1]                         # ...and loop to evaluate the argument
    E = frame[2]
```

The argument now descends through EVAL and lands its value in `V`.  APPLY pops the
`FRAME_CALL` note, and at last everything is in hand: the closure (in the note) and
the argument value (in `V`).  Making the call is what the looping evaluator already
did in Chapter 3: open a new scope on the closure's captured environment, bind the
parameter to the argument, and evaluate the body, except we do it by pointing the
registers, never by recursing:

```python
elif ftag == FRAME_CALL:             # (FRAME_CALL, closure); V is the argument value
    _, param, body, clo_env = frame[1]
    E = Environment( parent=clo_env, bindings={ param: V } )
    C = body                             # loop to evaluate the body
```

And here is the quiet, important thing: **making a call pushes no note.**  `FRAME_IF`
and `FRAME_ARG` and `FRAME_CALL` all get pushed while we descend into a
sub-expression whose value we still need, but the body of a called function is not
a sub-expression whose value we need *and then do more with*; its value *is* the
value of the call.  So we install the body directly into `C` with nothing pushed.
We will come back to why that one omission is the whole of tail-call optimization,
in §4.8.

Walk `((lambda x x) 7)`, the identity function applied to `7`, through it:

```
 step   C  (or action)         K (stack of notes)              V         what happens
 ----   -------------------    ----------------------------    -------    --------------------------
   1    ((lambda x x) 7)       []                               –        call: push FRAME_ARG, C := (lambda x x)
   2    (lambda x x)           [(ARG, 7, top)]                  –        lambda leaf: V := #<x>
   3    (consult K)            [(ARG, 7, top)]                  #<x>     pop ARG: push FRAME_CALL, C := 7
   4    7                      [(CALL, #<x>)]                   #<x>     leaf: V := 7
   5    (consult K)            [(CALL, #<x>)]                   7        pop CALL: bind x:=7, C := x (body)
   6    x                      []                               7        leaf: V := lookup(x) = 7
   7    (consult K)            []                               7        K empty -> return 7
```

(`#<x>` is the closure the `lambda` produced: parameter `x`, body `x`.)  The two
notes appear in order (`FRAME_ARG` while the function is evaluated, `FRAME_CALL`
while the argument is) and each is popped the instant its value arrives.  That is
every kind of note the language has: one for `if`, two for a call.


## 4.6 The complete machine

Three frame kinds, one closure kind, four registers, two states.  Here is the whole
evaluator, exactly as it stands in `examples/IttyBittyLisp4.py`.  The `Environment`
class is unchanged from Chapters 2 and 3 (the same linked chain of scopes with
`lookup` and `set`), so it is not reprinted here; the file has it.

```python
VAL_CLOSURE = 1        # tag for a closure value: (VAL_CLOSURE, param, body, env)

FRAME_IF   = 0         # (FRAME_IF, then, else, env)  -- waiting on a test value
FRAME_ARG  = 1         # (FRAME_ARG, arg, env)        -- waiting on a function value
FRAME_CALL = 2         # (FRAME_CALL, closure)        -- waiting on an argument value

def lEval( expr, env=None ):
    C = expr                                     # Control:      expression being evaluated
    V = None                                     # Value:        result flowing back in APPLY
    E = Environment() if env is None else env    # Environment:  lexical scope
    K = []                                       # Kontinuation: a stack of frames

    while True:

        # ----- EVAL: descend into C, pushing frames, until a leaf sets V -----
        while True:
            if isinstance( C, int ):             # a number -> itself
                V = C
                break
            elif isinstance( C, str ):           # a variable -> look it up
                V = E.lookup( C )
                break
            elif C[0] == 'lambda':               # (lambda param body) -> a closure
                V = ( VAL_CLOSURE, C[1], C[2], E )
                break
            elif C[0] == 'if':                   # (if test then else)
                K.append( (FRAME_IF, C[2], C[3], E) )
                C = C[1]                          # evaluate the test first
            else:                                # (fn arg) -- a function call
                K.append( (FRAME_ARG, C[1], E) )
                C = C[0]                          # evaluate the function first

        # ----- APPLY: feed V to the top frame -----
        while True:
            if not K:
                return V                          # nothing pending -> V is the answer

            frame = K.pop()
            ftag  = frame[0]

            if ftag == FRAME_IF:                  # V is the test value
                C = frame[2] if V == 0 else frame[1]   # 0 is false
                E = frame[3]
                break

            elif ftag == FRAME_ARG:               # V is the function value
                K.append( (FRAME_CALL, V) )       # remember the function
                C = frame[1]                      # evaluate the argument next
                E = frame[2]
                break

            elif ftag == FRAME_CALL:              # V is the argument value
                _, param, body, clo_env = frame[1]
                E = Environment( parent=clo_env, bindings={ param: V } )
                C = body                          # no frame pushed -> tail call reuses K
                break

        # fall through to the outer loop -- re-enter EVAL with the new C/E
```

Read it as the two states talking to each other.  The inner EVAL loop only ever
descends (it pushes frames and moves `C` inward) until a leaf (number, variable,
or `lambda`) drops a value into `V` and `break`s.  The inner APPLY loop only ever
resumes: it pops one frame, uses `V`, and either returns (K empty) or `break`s with
a fresh `C`/`E` for EVAL to descend into again.  The `break`s hand control back and
forth; nothing in the whole function calls `lEval`.  Every scrap of pending work,
tail and non-tail alike, is a tuple on `K`.


## 4.7 Watching K grow and shrink

The `if` trace in §4.3 and the call trace in §4.5 each held at most one note at a
time, so they never showed the thing `K` exists for: holding *several* pending jobs
at once, the way Python's stack used to.  For that we need a nested expression: a
call whose function part is *itself* a call.  Here is the curried constant function
from §4.1, `(((lambda x (lambda y x)) 3) 9)`, which is exactly that shape: to
evaluate the outer call we must first evaluate the inner call.

Two closures appear along the way, so name them for the trace: `#<x>` is the value
of `(lambda x (lambda y x))`, and `#<y>` is the value of the `(lambda y x)` it
returns.  Three environments appear too: `top` is where we start, `e1` binds `x:3`,
and `e2` binds `y:9`.  The `C` column shows the actual expression each pass is
working on, and in the `K` column `ARG(9)` and `CALL(#<x>)` abbreviate the
`(FRAME_ARG, 9, …)` and `(FRAME_CALL, #<x>)` tuples.

```
 step   C  (or action)                    K (bottom .. top)           V       what happens
 ----   ----------------------------      ----------------------      -----   ---------------------------------
   1    (((lambda x (lambda y x)) 3) 9)   []                           –      call: push ARG(9), C := ((lambda x (lambda y x)) 3)
   2    ((lambda x (lambda y x)) 3)       [ARG(9)]                     –      call: push ARG(3), C := (lambda x (lambda y x))
   3    (lambda x (lambda y x))           [ARG(9), ARG(3)]             –      lambda leaf: V := #<x>
   4    (consult K)                       [ARG(9), ARG(3)]             #<x>   pop ARG(3): push CALL(#<x>), C := 3
   5    3                                 [ARG(9), CALL(#<x>)]         #<x>   leaf: V := 3
   6    (consult K)                       [ARG(9), CALL(#<x>)]         3      pop CALL: bind x:3 in e1, C := (lambda y x)
   7    (lambda y x)   [in e1]            [ARG(9)]                     3      lambda leaf: V := #<y>
   8    (consult K)                       [ARG(9)]                     #<y>   pop ARG(9): push CALL(#<y>), C := 9
   9    9                                 [CALL(#<y>)]                 #<y>   leaf: V := 9
  10    (consult K)                       [CALL(#<y>)]                 9      pop CALL: bind y:9 in e2, C := x
  11    x   [in e2]                       []                           9      leaf: look up x -> e2 has y, e1 has x=3; V := 3
  12    (consult K)                       []                           3      K empty -> return 3
```

Follow the `K` column down and back up.  It starts empty, grows to **two** notes at
step 3 as the machine descends through the nested calls, then drains one note at a
time as each value arrives, back to empty at the end, and the answer, `3`, is the
`x` that the innermost lookup reaches two scopes up the chain.  That rise and fall
is the shape Python's call stack traced implicitly in Chapters 1–3; here it is out
in the open, a list we own.

And this is the payoff promised at the end of Chapter 3.  Nest the calls three deep,
ten deep, ten thousand deep, and nothing changes except how tall `K` grows at its
peak, and `K` is a list on the heap, with no fixed limit.  The non-tail depth that
overflowed Python's stack now lives somewhere that does not overflow.


## 4.8 Where the tail-call optimization lives now

Chapter 3 got tail calls into constant space by reusing the loop's frame instead of
recursing.  This machine keeps that property, and it is worth seeing *exactly* where
it comes from, because it is a single line, or rather, a single *missing* line.

Every frame on `K` is pushed for the same reason: the machine is about to descend
into a sub-expression whose value it will need *and then do more with*.  `FRAME_IF`
holds the branches while the test is evaluated; `FRAME_ARG` holds the argument while
the function is evaluated; `FRAME_CALL` holds the function while the argument is
evaluated.  Each records real leftover work, so each is a note on `K`.

But when `FRAME_CALL` finally makes the call, it installs the function's body into
`C` **and pushes nothing.**  There is no leftover work to record: the body's value
*is* the call's value, with nothing waiting to be done to it afterward.  So the body
runs at whatever `K` height the call happened at: it does not add a level.

Now suppose that body ends in another call: a tail call.  That call will push its
own `FRAME_ARG`/`FRAME_CALL` while it evaluates *its* function and argument, but
those pop off again before its body runs, leaving `K` exactly as tall as it started.
A function that tail-calls itself a million times rides that spot on `K` up and down
a million times and never climbs.  That is constant space, and it falls out of the
machine for free, not from any check for "is this a tail call?", but simply because
installing a body pushes no frame.  A *non*-tail call, by contrast, is always sitting
under a `FRAME_ARG` or `FRAME_CALL` that has not popped yet (that pending frame is
the leftover work), so it does add a level, exactly as it should.

Our shrunk language has no way to *count*, so we cannot write `countdown` here to
watch it loop flat.  That demonstration waits for Chapter 5, where the full language
returns on this very machine.  But the property is already built in: the CEK machine
gives tail calls constant space and gives non-tail calls a heap-allocated `K`, and
between the two, nothing in it ever touches Python's stack.

It is worth setting the three machines we have built side by side, because together
they tell one story about *where a program's unfinished work is kept*:

| interpreter | tail calls | non-tail pending work lives on... | a deep recursion |
|---|---|---|---|
| Recursive (Chapters 1–2) | a Python frame per call | Python's call stack | overflows, tail or not |
| Looping / TCO (Chapter 3) | constant space, one frame reused | Python's call stack | tail runs flat, non-tail overflows |
| CEK (Chapters 4–5) | constant space, no frame pushed | `K`, on the heap | both run flat |

Chapter 3 took the Python stack out of *tail* calls; this machine takes it out of
*non-tail* work too, by moving that pending computation onto `K`.  Reading down the
last column is the whole arc: overflow, then half of it fixed, then all of it.


## 4.9 Running it

The file's `main` runs the six programs the language can express.  Running
`python IttyBittyLisp4.py` prints:

```
>>> 42
==> 42

>>> ((lambda x x) 7)
==> 7

>>> (((lambda x (lambda y x)) 3) 9)
==> 3

>>> (if 1 100 200)
==> 100

>>> (if 0 100 200)
==> 200

>>> ((lambda f (f 3)) (lambda x x))
==> 3
```

A literal returns itself; identity returns its argument; the curried constant
returns `3` (traced in §4.7); the two `if`s pick their branch by whether the test is
`0`; and the last program passes one function as an argument to another and calls it:
a closure flowing through the machine as an ordinary value.  A closure that
reaches the top and becomes the answer prints as `#<procedure (x)>`, naming its
parameter, exactly as in Chapter 2.

It is worth sitting with how little the machine is: three frame tags, one closure
tag, two loops.  Everything a program does (choosing, calling, capturing,
returning) is that handful of tuples moving on and off `K`.  And the reason we
stripped the language to the studs was to make *this* visible with nothing else in
the frame.  Chapter 5 puts `let`, `set!`, `begin`, and the primitives back, and
every one of them is just more frame tags on the machine you now have whole.


## 4.10 Extending the machine

Chapter 5 grows this machine into the full language, and it does so entirely by
*adding frame tags*, the recipe from §4.5: a new tag, a push-site in EVAL, a handler
in APPLY.  Before we do that, it is worth naming what the machine guarantees and what
each new form has to decide, because those few facts are the whole of what you need
to extend it.  Chapter 5 is really just this list, applied.

1. **The machine is always in one of two states, and never both.**  EVAL descends: it
   takes `C` apart, pushes a note for each piece that must be evaluated first, and
   points `C` at that piece.  APPLY climbs: it takes the value in `V`, pops the top
   note off `K`, and acts on it.  Every form you add lives in exactly these two
   places, a push in EVAL and a handler in APPLY, and nowhere else.

2. **The switch between the two is mechanical, never a choice.**  EVAL becomes APPLY
   the instant a leaf drops a value into `V`.  APPLY becomes EVAL the instant a note
   hands back a fresh `C` to descend into.  APPLY finishes when `K` is empty.  The
   presence or absence of a value picks the state for you.

3. **A note is paused work made of data, and it must carry everything needed to
   resume.**  That means the sub-expressions still unevaluated, any values already
   computed that you will need later, and the environment to evaluate the rest in.
   Designing a new form is really answering one question: *what must the machine
   remember while this sub-expression runs?*  Whatever the answer is, that is the
   note.

4. **Carry the environment in any note that will later run code; leave it out of
   notes that only combine values.**  `FRAME_IF` and `FRAME_ARG` carry `E` because
   they still have an expression to evaluate when they resume; `FRAME_CALL` does not,
   because it builds a fresh environment of its own.  Forgetting to stash `E` is the
   commonest way an extension goes subtly wrong: the resumed expression evaluates in
   whatever scope happens to be current, not the one it was written in.

5. **A form with N pending sub-expressions needs N notes, or one note that
   accumulates.**  `if` had one thing to evaluate first, so one note; a call had two
   (function, then argument), so two.  A form with a whole list of sub-expressions can
   instead push a single note carrying "the values done so far and the expressions
   still to go," re-pushing a smaller copy of itself each time, the way a loop peels a
   list.  Chapter 5 uses both shapes.

6. **A note either sets up new code (and returns to EVAL) or produces a value (and
   stays in APPLY).**  `if`, `FRAME_ARG`, and `FRAME_CALL` all point `C` at a fresh
   expression and drop back to EVAL.  But a note can instead do its work, leave a
   value in `V`, and hand it straight to the note beneath it with no new code to run.
   Chapter 4 has no form of that second kind, which is why its APPLY loop never
   actually loops: every note bounces control back to EVAL.  The first forms that
   *stay* in APPLY are `set!` and the primitives in Chapter 5, and that is when the
   APPLY loop finally earns its name.  Deciding which kind each of a form's notes is
   is half the work of adding it.

7. **Push a note only when you will do more with the value; push nothing when the
   value is the answer.**  This is §4.8's point from the extender's side.  A
   sub-expression whose result you must still act on gets a note (it is non-tail); a
   body whose value *is* the enclosing form's value gets installed into `C` with
   nothing pushed (it is tail).  That one choice, note or no note, is where tail-call
   behavior comes from: push where you should not and you break constant-space tail
   calls; skip a push you needed and you lose the pending work outright.

With those seven in hand, every form in Chapter 5 reads as a variation on a note you
have already seen.


## 4.11 Challenges

- **Real booleans.**  Add `#t` and `#f` and make `if` test for `#f` instead of `0`
  (the way Chapters 1–3 did).  A number stops being true-or-false; you decide what
  `if` does when the test is a plain number, and say so.  Small change: a new leaf
  value and one line in the `FRAME_IF` case, and a good warm-up for the rest.

- **Primitives.**  Put `+`, `-`, `*`, and `=` back.  A one-argument call already
  works; a two-argument primitive like `(- n 1)` needs *both* arguments evaluated
  before it runs, which means one pending note per argument still to come: the same
  two-note pattern as a function call, generalized.  Sketch the frames you would
  need.  (This is a first taste of Chapter 5.)

- **Multi-argument `lambda`.**  Bring back `(lambda (a b) ...)`.  Now a call has a
  *list* of argument expressions to evaluate left to right, each landing in the new
  scope, before the body runs.  What does the note carry so the machine knows which
  arguments are done and which remain?

- **Print the continuation.**  Add a trace mode that prints `K` at the top of every
  pass, most-recent note last.  Run the curried-constant program and watch your
  output match the `K` column of §4.7 line for line.  You are printing the program's
  future (everything it still has to do), which in Chapters 1–3 was invisible
  inside Python.

- **First-class continuations (stretch).**  Here is why reifying `K` matters beyond
  avoiding overflow.  Because the continuation is now just a value, the list `K`,
  the machine can *hand it to a program*.  Add a form that captures the current `K`,
  wraps it as a one-argument function, and hands it to the program; calling that
  function later throws away whatever `K` is current and reinstates the captured one,
  resuming the old computation with the supplied value.  You have built `call/cc`,
  Scheme's most feared operator (early returns, loops, generators, and coroutines
  all fall out of it), and it is only reachable *because* the continuation stopped
  being Python's secret and became data you own.
