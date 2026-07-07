# Chapter 5: The CEK Machine, Complete

Chapter 4 built the CEK machine on a deliberately tiny language (pure functions
and `if`) so that the machine itself, the four registers and the two-state loop and
the stack of continuation frames, was the only new thing to look at.  That was the
hard part, and it is behind us.

Now we put the language back.  All of it: `#t` and `#f`, `quote`, `set!`, `begin`,
`let`, multi-argument functions, and the primitives: everything the looping
evaluator of Chapter 3 could do.  The point of this chapter is what *doesn't* happen
when we do.  The two loops do not change.  The `K` stack does not change.  Adding a
whole language turns out to be adding **more kinds of frame** and nothing else:
each new form contributes its own note describing its own pending work, and the
machine that shuttles notes on and off `K` runs them all without noticing the
difference.

There is exactly one exception, and it is a good one: multi-argument function calls
(or **applications**, the standard term for a function-call expression) force us to
*redesign* the application frame from Chapter 4, and the redesign is the answer to the
very question Chapter 4's challenges left you with.  We build up to
it.  And at the end of the chapter the deep `factorial` that overflowed Python's
stack in Chapter 3 runs to a depth of fifty thousand and returns: the promise made
back then, finally paid.


## 5.1 The language returns

Here is what comes back, with the surface syntax the parser already produces.  Most
of it you have seen since Chapter 1; read this as a refresher, with one genuine
change of rule flagged along the way.

- **Booleans, `#t` and `#f`**: and with them, real Scheme *truthiness*.  This is the
  one changed rule, so read it carefully: **`#f` is the only false value.**  Every
  other value is true, including `0`, including the empty list, including any
  closure.  So `(if 0 'yes 'no)` returns `yes`, because `0` is true.  This *reverses*
  Chapter 4's temporary rule, where a number was false when it was `0`; that rule
  existed only because Chapter 4's stripped language had no real booleans to test.
  Chapters 1–3 already used this Scheme rule, and now it is back for good.

- **`quote`**: `(quote (a b c))` returns the list `(a b c)` *unevaluated*, as data.
  It is the one form that evaluates none of its parts.

- **`set!`**: `(set! x 42)` evaluates `42` and binds it to `x`.

- **`begin`**: `(begin e1 e2 e3)` evaluates each expression in order and returns the
  last; the earlier ones are run for their side effects.

- **`let`**: `(let ((a 3) (b 4)) body)` binds `a` and `b` locally and evaluates the
  body.

- **Multi-argument functions**: `lambda` takes a *list* of parameters again, and a
  body of one or more expressions: `(lambda (x y) (+ x y))`.  Calls take as many
  arguments as the function has parameters: `(f 3 4)`.  (Chapter 4's one-parameter,
  parens-free `lambda` was another of its simplifications; this undoes it.)

- **Primitives**: `+`, `-`, `*`, `=`, `<`, and `print`, living in the global
  environment as before.

Put together, the language is exactly Chapter 3's again:

```scheme
(begin
  (set! square (lambda (n) (* n n)))
  (let ((a 3) (b 4))
    (+ (square a) (square b))))          ; => 25
```

Our job is to make the CEK machine run this, without disturbing the machine.


## 5.2 Values that just flow: `set!`

Every frame in Chapter 4 ended the same way: it used the value in `V` to set up a
*new expression* in `C`, and control dropped back into EVAL to evaluate it.  Picking
an `if` branch, installing a function body: each handed EVAL something new to chew
on.

Some of the returning forms are not like that.  When `set!` finally has its value,
there is no new expression to evaluate: it just stores the value and is done.  So it
needs a second way for a frame to finish: **do its work, and stay in APPLY** rather
than dropping back to EVAL.  In the code that is the difference between `break`
(leave APPLY, re-enter EVAL with a new `C`) and `continue` (stay in APPLY, carry the
value `V` straight to the next frame).

`set!` evaluates its value expression, remembering the name to bind it to.  In EVAL
that is one more frame:

```python
FRAME_SET = 1                        # tag: "value coming; then assign it to a name"

elif C[0] == 'set!':                 # (set! name valueExpr)
    K.append( (FRAME_SET, C[1], E) )     # remember the name...
    C = C[2]                             # ...and evaluate the value first
```

When the value lands in `V`, APPLY pops the note, performs the assignment, and, with
no new expression to evaluate, flows the value onward by staying in APPLY:

```python
elif ftag == FRAME_SET:              # (FRAME_SET, name, env); V is the value
    frame[2].set( frame[1], V )          # assign; V remains the result of set!
    continue                             # stay in APPLY -- nothing new to evaluate
```

That `continue` is the whole new idea of this section.  A frame no longer has to
produce work for EVAL; it can simply transform the value in flight and pass it to the
frame beneath it.  It is also the first time the APPLY loop actually *loops*: in
Chapter 4 every frame handed control back to EVAL, so APPLY never ran twice in a row,
and `set!` is the first frame that stays and lets it iterate (Chapter 4's §4.10 list,
point 6).  We will meet the same `continue` again in a moment, when a primitive
computes a result.


## 5.3 Sequencing: `begin` and multi-expression bodies

`begin` runs several expressions in order and keeps only the last one's value.  That
means the machine must evaluate the first, *throw its value away*, evaluate the next,
and so on.  The note that remembers "there are still forms to run" is `FRAME_SEQ`:

```python
FRAME_SEQ = 2                        # tag: "forms still to run in sequence"

elif C[0] == 'begin':                # (begin form1 form2 ...)
    forms = list( C[1:] )
    if len(forms) > 1:
        K.append( (FRAME_SEQ, forms[1:], E) )   # remember the rest...
    C = forms[0]                                # ...and evaluate the first
```

When the first form's value comes back, `FRAME_SEQ` does something none of the other
frames do: it *ignores* `V`.  The value of a non-final form in a `begin` is discarded:
only its side effects mattered.  The frame drops it, evaluates the next form, and
re-pushes itself if more remain:

```python
elif ftag == FRAME_SEQ:              # (FRAME_SEQ, remaining_forms, env); V discarded
    forms = frame[1]
    E     = frame[2]
    if len(forms) > 1:
        K.append( (FRAME_SEQ, forms[1:], E) )
    C = forms[0]
    break
```

A frame that re-pushes a smaller copy of itself is how the machine loops over a list
of forms without any recursion: each pop peels off one form and, if the tail is
non-empty, sets the tail aside again.  The last form is *not* re-pushed, so its value
is left in `V` to become the value of the whole `begin`.

This same frame does a second job for free.  A function body may now hold several
expressions, `(lambda (n) e1 e2 e3)`, and those run exactly like a `begin`: all but
the last for effect, the last for its value.  So when the machine calls a function
(in §5.4) it will push a `FRAME_SEQ` for the body's trailing forms, reusing this
frame with no new code.


## 5.4 Many arguments

Now the one place the machine's shape really changes, and the reason is multiple
arguments.

Recall how Chapter 4 evaluated the one-argument call `(f 3)`: it used *two* frames in
succession.  `FRAME_ARG` held the argument `3` while the function `f` was evaluated;
then `FRAME_CALL` held the evaluated function while the argument `3` was evaluated;
then it made the call.  Two frames, because there were two things, a function and an
argument, to evaluate one at a time, each needing the other held aside.

With `(+ a b c)` there are *four* things to evaluate in order (the operator `+` and
its three operands) and each one, as it is evaluated, needs everything already done
to be held aside.  (`+` has taken any number of arguments since Chapter 1, and a
user-defined `lambda` may too; the machinery below handles operands of any count.)
Chapter 4's challenge asked exactly this: *what does the note carry so the machine
knows which arguments are done and which remain?*  Here is the answer: a single frame
that carries **two lists**: `done`, the values evaluated so far, and `todo`, the
expressions still to evaluate:

```python
FRAME_ARG = 3                        # tag: "accumulating operator + operands"

else:                                # (fn arg1 arg2 ...) -- an application
    K.append( (FRAME_ARG, [], list(C[1:]), E) )   # done=[], todo=[args]
    C = C[0]                                       # evaluate the operator first
```

