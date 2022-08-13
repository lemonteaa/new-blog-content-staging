# Experience Sharing and Opinion Piece: DIY Clojure Web Framework? Lesson Learned

*(Note: The "meat" of this article is in the "Bilayer Design" subsection)*

## Background

### REPL-driven, Incremental development, and Unit-testing

*REPL-driven development* is a term commonly thrown around the Clojure community, and outsider may find this obsession to be strange, given that in 2022, REPL ain't really anything special anymore - most mainstream programming langauage have one (even those that are normally resistent, such as Java, have `JShell`). What's the deal?

While there are theoretical argument, Software Engineering is an empirical discipline - sometimes it does takes getting your hand dirty and experiencing things first hand to really feel the visceral effect of those ideas, to internalize them in your body. I think I experienced just that while building out my own web stack in Clojure.

The nature/category of a problem matters.

Some problems are *closed* - for example an algorithmic one, or an logic puzzle. All the information is already self-contained, and the main difficulty is in figuring out a *sufficiently clever compute*.

Other problems are *open* - example would be an integration project, where you want to combine existing pieces, that may be many, and scattered all around, and try to get them to work together nicely. Here the difficulties are many: complex enviornments, unknown/incomplete interfaces/specifications, incompatibilities, etc. Mathematical Logic seems to be rendered mostly powerless in such situation (if there are logic, they are mostly shallow, but widely spread and distributed).

Yet a third category of problem is *open* (in another sense of that word). Here even the problem itself is not specified at all, and the main problem, unironically, is precisely to write a problem *you* like. Example would be *exploratory programming*, where instead of having a higher up sending down an instruction to solve some problem according to some spec, you interact with the computer in a symbiotic way to co-create something.

--

For the second category, I try to build it by using an *incremental approach*. I recognize that I am not that smart - I have very limited working memory compared to the computer, and most likely sucks when trying to hold the whole system in my own head. So by using a REPL, I only work on one piece of the puzzle at a time, focusing on local information in my head while leaving the global picture to the computer. This reduces *cognitive load* and make the experience much more pleasant.

Many problems in the second category are, to a certain extent, also a problem of the third category. By using a REPL, we can become something like a scientist, doing experiment and observing its behavior. Only by experiencing first hand its behavior can one gradually begin to develop a taste for *which* behavior you like/don't like, which then allows you to formulate your own *taste* - a key ingredient in designing your own favorite framework.

--

As a quick side-note, hopefully this shows why one shouldn't be too dogmatic in "best practise" - very often there is a context/assumption under which it is applicable, and it is indeed unreasonable to demand any "laws" to be universally valid (scientific laws like QFT notwithstanding).

Case-in-point: Unit testing. They are intended to be a safety guard to automatically verify some behaviors *you* specified as "expected", so that you can iterate with more freedom. It is **not** intended as an artificial limit during the exploration phase. [^1]


### UNIX philosophy: why build your own framework?

> Isn't it re-inventing the wheel?

Many newcomers to Clojure are surprised that the ecosystem doesn't place as much of an emphasis on framework. Instead, one is encouraged to "DIY" by *composing* from small, single-purpose, focused libraries.

One argument for that is that framework inevitably have to make assumptions, and since web app often have their own specific circumstances, one usually ends up pushing the limit of the framework they use, trying to bend it to their will to do that one "very specific and special requirement". This results in contortion that are uncomfortable for everyone.

However, there are also argument against it:

- Building a framework is generally difficult. It requires maturity and experience to pull off, and otherwise could results in poorly written code that have far worse consequences in terms of maintainence nightmare than merely poorly written application code (that one can refactor). This is especially true if the framework have downstream users/dependencies.
- It can also be an ego trip due to the satisfaction from pulling off what is considered to be a challenging task. (Not-invented-here syndrome)
- Using a standardized public framework have the benefit of community support and resources, as well as decreased friction when changing developers and/or letting third party read the source code of your projects. You're on your own in the case of custom framework, unless you publish it to the public *and* it gains traction.

--

I don't have all the answer. However I do want to note that Clojure as a language is rather special. Because of its high expressive power compared to other mainstream language, *the cost of learning a new modular library is almost trivial compared to comparable libraries in other language*. The cost has indeed shrunk to such a ridiculous degree that for practical purpose, one can treat libraries in Clojure as "an atomic unit of thought", similar to how your treat a single functions in other languages. The implication is that if you're building your framework by composing libraries, it is very much feasible to just conceptualize entire framework in your head, *on-the-spot*. (You can still do this for frameworks in other langauges, but I find this is usually the end results after carefully studying and using it hands on, not at the beginning of the project as the first thing you do)

More importantly, it is feasible to quickly iterate on your framework idea by modifying said conceptual picture on the fly, as you prototype it out, touch ground with realities, and discover new inforamtion on how it's doing vs what you want.

## Ideas gained

### Review of Trends

Here we quickily review existing trends and patterns/architectures/ideas that influenced my thinking, and led me to the findings in the next subsections.

**From DSL to Data Driven**

