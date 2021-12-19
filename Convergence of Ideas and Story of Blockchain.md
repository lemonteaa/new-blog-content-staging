# Convergence of Ideas and Story of Blockchain

Despite the hype, trying to actually get into the blockchain/crypto space can be difficult for many people. In between blockchain being a new paradigm without much percedent, the large number of jargons being thrown around, and a number of tricky, highly interlocking concepts that must all be understood to truly understand blockchain, the invisible barrier of entry is rather real.

This post is a dumping log of how the author currently grok blockchain. Because of its tight interrelation, cryptocurrencies will also be covered somewhat (But see the clarification near the end).

And some caveat before hand: this post do have the pre-requisite of a good background in Computer Sciency stuff - given the wide range of ideas we will need to integrate, it is more important that you have an intuitive "sense" of how to do CS-like thing than particular knowledge in any specific fields.

So, ready? Let's go.

> tl;dr - Distributed System + Peer-to-peer + Cryptography + Economics/Game Theory = Blockchain and Cryptocurrency

(If you're a genius and instantly know what I am going to do already, please gently click the "Back" Button of your browser ;) )

# Ingredient 1: Distributed System

## Append-Only Log and some functional programming

Traditionally, a database's model is that of a mutable store of values. With RDBMS, the data model is built up from "relations"; with NoSQL, it can be key-value, documents, graph, etc.

A distinctive model has been proposed based on the abstraction known as the append-only log - see this article for details. For ordinary programmer, perhaps the fastest explanation is the log files used in application for debugging and other monitoring purpose. Often we write code such as

```
    if (balance + transactionAmount < 0) {
        log.info("Insufficient balance [accountId = " + accountId + ", balance = " + balance + ", transactionAmount = " + transactionAmount + "]");
        throw new ValidationError(accountId, balance, transactionAmount);
    }
```

And the string is output to `stdout` or, in case there is a logging framework setup, redirected to a log file in *append mode*. Server administrator, or DevOps people, would then examine the log using `tail -f server.log` to see a stream of new messages being appended to the file live.

The key point is that we cannot modify existing data once inserted - instead we can only keep adding on new data.

At first, this seems to be a step back and restrictive. So let's add a bit of functional programming magic. Remember `reduce` or `fold`? For those who don't know, you basically take some initial state, a "state-transition function" that tells you what the next state is given the current state and an action to apply, and a list/vector of actions. `reduce` then apply the state-transition function repeatedly to the actions, one at a time, in their list-specified order, to arrive at the final state.

If you also happen to have studied a bit of theoretical computer science, this should sounds familiar - what we are really doing is to take some finite state machine [1], then run/simulate it through an execution trace to get to the final state.

If we combine this two ideas, we get an alternative to the mutable data store: instead we store an audit log of all mutation actions performed. To make a "mutation", append a data describing the mutation we want as an entry to the log. To read the current value, replay the log from the beginning and apply the mutations to the value in-memory.

Now you may object: but this is crazy! It is going to be horribly inefficient!

There are some possible response to that:

...

## Replicated Database, NoSQL, and Highly Available System



## Consensus Algorithm

Distributed algorithm is hard. Very hard. It is hard because it is deceptively easy - see for example the fallacies of distributed computing.

Consensus algorithm is in a sense the culmination of Distributed algorithm. But for all the rap it got, what it does is, of course, deceptively simple: it is a protocol for many nodes on a network to agree on something. If you think you can already think of an algorithm to do that, please read the article above again and think very carefully. If you are still not convinced, try to subject your algorithm to these not very harsh tests:

- Is it resilient to nodes failing (and restarting)?
- Is it resilient to asynchronity - messages arriving/being processed out-of-order?
- Is it resilient to network partition? (Okay, this one's overly harsh ;) )

Anyway. Since we're not specializing in this area, we will content ourselves with just the result. (I planned a different article that do talk more in-depth about this - life goes on, and we need something new after the matter is apparently "settled" to keep life interesting)

The basic algorithm is called Paxos. It is also notoriously difficult to teach and understand. Because of that, a "new generation" algorithm called Raft is proposed. And we are in luck! Raft's reformulated the problem to an equivalent one that fits comfortably within the conceptual schema we're building up: instead of solving the consensus problem, it solves the "state machine replication problem".

What is even nicer is that our schema also neatly explain why the two problems are equivalent: this is mentioned in the reference I give above, quoting it directly:

> If two identical, deterministic processes begin in the same state and get the same inputs in the same order, they will produce the same output and end in the same state.

For this to work, of course we need a way for the nodes to agree on the order of the log messages (and the peer to peer part is to simply broadcast all log messages received to all other nodes).

We will cheat here and look a bit ahead to blockchain. In the field of distributed computing, while things are already nasty, we at least can assume that the nodes are cooperative, for instance because they are all owned and operated by the same entity. But in the world of blockchain, the basic assumption is that of "decentralization" and being "trustless", so we cannot just copy this algorithm.

However, it turns out there is a variant of the algorithm that is byzantime fault tolerant - that is to say, it can operate even if some node are hostile and may behave in arbitrary manner by sending carefully considered messages that are out of the rule of the protocol, in an attempt to "jam" or otherwise "mislead" the network into the wrong answer. The variant ensure that we are resistant to this, provided that at most 1/3 of the nodes are honest/good. And indeed some blockchain project do use this as their consensus mechanism. That being said, it is not a mainstream choice as the 1/3 threshold is considered too dangerous; in contrast, with "ordinary" consensus algorithm, we can tolerate up to 1/2 nodes failing. Indeed, even with 1/2, or simple majority threshold, there have been instance where this is breached in the real world - see "51% attack".


# Ingredient 2: Peer to peer

## Overlay network

Our current internet infrastructure is semi-hierarchical. While the top level is in a way peer to peer (e.g. Peering agreement in internet exchange among the ISPs), the rest of us get to enjoy a hierarchical organization. At the application level too there is an asymmetry with the client-server architecture. While there might be differences in computing power, the more important and relevant different here is that the server has a public IP in order to be reachable anyway from the public internet, while a client do not (it instead get assigned a shared, and perhaps dynamic IP, by the ISP using NAT).

...



## Gossip Protocol

## Incentives in P2P system


# Ingredient 3: Cryptography

## Hashing

## Asymmetric encryption/signature


# Problem formulation

So here is what we want:

> A decentralized, trustless, tampering-resistant database that is also public and permissionless.

That's a lot to stomach so let's break it down:

* Decentralized is stronger than just being distributed. While having multiple nodes that are potentially spread over the network is enough to qualify for the "distributed" label, "Decentralization" requires that there be no "Leader", or "Central Authority" that controls any thing. And it is not just about the engineering concern of there not being a single point of failure (which is already considered/accounted for in distributed computing) - it is more about the political problem of power and control. To be more specific, decentralization counter the popular maneuver of hostile take-over of a system by finding a point of key leverage (leader node) and then using whatever mean to convert it to being under the agent's control.
* Trustless summarize the idea that the system consists of nodes/entities that do not trust each other, coming together to still "cooperate" to achive some common goals. If this sounds self-contradictory, imagine any market transactions involving strangers, over high-stake items. In spite of the mutual distrust, agents still find a way to cooperate by trading.
* Permissionless is a jargon in the blockchain world. It capture the idea that any one should be able to join the network/system without having to ask for permission from any Central Authority (who have the power, in principle, to reject such a request on arbitrary ground).

But we haven't answered the question: why would anyone on earth want that? While everyone's motivation could be different, we will be content with one possible motivation, covered in the next section. (It is sort of a detour, but please bear with me)

# An interlude/prelude: a thought experiment on the origin of cryptocurrency

(If you are geniuely interested in cryptocurrency or even DeFi "in and of itself", then this is a prelude; if not it is a necessary interlude)

(Get ready for some mental gynmanastic. This part is non-technical, but it will challenge your ability to challenge your own deeply held, subconscious assumptions, the ability to see things in what may be a radically different angle. I would wager that much of the difficulty of grasping blockchain is a psychological one of being limited by one's own fixed "framing". As they say, a change of perspective is worth 100 IQ points.)

Let's think about what is really happening behind the scene of traditional banking/finance. In the ideal scenario, banking is something that "just work" - something that we don't need to think about at all. At the most basic level, banks provide a convinient way to store and transfer money. First you deposit cash into your account: using either ATM or human counter, you hand over the cash, and your account's balance is updated to reflect that. With modern e-banking, you just log into your account and then do most of the operations that do not requires a physical prescence. At the click of a button, you can transfer money to where you want: your friend's account, or may be even overseas. Finally, when you do need the cash, you can perform withdrawal: the steps' just the reverse of deposit.

At its core, what give this seamless experience its magic is down to two keys:

1. The ubiquity of online apps/websites means that we are used to working with it
2. The tight integration of online-offline activities to present the image of an unified, coherent service - we think of "the bank" as a single entity against which we perform actions, whose effect is immediate and universal.

If it is that good, what is it to complain of?

Turns out, some people prefer to peel back the layer to see "the truth", no matter how uncomfortable it might be. To begin with, we have implicitly equated cash on our hands with the account balance in the bank, even though it is something that we cannot touch physically. If we re-think the basic flow above with a stubborn, physical orientation, this is what actually happens:

* When you deposit your cash, you in fact handed over the cash. In return the staff pushed some buttons on the computer, which results in the modification of some records in some database.
* To be precise, this is a database sitting somewhere, in an infrastructure that is centrally owned by the bank.
* Afterwards, the cash is stashed away, and eventually moved to the bank's own storage facility. At this point, there is no marking at all to identify these bank notes as "owned by you" - for all we care, it is actually owned by the bank.
* From this point on-ward, the bank may 1) lend this cash out to other entities (fractional reserve system), or 2) deposit it into the central bank.
* Back to us. When we do e-banking, what we actually do is to login to a system through some server, that is solely controlled by the bank, then perform operations to further changes some database records (interbank transfer would be more complicated, but that's besides the point). Throughout, the success of the operations are all subject to the banks' approval, whether it is automated or manual.
* When we attempt cash withdrawl, we are implicitly asking the bank to honour and fulfill their implicit promise to us when we handed over our cash to them - that they would allow us to get back an equivalent amount of cash when we need it. The bank checks whether it have made such a promise by querying the records inside a database that is owned and operated by them, and if they deem everything alright, will push some button to change the database records again to mark that promise fulfilled, then take out cash from their stash, and hand it over to you.

Phrased in this way, suddenly the magic falls apart and the whole thing even appears scary. We voluntarily handed over our cash, all in exchange for a promise to give it back to you when you demand it? They don't even keep the cash but give it to someone else? The promise is recorded in a database that is fully owned and controlled by them? Isn't it like asking the suspect to monitor themselves, and trust the report - written by the suspect themselves - saying that they are trustworthy? And access to the database/reports itself is centrally managed by them too - there is never a guarantee that they won't cut off your access at any time, any where, without having to give any explanation.

## Trust, trust is the real issue

To most normal people, the above would read like a rant, an unjustified, paranoid fantasies of somehow being prosecuted by the banks. Normal people think there is no problem because we trusted the bank in the beginning, and the banks are so far trustworthy from its track record, therefore we should trust the banks.

But what is trust? And what is the nature of such a reasoning? It appears that this is a distinctive category of logic from the usual, naturalistic type. Instead, it is a form of social logic - one that is particular to the agents' self-interest. What I mean is this - the logic above would work exactly for those agents for whom no bad incidents have so far occured in the relationship working with the banks.

Some people, however, are not so fortunate. With repeated betrayal of trust, one is eventually forced to re-evaluate its relationship with the banks. And if one begins from a position of extreme mistrust, one may even go as far as to re-evaluate the entire philosophical basis of having any relation with the banks at all.

## How far are you willing to go?

When human potential is unleashed from what might have been a traumatic event, the results can be astonishing. In order not to re-experience the pain of being betrayed by the banks again, one may even re-invent one's very own "bank".

If we take a careful look at the basic flow above, "the banks" ultimately boils down to that database and some application logic governing how those records could be updated. And this is where the problem we posed in the last section enter the picture.

...

# Putting the ingredients together

We are now ready to reinvent blockchain by thinking through the steps.

1. First we use the append-only log as our data architecture. It should, of course, be durable.
2. Our network architecture is that of a simple, public P2P network with no identity needed to join. Conceptually, the network topology should be that of a mesh/fully-connected network to present the idea that there is no "special" node.
3. (This is the beginning of the non-trivial part) Any node can submit new log entry to the database by broadcasting. How are we going to deal with malicious entry? First thing we do is to change the semantic of receiving a message/entry: instead we are now only accepting a proposal. There can be multiple proposal for the "next" message/entry, and instead of nodes exchanging their opinion on accept/reject (which is too intricate/subtle with the distributed algorithm thing), all nodes simply note down all proposals without a definitive answer.
4. Here we introduce two ideas: chaining and Proof-of-Work. Remember that a list can be implemented as a linked list, where each entry contain a pointer/reference to the next/last entry. Also remember in some CPU course that there is an interesting form of addressing scheme known as content-addressable. Since we have the tempering-resistant requirement, an interesting idea is to use a variation of that by using the hash of the whole entry as the pointer/reference. All submitted entry should contain a field with a hash pointing to the previous entry. Then a node's store of all proposed entries will have the data structure of a tree, with the branches representing competing claim, supposedly by different agents, of what they want the next entry to be. Coupled with this idea is the Proof-of-Work problem. There can be many variations in how this problem is implemented, but in the classical example we require that an entry solves the hash-preimage problem. Of course this problem in full is computationally infeasible at all, so we loosen it to only require a partial pre-image. For reasons we will see later, we also make the "difficulty level" tunable. To be precise, the entry should have 4 fields currently: `prev-entry`, `content`, `difficulty`, `pow-solution`. `pow-solution` should be chosen such that `Hash(prev-entry || content || pow-solution)` has at least `difficulty` many leading zeros. Since our hash is secure, the only way to find even a partial preimage, is to use brute-force: enumerate all possible values of `pow-solution` and compute the hash repeatedly, until we find one that satisfy the constraint.
	1. (Block)
	2. The cumulative effect of what we did is that we slowed down the production of new blocks to prevent spamming. Also, a crucial property of using chaining with hashing (which is a form of digital signature without identity) is that it already confers some amount of tampering resistance. To see this, imagine Malory wanting to change some record in the middle of a chain. To do so, he need to essentially "rewrite history" and submit a new set of blocks containing his version of the entries, where only the records relevant to his aim are modified while the rest remain the same. However, changing the block content will completely alter the hash, so the next block pointed to by his modified block will also have the hash completely changed, even if the content of that block did not, because the hashing is over the entire block, including the headers. Applying the same logic, Malory will have to recompute all downstream blocks. Due to PoW, the only way to do this is to solve the PoW problem again for each reconstructed/modified blocks. Therefore, the deeper the target block is, the longer it will take to do it. (We will explain why Malory will need to submit the downstream blocks in the first place later)
5. And this is where the clever steps is. We again do two things at once: impose an economic incentive to produce new, valid blocks, and to add an implicit condition that the cannonical history is the longest branch in the tree.



# The Final ingredient: Game Theory


## A small note: Where did the economies go?

(Not a pun on the miserable state of real world economy, but, ya, I am sorry.)


# Emdedding cryptocurrency into a blockchain


# Conslusion

