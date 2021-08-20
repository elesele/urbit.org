+++
title = "Layer 2 Overview"
weight = 5
template = "doc.html"
+++

This document provides technical detail on Azimuth's "Layer 2" scaling solution
for Azimuth, known more formally as "naive rollups". We focus here primarily on the
"Hoon smart contract" located at `/lib/naive.hoon` in your ship's pier, as well
as other proximal topis.

This is intended for developers that desire a deeper understanding of how this
protocol works, how secure it is, and how to extract data from layer 2 to obtain
a complete picture of the Arvo network.

This is not intended for everyday users who only wish to know how
to either transfer their ship to layer 2 or perform layer 2 actions. For that,
see ((bridge documentation yet to be written)). For a casual overview of the
rationale and functionality of layer 2, please see this [blog
post](/blog/rollups).

This page is also not where to find instruction on how to run your own "aggregator"/"roller".
Documentation for this process is forthcoming. However, this page does contain
essential background information for anybody in this category.

## Summary

Naive rollups were developed in response to rising gas costs for performing
Azimuth actions. They also serve the dual purpose of making onboarding easier,
as it is now possible to acquire a planet and get on Urbit without any knowledge
of Ethereum or cryptocurrency.

In this section we give a high-level summary of how naive rollups function and
how they affect the end user. Later sections elaborate on this summary.

### Layer 1