The operator is evaluated first; its value will be the first thing to land in `done`.
Each time a value arrives, APPLY pops the frame, appends the value to `done`, and asks
one question: *is there anything left in `todo`?*  If so, it re-pushes the frame, now
with a longer `done` and a shorter `todo`, and evaluates the next expression.  If
not, everything is in hand and it makes the call:

```python
elif ftag == FRAME_ARG:              # (FRAME_ARG, done, todo, env); V just arrived
    done = frame[1] + [V]                # the operator, then each operand, accumulates
    todo = frame[2]
    if todo:                             # more to evaluate?
        K.append( (FRAME_ARG, done, todo[1:], frame[3]) )
        C = todo[0]
        E = frame[3]
        break
    # operator and all operands are evaluated: done[0] is the function, done[1:] the args
    fn, args = done[0], done[1:]
    if callable( fn ):                   # a primitive: compute the value and flow it on
        V = fn( args )
        continue                         # stay in APPLY (just like set!)
    _, params, body, clo_env = fn        # a closure: bind the parameters, run the body
    E = Environment( parent=clo_env, bindings=dict( zip(params, args) ) )
    if len(body) > 1:
        K.append( (FRAME_SEQ, body[1:], E) )   # trailing body forms -> a sequence frame
    C = body[0]
    break
```

Pause on three things here.

First, the frame **re-pushes a shrinking copy of itself**, the same trick `FRAME_SEQ`
used, so one frame kind walks the whole operator-and-operands list, no matter how
long, with no recursion.  If an operand is itself a call, evaluating it simply pushes
its *own* `FRAME_ARG` on top, and this one waits underneath, exactly the growing-and-
shrinking `K` you traced in §4.7.

Second, the two endings you already know reappear.  A **primitive** is a Python
callable: once its arguments are ready, we call it and `continue`, flowing the result
onward, the same "value that just flows" as `set!`.  A **closure** binds its
parameters (now several at once, `zip(params, args)`) in a fresh scope on its captured
environment, and installs its body, pushing a `FRAME_SEQ` for the trailing forms and
pointing `C` at the first.  And note what it does *not* do: pushing a frame for the
*call itself*.  As in Chapter 4, installing a body pushes nothing, so a tail call
reuses the current `K` depth.  Tail-call optimization survives the redesign untouched.

Third, `done[0]` is the function because the operator was evaluated first.  A short
trace of `(+ 3 2)` shows the accumulation, in the same step-by-step format as
Chapter 4:

```
step 1
   C  (+ 3 2)
   K  []
   V  –
   >  call: push ARG(done [], todo [3,2]); evaluate operator, C := +

step 2
   C  +
   K  [ARG([],[3,2])]
   V  –
   >  leaf: look up + in E, so V := #<+> (the primitive)

step 3
   C  (consult K)
   K  [ARG([],[3,2])]
   V  #<+>
   >  pop ARG: append V to done=[#<+>]; todo left, C := 3

step 4
   C  3
   K  [ARG([#<+>],[2])]
   V  #<+>
   >  leaf: a number is its own value, so V := C = 3

step 5
   C  (consult K)
   K  [ARG([#<+>],[2])]
   V  3
   >  pop ARG: append V to done=[#<+>,3]; todo left, C := 2

step 6
   C  2
   K  [ARG([#<+>,3],[])]
   V  3
   >  leaf: a number is its own value, so V := C = 2

step 7
   C  (consult K)
   K  [ARG([#<+>,3],[])]
   V  2
   >  pop ARG: done=[#<+>,3,2], todo empty; call + on (3,2)

step 8
   C  (consult K)
   K  []
   V  5
   >  + returned 5, so V := 5; K empty, so return 5
```

(`#<+>` is the `+` primitive.)  The operator lands in `done` first, each operand
follows, and when `todo` runs dry the primitive fires.  One frame kind, accumulating,
and it subsumes Chapter 4's `FRAME_ARG` *and* `FRAME_CALL` in a single note.

