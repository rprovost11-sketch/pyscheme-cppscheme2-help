# Evaluator 6 - A Bytecode VM

*Continues from `EVALUATOR5-DOC`: the full CEK machine.  This is the last toy in
the series.*

The CEK machine (#4, #5) re-walks the AST every step and re-decides which
transition to run.  But for a *fixed program* that decision never changes:
`(lambda (x) x)` is a lambda the first time the machine sees it and every time
after.  Re-deciding it on every step is wasted work.

A **bytecode VM** decides it once, at *compile time*.  This is the cleanest
possible statement of "it's downhill after #4": a VM is the CEK machine with its
dispatch precomputed.

## The idea: number every transition

Give each CEK transition an integer -- an **opcode** -- then walk the AST a
single time and emit a flat list of those numbered instructions (the *bytecode*).
At run time there is no AST to dispatch on and no EVAL/APPLY state flag; the loop
reads the next opcode and does it.

The CEK machine of #4 had transitions like "C is a lambda -> build a closure" and
"FRAME_ARG on top -> evaluate the argument."  Each becomes one opcode:

| opcode | the CEK transition it bakes in |
|---|---|
| `OP_INT n` | a number literal -> `V = n` |
| `OP_VAR name` | a variable -> `V = E.lookup(name)` |
| `OP_LAM param body_pc` | a lambda -> `V = closure(param, body_pc, E)` |
| `OP_APP_START` | begin an application -> push `FRAME_ARG` |
| `OP_APPLY_ARG` | function value ready -> push `FRAME_CALL` |
| `OP_CALL` / `OP_TCALL` | argument ready -> enter the body (non-tail / tail) |
| `OP_IF_START` | begin an `if` -> push `FRAME_IF` |
| `OP_APPLY_IF` | test ready -> jump to the chosen branch |
| `OP_RET` | body finished -> resume the caller |
| `OP_JUMP target` | bookkeeping: skip over an inline body |

## Registers: `pc` instead of `C`

Two registers carry straight over from the CEK machine -- `E` and `K` -- and `C`
is replaced:

- **`pc` - the program counter**: the index of the next instruction.  Where the
  CEK machine held *an expression* in `C`, the VM holds *a position* in the
  instruction stream.  "Descend into a sub-expression" becomes "set `pc`."

`K` gains one frame the CEK machine never needed: **`FRAME_RET`**, a saved return
address.  In the CEK machine, "what to do after this call returns" was always
findable -- the surrounding AST was right there.  A flat instruction stream has
no surrounding tree, so the return `pc` must be stored explicitly, on `K`.  A
**tail** call (`OP_TCALL`) stores *no* return address, so `K` stays flat across
tail calls and TCO is still structural -- the same guarantee as #3, #4, and #5.

## The compiler

The compiler walks the AST once, in the same five cases the EVAL loop had, and
emits opcodes instead of taking transitions:

```python
def compile_expr( expr, out, tail ):
    if isinstance( expr, int ):
        out.append( (OP_INT, expr) )
        if tail: out.append( (OP_RET,) )
    elif isinstance( expr, str ):
        out.append( (OP_VAR, expr) )
        if tail: out.append( (OP_RET,) )
    elif expr[0] == 'lambda':                 # ['lambda', param, body]
        lam_idx  = len(out); out.append( None )    # reserve OP_LAM
        jump_idx = len(out); out.append( None )    # reserve OP_JUMP over the body
        body_pc  = len(out)
        compile_expr( expr[2], out, tail=True )    # a body is always tail position
        out[lam_idx]  = (OP_LAM, expr[1], body_pc)
        out[jump_idx] = (OP_JUMP, len(out))
        if tail: out.append( (OP_RET,) )
    elif expr[0] == 'if':                      # ['if', test, then, else]
        ...                                    # push OP_IF_START, compile test/then/else
    else:                                      # [fn, arg] -- an application
        out.append( (OP_APP_START,) )
        compile_expr( expr[0], out, tail=False )
        out.append( (OP_APPLY_ARG,) )
        compile_expr( expr[1], out, tail=False )
        out.append( (OP_TCALL,) if tail else (OP_CALL,) )
```

Two facts the CEK machine recomputed at run time are settled here, once:

- **Tail position.** The `tail` flag rides down the tree.  A call in tail
  position compiles to `OP_TCALL` (no return frame); otherwise `OP_CALL`.  The
  machine never asks "is this a tail call?" again -- the opcode already says so.
- **Body layout.** A function body is emitted *inline*, right after the `OP_LAM`
  that builds its closure, with an `OP_JUMP` in front so the closure-building
  path skips over it.  The body is entered only by a later `OP_CALL`/`OP_TCALL`
  jumping to its `body_pc`.

## What compilation produces

`((lambda (x) x) 7)` compiles to eight instructions:

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

Read it: `APP_START` begins the call; `LAM` builds the identity closure whose
body lives at `pc 3`; `JUMP 5` hops over that body (it is not run now, only when
called); `APPLY_ARG` records the function; `INT 7` is the argument; `TCALL`
enters the body at `pc 3` -- `VAR x`, then `RET`.

## The VM loop

```python
def run_vm( prog ):
    pc, V, E, K = 0, None, Environment(), []
    while True:
        op = prog[pc][0]
        if   op == OP_INT:    V = prog[pc][1];                            pc += 1
        elif op == OP_VAR:    V = E.lookup( prog[pc][1] );                pc += 1
        elif op == OP_LAM:    V = (VAL_CLOSURE, prog[pc][1], prog[pc][2], E);  pc += 1
        elif op == OP_JUMP:   pc = prog[pc][1]
        elif op == OP_APP_START:  K.append( (FRAME_ARG, E) );             pc += 1
        elif op == OP_APPLY_ARG:  E = K.pop()[1]; K.append( (FRAME_CALL, V) );  pc += 1
        elif op == OP_CALL:
            closure = K.pop()[1]
            K.append( (FRAME_RET, pc + 1) )
            E, pc = Environment( parent=closure[3], bindings={closure[1]: V} ), closure[2]
        elif op == OP_TCALL:
            closure = K.pop()[1]
            E, pc = Environment( parent=closure[3], bindings={closure[1]: V} ), closure[2]
        elif op == OP_IF_START:   K.append( (FRAME_IF, prog[pc][1], prog[pc][2], E) );  pc += 1
        elif op == OP_APPLY_IF:
            frame = K.pop(); E = frame[3]
            pc = frame[2] if V == 0 else frame[1]
        elif op == OP_RET:
            if not K: return V
            pc = K.pop()[1]
```

The only run-time dispatch is the `if/elif` over the integer `op`.  No
`isinstance`, no keyword check, no AST.

## A trace

Running those eight instructions (each row shows the state *after* the
instruction at `pc` executes):

| step | pc | op | V | E | K |
|---|---|---|---|---|---|
| 1 | 0 | `APP_START` | -       | `{}`    | `[ARG]` |
| 2 | 1 | `LAM x 3`   | closure | `{}`    | `[ARG]` |
| 3 | 2 | `JUMP 5`    | closure | `{}`    | `[ARG]` |
| 4 | 5 | `APPLY_ARG` | closure | `{}`    | `[CALL clo]` |
| 5 | 6 | `INT 7`     | `7`     | `{}`    | `[CALL clo]` |
| 6 | 7 | `TCALL`     | `7`     | `{x:7}` | `[]` |
| 7 | 3 | `VAR x`     | `7`     | `{x:7}` | `[]` |
| 8 | 4 | `RET`       | `7`     | `{x:7}` | `[]` -> **return 7** |

Compare it to the CEK trace of the same program in `EVALUATOR4-DOC`: the *shape*
is identical -- push the argument frame, build the closure, record it, evaluate
the argument, enter the body.  But here every "which step is next" choice is just
`pc` arithmetic, fixed when the bytecode was emitted, not re-derived from the AST
at each step.  And because the call sat in tail position the compiler chose
`OP_TCALL` (step 6), so no `FRAME_RET` was pushed and `K` is empty at `RET`.

## Why the minimal language again?

Compilation is the new idea, so -- exactly as #4 did for the machine -- this toy
strips the language back to pure lambda calculus + `if` to keep the *compiler* in
focus.  The full language of #5 compiles the same way: each extra special form
and primitive becomes a few more opcodes, and the VM loop gains a few more
`elif`s.  The shape does not change.

## Running the Example

The complete working code is in `examples/IttyBittyLisp6.py` (it prints the
bytecode for a small program before running the demos).

```
python examples/IttyBittyLisp6.py
```

That is the whole arc: from a naive recursive evaluator that overflows on a deep
loop (#1), through making tail recursion (#3) and then *all* control (#4) into
explicit, inspectable data, to compiling that machine into a flat bytecode a real
virtual machine could run (#6).  Every interpreter in between is a complete,
working language -- a place you could have stopped.

## Challenges

- **Compile the full #5 language.** Add opcodes for `set!`, `begin`, and `let`
  (compile it as a lambda application, exactly as #5 desugared it), and the
  primitives.  Almost all the work is in the compiler; the VM loop just grows one
  `elif` per opcode.

- **A peephole optimizer.** Walk the emitted instruction list and simplify
  obvious patterns -- an `OP_JUMP` whose target is the very next instruction can
  be dropped, for one.  This is something you can only do *because* the program
  is now flat data instead of a tree.

- **Serialize the bytecode.** The instruction list is plain data -- numbers and
  strings.  Write it to a file and load it back, so compiling and running become
  separate programs.  That separation (`.pyc`, the JVM's `.class`) is what makes
  a bytecode language a bytecode language.
