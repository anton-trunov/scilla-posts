# A Scilla vs Move case study

The recent release by Facebook introduced the [Move][move-overview] programming
language for writing smart contracts for the Libra blockchain. Naturally, all
the language geeks from the [Scilla][scilla-lang] language team rushed to try
out this new and shiny piece of techonology. Here I present to you our findings
in the form of a Scilla vs Move case study. In case you are not very familiar
with Scilla I'd suggest to leaf through [Scilla Docs][scilla-manual].


## Earmarked Coin: problem definition


Let's consider the following [example][move-writing-modules] of the earmarked
coin:

> Consider this situation: Bob is going to create an account at address `a` at
> some point in the future. Alice wants to "earmark" some funds for Bob so that
> he can pull them into his account once it is created. But she also wants to be
> able to reclaim the funds for herself if Bob never creates the account.

This is an example of a more general notion of escrow contract. Yet another
example of this family of contracts is the helloworld of smart contracts -- the
[crowdfunding][scilla-crowdfunding] contract.


## Solution in Move: the big picture


Money in Move is handled using _resources_, an abstraction that makes having
multiple references to the same object not possible. This is to make sure money
is scarce in the system and facilitate transfer of ownership over money.
Technically, Move uses a linear type system to guarantee scarcity of resources.
Linearity ensures that each resource must be used _exactly_ one time in a
procedure. This looks a bit like the approach taken in the Rust programming
language, which has affine type system ensuring that variables are used zero or
one time only.

Move's programming model is based on changing the sender's account as execution
progresses. This point of view is quite different from Scilla's -- contracts are
_not_ allowed to change user accounts in any way. Contracts are independent
entites with their own accounts.

