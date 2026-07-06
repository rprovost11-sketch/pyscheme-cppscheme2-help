# Projects

*Larger challenges for the IttyBitty Lisp series.*

Each chapter ends with small challenges that fit naturally at that stage. This
document collects the bigger projects: ones that span several chapters, ask for a
real design decision, or change the interpreter in a qualitative way. They assume
you have worked through the main chapters, at least Chapters 1 through 3 and the
parser in Chapter 7.

Unless a project says otherwise, build on the **Chapter 3** looping evaluator
(`IttyBittyLisp3.py`). It is the most approachable base to extend: it has the full
early language, and its single `while` loop is easy to add a form to. The CEK
machine of Chapters 4 and 5 is more specialized; reach for it (specifically the
full-language `IttyBittyLisp5.py`) only where a project actually needs first-class
continuations.

A few projects you might expect here already live elsewhere, because a chapter had
the right place for them:

- a **REPL** with multi-line input and error recovery: Chapter 1, §1.6 and its
  challenges;
- `when`, `unless`, `cond`, `let*`: chapter challenges (Chapters 1 and 2);
- a **bytecode compiler**: that is Chapter 6, built in full;
- **`call/cc`** and first-class continuations: the stretch challenge in Chapter 4,
  which is where reifying `K` first makes it reachable.

What follows are the projects that do not have a chapter home of their own.


## 1. `define` versus `set!`

*Requires: Chapter 2 (the Environment chain).*

The toy interpreters use a single, lenient `set!`: assigning a name that is bound
nowhere creates it in the global scope. That one form quietly does the job of two
distinct Scheme operations:

- **`define`** *introduces* a binding in the current scope.
- **`set!`** *assigns* to a binding that already exists, and in real Scheme it is
  an **error** to `set!` a name that was never defined.

Conflating them is fine for a demo: it lets a top-level `(set! a 5)` work without a
separate definition form. But it hides a real distinction, and it lets a mistyped
`set!` silently create a global instead of reporting the mistake.

**Find the trap first.** Try to add a binding form (a new `define`, or even `let`)
by calling `set!` to bind each name in the fresh scope. It will not work: `set!`
walks *outward* looking for an existing binding, finds none, and writes to the
global scope, so your "local" binding leaks straight past the scope you meant it
for. Introduction and assignment genuinely need different mechanics:

- assignment walks the scope chain and changes the nearest existing binding;
- introduction writes unconditionally into one specific scope.

**The refactor:**

1. Give `Environment` a method that binds in *this* scope only, with no chain walk
   and no global fallback (call it `defLocal(name, value)`).
2. Add a `define` special form that uses it to bind in the current environment.
3. Make `set!` strict: walk the chain, and if the name is bound nowhere, raise an
   error rather than creating a global.
4. Point every binding form at the introduction path. Notice that the environment
   *constructor* (`Environment(parent, bindings=...)`) is already an "introduce
   these names" operation, which is why `let` and function-call argument binding
   worked without `set!` all along.

Then decide the design question: at the top level, with no enclosing scope, should
a bare `(set! x 1)` on an unbound `x` raise, or quietly `define`? Both are
defensible; many REPLs are lenient at the top level and strict inside functions.
Choosing, and being able to say why, is the real point of the project.


## 2. `letrec` and the recursive knot

*Requires: Chapter 2. Easier if you have already added `let*` (a Chapter 2
challenge).*

`let` binds names in parallel: every initializer is evaluated in the *outer* scope,
so the bindings cannot see one another. `let*` binds them in sequence, so each can
see the ones before it. `letrec` goes one step further: every binding can refer to
*all* the others, including ones defined later. That is exactly what mutual
recursion needs:

```scheme
(letrec ((even? (lambda (n) (if (= n 0) #t (odd?  (- n 1)))))
         (odd?  (lambda (n) (if (= n 0) #f (even? (- n 1))))))
  (even? 10))            ; => #t
```

Neither `let` nor `let*` can express this. With `let`, `even?`'s body cannot see
`odd?` at all; with `let*`, `even?` is defined first and so still cannot see the
later `odd?`.

**Find the trap first.** Try building `letrec` the way `let` works: evaluate the
initializers, *then* create the scope. It fails, and the failure is worth feeling.
Each `lambda` closes over the environment it was made in, and at the moment you
evaluate `even?`'s initializer that environment does not yet contain `odd?`, so
`even?` can never find it. The *order* of introduction is the whole problem.

**The fix inverts that order:**

1. Open a new scope with all the names present but unassigned (seed each key with a
   placeholder).
2. Evaluate each initializer *in that scope*.
3. Assign each result back into the scope.
4. Run the body in the scope.

