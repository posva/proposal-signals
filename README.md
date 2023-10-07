# Standard JavaScript Signals proposal

This document outlines the goals of a standard JavaScript signal proposal and a minimal API, with the intention of bringing signals as a TC39 proposal in the future. The effort here is driven by authors and maintainers of related frameworks and libraries in JavaScript, with a focus on developing signficant experience using shared JavaScript libraries to refine the proposal prior to standardization.

## Background: Why signals?

To develop a complicated user interface (UI), JavaScript application developers need to store, compute, invalidate, sync, and push state to the application's view layer in an efficient way. UIs commonly involve more than just managing simple values, but often involves rendering computed state which is dependent on a complex tree of other values or state that is also computed itself. The goal of signals is to provide infrastructure for managing such application state so developers can focus on business logic rather than these repetitive details.

Signal-like constructs have independently been found to be useful in non-UI contexts as well, particularly in build systems to avoid unnecessary rebuilds.

#### Example - A VanillaJS Counter

Given a variable, `counter`, you want to render into the DOM whether the counter is even or odd. Whenever the `counter` changes, you want to update the DOM with the latest parity. In Vanilla JS, you might have something like this:

```js
let counter = 0;
const setCounter = (value) => {
  counter = value;
  render();
};

const isEven = () => (counter & 1) == 0;
const parity = () => isEven() ? "even" : "odd";
const render = () => element.innerText = parity();

// Simulate external updates to counter...
setInterval(() => setCounter(counter + 1), 1000);
```

This has a number of problems...

* The `counter` setup is noisy and boilerplate-heavy.
* The `counter` state is tightly coupled to the rendering system.
* There is a transitive circular reference between `setCounter()` and `render()`.
* If the `counter` changes but `parity` does not (e.g. counter goes from 2 to 4), then we do unnecessary computation of the parity and unnecessary rendering.
* What if another part of our UI just wants to render when the `counter` updates?
* What if another part of our UI is dependent on `isEven` or `parity` alone?

Even in this relatively simple scenario, a number of issues arise quickly. We could try to work around these by introducing pub/sub for the `counter`. This would make the following improvements:

* Rendering would now be decoupled from the `counter` and the circular reference would be gone.
* Additional consumers of the `counter` could subscribe to add their own reactions to state changes.

However, we're still stuck with the following problems:

* The render function, which is only dependent on `parity` must instead "know" that it actually needs to subscribe to `counter`.
* It isn't possible to update UI based on either `isEven` or `parity` alone, without directly interacting with `counter`.
* We've increased our boilerplate.

Now, we could solve a couple issues by adding pub/sub not just to `counter` but also to `isEven` and `parity`. We would then have to subscribe `isEven` to `counter`,  `parity` to `isEven`, and `render` to `parity`. Unfortunately, not only has our boilerplate code exploded, but we're stuck with a ton of bookeeping of subscriptions, and a potential memory leak disaster if we don't properly clean everything up in the right way. So, we've solved some issues but created a whole new category of problems and a lot of code. To make matters worse, we have to go through this entire process for every piece of state in our system.

### Introducing Signals

