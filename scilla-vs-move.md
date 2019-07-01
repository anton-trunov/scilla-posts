# TODO: change title: Scilla vs Move

The recent release by Facebook introduced the [Move][move-overview] programming
language for writing smart contracts for the Libra blockchain. Naturally, all
the language geeks from the Scilla language team rushed to try out this new and
shiny piece of techonology. It turned out to be a work-in-progress project, so
we weren't able to implement the helloworld of smart contracts -- the
[Crowdfunding][scilla-crowdfunding] contract. With adding new features such as
associative arrays (maps), lists, etc., Move will eventually catch up and we
will revisit this. The [Move paper][move-paper], however, discusses several very
interesting and important topics, and we are going to cover some of them in this
blog post.

## Handling scarcity of resources

Move uses a linear type system to handle scarcity of resources. Linearity
ensures that each resource must be used _exactly_ once in a procedure. This
looks a bit like the approach taken in the Rust programming language, which has
affine type system ensuring that variables are used zero or one time only.

### Linear types to handle scarcity?

Linear types used in Move can work quite well if we need to ensure ownership of
such resources as file handlers or heap memory chunks, though it's not clear at
this point how to manage linear types to ensure safety of fungible resources
like money tokens. Move's type system, as the paper admits, will not ensure, for
instance, that
> the total value of all coins in existence is preserved by a call to `deposit`,
because deep down money is represented as an unsigned integer value.

### Fungible tokens, scarcity, and formal verification

Futhermore, sometimes one wants shared ownership of resources, see e.g. an
[ERC20-like contract][fungible-token] implemented in Scilla. In the
`TransferFrom` transition we allow a third party to manage someone's tokens up
to some limit. With each transition the amount of the owner's tokens gets
decreased and moved the destination account, but at the same time the limit gets
decreased by the same amount. Thus, there is no token preservation here per se.

The approach taken to ensure money-safety of Scilla contracts is to use [SMT
solvers][smt] to _formally_ prove properties that can be expressed in linear
integer arithmetic, which covers many practically important contracts. The great
advantage of SMT solvers is their ability to prove specified properties
completely _automatically_. Along with the properties mentioned above, we can
also ensure that there will never be any integer overflow or underflow
exceptions while executing SMT-verified contracts.

In addition, if the programmer is facing a highly complicated set of conditions
and possible interactions such that even SMT solvers cannot handle those, we
will always be able to prove the safety properties manually using the Coq proof
assistant using our work-in-progress framework.

## Verifiability of contracts

### Modularity

> Move modules enforce data abstraction and localize critical operations on
> resources. The encapsulation enabled by a module combined with the protections
> enforced by the Move type system ensures that the properties established for a
> module’s types cannot be violated by code outside the module.

This is pretty much what we have in Scilla with the only difference that Scilla
only allows one "module" (smart contract) per blockchain address. Scilla
transitions are the only interface a Scilla smart contract can have. External
contracts cannot mutate the contract's state directly, only through calling
transitions, incidentally, this makes impossible the Parity wallet hack.

The set of contract's transitions are supposed to keep some properties called
_invariants_. The contract's invariants describe the set of "good" states the
contract must never leave. If we start in a good state in which all the
invariants ensuring safety hold, then any well-behaved transition upon
termination ensures that the invariants of the contract state still hold.

This does not mean a transition can never violate any invariant, but that any
voilated invariant must be restored before exiting the current transition. The
ability to violate invariants locally, i.e. inside transtions, presents some
issues with so called _re-entrancy_, which together with dynamic dispatch (see
the next subsection) enabled the infamous DAO attack.

These issues are resolved in Scilla by two design choices:
- Scilla does not allowe external transition to mutate the calling contract's
  state directly;
- Scilla's transitions cannot be interrupted in the middle, i.e. all the calls
  to external transitions via messages happen at the end of the current
  transition.

Indeed, in the middle of execution, when our watch dog invariant is off guard,
it's easy to violate contract's safety by calling an external transition which
in its turn could call one of our currently executing contract's transitions
back. If we are currently in a "bad" state where our safety invariants don't
hold, all bets are off, there is nothing to protect us except for the sheer
luck.

### Dynamic dispatch

Unlike Scilla, Move only allows static linking of procedures:

> The target of each call site can be statically determined. This makes it easy
> for verification tools to reason precisely about the effects of a procedure
> call without performing a complex call graph construction analysis.

You can see how dynamic dispatch can be done in Scilla in the
[ping][scilla-ping]-[pong][scilla-pong] sample contracts. The `SetPongAddr` /
`SetPingAddr` transitions let us dynamically link contracts with a possiblity of
upgrades at run-time. This is something not possible with static linking of
called procedures and is well-known to be a challenge, at odds with static
guarantees and safety concerns. However, since, as we already mentioned,
external transitions cannot ever change the contract's state directly and
provided we have designed our contract with safety invariants in mind, we may as
well plan for some moderate flexibility and add carefully chosen extension
points to evolve the contract's behavior over its lifetime.

### Limited mutability

> Every mutation to a Move value occurs through a reference. References are
> temporary values that must be created and destroyed within the confines of a
> single transaction script. Move’s bytecode verifier uses a “borrow checking”
> scheme similar to Rust to ensure that at most one mutable reference to a value
> exists at any point in time. In addition, the language ensures that global
> storage is always a tree instead of an arbitrary graph. This allows
> verification tools to modularize reasoning about the effects of a write
> operation.

Scilla bypasses the issue of the global storage being an arbitrary graph by not
including mutable references/pointers in the language. To our experience, this
design choice leads to a safer code that is much easier to understand than
almost any pointer-manipulating code.

TODO: what else?

## Conclusion

TODO

<!-- References -->

[move-overview]: https://developers.libra.org/docs/move-overview
[move-paper]: https://developers.libra.org/docs/move-paper
[scilla-crowdfunding]: https://scilla.readthedocs.io/en/latest/scilla-by-example.html#a-second-example-crowdfunding
[scilla-ping]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/ping.scilla
[scilla-pong]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/pong.scilla
[fungible-token]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/fungible-token.scilla
[smt]: https://en.wikipedia.org/wiki/Satisfiability_modulo_theories

