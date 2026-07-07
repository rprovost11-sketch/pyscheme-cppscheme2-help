# Chapter 3: Tail Calls and the Looping Evaluator

Chapter 2's interpreter can do real work (recursion, closures, the lot), but it
carries the limit we flagged at the end of §2.6: every function call makes `lEval`
call itself, and each of those pushes a Python stack frame that lives until the
call returns.  Pile up enough of them and Python runs out of stack.

Here is a program that ought to be harmless, count down from a large number to
zero:

```scheme
(set! countdown (lambda (n) (if (= n 0) 0 (countdown (- n 1)))))
(countdown 100000)
```

On the Chapter 2 evaluator, `(countdown 100000)` does not return `0`.  It crashes,
somewhere past a thousand steps, with Python's `RecursionError`.  And yet nothing
about `countdown` is deep or elaborate: it is a loop wearing the costume of a
function.  This chapter makes programs like it run in flat, constant space, the
way a loop should.  The language does not change at all; only the evaluator learns
a new trick.


## 3.1 Tail calls: when a frame is dead weight

Why does `countdown` pile up frames?  Follow what a single call has to do.  To
compute `(countdown 5)`, the evaluator reaches the `if`, finds `n` is not `0`, and
must evaluate `(countdown 4)`.  Until *that* finishes, `(countdown 5)` cannot
finish, so Python holds its frame open, waiting.  `(countdown 4)` is in the same
spot, waiting on `(countdown 3)`, and so on down to `(countdown 0)`.  A hundred
thousand calls, all open at once, each holding a frame: that is the overflow.

But look closely at what `(countdown 5)` is actually *waiting to do* with the
result of `(countdown 4)`.  The body is `(if (= n 0) 0 (countdown (- n 1)))`, and
when `n` is not zero the whole answer is just `(countdown (- n 1))`, nothing
more.  `(countdown 5)` does not add to that result, or compare it, or wrap it in
anything; it hands it straight back to *its* own caller, untouched.  Once
`(countdown 4)` is under way, the frame for `(countdown 5)` has no work left to
do.  It is being kept alive for nothing.

That "for nothing" is the whole idea.  A function call is in **tail position**
when its result *is* the result of the enclosing function: when nothing happens
to that value afterward except being returned.  A function call in tail position
is a **tail call**.  The recursive `(countdown (- n 1))` is a tail call: it is the
last thing `countdown` does, and its value becomes `countdown`'s value directly.

Contrast it with the recursive call in `factorial`:

```scheme
(lambda (n) (if (= n 0) 1 (* n (factorial (- n 1)))))
```

Here the recursive call `(factorial (- n 1))` is *not* the answer: its result
gets multiplied by `n` first.  The multiply is waiting on the recursive call, so
`factorial`'s frame genuinely has work left after the call returns; it must stay.
That call is **not** in tail position.

So the two recursions differ in a way that matters to the machine.  In
`factorial`, each frame must survive the call it makes, because it still owes a
multiply.  In `countdown`, each frame owes *nothing* after its tail call, and a
frame that owes nothing is dead weight.  The fix writes itself: when a call is in
tail position, do not open a new frame for it at all.  Reuse the current one.


## 3.2 The looping evaluator

"Reuse the current frame instead of opening a new one" is exactly what a loop
does: one frame, run over and over, its working state overwritten each time
around.  So we turn `lEval` into a loop.