Data binding abstractions in UIs the model and view have long been core to UI frameworks across multiple programming languages, despite the absence of any such mechanism built into JS or the web platform. Within JS frameworks and libraries, there has been a large amount of experimentation across different ways to represent this binding, and experience has shown the power of one-way data flow in conjunction with a first-class data type representing a cell of state or computation derived from other data, now often called "signals".
This first-class reactive value approach seems to have made its first popular appearance in open-source JavaScript web frameworks with [Knockout](https://knockoutjs.com/) [in 2010](https://blog.stevensanderson.com/2010/07/05/introducing-knockout-a-ui-library-for-javascript/). In the years since, many variations and implementations have been created. Within the last 3-4 years, the signal primitive and related approaches have gained further traction, with nearly every modern JavaScript library or framework having something similar, under one name or another.

To understand signals, let's take a look at the above example, re-imagined with a signal API further articulated below.

#### Example - A Signals Counter

```js
const counter = new Signal.State(0);
const isEven = new Signal.Computed(() => (counter.get() & 1) == 0);
const parity = new Signal.Computed(() => isEven.get() ? "even" : "odd");

// A library or framework defines effects based on other signal primitives
declare function effect(cb: () => void): (() => void);

effect(() => element.innerText = parity.get());

// Simulate external updates to counter...
setInterval(() => counter.set(counter.get() + 1), 1000);
```

There are a few things we can see right away:
* We've eliminated the noisy boilerplate around the `counter` variable from our previous example.
* There is a unified API to handle values, computations, and side effects.
* There's no circular reference problem or upside down dependencies between `counter` and `render`.
* There are no manual subscriptions, nor is there any need for bookkeeping.
* There is a means of controlling side-effect timing/scheduling.

Signals give us much more than what can be seen on the surface of the API though:

* **Automatic Dependency Tracking** - A computed signal automatically discovers any other signals that it is dependent on, whether those signals be simple values or other computations.
* **Lazy Evaluation** - Computations are not eagerly evaluated when they are declared, nor are they immediately evaluated when their dependencies change. They are only evaluated when their value is explicity requested.
* **Memoization** - Computed signals cache their last value so that computations that don't have changes in their dependencies do not need to be re-evalulated, no matter how many times they are accessed.

## Motivation for standardizing signals

Given extensive experience with signals as a broadly useful construct, we propose to add a built-in signal construct to JavaScript.

In general, JavaScript has had a fairly minimal standard library, but a trend in TC39 has been to make JS more of a "batteries-included" language, with a high-quality, built-in set of functionality available. For example, Temporal is replacing moment.js, and a number of small features, e.g., `Array.prototype.flat` and `Object.group` are replacing many lodash use cases. Benefits include smaller bundle sizes, improved stability and quality, less to learn when joining a new project, and a generally common vocabulary across JS developers.

Apart from making signals available out-of-the-box with the generic benefits that brings, we see the following technical benefits:

#### Interoperability

Each signal implementation has its own auto-tracking mechanism, to keep track of the sources encountered when evaluating a computed signal. This makes it hard to share models, components and libraries between different frameworks--they tend to come with a false coupling to their view engine (given that signals are usually implemented as part of JS frameworks).

A goal of this proposal is to fully decouple the reactive model from the rendering view, enabling developers to migrate to new rendering technologies without rewriting their non-UI code, or develop shared reactive models in JS to be deployed in different contexts. Unfortunately, due to versioning and duplication, it has turned out to be impractical to reach a strong level of sharing via JS-level libraries--built-ins offer a stronger sharing guarantee.

#### Performance/Memory usage

It is always a small potential performance boost to ship less code due to commonly used libraries being built-in, but implementations of signals are generally pretty small, so we don't expect this effect to be very large.

We suspect that native C++ implementations of signal-related datastructures and algorithms can be slightly more efficient than what is achievable in JS, by a constant factor. However, no algorithmic changes are anticipated vs. what would be present in a polyfill; engines are not expected to be magic here, and the reactivity algorithms themselves will be well-defined and unambiguous.

Many signal implementations require computed signals which are not in use anymore to be manually disposed, as the rest of the signal graph ends up holding them alive. Using WeakRef/FinalizationRegistry to enable automatic garbage collection of computed signals has a performance cost. In the context of the implementation of built-in  signals, we plan to investigate whether this cost can be reduced.

The champion group expects to develop various implementations of signals, and use these to investigate these performance possibilities.

#### DevTools

With existing JS-language signal libraries, it can be difficult to trace things like
* The callstack across a chain of computed signals, showing the causal chain for an error
* The reference graph among signals, when one depends on another -- important when debugging memory usage

Built-in signals enable JS runtimes and DevTools to potentially have improved support for inspecting signals, particularly for debugging or performance analysis, whether this is built into browsers or through a shared extension. Existing tools such as the element inspector, performance snapshot, and memory profilers could be updated to specifically highlight signals in their presentation of information.

#### HTML/DOM Integration

Current work in W3C and by browser implementors is seeking to bring native templating to HTML. Additionally, the W3C Web Components CG is exploring the possibility of extending Web Components to offer a fully declarative HTML API. To accomplish both of these goals, eventually a reactive primitive will be needed by HTML. Additionally, many ergonomic improvements to the DOM through integration of signals can be imagined and have been asked for by the community.

> Note, this integration would be a separate effort to come later, not part of this proposal itself.

## Design goals for Signals

It turns out that existing signal libraries are not all that different from each other, at their core. This proposal aims to build on their success by implmeneting the important qualities of many of those libraries.

### Core features

* A signal type which represents state, i.e. writable signal. This is a value that others can read.
* A computed/memo/derived signal type, which depends on others and is lazily calculated and cached.
    * Computation is lazy, meaning computed signals aren't calculated again by default when one of their dependencies changes, but rather only run if someone actually reads them.
    * Computation is "glitch-free", meaning no unnecessary calculations are ever performed. This implies that, when an application reads a computed signal, there is a topological sorting of the potentially dirty parts of the graph to run, to eliminate any duplicates.
    * Computation is cached, meaning that if, after the last time a dependency changes, no dependencies have changed, then the computed signal is *not* recalculated when accessed.
    * Custom comparisons are possible for computed signals as well as state signals, to note when further computed signals which depend on them should 
* Reactions to the condition where a computed signal has one of its dependencies (or nested dependencies) become "dirty" and change, meaning that the signal's value might be outdated.
    * This reaction is meant to schedule more signficant work to be performed later.
    * Effects are implemented in terms of these reactions, plus framework-level scheduling.
* Enable JS frameworks to do their own scheduling. No Promise-style built-in forced-on scheduling.
    * Synchronous reactions are needed to enable scheduling later work based on framework logic.
    * Writes are synchronous and immediately take effect (a framework which batches writes can do that on top).
* Ability to read signals *without* triggering dependencies to be recorded (`untrack`)
* Enable composition of different codebases which use signals/reactivity, e.g.,
    * Using multiple frameworks together as far as tracking/reactivity itself goes (though there are lots of other problems to solve)
    * Framework-independent reactive datastructures (e.g., recursively reactive store proxy, reactive Map and Set and Array, etc.)

### Soundness

* Discourage/prohibit certain kinds of read and write patterns during computed signals' evaluation
    * Soundness risk: A computed signal represents a pure computation at a particular point in time over other parts of the signal graph. If another signal is both read and later written to, then there is no coherent single point in time during which this computation can be said to be taking place. (TODO: Make this explanation more concrete/practical.)
    * Solution: During a computed signal's evaluation, no writing to a signal which has been read within this same evaluation
* Discourage/prohibit naive misuse of synchronous reactions
    * Soundness risk: it may expose "glitches" if improperly used: If rendering is done immediately when a signal is set, it may expose incomplete application state to the end user. Therefore, this feature should only be used to intelligently schedule work for later, once application logic is finished.
    * Solution: Disallow reading and writing any signal from within a synchronous reaction callback
        * TODO: Is this too strict to be practical?
* Discourage `untrack` and mark its unsound nature
    * Soundness risk: allows the creation of computed signals whose value depends on other signals, but which aren't updated when those signals change. It should be used when the untracked accesses will not change the result of the computation.
    * Solution: The API is marked "unsafe" in the name.

### Surface API

* Must be a solid base for multiple frameworks to implement their signals/reactivity mechanisms
    * Should be a good base for recursive store proxies, decorator-based class field reactivity, and both `.value` and `[state, setState]`-style APIs.
    * The semantics are able to express the valid patterns enabled by different frameworks. For example, it should be possible for these signals to be the basis of either immediately-reflected writes or writes which are batched and applied later.
    * Enable subclassing, so that frameworks can add their own methods and fields, including private fields. This is important to avoid the need for additional allocations at the framework level.
* It would be nice if this API is usable directly by JavaScript developers
    * If a feature matches with an ecosystem concept, using common vocabulary is good.
        * However, important to not literally shadow the exact same names!
    * Tension between "usability by JS devs" and "providing all the hooks to frameworks"
        * Idea: Provide all the hooks, but name them such that it's more clear that they aren't for ordinary JS programmers, and include errors when misused if possible.
        * TODO: Are `effect`/`startEffects`/`stopEffects` appropriate names?

### Memory management

* If possible: A computed signal should be garbage-collectable if nothing live is referencing it for possible future reads, even if it's linked into a broader graph which stays alive (e.g., by reading a state which remains live).
    * Note that most frameworks today require explicit disposal of computed signals if they have any reference to or from another signal graph which remains alive.
    * This ends up not being so bad when their lifetime is tied to the lifetime of a UI component, and effects need to be disposed of anyway.
    * If it is too expensive to execute with these semantics, then we should add explicit disposal (or "unlinking") of computed signals to the API below, which currently lacks it.
* A separate related goal: Minimize the number of allocations, e.g.,
    * to make a writable signal (avoid two separate closures + array)
    * to implement effects (avoid a closure for every single reaction)
    * In the API for observing signal changes, avoid creating additional temporary data structures
    * Solution: Class-based API enabling reuse of methods and fields defined in subclasses

## API sketch

An initial idea of a signal API is below. Note that this is just an early draft, and we anticipate changes over time. Let's start with the full `.d.ts` to get an idea of the overall shape, and then we'll discuss the details of what it all means.

```ts
// A cell of data which may change over time
class Signal<T> {
    // Get the value of the signal
    get(): T;
}

namespace Signal {
    // A read-write signal
    class State<T> extends Signal<T> {
        // Create a state signal starting with the value t
        constructor(t: T, options?: SignalOptions<T>);

        // Set the state signal value to t
        set(t: T): void;
    }
    
    // A signal which is a formula based on other signals
    class Computed<T> extends Signal<T> {
        // Create a signal which evaluates to the value of cb.call(the signal)
        constructor(cb: (this: Computed<T>) => T, options?: SignalOptions<T>);

        // Subscribe to options.effect for when a (recursive) source changes
        startEffects();

        // Unsubscribe to options.effect
        stopEffects();
    }

    namespace unsafe {
        // Run a callback with all tracking disabled (even for nested computed).
        function untrack(cb: () => T): T;
    }
}

type SignalComparison<T> = (this: Signal<T>, t: T, t2: T) => boolean;

interface SignalOptions<T> {
    // Custom comparison function between old and new value. Default: Object.is.
    equals?: SignalComparison<T>;
}

type SignalEffect<T> = (this: Signal<T>) => void;

interface ComputedOptions<T> extends SignalOptions<T> {
    // For effects: call synchronously when a (recursive) source changes
    effect?: SignalEffect<T>;
}
```

### How Signals work

A signal represents a cell of data which may change over time. Signals may be either "state" (just a value which is set manually) or "computed" (a formula based on other signals).

Computed signals work by automatically tracking the which other signals are read during their evaluation. When a computed is read, it checks whether any of its previously recorded dependencies have changed, and re-evaluates itself if so. When multiple computed signals are nested, all of the attribution of the tracking goes to the innermost one.

Computed signals are lazy, i.e., pull-based: they are only re-evaluated when they are accessed, even if one if their dependencies changed earlier.

The callback passed into computed signals should generally be "pure" in the sense of being a deterministic, side-effect-free function of the other signals which it accesses. At the same time, the timing of the callback being called is deterministic, allowing side effects to be used with care.

Signals feature prominent caching/memoization: both state and computed signals remember their current value, and only trigger recalculation of computed signals which reference them if they actually change. A repeated comparison of old vs new values isn't even needed--the comparison is made once when the source signal is reset/re-evaluated, and the signal mechanism keeps track of which things referencing that signal have updated based on the new value yet. Internally, this is generally represented through "graph coloring" as described in (Milo's blog post).

