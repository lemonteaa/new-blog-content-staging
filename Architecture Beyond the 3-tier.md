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
