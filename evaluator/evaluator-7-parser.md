# Chapter 7: The Parser: From Text to Tree

Since Chapter 1 you have been able to *type* Lisp.  Back in §1.6 we handed you a
small tool called `parse` and a three-line REPL, as a black box: you typed
`(+ 1 2)` at a prompt, and the evaluator you had just built answered `3`.  Behind
that prompt, `parse` was quietly turning the string `"(+ 1 2)"` into the nested
Python list `['+', 1, 2]`, the very AST every chapter since has evaluated.  We
deferred its insides on purpose, so the evaluator could stay the protagonist.

This closing chapter opens the box.  And it is a small box, smaller than any
evaluator in the series, for a reason Chapter 1 gave you before you had built
anything at all: **the parentheses already are the tree.**  A Lisp program, written
out, is already shaped exactly like the structure a parser is supposed to recover.
So recovering it is almost no work.  By the end of this chapter you will have seen
the whole front end, and you will understand why a Lisp reader fits on a page while
the parser for most languages fills a book.


## 7.1 From text to tree, in two stages

A parser's job is to turn a flat string of characters into a structured tree.  Ours
does it in two stages, and the split is the same one real compilers use:

```
   "(+ 1 2)"                            source: a flat string
       |
       v   tokenize
   ['(', '+', '1', '2', ')']           tokens: a flat list of pieces
       |
       v   read_from
   ['+', 1, 2]                         AST: a nested tree
       |
       v   lEval   (Chapters 1-6)
   3                                   the result
```

The first stage, **tokenizing** (also called lexing), chops the raw string into its
smallest meaningful pieces (parentheses, names, numbers), throwing away spaces and
comments.  It knows nothing about structure; it just produces a flat list of token
strings.  The second stage, **reading**, walks that flat list and assembles it into
the nested tree, using the parentheses as its guide.  Splitting the work this way
keeps each stage simple: the tokenizer worries only about characters, the reader only
about shape.


## 7.2 Tokenizing

Here is the whole tokenizer:

```python
def tokenize( source ):
    lines = []
    for line in source.splitlines():
        comment = line.find( ';' )
        if comment >= 0:
            line = line[:comment]          # drop everything after a ';'
        lines.append( line )
    source = ' '.join( lines )
    return source.replace( '(', ' ( ' ).replace( ')', ' ) ' ).split()
```

It first strips comments (in Lisp a `;` runs to the end of its line) by cutting each
line at its first `;`.  Then comes the trick that makes S-expressions so easy to
tokenize: **there are only two special characters, `(` and `)`.** Everything else
(names, numbers, operators) is separated by spaces already.  So we do not need a
character-by-character scanner at all.  We just pad every parenthesis with spaces,
turning `(+ 1 2)` into ` ( + 1 2 ) `, and then let Python's ordinary `str.split()` do
the cutting.  Splitting on whitespace gives exactly the tokens we want:

```
tokenize("(if (= a 2) (+ a 1) (- a 1))")
  -> ['(', 'if', '(', '=', 'a', '2', ')', '(', '+', 'a', '1', ')', '(', '-', 'a', '1', ')', ')']
```

Every parenthesis is its own token, and every name and number stands alone.  That is
the entire lexer: two `replace`s and a `split`.  A language whose tokens are *not*
so cleanly space-separated (where `1+2*3` must be pulled apart into `1`, `+`, `2`,
`*`, `3` with no spaces to help) needs a real scanner here; Lisp's syntax spares us
one.


## 7.3 Reading: recursive descent

The reader turns that flat token list into the tree.  It is a single recursive
function, and its recursion mirrors the nesting of the parentheses exactly the way
`lEval`'s recursion mirrored the nesting of the tree.  If the evaluator felt natural,
this will too; it is the same shape of walk.