We briefly review how "Layer 1", i.e. the [Azimuth](/docs/glossary/azimuth) smart
contract suite, functions. An update to the Azimuth PKI data stored on your urbit
occurs with four steps:

 1. A transaction is posted to the Ethereum blockchain.
 2. The [Ethereum Virtual Machine](https://ethereum.org/en/developers/docs/evm/)
    calculates the resulting state transition and checks its validity, then
    updates the state if it is a valid transition.
 3. Your urbit downloads the new state from an Ethereum node.
 4. Your urbit makes the final decision on whether the new state is valid.

By default, step four always succeeds. It has always been possible in theory for
your urbit to dispute what it read on Ethereum, but there has never been any
reason to do so.

### Layer 2

Layer 1 still functions identically today as it did before naive rollups. Naive
rollups work via the following process.

 1. A batch of one or more transactions is posted to the Ethereum blockchain by
    an urbit node called a roller or aggregator.
 2. Your urbit downloads the transactions from an Ethereum node.
 3. Your urbit computes the resulting state transitions from the transactions
    and checks them for validity.
 4. Your urbit updates its locally stored Azimuth state using state transitions
    from the batch that have been deemed valid.
    
 In comparison with Layer 1, the EVM no longer checks the validity or computes
 the state transitions for any given transaction. It is now being used solely as
 a database of submitted transactions, and the business logic of computing what
 these transactions mean has been offloaded to your urbit. Thus we think of
 `naive.hoon` as being the first "Hoon smart contract". You could also consider
steps 3 and 4 of the Layer 2 process as being a fattening of the trivial step 4
 of the layer 1 process.

Here we briefly elaborate on the layer 2 steps, but see below for more technical
detail.

A roller is any urbit node - even a moon, comet, or fakezod will do - to which
batches of transactions are submitted. You could use your own ship as a roller
if you wanted. The roller collects batches of transactions from whichever
parties they choose and submits them to the Ethereum blockchain all at once.
We expect this to either happen on a regular interval, or once some minimum
threshold of transactions is reached, but the decision on when to submit is
ultimately up to the roller.

Computing the resulting state transitions obtained from downloaded Ethereum
transactions is done using `/lib/naive.hoon`, which is a gate that accepts both layer
1 transaction logs and layer 2 batches, and then computes the resulting state
transitions and updates the ship's internal Azimuth data held in
the Gall agent `/app/azimuth.hoon` accordingly.

### Savings in gas costs

There are several dimensions by which naive rollups saves on gas over layer 1.
They are:

 1. Gas is not spent on instructing the EVM to compute state transitions and
    confirm validity.
 2. Layer 2 Azimuth state is not stored on Ethereum, so the only data storage
    gas costs is for the transactions themselves.
 3. Layer 2 transactions are written in a highly compressed form. E.g., instead
    of calling the spawn action by name, it is simply referred to as action `%1`.
 4. By collecting multiple layer 2 transactions and submitting them as a single
    "batch transaction", additional gas savings are achieved by not needing to
    duplicate information such as which smart contract the transactions are
    intended for.
    
Put together, these create a reduction in gas costs of at least 65x. This is
true even for batches of L2 transactions that contain only a single transaction.
Thus there is reason to use one's own ship as an aggregator.

### One way trip

Moving to layer 2 is a one-way trip for now. This means that once a ship moves
to layer 2, it cannot be moved back to layer 1. We believe it to be technically
possible to engineer a return trip, and expect that someday this will be the
case, but there are no plans to implement this in the near future.

## Interacting with L2

Layer 2 affects all Azimuth points on the network, even if they are on layer 1. Here we
outline what each sort of ship may perform on layer 2. How sponsorship works on layer 2 is
more complex, and has its own [section](#sponsorship).

There are a total of eleven layer 2 actions, each corresponding to a familiar
layer 1 action: `%transfer-point`, `%spawn`, `%configure-keys`, `%escape`,
`%cancel-escape`, `%adopt`, `%reject`, `%detach`, `%set-management-proxy`,
`%set-spawn-proxy`, and `%set-transfer-proxy`.

Once a ship moves to layer 2, the owner will still utilize the same private keys
they used before the transfer to perform Azimuth actions. This includes the ownership
key as well as proxies. Stars and galaxies may move their spawn proxy to layer 2
while otherwise remaining on layer 1, but it is not possible to transfer only
the management proxy to layer 2; it may only happen as a side-effect of
transferring ownership to layer 2.

### Moving a pre-existing ship to L2

In order to move your ship from layer 1 to layer 2, you use
[Bridge](/docs/glossary/bridge) to transfer ownership of your ship to the
address `0x1111111111111111111111111111111111111111` TODO: Presumably this is
surfaced in the UI in Bridge, and one doesn't need to know the address. Probably
this should just be a link to the Operator's Manual.

### Dominion

Layer 2 Azimuth data for a given ship includes which layer that ship is on. We
call this the ship's _dominion_. There are three dominions: `%l1`, `l2`, and
`%spawn`. Planets may exist in dominion `%l1` or `%l2`, stars may exist in any
of the three dominions, and galaxies may exist in dominion `%l1` or `%spawn`. We
detail what this means in each case in the following.

### Planets

*Permitted dominions:* `%l1`, `%l2`.

#### `%l1` planets

*Permitted layer 2 actions:* owner: `%escape`, `%cancel-escape`. management
proxy: `%escape`, `%cancel-escape`. transfer proxy: none.

A planet in dominion `%l1` is said to exist on layer 1, which is the default
state for all planets prior to the introduction of naive rollups. In addition to
the ordinary layer 1 Azimuth actions a planet can perform, they may also choose to
`%escape` or `%cancel-escape` on layer 2 using either their ownership key or
[management proxy](/docs/glossary/proxies). See the [sponsorship](#sponsorship)
section below for more information on layer 1 ships performing layer 2
sponsorship actions.

Layer 1 planets may also move to dominion `%l2` by depositing their ownership to
the layer 2 deposit address.

#### `%l2` planets

*Permitted layer 2 actions:* owner: `%transfer-point`, `%configure-keys`,
`%escape`, `%cancel-escape`, `%set-management-proxy`, `%set-transfer-proxy`.
management proxy: `%configure-keys`, `%escape`, `%cancel-escape`,
`%set-management-proxy`. transfer proxy: `%transfer-point`, `%set-transfer-proxy`.

A planet in dominion `%l2` is said to exist on layer 2. A planet may be on layer
2 either by previously being a layer 1 planet deposited to the layer 2 address,
or by being spawned by a star in dominion `%spawn` or `%l2`, in which case it
will always be on layer 2.

A layer 2 planet is no longer capable of performing any layer 1 actions, and
cannot move to layer 1.

### Stars

*Permitted dominions:* `%l1`, `%spawn`, `%l2`.

#### `%l1` stars

*Permitted layer 2 actions:* owner: `%escape`, `%cancel-escape`, `%adopt`,
`%reject`, `%detach`. management proxy: `%escape`, `%cancel-escape`, `%adopt`,
`%reject`, `%detach`. spawn proxy: none. transfer proxy: none.

A star in dominion `%l1` is said to exist on layer 1, which is the default state
for all stars prior to the introduction of naive rollups. In addition to the
ordinary Azimuth actions a star can perform, they may also perform any
sponsorship-related actions on layer 2.

A `%l1` dominion star may move to dominion `%spawn` by depositing its spawn proxy to the
layer 2 deposit address, or may move to dominion `%l2` by depositing its ownership
to the layer 2 deposit address. Both actions are irreversible.

#### `%spawn` stars

*Permitted layer 2 actions:* owner: `%escape`, `%cancel-escape`, `%adopt`,
`%reject`, `%detach`, `%spawn`, `%set-spawn-proxy`. management proxy: `%escape`,
`%cancel-escape`, `%adopt`, `%reject`, `%detach`. spawn proxy: `%spawn`,
`%set-spawn-proxy`. transfer proxy: none.

A star in dominion `%spawn` is said to exist on layer 1.

A star in dominion `%spawn` may spawn planets directly on layer 2, but will no
longer be able to spawn layer 1 planets and will no longer be able to set its
spawn proxy on layer 1. All other layer 1 Azimuth actions may still be performed
by the star.

A star moving from `%l1` to `%spawn` has no effect on sponsorship status of any
of its sponsored planets. Moving to `%spawn` from `%l1` is currently
irreversible - the only further change to dominion permitted is moving to `%l2`.

#### `%l2` stars

*Permitted layer 2 actions:* owner: `%transfer-point`, `%spawn`, `%configure-keys`, `%escape`,
`%cancel-escape`, `%adopt`, `%reject`, `%detach`, `%set-management-proxy`,
`%set-spawn-proxy`,`%set-transfer-proxy`. management proxy: `%escape`,
`%cancel-escape`, `%adopt`, `%reject`, `%detach`, `%configure-keys`,
`%set-management-proxy`. spawn proxy: `%spawn`, `%set-spawn-proxy`. transfer
proxy: `%transfer-point`, `%set-transfer-proxy`.

A star in dominion `%l2` is said to exist on layer 2. A star may exist on layer
2 by being deposited to the layer 2 deposit address from layer 1, or by being
spawned by a `%spawn` dominion galaxy.

A star in dominion `%l2` cannot perform any layer 1 actions.

### Galaxies

*Permitted dominions:* `%l1`, `%spawn`.

#### `%l1` galaxies

*Permitted layer 2 actions:* owner: `%adopt`, `%reject`, `%detach`. management
proxy: `%adopt`, `%reject`, `%detach`. spawn proxy: none. transfer proxy: none.
voting proxy: none.

A galaxy in dominion `%l1` is said to exist on layer 1, which is the default state
for all galaxies prior to the introduction of naive rollups. In addition to the
ordinary Azimuth actions a galaxy can perform, they may also perform any
sponsorship-related actions on layer 2. `%l1` galaxies can perform all the usual
layer 1 Azimuth actions, and may also perform layer 2 sponsorship actions.

A `%l1` dominion galaxy may move to dominion `%spawn` by depositing its spawn proxy to the
layer 2 deposit address. This action is irreversible. Note, however that the
majority of galaxies already have all of their stars spawned in the [Linear
Star Release
Contract](https://etherscan.io/address/0x86cd9cd0992f04231751e3761de45cecea5d1801).
Layer 2 has no interactions with this contract - all stars released in this
manner are `%l1` dominion stars. Moving to the `%spawn` dominion has no effect
on sponsorship status.

#### `%spawn` galaxies

*Permitted layer 2 actions:* owner: `%adopt`, `%reject`, `%detach`, `%spawn`,
`%set-spawn-proxy`. management proxy: `%adopt`, `%reject`, `%detach`. spawn
proxy: `%spawn`, `%set-spawn-proxy`. transfer proxy: none. voting proxy: none.

Galaxies may either remain on layer 1, or, similar to stars, they may
deposit their spawn proxy to layer 2. They cannot move their ownership,
management proxy, or voting proxy to layer 2. However, as with stars,
sponsorship actions may be performed on layer 2 using the ownership or
management proxies regardless of the dominion status of the galaxy.

### Sponsorship

Due to the possibility of sponsors and sponsees existing on different layers,
the precise logic of how sponsorship works is complex. However, under common
circumstances it is simple.

If either the sponsor or sponsee are on layer 2, then sponsorship actions must
occur on layer 2. The only exception to this is detaching. A sponsor on layer 1
may perform a layer 1 detach action on a layer 2 sponsee, and this will result
in the sponsee having no sponsor on layer 1, and layer 2 as well if they were
the sponsor on layer 2. This is necessary for logical reasons, but it also
guarantees that there is no hard requirement to ever utilize layer 2. Without
this exception, sponsors with sponsees that move to layer 2 would be forced to
detach them as a layer 2 action if they wanted to cease sponsorship.

If both sponsor and sponsee are on layer 1 then sponsorship actions may occur on
either layer. As long as all sponsorship actions betweeen the two parties occur
on a single layer, behavior will be as expected.

In most cases this is sufficient to understand how sponsorship works. However
there are a number of edge cases that make this more complicated that developers
may need to concern themselves with in scenarios where layer 1 sponsor and
sponsees are mixing layer 1 and layer 2 actions. In the [Sponsorship state
transitions](#sponsorship-state-transitions) section below, we give a table that
shows how the sponsor and escape status of a ship changes according to which
actions are taken.

## Azimuth state

The introduction of layer 2 presents additional complication in understanding
Azimuth state. In order to be precise we define the following terminology:

 - _Layer 1 Azimuth state_ refers to the state of Azimuth as reflected on the
   Ethereum blockchain. This excludes all layer 2 transactions. Depositing to
   layer 2 is considered a layer 1 action, so the layer 1 Azimuth state is aware
   of which ships are on layer 2, but is blind to everything that happens to
   them afterward.
 - _Layer 2 Azimuth state_ refers to the state of Azimuth as stored in
   `/app/azimuth.hoon` on your ship. The state here takes into account
   transactions that occur on both layers. No distinction between the
   layers is made in the state here - e.g. a ship only has one sponsor in Layer 2
   Azimuth state, not a layer 1 sponsor and a layer 2 sponsor. This is the state
   actually in use by Urbit. Layer 1 Azimuth state is now only an input for
   generating Layer 2 Azimuth state, so any time a ship needs to check e.g. the
   public key of a ship (regardless of which layer it is on), it will check the
   Layer 2 Azimuth state, not the Layer 1 Azimuth state.
 - _Layer-2-Only Azimuth state_ refers to the state of Azimuth as reflected
   solely by layer 2 transactions. This state is not explicitly stored anywhere,
   but is computed as part of the process to create the Layer 2 Azimuth state.
   We do not make any further references to this state, but it is important to
   keep in mind conceptually.
   
Layer 1 Azimuth state is computed by the Ethereum Virtual Machine. Layer 2
Azimuth state is computed by taking in the Layer 1 Azimuth state and modifying
it according to layer 2 transactions using `/lib/naive.hoon`. When we are being
precise about which state we are referring to we will utilize capitalization as above.

Layer 2 Azimuth state is held by the Azimuth Gall app located at
`/lib/azimuth.hoon`. Layer 1 and layer 2 state are not held separately - your
ship holds only one canonical Azimuth state, generated by parsing both layer 1
and layer 2 Ethereum transactions using `/lib/naive.hoon`. It is important to
keep in mind that Layer 1 Azimuth state is entirely unaware of Layer 2 Azimuth
state. Thus, for instance, the Azimuth PKI on Ethereum (Layer 1 Azimuth state)
may claim that the sponsor of `~sampel-palnet` is `~marzod`, while the Azimuth
state held on your ship (Layer 2 Azimuth state) claims that the sponsor of
`~sampel-palnet` is `~dopzod`. Under this circumstance, this would mean that the
sponsor of `~sampel-palnet` was `~marzod` before `~sampel-palnet` was deposited
to layer 2, and thus the Azimuth PKI on Ethereum will forever reflect this.

### Sponsorship state transitions

When either a sponsor or sponsee is on layer 2, then all sponsorship actions
occur on layer 2 and layer 1 Azimuth state is ignored. The exception to this, as
noted [above](#sponsorship), is when a layer 1 sponsor performs a layer 1 detach
action on a layer 2 sponsee. Furthermore, any time a ship moves from layer 1 to
layer 2, its sponsorship status is automatically maintained in layer 2.

The only potentially complicated scenario is when both sponsee and (potential)
sponsor exist on layer 1. Then because layer 1 actions can modify layer 2 state,
careful consideration is required for interactions that mix the two. If you
and your sponsor/sponsee are not mixing layer 1 and layer 2 sponsorship actions
between yourselves, then you have nothing to worry about and may safely ignore this
section.

But, for instance, if both `~sampel-palnet` and `~dopzod` are on layer 1, it is
technically possible for `~sampel-palnet` to escape to `~dopzod` on layer 1, and
then `~dopzod` can accept the escape on layer 2. This will result in `~dopzod`
appearing as the sponsor in the Layer 2 Azimuth state (and thus be
`~sampel-palnet`'s "true" sponsor), and the sponsor in the Layer 1 Azimuth state
will remain unchanged. While it is difficult to imagine a good reason to do
this, developers working with layer 2 need to keep in mind these edge cases and
ought to read on.

In the following table, columns `E_1` and `S_1` represent the escape status and
sponsor of a given ship as reflected by the Layer 1 Azimuth state. Columns `E_2`
and `S_2` represent the escape status and sponsor of a given ship as reflected
in the Layer 2 Azimuth State. The "true" escape status and sponsor of a ship is
always what is listed in the Layer 2 Azimuth state. In other words, at the end
of the day, `S_2` is always the sponsor that matters, but layer 1 actions can
affect the values of `E_2` and `S_2`.

A tar `*` entry represents any value and if an event shows a transition
from `*` to `*` that means that value is not altered by the transition. The
transitions marked with `!!` are prohibited by the layer 1 Azimuth smart
contract and thus never occur. `A1` and `A2` represent two distinct ships.
```
Event        | E_1 | E_2 | S_1 | S_2 | -> | E_1 | E_2 | S_1 | S_2
L1-escape A1 | *   | *   | *   | *   | -> | A1  | A1  | *   | *
L1-cancel A1 | ~   | *   | *   | *   | -> !! :: no cancel if not escaping
L1-cancel A1 | A1  | *   | *   | *   | -> | ~   | ~   | *   | *
L1-adopt  A1 | A1  | *   | *   | *   | -> | ~   | ~   | A1  | A2
L1-adopt  A1 | ~   | *   | *   | *   | -> !! :: no adopt if not escaping
L1-adopt  A1 | A2  | *   | *   | *   | -> !! :: no adopt if not escaping
L1-detach A1 | *   | *   | A1  | A1  | -> | *   | *   | ~   | ~
L1-detach A1 | *   | *   | A1  | A2  | -> | *   | *   | ~   | A2
L1-detach A1 | *   | *   | A1  | ~   | -> | *   | *   | ~   | ~
L2-escape A1 | *   | *   | *   | *   | -> | *   | A1  | *   | *
L2-cancel A1 | *   | *   | *   | *   | -> | *   | ~   | *   | *
L2-adopt  A1 | *   | A1  | *   | *   | -> | *   | ~   | *   | A1
L2-adopt  A1 | *   | A2  | *   | *   | -> | *   | A2  | *   | *
L2-adopt  A1 | *   | ~   | *   | *   | -> | *   | ~   | *   | *
L2-reject A1 | *   | A1  | *   | *   | -> | *   | ~   | *   | *
L2-reject A1 | *   | A2  | *   | *   | -> | *   | A2  | *   | *
L2-reject A1 | *   | ~   | *   | *   | -> | *   | ~   | *   | *
L2-detach A1 | *   | *   | *   | A1  | -> | *   | *   | *   | ~
L2-detach A1 | *   | *   | *   | A2  | -> | *   | *   | *   | A2
L2-detach A1 | *   | *   | *   | ~   | -> | *   | *   | *   | ~
```

## Processing L1 transactions

## Processing L2 batches

## Aggregators

An "aggregator" or "roller" is any Urbit node that collects signed layer 2
transactions (typically via `/app/azimuth-rpc.hoon`), combines them into a
"batch", and then submits the batch as an Ethereum transaction. Any urbit can be
a roller, including moons, comets, and even fakezods. You can also use your own
ship as a roller.

Tlon has set up our own roller that is free to use by the community. Using
Bridge, a ship may submit X transactions to Tlon's roller per Y period free of
charge. Tlon's roller submits on a regular schedule: a submission occurs when a
total of A layer 2 transactions have been submitted to it since the the previous
submitted Ethereum transaction, or every Z time, whichever occurs first. (this
should probably go in the operations manual)

There are no security risks in utilizing an aggregator. The transactions you
submit to it are signed with your private key, and so if an aggregator alters
them the signature will no longer match and `naive.hoon` will reject it as an
invalid transaction. The worst an aggregator can do is not submit your transaction.

## Multi-keyfiles

Maybe this section goes in operating manual?

As part of the layer 2 upgrade, Tlon has expanded the role of
[keyfiles](/docs/glossary/keyfile). One of our goals with layer 2 was to reduce
the amount of friction experienced when getting onto Urbit. The enormous
reduction in fees has made a new boot method which allows instantaneous sale of
layer 2 planets or stars to be cost effective.

The ideal situation would be for the end user to be able to buy or receive a
planet and immediately boot it without having to wait for an aggregator to
submit a transaction that spawns the planet. In order to bring about this
circumstance, "multi-keyfiles" have been introduced.

Multi-keyfiles are keyfiles used to boot an urbit for first time that contains
more than one set of keys, the purpose of which is to initially utilize one set
of keys on a temporary basis, and then the other set of keys soon after. This
works as follows. A star owner prespawns a number of layer 2 planets ready to be
sold at any time for which they posess the initial key. When a user acquires a
planet from a star owner, the star owner immediately hands the initial keys to
boot the planet to the buyer. The buyer then queues a transaction at a roller to
rotate the keys to a new value only known by the buyer (this step is performed
automatically by Bridge). The buyer then downloads a keyfile from Bridge
containing both keys and uses that to boot their planet for the first time.
After the transaction setting the new keys is submitted by the roller to
Ethereum, the purchased planet will automatically switch to them once it reads
the corresponding transaction from Ethereum.

This process introduces a short period of time in which both the buyer and
seller are in possession of the keys of a planet. A malicious seller could
theoretically sell a planet to more than one party in this time period. However,
they would quickly be found out as two identical ships on the network
immediately creates problems, and the seller's reputation would be tarnished.
Due to the expense or effort needed to acquire a star, this seems an unlikely
scenario as the reward is much less than the cost. Nonetheless, buyers should
always make an effort to purchase from a reputable star, as is the case with all
transactions in life.

Multi-keyfiles were possible before layer 2, but as the cost of configuring keys
was comparable to the cost of buying a planet, they were not practical.

## Security measures

In the process of designing naive rollups, we felt it to be of the utmost
importance that there not be any loss in the security of a layer 2 ship over a
layer 1 ship. In this section we outline several relevant facets of Urbit as
well as particular measures that were taken to ensure that naive rollups were
free of bugs and exploits. We think of `naive.hoon` as being the first "Hoon
smart contract", and thus its functionality needs to be as rock-solid and
guaranteed as the Azimuth Ethereum smart contracts.

### Arvo is deterministic

Crucial to the functionality of Ethereum smart contracts is they work the same
way every time since the Ethereum Virtual Machine is determinsistic. Similarly,
as the state of Arvo is evolved via [a single pure function](/docs/arvo/overview#an-operating-function), Urbit is
deterministic as well. This property makes it well-suited for cases where side
effects are unacceptable such as smart contracts, and thus `naive.hoon` is
worthy of the name "Hoon smart contract".

### Restricted standard library

A standard security practice is to reduce the surface area of attack to be as
minimal as possible. One common source of exploits among all programming
languages are issues with the standard library of functions, and this often
leads to there being multiple implementations of standard library functions for
a given programming language. For instance, `glibc` is the most widely used
standard library for the C programming language, but sheer size gives a large
surface area in which to find exploits. Thus, other standard libraries have been
written such as `musl` that are much smaller, and some argue to be more secure
at least partially due to fewer lines of code. Hoon is not yet popular to have
multiple standard library implementations, but `naive.hoon` shucks the usual
standard library and so its subjects contains only the exact standard library
functions needed for it to function (`/lib/std.hoon`).

### Unit tests

`naive.hoon` is among the most well-tested software in Urbit. The test suite,
which may be run with `-test %/lib/tests/naive ~` from dojo, is larger than any
other test suite both in number of lines of code and number of tests. We believe
branch coverage to be at or very close to 100%

### Nonces

In the Layer 2 Azimuth state, each proxy belonging to a given ship (including
the ownership "proxy") has a non-negative integer associated to it called a
"nonce". Each transaction submitted by a given proxy also have a nonce value. If
the current nonce of a proxy in the Layer 2 Azimuth state is `n`, then only a
transaction from that proxy with a nonce of `n+1` will be considered valid.
Otherwise the transaction is discarded. A valid transaction, by which we
mean one in which the nonce and signature are correct, will increment the nonce
of the proxy in the Layer 2 Azimuth state by one once processed by `naive.hoon`.
Note that "valid transactions" also include ones where the action will fail,
such as a planet attempting to `%spawn` another planet. For the purposes of
incrementing the nonce, only the nonce and signature matter.

The use of nonces prevents "replay attacks", which is when a malicious party collects
valid transactions and attempts to resubmit them in order to perform the action
again. Since a valid transaction increments the nonce associated to the proxy
that submitted it, it is only ever possible for a given transaction to alter the
Layer 2 Azimuth state once.

The use of nonces also eliminates potential issues caused by submitting a
transaction to more than one roller. This might happen if the first roller
submitted to is taking too long for your liking, and you want to try again with
another. If both rollers end up submitting the transaction, only the first one
will succeed, as the second one will be ignored by `naive.hoon` for having the
wrong nonce.
