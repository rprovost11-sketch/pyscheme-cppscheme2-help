# Chapter 1: A Tiny Interpreter for a Tiny Lisp

An **interpreter** is a program that runs another program.  When you type
`python foo.py`, Python is acting as an interpreter: it reads the instructions in
`foo.py` and carries them out.  In this series we build an interpreter of our
own (a small one, for a small Lisp) written in Python, and we grow it one idea
at a time.  Each version is short enough to read start to finish and hold in your
head all at once.

This first chapter builds the smallest interpreter that actually runs programs.
Before we write a line of it, we need to be clear about two things: *what
language it runs*, and *how a program in that language is handed to it*.


## 1.1 The language we're building in this chapter

Our Lisp starts tiny.  A whole program is a single **expression**, and to *run*
the program is to **evaluate** that expression: to work out the value it
**returns**.  Every expression returns a value; that is the whole point of
evaluating one.  Expressions come in just a few shapes:

- **Data that stands for itself.**  Numbers like `42` or `3.14`, and the two
  booleans `#t` (true) and `#f` (false).  Evaluating one of these just returns
  it unchanged: `42` evaluates to `42`.

- **A name.**  Something like `a` or `x`.  Evaluating a name looks up the value
  currently stored under it and returns that.

- **A parenthesized list**, written `(head ...)`, the same idea as a Python
  function call `f(a, b)`, but with the opening parenthesis slid to the front: `(f a b)`.
  The `head`, the first element, decides what happens, and it comes in two
  kinds:
  - A small set of **special forms** with their own built-in rules: `set!`,
    `if`, `begin`, and `quote`.  (They are "special" because each bends the
    normal rules in some way.  More on that when we build them.)
  - Anything else is a **function call** to a built-in operation: `+`, `-`, `*`, `=`,
    `<`, or `print`.  Even arithmetic follows this operation-first shape, so
    `(+ 1 2)` is what other languages spell `1 + 2`.

These lists nest.  A list's parts can themselves be lists, to any depth, and
that nesting is where evaluation gets its order: to evaluate a function call, we
work out its parts first: the innermost lists return their values, and those values
become the parts of the list around them, working outward.  In `(+ (- 10 7) a)`, the
inner `(- 10 7)` returns `3` before `+` ever runs.  (The special forms are
exactly the cases that *don't* follow this inner-first rule: `quote`, for one,
evaluates nothing inside it, which is what earns it the name.)

That is the entire language for this chapter.  Here is a short program in it,
with the value each line returns:

```scheme
(set! a (+ 1 1))     ; store 2 under the name a; returns 2
(+ (- 10 7) a)       ; 3 + 2                        =>  5
(if (= a 2)          ; if a equals 2 ...
    (+ a 1)          ;   ... evaluate this branch
    (- a 1))         ;   ... otherwise this one     =>  3
(begin (set! b 10)   ; evaluate these in order;
       (+ b 5))      ; returns the last one         =>  15
(quote (a b c))      ; returns the list unevaluated =>  (a b c)
```

The four special forms are the whole grammar you need to read the rest of the
chapter:

- **`set!`** stores a value under a name (`(set! a 2)`), and returns that value.
- **`if`** evaluates its test, then evaluates *one* of its two branches (never
  both) and returns that branch's value.
- **`begin`** evaluates a sequence of expressions in order and returns the value
  of the last.
- **`quote`** returns its argument *without* evaluating it: the only way to
  talk about a list as data rather than as something to run.

Small as it is, that is a real language, and the rest of this chapter builds the
interpreter that runs it.


## 1.2 How a program is represented: the syntax tree

We have described what our programs *look like* as source code.  But an
interpreter does not work on raw source code; it works on **structure**.
Consider this ordinary Python line:

```python
x = 1 + 2 * 3
```

Written out, it is just a flat row of characters.  Its *meaning*, though, is a
nested structure, a **tree**:

```
      assign
      /    \
     x    add
          /  \
         1   mul
             /  \
            2    3
```

Read the tree from the bottom: multiply `2` and `3`, add `1` to the result,
assign that to `x`.  Notice that the tree records something the flat text only
*implies*: that `*` happens before `+`.  Recovering that structure from the raw
characters (deciding what groups with what) is a real job, and the program
that does it is called a **parser**.  In Python you never see this tree, but it
is built every time your code runs.

