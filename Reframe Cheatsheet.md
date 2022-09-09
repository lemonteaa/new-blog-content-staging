# Reframe Cheatsheet

This is written for those of you who don't want to wade through pages after pages of doc and "propaganda". I do assume you have a decent background in programming.

## Background: React and Flux

This is a pre-requisite. As it should be common knowledge for frontend developer, we just do an executive summary.

React gives you the conceptual model of `View = f(Model State)`, where `f` is a pure function. It is actually more of a nice interface/API that it presents to the developer/end-user - React works behind the scene to make this a reality. A completely naive implementation would just watch the state and trigger re-computation of `f`, then refresh the whole DOM-tree. Because performing DOM mutations/CRUD through the API provided by the browser has cost, we want to minimize it. Therefore, React maintain a virtual DOM and have a reconciliation process to sync between the virtual vs actual DOM by doing a diff and then only making necessary changes.

After this, there is a bunch of (complected?) problems that get summed up as the **state management** problem.

First, even with the DOM call cost problem solved, it still seems silly to fully recompute `f` each time there is any change to state, as state is in general a composite data structure. One solution is to use Reactive Programming - have a dataflow/signal graph to support deriving computed values from (multiple) source states, and make it possible to update the computed value efficiently as the source state changes. 

As people begin building larger and complex app, it becomes clear that 1) various part of the View (described as a tree of components) may depends on various part of the (global) state, and 2) clarifications need to be made regarding how should interactivity lead to changes in state. (2) is hopefully clear - while libraries like Angular have two-way data binding, by opting into the reactive model a one-way model become the default answer - events in the browser trigger actions that changes the state. Observation 1 lead to questions like how should the state be distributed/organized, and in combination with (2) there's the coupled question of how should actions be scheduled.

There are two general philosophies:

- Decentralized (Vanilla React, Reagent, Fulcro (formerly om.next))

In this approach states are split and distributed across the various component, usually using locality/co-located principles. In Vanilla React Unidirectional Data Flow describes the bundled ideas of (2) plus the restriction that state scoped locally to a component can only affect itself and its child component, but not its sibling or parents. While Reactive Programming is used, there is the further choice of it happening implicitly/tightly coupled to the unidirectional data flow model, or more explicitly, via a separate graph that is visible to the developer and can be manipulated directly.

- Centralized (Flux/Redux, Re-frame)

Here all states are centralized into a single store/mutable value, and the components read from parts of this global state that is relevant to them (in re-frame, through subscription). Again, due to concern with efficiency the use of reactive programming is usually implied. In the usual presentation (2) is also bundled with the idea of centralized state, and a more rigorous event scheduling discipline can be imposed to form the Flux architecture. In this model events/actions are isolated and ordered by being put into a queue. Changes in state happen through reducer, which are **pure functions** of the form `R: (Current State, Action) -> (Updated State)`. The name of reducer comes from basic functional programming - if we have an ordered list of the event/action, and an initial state, then the higher order function `reduce` (also called `fold` in some language) will let us compute the final state by doing something like `reduce(list, R, s0)`. Handling side-effects is a problem in this approach and different implementations do this differently, however the one consensus is that the reducer must remain a pure function to retain the benefit of this conceptual framework (easier to reason about state change in face of what can be a messy network of events).

Finally, it should be noted that while there are indeed tension between these two philosophy, one can also mix them to leverage their respective strengths - allocate "low level" states that are only related to the interactive functioning of a widget to the local state in the decentralized approach, while putting application level states that many components may depends on into the centralized global state store.


### References

https://stackoverflow.com/questions/71855096/reactive-programming-in-react

https://github.com/reagent-project/reagent/blob/master/doc/ManagingState.md

https://flaviocopes.com/react-unidirectional-data-flow/

https://reactjs.org/docs/faq-internals.html

https://reactjs.org/docs/faq-state.html

https://blog.mathpresso.com/after-the-frontend-framework-war-here-is-the-state-management-one-daec38f22698

https://frontendmastery.com/posts/the-new-wave-of-react-state-management/

## First half of Cycle: Implementing Flux

Re-frame follows part of the Flux architecture, but with its own innovation added. To increase power and flexibility, two rather powerful machinary are introduced, which we explain below:

### Ingredient 1: Interceptor

Originally a backend concept, interceptor let us modify a handler in a very flexible way as interceptor is a further generalization of middleware. In the simpler model, a handler is a function `request -> response`, and middlewares are Higher Order Functions of the form `handler -> handler`. Middleware usually "wrap" handler and do some pre/post processing. As an example:

```clojure
(defn middleware [handler]
  (fn [request]
    (-> request
        (pre-process other-arg)
        (handler)
        (post-process other-arg2))))
```