```python
def read_from( tokens ):
    if not tokens:
        raise SyntaxError( 'unexpected end of input' )
    token = tokens.pop( 0 )               # take the next token off the front
    if token == '(':                      # a list begins
        result = []
        while tokens[0] != ')':
            result.append( read_from( tokens ) )   # read each element, recursively
        tokens.pop( 0 )                   # discard the matching ')'
        return result
    elif token == ')':
        raise SyntaxError( 'unexpected )' )
    elif token == "'":                    # quote shorthand -- §7.4
        return [ 'quote', read_from( tokens ) ]
    else:
        return atom( token )              # a number or a name
```

`read_from` consumes one complete expression from the *front* of the token list,
mutating the list in place so that each recursive call sees only the tokens that are
left.  The logic is a direct reading of what the tokens mean:

- An opening `(` starts a list.  We make an empty list and, until we meet the matching
  `)`, call `read_from` again for each element, and *that* call handles nested lists
  by the same rule, to any depth.  When we reach the `)`, we discard it and return the
  assembled list.
- A stray `)` with no `(` before it is a syntax error.
- Anything else is a leaf, handed to `atom`.

That recursion is the whole parser's engine.  A `(` calls `read_from` for its
contents; a `(` inside those contents calls `read_from` again; the Python call stack
tracks exactly how deep in parentheses we are (one frame per open paren) and unwinds
as each `)` closes.  The nesting of the calls *is* the nesting of the tree.

The leaves are classified by `atom`:

```python
def atom( token ):
    try:
        return int( token )               # an integer?
    except ValueError:
        try:
            return float( token )         # a float?
        except ValueError:
            return token                  # otherwise a symbol -- a plain string
```

It tries to read the token as an `int`, then as a `float`, and if both fail leaves it
as a string, a **symbol**.  This is why numbers in the AST are Python numbers while
names are Python strings, exactly the split every evaluator in this series relied on:
`lEval` looked things up by string name and did arithmetic on the numbers.  The
booleans `'#t'` and `'#f'` fall through to the symbol case as the strings the
evaluator expects, with no special handling.


## 7.4 The quote shorthand

One convenience lives in the reader: `'expr` is shorthand for `(quote expr)`, so you
can write `'(a b c)` instead of `(quote (a b c))`.  You saw `quote` first in Chapter
1 (the special form that returns its argument untouched, as data), and this is just
a friendlier way to write it.

The reader handles it in the one line you already saw: when `read_from` meets a `'`
token, it reads the next whole expression and wraps it, returning `['quote', <that
expression>]`.  That is a **reader macro**, a rewrite that happens during parsing,
turning surface sugar into the plain form the evaluator understands, so the evaluator
never has to know the shorthand exists.

There is one small wrinkle.  Our tokenizer only isolates `'` when it sits next to a
parenthesis (the paren-padding does the separating); against a bare name, `'x` stays
glued together as a single token.  So `parse` does a quick fix-up first, peeling a
leading `'` off any such token before the reader runs:

```python
def parse( source ):
    tokens = tokenize( source )
    expanded = []
    for tok in tokens:
        if tok.startswith( "'" ) and len( tok ) > 1:
            expanded.append( "'" )        # split 'x into ' and x
            expanded.append( tok[1:] )
        else:
            expanded.append( tok )
    return read_from( expanded )
```

With that, `'x` and `'(a b c)` both reach the reader as a `'` token followed by an
expression, and both come out wrapped in `quote`.


## 7.5 The complete parser, and closing the loop

That is the entire front end: `tokenize` to cut the string into tokens, `read_from`
to fold the tokens into a tree, `atom` to classify the leaves, and `parse` to tie them
together (with the quote fix-up).  Run it and watch the string become the exact AST
the series has been evaluating by hand all along:

```
Tokenizer output:
  source:  (if (= a 2) (+ a 1) (- a 1))
  tokens:  ['(', 'if', '(', '=', 'a', '2', ')', '(', '+', 'a', '1', ')', '(', '-', 'a', '1', ')', ')']

Parser output (this is the AST lEval operates on):
  source:  (if (= a 2) (+ a 1) (- a 1))
  ast:     ['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]]
```