This tree has a name: the **abstract syntax tree**, or **AST**.  *Syntax tree*
because it is the program's syntax arranged as a tree; *abstract* because it
throws away the incidental surface details (the spaces, the exact punctuation)
and keeps only the meaningful structure.  Each branching point is a **node** (an operation together with its parts) and each tip is a **leaf**: an **atom**, a
single indivisible piece like the number `1` or the name `x`, with nothing
further inside.

Now here is the same computation in our Lisp:

```scheme
(set! x (+ 1 (* 2 3)))
```

Draw *its* tree: the head of each list is the operation, the rest are its parts:

```
      set!
      /  \
     x    +
         / \
        1   *
           / \
          2   3
```

It is the same tree, but compare the two pieces of source code that produced it.
In the Python line `x = 1 + 2 * 3`, the tree is hidden: a parser has to *recover*
the grouping, working out that `*` binds tighter than `+`.  In the Lisp source
`(set! x (+ 1 (* 2 3)))`, there is nothing to recover: the parentheses spell the
grouping out directly, because we wrote it by hand.  **In Lisp, the source code
you write already is the tree.**  That property is the whole reason Lisp is built
out of parentheses, and it pays off immediately for us.

Because the written form already is the tree, we can represent an AST in Python
with the most direct structure imaginable: a nested list, where a name is a
Python string, a number is a Python number, and the two booleans are just the
strings `'#t'` and `'#f'`:

```python
['set!', 'x', ['+', 1, ['*', 2, 3]]]
```

That list *is* the tree above, written as data.  So in this chapter we skip the
parser entirely: instead of turning source code into a tree, we simply hand-write
the tree as nested Python lists and feed those straight to the interpreter.  (A real
parser, which turns the string `"(set! x (+ 1 (* 2 3)))"` into that list, is a
chapter of its own later on.)  Everything the interpreter does, from here on, is
walk a tree like this one.


## 1.3 The evaluator

Everything the interpreter does happens in one function.  Call it `lEval`.  It
takes an expression (one of those nested-list trees) and returns its value:

```python
value = lEval( expr, env )
```

The second argument, `env`, is the **environment**: the place where names' values
are kept.  It is just a Python dictionary mapping each name to its current value,
so after `(set! a 2)` the environment holds `{'a': 2}`.  When we said back in the
language tour that `set!` "stores a value under a name" and that evaluating a name
"looks it up," the environment is that store.  Every call to `lEval` is handed
the environment so it can read names out of it and write new ones into it.  (Why
pass `env` at all, when there is only one environment and `lEval` could just
reach for `global_env` directly?  Because evaluation is *always relative to an
environment*: a name like `a` has no value on its own: only a value *in* some
environment.  Making `env` an argument states that plainly: the thing `lEval`
evaluates against is an input, not a fixed global.  It happens that there is just
one environment in this chapter; Chapter 2 is where there start to be many.)

The way `lEval` works is to ask *what shape* the expression is (the very same
handful of shapes from the language tour) and deal with each shape in turn.
Here is the whole thing:

```python
def lEval( expr, env ):
    if expr in ('#t', '#f'):           # a boolean: return it unchanged
        return expr
    elif isinstance(expr, str):        # a name: look up its value in env
        return env[expr]
    elif not isinstance(expr, list):   # a number (any non-list): return unchanged
        return expr

    # From here down, expr is a list -- a special form or a function call.

    elif expr[0] == 'set!':            # (set! name value)
        name, valExpr = expr[1:]
        val = lEval(valExpr, env)      # evaluate the value ...
        env[name] = val                # ... and store it under the name
        return val

    elif expr[0] == 'if':              # (if test then else)
        condExpr, thenExpr, elseExpr = expr[1:]
        condVal = lEval(condExpr, env)
        return lEval(elseExpr if condVal == '#f' else thenExpr, env)

    elif expr[0] == 'begin':           # (begin e1 e2 ... eN)
        for subExpr in expr[1:-1]:     # run all but the last for their effects
            lEval(subExpr, env)
        return lEval(expr[-1], env)    # the value of the last one is the result

    elif expr[0] == 'quote':           # (quote datum)
        return expr[1]                 # hand back datum, evaluating nothing

    else:                              # (op arg1 arg2 ...) -- a function call
        fn, *args = [ lEval(elt, env) for elt in expr ]
        return fn( args )
```

