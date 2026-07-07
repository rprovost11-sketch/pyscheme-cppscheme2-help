# Chapter 6: The Bytecode VM

Chapters 4 and 5 built the CEK machine and scaled it to the whole language.  Look
closely at what it does every time around its loop: before it can do anything, it
asks *what kind of expression is this?* (is `C` a number, a variable, a
`lambda`, an `if`, an application?) and picks the matching transition.  It asks
that question afresh on every single step.

But for a fixed program the answer never changes.  In `((lambda x x) 7)`, the head
`(lambda x x)` is a `lambda` the first time the machine looks and a `lambda` the
millionth time; running a loop a hundred thousand times re-asks "is this an `if`?"
about the very same `if`, and gets the very same yes, a hundred thousand times.
That re-deciding is pure waste: the shape of the program is known before it ever
runs.

This chapter removes the waste by deciding **once, ahead of time**.  We walk the
AST a single time and, wherever the CEK machine *would* have dispatched, we write
down the decision as a numbered instruction, an **opcode**.  The result is a flat
list of opcodes, the program's **bytecode**.  Then a second loop runs the bytecode:
it reads the next opcode and does it, with no AST to inspect and no "what kind of
expression is this?" left to ask.  That two-step shape (**compile**, then **run**)
is the whole idea, and it is what nearly every real language runtime does, Python's
own included.

A bytecode VM, in one line, is *the CEK machine with its dispatch precomputed*.
Because it is the same machine seen from a new angle, this chapter leans hard on
Chapter 4: most of the opcodes are transitions you already know, wearing numbers.
As in Chapter 4, we compile only the tiny language (pure functions and `if`, a
number true unless it is `0`) so that the one new idea, *compilation*, stands
alone.  The full language of Chapter 5 compiles exactly the same way, with more
opcodes; shrinking the language again is the same "isolate the new thing" move
Chapter 4 made, not a step backward.


## 6.1 Dispatch, precomputed

Look at what one pass of the CEK machine's EVAL loop actually spends its effort on:

```python
if   isinstance(C, int):   ...      # is it a number?
elif isinstance(C, str):   ...      # a variable?
elif C[0] == 'lambda':     ...      # a lambda?
elif C[0] == 'if':         ...      # an if?
else:                      ...      # an application
```

Every one of those tests is a question about the *shape* of the code, and the
shape of the code is fixed the moment the program is written.  The machine is
re-answering, at run time, questions whose answers were already settled at compile
time.  A loop that runs its body a million times pays for a million of these
identical inspections.

So we split the work in two.  **Compile time:** walk the AST once, and at each node,
instead of *acting* on it, write down *which action the machine would take* as a
small numbered instruction.  A number becomes an `INT` instruction, a variable a
`VAR`, a `lambda` a `LAM`, and so on.  The output is a flat Python list of these
instructions, no longer a tree, just a sequence.  **Run time:** a loop walks that
sequence, and for each instruction does the one thing its number names.  No
`isinstance`, no `C[0] == ...`, no choosing: the choosing already happened, once,
and is baked into the opcodes.

```
   AST                compile (walk once)        bytecode              run (VM loop)
   ---                -------------------        --------              -------------
   nested lists   ->  decide every dispatch  ->  flat opcode list  ->  read op, do it
```

Two whole sources of run-time work vanish in this move.  The shape tests go, as we
have seen.  And so does the EVAL/APPLY state flag from Chapter 4: the compiler lays
the instructions down in the order they must run, so "am I descending into code or
feeding a value back?" is no longer a question the loop asks: it just runs the next
instruction.  What is left at run time is close to the smallest loop imaginable:
fetch an opcode, do it, advance.


## 6.2 Opcodes are the CEK transitions, named

Here is the leverage of having built the CEK machine: you already know what every
opcode does, because each one is a CEK transition with a number bolted on.  The
machine's registers even carry straight over: `E` (environment) and `K` (the
continuation stack, holding the very same frame kinds), plus the value register `V`.

Line up the transitions from Chapter 4 against the opcodes that record them:

| Chapter 4 CEK transition | Opcode | What it does at run time |
|---|---|---|
| `C` is a number → `V = C` | `INT n` | `V = n` |
| `C` is a variable → `V = E.lookup(C)` | `VAR name` | `V = E.lookup(name)` |
| `C` is a `lambda` → `V =` a closure | `LAM param body` | `V = (closure, param, body, E)` |
| `if`: push `FRAME_IF`, evaluate test | `IF_START then else` | `K.push((FRAME_IF, then, else, E))` |
| `FRAME_IF`: pick a branch by `V` | `APPLY_IF` | pop `FRAME_IF`, jump to `then`/`else` by `V` |
| application: push `FRAME_ARG`, eval fn | `APP_START` | `K.push((FRAME_ARG, E))` |
| `FRAME_ARG`: remember fn, eval arg | `APPLY_ARG` | pop it, `K.push((FRAME_CALL, V))` |
| `FRAME_CALL`: enter the body | `CALL` / `TCALL` | bind the parameter, enter the body |

Every line is something you have already traced by hand.  `IF_START`/`APPLY_IF` is
the `FRAME_IF` pair from §4.3.  `APP_START` / `APPLY_ARG` / `CALL` is the
function-call sequence from §4.5.  The opcodes are not a new machine: they are the
old machine's moves, given names so they can be written down.

One detail here is the theme of the whole chapter.
Wherever the CEK machine stored an *expression*, the compiled version stores a
*position* instead.  `FRAME_IF` in Chapter 4 held the two branch *expressions*; here
it holds their two *addresses* in the bytecode (`then`, `else` are instruction
indices).  And `FRAME_ARG`, which in Chapter 4 had to carry the argument expression
so it could be evaluated later, here carries only `E`: because the argument's
instructions simply come next in the stream, with nothing to remember.  The flat
sequence does the remembering that the tree used to.

Two opcodes in the toy have no Chapter 4 counterpart at all: `JUMP` and `RET`.  They
exist precisely *because* the program is now a flat stream with no surrounding tree
to navigate: `JUMP` moves around within it, and `RET` marks the end of a function
body.  They are the price of flattening, and we meet each where it is built: `JUMP`
with the compiler in §6.4, `RET` with its frame in §6.5.


## 6.3 `pc` replaces `C`

The CEK machine kept the expression under evaluation in the register `C`.  The VM
has no expressions at run time (only a flat list of instructions) so it needs a
different notion of "where we are."  That is the one genuinely new register: **`pc`**,
the **program counter**, holding the *index of the next instruction to run*.

Where `C` pointed at a subtree, `pc` points at a position in the stream.  And where
the CEK machine moved through a program by walking into sub-expressions, the VM
moves through it by changing `pc`:

- Most instructions just **advance**: do the thing, then `pc += 1` to fall through to
  the next instruction.  Because the compiler laid the code down in evaluation order,
  "descend into the next thing to evaluate" is nothing more than `pc += 1`.
- A few instructions **jump**: they set `pc` to some other index instead of adding
  one.  Picking an `if` branch sets `pc` to that branch's address; entering a
  function body sets `pc` to the body's address; skipping over an inline body (that
  `JUMP`) sets `pc` past it.  These are exactly the moments the CEK machine would
  have leapt to a different part of the tree.

So the register that changes the most between the two machines is just this: `C`, an
expression, becomes `pc`, an integer offset.  `V`, `E`, and `K` are unchanged: the
value still flows in `V`, scopes still chain through `E`, and the continuation is
still an explicit stack `K`.  The bytecode VM is the CEK machine with `C` replaced by
`pc` and the dispatch compiled away; everything else you already own.


## 6.4 The compiler

The compiler is a single recursive walk over the AST (`compile_expr(expr, out, tail)`)
that appends instructions to a growing list `out`.  It has one branch per
expression kind, mirroring the CEK machine's EVAL dispatch, except that instead of
*doing* each thing it *emits the opcode* for it.  The extra argument `tail` rides
along to record whether the current expression sits in tail position; we will see
what it controls as we go.

Start with the leaves, which are almost too simple to comment:

```python
def compile_expr( expr, out, tail ):
    if isinstance( expr, int ):             # a number
        out.append( (OP_INT, expr) )
        if tail: out.append( (OP_RET,) )

    elif isinstance( expr, str ):           # a variable
        out.append( (OP_VAR, expr) )
        if tail: out.append( (OP_RET,) )
```

A number emits one `INT` instruction; a variable emits one `VAR`.  The `if tail:`
line is our first sight of tail handling: when an expression is the last thing a
function body does (its tail position) the value it produces *is* the function's
result, so we follow it with an `OP_RET` meaning "this body is finished, hand the
value back to the caller."  Exactly how `RET` hands it back is the VM's job, in §6.5;
here we only note *where* the compiler stamps one: after any value produced in tail
position.

Next the easy compound case, an application, because it lays its instructions down
in a straight line, with nothing to patch:

```python
    else:                                   # (fn arg)
        out.append( (OP_APP_START,) )
        compile_expr( expr[0], out, tail=False )   # fn  -- not tail
        out.append( (OP_APPLY_ARG,) )
        compile_expr( expr[1], out, tail=False )   # arg -- not tail
        out.append( (OP_TCALL,) if tail else (OP_CALL,) )
```

This emits, front to back, the exact sequence the CEK machine stepped through in
§4.5: `APP_START` (remember `E`), the function's code, `APPLY_ARG` (remember the
function value), the argument's code, and finally the call.  The function and
argument are each compiled with `tail=False`: neither one is the last thing the
whole call does, so neither ends in a `RET`.  The call itself is the one real
decision: if the application is *itself* in tail position it emits `OP_TCALL`,
otherwise `OP_CALL`.  That single line is where tail-call optimization is chosen:
**once, at compile time**, instead of tested on every call at run time as it was in
Chapters 3–5.

Now the case that needs real care, because it is where the flat stream first fights
back: a `lambda`.

A `lambda` has to emit two things.  First, a `LAM` instruction that, at run time,
builds the closure, and a closure needs to know *where its body's code lives*, so
`LAM` must carry the body's starting address.  Second, the body's instructions
themselves, which we lay down **inline**, right after the `LAM`.  But that creates a
problem: the normal, straight-line path would run `LAM` and then fall directly into
the body, running the body immediately, which is wrong; a body must run only when
the closure is *called*.  So we also need a `JUMP` right after `LAM` that leaps over
the body.

Here is the knot.  To emit `LAM` we need the body's *start* address, and to emit that
skip-`JUMP` we need the body's *end* address, and we know neither until we have
compiled the body, which we cannot do until after the `LAM` and `JUMP` we are trying
to emit.  The instruction has to point at code that does not exist yet: a **forward
reference**.

The way out is **backpatching**: emit a blank placeholder now, remember where it is,
and fill it in once the answer is known.

```python
    elif expr[0] == 'lambda':               # (lambda param body)
        lam_idx  = len(out); out.append( None )   # reserve the LAM slot
        jump_idx = len(out); out.append( None )   # reserve the JUMP slot
        body_pc  = len(out)
        compile_expr( expr[2], out, tail=True )   # a body is always in tail position
        past_body = len(out)
        out[lam_idx]  = (OP_LAM, expr[1], body_pc)   # now we know where the body starts
        out[jump_idx] = (OP_JUMP, past_body)         # ...and where it ends
        if tail: out.append( (OP_RET,) )
```

Read it as four steps.  We drop two `None` placeholders and remember their indices
(`lam_idx`, `jump_idx`).  We record `body_pc`, the index where the body is about to
begin, and compile the body inline, with `tail=True`, because whatever a body's last
expression yields is the function's result, so the body ends in a `RET`.  After it,
`past_body` is the index just past the body.  Now both answers exist, so we go back
and overwrite the two reserved slots with real instructions: `LAM param body_pc` and
`JUMP past_body`.  The layout that produces is:

```
   lam_idx   ->  LAM   param body_pc      build the closure (its body starts at body_pc)
   jump_idx  ->  JUMP  past_body          skip over the inline body
   body_pc   ->  ...the body, compiled inline, ending in RET...
   past_body ->  ...whatever comes after the lambda...
```