Two pieces of state change as it runs: *which expression* we are evaluating, and
*in which environment*.  We give them short names (`C` and `E`) and call them
**registers**: the machine's live working state, the slots the loop overwrites in
place on each pass (the same sense as a CPU's registers).  They are nothing but
the old `expr` and `env`, promoted from fixed parameters into variables the loop
is allowed to overwrite:

```python
def lEval( expr, env ):
    C = expr   # Control:     the expression being evaluated right now
    E = env    # Environment: the scope it is evaluated in
    while True:
        ...
```

The Chapter 2 dispatch now moves *inside* `while True`, and each case does one of
two things.  If the next thing to evaluate is in **tail position**, we do not
recurse: we overwrite `C` (and `E`, if the scope changed) with it and `continue`,
jumping back to the top of the loop in the same frame.  If it is *not* in tail
position, we recurse with `lEval`, just as before.

Take `if`.  Its condition is not in tail position, its value is only used to pick
a branch, so we evaluate it by recursion.  But the chosen branch *is* in tail
position (its value is the value of the whole `if`), so instead of returning
`lEval(branch, E)`, we point `C` at the branch and loop:

```python
elif C[0] == 'if':
    condExpr, thenExpr, elseExpr = C[1:]
    condVal = lEval(condExpr, E)                     # condition: not tail -> recurse
    C = elseExpr if condVal == '#f' else thenExpr    # chosen branch: tail -> loop
    continue
```

Now the payoff: the user-defined function call.  In Chapter 2 we opened a
new scope on the function's captured environment and *recursed* into its body.
Now we reassign the registers instead: point `E` at the new scope, point `C` at
the body's final form, and loop.

```python
else:
    fn, *args = [ lEval(elt, E) for elt in C ]       # head + arguments: not tail -> recurse
    if callable(fn):                                 # a primitive
        return fn(args)
    else:                                            # a user-defined function
        E = Environment(parent=fn.env, bindings=dict(zip(fn.params, args)))
        for subExpr in fn.body[:-1]:                 # non-tail body forms -> recurse
            lEval(subExpr, E)
        C = fn.body[-1]                              # tail form of the body -> loop
        continue
```

Now walk `(countdown 100000)` through this.  Evaluating the body reaches the tail
call `(countdown (- n 1))`; the machine binds the new `n` in a fresh scope, sets
`E` to it, sets `C` to `countdown`'s body again, and loops.  No frame was pushed:
the *same* Python frame is simply running the loop one more time, with `C` and `E`
overwritten.  A hundred thousand tail calls become a hundred thousand trips around
one loop, in one frame.

That is what **constant space** means: the memory the evaluator uses does not grow
with the number of iterations, because each tail step *replaces* the current state
(`C`, `E`) instead of stacking a fresh copy on top of it.  Recursion stacks: a
frame per pending call; the loop overwrites: one frame, reused.  `countdown` now
runs all the way to `0` and returns.

### 3.2.1 Watching the loop

Watch it happen.  Here is `(countdown 3)` traced pass by pass through the `while`
loop.  It is all one Python frame, reused every time, with `C` and `E` overwritten
and nothing pushed.  Each pass shows the two registers `C` and `E`, and on the line
marked `>`, what the pass does and where the values it uses come from:

```
pass 1
   C  (countdown 3)
   E  (top level)
   >  call countdown: bind n:=3 in a new scope, C := its body

pass 2
   C  (if (= n 0) 0 (countdown (- n 1)))
   E  n = 3
   >  if: (= n 0) is #f, so C := else branch (countdown (- n 1))

pass 3
   C  (countdown (- n 1))
   E  n = 3
   >  tail call: (- n 1) = 2, bind n:=2 in a new scope, C := body

pass 4
   C  (if (= n 0) 0 (countdown (- n 1)))
   E  n = 2
   >  if: (= n 0) is #f, so C := else branch (countdown (- n 1))

pass 5
   C  (countdown (- n 1))
   E  n = 2
   >  tail call: (- n 1) = 1, bind n:=1 in a new scope, C := body

pass 6
   C  (if (= n 0) 0 (countdown (- n 1)))
   E  n = 1
   >  if: (= n 0) is #f, so C := else branch (countdown (- n 1))

pass 7
   C  (countdown (- n 1))
   E  n = 1
   >  tail call: (- n 1) = 0, bind n:=0 in a new scope, C := body

pass 8
   C  (if (= n 0) 0 (countdown (- n 1)))
   E  n = 0
   >  if: (= n 0) is #t, so C := then branch (0)

pass 9
   C  0
   E  n = 0
   >  leaf: a number is its own value; nothing pending, so return 0
```

Every pass either points `E` at a fresh scope for the next `n` (a tail call) or
points `C` at the branch the `if` chose, and loops.  Not one pass pushes a frame.
Where the Chapter 2 evaluator would by now be several `lEval` calls deep and
climbing, this is a single frame with two variables rewritten in place.  Scale the
`3` up to `100000` and the only thing that changes is the number of passes; the
frame count stays at one.


## 3.3 Two kinds of sub-expression

The whole method rests on sorting every sub-expression into one of two kinds.
Some are in tail position: their value is the value of the enclosing form, so the
evaluator can `continue` to them and reuse the frame.  The rest are not: their
value feeds into more work, so the evaluator must get it *back*, which means
recursing.  Here is the split for our language:

| Sub-expression | Tail position? | Evaluator |
|---|---|---|
| the chosen branch of an `if` | yes | `C = branch; continue` |
| the last form of a `begin` | yes | `C = last; continue` |
| the last form of a `let` body | yes | `C = last; continue` |
| the last form of a function's body | yes | `C = last; continue` |
| the condition of an `if` | no | `lEval(...)`, recurse |
| the init expressions of a `let` | no | `lEval(...)` |
| the head and arguments of a function call | no | `lEval(...)` |
| every non-last form of a `begin` / `let` / body | no | `lEval(...)` |

The pattern is easy to hold in mind: **the last thing a form does is a tail
position; everything it does along the way is not.**  A function call sitting in a
tail position is a tail call and loops; the same call sitting in a non-tail
position (as an argument, say) recurses.

Notice this split hands the right job to the right mechanism.  How deep can
non-tail recursion go?  Only as deep as expressions *nest in your source*, and a
program's nesting is fixed and shallow, a few dozen levels in anything sane.  How
deep can tail recursion go?  As deep as a loop *runs*, driven by data, with no
fixed bound.  This chapter gives the unbounded one (iteration count) to the loop
and leaves the bounded one (expression nesting) on Python's stack.  That is why
`(countdown 100000)` is fine and Python's ~1000-frame limit is never threatened:
the 100000 lives in the loop, not on the stack.

There is a reason Scheme cares about this more than most languages do.  Scheme has
no separate looping form you are *required* to reach for, no built-in `for` that
is secretly different from a function call.  In Scheme you loop *by making a tail
call*, and the language standard *guarantees* those run in constant space (it
calls them **proper tail calls**).  So this is not an optimization bolted on for
speed; it is what makes iteration expressible at all.  Chapters 1 and 2 were
Schemes that quietly broke that guarantee.  Chapter 3 keeps it.

## 3.4 The continuation

One limit is still with us.  Removing it is the whole subject of the next chapter,
so let us be clear about what it is.

The tail calls loop now, but the *non-tail* recursions still ride Python's stack,
and that stack is doing real work for us.  Think about what happens when the
evaluator recurses to evaluate a sub-expression: it computes a value and then
*returns* to the spot it left, where some leftover job is always waiting to be done
with that value.  In `(factorial 5)` the recursive call `(factorial 4)` comes back
to be *multiplied by 5*: the multiply is the leftover job.  Python's stack is what
remembers it: each paused `lEval` frame is holding one pending "…and then do this
with the result."

That leftover job (everything still waiting to be done once a sub-expression
produces its value) has a name: the **continuation** of that sub-expression.  Every
non-tail sub-expression has one, and it is always some small, specific task:

- the recursive call in `(* n (factorial (- n 1)))`: its continuation is *multiply
  me by `n`*;
- the condition of an `if`: its continuation is *choose a branch with me*;
- an argument mid-evaluation in a call: its continuation is *hold me, evaluate the
  next argument, then make the call*.

Stack these up and you have the whole of "what the program still has to do."  When
`factorial` is ten calls deep, Python's stack is holding ten continuations (ten
pending multiplies, one per paused frame) each waiting its turn as the values come
back.