By wrapping a handler using different middlewares repeatedly `(m1 (m2 (m3 handler)))`, we have an "orion" - request goes through the pre-processing from the outer-most middleware to the inner-most one, then the actual handler, then goes through the post-processing from inner-most middleware back to the outer-most one.

Interceptor is a data-driven take on middleware. We have the **context** `ctx`, which is a value with type of a map, where various info in the "orion" model above are attached. Interceptors are then pair of functions (`:pre` and `:post`) that modify the context - pure functions `context -> context`. In this approach handler can also be unified as just a special type of interceptor. Unlike middleware, where the final execution happens by calling the resulting function directly, here an explicit interceptor chain is formed - a list of interceptors, then an executor function is used to initial context, put the request into the context, and then run it against the interceptor one by one.

What make this more powerful than middleware is that the (runtime) interceptor chain itself is included in the context map as `:done` and `:todo`. Hence it is possible to change the interceptor chain itself, in the middle of execution/on the fly/during runtime, by returning a context map with those keys' value changed.

Example of turning handler into interceptor:

```clojure
(defn handler->interceptor [handler]
  {:post nil
   :pre (fn [{ :keys [req] :as ctx }]
           (assoc ctx :res (handler req)))})
```


### Ingredient 2: (Co)-effect System

Main Reference: http://tomasp.net/coeffects/

See also: https://codesai.com/posts/2016/11/using-effects-in-re-frame