Computed signals track their dependencies dynamically--each time they are run, they may end up depending on different things, and that precise dependency set is kept fresh in the signal graph. This means that if you have a dependency needed on only one branch, and the previous calculation took the other branch, then a change to that temporarily unused value will not cause the computed signal to be recalculated, even when pulled.

Unlike JavaScript Promises, everything in signals runs synchronously:
- Setting a signal to a new value is synchronous, and this is immediately reflected when reading any computed signal which depends on it afterwards. There is no built-in batching of this mutation.
- Reading computed signals is synchronous--their value is always available.
- The `effect` callback, used to implement effects, as explained below, runs synchronously, during the `.set()` call which triggered it (but after graph coloring has completed).

Like Promises, signals can represent an error state: If a computed signal's callback throws, then that error is cached just like another value, and rethrown every time the signal is read.

### Understanding the Signal class

A `Signal` instance represents the capability to read a dynamically changing value whose updates are tracked over time. It also implicitly includes the capability to subscribe to the signal, implicitly through a tracked access from another computed signal.

The API here is designed to match the very rough ecosystem consensus among a large fraction of signal libraries:
- Access is through calls, e.g., `mySignal()` (both for computed and state). [Note: the other half of the signal world uses `.value`-style accessors, and we could switch to that if desired.]
- Names "state", "computed" and "signal" itself are chosen to match names used elsewhere.

