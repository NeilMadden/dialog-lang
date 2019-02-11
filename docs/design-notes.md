# Dialog Design Notes

*Work in Progress!*

Dialog is a modern variant of [Prolog](https://en.m.wikipedia.org/wiki/Prolog), crossed with some ideas from
[Smalltalk](https://en.m.wikipedia.org/wiki/Smalltalk) (hence the name), but aiming for the distribution and concurrency
properties of [Erlang](https://en.m.wikipedia.org/wiki/Erlang_%28programming_language%29). Dialog exists purely in note
form at present, and there are large conceptual gaps still to be filled. 

## Aims
Dialog is intended as a language to learn about programming, explore ideas, and have fun. It is not aimed at large team
software engineering. It is squarely aimed at enthusiastic amateurs (and "off-duty" professionals) playing with
programming for the sheer fun of it. 

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
Concurrency is about interaction, parallelism is a control strategy.

## Overview Sketch

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