At run time the straight-line path reaches `LAM`, builds the closure, and falls into
`JUMP`, which leaps to `past_body`, so the body is stepped over, not run.  The body
runs only later, when the closure is called and a `CALL`/`TCALL` sets `pc` to
`body_pc`.  That is backpatching doing exactly one job: letting an instruction refer
to code that will only exist further down the stream.

Finally `if`, which uses the very same reserve-and-fill trick, now for the two branch
addresses:

```python
    elif expr[0] == 'if':                   # (if test then else)
        if_idx = len(out); out.append( None )     # reserve IF_START
        compile_expr( expr[1], out, tail=False )  # test -- never tail
        out.append( (OP_APPLY_IF,) )
        then_pc = len(out)
        compile_expr( expr[2], out, tail=tail )   # then inherits our tail context
        if not tail:
            then_jump_idx = len(out); out.append( None )   # skip the else branch
        else_pc = len(out)
        compile_expr( expr[3], out, tail=tail )   # else inherits our tail context
        if not tail:
            out[then_jump_idx] = (OP_JUMP, len(out))
        out[if_idx] = (OP_IF_START, then_pc, else_pc)
```

It reserves the `IF_START` slot, compiles the test (never in tail position), emits
`APPLY_IF`, then lays the two branches down inline and, once their addresses are
known, backpatches `IF_START` with `then_pc` and `else_pc`.  The branches inherit the
`if`'s *own* tail context (a tail `if` has both arms in tail position) so a tail
`if` needs no extra jump (each arm ends in its own `RET`), while a non-tail `if`
inserts a `JUMP` after the `then` arm to skip past the `else` and rejoin the code that
follows.  The branch structure the CEK machine used to read off the tree is now
computed here, at compile time, and frozen into addresses.

One line drives the whole thing:

```python
def compile_program( expr ):
    out = []
    compile_expr( expr, out, tail=True )     # the whole program is in tail position
    return out
```

The program is compiled in tail position, because its final value is the program's
result, so the last thing it does ends in a `RET`, and that `RET`, finding nothing
left on `K`, is what halts the machine.


## 6.5 The VM loop

With the bytecode in hand, the run loop is startlingly small.  There is no AST, no
`isinstance`, no EVAL/APPLY flag, just: fetch the instruction at `pc`, branch on its
integer opcode, do the one thing it names, and move `pc`.

```python
def run_vm( prog ):
    pc = 0
    V  = None
    E  = Environment()
    K  = []

    while True:
        instr = prog[pc]
        op    = instr[0]

        if op == OP_INT:                    # V = n
            V = instr[1]; pc += 1
        elif op == OP_VAR:                  # V = lookup
            V = E.lookup( instr[1] ); pc += 1
        elif op == OP_LAM:                  # V = closure, capturing E
            V = (VAL_CLOSURE, instr[1], instr[2], E); pc += 1
        elif op == OP_JUMP:                 # skip (over an inline body, say)
            pc = instr[1]

        elif op == OP_APP_START:            # remember E for the argument
            K.append( (FRAME_ARG, E) ); pc += 1
        elif op == OP_APPLY_ARG:            # fn value in V; remember it, restore E
            E = K.pop()[1]
            K.append( (FRAME_CALL, V) ); pc += 1

        elif op == OP_CALL:                 # non-tail: save return address, enter body
            closure = K.pop()[1]
            K.append( (FRAME_RET, pc + 1) )
            E  = Environment( parent=closure[3], bindings={ closure[1]: V } )
            pc = closure[2]
        elif op == OP_TCALL:                # tail: enter body, push NO return frame
            closure = K.pop()[1]
            E  = Environment( parent=closure[3], bindings={ closure[1]: V } )
            pc = closure[2]

        elif op == OP_IF_START:             # remember both branch addresses and E
            K.append( (FRAME_IF, instr[1], instr[2], E) ); pc += 1
        elif op == OP_APPLY_IF:             # V is the test; 0 is false, else true
            frame = K.pop(); E = frame[3]
            pc = frame[2] if V == 0 else frame[1]

        elif op == OP_RET:                  # end of a body
            if not K:
                return V                    # nothing left to return to -> done
            pc = K.pop()[1]                 # resume the caller
```