The API is designed to reduce the number of allocations, to make signals suitable for embedding in JavaScript frameworks while reaching same or better performance than existing framework-customized signals. This implies:
- Writable state signals are a single object, which can be accessed and set in one. (See implications below in the "Capability separation" section.)
- Both state and computed signals are designed to be subclassable, to facilitiate frameworks' ability to add additional properties through public and private class fields (as well as methods for using that state).
- The `effect` callback is called on just one signal; no surrounding event or array-based datastructure surrounds it.
- Various callbacks (equals, effect, the computed callback) are called with the appropriate `this`, so that a new closure isn't needed per signal.

Some error conditions enforced by this API:
- Setting state after getting it within the same computed throws.
- It is an error to read a computed recursively.
- If a computed signal's callback throws, then subsequent accesses of the signal rethrow that cached error, until one of the dependencies changes and it is recalculated.

### Implementing effects

The `effect`/`startEffects`/`stopEffects` interface defined above gives the basis for implementing "effects": callbacks which are re-run when other signals change, purely for their side effect. A computed signal where an `effect` option was provided is called an "effect signal". The `effect` function used above in the initial example can be defined as follows:

```ts
// This function would usually live in a library/framework, not aplication code
// NOTE: This scheduling logic is too basic to be useful. Do not copy/paste.
function effect(cb) {
    // Create a new computed signal which evalutes to cb, which schedules
    // a read of itself on the microtask queue whenever one of its dependencies
    // might change
    let e = new Signal.Computed(cb, { effect() { queueMicrotask(this); } });
    // Run the effect the first time and collect the dependencies
    e.get();
    // Subscribe to future changes to call effect()
    e.startEffects();
    // Return a callback which can be used for cleaning the effect up
    return () => e.stopEffects();
}
```

The Signal API does not include any built-in function like `effect`. This is because effect scheduling is subtle and often ties into framework rendering cycles and other high-level framework-specific state or strategies which JS does not have access to.

Walking through the different operations used here: The `effect` callback passed into the computed signal constructor is the function that is called (during the range between a `.startEffects()` and `.stopEffects()` call) when the signal goes from a "clean" state (where we know the cache is initialized and valid) into a "checked" or "dirty" state (where the cache might or might not be valid because at least one of the states which this recursively depends on has been changed).