That is the entire interpreter for this chapter.  It is one long
`if` / `elif` / … / `else` chain, so exactly one branch runs on each call:
you can read it straight down as the list of shapes from the language tour.

**The three atom cases** come first.  A boolean returns itself; a number (in
fact anything that is not a list) returns itself; a name is looked up in `env`
and its stored value is returned.  Nothing recursive yet: these are the leaves
of the AST, where the walk bottoms out.

One thing worth noticing before we go on: the *order* of these branches is
load-bearing, not just tidy.  The boolean test has to come before the name test:
`'#t'` and `'#f'` are themselves strings, so putting the `isinstance(expr, str)`
name test first would swallow them.  And all three atom cases have to come before
the compound ones below, which begin by indexing `expr[0]`: that indexing only
makes sense once every non-list has already returned.  Reordering the chain and
re-running the examples is a good way to see what each case is quietly
protecting: the arrangement is for correctness as much as for reading.

**The four special forms** are the interesting part, and they are what the word
*special* has been pointing at all along.  Look at what they have in common: each
one matches on `expr[0]` and then decides *for itself* which of the remaining
parts to evaluate.

- **`set!`** evaluates the value expression, stores the result under the name,
  and returns it.  Notice it does *not* evaluate the name `a`: it uses the name
  as-is, as the dictionary key.  (Evaluating `a` would look up its old value;
  we want the name itself.)
- **`if`** evaluates the test, then evaluates and returns *only* the chosen
  branch.  The other branch is never touched, which is the whole reason `if`
  cannot be an ordinary function call: a function call evaluates all its parts,
  but `if` must skip one.
- **`begin`** evaluates its expressions in order, throwing away every value but
  the last, which it returns.
- **`quote`** evaluates nothing at all: it just returns whatever is written
  inside it, so `(quote (a b c))` returns the list `(a b c)` as plain data.

**The final `else` is the general case: a function call.**  If the head is not one of the
four special names, the expression is an ordinary call, and the usual
rule applies: evaluate *everything*, then combine.  The one line

```python
fn, *args = [ lEval(elt, env) for elt in expr ]
```

runs `lEval` on every element of the list (the head and each argument alike)
and this is the inner-first rule from the language tour, made literal: to
evaluate the list, we first evaluate its parts, and because each part is itself
run through `lEval`, the function is *recursive*: it calls itself on every
subtree.  The head evaluates to a function (one of the primitives), the rest
evaluate to its arguments, and `fn( args )` calls it.  (We hand the whole
argument list to `fn` as a single Python list, so `args[0]` is the first
argument, `args[1]` the second; keeping one uniform shape means every primitive
has the same signature.  The primitives themselves are the next section.)

So, stated plainly at last: a **special form** is a head that `lEval` intercepts
*before* the evaluate-everything rule, so that it can control which of its arguments
get evaluated.  Everything else is a **function call**, and its parts are always all
evaluated first.  That single distinction is the backbone of the whole
interpreter.

### 1.3.1 Watching it run

Suppose the name `a` already holds `2`, and we evaluate `(+ (- 10 7) a)`.  Because
`lEval` calls itself on each subtree, the work unfolds inside-out:

```
lEval( ['+', ['-', 10, 7], 'a'] )          head '+' is not special -> a function call
  evaluate every element:
    lEval( '+' )            -> env['+']  -> the addition function
    lEval( ['-', 10, 7] )                   head '-' is not special -> a function call
      evaluate every element:
        lEval( '-' )        -> env['-']  -> the subtraction function
        lEval( 10 )         -> 10
        lEval( 7 )          -> 7
      call subtract(10, 7)  -> 3
    lEval( 'a' )            -> env['a']  -> 2
  call add(3, 2)            -> 5
```

The inner `(- 10 7)` returns `3` *before* `+` runs (inner-first) and `lEval`
reached that `3` by calling itself, the same way a recursive walk over any tree
bottoms out at its leaves and builds its answer back up.  That is the whole
engine.  The only thing still missing is the primitives behind `+`, `-`, and
`print`: the functions those names look up to.


## 1.4 Primitives

The evaluator's `else` branch looks up the head of a function call, gets a function, and
calls it, but where do `+`, `-`, and `print` actually come from?  They are not
built into `lEval` at all.  They are plain Python functions, kept in the
environment under their names.  That is what a **primitive** is: a built-in
operation the interpreter can *call* but does not itself *define*.  From `lEval`'s
point of view a primitive is nothing special: just a value in the environment
that happens to be callable.

