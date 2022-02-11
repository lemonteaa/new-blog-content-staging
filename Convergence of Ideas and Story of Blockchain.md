# Convergence of Ideas and Story of Blockchain

Despite the hype, trying to actually get into the blockchain/crypto space can be difficult for many people. In between blockchain being a new paradigm without much percedent, the large number of jargons being thrown around, and a number of tricky, highly interlocking concepts that must all be understood to truly understand blockchain, the invisible barrier of entry is rather real.

This post is a dumping log of how the author currently grok blockchain. Because of its tight interrelation, cryptocurrencies will also be covered somewhat (But see the clarification near the end).

And some caveat before hand: this post do have the pre-requisite of a good background in Computer Sciency stuff - given the wide range of ideas we will need to integrate, it is more important that you have an intuitive "sense" of how to do CS-like thing than particular knowledge in any specific fields.

So, ready? Let's go.

> tl;dr - Distributed System + Peer-to-peer + Cryptography + Economics/Game Theory = Blockchain and Cryptocurrency

(If you're a genius and instantly know what I am going to do already, please gently click the "Back" Button of your browser ;) )

Warning: This post is really long. If you want to get something fast, skip straight to "Problem formulation", then skip the interlude onto "Putting the ingredients together", and read onward. Refer back to previous section for the background knowledge if you get stuck.

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

* In terms of data storage, well it is cheap nowadays. More importantly many mission critical systems do have a requirement of having audit trails anyway, so all those data are in fact required and not going to just waste space.
* In terms of computational power, replaying from the very beginning are of course going to be expensive - and that's why there is usually an optimization applied where we have periodic snapshots of the state and start from there. (Compare to transaction log of Database system and the journal log of a file system)

## Replicated Database, NoSQL, and Highly Available System

TODO

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

## NAT Traversal and Overlay network

Our current internet infrastructure is semi-hierarchical. While the top level is in a way peer to peer (e.g. Peering agreement in internet exchange among the ISPs), the rest of us get to enjoy a hierarchical organization. At the application level too there is an asymmetry with the client-server architecture. While there might be differences in computing power, the more important and relevant difference here is that the server has a public IP in order to be reachable anyway from the public internet, while a client do not (it instead get assigned a shared, and perhaps dynamic IP, by the ISP using NAT).

This present a problem in trying to create a peer to peer network if we want anyone with internet access to be able to participate - if they don't have a public IP, they are not reachable from outside by third party. Traditionally this is solved by using a tunnel - the client connect to a specialized server who do have a public IP, and the server routes request to the client as needed. However, our goal here is decentralization, and a centralized server is the last thing we want.

A peer to peer solution is a family of techniques known as *NAT traversal*. As an example of this is done in real application, see IPFS's libp2p documentation on [NAT](https://docs.libp2p.io/concepts/nat/) and [Circuit Relay](https://docs.libp2p.io/concepts/circuit-relay/).

Once peer to peer connectivity can be established, we may then create an overlay network by having our own network protocol layered on top of this base connectivity (or TCP/IP). For example, we may reimplement routing, but this time in an egalitarian, peer to peer fashion: all peer nodes may broadcast the list of peers it have direct connectivity to, all peer node may receive this information from other peers, and all peers may participate in routing by helping to relay a message closer to its target destination. Note that from the point of view of the underlying network, the link between peers is a logical link as it may have to travel through various routers not in the peer to peer network.

At the end of the day, we are able to create our own network, mostly free from the constraint of the public Internet.

## Gossip Protocol

If a node in the network want to broadcast some public information, one possible method is through the Gossip Protocol. In short, the source randomly (or deterministically) select some subset of the peers it knows to send to. Each peer that received the message then do the same to the peers they know, and so on.

## Incentives in P2P system

TODO

# Ingredient 3: Cryptography

Just a quick review.

## (Cryptographically secure) Hashing

A hash is a way to mash up the data of some content and condense it down to a small, constant size. One application is for error checking when downloading file. However, while ordinary hash may protect against accidental error (e.g. bit flip in network), it cannot protect against malicious, deliberate change in the content. A Cryptographically secure hash uses different algorithm to make it difficult to find a *hash collision* - that is any other content that hashes to the same as the target. In more detail we can distinguish between pre-image resistance, second pre-image resistance, and collision resistance.

Applications of such a hash relevant in this article include digital signature and Proof of Work (PoW).

## Asymmetric encryption/signature

In asymmetric encryption, the encryption and decryption keys are different. An example of this is RSA. There is an algorithm to generate a public/private key pairs. The public key is published publicly and can be considered to be the key owner's *identity*. Someone who wish to send a secret message can use the recipient's public key to encrypt it, then the recipient decrypt using his own private key, that only he knows.

In the case of RSA, the same algorithm can be turned around and become a signature scheme. To authenticate a message, the author of the message compute the "encryption" of the hash/digest of the message using his own private key (note private, not public). The message is sent with this "encryption" appended as a signature. Any receiver of the message can verify by "decrypting" the signature using the public key of the author, and check that it is the same as the hash/digest of the message. Malory would not be able to forge a message and pretend that it comes from the author because he would be unable to compute a signature that passes the check above without knowledge of the private key. (This is cryptographically due to a combination of the collision resistance of the hash as well as what is known as the [RSA problem](https://en.wikipedia.org/wiki/RSA_problem))

(The hash bring down the size of the material to perform RSA operations on, and ensure that the signature size is small even if the payload is large)

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

TODO

# Putting the ingredients together

We are now ready to reinvent blockchain by thinking through the steps.

1. First we use the append-only log as our data architecture. It should, of course, be durable.
2. Our network architecture is that of a simple, public P2P network with no identity needed to join. Conceptually, the network topology should be that of a mesh/fully-connected network to present the idea that there is no "special" node.
3. (This is the beginning of the non-trivial part) Any node can submit new log entry to the database by broadcasting. How are we going to deal with malicious entry? First thing we do is to change the semantic of receiving a message/entry: instead we are now only accepting a proposal. There can be multiple proposal for the "next" message/entry, and instead of nodes exchanging their opinion on accept/reject (which is too intricate/subtle with the distributed algorithm thing), all nodes simply note down all proposals without a definitive answer.
4. Here we introduce two ideas: Chaining and Proof-of-Work. Remember that a list can be implemented as a linked list, where each entry contain a pointer/reference to the next/last entry. Also remember in some CPU course that there is an interesting form of addressing scheme known as content-addressable. Since we have the tempering-resistant requirement, an interesting idea is to use a variation of that by using the hash of the whole entry as the pointer/reference. All submitted entry should contain a field with a hash pointing to the previous entry. Then a node's store of all proposed entries will have the data structure of a tree [^tree], with the branches representing competing claim, supposedly by different agents, of what they want the next entry to be. Coupled with this idea is the Proof-of-Work problem. There can be many variations in how this problem is implemented, but in the classical example we require that an entry solves the hash-preimage problem. Of course this problem in full is computationally infeasible at all, so we loosen it to only require a partial pre-image. For reasons we will see later, we also make the "difficulty level" tunable. To be precise, the entry should have 4 fields currently: `prev-entry`, `content`, `difficulty`, `pow-solution`. `pow-solution` should be chosen such that `Hash(prev-entry || content || pow-solution)` has at least `difficulty` many leading zeros. Since our hash is secure, the only way to find even a partial preimage, is to use brute-force: enumerate all possible values of `pow-solution` and compute the hash repeatedly, until we find one that satisfy the constraint.
	* The cumulative effect of what we did is that we slowed down the production of new blocks to prevent spamming. Also, a crucial property of using chaining with hashing (which is a form of digital signature without identity) is that it already confers some amount of tampering resistance. To see this, imagine Malory wanting to change some record in the middle of a chain. To do so, he need to essentially "rewrite history" and submit a new set of blocks containing his version of the entries, where only the records relevant to his aim are modified while the rest remain the same. However, changing the block content will completely alter the hash, so the next block pointed to by his modified block will also have the hash completely changed, even if the content of that block did not, because the hashing is over the entire block, including the headers. Applying the same logic, Malory will have to recompute all downstream blocks. Due to PoW, the only way to do this is to solve the PoW problem again for each reconstructed/modified blocks. Therefore, the deeper the target block is, the longer it will take to do it. (Malory need to broadcast these "fake" blocks and get it accepted by those nodes, and as the cannonical branch too, because for data read, client may contact any node in the P2P network)
	* This still leave us with a question - why is it called a *Block*chain? This is because the fundamental unit of data in the network is a block. A block is a header (consisting of stuff required to make it work like the PoW, link to previous entry, other important metadata, etc) plus the content. But here is the thing: the content in this case usually *batched*. At the application/user level, anyone can submit a transaction containing some user level data (conforming to the append-log data model) by contacting any node in the P2P network who accept it. These node will collect them into what is called a *mempool*, and then periodically select some subset of the entries there to add to a new block, and then it will try to link it with the latest block, solve the PoW problem, then broadcast it. [^uncle]
5. And this is where the clever steps is. We again do two things at once: impose an economic incentive to produce new, valid blocks, and to add an implicit condition that the cannonical history is the longest branch in the tree.
    * For the economic incentive, we stipulate that the entity who mined a block (that is, produced a valid block and solved the PoW for that block successfully), should receive a monetary reward. To this end there should be an additional field in the block indicating who mined the block. Upon a block being accepted as cannonical in the network, payment should be made to the miner. (As you will see in the last section, *how* this payment is made is done in a really clever manner) Basically, the miner sign his own paycheque.
    * Remember back when we say that there is no explicit accept/reject of any blocks sent by any one in the network (as long as it passes the validity tests). We then reasoned our way to the possible blocks having the structure of a tree. We then now declare that honest nodes/the spec/the official implementation should consider that the longest branch is the cannonical one. So that if any client query an honest node for a state, the node should compute the current state by replaying transactions along the cannonical branch.
    * Why would this work to make the blockchain temper-proof? Because the rate of growth of a branch is proportional to the computing power working on growing that branch, which in turn is proportional to the number of nodes working on it (assuming that each node have equal computing power). If a majority of nodes are honest and all work on the longest branch, then even if the rest collude in a conspiracy plot to takeover the network, in the long term the alternative branch's growth will be out-paced by the main branch.

## Note on Block Confirmation

We already saw that there is no explicit accept/reject of a valid block. While the longest branch rule implicitly specify which block is cannonical, one may ask whether the longest branch can change over time. Together with the observation that changing blocks deeper back in the past (by creating a branch/fork) takes more effort, blockchain system often employ the following rule:

> A block is considered **confirmed** if it is at least n blocks deep in the longest branch from the most recent block.

There is a more theoretical rationale for this setting. Some analysts have performed mathematical modelling of an attacker, and they found out that the probability of success of an attack that target specific block by immediately forking and trying to overtake the main branch (at least in the short term) decreases exponentially as the block get burried deeper in the branch. Therefore, the parameter `n` is a security parameter that is tuned to make this probability low, while balancing against the time delay that comes from having to wait for newer blocks to pile on.

See: [Bitcoin wiki](https://en.bitcoin.it/wiki/Irreversible_Transactions#Attack_vectors), [Ethereum Blog](https://blog.ethereum.org/2015/09/14/on-slow-and-fast-block-times/).

## Note on Block Validation

We have repeatedly mentioned "valid block". Because any one can propose a block, and malicious actor cannot be ruled out, we need some safeguard in place.

A block is subject to a bunch of validation rules. This include, but are not limited to:

* The PoW have to be solved properly.
* The content in the block should pass any application-specific check
  * e.g. if the transactions are signed by the initiator, then those signatures should be verified
  * e.g. The format (if there is one imposed by the application) should be valid
* The transactions should transit the *global state machine* to a valid next state. (This is important, see the last section)
* And any additional checks for implementing various chain-related mechanisms.

When the rest of the blockchain is correctly implemented, these properties then become something gauranteed as part of the temper-resistance of the blockchain as a consequence of the consensus. The reason is that if all honest nodes follow these rules (we have no reason to expect malicious node to play by these rules), then all blocks in the cannonical branch would have already passed these checks to be accepted by honest nodes in the first place.

## Note on Dynamic Difficulty

Now let's address this. The reason is that "computing power" is also dynamic. For instance new and better CPU/GPU/ASIC may come out over the years, changing the time required to solve a PoW problem. Another reason is that since the production rate of blocks is proportional to the number of honest agents [^pooling], if more and more people join the blockchain (as Miner), the time to produce a block may change.

Since our goal is to maintain a roughly constant average time to produce a block, preferrably one that balances the need to combat spam versus having fast block time to improve user experience (i.e. time from submitting a transaction to having that be confirmed), it makes sense to have the difficulty adjustable. In fact we would bake in some form of negative feedback/PID style control theory in - take the moving average of the block time for the most recent blocks, and increase difficulty if the time is shorter than the target average, and vice versa.

This can again be implemented by using the general philosophy of how we incorporate new rules into the system's spec so that the consensus mechanism takes care of the rest. In particular, we further add a field called `timestamp` into the block so that the time to produce blocks can be estimated in a cannonical way. Then we specify that the block difficulty must not be less than the bar calculated according to the moving average law above in order to be accepted. If a majority of agents observe this rule in validation, cheaters would generally not be able to get away with using a lower difficulty. On the other hand, economic efficiency would ensure that honest agent would attempt to use the lowest permitted difficulty possible.

# The Final ingredient: Game Theory

Notice that we still haven't specified/used any consensus algorithms from ingredient 1. But we don't have to - why? If we assume that the majority of nodes are honest, then they would all work on the cannonical branch, and this branch will grow fast enough that its cannonical status cannot be contested (at least among the honest nodes themselves). Thus instead of an algorithm, Consensus is implicitly reached by nodes all following the same rule to reach the same outcome, under an economic incentive to do work.

But this doesn't completely answer the question. Why should a majority of node be honest?

Turns out, there is no intrinsic reason to. However, if the network started out with such a majority, and provided that the network size is reasonably large, then this majority tends to be stable over time. And this is because of a concept in Game Theory known as Nash equilibrium.

In short, if a miner is honest, then he can expect to get the normal financial reward for participation. if he expects the majority in the network to also be honest, then there is no incentive for him to unilaterally become dishonest (say, by joining the largest dishonest group) as he will waste electricity to run the program, but will not receive rewards as his blocks won't be accepted as cannonical (since the dishonest group is still a minority), and this will result in a financial net loss.

The way this equilibria can be broken (in the game theoretic context, not counting other attack vectors) would be for a sufficiently large group to collude and become dishonest together, overpowering the honest group. Hopefully, this is hard enough to be infeasible in practise if your network is large. (Unless you have nation state level of resources and even then - we are talking about overpowering the combined computing power of the honest group in the *entire network*, which is international and contains player as diverse as individual miner to large corporations using data centers to mine)

Reference: [CoinMonks article](https://medium.com/coinmonks/blockchain-consensus-protocols-game-theory-fc3504e4da99)

# Emdedding cryptocurrency into a blockchain

This still leaves us with one final, perhaps most important question: how do miners get paid? The title of this section is the answer.

Cryptocurrency in this context is just a very particular application of blockchain, one that is necessary to its basic functioning (yes, there is a recursive/self-referential reasoning involved here, and this is what makes it clever, but also what makes cryptocurrency "The ghost of blockchain")

## Distributed Ledger

Imagine trying to model a bank using blockchain. How'd we do it?

* Bank Account: Just an ID, that is, a public key. The owner of the corresponding private key have complete control over the account.
* Account Balance: Just a number (denominated in the currency of the crypto). Also, "just a database record" (see the Interlude).
* The Global State Machine:
  * State: A map of `bank account -> account balance`.
  * Transition: A fund transfer operation

```
{
  "from": <sender bank account>, 
  "to": <receiver bank account>, 
  "amount": <a number>
}
```
* State Update is what common sense would dictate (note that negative account balance is not allowed - this can be enforced as an application specific rule during block validation)
* To prevent unauthorized fund transfer, transaction must be signed by the owner of the account in the `sender` field of the transaction. That is, only the owner of an account can transfer fund out of that account.

And that'd be it (!). While nothing is "real", if the consensus in the network says that you have this amount of money, then for some people that's about as good as real (at least, compared to the counterpart of a real world bank as elaborated in the interlude section).

(Note that one notable downside of this implementation is that the ledger is public - everyone can see the flow of money and who owns how much money. In some application, such as the treasury of NGO, it can be an advantage like having transparency and accountibility. But for most people this is an obvious cons. Therefore it can be important to use it in an annoymous manner so that people only know that "someone" owns this amount of money, but are unable to link it back to real world identity)

## Miner Fees and Printing Money

## Circular logic

## Note on the Role of Gas fee

## A small note: Where did the economies go?

(Not a pun on the miserable state of real world economy, but, ya, I am sorry.)


# Types of node - Miner vs Validator, and Light client


# Conslusion

This has been an unusually long post. Sadly, even then, we've only touched the surface of the blockchain world. However, if you made it this far and thoroughly thought through it, you should now have a solid grasp of the fundamentals, which make it possible to read other technical articles in the blockchain world without getting completely lost instantly.

# Post in this series

1. Convergence of Ideas and the Story of Blockchain (this post)
2. Wallets in cryptocurrencies
3. Smart Contracts, Tokens, Decentralized Apps/DApp, and DAO
4. Advanced Topics: Layer 2, Sidechain, oh my!

[^tree]: We can rule out forest by hardcoding a special root node, aka the Genesis Block, into the blockchain program and asking that programs only recognize blocks whose lineage can be traced back to having this block as an ancester. The program should suspend some of the usual validation rules for this Genesis Block as needed. Note that together with constraint on the data structure (each block have exactly one field for `prev-entry`), cycles and DAG are also ruled out.

[^uncle]: Astute reader may point out that an unlucky node may have a stale pointer to the "latest" entry since producing new blocks take time. That is, one may have the "right" pointer at the moment the production start, but get snooped as a new legitimate block appear in the middle of solving PoW. Different Blockchain solves this problem in a different way. For instance, Ethereum solves this by having the concept of an *uncle* - subsequent blocks can, aside from the link to the previous block, also add links to these branched, but otherwise legitimate, blocks. This makes sure that the transactions there are still considered as cannonical. Ethereum incentivise this behavior by allowing partial reward.

[^pooling]: You may think that this almost, but doesn't work due to the "quantum" problem - that only one miner can work on a single block, even if the PoW is difficult. This can be solved somewhat using a *mining pool*. A mining pool is like a cooperative, where people come together to "pool" their computing power. The owner of the pool is the one that actually proposes block in the blockchain system, but it delegates the PoW problem to the Workers in the pool like in the Distributed Computing Projects using BOINC for Folding@home. Since PoW is an embarassingly parallel problem, we can break it down into very small pieces (say by segmenting the "search space") and hand it out to the Workers. The owner should distribute the reward to the Workers according to their contribution (There are many possible scheme for calculating the share).

