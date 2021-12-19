# Clojure's lower async layer: from manifold to ring 2.0

Clojure has a range of nice facilities for working with concurrency. However, people who do works on a library level - especially foundational libraries in an ecosystem, have different needs from end-users. This post is an outsider's view into some special designs that appears in those dimensions. To be honest, this post is just an excuse to talk about a bunch of random stuffs.

## The compatibility layer of Manifold

Manifold provides two concurrency primitives that act as a compatibility layer for multiple similar, but sometimes incompatible constructs. Sorting by popularity, the gist of those constructs are:

* At higher level, Reactive Stream (RxJS in Javascript, RxJava in Java) and Go Channel are both powerful constructs.
* At lower level, we have Promise/Future, async/await, and if one is willing to go really low level, (asynchronous) callbacks.

To achieve its aim, Manifold provide the lowest-common-denominators of these different looking constructs. Also, it attempt to unifies them by performing certain universal transformation on a as-needed basis (e.g. turns synchronous construct back into asynchronous one by "wrapping threads around synchronous objects"). It also cleans up the API of these constructs, making it more uniform/consistent. Finally, it patches/cover up differences in the underlying execution models (especially across programming language).

## Deferred

"Deferred is like Promise" (Future is not used here as in a sense Promise is more powerful). Also, deferreds represent a single asynchronous value. The later is good for conceptual model, but the former is good for compairson with similar constructs in existing programming language to see how we can patch over fundamental difference in execution model (Java's Thread pool vs Javascript's event loop, say).

So, let's look at the API of Javascript's `Promise`, as well as its semantics.

* Constructor: `new Promise((resolve, reject) => { /* code */ })`. True to Javascript's async focus, it immediately launch the "executor" in async manner, i.e. the function you supplied as the solo argument. (Compare with constructor of `Future`) Note that either `resolve` or `reject` should be eventually called to deliver the result, thus it is like the Continuation Passing Style. From this we also see the first complication for what should be a simple concept: that there are two distinct type of values that can be delivered, and this is encoded directly at the construct level. This will have implications for all the rest of our APIs.
* Callback Registration using `.then`: `p.then(resolveCallback [, rejectCallback])`. From my biased view point, this is a heavily overloaded method as it achieve multiple things at once (some of it is unavoidable due to the complication mentioned above though). First, because there are two types of possible value to be delivered, it make sense to accept two callback arguments. However, the latter callback is optional. And this is related to the justification for making that complication: the rejection case is intended to be integrated with exception handling in async code, or more precisely, to try to recreate the nice semantics one has for exception handling in normal code. We will elaborate this over subsequent points.
* Return value of `.then` and Promise chaining: The `.then` method return a new Promise object. It resolves when one of the callback function in the `.then` method is called and return a result. Here we need to specify new semantics to disambiguate resolution vs rejection again: If it returns a value, the new promise is resolved with that value. If it throws an error, the new promise is rejected. If it returns a promise, the new promise will wait for the result of that promise. This allows us to do what is called "Promise chaining" using code like below:

```javascript
let p = fetchFromWebsite();

p.then((res) => {
    let data = parse(res);
    return saveToDB(data);
}).then(() => {
    return sendEmailNotification(user);
}).then(() => {
    updateUI();
});
```

TODO

* Error handling and `.catch()`/`.finally()`:
* `async/await`: 

Here's the Clojure equivalent/re-formulation in Manifold:

* Constructor and resolving `deferred`: We have an asynchronous model. But we do not force the coding style onto the user. Therefore, instead of "executor", Manifold provide explicit functions to resolve it either way, and it is up to you whether you want to do it synchronously in the same thread, or start a `future` yourself and do it there.

```clojure
(require '[manifold.deferred :as d])

(def d (d/deferred))

(d/success! d :foo)
;; Or this
;; (d/error! d (Exception. "boom"))
```

* Reading resolved value: Again, both synchronous and asynchronous methods are provided (we don't need `async/await` magic in Clojure world).

```clojure
;; Register callbacks/async
(d/on-realized d
    (fn [x] (println "success!" x))
    (fn [x] (println "error!" x)))

;; Wait for value/sync/blocking
@d
```

* Chaining and Error Handling: Because this is such an important, arguably the central use case, Manifold provides dedicated functions for them, and it respect the usual semantics from piror acts. `(d/chain d f1 f2 ...)` is roughly the equivalent of Javascript's `d.then(f1).then(f2)...`. Meanwhile, `(d/catch d ex-type handler)` catches "rejection" just like the Javascript counter-part.

An example of formatting the code using threading macro to make it looks natural/just like ordinary, synchronous programming using `try { ... } catch (Exception e) { ... }` block.

```clojure
(-> d
    (d/chain dec #(/ 1 %))
    (d/catch Exception #(println "whoops, that didn't work:" %)))
```

* Synchronous style programming with `deferred`: This is in my opinion the most interesting part. While Javascript tackle this with a new language construct `async/await`, Clojure is a functional style Lisp-dialect, and any long time lisper knows that we have Macros as our weapon. Therefore, we don't need to modify the Clojure language. Instead, `let-flow` is the answer. However, true to its philosophy of discouraging mutation, you need to declare all values that you need to `await` on in the identifier binding blocks. Note that it does work lots of magic behind the scene: it infers dependencies between the values, and it tries to concurrently execute independent paths if possible. Furthermore, a common pattern in Javascript is to have `async/await` in a loop. To avoid re-inventing the wheel, Manifold also provide the `manifold.deferred/loop` macros.

* Composing/Higher Order Function over `deferred`: nothing special here, the principles work just like in Javascript. (Referring to the `(d/zip)` and `(d/timeout)` functions.

* Conversion to other concurrency primitive: To convert into a `deferred`, just use value coercion.


## Streams



Moreover, it tries to gather APIs into one place, 

so that one would not experience the frustration that "this API is available on concurrency construct A, but I don't like the feel of construct A; on the other hand, I like the feel of construct B, but the API I want, that is available on A, does not exists on B".

we want there to be an alternative way to handle rejection value using the