Calls to `effect` are ultimately triggered by a call to `.set()` on some state signal. This call is synchronous: it happens before `.set` returns. But there's no need to worry about this callback observing the signal graph in a half-processed state, because during a `effect` callback, no signal can be read or written, even in an `untrack` call. Because `effect` is called during `.set()`, it is interrupting another thread of logic, which might not be complete. To read or write signals from `effect`, schedule work to run later, e.g., by writing the signal down in a list to later be accessed, or with `queueMicrotask` as above.

To avoid mistakes, an error is thrown if `.startEffects()` or `.stopEffects()` are called when no `effect` callback had been passed to the computed signal constructor.

Note that it is perfectly possible to use `Signals` effectively without `effect`/`stopEffects`/`startEffects`, implementing effects by scheduling polling of computed signals, as Glimmer does. However, many frameworks have found that it is very often useful to have this scheduling logic run synchronously, so the Signals API includes it.

Both computed and state signals are garbage-collected like any JS values. When `.startEffects()` has not yet been called, a computed signal will hold alive the signals that it references, but not vice versa. However, after `.startEffects()` and before `.stopEffects()`, the computed signal *will* be held alive by any state signals which it (recursively) references, since this may trigger a future `.effect()` call for its side effect. For this reason, remember to call `.stopEffects()` when cleaning up effects.

> TODO: Investigate whether this GC guarantee is implementable without too much overhead

### An unsound escape hatch

`Signal.unsafe.untrack` is an escape hatch allowing reading signals *without* tracking those reads. This capability is unsafe because it allows the creation of computed signals whose value depends on other signals, but which aren't updated when those signals change. It should be used when the untracked accesses will not change the result of the computation.

The ability to perform untracked reads is necessary for efficiency because (*TODO*).

## Possible features under discussion

The above API provides a sort of Minimum Viable Product for signals which enables many important use cases. However, it actually wouldn't be viable to integrate into certain frameworks: it's missing some capabilities or performance properties which *some* frameworks take advantage of. It is still to be determined whether any extensions will be included as part of the initial Signal proposal; the signal collaborators have not come to agreement about any of these yet.

- For ergonomics: Various convenience methods (e.g., `Signal.State.prototype.update` as in Angular); brand checks
- For SSR: introspection of signal dependencies
- For incremental hydration/resumability: a signal which can be created with a fixed value and later transformed into a computed signal
- For handling asynchronicity: A "loading" state of signals which propagates automatically through the dependency graph, allowing a React Suspense-like system to be created
- For state management: A single object which represents state and processes for updating it, including lifecycle events which simplify ownership management

TODO: Link to more detailed resources investigating each of these
 
## Status and development plan

This proposal has not yet been presented at TC39, but the intention is to bring it to the committee as soon as it's in a good shape. In other words, it's at Stage 0.

The collaborators on the signal proposal want to be especially conservative in how we push this proposal forward, so that we don't land in the trap of getting something shipped which we end up regretting and not actually using. Our plan is to do the following extra tasks, not required by the TC39 process, to make sure that this proposal is on track:
- Before proposing for Stage 1: A concrete API is sketched out, and implemented in JS, with experiments showing that it can integrate reasonably well into some JS frameworks. Some medium-sized worked examples using the API directly are prepared.
- Before proposing for Stage 2: The proposed signal API has been integrated into a large number of JS frameworks that we consider somewhat representative, and some large applications work with this basis. The collaborators have a solid grasp on the space of possible extensions to the API, and have concluded which (if any) should be added into this proposal. A polyfill implementation exists which is solid, well-tested (in a shared test format), and competitive in terms of performance.
- Before proposing for Stage 3: There is an optimized native JS engine implementation of the signal API, allowing investigation of performance and memory management properties.

## Signal algorithms

This section describes each of the APIs exposed to JavaScript, in terms of the algorithms that they implement. This can be thought of as a proto-specification, and is included at this early point to nail down one possible set of semantics, while being very open to changes.

Some aspects of the algorithm:
- The order of reads of signals within a computed is significant, and is observable in the order that three types of callbacks (`effect`, `equals` and `new Signal.Computed`) are executed. This means that the sources of a computed signal must be stored as an ordered set, with duplicates removed.
- These three callbacks might all throw exceptions, and these exceptions are propagated in a predictable manner to the calling JS code. The exceptions do *not* halt execution of this algorithm or leave the graph in a half-processed state. For `effect`, that exception is sent to the `.set()` call which triggered it, using an AggregateError if multiple exceptions were thrown. The others are stored in the value of the signal, to be rethrown when read, and such a rethrowing signal can be marked `~clean~` just like any other with a normal value.
- The data structures are heavily circular, and some use of WeakRefs (for the sets representing sinks?) would be required to enable automatic garbage collection of non-effect computed signals when the other parts of the graph remain alive.

### Hidden global state 

Signal algorithms need to reference certain global state. In JavaScript specification terms, we refer to these as fields of the execution context.

- `computing`: A computed signal currently being reevaluated due to a `.get` call, or `undefined`. Initially `undefined`.
- `notifying`: Boolean denoting whether there is an `effect` callback currently executing. Initially `false`.

