# Dialog Design Notes

*Work in Progress!*

Dialog is a modern variant of [Prolog](https://en.m.wikipedia.org/wiki/Prolog), crossed with some ideas from
[Smalltalk](https://en.m.wikipedia.org/wiki/Smalltalk) (hence the name), but aiming for the distribution and concurrency
properties of [Erlang](https://en.m.wikipedia.org/wiki/Erlang_%28programming_language%29). Dialog exists purely in note
form at present, and there are large conceptual gaps still to be filled. 

## Aims
Dialog is an experimental language for exploring some new points in the programming language design space. It probably will never have a fully functioning implementation.

## Philosophy

In 1979, Robert Kowalski of Imperial College [famously articulated the dream of Prolog][1] using the equation:

    Algorithm = Logic + Control

The ideal was that the logic of an algorithm can be specified independently of the control aspects, allowing both the
logic and the control aspects to specified declaratively. In reality, the declarativity of Prolog is compromised by
non-logical operators such as the *cut*, which can render a program with only an imperative/procedural reading. Still,
it is possible to write efficient and logical Prolog code, and variants such as Datalog or XSB's tabling show that
alternative control strategies can lead to efficient evaluation of expressive declarative languages.

In Dialog, we extend this ideal from algorithms to real-world *agents* with the equation:

    Agent = Logic + Control + Interaction

Here *interaction* captures interactions with users, other components (agents in Dialog) and the wider environment in
which the program is situated. It is the messy stuff on the edges: user interfaces, networking, I/O and so on.

Several agent-oriented programming languages already exist, but hopefully Dialog differs from them in pushing the core ideas a little bit further.

## Overview

NB: sections named with a hat tip to [Functional core, imperative shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell).

### Declarative core

Dialog aims, as far as possible, to stick to declarative programming. The core is therefore a declarative logic programming language, with the following features:

* No features that lack a declarative, logical semantics
* Syntactic sugar for functions and typical functional programming idioms
* Arithmetic via constraints 
* Strong negation and some form of nonmonotonic negation-as-failure, probably using the well-founded semantics. I'm also interested in exploring support for abduction.
* A rich set of basic datatypes: unicode strings, arbitrary-precision integers (at least), lists, sets, maps/dictionaries, vectors/arrays/matrices, etc.
* Support for DCGs or a similar parsing/grammar definition mechanism.

Note: my Prolog knowledge is a little out of touch, so this will probably evolve as I learn what the state of the art is.

This constitutes the "logic" part. The "control" part will allow specifying aspects of how queries are performed over this logical core, borrowing from relational databases and other techniques. This is even less worked out than the logic part, but will likely include:

* Ability to specify control strategies independently of logic
* Support for parallel execution
* Support for alternative search strategies, such as iterative deepening
* Ability to specify indexing, tabling, etc

### Imperative shell

At some point, all programming language have to let you interact with the world. Philosophically, Dialog makes this split:

* Deciding *what to do* is best done in declarative code
* Deciding *how to do it* is done imperatively

So Dialog has an imperative shell that gets things done. Imperative code can call into declarative code to make decisions, but the reverse is not possible. This is similar to IO in Haskell, but enforced by the language rather than the type system. (Dialog is not statically typed. I like type systems, but that's not what I'm interested in for this language).

The imperative part of Dialog is quite different to a normal procedural or OO language. Firstly, you don't directly execute actions in Dialog. Rather, a *method* in Dialog is a (pure) function that *returns* one or more commands to carry out. These commands are then executed by the runtime. As well as atomic commands, there is a basic set of the usual imperative control structures to allow compound commands to be created:

    do Command; Command ... end
    for each Var in Query ...
    for some Var in Query ...
    try Command on Signal -> Handler ... end

and so on. This is deliberately quite a restricted set of things you can do. In particular, there are no unbounded loops. As well as executing such commands locally, an agent can also *send* commands to another agent, in which case they execute as an atomic transaction from the point of view of observers:

    foo <- do
        bar ...
        bob ...
        quux ...
    end
    
where `foo` is the agent being sent the commands and `bar ...` etc are verbs provided by that agent. In this way, Dialog's commands are an instantiation of the Command Pattern popular in OO. They are data structures that can be examined, manipulated, logged, and sent over the network. You can pattern-match against them and transform them however you like.

### Methods and contracts

Agents declare the verbs that they provide using declarations like

    verb goto(Destination)
    
They can then implement these with one or more method declarations:

    to goto(Destination)
        ...
    end

These methods can declare pre-conditions that are used to select an appropriate implementation:

    to goto(Destination)
        requires access_to(car)
        ...
    end

In this case, the method will only be chosen if the access_to(car) predicate is satisfied. If it isn't then another method implementing the same verb will be chosen. If no methods are applicable then the command will fail.

Methods can also declare post-conditions, using the `achieves` declaration:

    to goto(Destination)
        requires access_to(car)
        achieves at(Destination)
        ...
    end

If the method is executed and doesn't achieve the post-condition then that will cause a failure too.

You can also attach pre- and post-conditions to verbs, in which case they apply to all methods implementing that verb.

The `to` form is syntactic sugar for defining a special `to(Action, Commands)` relation. That selects commands based on the action being executed and any pre-conditions. Preconditions translate into normal conditions on this relation:

    to(goto(Destination), Commands) :-
        access_to(car),
        Commands = ...
    to(goto(...)) :- ...

Thus action selection is normal back-tracking search, guided by pre-conditions. This is committed-choice though; once a method has been selected it is committed to and failures will cause the whole action to abort.

### Teleoreactive programming

You may have noticed that there was no conditional statements in the command language fragment I showed in the last section. This is deliberate. Instead of having complex if-then statements all over the place, decisions are handled by declarative code. The syntax of methods allows them to include conditions in the following form:

    to goto(Destination)
      when at(Destination) -> done
      when at(Here), route(Here, Destination) -> drive(Here, Destination)
      else -> fail "no route"
    end

But this isn't just a simple if-then-else chain with different syntax. Instead, agents implement a variant of Nils Nilsson's [Teleo-Reactive Programs](http://teleoreactiveprograms.net). When a method is selected the agent adopts it as an *intention* and keeps evaluating it until either it succeeds (post-conditions are satisfied) or fails. On each cycle, the agent scans through the list of conditions and evaluates the first one that is true. Conditions are queries against the declarative core of the agent.

Intentions are like threads. Multiple intentions can be active at a time. If a method never succeeds or fails then it remains active forever, allowing maintenance tasks to be programmed.

Intentions execute in a [turn-based fashion as in E](http://erights.org/elib/concurrency/turns.html). Conceptually, each agent is its own vat: calling other methods on the same agent happens synchronously, while sending messages to other agents is asynchronous. (I will probably ignore promises for now, until I find I need them; sending compound commands to other agents seems conceptually simpler and with a more easily understood semantics and concurrency story.

### Goal-driven execution

The pre- and post-conditions on methods and verbs can be used to perform classical STRIPS-like planning. (And maybe moving beyond STRIPS in future). An agent can execute the command

    achieve at(some_destination)

to activate goal-based execution. This will find verbs whose post-conditions match the goal, then recursively try to `achieve` all pre-conditions for those verbs, and so on until it finds a plan or gives up. If a plan is found then it is executed in the normal teleoreactive execution model described before. 

### Event-driven execution

`on Event -> Commands‘

`foo <- event(...)!`

### Services and multi-agent alliances

Agents can advertise services within a vat. Agents can make acquaintances across vats (and distributed). These are always typed in terms of services. Service definitions include verbs with pre- and post-conditions. If an agent trusts another agent then `achieve` will consider verbs provided by that agent, resulting in multi-agent planning.

Pre- and post-condition predicates are evaluated in the context of the agent providing the service, so queries may be sent to determine plan viability. 

### RESTful capability security (RESTCaps)

## Some details

A Dialog program consists of sets of *agents* collaborating to provide *services*. A service defines a set of operations
that are available to consumers of that service. One or more agents will provide that service, and the relationship
between agents and services can be dynamic (e.g., to respond to agents becoming unavailable or acting maliciously). The
operations in a service contract are defined by *verbs* that can have pre- and post-conditions:

``` 
service bookshop:
    pred in_stock(Book)
    func cost(Book) -> Amount

    verb buy(Book, Money):
        requires in_stock(Book), Money = cost(Book), sender(Customer)
        achieves bought(Customer, Book)
    end
end
```
Here we have a fictitious service that defines a single operation for buying a book using some money. It has three
pre-conditions:

1. That the book is in stock (determined by a predicate, also defined by the service).
2. That the money is equal to the cost of the book.
3. That there exists a sender of this message, that we unify with the variable `Customer`.

The post-condition just asserts that after the operation completes, the customer has bought the book (using the variable
bound in the pre-condition).

An implementation of the bookshop service can be provided along the following lines:

``` 
bookshop waterstones:
    -- Predicate definitions
    in_stock(Book) if ...
    cost(Book) -> ...

    -- Action definitions
    to buy(Book, Money) ->
        ...
end
```

An agent can make use of this service as follows:

``` 
    require bookshop(Shop), Shop:in_stock(some_book)
    Shop:buy(some_book, Cash)
```

The first statement unifies the variable `Shop` with some implementation of the bookshop service (located from the
environment), and aborts if none can be found. More complex queries can be performed to specify exactly what kind of
bookshop is required, for instance finding one with a particular book in stock.

In addition to this imperative service interface, agents can also use a more declarative goal-based approach to
performing actions. For instance, the same can be achieved via the single command:

``` 
    achieve bought(self, some_book)
```

This goal-achievement command will automatically look for verbs whose post-conditions satisfy the goal, potentially then
finding other verbs that will satisfy the pre-conditions of that verb, and so on in the style of operator-based
planning systems (e.g., STRIPS). Once a plan has been formed, it will then find service providers that satisfy the plan
operators and recruit them to perform the plan. In this way, a single command might result in a dynamic collaboration of
agents fulfulling service contracts across distributed resource clusters. 

When operations fail, services can *signal* events. This allows other agents to react to plan failure and dynamically
replan or carry out compensating transactions as appropriate. Signals play the role of exceptions in other languages,
but are more similar to Common Lisp conditions or coroutines. Rather than unwinding a stack, a signal temporarily pauses
the flow of execution and begins an interaction with other agents to determine how to proceed. Once this interaction
ends, the flow may either continue with some alternative, or abort.

This vision may sound inspiring or terrifying depending on your outlook and experience. I suspect it will mostly be the
latter for those with most experience. This is why I firmly believe that Dialog should be an experimental language and
not attempting to be a production-ready language. I believe that only by significantly raising the level of abstraction
in programming languages can we hope to embrace the planet-scale distributed and complex systems of the coming century.
Only by building such languages and learning from them will we understand the tools and techniques that are most suited
to such powerful programming abstractions.

This draft is also something of a personal manifesto for what an agent-oriented programming language should be. Not
simply a logic programming language with some reactive rules and “BDI” concepts bolted on to the side, but to embrace
the principles of goal-driven declarative programming and dynamic agent collaborations.

One day I will build this, perhaps when my own little agent has grown up.

## Agents and Messages

Agents are similar to processes in Erlang, or actors in actor-based programming languages, or even a bit like
active-objects. Rather than having variables or fields, however, state in a Dialog agent consists of a Prolog-like
declarative *belief-base*. This data language is quite expressive, and goes beyond Prolog in a number of ways, discussed
in the following sections.

Wrapped around this declarative core is an [imperative shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
that allows an agent to carry out transactions: updating its beliefs and sending messages to other agents. Messages come
in one of four forms:

1. **Declarative** messages end with a full stop and *tell* another agent some statement of fact: 
`recipient <- some_fact(a, b).`
2. **Interrogative** messages end with a question mark and *ask* another agent a question:
`recipient <- some_query(A, B)?`
3. **Imperative** messages do not end with any symbol and send a *command* to another agent:
`recipient <- some_command(...)`
4. **Exclamatory** messages end with an exclamation mark and inform the recipient of some *event*:
`recipient <- some_event(...)!`

Agents defined within the same module implicitly trust each other, and so will blindly believe what they are told and
happily let others see what they know (although good style would suggest to minimise sharing such details). Agents are
not directly visible to agents defined in other modules, but only via more controlled service contracts, which reduces
the potential for abstraction violations.

The first two forms of message are based on the predicate symbols that the recipient understands, and can include
free-form Dialog facts and queries. Imperative messages are based on verb symbols. Exclamatory messages are based on
event symbols.

``` 
to do_something(...) ->
    ...
on some_event(...) ->
    ...
```

## Lexical details
Source files are valid Markdown/CommonMark files. All Dialog code is either indented or fenced as per Markdown rules.

Dialog mostly follows Prolog conventions, but with some syntactic sugar. 

The form `f(a,b) -> ...` expands into `f(a, b, X) if X = ...`, providing some sugar for functions definitions, but can also be used to define datatypes. 
```
tree -> empty 
      | leaf(A) 
      | branch(tree, tree).
```
Expands into:

```
tree(X) if X = empty
         | X = leaf(A)
         | X = branch(X0, X1), tree(X0), tree(X1).

``` 

A convenient syntax is also provided for making a series of statements about a single object. For example, the following statements about a man called John:

```
person(john).
name(john, “John”).
date_of_birth(john, 1972-12-02).
spouse(john, mary).
```
Can be written using the colon-block form:

```
person john:
    name(“John”).
    date_of_birth(1972-12-02).
    spouse(mary).
end
```
We can combine this with the function form just given and so have:

```
person john:
    name -> “John”.
    date_of_birth -> 1972-12-02.
    spouse -> mary.
end
```


```
person john:
    name -> “John”
    date_of_birth -> 1972-12-02
    age(Now) -> Now:year - date_of_birth:year
end
```

## Language Elements
### Basic types

Dialog has a rich set of basic types:
* Arbitrary precision decimals: `1.23`, `42`, `-13e9`, plus hex notation: `0xFF`. A single numeric type for now, for
  simplicity.
* Strings: `“Hello, World!”` – Unicode (UTF-8). Note that balanced “smart quotes” can be used (English-style only). It's
  2019, we shouldn't have to be escaping quotes inside strings for lack of balanced quote characters. Single or double
  quotes can be used to delimit strings, in both straight and "smart" forms.
* [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) dates and times: `2018-04-04`, `2018-04-04T14:54:31Z`,
  `12:02:33.123` etc.
* Logical variables, starting with an underscore or capital letter: `_`, `Foo`,
* Function symbols: `foo`, `father(john)`. These form rational trees (i.e., are potentially infinite structures) as in [Prolog II](http://prolog-heritage.org/en/ph20.html). 
* Composite data types: lists/arrays `[1, 2, 3]` and sets `{1, 2, 3}` (conceptually treated as balanced trees, but implementations are free to use other implementations). Associative maps can be specified using a syntax like `{ foo -> 12, bar -> 13 }` but this is just sugar for a set of function symbols `{ foo(12), bar(13) }` as we shall soon see. 

### Sugar
There are some syntactic sugar beyond Prolog to make programming natural and enjoyable. Functions can be declared using an Erlang-like syntax:
```erlang
sum([]) -> 0
sum([X,..Xs]) -> X + sum(Xs)
```
This gets translated into equivalent clauses (note, no attempt is made to ensure that functions are functional, this is deliberate):
```prolog
sum([], 0)
sum([X,..Xs], Y) if sum(Xs, Y’), Y = X + Y’
```
Actually the form `f(x, y) -> z` is translated into `f(x, y, z)` while f is registered as an evaluated function in the current scope. Evaluated functions are replaced with fresh existential variables, in a sort of reverse [Skolemization](https://en.m.wikipedia.org/wiki/Skolem_normal_form). A new goal condition is then added to each clause that unifies this fresh variable as the last argument to the function. Thus:
```
f(X) -> X
g(X) -> f(X+1)
```
Becomes:
```
f(X,X)
g(X,Y) if f(X+1,Y)
```
Note that arithmetic expressions are evaluated as [constraints under CLP(R)](https://en.m.wikipedia.org/wiki/CLP(R))) semantics. 
Note that unlike Prolog, Dialog uses an evaluation strategy that does not depend on order of clauses or goals for completeness. 
The translation is intended to mimic DCG grammar rules. Where the DCG `-->` syntax adds two implicit arguments, `->` adds one. 
Some examples of things you can do with this syntax, apart from defining functions include:
* Constants: `pi -> 3.14159...` then use as `area(R) -> pi * R^2`
* Defining associative maps as sets of function symbols: `{ a -> 42, b -> 13 }` expands into `{ a(42), b(13) }`. Equality of function symbols includes all arguments so the same “key” here could appear multiple times with different mappings (a multimap). Note that as the arrow notation binds symbols in the current scope (and brackets delimit a scope), mappings can refer to each other, mutually recursively: `{ a -> b, b -> a + 1 }`. 
* Defining algebraic datatypes: `color -> red | blue | green` becomes: `color(X) if X = red | X = blue | X = green`, and `tree -> empty | branch(Left, Val, Right)`
### Declaration blocks
A declaration block can be used as follows:
``` python
person john:
	name -> “John”
	date_of_birth -> 1972-Jan-30
	age(Now) -> year(Now) - year(dob)
```
In this form each sub-declaration becomes a new toplevel declaration with the term immediately preceeding the colon as an extra first argument:
```prolog
person(john)
name(john, “John”)
date_of_birth(john, 1972-Jan-30)
age(john, Now, Age) if
	year(Now, NY), dob(john, DoB), year(DoB, BY), Age = NY - BY
```
These blocks can be nested, allowing quite succinct definition of complex relationships. 
### Guards
The colon form can also be used within the arguments of a function symbol. It works in much the same way, but rather than creating new declarations it adds new constraint goals to the clause it is part of:
```
fac(0) -> 1
fac(N : integer) -> N * fac(N-1)
```
Becomes:
```
fac(0, 1)
fac(N, F) if integer(N), fac(N-1, F’), F = N * F’
```
Guards can also be used as accessors:
```
John = person(“John”, ...)
name(person(Name, ...)) -> Name
John:name // expands to name(John)
```
## Higher order programming
Dialog supports passing functions and predicates as arguments and returning them. 
First-class functions can be defined using the arrow syntax: `f(g(Y) -> Y+1, ...)`. Anonymous (but non-recursive) functions can then be defined using an anonymous functor: `f(_(Y) -> Y + 1, ...)`

[1]: https://www.doc.ic.ac.uk/~rak/papers/algorithm%20=%20logic%20+%20control.pdf