That trace stayed one frame deep, because every operand was a leaf.  Nest one call
inside another and `K` finally grows.  Take `(+ (* 2 3) 1)`, and the moment to
watch comes as the machine turns to evaluate the operand `(* 2 3)`: the outer `+`
is only partway done (it has its operator, and still has `1` left to do), so its frame
is sitting on `K` when the inner call pushes its own on top:

```
K (bottom .. top):   [ ARG([#<+>], [1]),   ARG([], [2, 3]) ]
                         outer + waiting      inner * running
```

`K` is two frames deep.  The `*` frame runs to completion exactly as `(+ 3 2)` did
above, produces `6`, and pops off; the `+` frame underneath then resumes with `6` as
its next value, evaluates the remaining `1`, and fires `+` on `(6, 1)` to give `7`.
That two-deep moment is the call stack you watched grow and shrink in §4.7, now in
plain arithmetic: each nested call parks the outer computation on `K` and picks it up
again when the inner one returns.  Deeper nesting only stacks more frames, and every
one of them is a tuple on the heap, never a Python frame.


## 5.5 `let` for free

One form is left, and it needs no machinery at all.

A `let` is just a function call wearing different clothes.  `(let ((a 3) (b 4)) body)`
means: make a function whose parameters are `a` and `b`, whose body is `body`, and
call it on `3` and `4`.  Written out, that is `((lambda (a b) body) 3 4)`, an
ordinary application, which the machine already runs.

So `let` needs no frame and no APPLY case.  In EVAL we simply *rewrite* it into the
application it stands for and let the loop dispatch on the result:

```python
elif C[0] == 'let':                  # (let ((name init)...) body...)
    names = [ pair[0] for pair in C[1] ]
    inits = [ pair[1] for pair in C[1] ]
    C = [ ['lambda', names] + list(C[2:]) ] + inits   # ((lambda (names) body) inits)
    # no frame, no break -- the loop re-dispatches on the rewritten C
```

This is **desugaring**: expressing a convenient form as a combination of forms the
machine already has, by rewriting the code before evaluating it.  `let` is *syntactic
sugar* for a `lambda` applied to its initializers, the same "code is just lists you
can transform" idea the macro challenge in Chapter 3 pointed at, here doing real work
in one case.  The machine never learns what `let` is; it only ever sees the
application `let` expands to.


## 5.6 The complete machine

Four frame kinds now (`FRAME_IF` carried over from Chapter 4, and `FRAME_SET`,
`FRAME_SEQ`, `FRAME_ARG` added here) one closure kind, the same four registers, the
same two loops.  Here is the whole evaluator, as it stands in
`examples/IttyBittyLisp5.py`.  The `Environment` class is unchanged all the way back
to Chapter 2 and is not reprinted.