Most of these you have already read across from Chapter 4 in the §6.2 table: `INT`,
`VAR`, `LAM`, `APP_START`, `APPLY_ARG`, `IF_START`, `APPLY_IF` are its EVAL and APPLY
transitions, doing exactly what they did there, dispatched on an integer instead of
on the shape of `C`.  Two things are genuinely new at run time, and both come from
the same fact: the bytecode is *flat*, with no surrounding tree to consult.

The first is the **return address**.  When a CEK function returned, the machine knew
what to do next because the caller's frame was sitting right there on `K`, built from
the tree.  A flat stream has no tree, so after a body's `RET` the VM must be *told*
which instruction to resume at.  That is what `FRAME_RET` is: `OP_CALL`, just before
it jumps into the body, pushes `(FRAME_RET, pc + 1)`, the address of the instruction
right after the call.  When the body reaches `OP_RET`, it pops that frame and sets
`pc` to it, landing exactly where the caller left off.  And if `K` is empty at a
`RET`, there is no caller waiting: the program is finished and `V` is its answer.

The second is how **`OP_TCALL` earns tail-call optimization**, and it is a single
missing line.  `TCALL` does everything `CALL` does, *except push the `FRAME_RET`*.  A
tail call is the last act of its caller, so there is nothing to come back to: whatever
the callee returns is the caller's result, and the callee can hand it straight past
the caller to whoever the caller would have returned to.  By not saving a return
address, a tail call leaves `K` exactly as tall as it found it: a hundred thousand
tail calls ride one height on `K` and never climb.  It is the same constant-space
guarantee as Chapters 3–5, now split cleanly across the two phases: *decided* at
compile time (the `TCALL`-vs-`CALL` choice in §6.4) and *realized* at run time as one
push that does not happen.


## 6.6 Watching it compile and run

Take the smallest program with a call in it, `((lambda x x) 7)`, and compile it.  The
toy's `disassemble` prints the bytecode:

```
    0  APP_START
    1  LAM        x 3
    2  JUMP       5
    3  VAR        x
    4  RET
    5  APPLY_ARG
    6  INT        7
    7  TCALL
```

Read it against §6.4 and every instruction has a reason.  `APP_START` (0) opens the
application.  The function `(lambda x x)` compiles to `LAM x 3` (1), build a closure
whose body starts at pc 3, followed by `JUMP 5` (2) to skip the inline body, which is
`VAR x` (3) and `RET` (4).  Then `APPLY_ARG` (5), the argument `7` as `INT 7` (6), and
because the whole program is in tail position, the call is `TCALL` (7).  The two
addresses inside `LAM` and `JUMP` are exactly what backpatching filled in: the body
sits at 3, and the code past it is at 5.

Now run it.  Here is the VM stepping through, one step per block; `#<x>` is the
closure `LAM` builds and `{x:7}` the scope `TCALL` opens.  Each step shows the
program counter `pc` (with the instruction it points at), the stack `K`, and the
value `V`, and on the `>` line what the step does:

```
step 1
   pc  0  APP_START
   K   []
   V   –
   >   push (FRAME_ARG, E) to remember the scope; pc -> 1

step 2
   pc  1  LAM x 3
   K   [ARG]
   V   –
   >   V := closure #<x> (its body starts at pc 3); pc -> 2

step 3
   pc  2  JUMP 5
   K   [ARG]
   V   #<x>
   >   jump past the inline body: pc -> 5

step 4
   pc  5  APPLY_ARG
   K   [ARG]
   V   #<x>
   >   pop ARG (restore E); push (FRAME_CALL, V=#<x>); pc -> 6

step 5
   pc  6  INT 7
   K   [CALL #<x>]
   V   #<x>
   >   V := 7 (the instruction's constant); pc -> 7

step 6
   pc  7  TCALL
   K   [CALL #<x>]
   V   7
   >   pop closure #<x>; bind x:=V=7; pc -> 3 (tail: no return frame)

step 7
   pc  3  VAR x
   K   []
   V   7
   >   V := lookup(x) in E = 7; pc -> 4

step 8
   pc  4  RET
   K   []
   V   7
   >   K empty, so return V = 7
```