Look at that AST and then look back at Chapter 1 §1.5, where we wrote
`['if', ['=', 'a', 2], ['+', 'a', 1], ['-', 'a', 1]]` *by hand* and fed it to `lEval`.
They are the same list, character for character.  Everything you wrote out by hand as
a nested Python list, from Chapter 1 on, `parse` now builds for you from a string.
And the whole pipeline, string to answer, is nothing more than the two stages of
this chapter feeding the evaluator of the last six:

```
Full pipeline: source string -> parse -> lEval -> result
  source:  (set! b (* 6 7))
  ast:     ['set!', 'b', ['*', 6, 7]]
  result:  42

  source:  '(a b c)
  ast:     ['quote', ['a', 'b', 'c']]
  result:  (a b c)
```

This is exactly what the REPL from §1.6 was doing all along: read a line, `parse` it
to an AST, hand the AST to `lEval`, print the result.  The black box is open, and
there was nothing in it but the two small stages you just read.


## 7.6 Why it was so small

It is worth ending on why this chapter was the shortest in the series, because the
answer is the whole reason Lisp is written the way it is.

Cast your mind back to Chapter 1 §1.2.  There we wrote the arithmetic `1 + 2*3` and
drew its tree, and the point was that recovering that tree from the text is *work*:
you have to know that `*` binds tighter than `+`, that the multiplication happens
first even though it is written second, that precedence and associativity conspire to
imply a shape the flat text does not show.  A parser for such a language carries all
of that knowledge (grammar rules, precedence tables, sometimes thousands of lines of
it) precisely to reconstruct the tree the syntax only hints at.

A Lisp program hints at nothing.  It states its tree outright: `(+ 1 (* 2 3))` says,
with its parentheses, *exactly* how it nests, leaving the reader nothing to figure
out.  There is no precedence to encode because everything is parenthesized; there is
no grammar to consult because the shape is on the surface.  So the "parser" collapses
into what you saw: split on parentheses, then fold the pieces back up along those same
parentheses.  The parentheses that some find off-putting are exactly what let the
reader be fifty lines instead of five thousand, and, not coincidentally, exactly what
make Lisp code so easy to treat *as data* and transform, the trick behind the macro
challenges scattered through this series.

That is the front end, and with it the picture is complete.  You have built, from
nothing, the whole path a program takes: characters into tokens, tokens into a tree,
and a tree through six successively deeper machines (tree-walker, closures, loop,
CEK machine, its full-language form, and a compiler feeding a bytecode VM) into a
value.  Every idea in it, scaled up, is an idea inside the language you wrote it in.


## 7.7 Challenges

- **String literals.**  Add double-quoted strings, like `"hello"`, as a new kind of
  atom.  This is the one place the tidy pad-and-split tokenizer breaks down: a space
  *inside* a string must not split it, so you will need to scan characters for this
  token kind, a first taste of why real lexers are written the way they are.

- **Better errors.**  The reader says `unexpected end of input` but not *where*.  Have
  the tokenizer record each token's line and column, and make the syntax errors point
  at the offending spot: the difference between a toy reader and a usable one.

- **Booleans and characters.**  Right now `#t` and `#f` arrive as ordinary symbols,
  which happens to be just what our evaluator wants.  Make `atom` recognize them (and
  character literals like `#\a`) explicitly, so the reader's output does not depend on
  a lucky coincidence downstream.

- **A real REPL.**  Combine `parse` with any evaluator from Chapters 2–6 into a loop
  that reads a line, evaluates it, and prints the result, keeping one global
  environment across lines so `set!` persists.  Then add the multi-line reading from
  Chapter 1's challenge: when the parentheses in a line are not yet balanced, prompt
  for more instead of failing.

- **Quasiquote.**  Add the backtick family (`` `expr ``, `,expr`, `,@expr`) as reader
  shorthands the way `'` is handled, expanding to `quasiquote`, `unquote`, and
  `unquote-splicing`.  Then think about what it would take to *evaluate* them; that is
  a small language of its own, and a good place to go next.
