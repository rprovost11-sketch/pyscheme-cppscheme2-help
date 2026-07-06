# Interlude: Your Closures Are Already Objects

### *Object-orientation, already present in the Chapter 2 closure*

*This is an optional side road from Chapter 2. You can take it now and return, or
skip it entirely: nothing in Chapters 3 through 7 depends on it, and nothing in
it changes the evaluator. It assumes only that you have read Chapter 2, in
particular §2.2 ("Closures, from the programmer's seat"), whose `make-counter` we
pick up below.*

Every core idea in object-oriented programming, an object that bundles private
state with the operations on it, encapsulation, message dispatch, polymorphism,
even inheritance, is *already present* in the closure you built in Chapter 2. Not
"can be bolted on." Already there. This interlude adds nothing to the evaluator
and asks nothing of the language that Chapter 2 does not already provide. All it
does is build one ordinary object out of a `lambda` that returns a `lambda`, and
then set the vocabulary of object-orientation beside the parts of that closure,
at which point you find the two are the same list.

So the surprise here is not *what* we are building. That is announced, and you
should read knowing the destination. The surprise is *how little it takes to get
there*: that the objects turn out to have been free, riding on the two
subexpressions Chapter 2 already spent on plain lexical scope, in the smallest
and first interpreter of the series. So read with one question held live:
**where is the object machinery?** We will reach the end and find there was none.

We build in four moves, an object with private state (`make-account`), the
encapsulation that seals it, polymorphism across two kinds of account that share
a message set, and inheritance by one object delegating to another, and then stop
and account for what it cost.


## 1. Recall the counter

Chapter 2 left you with this closure:

```scheme
(set! make-counter
  (lambda ()
    (let ((count 0))
      (lambda () (begin (set! count (+ count 1)) count)))))

(set! c1 (make-counter))
(c1)   ; => 1
(c1)   ; => 2      count is captured, and set! mutates it in place
```

`c1` is a function carrying private, persistent state that you can reach only by
calling it. Hold that shape in mind: a function with a memory of its own. It is
already half of an object, the private, persistent state. What it lacks is a way
to offer *more than one* operation on that state. We add exactly that, and get
the whole thing.


## 2. Build an object: an account

An object, built as a closure, is an outer function that takes the object's
initial fields and returns an inner function that takes a *message* and acts on
it. Here is one. `make-account` closes over a `balance` and returns a function
that branches on a `msg`, `deposit`, `withdraw`, or `balance`, deciding for
itself what each one does:

```scheme
(set! make-account
  (lambda (balance)
    (lambda (msg amount)
      (if (= msg 'deposit)
          (begin (set! balance (+ balance amount)) balance)
          (if (= msg 'withdraw)
              (begin (set! balance (- balance amount)) balance)
              (if (= msg 'balance)
                  balance
                  (print 'unknown-message)))))))
```

One piece of notation before we run it. `'deposit` is shorthand for
`(quote deposit)`, the same `quote` you met in Chapter 1; the parser turns one
into the other, and its full story is Chapter 7. Here it just makes a **symbol**,
a plain value, to use as the name of a message. The dispatch is then nothing but
an `if`-chain comparing `msg` against each known symbol with `=`. Symbols compare
by name, so `(= msg 'deposit)` is true exactly when the caller sent `'deposit`.

Evaluate `make-account` and nothing visible happens; you get back a function.
Call that function and, still, nothing much:

```
lisp> (set! acct (make-account 100))
#<procedure (msg amount)>
```

(`#<procedure (msg amount)>` is how the interpreter prints one of our functions,
"procedure" being Scheme's word for a function, listing its parameter names.)
`acct` is an object holding a private `balance` of 100. So far it has *done*
nothing, because an object acts only when it is sent a message.


## 3. Send it messages

Send it a `msg` and an `amount`, one at a time:

```
lisp> (acct 'deposit 50)
150
lisp> (acct 'withdraw 30)
120
lisp> (acct 'balance 0)
120
```

Three calls, three different behaviors, chosen by the first argument. The
`balance` climbs and falls and *persists* between calls: it is the same captured
`balance`, living on in `acct`'s closed-over scope. (The `balance` message
ignores its `amount`, but the function still takes two parameters, so we hand it
a throwaway `0`.) Try to read the balance from outside and you cannot: there is
no expression in the language that reaches `balance` except by calling `acct`
with a message it chooses to answer.

And a second account keeps its own books entirely, untouched by the first:

```
lisp> (set! acct2 (make-account 500))
#<procedure (msg amount)>
lisp> (acct2 'withdraw 200)
300
lisp> (acct 'balance 0)
120
```

`acct2` starts fresh at 500 and falls to 300, while `acct` is still exactly where
we left it. Each call to `make-account` opened its own scope with its own
`balance`, so the two never collide.


## 4. Name the parts

Take inventory of `acct`:

