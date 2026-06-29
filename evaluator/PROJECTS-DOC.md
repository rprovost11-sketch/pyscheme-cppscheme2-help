# Projects

*Larger challenges for the IttyBittyLisp series.*

Each EVALUATOR doc ends with a short Challenges section -- small additions
that fit naturally at that stage.  This document collects the bigger
projects: ones that span multiple docs, require deeper design decisions,
or result in a qualitatively different interpreter.  They assume you have
worked through at least EVALUATOR1, EVALUATOR2, EVALUATOR3, and
PARSER-DOC.

Unless a project says otherwise, build on the **IttyBittyLisp3** looping
evaluator -- it is the most approachable base to extend.  **IttyBittyLisp4**
(the CEK machine) is specialized: modifying it takes a working understanding
of the CEK model, so reach for it only where a project explicitly needs
first-class continuations -- the `call/cc` and `dynamic-wind` projects.  (The
CEK machine's reified `K` stack is the whole reason it exists.)

The **Starter Ladder** below is the gentle exception to "bigger projects": four
small, closely related special forms, sequenced so each one models the next.
It is the recommended on-ramp -- start here before the larger numbered projects.


## Starter Ladder: New Special Forms

*Requires: EVALUATOR3 (the looping evaluator).*

These four forms are individually small, but they are arranged deliberately:
each one is modeled by a form you already have, and each leaves you better
equipped for the next.  Work them in order.  The first is engineered to be a
fast, unambiguous win -- the point of the ladder is to get you over the first
hump (*"I changed the interpreter and it worked"*) before anything asks you to
think hard.

One thing to notice up front: every one of these forms has a *body* -- a
sequence of expressions whose last value is the result.  That is just an
implicit `begin`, the very first compound form you met back in EVALUATOR1.  You
already understand the engine of all four; you are only changing how the body
gets *selected*.  Build them on **IttyBittyLisp3**.

*A tidy-up worth doing early (not really a project):* that body-sequencing loop
-- `for sub in body[:-1]: lEval(sub, E)` then `C = body[-1]` -- is already copied
into `lambda`, `let`, and `begin`, and the forms below will copy it again.  Once
a form or two has put the pattern in your fingers, pull it into a single
`eval_sequence(body, E)` helper and point every body at it.  It is the kind of
consolidation worth doing while the duplication is still small enough to see all
at once -- a five-minute refactor rather than a challenge, but a good habit to
form on a codebase you are about to grow.

(If you have done Project 3, every form here can *also* be written as a macro
with no evaluator change at all.  Doing one of them both ways -- once as a
Python special form, once as a derived macro -- is its own lesson about which
constructs are *core* and which are *sugar*.  See the macro versions of `when`
and `let*` in Project 3.)


### A. `when` and `unless` -- the fast win

A one-armed conditional with an implicit `begin` body:

- `(when test body...)` runs the body only if `test` is true.
- `(unless test body...)` is the mirror: the body runs only if `test` is false.

When the test fails, the result is unspecified (return `'()` or `#f` -- your
call).

**Model: `if`.** You already have `if` (pick one of two branches) and
body-sequencing (the `for subExpr in body[:-1]` / `C = body[-1]` loop inside
`let` and `begin`).  `when` is just those two pieces glued together: evaluate
the test; if it holds, sequence the body with the last form in tail position
(reassign `C`, `continue`); otherwise return the unspecified value.  It is a
~4-line `elif` arm, and it runs the moment you type it:

```scheme
(when (= a 2) (set! b 10) (+ b 5))     ; => 15 when a is 2, unspecified otherwise
```

That is the whole exercise.  The win is supposed to be quick -- take it, then
climb.


### B. `cond` -- your first real language feature

`cond` is the multi-armed generalization of `if`: a list of clauses, each
`(test body...)`, tried top to bottom.  The first clause whose test is true
runs its body (an implicit `begin`) and supplies the value.  An `else` clause,
which must come last, is the catch-all.

```scheme
(cond ((= n 0) 'zero)
      ((< n 0) 'negative)
      (else    'positive))
```

**Model: `if`.** A `cond` is just nested `if`s -- each clause's "miss" falls
through to the next clause.  Walk the clauses; evaluate each test in `E`; on the
first true test, sequence that clause's body and put its tail form in `C`, then
`continue` (a tail position -- no stack growth).  If nothing matches and there
is no `else`, return the unspecified value.

This is the rung where it stops feeling like a toy edit and starts feeling like
you are implementing *the language*: multiway branching is what you actually
reach for when writing Scheme, and `cond` becomes useful in your own test
programs the instant it works.

*Stretch:* real `cond` also allows a bare-test clause `(test)` -- which yields
the test's own value -- and the `(test => proc)` form, which applies `proc` to
the test's value.  Add the bare-test form first (easy); leave `=>` for later.


### C. `let*` -- sequential scope

You have `let`, which evaluates *all* of its init expressions in the *outer*
environment and then opens a single new scope holding them -- parallel binding.
`let*` is the sequential cousin: each binding can see the ones before it.

```scheme
(let* ((a 1)
       (b (+ a 1))      ; b sees a
       (c (+ b 1)))     ; c sees b
  (+ a b c))            ; => 6
```

In a plain `let`, the init `(+ a 1)` for `b` would fail -- `a` is not visible
yet.  `let*` fixes that by threading a *fresh scope per binding*: bind `a`,
evaluate `b`'s init *in the scope that now holds `a`*, bind `b`, and so on.

**Model: `let`.** The diff is small; the concept -- *when* each init runs and
*what scope it sees* -- is the lesson.  Two clean routes:

1. **Nest.** Rewrite `(let* (b1 b2 b3) body)` as
   `(let (b1) (let (b2) (let (b3) body)))` and hand it to your existing `let`.
   (This is exactly the macro shown in Project 3.)
2. **Loop.** Walk the bindings, extending `E` by one binding at a time, and
   evaluate each init in the current `E` before adding its name.

Either way you will feel the difference between *parallel* and *sequential*
binding directly -- the same axis that will separate `let*` from `letrec` next.


### D. `letrec` -- tying the recursive knot

`letrec` goes one step past `let*`: every binding can refer to *all* the
others, including ones defined later.  That is what mutual recursion needs.

```scheme
(letrec ((even? (lambda (n) (if (= n 0) #t (odd?  (- n 1)))))
         (odd?  (lambda (n) (if (= n 0) #f (even? (- n 1))))))
  (even? 10))            ; => #t
```

Neither `let` nor `let*` can express this: with `let`, `even?`'s body cannot
see `odd?`; with `let*`, `even?` (defined first) cannot see `odd?` (defined
later).

**Discover the trap first.** Try implementing `letrec` the way `let` works --
evaluate the inits, *then* build the scope.  It fails: each `lambda` closes over
an environment that does not yet contain its siblings, so `even?` can never find
`odd?`.  *Introduction order* is the whole problem.

**Model: `let`, with the order inverted.**

1. Open a new frame with all the names present but unassigned (pre-seed the keys
   with a placeholder).
2. Evaluate each init expression *in that frame*.
3. Assign each result back into the frame.
4. Run the body in the frame.

Because every `lambda` now closes over a frame that *already holds all the
names*, mutual recursion works: by the time `even?` is ever *called*, `odd?`
has been filled in.  This is the rung that teaches the recursive knot -- a
closure and the environment it captures can be mutually referential because the
environment is a *mutable object that exists before the closures' bodies ever
run*.  That single insight is also why a top-level `define` of two
mutually-recursive functions Just Works: `letrec` is that same mechanism, given
a scope.


## 1. `define` vs `set!`

*Requires: EVALUATOR2 (the `Environment` chain).*

The toy interpreters use a single, lenient `set!`: assigning a name that is
not bound anywhere creates it (in the global scope).  That one form quietly
does the job of two distinct Scheme operations:

- **`define`** *introduces* a binding in the current scope.
- **`set!`** *assigns* an existing binding -- and in real Scheme it is an
  **error** if the name is unbound.

Conflating them is fine for a demo: it lets a top-level `(set! a 5)` work
without a separate definition form.  But it hides a real distinction, and it
lets a mistyped `set!` silently create a global instead of reporting the
mistake.  Splitting them is a small, instructive refactor.

**Discover the trap first.** Try implementing a binding form (`let`, or the
new `define`) by calling `set!` to bind each name in the fresh scope.  It
will not work: `set!` walks *outward* looking for an existing binding, finds
none, and writes to global -- so your "local" binding leaks.  *Introduction*
and *assignment* fundamentally need different mechanics:

- assignment walks the scope chain and mutates the nearest existing binding
- introduction writes unconditionally into one specific scope

**The refactor:**

1. Give the environment a method that binds in *this* scope only -- no chain
   walk, no global fallback (e.g. `defLocal(name, value)`).
2. Add a `define` special form that uses it to bind in the current env.
3. Make `set!` strict: walk the chain, and if the name is unbound anywhere,
   raise an error instead of creating a global.
4. Point every binding form at the introduction path.  Note that the
   environment *constructor* (`Environment(parent, bindings=...)`) is already an
   "introduce these bindings" operation -- which is why `let` and the
   function-call argument binding work without `set!`.

Then decide the design question: at the top level (no enclosing scope),
should a bare `(set! x 1)` on an unbound `x` raise, or implicitly `define`?
Both are defensible -- many REPLs are lenient at the top level but strict
inside functions.  Choosing, and justifying the choice, is the design half
of the exercise.


## 2. A Working REPL

*Requires: PARSER-DOC + EVALUATOR2 or later.*

Wire the parser and evaluator together into a read-eval-print loop:

```
while True:
    source = input('> ')
    if source == 'quit':  break
    ast    = parse( source )
    result = lEval( ast, global_env )
    print( result )
```

This is only a skeleton.  Making it pleasant to use requires:

- **Multi-line input.** A single `input()` call drops the expression if the
  user hits Enter before closing a paren.  Count unmatched `(` and `)` as
  you read; keep prompting (with a continuation prompt like `  `) until
  the expression is balanced.

- **Error recovery.** A bad expression should print an error and return to
  the prompt -- not crash the REPL.  Wrap the eval in a `try/except` that
  catches `NameError`, `TypeError`, and your own interpreter exceptions.

- **History.** Python's `readline` module gives you arrow-key history for
  free on Linux/macOS.  Wire it in: `import readline` is enough.

- **Pretty-printing.** A flat Python `repr` is readable for small values
  but ugly for nested lists.  Write a `pprint` function that formats lists
  as `(a b c)`, indenting nested lists when they exceed a width limit.

Once the REPL is working you have a complete, self-contained Lisp
interpreter in roughly 80 lines.


## 3. Macros

*Requires: EVALUATOR2 (closures).*

A **macro** is a function that takes unevaluated AST and returns a new AST
that is then evaluated in its place.  It runs at expansion time, not
call time, so it can generate any code the evaluator understands.

Add a `define-macro` special form that stores the transformer function, and
an expansion step that runs before evaluation.  (`define-macro` is the
non-standard *procedural* macro found in many Schemes; it's the simplest macro
to build.  Standard Scheme's `define-syntax` / `syntax-rules` is covered below.)

```python
macros = {}

def expand( expr ):
    if not isinstance( expr, list ) or len( expr ) == 0:
        return expr
    head = expr[0]
    if head in macros:
        transformer = macros[head]
        return expand( transformer( expr[1:] ) )   # expand recursively
    return [ expand( sub ) for sub in expr ]

def lEval( expr, env ):
    expr = expand( expr )
    ...                        # existing evaluator unchanged
```

With `define-macro` you can add `when`, `unless`, `and`, `or`, `cond`, and
`let*` entirely in Scheme, without touching the Python evaluator.  The macro's
formals use a dotted rest parameter (`. body`), just like a variadic `lambda`:

```scheme
(define-macro (when condition . body)
  (list 'if condition (cons 'begin body) '()))

(define-macro (let* bindings . body)
  (if (null? bindings)
      (cons 'begin body)
      (list 'let (list (car bindings))
            (cons* 'let* (cdr bindings) body))))
```

The macro system is the point where the language becomes self-extending.
Nearly every high-level construct in Scheme -- `when`, `unless`, `cond`,
`case`, `do`, `let*` -- is (or can be) a derived form built this way.

**Hygiene (the real Scheme story)**: the `define-macro` above is *unhygienic* --
a name it introduces (say a temporary) can accidentally capture, or be captured
by, a name in the user's code.  Standard Scheme avoids this with `define-syntax`
and `syntax-rules`, a *hygienic*, pattern-based macro system that renames
introduced identifiers automatically.  Implementing `syntax-rules` (pattern
matching + hygiene) is a substantial project of its own, and the natural
next step after `define-macro`.

**Harder extension**: implement `quasiquote` (`` ` ``) and `unquote`
(`,`) in the parser (see the backquote challenge in PARSER-DOC), then
rewrite the macros above using template syntax instead of `list`/`cons`
calls.  The difference in readability is striking.


## 4. Tail Calls in the Macro Expander

*Requires: Project 3 (Macros) + EVALUATOR3 (TCO).*

Once you have both macros and the looping evaluator, there is a subtle
interaction: the expander must run before each loop iteration, not just
once at the top level.  Move the `expand` call inside the `while True:`
loop:

```python
def lEval( expr, env ):
    while True:
        expr = expand( expr )        # expand at every iteration
        ...
```

Without this, a macro that expands to a tail call will not get TCO
because the expansion happens before the loop and the tail call appears
as a raw function application the first time through.

Once the placement is correct, write a macro that wraps a tail-recursive
loop and verify that it runs a million iterations without stack overflow.


## 5. A Self-Hosting Evaluator

*Requires: Project 3 (Macros) + a reasonably complete primitive set.*

Implement `lEval` *in the Lisp you have built*:

```lisp
(define (leval expr env)
  (cond
    ((symbol? expr)  (lookup expr env))
    ((not (list? expr))  expr)
    ((eq? (car expr) 'quote)  (cadr expr))
    ((eq? (car expr) 'if)
     (if (leval (cadr expr) env)
         (leval (caddr expr) env)
         (leval (cadddr expr) env)))
    (else
     (let ((fn   (leval (car expr) env))
           (args (map (lambda (a) (leval a env)) (cdr expr))))
       (apply fn args)))))
```

This is the **metacircular evaluator** -- a Lisp evaluator written in
Lisp.  It is not a curiosity: it is the canonical demonstration that
the language is powerful enough to describe itself.

Getting it to work requires:
- A complete environment representation in Lisp (association lists or
  a Lisp-level hash table)
- A `lookup` function written in Lisp that navigates that representation.
  For an association-list environment it is a short recursive search;
  for a dict environment it collapses to `(at sym env)`.  Either way,
  `lookup` is not a primitive -- it is part of the meta-evaluator you
  are building.
- Lisp-level `apply` that can call both primitive and user-defined
  functions
- Bootstrapping: the meta-evaluator runs on top of your Python
  interpreter, which provides the primitive operations

Once it works, you can add features to the meta-evaluator (new special
forms, a different scoping rule) without touching any Python at all.


## 6. A Bytecode Compiler

*Requires: EVALUATOR3 (TCO) + a complete evaluator.*

A tree-walk interpreter traverses the AST on every evaluation.  A
**bytecode compiler** converts the AST once to a flat sequence of
simple instructions, then a small loop (the virtual machine) executes
them.  The VM loop is faster because it avoids the overhead of recursive
function calls and `isinstance` checks on every step.

Design a minimal instruction set:

| Instruction | Effect |
|---|---|
| `LOAD_CONST val` | push a literal value |
| `LOAD_VAR name` | look up `name` in env, push result |
| `STORE_VAR name` | pop top of stack, bind to `name` in env |
| `CALL n` | pop `n` args and the function, push result |
| `JUMP_IF_FALSE offset` | pop condition; if false, skip `offset` instructions |
| `JUMP offset` | unconditional jump |
| `RETURN` | pop and return the top of stack |

A compiler pass walks the AST and emits instructions:

```python
def compile_expr( expr, code ):
    if isinstance( expr, int ):
        code.append( ('LOAD_CONST', expr) )
    elif isinstance( expr, str ):
        code.append( ('LOAD_VAR', expr) )
    elif expr[0] == 'if':
        compile_expr( expr[1], code )          # condition
        j = len( code )
        code.append( ('JUMP_IF_FALSE', None) ) # placeholder
        compile_expr( expr[2], code )          # then
        code[j] = ('JUMP_IF_FALSE', len(code))
        ...
```

The VM is a `while` loop over the instruction list with a program counter
and an operand stack.  It is simpler than the tree-walk evaluator --
each instruction does exactly one thing.

**Further work**: implement a simple peephole optimizer that eliminates
redundant `LOAD`/`STORE` pairs, or add a `TAIL_CALL` instruction that
replaces the current frame instead of pushing a new one.


## 7. Continuations and `call/cc`

*Requires: EVALUATOR4-DOC (the CEK machine).*

A **continuation** is the rest of the computation from some point forward.
In a CEK machine the continuation is already reified as the `K` stack --
`call/cc` just captures it.

```lisp
; escape from a deep computation
(call/cc (lambda (k)
  (search-tree tree (lambda (node)
    (if (found? node)
        (k node)     ; jump out immediately with the answer
        #f)))))
```

The implementation:
- `call/cc` calls its argument with a special `Continuation` object
- Invoking the `Continuation` replaces the current `K` with the saved one
  and injects the argument as the current value -- an immediate long jump

A subtlety: the continuation captured by `call/cc` is an **escape**
continuation only if you never invoke it after the call/cc expression
has returned.  Full, re-invocable continuations (which can re-enter
completed computations) require that the entire continuation stack be
*copied*, not just referenced.  The copying version enables coroutines,
generators, and `amb`.

This interpreter implements **full re-invocable continuations**: K is
deep-copied at capture time (mutable frames get new instances; immutable
frames are shared).  Start with escape continuations to get the basic
mechanism working, then add the per-frame `copy()` protocol to reach
full continuations.


## 8. `dynamic-wind`

*Requires: Project 7 (call/cc).*

`dynamic-wind` is the control mechanism that ensures cleanup actions run
even when a continuation escape jumps past them -- the equivalent of
`try/finally` but interacting correctly with `call/cc`.

```lisp
(dynamic-wind
  (lambda () (open-resource))    ; before thunk -- runs on entry
  (lambda () (use-resource))     ; body thunk
  (lambda () (close-resource)))  ; after thunk -- runs on exit, always
```

The tricky part: if a continuation captured inside the body thunk is
invoked from outside, the *before* thunk must run again on re-entry and
the *after* thunk must run on re-exit.  This requires the VM to maintain
a "wind stack" that is consulted every time a continuation is invoked
or escapes.

`dynamic-wind` is the foundation for `with-exception-handler`,
`parameterize`, and composable resource management.  Implementing it
correctly is one of the harder exercises in the series -- but the design
insight (continuations need to know about dynamic extent) applies to
every serious language implementation.