The subtle part is this: **you cannot point at any of them.**  The continuation is
never a thing inside our interpreter: it is *implied* by where Python happens to be
in its own recursion.  When `lEval` recurses to evaluate `(factorial 4)`, the
"multiply by 5" step is simply the Python code sitting on the line after the
recursive call, waiting to run.  Real, and doing work, but living in Python's
runtime, not in ours, and entirely outside our control.  It is the one piece of the
evaluation our machine still borrows from its host.

That borrowing is exactly why a deeply *non-tail* program still overflows.  A
`factorial` run to enormous depth owes a multiply at every level; each pending
multiply is a continuation Python is holding on its stack for us, and deep enough,
that stack runs out, the very failure §3.1 opened with, now named.  This chapter
did nothing for it, and could not: leaning on Python's stack for non-tail work was
the whole design of the looping evaluator.

Removing that last reliance means making the continuation *ours*: taking it off
Python's stack and putting it in a register the machine owns outright, so that even
non-tail work lives on the heap where nothing can overflow.  That is the subject of
Chapters 4 and 5.  Chapter 4 builds that machine on the smallest language that still
shows it clearly; Chapter 5 puts the full language back on it, and the deep
`factorial` that overflows here runs in flat space.


## 3.5 Writing code that loops

Back in §3.1 we set `factorial` aside as the cautionary example: its recursive call
is *not* in tail position, because the result comes back to be multiplied by `n`,
so its frames pile up and, deep enough, overflow.  Is `factorial` simply doomed,
then?  No, and seeing why is the practical heart of this chapter.  You can very
often *rewrite* a recursion so its recursive call lands in tail position, and then
it loops like any other.