- it holds **private state** (`balance`) that **persists** across calls,
- it responds to a fixed set of **named operations** (`deposit`, `withdraw`, `balance`),
- its state is reachable **only** through those operations,
- and each one you make is **independent** of the rest.

Private state, a public set of named operations, the state sealed behind them,
many independent instances: that is an **object**, and, as promised, you built it
with nothing but a `lambda` that returns a `lambda`. Set the vocabulary of
object-orientation beside the parts of the closure you just wrote and, in this
language, the two lists are the same list:

| Object-oriented idea | is, here |
|---|---|
| an object | the inner function `make-account` returns |
| a constructor (`new`) | `make-account` itself |
| private fields | the variables it closes over (`balance`) |
| a method | one branch of the `if`-chain |
| message dispatch | the `if`-chain testing the quoted `msg` symbol |
| encapsulation | lexical scope: the fields are unreachable except through the body |

None of those rows asked anything of the evaluator that Chapter 2 does not
already provide. Every row is a *use* of closures, not an extension of them.
(Keep the question in view: still no object machinery in sight.)


## 5. Encapsulation, and why it is stronger than it looks

There is no expression in the language that reads `balance` directly. The name
lives in the constructor's scope; the only reference to that scope is inside the
object function's body. You reach the state *only* by sending a message the body
chooses to answer. That is not a convention enforced by an access keyword like
`private`. It is a structural fact of lexical scope: the field is private because
there is physically no path to it. This is the same guarantee the evaluator's own
`Environment` class relies on (its `_bindings` are reachable only through
`lookup` and `set`), now available to programs written *in* the language.


## 6. Polymorphism: same messages, a different object

A second constructor answers the identical message set but behaves differently.
An overdraft account lets the balance fall as low as `-limit`:

```scheme
(set! make-overdraft-account
  (lambda (balance limit)
    (lambda (msg amount)
      (if (= msg 'deposit)
          (begin (set! balance (+ balance amount)) balance)
          (if (= msg 'withdraw)
              (if (< (- balance amount) (- 0 limit))
                  (print 'overdraft-refused)
                  (begin (set! balance (- balance amount)) balance))
              (if (= msg 'balance)
                  balance
                  (print 'unknown-message)))))))
```

(The `(- 0 limit)` is just how we negate `limit`, since our `-` takes two
arguments; `(- 0 50)` is `-50`.) A client that only ever *sends messages* neither
knows nor cares which kind of account it holds:

```scheme
(set! net-after-fee
  (lambda (account)
    (begin (account 'withdraw 5)     ; a fee, through the shared interface
           (account 'balance  0))))
```

```
lisp> (net-after-fee (make-account 100))
95
lisp> (net-after-fee (make-overdraft-account 100 50))
95
```

Dispatch happens *inside* each object, so the caller is polymorphic for free.
`net-after-fee` sends `withdraw` and `balance` and never asks what it is holding;
any object that answers those two messages qualifies. The interface is a matter
of behavior ("responds to `withdraw` and `balance`"), not of type. (That
behavior-over-type style of interface has a name in the wider programming world,
*duck typing*, but the mechanism is just the message-send you already have.)


## 7. Inheritance by delegation

An object can close over *another object* and forward the messages it does not
handle itself. A logging account adds behavior to `deposit` and hands everything
else to a plain account held as `parent`:

```scheme
(set! make-logging-account
  (lambda (balance)
    (let ((parent (make-account balance)))
      (lambda (msg amount)
        (if (= msg 'deposit)
            (begin (print 'logging-deposit) (parent msg amount))
            (parent msg amount))))))          ; delegate everything else
```

```
lisp> (set! log-acct (make-logging-account 200))
#<procedure (msg amount)>
lisp> (log-acct 'deposit 25)
logging-deposit
225
lisp> (log-acct 'withdraw 25)
200
lisp> (log-acct 'balance 0)
200
```

The captured `parent` *is* the inheritance link. Method lookup, "handle it here
or pass it up," is the same walk that `Environment.lookup` performs over scopes,
now performed by hand over objects. Override is "handle it here"; inheritance is
"pass it up." Both are ordinary function calls.


## 8. The double surprise

Now collect on the question you have been holding. Re-open `IttyBittyLisp2.py`
and look for the machinery that made any of that possible. There is none. The
whole of it, fields, methods, dispatch, encapsulation, polymorphism, inheritance,
rode on two subexpressions that were already there to make plain variables and
functions work:

```python
elif expr[0] == 'lambda':
    params, *body = expr[1:]
    return Function(params, body, env)     # captures the defining env: this is the closure
```

```python
initialBindings = dict(zip(fn.params, args))
new_env = Environment(parent=fn.env, bindings=initialBindings)   # chains the call off the CAPTURED env
```

The first packages code together with the meaning of its free variables into a
single value you can hold and pass around. The second makes a call see its
definition-time environment rather than the caller's. Both lines were spent on
lexical scope, which the language needed anyway. The objects were free. *That* is
the first surprise.