```python
VAL_CLOSURE = 1        # a closure value: (VAL_CLOSURE, params, body, captured-env)

FRAME_IF  = 0          # (FRAME_IF, then, else, env)   -- waiting on a test value
FRAME_SET = 1          # (FRAME_SET, name, env)        -- waiting on a value to assign
FRAME_SEQ = 2          # (FRAME_SEQ, remaining, env)   -- forms still to run in sequence
FRAME_ARG = 3          # (FRAME_ARG, done, todo, env)  -- accumulating operator + operands

def lEval( expr, env ):
    C = expr                                 # Control:      expression being evaluated
    V = None                                 # Value:        result flowing back in APPLY
    E = env                                  # Environment:  lexical scope
    K = []                                   # Kontinuation: a stack of frames

    while True:

        # ----- EVAL: descend into C, pushing frames, until a leaf sets V -----
        while True:
            if C in ('#t', '#f'):                    # boolean -> itself
                V = C; break
            elif isinstance( C, (int, float) ):      # number -> itself
                V = C; break
            elif isinstance( C, str ):               # variable -> look it up
                V = E.lookup( C ); break
            elif C[0] == 'quote':                    # (quote datum) -> the datum, unevaluated
                V = C[1]; break
            elif C[0] == 'lambda':                   # (lambda params body...) -> a closure
                V = ( VAL_CLOSURE, C[1], list(C[2:]), E ); break
            elif C[0] == 'if':                       # (if test then else)
                K.append( (FRAME_IF, C[2], C[3], E) )
                C = C[1]
            elif C[0] == 'set!':                     # (set! name valueExpr)
                K.append( (FRAME_SET, C[1], E) )
                C = C[2]
            elif C[0] == 'begin':                    # (begin form...)
                forms = list( C[1:] )
                if len(forms) > 1:
                    K.append( (FRAME_SEQ, forms[1:], E) )
                C = forms[0]
            elif C[0] == 'let':                      # (let ((name init)...) body...)
                names = [ pair[0] for pair in C[1] ]
                inits = [ pair[1] for pair in C[1] ]
                C = [ ['lambda', names] + list(C[2:]) ] + inits   # desugar; re-dispatch
            else:                                    # (fn arg...) -- an application
                K.append( (FRAME_ARG, [], list(C[1:]), E) )
                C = C[0]

        # ----- APPLY: feed V to the top frame -----
        while True:
            if not K:
                return V

            frame = K.pop()
            ftag  = frame[0]

            if ftag == FRAME_IF:                     # V is the test value
                C = frame[1] if V != '#f' else frame[2]   # #f is the ONLY false value
                E = frame[3]
                break

            elif ftag == FRAME_SET:                  # V is the value to assign
                frame[2].set( frame[1], V )
                continue                             # value flows on; stay in APPLY

            elif ftag == FRAME_SEQ:                  # V (the previous form's value) discarded
                forms = frame[1]; E = frame[2]
                if len(forms) > 1:
                    K.append( (FRAME_SEQ, forms[1:], E) )
                C = forms[0]
                break

            elif ftag == FRAME_ARG:                  # V just arrived (operator, then each operand)
                done = frame[1] + [V]
                todo = frame[2]
                if todo:
                    K.append( (FRAME_ARG, done, todo[1:], frame[3]) )
                    C = todo[0]; E = frame[3]
                    break
                fn, args = done[0], done[1:]
                if callable( fn ):                   # primitive: compute and flow on
                    V = fn( args )
                    continue                         # stay in APPLY
                _, params, body, clo_env = fn        # closure: bind params, run body
                E = Environment( parent=clo_env, bindings=dict( zip(params, args) ) )
                if len(body) > 1:
                    K.append( (FRAME_SEQ, body[1:], E) )
                C = body[0]                           # no frame for the call -> tail calls reuse K
                break
```

The only line in the whole EVAL/APPLY skeleton that changed from Chapter 4 is the one
marked "the ONLY false value": `FRAME_IF` now tests `V != '#f'` instead of `V == 0`,
restoring Scheme truthiness.  Everything else grew by *addition* (three new frame
tags, three new EVAL cases, three new APPLY cases, one desugaring) laid onto a
skeleton that did not move.

The primitives and the global environment are the same idea as Chapter 3, a
dictionary of Python callables that take a list of already-evaluated arguments:

```python
globalBindings = {
    '+': lambda a: sum(a),        '-': lambda a: a[0] - a[1],     # + is variadic; (+) is 0
    '*': lisp_mul,                '=': lambda a: '#t' if a[0] == a[1] else '#f',
    '<': lambda a: '#t' if a[0] <  a[1] else '#f',
    'print': lisp_print,
}
global_env = Environment( bindings=globalBindings )
```

`+` and `*` take any number of arguments (`+` folds them with `sum`, `*` with a tiny
`lisp_mul` helper) so `(+ a b c)` and `(* 2 3 4)` work directly.  `-`, `=`, and `<`
stay two-argument for now (the challenges widen them).  Each primitive receives the
whole list of already-evaluated arguments, which is exactly what the accumulating
`FRAME_ARG` hands it.

`lisp_str` gains one line since Chapter 4 (a bare primitive prints as `#<primitive>`)
and renders a closure with its full parameter list, `#<procedure (x y)>`.


## 5.7 Running it

`python IttyBittyLisp5.py` runs the whole language:

```
>>> (+ (- 10 7) 2)
==> 5

10
>>> (+ (print 10) 5)
==> 15

>>> (set! x (* 6 7))
==> 42

>>> x
==> 42

>>> (set! square (lambda (n) (* n n)))
==> #<procedure (n)>

>>> (square 5)
==> 25

>>> (let ((a 3) (b 4)) (+ (* a a) (* b b)))
==> 25

>>> (begin (set! y 1) (set! y (+ y 9)) y)
==> 10

>>> (if 0 100 200)
==> 100

>>> (quote (a b c))
==> (a b c)
```

Everything Chapter 3 could do, now running on the CEK machine.  Two lines stand out.
`(if 0 100 200)` returns `100`: `0` is *true*, the restored Scheme rule.
And in `(+ (print 10) 5)` the bare `10` appears *above* the `>>>` line: `print` is a
side effect that fires while the expression is being evaluated, before `run` echoes
the form, and it *returns* its argument, so the `10` flows on into the `+` and the
value is `15`.

Now the two lines that were the whole point.  First, a tail-recursive `countdown`:

```
>>> (set! countdown (lambda (n) (if (= n 0) 0 (countdown (- n 1)))))
==> #<procedure (n)>

>>> (countdown 100000)
==> 0
```

A hundred thousand tail calls, and `K` never grows past the depth it started at,
because calling a function pushes no frame.  This is Chapter 3's constant-space tail
recursion, now living on the explicit stack.

Second, the promise from the end of Chapter 3.  Define `factorial`, whose recursive
call is *not* in tail position (its result is still owed a multiply):

```scheme
(set! factorial (lambda (n) (if (= n 0) 1 (* n (factorial (- n 1))))))
```

Run it deep.  On the **Chapter 3 evaluator**, `(factorial 1000)` overflows Python's
stack, a `RecursionError`, because each of the thousand pending multiplies is a
paused `lEval` frame, and Python's stack tops out under that. On **this** machine,
`(factorial 1000)` returns its 2,568-digit answer, and `(factorial 50000)` returns a
213,237-digit integer, fifty thousand nested non-tail multiplies, without a stumble.
The reason is the whole arc of Chapters 4 and 5 in one sentence: **`lEval` never calls
itself**, so those fifty thousand pending multiplies are not Python stack frames but
`FRAME_ARG` notes on `K`, a list on the heap that grows as far as memory allows.

That is the machine complete.  Tail calls run in constant `K`; non-tail calls grow
`K` on the heap; and *nothing*, tail or non-tail, rides Python's call stack anymore.
The continuation is fully ours.


## 5.8 Challenges

- **Finish the variadic primitives.**  `+` and `*` already take any number of
  arguments; bring the rest up to full Scheme.  `-` should negate a lone argument
  (`(- 5)` is `-5`) and left-fold two or more (`(- 10 3 2)` is `5`); `=` and `<`
  should *chain*, so `(< 1 2 3)` is true when each argument is less than the next.
  This is a change to the primitives alone, folding over the `args` list the machine
  already hands them, and the machine does not move.

- **`cond`, `and`, `or` by desugaring.**  Add these the way `let` was added: as
  rewrites in EVAL into forms the machine already runs, with no new frames.  `(and a
  b c)` becomes nested `if`s; `cond` becomes nested `if`s; `or` too, though `or` must
  take care to evaluate each test only once.  Make sure the final branch stays in tail
  position so a loop written with `cond` runs in constant space.

- **`define`.**  Add `(define name value)` and `(define (f a b) body...)` as sugar for
  `set!` plus `lambda`, again by rewriting in EVAL.  Now the examples read like real
  Scheme instead of a string of `set!`s.

- **Trace the continuation.**  Bring back the trace idea from Chapter 4: print `K` at
  the top of each pass.  Run `factorial` a few levels deep and watch the tower of
  `FRAME_ARG` notes, one pending multiply each, build up and drain.  You are looking
  directly at the stack that used to be Python's and is now yours.

- **`call/cc` (stretch).**  With the full language in place, finish the first-class
  continuation sketched in Chapter 4: a primitive `call/cc` that reifies the current
  `K` as a callable escape procedure.  Because `K` is an ordinary list you own, capturing
  it is copying a list, and invoking it is replacing the current `K` with the copy.
  Early exit, retry, generators, and coroutines all become expressible: the deepest
  dividend of having taken the continuation away from Python.