In some sense Move's model assumes we have a _global_ bank with the global mapping
of addresses (representing the "bank"'s clientele) to assets (resources). 
Everybody can change their account by publishing new resources. Often a new
resource will be a coin with some _relaxed_ notion of ownership.
Linearity of resources ensures that users cannot loose or duplicate their assests
when transforming their assets from one kind to another.

On the other hand, Scilla's model presupposes that each contract is like a local
bank. This means contracts commonly have their own local mappings of addresses
to users' assets. The necessity of maintaining those local maps can be handled
with by using a generic contract published alongside with the specific contract
the programmer wants to deploy.

To better understand the point made above, let's go over the solution in Move,
translating it in a piecewise manner into Scilla. But before we dive into the
implementation of the earmarked coin contract in Move, let me show you how one
would _use_ the contract we are going to take apart and explain.

To use the earmarked coin contract, Alice writes a _transaction script_ to
program her interaction with the system. Suppose Alice owns some (let's say 100
for the sake of example) Libra coins, the primary currency of the Libra
blockchain. The values of `LibraCoin.T` type represent the primary currency on
the formal level. The fact that Alice owns some Libra coins is reflected in the
`balance` field of her `LibraAccount.T` resource. `LibraAccount` is sort of a
high-level wrapper for `LibraCoin` as it supports more advanced authentication,
keeps track of some statistics, etc. The `balance` field of `LibraAccount.T`
resource has type `R#LibraCoin.T`, where `R#` is a syntactic clue hinting that
we are dealing with a "resource". Bob has no means of accessing Alice's Libra
coins because `LibraCoin.T` resource is very strict about ownership. So Alice
needs to create a new resource `EarmarkedLibraCoin.T` (see below for its
definition), which lets strangers get your money, but only if you explicitly
let them. Creating a new earmarked coin _consumes_ a Libra coin of the same
worth, meaning Alice needs to split a new coin off her account and tranform it
into the earmarked coin she wants to create. So she invokes the following
transaction script:

```rust
  // split a Libra coin from Alice's account into a temporary Libra coin
  alice_libracoin = LibraAccount.withdraw_from_sender(move(amount));
  // create and publish the earmarked coin under Alice's account
  EarmarkedLibraCoin.create(move(alice_libracoin), move(bob_address));
  // alice_libracoin is not accessible anymore, it's been consumed
```

Notice that the move semantics makes it not possible to loose or duplicate money
in this case.

Now it's possible for Bob to get the earmarked coin, convert it to a Libra coin
and publish under his account. To get the earmarked coin Bob needs to know the
address it's been published under (`alice_address` below).

```rust
  // acquire ownership over the earmarked coin, this is not possible with Libra coin
  bob_earmarked_coin = EarmarkedLibraCoin.claim_for_recipient(alice_address)
  // turn the earmarked coin into a Libra coin
  // the earmarked coin has been consumed and now Alice can't get the earmarked coin back
  bob_libracoin = EarmarkedLibraCoin.unwrap(bob_earmarked_coin)
  // publish the temporary coin under Bob's account
  LibraAccoount.deposit(bob_address, move(bob_libraconi))
```


## Creating resources


The earmarked coin contract in Move starts as follows:

```rust
// A module for earmarking a coin for a specific recipient
module EarmarkedLibraCoin {
  import 0x0.LibraCoin;

  // A wrapper containing a Libra coin and the address of the recipient the coin is earmarked for.
  resource T {
    coin: R#LibraCoin.T,
    recipient: address
  }
  ...
```

Move's modules are a lot like Scilla's contracts: a module in Move is a
"long-lived piece of code published in the global state".

> Move modules enforce data abstraction and localize critical operations on
> resources. The encapsulation enabled by a module combined with the protections
> enforced by the Move type system ensures that the properties established for a
> module’s types cannot be violated by code outside the module.

The solution in Move creates a custom resource type `T` by pairing a
`LibraCoin.T` resource with the recipient's address.


To create an earmarked coin, the sender's transaction script would call the
function `create` defined within `EarmarkedLibraCoin` module:

```rust
  ...
  // Create a new earmarked coin with the given recipient.
  // Publish the coin under the transaction sender's account address.
  public create(coin: R#LibraCoin.T, recipient: address) {
    let t: R#Self.T;
    t = T {
      coin: move(coin),
      recipient: move(recipient),
    };
    move_to_sender<T>(move(t));
    return;
  }
  ...
```

In the snippet above the `coin` parameter representing the funds the sender
wants to earmark gets _moved_ into earmarked resource `t` (with `coin:
move(coin)` statement), which makes the funds associated with `coin` not
immediately available for the sender.
The line
```rust
    move_to_sender<T>(move(t));
```
publishes the newly created earmarked coin under the sender's account.

Note that this module (contract) only allows each sender to earmark some funds for
one person only, as the second call to `create` will fail at `move_to_sender`
invocation, because

> accounts can contain at most one resource value of a given type.
 
This design decision provides, according to the Move whitepaper, "a predictable
storage schema for top-level account values". There is, however, a workaround
for the use case when one wants to have multiple values of a resource type under
one account. This can be done

> by defining a custom wrapper resource (e.g.,
> ```
> resource TwoCoins { c1: 0x0.Currency.Coin, c2: 0x0.Currency.Coin }).
> ```

If multiple users wanted to earmark funds for the recipients of their choice
they would use a transaction script like the following one:
```rust
import 0x0.LibraAccount;
import 0x0.LibraCoin;
import 0x42.EarmarkedLibraCoin;
main(recipient: address, amount: u64) {
  let coin: R#LibraCoin.T;

  coin = LibraAccount.withdraw_from_sender(move(amount));
  EarmarkedLibraCoin.create(move(coin), move(recipient));
  return;
}
```

This script withdraws some `amount` from the caller's account of Libra coins and
re-publishes that amount as an earmarked coin again under the caller's account.
At first sight it might seem that we have gained nothing, because there was no
transfer of ownership, however the notion of ownership for the earmarked coin is
a relaxed one (as we will see below).

The runtime system of Move keeps track of resources for each its user, hence
multiple users can earmark funds for someone by creating new resources. In a
sense, the approach taken for the Move implementation is user-centric, as
opposed to Scilla contracts, which are contract-centric. Scilla contracts have
their own balances to aqcuire ownership of funds transfered to them. A
contract's `_balance` special mutable field stores a 128-bit unsigned integer
reflecting the total funds accumulated by the contract, hence Scilla escrow-like
contracts usually keep record of who exactly has transferred funds using an
explicit map of donors or backers.

Putting the explanation I just gave to work, here is how the corresponding
contract could look like in Scilla (the version here is simplified for the sake
of clarity of exposition, see the full version [here][scilla-earmarked-coin]):

```ocaml
(* purely functional part of the contract *)
library EarmarkedCoin

(* funds and their future owner *)
type EarmarkedCoin =
| EarmarkedCoin of Uint128 ByStr20

contract EarmarkedCoin
(* this contract does not have any parameters *)
()

field earmarked_coins : Map ByStr20 EarmarkedCoin = Emp ByStr20 EarmarkedCoin
...
```

The type `EarmarkedCoin` is analogous to `EarmarkedLibraCoin.T`, except for it's
not linear, because Scilla's type system is based on second-order polymorphic
lambda calculus. Also, notice the explicit `earmarked_coins` mutable map from
senders' addresses to paired funds and the intended recipients of those funds.
The `earmarked_coins` map is empty when the contract is initialized.

Let's look at thow the Scilla analogue of `create` procedure looks like:
```ocaml
...
transition Earmark (recip : ByStr20)
  (* have we already earmarked funds? *)
  c <- exists earmarked_coins[_sender];
  match c with
  | False =>
    accept;
    e_coin = EarmarkedCoin _amount recip;
    earmarked_coins[_sender] := e_coin;
  | True => (* do nothing *)
  end
end
...
```

The transition is named `Earmark` to indicate that there is a difference between
Scilla and Move. The transition works as follows. First, we check that the
sender haven't already earmarked funds, because each sender is only allowed to
earmark for at most one person. If the sender has not earmarked funds yet, the
contract accepts the transfer of funds by using the special `accept` builtin
statement. This increases the contract's `_balance` mutable field by `_amount`
-- the implicit parameter of each Scilla transition and decreases the sender's
balance by the same quantity. The next three statements create a new value of
datatype `EarmarkedCoin`, put it into `earmarked_coins` map to remember that
`_sender` earmarked `_amount` of tokens for `recip` recipient. Abstracting over
the exact implementation, this is what Move's runtime does implicitly.


## Claiming funds for recipient


Here is the Move implementation of `claim_for_recipient` function, returning an earmarked coin reserved for the caller:

```rust
...
  // Allow the transaction sender to claim a coin that was earmarked for her.
  public claim_for_recipient(earmarked_coin_address: address): R#Self.T {
    let t: R#Self.T;
    let t_ref: &R#Self.T;
    let sender: address;
    t = move_from<T>(move(earmarked_coin_address));
    t_ref = &t;
    sender = get_txn_sender();
    assert(*(&move(t_ref).recipient) == move(sender), 99);
    return move(t);
  }
...
```

The `move_from` function call lets the funds change ownership

```rust
    t = move_from<T>(move(earmarked_coin_address));
```

as the Move type system guarantees the original sender won't have access to that
earmarked coin anymore.

I should mention that the following line
```rust
    assert(*(&move(t_ref).recipient) == move(sender), 99);
```
ensures that only the intended recipient can get the ownership over the earmarked coin.

The corresponding transition in Scilla would look like so:
```ocaml
transition ClaimForRecipient (earmarked_coin_address : ByStr20)
  e_coin_opt <- earmarked_coins[earmarked_coin_address];
  match e_coin_opt with
  | Some (EarmarkedCoin amount recipient) =>
    (* transfer only if the funds have been earmarked for the caller *)
    authorized_to_claim = builtin eq recipient _sender;
    match authorized_to_claim with
    | True =>
      TransferFunds amount recipient;
      delete earmarked_coins[earmarked_coin_address];
    | False => (* do nothing if not authorized *)
    end
  | None =>
    (* nobody with account at `earmarked_coin_address`
       earmarked any funds, so do nothing *)
  end
end
```

Here the code first checks that there is a coin earmarked by a sender with
`earmarked_coin_address` account address. Next, the code checks the coin has
been earmarked for the transition's caller by checking the implicit `_sender`
parameter. The `TransferFunds` procedure (not shown for brevity) transfers funds
from the contract's balance to the recipient's. The next line deletes the record
of the just transfered earmarked coin from the global `earmarked_coins` map.
Because the contract's balance accumulates funds from many senders, forgetting
to delete that record might result in a malicious user performing multiple
claims of the same coin until the user drains the contract's account preventing
the other recipients from rightfully claiming their funds.

It's easy for the programmer to forget to remove the record, because the Scilla
type system does not track ownership. This is where Move shows its strength
because this exact mistake is not possible by virtue of its linear type system.
However, Move's type system, as the whitepaper admits, will not ensure, for
instance, that

> the total value of all coins in existence is preserved ...

because deep down money is represented as an unsigned integer value.

An approach being explored by the Scilla team to ensure money-safety of Scilla
contracts is the usage of [SMT solvers][smt] to _formally_ prove properties that
can be expressed in linear integer arithmetic, which covers many practically
important contracts. The great advantage of SMT solvers is their ability to
prove specified properties completely _automatically_, without user
intervention. Along with the properties mentioned above, we should be also able
to ensure that there will never be any integer overflow or underflow exceptions
while executing SMT-verified contracts.

In addition, if the programmer is facing a highly complicated set of conditions
and possible interactions such that even SMT solvers cannot handle those, we
will always be able to prove the safety properties manually using the Coq proof
assistant using our work-in-progress framework.


## Reclaiming funds for creator


The Move implementation of `claim_for_creator` is pretty straightforward:

```rust
...
  // Allow the creator of the earmarked coin to reclaim it.
  public claim_for_creator(): R#Self.T {
    let t: R#Self.T;
    let coin: R#LibraCoin.T;
    let recipient: address;
    let sender: address;

    sender = get_txn_sender();
    t = move_from<T>(move(sender));
    return move(t);
  }
...
```

It unpublishes the earmarked coin by calling the same `move_from<T>` as in
`claim_for_recipient`. After calling the function, the original sender will hold
the only possible reference to the resource, efficiently locking out the
intended recipient from aqcuiring the ownership of it.

The analogous transition in Scilla is also simpler than the `ClaimForRecipient` transition:

```ocaml
transition ClaimForCreator ()
  e_coin_opt <- earmarked_coins[_sender];
  match e_coin_opt with
  | Some (EarmarkedCoin amount _) =>
    (* get back earmarked money *)
    TransferFunds amount _sender;
    delete earmarked_coins[_sender];
  | None =>
    (* Sender has not earmarked, do nothing *)
  end
end
```

The same line or reasoning as above applies here too.


## Transforming earmarked coins to Libra coins

Note: this section only applies to the Move programmign language.

Transaction scripts claiming an earmarked coin would have to call the `unwrap`
function:

```
...
  // Extract the Libra coin from its wrapper and return it to the caller.
  public unwrap(t: R#Self.T): R#LibraCoin.T {
    let coin: R#LibraCoin.T;
    let recipient: address;
    T { coin, recipient } = move(t);
    return move(coin);
  }

} // end of module EarmarkedLibraCoin
```

to strip the address component off the earmarked coin and turn it into a
spendable Libra coin.


## Conclusion


In this blog post I tried to show the difference between the programming models
of Move and Scilla. Each of the models has different sets of trade-offs and
represent user-centric vs bank-centric points of view for Move and Scilla
respectively.


## Acknowledgements 

Many thanks to Ilya Sergey, Amrit Kumar and Vaivaswatha Nagaraj for their numerous suggestions on this blog post.

<!-- References -->

[move-overview]: https://developers.libra.org/docs/move-overview
[move-writing-modules]: https://developers.libra.org/docs/move-overview#writing-modules
[move-paper]: https://developers.libra.org/docs/move-paper
[scilla-lang]: https://scilla-lang.org
[scilla-manual]: https://scilla.readthedocs.io/en/latest/index.html
[scilla-crowdfunding]: https://scilla.readthedocs.io/en/latest/scilla-by-example.html#a-second-example-crowdfunding
[scilla-earmarked-coin]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/earmarked-coin.scilla
[scilla-ping]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/ping.scilla
[scilla-pong]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/pong.scilla
[fungible-token]: https://github.com/Zilliqa/scilla/blob/master/tests/contracts/fungible-token.scilla
[smt]: https://en.wikipedia.org/wiki/Satisfiability_modulo_theories

<!-- more links  -->
<!-- https://www.blockchainandy.com/post/want-to-build-your-latest-dapp-on-one-of-the-fastest-and-lowest-fee-platforms -->
<!-- https://levelup.gitconnected.com/getting-started-with-the-facebook-libra-programming-language-a1d21aa837e0 -->
<!-- https://hackernoon.com/move-programming-language-the-highlight-of-libra-122a910d6e0f -->

<!-- https://github.com/libra/libra/blob/4e27604264bd0a5d6c64427f738cbc84d9258a61/language/stdlib/modules/libra_account.mvir#L9 -->