The trick is an **accumulator**: an extra argument that carries the
work-in-progress along, so the pending operation happens *before* the recursive
call instead of after it.  Here is `factorial`, both ways:

```scheme
;; non-tail: the multiply waits for the call to return
(set! factorial
  (lambda (n) (if (= n 0) 1 (* n (factorial (- n 1))))))

;; tail: the multiply happens first and rides along in acc
(set! fact-iter
  (lambda (n acc) (if (= n 0) acc (fact-iter (- n 1) (* n acc)))))

(fact-iter 5 1)     ; => 120
```

Look at what moved.  In the first version, when `(factorial (- n 1))` returns there
is still a `(* n ...)` waiting, and that waiting work is exactly what forces the
frame to stay.  In the second, the multiply has *already been done*: `(* n acc)` is
computed and handed forward as the new `acc`, so when `fact-iter` makes its
recursive call, nothing is left waiting on it.  The call is the last thing the
function does, a tail call, and it loops.  The computation is identical; only the
*place* the pending work sits has changed, from *after* the call to *inside its
arguments*.

This is the whole technique, and it carries everywhere.  Stated as a rule:

> The shape of the code does not decide whether it loops or stacks.  *Where the
> pending work sits* does.  Move that work into an accumulator argument, and a
> stacking recursion becomes a flat loop.

An accumulator is not just a trick for salvaging a throwaway result, either.  It is
how you write genuine iterative computations.  Summing the first hundred thousand
integers is a loop that carries a running total:

```scheme
(set! sum-to
  (lambda (n acc) (if (= n 0) acc (sum-to (- n 1) (+ acc n)))))

(sum-to 100000 0)   ; => 5000050000
```

A hundred thousand additions, a real answer, one flat frame.

**Tail calls between different functions loop, too.**  Nothing in the mechanism
cares whether a tail call lands back in the same function or a new one: it simply
points `C` and `E` at whatever body comes next.  So two functions can hand control
back and forth indefinitely without growing the stack:

```scheme
(set! even? (lambda (n) (if (= n 0) #t (odd?  (- n 1)))))
(set! odd?  (lambda (n) (if (= n 0) #f (even? (- n 1)))))

(even? 100000)      ; => #t
```

`even?` tail-calls `odd?`, which tail-calls `even?`, a hundred thousand times
alternating, and the stack stays flat the whole way.  An ordinary `while` loop
cannot express "two functions taking turns" nearly so cleanly; tail calls can, and
that is a large part of why Scheme leans on them instead of on special loop syntax.


## 3.6 The complete evaluator