Because of Clojure's homoiconicity, it is particularly well-suited to using macros to invent one's own Domain Specific Language (DSL) for a complex task at hand, in order to express thoughts more naturally.

However, unlike previous Lisps, Clojure positioned itself as a modern language which embraces the functional paradigm, placing emphasis on the design/architectural advanatges of pure functions (composable, referential transparency, testable...). Macros do violates the functional principles...

Then come "Data-driven". Instead of macro forms and/or function calls:

```clojure
(route "/api"
  (GET my-handler)
  (POST (fn [req] (http-response "Hello!")))
  (route "/foo"
    (GET handler-b))
  (route "/bar"
    (GET handler-c)))
```
(here `route`, `GET`, `POST` are all functions and/or macros)

We solve it with "one more layer of indirection". Express it in a syntactically very similar way using literal syntax for *plain values*:

```clojure
(def routes
  ["/api"
    {:get my-handler, :post (fn [req] (http-response "Hello!"))}
    ["/foo" {:get handler-b}]
    ["/bar" {:get handler-c}]])
```

(Look at it hard - despite superficial similarity it actually is "just data", like JSON. Remember that `[item1 item2 ...]` is a vector, while `{key1 val1, key2 val2...}` is a map)

Then have a special "compiler" function that can transform it into an actual object/thing capable of doing what it promised to do:

```clojure
(create-router routes)
```

Aside from the usual advantages of functional style, one advantage is that because it is just data, it can be programmatically/dynamically transformed using plain functions. This extra flexibility can be powerful in more advanced problems.

The data driven approach have seen wide-spread adoption across a range of libraries in their "new generation" variants.

**Hexagonal Architecture**

One major critique of pure functional style is that the world we live in is in fact heavily effectful and impure. How can we interact with the outer world (which is unavoidable), without losing its benefit?

The hexagonal, or "inside out" architecture is one such proposal. [^2] We separate our system into layers that are sorted into concentric circles. Layers are assigned according to the degree of purity - the purer components are placed closer to the center. It goes roughly like this:

- Functional core: completely pure function, our business logic and domain model.
- Intermediate: Business operations, using the core to compute the effect needed, then translate to actions needed on DB and other physical components.
- System integration/periphery: actually perform actions, hooking up the physcial components together, do their configs/setups, trigger calls to inward layers.

There is an inside-out effect going on because while from the point of view of actual actions triggered, the flow goes from outside to the inside; the way we think about it is the opposite: information/effect flows from the core to the periphery. (TODO: not entirely sure about this, just an intuition) (can resolve apparent contradiction if we visualize the entire flow? Out to in then back to out)

**Inversion of Control**

(TODO: Classical Dependency Injection etc)


### Bilayer Design

While implementing my own framework, I originally thought that DI (Dependency Injection) is pretty much a solved problem by now and ought to be trivial. Clojure already have some mature libraries there, which I have previously used, and the experience have been mostly smooth.

But then two gaps showed up:

- The one that I used depends on Clojure namespace mechanism. This is convinient, but violates the functional principle somewhat.
- There can be problem with recursion, namely circular dependency in the namespaces.

My intuition is that there is some complecting going on here. So let's move on to a data driven DI library? Like `Integrant`. As it no longer uses namespace at all, things improved quite a bit. But quickly the problem apparently returned.

In integrant, the system is centralized - specified as a map with reference to components allowed. You supply methods to initialize a particular components, and integrant helps with the dependency ordering.

```clojure
; System as data
(def system-map
  {:comp-a conf-for-comp-a
   :comp-b conf-for-comp-b...})

; Supply your lifecycle methods
(defmethod ig/init-key :comp-a [_ conf]
  (do your stuff here))

; Start the whole thing
(def system
  (ig/init system-map))

; Get components
(:comp-a system)
```

The problem is that we both need to setup a component **and** use it. If we think about the flow, it is basically `<setup code> --> system --> <consumption code>`.

- If we group code by having each component use its own namespace, and the centralized system also get its own separate namespace, then we are back to circular references in namespaces (system need to import component for the setup code, while component need to import system to use it). [^3]

```clojure
(ns abc.comp-a)

(defn complicated-setup [opt]
 ...)
```

```clojure
(ns abc.di
  (:require [abc.comp-a :as comp-a]))

(defmethod ig/init-key :comp-a [_ {:keys [port user db]}]
  (comp-a/complicated-setup {:conn (assemble-conn-url user port db) ...}))
```

- On the other hand, if we move all setup codes into the system/DI's namespace, this does remove the problem, but seems to defeat modularity as now everything's crushed back together. (In general, "throw everything into a single namespace" is a code smell)

I think my counudrum at a deeper level is this: on one hand, from the point of view of programming, circular dependency of any form at all is generally a code smell/antipattern and should be refactored away. On the other hand, from a purely operational/system level point of view a high degree of interdependency is almost a given/fact of life.

--

Let's be patient and untangle it slowly. Maybe this is why integrant uses a centralized approach - if the physical component's interdependency is intrinsic/unavoidable, then let's push it all into a single place that can then be managed away.