Two things to see.  Follow the `pc` line: it is no longer a walk over a tree but a
path through a list, mostly `+1`, with three jumps (the `JUMP` at step 3, the `TCALL`
into the body at step 6, and the final `RET`).  And follow `K`: it never holds a
`FRAME_RET`, because the only call in the program is a *tail* call, so no return
address is ever saved and `K` stays flat.  A non-tail call (the function position of
a nested application, which the compiler emits as `OP_CALL`) is where you would see a
`FRAME_RET` pushed and later popped by the body's `RET`, restoring `pc` to just after
the call.  This program has none, which is the whole point of `TCALL`.


## 6.7 Running it, and the arc closes

`python IttyBittyLisp6.py` disassembles that first program and then runs the same six
that Chapter 4 did, identical answers, a different machine underneath:

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

That is the last of the six machines.  Look back down the whole staircase now,
because the same tiny language ran on every step:

1. **The naive evaluator** (Ch 1): `lEval` walks the AST by recursion; the Python
   call stack *is* the interpreter's memory, and deep programs overflow it.
2. **Closures and environments** (Ch 2): functions capture their defining scope; the
   environment becomes a chain.
3. **The looping evaluator** (Ch 3): tail calls stop recursing and start looping, so
   they run in constant space.
4. **The CEK machine** (Ch 4): the continuation is pulled off Python's stack and made
   an explicit register `K`; now *no* control, tail or not, rides the host stack.
5. **The CEK machine, complete** (Ch 5): the whole language returns as more frame
   kinds, the machine's shape unchanged.
6. **The bytecode VM** (Ch 6): the machine's dispatch is precomputed into a flat
   instruction stream, and running a program becomes: compile once, then loop.

Each step kept the meaning of the language identical and changed only the machine
that carried it, and each change was a move toward how real language runtimes are
actually built.  What you have now, in miniature, is the shape of a working
implementation: a parser (Chapter 1 handed you that), a compiler, and a virtual
machine, with a constant pool, closures, tail-call optimization, and an explicit
continuation stack.  Those are not toys' ideas; they are the ideas inside Python's own
bytecode interpreter, the JVM, and every production Scheme.  You built the small
version of each, and you can read the large ones now.


## 6.8 Challenges

- **Compile the full language.**  Bring Chapter 5's forms (`set!`, `begin`, `let`,
  multi-argument calls, primitives, real booleans) onto the VM.  Each one is a new
  opcode or two (an `OP_SET`, an `OP_SEQ`, an `OP_PRIM`), compiled the same way you
  compiled `if` and application here.  The machine's shape does not change; the opcode
  table just grows: the exact lesson Chapter 5 taught for frames, now for opcodes.

- **A real bytecode.**  Right now each instruction is a Python tuple.  Turn the program
  into a flat array of *integers* (opcodes and small operands) with a separate
  constant pool for numbers and variable names.  This is what "bytecode" literally
  means, and it is the representation an interpreter written in C would use.

- **Peephole optimization.**  Add a pass over the finished bytecode that cleans it up:
  fold a constant expression like `(+ 1 2)` into a single `INT 3`, or collapse a
  `JUMP` that lands on another `JUMP`.  Because the program is now a flat list, these
  are simple local rewrites, the first taste of an optimizing compiler.

- **A step debugger.**  Extend `disassemble` and `run_vm` into a stepper that prints
  `pc`, `V`, and `K` before each instruction, and pauses.  You already saw its output
  by hand in §6.6; make the machine print it, and you can watch any program execute
  one opcode at a time.

- **First-class continuations (stretch).**  As in Chapters 4 and 5, `K` is an ordinary
  list you own, but now its frames include return addresses, so a captured `K` is a
  frozen snapshot of *the entire rest of the computation*, program counter and all.
  Add a `call/cc` opcode that captures `K`, and one that reinstates a captured `K`, and
  you have first-class continuations on the VM, the deepest idea of the series, riding
  the simplest machine of the series.