Here is the whole looping evaluator.  The `Environment` and `Function` classes are
unchanged from Chapter 2: closures are exactly what they were; only the way
`lEval` drives itself is new.  Read it straight down, watching for the two
endings each case can have: it either `return`s a finished value, or reassigns
`C`/`E` and `continue`s.

```python
def lEval( expr, env ):
    C = expr   # Control:     the expression being evaluated right now
    E = env    # Environment: the scope it is evaluated in
    while True:
        if C in ('#t', '#f'):          # a boolean: return it unchanged
            return C
        elif isinstance(C, str):       # a name: look it up along the chain
            return E.lookup(C)
        elif not isinstance(C, list):  # a number (any non-list): return unchanged
            return C

        elif C[0] == 'set!':
            name, valExpr = C[1:]
            val = lEval(valExpr, E)                        # value: not tail -> recurse
            return E.set(name, val)

        elif C[0] == 'if':
            condExpr, thenExpr, elseExpr = C[1:]
            condVal = lEval(condExpr, E)                   # condition: not tail -> recurse
            C = elseExpr if condVal == '#f' else thenExpr  # chosen branch: tail -> loop
            continue

        elif C[0] == 'begin':
            for subExpr in C[1:-1]:                        # non-tail forms -> recurse
                lEval(subExpr, E)
            C = C[-1]                                      # tail: last form -> loop
            continue

        elif C[0] == 'quote':
            return C[1]

        elif C[0] == 'lambda':                            # capture the current env
            params, *body = C[1:]
            return Function(params, body, E)

        elif C[0] == 'let':
            bindingPairs, *body = C[1:]
            initialBindings = { name: lEval(initExpr, E) for name, initExpr in bindingPairs }
            E = Environment( parent=E, bindings=initialBindings )   # open the new scope
            for subExpr in body[:-1]:                      # non-tail body forms -> recurse
                lEval(subExpr, E)
            C = body[-1]                                   # tail: last body form -> loop
            continue

        else:
            fn, *args = [ lEval(elt, E) for elt in C ]     # head + arguments: not tail -> recurse
            if callable(fn):                               # a primitive
                return fn(args)
            else:                                          # a user-defined function
                E = Environment(parent=fn.env, bindings=dict(zip(fn.params, args)))
                for subExpr in fn.body[:-1]:               # non-tail body forms -> recurse
                    lEval(subExpr, E)
                C = fn.body[-1]                            # tail call: loop, no stack growth
                continue
```

Every case that `return`s is a place where evaluation is genuinely finished: an
atom, a `quote`, a freshly built closure, a primitive result, or a `set!`.  Every
case that ends in `C = ...; continue` is a tail position, handed back to the loop.
The recursive `lEval` calls that remain (a condition, the arguments, the non-tail
body forms) are the non-tail work, and they are the only thing still using
Python's stack.


## 3.7 Running it

Nothing in the plumbing changed.  The global environment is still the root
`Environment` holding the primitives, and `lisp_str` still renders numbers, lists,
and `#<procedure ...>` exactly as in Chapter 2: this chapter touched only
`lEval`.  What changed is what the interpreter can *survive*.  Point the §1.6 REPL
at `IttyBittyLisp3` and run the programs from this chapter:

```
lisp> (set! countdown (lambda (n) (if (= n 0) 0 (countdown (- n 1)))))
#<procedure (n)>
lisp> (countdown 100000)
0
lisp> (set! sum-to (lambda (n acc) (if (= n 0) acc (sum-to (- n 1) (+ acc n)))))
#<procedure (n acc)>
lisp> (sum-to 100000 0)
5000050000
lisp> (set! even? (lambda (n) (if (= n 0) #t (odd? (- n 1)))))
#<procedure (n)>
lisp> (set! odd? (lambda (n) (if (= n 0) #f (even? (- n 1)))))
#<procedure (n)>
lisp> (even? 100000)
#t
```

Every one of these is a `RecursionError` on the Chapter 2 evaluator.  Here each
runs in a single, flat Python frame: the countdown counting, the accumulator
summing a real answer, the two predicates taking turns a hundred thousand times.
The complete file is `examples/IttyBittyLisp3.py`, and `python IttyBittyLisp3.py`
runs the countdown as its finale.