Here is the environment stocked with arithmetic primitives:

```python
def lisp_mul( args ):        # multiply a whole list of numbers together
    result = 1
    for x in args:
        result *= x
    return result

global_env = {
    '+': lambda args: sum( args ),
    '-': lambda args: args[0] - args[1],
    '*': lisp_mul,
    '=': lambda args: '#t' if args[0] == args[1] else '#f',
    '<': lambda args: '#t' if args[0] <  args[1] else '#f',
}
```

Each one is a Python function of a single parameter `args`: the list of
already-evaluated arguments that `lEval`'s `else` branch built.  `+` adds up the
whole list and `*` multiplies all of it together, so both take *any* number of
arguments: `(+ 1 2 3 4)` returns `10`.  `-` subtracts its second argument from its
first, and `=` and `<` compare their two arguments, returning one of our booleans
`'#t'` or `'#f'` (recall that the interpreter represents true and false as those two
strings).  This is the calling convention from §1.3 seen from the other side: `lEval`
hands over the whole argument list, and each primitive takes what it needs: all of
it, for `+` and `*`; the first two, for the rest.

Notice how the labor divides.  `global_env` is the very same dictionary `lEval`
reads names out of, so it holds both the *values* a program stores (like `a`,
after `(set! a 2)`) and the *operations* it calls (like `+`).  To `lEval` there
is no difference between them: looking up `+` is the exact same act as looking up
`a`.  The only thing that sets `+` apart is that the value stored under it happens
to be a Python function, so `lEval`'s `else` branch can call it.

### 1.4.1 Adding your own primitive

Adding a primitive costs nothing in the evaluator: you write a Python function
and drop it into the environment under a name.  Here is `print`, our one
operation that reaches outside the interpreter, to the screen.  A bare `lambda`
can hold only a single expression, and `print` needs to do two things (print,
then return a value), so we write it with `def`:

```python
def lisp_print( args ):
    print( args[0] )        # the side effect: show the value
    return args[0]          # ... then return it, so print composes in a bigger function call

global_env['print'] = lisp_print
```

Two things are worth noticing.  First, `lisp_print` *returns* its argument.  In
many languages printing is a statement that yields nothing; here every function call has a
return value, so `(+ (print 10) 5)` prints `10` and then computes `15`: the
`print` hands its `10` straight back to the `+`.  Second, that function plus one
dictionary entry is the *entire* recipe for extending the language with ordinary
operations: no change to `lEval`, just one more name in the environment.

### 1.4.2 Primitive or special form?

You now have the two ways to add something to the interpreter, and it is worth
being clear about when to reach for each.  It is the distinction §1.3 drew
(special form versus function call) turned into a practical rule.

- If your new operation just computes a result from arguments that are **always
  evaluated** (arithmetic, comparison, printing, taking a list apart), make it a
  **primitive**.  Add a Python function to `global_env` and leave `lEval`
  untouched.  This is the common case, and the cheap one: the evaluator never
  grows.
- If your operation needs to **control evaluation itself** (to skip an argument,
  choose among arguments, or treat one as a name rather than a value), it *cannot*
  be a primitive, because by the time a primitive runs, its arguments have already
  been evaluated.  It must be a **special form**: a new `elif` branch inside
  `lEval`, like `if` or `quote`.

One question settles almost every case: *does the operation need to see any of its
arguments unevaluated?*  If no, it is a primitive; if yes, it is a special form.
(`and` and `or` are the classic trap.  They look like ordinary operations, but
`(or x y)` must not evaluate `y` when `x` is already true, so they have to be
special forms, not primitives.)


## 1.5 Running it

We now have every piece: `lEval`, the recursive walk over the tree, and
`global_env`, the environment stocked with primitives.  To run a program, we hand
its AST and the environment to `lEval`:

```python
lEval( ['set!', 'a', ['+', 1, 1]], global_env )   # returns 2
```

One small nicety is left.  `lEval` returns ordinary Python values: a number
comes back as a Python `int`, and a quoted list comes back as a Python `list`.
If we printed such a list with Python's own `print`, we would see
`['a', 'b', 'c']`, not `(a b c)`.  So we add a helper, `lisp_str`, that renders a
value back in Lisp's parenthesized surface syntax:

```python
def lisp_str( val ):
    if isinstance( val, list ):
        return '(' + ' '.join( lisp_str( x ) for x in val ) + ')'
    if callable( val ):
        return '#<primitive>'
    return str( val )
```

This is the exact inverse of §1.2's observation.  There, we saw that a written
Lisp program already *is* its tree; here we take a tree (a nested Python list)
and write it back out as a Lisp program, wrapping each list in parentheses and
joining the parts with spaces, recursively, so nested lists nest.  (A primitive
function has no meaningful written form, so it prints as `#<primitive>`; anything
else falls back on Python's `str`.)

A tiny `run` helper ties it together: evaluate an expression, then print both the
program and its return value in Lisp syntax:

```python
def run( expr ):
    result = lEval( expr, global_env )
    print( f'>>> {lisp_str( expr )}' )    # the program, in Lisp syntax
    print( f'==> {lisp_str( result )}' )  # its return value
    print()

run( ['set!', 'a', ['+', 1, 1]] )
run( ['+', ['-', 10, 7], 'a'] )
run( ['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]] )
run( ['+', 1, 2, 3, 4] )
```

which prints:

```
>>> (set! a (+ 1 1))
==> 2

>>> (+ (- 10 7) a)
==> 5

>>> (if (= a 2) (+ a 1) (- a 1))
==> 3

>>> (+ 1 2 3 4)
==> 10
```

Look at what just happened.  We wrote each program by hand as a nested Python
list (the AST, exactly as §1.2 promised, with no parser anywhere), handed it to
`lEval`, and got its return value back.  `lisp_str` printed the session in
parentheses, so it reads like Lisp even though every program was really a Python
list all along.  The last line is a small reminder from §1.4: `+` receives the
whole argument list, so it adds four numbers as happily as two.

The complete, runnable file is `examples/IttyBittyLisp1.py`:

```
python examples/IttyBittyLisp1.py
```

And that is a whole interpreter.  In well under a hundred lines you have all of
it: an **AST** (nested lists), a recursive **evaluator** (`lEval`) that walks it,
an **environment** (a dict) that remembers names, a handful of **special forms**,
and a set of **primitives** for it to call.  An interpreter built this way, one
that runs a program by walking its syntax tree and acting on each node as it goes,
is called a **tree-walk interpreter**: the most direct way there is to run a
language.  The next several chapters keep this same tree-walking shape and make it
do more, starting with the one thing this interpreter still cannot do: let a
program define functions of its own.  (Only Chapter 6 departs from it, compiling
the tree to a flat list of instructions first, for speed.)


## 1.6 A REPL, for free

You have a working interpreter, but so far you *feed* it programs by hand,
writing each one as a nested Python list.  That was the whole point of §1.2, and
it is the right way to learn how evaluation works.  It is not, however, how you
would want to *use* a language.  You want to type `(+ 1 2)` and see `3`.

The missing piece is a **parser**: a function that turns Lisp source *text* (the
string `"(+ 1 2)"`) into the nested-list AST the evaluator already understands,
`['+', 1, 2]`.  And because, as §1.2 showed, Lisp's written form already mirrors
the tree, that translation is nearly mechanical: read across the characters and
turn each `(...)` into a Python list.  We hand you one, `parse`, in
`examples/IttyBittyParser.py`:

```python
from IttyBittyParser import parse

parse("(+ 1 2)")                        # -> ['+', 1, 2]
parse("(if (= a 2) (+ a 1) (- a 1))")   # -> ['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]]
```

Its output is *exactly* the AST you have been writing by hand, nothing more.  Put
`parse` in front of `lEval` and the pipeline is complete: text → `parse` → tree →
`lEval` → value.  A handful of lines wire it into an interactive prompt, a
**REPL**, for *read–eval–print loop*:

```python
while True:
    source = input('lisp> ')
    if source in ('quit', 'exit'):
        break
    print( lisp_str( lEval( parse(source), global_env ) ) )   # read, eval, print
```

Now you can talk to the interpreter you built:

```
lisp> (set! a (+ 1 1))
2
lisp> (if (= a 2) (+ a 1) (- a 1))
3
lisp> (quote (a b c))
(a b c)
```

That is a real Lisp, and it is *your* Lisp: `lEval` is doing all the work;
`parse` only handed it the tree.

We are treating `parse` as a black box on purpose.  How it turns text into trees
is a satisfying little problem in its own right, and it gets a full chapter later
in the series; nothing about evaluation depends on it.  But from here on, run the
examples by typing them at the REPL rather than hand-writing lists: the language
is real enough now to simply use.


## 1.7 Challenges

Each of these builds on what this chapter gave you: the evaluator, or the REPL
from §1.6.  Try them against `IttyBittyLisp1.py`.

- **Make the REPL read multi-line expressions.**  The §1.6 REPL reads one line at
  a time, so a whole expression has to be typed (and read back) on a single
  line, which gets unwieldy fast.  Let it span lines instead: keep reading, with a
  continuation prompt (say `...`), until the parentheses balance, and only then
  hand the accumulated text to `parse`:

  ```
  lisp> (if (= a 2)
  ...       (+ a 1)
  ...       (- a 1))
  3
  ```

  The test is simple: count the `(` and `)` read so far, and keep prompting while
  the opens still outnumber the closes.  A first version can just count
  parentheses in the raw text; a nicer one strips `;` comments first (a comment
  might hide a stray paren), exactly as the tokenizer does.

- **Add list primitives.**  Add five list operations to `global_env` as plain
  Python functions.  A Lisp list, in our interpreter, is just a Python list, so
  each of these is short:

    - `(car lst)`: the **first** element of a list.  `(car (quote (a b c)))`
      returns `a`.
    - `(cdr lst)`, the **rest** of a list: the same list with its first element
      removed.  `(cdr (quote (a b c)))` returns `(b c)`.
    - `(cons x lst)`: a **new** list with `x` added to the front.
      `(cons 1 (quote (2 3)))` returns `(1 2 3)`.
    - `(list a b ...)`: **build** a list from the given arguments.
      `(list 1 2 3)` returns `(1 2 3)`.
    - `(null? lst)`: `#t` if the list is **empty**, `#f` otherwise.

  (`car` and `cdr` are Lisp's traditional, and admittedly cryptic, names for
  "first" and "rest"; the names are a historical accident, but the operations are
  simple.)  These are also what finally make `quote` pay off: with them you can
  build lists and take them apart, and so write real list-processing programs
  without touching `lEval` at all.  That is worth sitting with: a whole class of
  programs added purely as *library* (primitives), with the *language* (the
  evaluator) left unchanged.  By §1.4.2 every one of these is a primitive: each
  only computes from already-evaluated arguments.

- **Add `not`, `and`, `or`.**  These are the logical operators, with the same
  meaning as in Python: `(not x)` flips a boolean, `(and x y ...)` is true only
  if *every* argument is true, and `(or x y ...)` is true if *any* argument is.
  The catch is *short-circuiting*: `(and x y)` must not bother evaluating `y`
  once `x` turns out false, and `(or x y)` must not evaluate `y` once `x` is
  already true.  `not` is a plain primitive, but, exactly as §1.4.2 warned, that
  short-circuit rule means `and` and `or` cannot be: a primitive's arguments are
  all evaluated before it ever runs.  Implement them as special forms and feel
  the boundary from the inside.

- **Add `cond`.**  `cond` is a multi-branch conditional, Lisp's version of a
  Python `if / elif / elif / else` chain.  It takes a series of clauses, each of
  the form `(test result)`, and tries the tests from top to bottom; the first
  test that is true supplies the value:

  ```scheme
  (cond ((= n 0) (quote zero))
        ((< n 0) (quote negative))
        (else    (quote positive)))   ; else: the catch-all, taken if reached
  ```

  Implement it as a special form: evaluate each test in turn and, for the first
  one that holds, evaluate and return its result (a final `else` clause, if
  present, always holds).  It is really just a chain of `if`s, written as a loop
  over the clauses inside `lEval`.  Notice that you are writing that
  transformation *in Python*; much later, macros will let you write
  transformations like it *in Lisp itself*.

- **Friendlier errors.**  Evaluate `(+ a 1)` when `a` was never set and you get a
  raw Python `KeyError`.  Catch it and report the problem in the program's own
  terms (`unbound name: a`).  Then hunt down the other ways a bad program can
  crash the interpreter (a malformed `(if)`, an attempt to call something that is
  not a function) and decide where each one is best handled.



