# Evaluator 1 - The Minimal Recursive Evaluator

*Continues in `EVALUATOR2-DOC`: closures and lexical scoping.*

This is a **tree-walk interpreter**.  To evaluate an expression it walks the
abstract syntax tree (AST) and interprets each node directly -- no bytecode,
no compilation step.

In this minimal version the AST is built from plain Python types: a symbol is
a Python `str`, a number is a Python `int` or `float`, and a list is a Python
`list`.  The environment is a plain dictionary mapping names to values.

This series introduces the implementation of a Scheme interpreter, starting
from the smallest evaluator that works and adding one idea at a time.

## A Minimal Lisp Evaluator

The following self-contained Python program implements a working Lisp
evaluator.  It strips away all the machinery of a full interpreter and shows
just the essential structure.

```python
def lEval( expr, env ):
    # Boolean literals self-evaluate -- they are data, not identifiers, so
    # they must be recognized before the symbol-lookup branch.
    if expr in ('#t', '#f'):
        return expr
    # A string is a variable name -- look it up in the environment.
    if isinstance(expr, str):
        return env[expr]
    # Any other non-list atom (int, float, ...) evaluates to itself.
    elif not isinstance(expr, list):
        return expr

    # expr is a non-empty list -- a special form or procedure call.  (A bare ()
    # is not a valid Scheme expression; this evaluator assumes a well-formed AST.)
    head = expr[0]

    # Special forms: the head names a form that controls its own evaluation.
    # 'if' must NOT evaluate both branches -- only the one that is taken.
    if head == 'if':
        condValue = lEval( expr[1], env )
        # Scheme truthiness: every value except #f is true (so 0 and the
        # empty list are true).  Dispatch on #f, not Python truthiness.
        return lEval( expr[2] if condValue != '#f' else expr[3], env )

    elif head == 'begin':
        for sub in expr[1:-1]:
            lEval( sub, env )             # non-tail forms: recurse
        return lEval( expr[-1], env )

    # 'set!' must NOT evaluate the variable name -- only the value expression.
    elif head == 'set!':
        var, valExpr = expr[1:]
        val = lEval(valExpr, env)
        env[var] = val
        return val

    # 'quote' returns its argument unevaluated.
    elif head == 'quote':
        return expr[1]

    # Regular function call: evaluate everything -- the head and all arguments --
    # then call the resulting function with the resulting argument values.
    fn, *evaluatedArgs = [ lEval(subExpr, env) for subExpr in expr ]
    return fn( evaluatedArgs )
```

A note on `set!`: real Scheme separates `define` (introduce a new binding) from
`set!` (assign an existing one).  This tiny Lisp has just one binding form -- a
lenient `set!` that assigns the name, creating it if it does not yet exist.

## Primitives

Primitive functions are plain Python functions stored in the environment
by name.  From the evaluator's perspective they are just values that happen
to be callable.

```python
# the global environment: maps names to values
global_env = {
    '+':     lambda args: args[0] + args[1],
    '-':     lambda args: args[0] - args[1],
    '*':     lambda args: args[0] * args[1],
    '=':     lambda args: '#t' if args[0] == args[1] else '#f',
    '<':     lambda args: '#t' if args[0] <  args[1] else '#f',
}
```

## Equivalent Lisp

The following expressions exercise the evaluator.  `lisp_str` renders a value
in Lisp surface syntax (so a list prints as `(a b c)` rather than
`['a', 'b', 'c']`); see `IttyBittyLisp1.py` for its definition.

```python
def run( expr ):
    print( f'==> {lisp_str( lEval( expr, global_env ) )}' )

def main():
    run( ['set!', 'a', ['+', 1, 1]] )                          # ==> 2
    run( ['+', ['-', 10, 7], 'a'] )                            # ==> 5
    run( ['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]] ) # ==> 3

if __name__ == '__main__':
    main()
```

```lisp
; set! is a special form: the name 'a' is NOT evaluated, only the value.
(set! a (+ 1 1))                ; ==> 2

; Regular function call: +, (- 10 7), and a are all evaluated first.
(+ (- 10 7) a)                  ; ==> 5

; if is a special form: only ONE branch is evaluated, never both.
(if (= a 2) (+ a 1) (- a 1))    ; ==> 3
```

## Running the Example

The complete working code is in `examples/IttyBittyLisp1.py`.

```
python examples/IttyBittyLisp1.py
```

*Next: `EVALUATOR2-DOC` adds lexical scopes, `let`, and closures.*

## Challenges

- **Add list primitives.** Add `car`, `cdr`, `cons`, `list`, and `null?` to
  `global_env` as Python lambdas.  With these plus `quote` you can write
  list-processing programs without touching `lEval` at all -- which tells
  you something about where the boundary between the evaluator and the
  library really is.

- **Add `not`, `and`, `or`.** `not` is a straightforward primitive, but
  `and` and `or` require short-circuit evaluation: `(and x y)` must not
  evaluate `y` if `x` is `#f`.  That means they cannot be plain lambdas.
  They need to be special forms.  What does that boundary tell you about
  the difference between a primitive and a special form?

- **Add `cond`.** Implement `(cond (test1 result1) (test2 result2) ...)` as
  a special form -- it's just nested `if`, but written as a loop over
  clauses inside `lEval`.  Notice that you are writing the transformation
  *in Python* rather than in Lisp.  That distinction -- built-in special
  form versus user-defined macro -- will matter later.

- **Improve error messages.** An unbound variable currently raises a Python
  `KeyError`.  Catch it and produce a friendlier message.  Then look for
  the other places a bad program can crash the Python runtime and consider
  where the right boundary for error handling is.
```