Also, I then question myself: *why* do I want to use the components in the system like the old fashioned, namespace based approach of the last era?

```clojure
(ns abc.di)

(def system
  (ig/init system-map))
```

```clojure
(ns abc.comp-c
  (:require [abc.di :refer [system]]))

(defn submit-cart [cart]
  (let [db (:db/node system)] ; Old habits die hard - this is no longer pure!
    (transact! db (transform-cart cart))))
```

At this point, something about the combination of data-driven, hexagonal, and IoC "clicked" in my mind, and a small voice appeared.

> Let's actually execute Hexagonal? Using classical IoC as a help.

So we want to make `submit-cart` as pure as possible. Let's use the function-argument injection form of IoC/DI:

```clojure
(defn submit-cart [db cart]
  (transact! db (transform-cart cart)))
```

But wait, we know a weakness of this form is that if you need to inject a lots of stuff, the function's signature can become ugly. Luckily, due to Clojure's power, this is easy to solve: just "universalize" the injection:

```clojure
(defn submit-cart [ctx cart]
  (let [db (get-in [:system :db/node] ctx)]
    (transact! db (transform-cart cart))))
```

But our function is executed in the context of a web server, how to actually place our system into the context? Well, use an interceptor/middleware:

```clojure
(defn make-inject-context-interceptor [system]
  (fn [ctx] (assoc ctx :system system)))
```

Then arrange for it during the setup phase of our server, which is just one of the component of the system. The other components are made available during its construction thanks to integrant's managing this dependency for us:

```clojure
{:db/node ...
 :email-serv/api-client ...
 :server {:db        #ig/ref :db/node,
          :email-cli #ig/ref :email-serv/api-client
          ...}}
```

```clojure
(defmethod ig/init-key :server [_ {:keys [db email-cli]}]
  (create-server 
    {:interceptors [interceptor-a
                    (make-inject-context-interceptor 
                      {:db/node db
                       :email-serv/api-client email-cli})] 
     ...}))
```

--

Looking back at this solution, we have basically two different layer going on in parallel:

- At the level of structural dependency, we have made it unidirectional by making the application level function *nominally* independent of anything else, so it is now just: `<app level code> --> <comp setup code> --> system`.
- But at the level of runtime dynamics, it is just the opposite: `server in system -> intercept inject ctx -> app level code`.

Overall, everything fit together nicely, and life is good again.

--

I'd admit there is one tradeoff made though. In general, while pure function is a good thing (so our new version of `submit-cart` is "nicer"), because its argument is now the end result of a long chain of execution/setups (first by saving the components into the server via integrant, then injected by the interceptor at runtime), it can become harder to work directly with the function in the REPL, hampering quick iteration somewhat. (But we do gain the ability to independently unit test it with mock db!)

### Commodification of API server

One interesting thing I come across while developing the framework. I have been adding GraphQL capability using `Lacinia`, and also been defaulting to using `xtdb` as the database of choice as it have all the right resonance with me (the flexibility of datalog query [^4], plus the very nice Event-Sourcing Architecture that makes concurrency/transactions no longer scary, what's not to like?).

But how do you implement GraphQL resolvers using `xtdb`? Wait, speaking of which, isn't it that the datalog query language is spiritually similar to GraphQL? If that's the case, is it possible, in principle, to have a universal resovler? Oh wait, it happened already: https://github.com/xtdb/xtdb/issues/456 and then https://github.com/juxt/site

And come to think of it, it kind of make sense too: RESTful have been about *Resource orientation*, and as much as we would like to deny it, to some extent it does involves removing "behavior" from API, making it more about "just standardized, raw CRUD". But if there's no behavior, and CRUD is standardized, then there is no room left for a custom API server (!).

One may ask, but surely one cannot remove *all* behavior? Some transactions does involves business logic and are not reducible to "just raw CRUD". Well yes that's true, and that'd be my plan for my next experiment: exploring microservice architectures. (Event Driven, async + treating transaction as its own object, Transactional Outbox...)

## Technical Details

TODO: This is where I review my commit history for the repo `lemonteaa/abc` and examine the practical difficulties encountered during integration.

e.g. threeway merge between Lacinia, pedestal, and reitit.


[^1]: It is still possible to use Unit-testing as an aid during exploration, perhaps as a quick "mini-verifier" while you want to build some prototype. But its nature is different - as you quickly iterate the hypothesis, and hence the program spec itself, also undergoes rapid iterations. Therefore both your PoC code **and** the unit test co-evolve together and is best treated as one integrated entity.

[^2]: It also goes by many other names, some may be so different as to be completely unrecognizable. But to the best of this author's knowledge, they all refer to essentially the same basic idea.

[^3]: Now that I think about it some more, I am not sure if this is solved by moving the whole `defmethod` into the respective component's namespace too. I could be wrong though.

[^4]: Again thanks to a form of "universalization" of its data model of RDF-triple/EAV (entity attribute value).
