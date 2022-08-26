# Architecture Beyond the 3-tier

Once one finishes a typical Software Architecture Course in university and goes to work, a likely scenario is that he/she get some CRUD app development. Unfortunately it is a rather well-trodden field, and so one may get the impression that there is not much room for anything new in terms of architecture. This article (summarizing the author's experience and readings) hope to show otherwise.

## Review: The Orthodoxy

The *de facto* standard architecture for a CRUD web-app in the web 2.0 age is what is called a **3-tier/3-layer + MVC** architecture:

- Presentation Layer: Responsible for taking data and format it in a way that can be shown to the end user. Does HTML templating before the advert of SPA/modern frontend. The MVC of a web framework is also placed here.
- Business/Domain Layer: Handle the core business/domain logic. Can be a (internal or public) web service in a Service Oriented Architecture (SOA).
- Data/Persistence Layer: Work with the DB. May use a DAO or ORM (Object Relational Mapping) framework.


Short-coming?

## A Few Words on Thinking about Architectures

The following sources influenced my thinking on Architectures:

- Language of System: A famous talk by Rich, it tries to approach this problem in a more systematic manner by arguing for decomposition, as well as emphasis on identifying general purpose, reusable components for a system that is applicable across projects. The goal is to build a *language of the system* where we can talk about them at a higher level of abstraction.
- 4C view: The idea that there is different perspectives/concerns/aspects going on, and that each of them is legitimate and should be treated separately but equally. More importantly, clarify and define things clearly to avoid conflating things across views.
  - As an example, less experienced developer (me many years ago) may mixes up database schema and class diagrams at the conceptual level vs at the implementation level (the former should take conceptual clarity as the highest pirority and can elide certain details - implementation need not be a 100% match nor does it need to be perfect, say it can break down a table into 2 etc).
  - Another example: although there can sometimes be an apparent alignment between the implementation (code projects or modules) vs physical components, this can changes at a whim. Breaking tacit assumption this way can have negative consequences when concurrency is involved (e.g. calling API, syncing database...)

low vs high level, layer in the stack, importance of intention.

How overall industry trends and context affect architecture


## Dealing with Complex Data Domain

A common problem leading to CRUD app becoming increasingly unmaintainable is it having a complex data domain - when the number of table/field/relation grow beyond some numbers, SQL query becomes increasingly difficult to keep in ones head and reason about intuitively.

Here are some ideas proposed by other people over the years:

- Domain Driven Design and CQRS (Command Query Responsibility Separation): Proposed by Martin Fowler in ThoughtWork, its basic premise is that different departments may have fundamentally incompatible point of view on what appears to be the "same" object/data. It then proposes *bounded context*: separate things into domain using *boundary* to allow multiple view of what appear to be the "same" thing. On top of this conceptual design, one would then define an abstraction using *aggregate root* (consisting of a nestable bunch of entity/value) to represent the data. Operations are mediated through *repository*, and the high level goal is to implement a *Domain Specific Language* (DSL) that allows you to fluently talk about the domain in its "native language". This is known as *Ubiquitous Language*.
  - Although this may outwardly look just like the usual DAO/ORM, and can be a thin proxy for it in simple cases, the two are different. As DDD is implemented by hand, it can work around difficult situations, such as silo'ed, multiple databases; and heavily denormalized schema where the database's relational feature is no longer used, but act simply as a data dump for everyone.
  - Their semantics/goal are also different: a plain DAO is oriented towards faithfully representing the schema existing in a DB, a DDD aggregate/repository is aimed at providing its own "view" on the data by providing its own schema, and its own behavior. One can compare this to the 4C view mentioned above. One benefit is that we no longer have to comprimose between conceptual vs implementation schema - DDD provides the translation layer so that one can (approximately) code in the conceptual.
  - In short, one can think of it as "all problem in CS can be solved by an additional layer of indirection". Perhaps the real surprise here is that this still works even if the new layer is syntactically similar to the last one (!)
  - The tradeoff here is organizational complexity for flexibility to deal with domain complexity
  - TODO: Explain CQRS


- Alternative Database System. In this direction one interesting idea is Datomic in Clojure ecosystem - it uses RDF triple as its data model, instead of relation (aka table minus some technical difference) in a traditional RDBMS. The query language is usually datalog/variant of prolog/logic programming. One may think of it as the result of converting data to be fully dynamic and typeless. A benefit of doing this is that some graph-like query, which would have involved recursive query in normal SQL, would become easier to express.
  - TODO: concrete example with tree and children/ancestor.


## Dealing with Concurrency

A second problem - concurrency, is traditionally known to be difficult in CS. (One interesting tidbit is that concurrency spans across the whole tech stack - we have concurrency models inside the CPU's out-of-order optimization, we have concurrency at the OS level via threads and scheduling, and we have concurrency at the upper layer as well)

Although traditional RDBMS solved a part of it with ACID transactions (caveat: provided one use it correctly), these thorny issues re-emerge when one need to conduct actions *across systems components*, which can be inevitable in many cases.

As reasoning about concurrency in general is hard (interleaving), it would be nice if there are systematic design patterns that are known to work well. Well here are some that I particularly like:

- Idempotency + Distributed Saga + Transactional Outbox: Coming from the microservice/event-driven world, these solutions are still applicable even if you don't go all in on microservice.
  - Problem formulation: You want to do a business transaction (*not* just a DB transaction), that involves doing a sequence of actions on different places. These actions can be normal DB writes, as well as external API calls outside of your control. You want it to have as much of the usual ACID properties as it can.
  - A naive solution may involves just doing external API calls in a DB transaction. But what if it fails? We will need a way to rollback the api call as well. There have been idea along the line of a long-lived, distributed transaction, however, those requires special transaction coordinator, and requires that all participants conform to a rather strict spec.
  - A more adaptable method is to intrepret these requirement more loosely - instead of rollback, do a *compensating transaction* that undo the effect, but let the fact that both happened remains on record.
  - Idempotency is a simple idea, but very versatile in its application. With idempotency, you are free to retry as many times as you want until it succeed - we no longer need to worry about whether a non-response means a true failure, or just the network dropping packets. Its prominance perhaps comes from the detail that in distributed system's event bus/append-only log (two of the most important reuseable components), delivering messages *at-least-one* is easier to achieve than *exactly-once*.
  - Finally, the Transactional outbox can be considered the final missing piece of the puzzle. The problem it solves is this: observe that the main problem with RDBMS's transaction is that it only works within the boundary of the DB itself - *anything* outside the DB and you're out. But isn't the message queue/append-only log also a component outside the DB? Thus if the saga is initiated by a DB transaction, we'd be back at square one. Well, the answer provided by this pattern is to have a special outbox table in the DB that act kind of like the "in-DB shadow" of the queue - because it is in DB, including it in the DB transaction works. External components can then pick up on the outbox and mirror it in the actual queue asychroniously.
    - Why not have the external component scan the relevant tables directly? - Mainly for semantics, to provide a unified point of contact.
    - And won't performance be an issue? Yes, that's why a scalable implementation would use special CDC (Change Data Capture) technique specific to each DB. This involves interacting more directly with the low level internals of the DB.
- A different direction focuses on the *append-only log* abstraction, provided by generic components such as Kafka.
  - Using *consensus algorithms* in distributed system, it is possible to have strong consistency in the order of messages added to such a log.
  - A rather radical thought is that the order of messages in such a log can be considered the authoritative source of truth about the evolution of states of the whole system - this idea again have seen a long and deep influence, reaching to the world of blockchain even.
  - As an example, the *xtdb* database in Clojure (which share many ideas with Datomic), take this idea and uses the log coupled with an external document store (and an indexer that scan through the log to improve read performance).
  - One pattern (from the book "Designing Data Intensive Application") is to somehow combine this with distributed saga. An asynchronous, restartable worker scan the log, and do some idempotent work for each message - in order. The result is saved to another topic in the log.
  - This can be made scalable - partition/shard the log using domain specific knowledge to gaurantee that ordering won't affect the outcome. Example: different users contend for reserving (locking exclusive access to) some resources. If these resources are independent, we can shard by the resource id: order of reserving different resources are irrelevant compared to the order of reserving the same resources.


## Interlude: Pattern in REST API

- API gateway: rate-limiting
- Idempotency key
- Task Object and Long running operation
- Advanced query
- (Bonus) Patterns for webhooks

## Ergonomics of Semantics: Microservice and Clean/Hexagonal


## Keeping up with a different world (I): Modern Frontend

- Backend-for-Frontend (BFF)
- SSG, CMS and JAMStack
- (?) Island Architecture

## Keeping up with a different world (II): Cloud-Native

- What is cloud native?
- Containerization
- 12 factor app
- Patterns specific to k8s
  - Ingresses
  - Synchronization loop
  - (Advanced) Service Mesh, Observability, Distributed Tracing (not absolutely specific, but still)
- PaaS built on top of Kubernetes
- Patterns in hybrid cloud
  - Lift and Shift
  - Spike/overflow workload

## Bonus: Dealing with Legacy System

- Straggler pattern

## Bonus: Radically different Architecture

- End-to-end architecture
- Free Monad

## Conclusion

- software architecture is at the edge of soft vs hard field
- slowness of inventing new architecture vs even slower for software to support it. Why?
- hardness of changing mindware
- live in the present, but look at the future
- long term impact of progress

## References

### General

- https://www.martinfowler.com/books/eaa.html
- [POSA](https://en.wikipedia.org/wiki/Pattern-Oriented_Software_Architecture)
- https://www.youtube.com/watch?v=Y6B4jYBR4Y8 (Talking Architecture With Kevlin Henney - Wix Engineering Tech Talks)
- https://www.youtube.com/watch?v=1VLN57wJAcU (Fundamental Principals of Software - David Nolen)
- https://www.youtube.com/watch?v=0AkoddPeuxw (Leprechauns of Software Engineering by Laurent Bossavit)
- https://www.youtube.com/watch?v=ROor6_NGIWU (The Language of System - Rich Hickey)
- http://videos.ncrafts.io/video/223268149 (NewCrafts Videos - Laurent Bossavit - Perpetual Infancy - why software doesn't seem to be growing up)

### Append-Only Log

- https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying

### Microservice Patterns on Concurrency

- https://yos.io/2017/10/30/distributed-sagas/
- https://pradeepl.com/blog/transactional-outbox-pattern/

### APIs

- https://cloud.google.com/apis/design/design_patterns
- https://www.linkedin.com/pulse/what-we-learned-webhooks-after-integrating-100-apis-iacobelli/
- https://apisyouwonthate.com/blog/picking-api-paradigm

### DevOps Concerns

- https://www.eightypercent.net/post/layers-in-the-stack.html (Anatomy of a Modern Production Stack)

### Others

- https://docs.microsoft.com/en-us/azure/architecture/patterns/strangler-fig
- https://cloud.google.com/architecture/hybrid-and-multi-cloud-architecture-patterns
- https://jasonformat.com/islands-architecture/
- https://roca-style.org/
- https://tonsky.me/blog/the-web-after-tomorrow/
- https://www.stephenzoio.com/free-monads-and-event-sourcing/
- https://deque.blog/2017/07/06/hexagonal-architecture-a-less-declarative-free-monad/

(TODO: I still haven't included the *main* reference for free monad as web architecture)

(Note: Many of the idea in the first half of this article are already covered by the book "Designing Data Intensive Application". I just thought it may be worth rethinking them from the perspective of higher level software/system architecture.)