This is something from the camp of advanced (static) type system/type theory. It is well known that one way to manage side-effect in a purely functional world is through the infamous Monad. (Monad is not limited to side-effect, though that's one of its application. Even within that specific application, more precisely it is about **management** of the side effects than the side effect themselves. Yes the world of Monad is full of [confusions](https://wiki.haskell.org/What_a_Monad_is_not) like that.)

There's an alternative/supplementary method though - (Co)-effect system. You basically tag extra info alongside the normal type to indicate side effects, and change the rules of type inference to keep track of them. Just like it is common in type theory (which is closely related to Category Theory in pure math), you can often take the dual and have some interesting intrepretation. In our case, the dual of side effect will be dependence on external world for a read only value that is used during the computation. And just like the analogy of monad vs effects, coeffects can be compared to comonads.

Back to Clojure. While Clojure is at its heart a dynamic language, it doesn't mean one cannot take inspiration from the static typing world and borrow its pattern. To compensate for not being able to get the benefit of type correctness, one can often be somewhat more flexible in adopting the pattern and doesn't need to adhere to every single law required by typeclasses in Haskell say.

So, in Re-frame, event handlers are related to the (co)-effect system in interesting ways. The handler (which remains a pure function) now has signature `(Coeffects, Event) -> Effects`. As there can be more than one type of (co)-effects, both effects and co-effects here are a map of all the (co)-effects collected together.


### Putting it together

We now finally have enough background to talk about Re-frame. First from the point of view of end user (i.e. developer), it is an implementation of Flux. User actions (and other sources) trigger Events (**not** the same as event in browser DOM - more on this in next paragraph), which are processed by event handlers (analogue of the reducer). Side effects are handled using the effect system. To further encourage having your event handler be a pure function, coeffects are used to inject values dependent on external world. Here is a clever unification/generalization: recall that in Flux, one of reducer's argument is the global state. In re-frame, it is recognized that this global state is just a coeffect (and you may have other coeffects alongside it).

**Event in re-frame vs DOM Event:** to make the re-frame library more general, these two are de-coupled. Event in re-frame is really just a pure value. More precisely it is a vector, the first item is a key indicating its type. The function `dispatch` then serves as the entry point of the library, so usually when you program a frontend app you'd call the entrypoint from the DOM event handler, stuffing in param as additional items in the re-frame event vector.

Next, we talk about Re-frame from an implementation point of view. Since interceptor is so powerful, it is the foundation upon which both event and effect are implemented.

**Relation of Event vs Effect:** This is a tricky one. One on hand, an event can be considered just one kind of effect right alongside other more normal ones such as updating the global stores or making external HTTP calls. One the other hand it can also be considered the most general type of effect. When dispatched, it will be run through the whole machinary of re-frame core and results in triggering any further effects, including other events.

**How to implement it:**

Let's do event handler first. The key to making it works is to insert default interceptors that implements a generic, extensible (co)-effect system.

- `do-fx` is an interceptor that will performs all the side-effects at the end, after handler returns.
- `inject-cofx` is a function to return an interceptor that will inject the specified co-effect. (See following discussion though)
- `fx-handler->interceptor` is a HOF/Factory function that turns an event handler into an interceptor wrapping said handler.

Extensible means that a registration system is used. The clearest example is the first one/effects. `reg-fx` takes an id and your side-effectful "effect handler", and saves it to a registry. Then when `do-fx` receives a map of the effects to execute, it will lookup the "effect handler" from said registry and execute it. The advantage is that it allows a centralization and enable an "universal handler" like `do-fx`.

The case for co-effect is more complicated and in fact re-frame's public API has an assymmetry here. This is due to the consideration that different events may need different kinds of co-effects - hence more control is handed to the end-user to specify them on a case-by-case basis.

Another difference is that a kind of HOF/Factory pattern is used here also. This allows the flexibility of parametrizing your coeffects. `reg-cofx` let you register such a parametrized coeffects, then you instantiate it using `inject-cofx` and supply the parameters.

Finally, the actual event handler. There are a few points to note:

- You can customize the interceptor chain associated to an event (e.g. to inject your own co-effects), though some defaults are inserted for you.
- The registry is looked up when processing the dispatch event queue to retrieve the interceptor chain to execute the request on.
- As event itself is also parametrized, we need to include it as an input - aka yet another coeffect.
- The function mentioned above does some (de)sugaring with the context map so that end user can supply a handler that's more like `req -> res`.
- For convinence, in the common case where the global state store is the only coeffect *and* effect (aside from the event itself), a variant of the main `reg-event-fx` that has a simplified function signature for the event handler is provided.
- So `reg-event-fx` will take the handler, wrap it into an interceptor, add the default interceptor for the (co)-effect system, insert user-supplied interceptor, then register the chain.

### API Summary

**Primitive data type:**

- Event: A vector, the first item of which is the event type/id, and the rest are user designed parameter. E.g. `[:submit-form form-values url]`.
- (Co)-Effect map: Map whose key is the (co)-effect id and value is the corresponding (co)-effect value. 
- Id: Usually a keyword. May be namespaced.

**Core functions/public API:**

- `(dispatch event)`: Dispatch an event.
- `(reg-fx id handler)`: Register an effect handler. Effect handler is a side-effectful function `effect-value -> nil`.
- `(reg-cofx id handler)`: Register a coeffect handler, which is a side-effectful function `(coeffects, param) -> coeffects`. It should updates the coeffect map by assoc'ing its own coeffect type with the obtained value, input is the existing one from context. `param` is arbitrary, user designed value to allow parametrization of coeffect.
- `(inject-cofx id value)`: Return an interceptor that will inject the specified coeffect. `value` will be injected into the `param` argument of the coeffect handler.
- `(reg-event-fx id interceptors handler)`: Register event handler. `interceptors` is a list of interceptors supplied by the user and will be inserted into the resulting interceptor chain. The event handler `handler` is a pure function with signature `(coeffects, event) -> effects`. (Effect and coeffect *maps*)
- `(reg-event-db id interceptors handler)`: Simplified variant of `reg-event-fx`. Both (co)-effect maps in the handler are replaced with just the `db` value (before and after).

**Builtin (co)-effects:**

- `:db` (Both coeffect and effect): Manipulate the global state store, which is a cljc atom. The new value will be `reset!`'ed onto the atom. Expected to be a map, but the exact scheme is up to the user's design of application state.
- `:fx` (Effect): An effect which actions other effects, sequentially. Value is a list of `[effect-id effect-value]`. It is a list instead of map due to the need for ordering.
- `:dispatch` (Effect): Dispatch the value as an event.
- `:dispatch-later` (Effect): dispatch one or more events after a given delay. Value is a map with two keys: `:ms` (milliseconds delay), and `:dispatch`: The single event or a list of events. As the event type is a list, `:dispatch` can be either a list of a list of list.
- `:event` (Coeffect): For internal use to supply value to the event handler. Value is the event currently being processed.

**Example:**

Some code that demo everything in one go (We make use of nested vector a lot, and this can be confusing. Check the signatures listed above to clear your head. Also ignored argument are by convention named `_`):

```clojure

(reg-fx :localStorage (fn [[k v]] ...))

(reg-cofx :random (fn [coeffects [lower upper]]
                    (let [r (random lower upper)]
                      (assoc coeffects :random r))))
(def my-random-interceptor (inject-cofx :random [0 100]))

;; Can register more than one kinds of event handler

(reg-event-fx :demo 
              [my-random-interceptor]
              (fn [{ :keys [db random] } [_ form-values other]]
                ;;...
                { :db new-db 
                  :fx [ [:dispatch [:show-loading]]
                        [:dispatch [:submit-form form-values url]] ] 
                  :localStorage [:backup form-values] }))

;; Later on inside some JS DOM event handler
(fn [evt]
  (let [form-values (extract-form evt)
        other (sth)]
    (dispatch [:demo form-values other])))
```


## Second half of Cycle (TODO)