### The `Signal` class

`Signal` is an abstract class whose constructor throws. All signals are either state signals or computed signals. They have a common structure and `get` algorithm, described below.

#### Common hidden fields

- `value`: The previous cached value of the signal, or `~uninitialized~` for a never-read computed signal. The value may be an exception which gets rethrown when the value is read.
- `state`: May be `~clean~`, `~checked~` or `~dirty~`. State signals are always `~clean~`.
- `sources`: An ordered set of signals which this signal depends on. For state signals, this will be the empty set.
- `sinks`: An ordered set of signals which depend on this signal. These will all be computed signals.
- `equals`: The equals method provided in the options, defaulting to `Object.is`.

#### Method: `Signal.prototype.get`

1. If the current execution context is `notifying` or if this signal is `recalculating`, throw an exception.
1. If `computing` is not `undefined`, add this signal to `computing`'s `sources` set, and add `computing` to this signal's `sinks` set.
1. If this signal's state is `~dirty~` or `~checked~`: Repeat the following steps until this signal is `~clean~`:
    1. Recurse up via `sources` to find the deepest, left-most (i.e. earliest observed) recursive source which is marked `~dirty~` (cutting off search when hitting a `~clean~` signal, and including this signal as the last thing to search).
    1. Perform the "recalculate dirty computed signal" algorithm on that signal.
1. At this point, this signal's state will be `~clean~`, and no recursive sources will be `~dirty~` or `~checked~`. Return the signal's `value`. If the value is an exception, rethrow that exception.

##### Algorithm: recalculate dirty computed signal

1. Clear out this signal's `sources` set, and remove it from those sources' `sinks` sets.
1. Save the previous `computing` value and set `computing` to this signal.
1. Set this signal's `recalculating` to true.
1. Run this computed signal's callback, using this signal as the this value. Save the return value, and if the callback threw an exception, store that for rethrowing.
1. Set this signal's `recalculating` to false.
1. Restore the previous `computing` value.
1. Apply the "set signal value" algorithm to the callback's return value.
1. If that algorithm returned `~dirty~`: mark all sinks of this signal as `~dirty~` (previously, the sinks may have been a mix of checked and dirty).
1. Otherwise, that algorithm returned `~clean~`: In this case, for each `~checked~` sink of this signal, if all of that signal's sources are now clean, then mark that signal as `~clean~` as well. Apply this cleanup step to further sinks recursively, to any newly clean signals which have checked sinks.

### The `Signal.State` class

The constructor sets `value` to its parameter, `equals` based on options, and `state` to `~clean~`.

#### Method: `Signal.State.prototype.set`

1. If the current execution context is `notifying` or if this signal is in the execution context's `computing` signal's `sources` list [TODO: what about nested computeds? how does untrack relate to this?], throw an exception.
1. Run the "set signal value" algorithm with this signal and the first parameter for the value.
1. If that algorithm returned `~clean~`, then return undefined.
1. Set the `state` of all `sinks` of this signal to `~dirty~` if they weren't already dirty.
1. Set the `state` of all of the sinks' dependencies (recursively) to `~checked~` if they weren't already dirty or checked (that is, leave dirty markings in place).
1. For each signal newly set to `~dirty~` or `~checked~`, if they have `necessary` set to true, then in depth-first order,
    1. Set `notifying` to true while calling their `effect` method (saving aside any exception thrown, but ignoring the return value of `effect`), and then restore `notifying` to false.
1. If any exception was thrown from the `effect` callbacks, propagate it to the caller after all `effect` callbacks have run. If there are multiple exceptions, then package them up together into an AggregateError and throw that.
1. Return undefined.

##### Set signal value algorithm

1. If this algorithm was passed a value (as opposed to an exception for rethrowing, from the recalculate dirty computed signal algorithm):
    1. Call this signal's `equals` function, given this signal as the `this` value, and passing as parameters the current `value` and the new value. If an exception is thrown, save that exception (for rethrowing when read) as the value of the signal and continue as if the callback had returned false.
    1. If that returned true, return `~clean~`.
1. Set the `value` of this signal to the parameter.
1. Return `~dirty~`

### The `Signal.Computed` class

The constructor sets
- `callback` to its first parameter
- `equals` and `effect` based on options
- `necessary` and `recalculating` to false
- `state` to `~dirty~`
- `value` to `~uninitialized~`

#### Additional hidden fields

- `callback`: The callback which is called to get the computed signal's value. Set to the first parameter passed to the constructor.
- `effect`: The callback called when the signal goes into a `~checked~` state, or undefined.
- `necessary`: Boolean indicating whether this signal is currently started. Initialy set to false on construction.
- `recalculating`: A boolean indicating whether this signal is being recomputed due to a `.get()`, initially false.

#### Method: `Signal.Computed.prototype.startEffects()`

1. If this signal's `effect` is undefined, or if `necessary` is already true, throw an exception.
1. Set `necessary` to true.
1. If this signal's `state` is `~dirty~` or `~checked~`, then call the effect function (propagating any exception, but ignoring the return value).
1. Return undefined