Keep two limits in view.  First, from §3.5: not every recursion
turns tail-recursive so easily: a computation that genuinely must remember a stack
of unfinished work (walking a deep tree, say) still stacks frames, and deep enough,
still overflows.  Chapters 4 and 5 remove even that last reliance on Python's stack.

Second (the question lurking under this whole chapter), *why did we have to build
any of this by hand?*  Some languages perform tail-call optimization for you: the
compiler spots a tail call and reuses the frame.  Python, by deliberate design,
does not (its authors preferred clearer stack traces to the feature).  So on
Python, constant-space iteration is not free: if you want it, you build it, which
is exactly what our `while` loop is.  A driving loop that replaces call-stack
recursion with overwritten state has a name: a **trampoline**.  Ours is a small
one, but the pattern is the standard way to get proper tail calls on a host that
will not hand them to you.


## 3.8 Challenges

Every one of these leans on the looping mechanism: a tail position is
`C = ...; continue`.  Try them at the REPL against `IttyBittyLisp3.py`.

- **Add `while`.**  A genuine loop form: `(while test body...)` evaluates `test`,
  and as long as it is true, evaluates the body and goes round again.  It is a
  natural fit for the looping evaluator: implement it as a special form that runs
  the body, then sets `C` back to the whole `(while ...)` expression and
  `continue`s, so the test is checked afresh.  Because it loops rather than
  recurses, it will spin a million times without touching the stack.  (Decide what
  it returns once the test is finally false, `0`, or your unspecified value, and
  say so.)

- **Give `cond` proper tail calls.**  If you built `cond` back in Chapter 1 (the
  multi-branch `if`), bring it over, but make sure the winning clause's body ends
  in tail position: set `C` to that body's last form and `continue`, rather than
  recursing into it.  Then a tail-recursive function that loops through `cond` runs
  in constant space, exactly like one that loops through `if`.  (Quick check:
  rewrite `countdown` with `cond` instead of `if` and run it to 100000.)

- **Add named `let`.**  Scheme's idiomatic loop is a `let` with a name:
  `(let loop ((n 100000)) (if (= n 0) 0 (loop (- n 1))))`.  It binds `loop` to a
  function of the listed variables and immediately calls it; calling `loop` again
  inside the body is the next turn of the loop.  Add it as a special form.  The
  nice part: you need do nothing special for the tail call.  `(loop (- n 1))` is
  already in tail position, so the machinery you built loops it for free.

- **Add a tracing mode.**  Behind a flag, make the evaluator print each function
  call as it enters (the function's name and arguments, indented by how deeply
  calls are nested) and its result as it leaves.  Run a non-tail function like
  `factorial` and you get the familiar nested trace.  Now run a *tail*-recursive
  one like `countdown`: because each tail call loops instead of returning, it never
  prints a matching "leaving" line: a vivid, moving picture of exactly the frames
  that tail-call optimization makes vanish.

- **Add a macro expander (stretch).**  A **macro** is a rule that rewrites code
  *before* it is evaluated: it takes the unevaluated expression and returns a new
  one, and the evaluator runs *that* instead.  In the loop it is a two-line move:
  before the function-call case, check whether `C[0]` names a macro; if so, replace
  `C` with the macro's output and `continue`, so the rewrite is evaluated in turn.
  That "code is just lists you can transform" is the deepest trick Lisp has, and it
  turns `define`, `cond`, `and`/`or`, and much more into things written *in* the
  language rather than baked into the evaluator.  The full, *hygienic* version is a
  project of its own, but a basic `define-macro` is within reach right here.  For
  instance, `when` (a one-armed `if` with an implicit `begin`) is a macro in one line:

  ```scheme
  (define-macro (when test . body)
    (list 'if test (cons 'begin body)))
  ```

  which rewrites `(when ready (go) (log))` into `(if ready (begin (go) (log)))` before
  the evaluator ever sees it.  (We build the output with `list` and `cons` because that
  is all the language has; a prettier template syntax, quasiquote, is one of the
  projects.)