Here is the second. The evaluator you built all of that on is the **smallest and
first** one in the series, a plain recursive interpreter you can read in a single
sitting, on the order of a hundred lines, with no object feature anywhere in it.
And you could not have made the language more expressive than it already was:
that was settled the moment `lambda` captured its environment. Everything the
series does after Chapter 2, trampolining for tail calls (Chapter 3), the CEK
machine (Chapter 4), the bytecode VM (Chapter 6), changes *how* programs run,
never *what* can be written. (The lone exception is Chapter 4, where the
continuation becomes a first-class value and genuinely adds to the language, a
different fork.) Objects were not waiting on a bigger engine. They were reachable
from the smallest one, because they were never a feature to be added, only a use
of what was already there.


## 9. Running it, and challenges

The complete working code is in `examples/IttyBittyOO.py`. It imports `lEval` and
the global environment from `IttyBittyLisp2.py` *unchanged* and runs the programs
above:

```
python examples/IttyBittyOO.py
```

Three things to try, each building on what this interlude gave you:

- **Give the logging account a real parent chain.** Have the *logging* account
  wrap an *overdraft* account instead of a plain one, so behavior composes
  through two delegation layers. Confirm that `overdraft-refused` still surfaces
  all the way back to the caller.

- **Make dispatch data, not code.** Replace the `if`-chain with a lookup in a
  list of message-to-function pairs, built once when the object is created. This
  is the closure-object turning into a *method table* before your eyes, which is
  exactly how real object systems dispatch (the traditional name for that table
  is a *vtable*).

- **Build a `class`, then find its ceiling.** Write a function that takes a list
  of `(message . method)` pairs and returns a dispatching object, so you stop
  writing the `if`-chain by hand. Everything it needs is already in Chapter 2, so
  you can write it today. Then notice what it *cannot* reach: the helper still
  makes you quote every message name, wrap each method in a `lambda`, and cons
  the pairs together yourself. A function runs on *values*, so it can tidy the
  values but never the code you type. Removing that last layer of noise, so you
  could write simply `(define-class account (balance) (deposit (amount) ...) ...)`,
  means rewriting code *before* it runs. That is a **macro**, and it is where
  this side road rejoins the main series.


## 10. Coda: two names for one thing

You have just lived one half of a well-known koan, written by Anton van Straaten
in 2003:

> The venerable master Qc Na was walking with his student, Anton. Hoping to
> prompt the master into a discussion, Anton said, "Master, I have heard that
> objects are a very good thing, is this true?" Qc Na looked pityingly at his
> student and replied, "Foolish pupil, objects are merely a poor man's closures."
>
> Chastised, Anton took his leave from his master and returned to his cell,
> intent on studying closures. He carefully read the entire "Lambda: The
> Ultimate..." series of papers and its cousins, and implemented a small Scheme
> interpreter with a closure-based object system. He learned much, and looked
> forward to informing his master of his progress.
>
> On his next walk with Qc Na, Anton attempted to impress his master by saying,
> "Master, I have diligently studied the matter, and now understand that objects
> are truly a poor man's closures." Qc Na responded by hitting Anton with his
> stick, saying, "When will you learn? Closures are a poor man's object." At that
> moment, Anton became enlightened.

The joke has a precise technical core: the two constructs are *interconvertible*,
so neither is fundamental.

- **"Objects are a poor man's closures."** Anything an object does, bundle state
  with behavior and hide the state, a closure does directly. The captured
  variables are the private fields; the body is the method. That is the direction
  you just took.

- **"Closures are a poor man's object."** It runs the other way too. A closure is
  just an object with a single method (call it) and its captured environment as
  the instance fields. In an object-oriented language you would build a closure
  by making a one-method class.

The stick-blow is the point: each is definable in terms of the other, so treating
either as the "real" primitive is a confusion. They are two surface presentations
of the same underlying thing, *code paired with the environment it closes over*.
Which one looks primitive is an artifact of the language you happened to start
in.

One caution keeps this from being over-read. *Interconvertible* is a claim about
what can be defined in terms of what, not about how real systems are built.
Production object systems are implemented with method tables and class pointers,
not closures, and production closures are frequently compiled into objects. C++
makes the reversal concrete: a lambda there is turned into a class with a
call method and the captured variables as fields, so in C++ closures are built
literally *out of* objects, the exact opposite of what you did in Scheme. So "the
same thing" means the same *idea*, behavior bundled with private state, code
paired with the environment it closes over, not the same machine-level layout.
Which construct a language treats as primitive, and which it synthesizes from the
other, is a design choice, and different languages choose oppositely.

That is the whole reason the Chapter 2 evaluator runs object-oriented programs
without an object feature in it. It was never a matter of building objects on top
of closures: in this language the two are one idea, code paired with its captured
environment, so the objects were already there the moment the closures were.