#### Method: `Signal.Computed.prototype.stopEffects()`

1. If this signal's `effect` is undefined, or if `necessary` is already false, throw an exception.
1. Set `necessary` to false.
1. Return undefined

### `Signal.unsafe.untrack(cb)`

1. Let c be the execution context's current `computing` state.
1. Set `computing` to undefined.
1. Call `cb`.
1. Restore `computing` to c (even if `cb` threw an exception).
1. Return the return value of `cb` (rethrowing any exception).

## FAQ
**Q**: Is it a good idea to enable application state which is distributed, rather than at the component level or at the top level of the application?

**A**: This flexibility is what allows signals to be used to implement complex reactive data structures and models which can be composed--quite useful things when sharing logic in UI development. At the same time, we expect that storing state at either the component or application level will be quite a common pattern with signals, as it is without 

**Q**: Isn't it a little soon to be standardizing something related to signals, when they just started to be the hot new thing in 2022? Shouldn't we give them more time to evolve and stabilize?

**A**: The current state of signals in web frameworks is the result of more than 10 years of continuous development. As investment steps up, as it has in recent years, almost all of the web frameworks are approaching a very similar core model of signals. This proposal is the result of a shared design exercise between a large number of current leaders in web frameworks, and it will not be pushed forward to standardization without the validation of that group of domain experts in various contexts.

#### How are signals used?

**Q**: Can built-in signals even be used by frameworks, given their tight integration with rendering and ownership?

**A**: The parts which are more framework-specific tend to be in the area of effects, scheduling, and ownership/disposal, which this proposal does not attempt to solve. For performance reasons, some frameworks may need some of the extension APIs to integrate well, and this is something we are investigating. Our first priority with prototyping standards-track signals is to validate that they can sit "underneath" existing frameworks compatibly and with good performance.

**Q**: Is the Signal API meant to be used directly by application developers, or wrapped by frameworks?

**A**: The design is intended to fit either. The core functionality (state and computed constructors, reading and writing signals) is designed to be similar to existing popular signal implementations. On the other hand, features like `untrack` and `effect` are more error-prone and should probably be left to libraries and frameworks. Frameworks provide many other important features, such as managing ownership and disposal, and scheduling rendering to DOM--this proposal doesn't attempt to solve those problems. Some frameworks may put a different skin around `Signal.State` and `Signal.Computed` for ergonomic reasons.

**Q**: Do I have to tear down signals related to a widget when that widget is destroyed? What is the API for that?

**A**: The relevant teardown operation here is `Signal.computed.prototype.stopEffects`, for signals with a `effect` callback. "Pure" computed signals without a `effect` callback do not need to be cleaned up.

**Q**: Do signals work with VDOM, or directly with the underlying HTML DOM?

**A**: Yes! Signals are independent of rendering technology. Existing JavaScript frameworks which use signal-like constructs integrate with VDOM (e.g., Vue), the native DOM (e.g., Solid) and a combination (e.g., Preact). The same will be possible with built-in signals.

**Q**: Is it going to be ergonomic to use signals in the context of class-based frameworks like Angular and Lit? What about compiler-based frameworks like Svelte?

**A**: Class fields can be made signal-based with a simple accessor decorator, as shown in (link to the below content). Signals are very closely aligned to Svelte 5's Runes--it is simple for a compiler to transform runes to the signal API defined here, and in fact this is what Svelte 5 does internally (but with its own signals library).

**Q**: Do signals work with SSR? Hydration? Resumability?

**A**: Yes. Qwik uses signals to good effect with both of these properties, and other frameworks have other well-developed approaches to hydration with signals with different tradeoffs. One possible extension of signals to support SSR and resumability adds introspection and incremental construction of the signal graph; we'll be researching whether this capability is necessary to include in the proposal to make SSR work in practice.

**Q**: Do signals work with one-way data flow like React does? Can they be integrated with state management systems like Redux?

**A**: Yes and yes. A graph of state and computed signals is acyclic by construction (though it is still possible to schedule effects to write to state, in a similar antipattern to writing to state from `useEffect` in React). Redux in particular has already been integrated into signal-based frameworks, e.g., (link to some Redux/Solid integration that probably exists)

#### How do signals work?

**Q**: Are signals push-based or pull-based?

**A**: Evaluation of computed signals is pull-based: computed signals are only evaluated when they are read, even if the underlying state changed. At the same time, changing state eagerly pushes cache invalidation to computed signals which depend on it, potentially triggering a `effect` callback. So signals may be thought of as a "push-pull" construction.

**Q**: Do signals introduce nondeterminism into JavaScript execution?

**A**: No. For one, all signal operations have well-defined semantics and ordering, and will not differ among conformant implementations. At a higher level, signals follow a certain set of invariants, with respect to which they are "sound". A computed signal always observes the signal graph in a consistent state, and its execution is not interrupted by other signal-mutating code (except for things it calls itself). See the description above.

