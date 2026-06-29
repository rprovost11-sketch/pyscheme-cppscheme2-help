# The Parser

*Companion to the `EVALUATOR1-DOC` series.*

The IttyBitty evaluator examples write their input as Python list structures
directly -- `['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]]` -- and
pass them straight to `lEval`.  A real interpreter starts from a source
string.  The parser bridges the gap.

The complete pipeline is:

```
source string  ->  tokenize()  ->  read_from()  ->  lEval()  ->  result
```

S-expressions are unusually easy to parse.  The entire parser fits in about
30 lines.

## Phase 1: Tokenizing

The tokenizer converts a source string into a flat list of token strings.

```python
def tokenize( source ):
    return source.replace( '(', ' ( ' ).replace( ')', ' ) ' ).split()
```

S-expressions have only two special characters: `(` and `)`.  Padding them
with spaces means Python's `str.split()` separates every token cleanly --
no character-by-character scanning required.

```
source:  (if (= a 2) (+ a 1) (- a 1))
tokens:  ['(', 'if', '(', '=', 'a', '2', ')', '(', '+', 'a', '1', ')',
          '(', '-', 'a', '1', ')', ')']
```

The nested structure is gone -- parentheses are just tokens in a flat list.
The reader's job is to put the nesting back.

## Phase 2: Reading

The reader converts the flat token list into a nested AST by consuming one
complete expression at a time.

```python
def read_from( tokens ):
    token = tokens.pop( 0 )
    if token == '(':
        result = []
        while tokens[0] != ')':
            result.append( read_from( tokens ) )
        tokens.pop( 0 )          # discard closing ')'
        return result
    else:
        return atom( token )
```

The logic matches the grammar exactly: a list starts with `(`, contains
zero or more expressions, and ends with `)`.  Each inner expression is
another call to `read_from` -- the recursion handles arbitrary nesting
without any explicit stack management.

`atom` converts a token string to the appropriate Python type:

```python
def atom( token ):
    try:    return int( token )
    except ValueError:
        try:    return float( token )
        except ValueError:
            return token             # symbol -- a plain Python string
```

## Code is Data

After parsing, the AST for `(if (= a 2) (+ a 1) (- a 1))` is:

```python
['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]]
```

This is *identical* to what the IttyBitty examples wrote by hand.  The
evaluator never knew the difference -- it just sees a Python list.  This
is the **code is data** principle: Lisp source code and Lisp data share
the same representation.  A list that the parser produces can be
manipulated, transformed, and evaluated just like any other list.

It also means the evaluator is complete as written.  Adding a parser does
not require any changes to `lEval` -- the two phases are entirely
independent.

## Quote Shorthand

The `quote` special operator suppresses evaluation -- `(quote (a b c))`
returns the list `(a b c)` without evaluating it.  Writing `quote`
explicitly is verbose, so Lisp provides a shorthand: `'expr` is reader
syntax for `(quote expr)`.  The expansion happens in the parser, not the
evaluator.

```python
elif token == "'":
    return [ 'quote', read_from( tokens ) ]
```

```
'(a b c)          ->  ['quote', ['a', 'b', 'c']]
(quote (a b c))   ->  ['quote', ['a', 'b', 'c']]
```

Both forms produce identical ASTs and evaluate identically.  The evaluator
sees only `(quote ...)` -- it has no knowledge of the `'` shorthand at all.
This is a clean separation of concerns: surface syntax belongs to the
parser, semantics belong to the evaluator.

## Running the Example

The complete working code is in `examples/IttyBittyParser.py`.

```
python examples/IttyBittyParser.py
```

It demonstrates the tokenizer output, the parser output, the full pipeline
from source string to evaluated result, and the quote shorthand producing
the same AST as `(quote ...)`.

## Challenges

- **Build a REPL.** You now have both a parser and an evaluator.  Wire them
  together: read a line, parse it, evaluate it, print the result, repeat.
  Add a way to exit (e.g. `quit`) and you have a working Lisp interpreter
  in under 60 lines of total code.

- **Add string literals.** Extend the tokenizer to recognize double-quoted
  strings.  This requires scanning for the closing `"` rather than relying
  on `split()` -- the whitespace trick breaks down here.  You will need a
  small character-by-character scan for the quoted region before handing
  off to `split()` for everything else.

- **Add `#t` / `#f` booleans.** Extend `atom` to recognize the tokens `#t`
  and `#f` and produce the boolean values the evaluator expects.  Then add
  `not` as a primitive.
  When you try to add `and` and `or`, you will find they need short-circuit
  evaluation -- which means they belong in the evaluator as special forms,
  not in the parser or `global_env`.

- **Add backquote and unquote.** `` ` `` and `,` are reader syntax for
  building list templates: `` `(a ,b c) `` expands to `['list', ['quote', 'a'] 'b', ['quote', 'c']]`.
  Implement them in `read_from` alongside the `'` expansion.  This is the
  foundation of macro template syntax -- once you have it, writing a macro
  expander becomes much more natural.

## What the Real Parser Adds

The parser above handles the core of S-expression syntax.  The full parser
in `pyscheme/Parser.py` extends it with:

- **Strings**: double-quoted literals with escape sequences
- **Characters and rationals**: additional atom types
- **Backquote / unquote**: reader syntax for macro templates
- **Error reporting**: line and column numbers in error messages
- **Incremental parsing**: `parseOne()` returns one expression plus the
  number of characters consumed, for use in the REPL