Because every `lambda` now closes over a scope that *already holds all the names*,
mutual recursion works: by the time `even?` is ever called, `odd?` has been filled
in. This is the project that teaches the recursive knot, and the insight is one
sentence: a closure and the environment it captures can refer to each other because
the environment is a *mutable object that exists before the closures' bodies ever
run*. That same mechanism is why a pair of top-level `define`s of mutually
recursive functions Just Works; `letrec` is that mechanism handed a local scope.


## 3. A `Symbol` type

*Requires: Chapter 2, and the parser (Chapter 7).*

Look at how our interpreter represents a name. A symbol like `x` or `+` is just a
Python string; the parser's `atom` returns the raw token string when it is not a
number, and `lEval` treats any Python `str` as a name to look up. That works, but
it spends a resource we will want back: Python's `str` is now *taken*. There is no
way to have a Lisp **string** as a piece of data, because a string and a symbol
would both be Python `str` and the interpreter could not tell them apart.

This project frees `str` by giving symbols a type of their own. It is pure
representation work; the language the reader sees does not change at all.

**The change:**

1. Add a tiny class:

   ```python
   class Symbol:
       def __init__( self, name ):
           self.name = name
   ```

2. In the parser, have `atom` return `Symbol(token)` instead of the bare string for
   anything that is not a number.
3. In `lEval`, the name case changes from `isinstance(expr, str)` to
   `isinstance(expr, Symbol)`, looking up `expr.name`.
4. `lisp_str` renders a `Symbol` as its `name`.
5. The `=` primitive compares two symbols by name (two `Symbol`s are equal when
   their names match).

**The one real decision** is what happens to `#t` and `#f`, which the toys also
represent as strings. Once symbols are their own type, you have to choose: are the
booleans two special `Symbol`s named `"#t"` and `"#f"`, or a dedicated boolean
value of their own? Either works; make the choice and follow it through every place
the old code wrote `'#f'`.

None of this changes a single program. What it buys you is the next project: with
`str` no longer standing in for symbols, a Python string can finally *be* a Lisp
string.


## 4. Strings

*Requires: Project 3 (the `Symbol` type).*

Now that `str` is free, add a string data type. Two things stand in the way, and
the first is the parser.

**Find the trap first.** Try to type a string at the REPL and watch the tokenizer
shred it:

```
>>> (display "hello world")
```

The tokenizer works by padding `(` and `)` with spaces and splitting on
whitespace, so `"hello world"` splits into two tokens, `"hello` and `world"`, and
even `"hello"` (no space) comes back as the *symbol* `"hello"`, quotation marks and
all. Whitespace-splitting can never keep a string in one piece, because a string is
the one kind of token that is allowed to contain spaces.

**So the tokenizer needs to actually read characters.** Scan the source left to
right; most of the time you behave exactly as before, but when you hit a `"`,
switch into "reading a string" mode and consume characters (including spaces) until
the closing `"`, emitting the whole thing as a single token. That token carries its
quotes, so the reader can recognize it as a string literal rather than a symbol or
a number.

**The representation.** With Project 3 done, a Lisp string is simply a Python
`str`. The reader turns a `"..."` token into that string (with the quotes stripped);
`lEval` returns any Python `str` as itself, the way it returns a number; and
`lisp_str` prints one *with* quotes, so output round-trips back to source.

**Then give strings something to do.** Add a few primitives, each a short Python
function: `string-length`, `string-append`, `substring`, and a `string->symbol` /
`symbol->string` pair that converts between the two types you have now cleanly
separated. That last pair is the quiet reward for Project 3: the conversion is only
*expressible* because the two are finally distinct.


## 5. Macros

*Requires: Chapter 2 (closures); Chapter 3 if you want tail calls to survive
expansion.*

A **macro** is a rule that rewrites code *before* it is evaluated. It takes the
unevaluated expression and returns a new one, and the evaluator runs *that* instead.
Chapter 3's macro challenge showed the two-line hook in the loop and one example;
this project builds the whole thing.

Add a `define-macro` special form that stores a transformer function, and an
expansion pass that runs before evaluation:

```python
macros = {}

def expand( expr ):
    if not isinstance( expr, list ) or len( expr ) == 0:
        return expr
    head = expr[0]
    if head in macros:
        transformer = macros[head]
        return expand( transformer( expr[1:] ) )   # expand the result, too
    return [ expand( sub ) for sub in expr ]        # otherwise, expand the parts
```

Then expand once, at the top of `lEval`, before the existing dispatch runs. With
`define-macro` in hand, whole families of forms become things you write *in* the
language instead of new `elif` arms in Python:

```scheme
(define-macro (when test . body)
  (list 'if test (cons 'begin body)))

(define-macro (unless test . body)
  (list 'if test '() (cons 'begin body)))
```

That `(message . body)` shape is a dotted rest parameter, the same variadic idea as
a rest-argument `lambda`: `test` binds the first form, `body` binds the list of the
rest. Build `when`, `unless`, `cond`, and `let*` this way and watch the evaluator
*not grow*: every one of them is now a rewrite into forms the machine already has.

Three natural extensions, in rising order of difficulty:

- **Tail calls through the expander.** If you are on the Chapter 3 looping
  evaluator, a macro whose expansion ends in a tail call will only stay in constant
  space if the expansion happens *inside* the loop, not once at the top. Move the
  `expand` call to the start of each loop iteration and confirm a macro-wrapped loop
  runs a million times without growing the stack.
- **Quasiquote.** Writing macro output with `list` and `cons` is clumsy. Add
  quasiquote (`` ` ``), unquote (`,`), and unquote-splicing (`,@`) in the parser,
  then rewrite the macros above with template syntax: `` `(if ,test (begin ,@body)) ``
  says the same thing as the `list`/`cons` version, far more readably.
- **Hygiene.** The `define-macro` here is *unhygienic*: a name the macro introduces
  can accidentally capture, or be captured by, a name in the user's code. Standard
  Scheme avoids this with `define-syntax` and `syntax-rules`, a pattern-based system
  that renames introduced names automatically. Implementing it is a substantial
  project in its own right, and the natural sequel to this one.


## 6. The metacircular evaluator

*Requires: `cond` and the list primitives (`car`, `cdr`, `cons`, `null?`, `pair?`,
`eq?`, `symbol?`) from the Chapter 1 and 2 challenges, plus `map`.*

Here is the project that closes the loop the whole series has been drawing. You have
spent seven chapters writing an evaluator *in Python*. Now write one *in the Lisp
you built* and run it on its own interpreter. It has a name, the **metacircular
evaluator**: a Lisp evaluator written in Lisp.

The surprising part is how little there is to write, and why. Look back at Chapter
2's `lEval`: dispatch on the shape of the expression, look a symbol up, handle
`quote` and `if`, and for anything else evaluate the parts and apply. That is not a
big program. And because the Lisp you built has functions, `cond`, and lists, it can
say all of that about *itself*, almost line for line:

```scheme
(define (leval expr env)
  (cond
    ((symbol? expr)      (lookup expr env))             ; a name
    ((not (pair? expr))  expr)                           ; a number: itself
    ((eq? (car expr) 'quote)  (car (cdr expr)))          ; quote
    ((eq? (car expr) 'if)
     (if (leval (car (cdr expr)) env)
         (leval (car (cdr (cdr expr))) env)
         (leval (car (cdr (cdr (cdr expr)))) env)))      ; if
    ((eq? (car expr) 'lambda)
     (make-closure (car (cdr expr)) (car (cdr (cdr expr))) env))  ; lambda
    (else
     (lapply (leval (car expr) env)                      ; a call: eval operator...
             (map (lambda (a) (leval a env)) (cdr expr)))))) ; ...and operands
```

Set that beside Chapter 2's Python `lEval` and the two are the same evaluator in two
languages. That correspondence *is* the payoff: the language is simple enough to
describe itself, and you can see the description fits on a page. This is not a
curiosity. It is the canonical demonstration, from the heart of the Lisp tradition,
that a language can be powerful enough to hold its own meaning.

To make it run you supply the few pieces the skeleton leans on, each short:

- an **environment** represented as Lisp data (a list of name-value pairs is enough),
  and a `lookup` that searches it;
- `make-closure`, which bundles parameters, body, and environment into a value (a
  tagged list), and `lapply`, which either calls a primitive or binds a closure's
  parameters and calls `leval` on its body;
- a starting environment holding a handful of primitives (`+`, `car`, `cons`, and
  friends), taken from the interpreter underneath.

That is the whole engine, riding on the Python interpreter beneath it exactly as
that one rides on Python.

**Then extend it, which is where the real lesson is.** Once it runs, add to the
*meta*-evaluator without touching any Python:

- add a special form (`begin`, or `let`) as one more `cond` arm in `leval`;
- change its scoping rule and watch the language it evaluates change with it;
- feed `leval` its own source and evaluate a program two interpreters deep.

Each of those changes the language from *inside* the language. That is the thing the
metacircular evaluator is really for, and it is only a few lines away now because
the evaluator was small from the very first chapter.