**Q**: When I write to a state signal, when is the update to the computed signal scheduled?

**A**: It isn't scheduled! The computed signal will recalculate itself the next time someone reads it. Synchronously, a `effect` callback may be called, enabling frameworks to schedule a read at the time that they find appropriate.

**Q**: When do writes to state signals take effect? Immediately, or are they batched?

**A**: Writes to state signals are reflected immediately--the next time a computed signal which depends on the state signal is read, it will recalculate itself if needed, even if in the immediately following line of code. However, the laziness inherent in this mechanism (that computed signals are only computed when read) means that, in practice, the calculations may happen in a batched way.

**Q**: What does it mean for signals to enable "glitch-free" execution?

**A**: Earlier push-based models for reactivity faced an issue of redundant computation: If an update to a state signal causes the computed signal to eagerly run, ultimately this may push an update to the UI. But this write to the UI may be premature, if there was going to be another change to the originating state signal before the next frame. Sometimes, inaccurate intermediate values were even shown to end-users due to such glitches. Signals avoid this dynamic by being pull-based, rather than push-based: At the time the framework schedules the rendering of the UI, it will pull the appropriate updates, avoiding wasted work both in computation as well as in writing to the DOM.

**Q**: What does it mean for signals to be "lossy"?

**A**: This is the flipside of glitch-free execution: Signals represent a cell of data--just the immediate current value (which may change), not a stream of data over time. So, if you write to a state signal twice in a row, without doing anything else, the first write is "lost" and never seen by any computed signals or effects. This is understood to be a feature rather than a bug--other constructs (e.g., async iterables, observables) are more appropriate for streams.

**Q**: Will native signals be faster than existing JS signal implementations?

**A**: We hope so (by a small constant factor), but this remains to be proven in code. JS engines aren't magic, and will ultimately need to implement the same kinds of algorithms as JS implementations of signals. See above section about performance.

#### Why are signals designed this way?

**Q**: Why doesn't this proposal include an `effect()` function, when effects are necessary for any practical usage of signals?

**A**: Effects inherently tie into scheduling and disposal, which are managed by frameworks and outside the scope of this proposal. Instead, this proposal includes the basis for implementing effects through the `effect`/`startEffects`/`stopEffects` API.

**Q**: Why are subscriptions automatic rather than providing a manual interface?

**A**: Experience has shown that manual subscription interfaces for reactivity are unergonomic and error-prone. Automatic tracking is a core feature of signals.

**Q**: Why does the `effect` callback run synchronously, rather than scheduled in a microtask?

**A**: Because `effect` cannot read or write signals, there is no unsoundness brought on by calling it synchronously. A typical `effect` callback will add a signal to an Array to be read later, or mark a bit somewhere. It is unneccessary and expensive to make a separate microtask for these sorts of actions.

**Q**: This API is missing some nice things that my favorite framework provides, which makes it easier to program with signals. Can that be added to the standard too?

**A**: Maybe. Various extensions are still under consideration. Please file an issue to raise discussion on any missing feature you find to be important.

**Q**: Can this API be reduced in size or complexity?

**A**: It's definitely a goal to keep this API minimal, and we've tried to do so with what's presented above. If you have ideas for more things that can be removed, please file an issue to discuss.

#### How are signals being standardized?

**Q**: Shouldn't we start standardization work in this area with a more primitive concept, such as observables?

**A**: Observables may be a good idea for some things, but they don't solve the problems that signals aim to solve. As described above, observables or other publish/subscribe mechanisms are not a complete solution to many types of UI programming, due to too much error-prone configuration work for developers, and wasted work due to lack of laziness, among other issues.

**Q**: Why are signals being proposed in TC39 rather than DOM, given that most applications of it are web-based?

**A**: Some coauthors of this proposal are interested in non-web UI environments as a goal, but these days, either venue may be suitable for that, as web APIs are being more frequently implemented outside the web. Ultimately, signals don't need to depend on any DOM APIs, so either way works. If someone has a strong reason for this group to switch, please let us know in an issue. For now, all contributors have signed the TC39 intellectual property agreements, and the plan is to present this to TC39.

**Q**: How long is it going to take until I can use standard signals?

**A**: A polyfill should be available within weeks, but is initially not expected to be stable, as this API evolves during its review process. In some months or a year, a high-quality, high-performance stable polyfill should be usable, but this will still be subject to committee revisions and not yet standard. Following the typical trajectory of a TC39 proposal, it is expected to take at least 2-3 years at an absolute minimum for signals to be natively available across all browsers going back a few versions, such that polyfills are not needed.

**Q**: How will we prevent standardizing the wrong kind of signals too soon, just like {{JS/web feature that you don't like}}?

**A**: The authors of this proposal plan to go the extra mile with prototyping and proving things out prior to requesting stage advancement at TC39. See "Status and development plan" above. If you see gaps in this plan or opportunities for improvement, please file an issue explaining.