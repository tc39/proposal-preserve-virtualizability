# Preserve Host Virtualizability

Champions: Mark S. Miller, ...

Prohibit hosts from adding new non-deletable extensions, or accidentally breaking virtualizability by other means.

## Background: Language Virtualizability

Javascript is one of the few languages to make a strong distinction between the language and the hosting environment. The EcmaScript standard language is almost completely free of I/O. Different hosts---such as browsers, servers, embedded devices, blockchains, and others---reify I/O abilities as *host-objects*. These host-objects are initially provided only on the global object, i.e., in the global scope, bound to names such as `document`, `XMLHttpRequest`, `process`, or `require`.

The EcmaScript language is thus analogous to the user-mode instructions of an instruction set. It is pleasantly Turing Universal, i.e., EcmaScript code can compute any computable function, with powerful abstraction mechanisms for expressing and organizing computation. Each host is analogous to a set of I/O devices and system-mode instructions. The APIs of the host objects are analogous to an OS kernel's system-call interface. When an EcmaScript object calls a host object, it is effectively making a system call. All EcmaScript access to host objects starts only from looking up names on the global object. *Ideally*, EcmaScript startup code can replace these global name bindings to emulate any other possible host to any post-startup EcmaScript code in the same environment.

Following [Popek and Goldberg's seminal work](https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements), we say an EcmaScript system is *fully virtualizable* when EcmaScript startup code can easily and perfectly emulate any possible host to any post-startup EcmaScript code, without rewriting the post-startup code. Extending our analogy, such startup code is like the first bootstap code that sees the real system-mode architecture and devices. If it installs a virtual machine---an emulation of a different system-mode architecture and devices---the next code may be startup code for that other architecture, unaware that it is not running first and not running on the "actual" architecture.

Turing universality guarantees that any universal machine can perfectly emulate any other machine; so how can "virtualizability" be a crisp distinction? Again following Popek and Goldberg, this is why *without rewriting* is the important litmus test. The x86 is not a *fully virtualizable* architecture. Nevertheless, QEMU and VMWare can perfectly emulate an x86 on an x86, but only by complicated, expensive, and fragile means equivalent to interpretation or rewriting. Because EcmaScript is so expensive to parse and so hard to parse accurately, we have an even greater need to uphold full virtualizability.

## Lessons from TC39 History

In current actual EcmaScript/host deployments, there are few violations of virtualizability in practice, all are accidental, and none are fatal. However, the EcmaScript spec currently allows hosts to irreparably damage virtualizability in ways that would serve no useful purpose. Our history shows that we must tighten the spec, to avoid losing virtualizability simply due to unfortunate accidents.

Before EcmaScript 5, DOM objects were very magical. They had all kinds of bizarre behavior which objects coded in EcmaScript could not emulate. For example, assigning to a property of a DOM object could trigger behavior. To close this gap, EcmaScript 5 standardized *accessor properties*, i.e., properties with getters and setters. DOM objects could have non-assignable or non-deleteable properties, so EcmaScript 5 introduced [explicit property descriptors](https://ai.google/research/pubs/pub37741) with control over writability and configurability. However, while tc39 was closing this gap, w3c/whatwg was simultaneously widening it---introducing more host object behaviors, like [local storage](https://html.spec.whatwg.org/multipage/webstorage.html#dom-localstorage), that could not be emulated even by EcmaScript 5 objects.

To repair this mis-coordination between language and host, we invented new constraints. On the EcmaScript side, we codified a set of [object invariants](https://www.ecma-international.org/ecma-262/#sec-invariants-of-the-essential-internal-methods) that no object, EcmaScript or host, may violate. We provided a new abstraction mechanism, [direct proxies](https://ai.google/research/pubs/pub40736), that could emulate almost any host object that obeys these invariants, and that could not itself violate these invariants. On the w3c/whatwg side, we designed a new WebIDL that can only specify behaviors that proxies can emulate. Once repaired, this particular virtualizability break has stayed repaired.

# Remaining Threats to Virtualizability

Generalizing from these historical lessons, in what other ways does the EcmaScript spec allow the host too much freedom, in ways that could accidentally destroy virtualizability? How should we tighten the spec to preclude these hazards?

## Undeletable Extensions

The EcmaScript spec allows hosts to add new properties to any object, as well as the global object, containing any possible property configuration. For example, a host may add non-configurable, non-writable `peek` and `poke` data properties to `Array.prototype` whose values are builtin functions for directly reading or writing any physical memory address. Were these functions to remain available, memory safety would be lost. However, startup code cannot eliminate them without rewriting subsequent code. Evaluating `[]` creates an array inheriting from the original `Array.prototype`. Hence, the original `Array.prototype` is *undeniable*, it remains available even if another object is bound to the name `Array` or to the `prototype` property of `Array`.

The startup code of the SES-shim employs a [whitelist](https://github.com/Agoric/SES/blob/master/src/bundle/whitelist.js) to delete any properties that it doesn't recognize, which would eliminate these hypothetical `peek` and `poke` properties, if it could delete them. (If the whitelist mechanism fails to delete these properties, then the SES-shim halts, failing safe, but still failing.) We propose that any such additional properties added by the host to any object, *including the global object*, must be deletable. This is stronger than saying that the property must be configurable. Since a non-configurable property cannot be deleted, a deletable property must be configurable. But a configurable property may still not actually be deletable. Instead, we mandate that for each additional host provided property, an attempt to delete it with `delete` or `Reflect.delete` must successfully eliminate the property.

A recent example is `RegExp.leftContext`, which was a non-standard host addition that provided a security-breaking global communications channel. On most browsers, this property was non-deletable. To fix this special case, we proposed [Legacy RegExp features in JavaScript](https://github.com/tc39/proposal-regexp-legacy-features) to both codify these properties as *normative optional*, so that a conforming implementation could omit them, and *deletable*, so startup code could reliably eliminate them anywhere they were not omitted. (As of this writing, `RegExp.leftContext` is still non-deletable on Firefox, which is not fatal only because [the RegExp constructor is deniable](https://github.com/Agoric/SES/blob/master/src/bundle/tame-regexp.js).) [Error Stacks](https://github.com/tc39/proposal-error-stacks) is a similar special case repair for the encapsulation breaking, currently-non-standard, `error.stack` property.

Rather than a never ending stream of such proposals to fix such special cases as they arise, this proposal seeks to solve the general problem, precluding the need for more such special cases. Had this proposal already been standard, no non-standard non-deletable `RegExp.leftContext` would have remained, and no special case fix would have been needed.

## Undeletable Globals

In the post EcmaScript 5 era, dangerous non-standard non-deletable properties on normal objects, such as `RegExp.leftContext`, have been mercifully rare. However, on the browser, there remain dangerous non-standard non-deletable properties on the global object. Until recently, we all thought these were unrepairable. Hence, all efforts to support virtualization---such as [the 8 magic lines of code](https://www.youtube.com/watch?v=mSNxsn0pK74&list=PLzDw4TTug5O0ywHrOz4VevVTYr6Kj_KtW) at the heart of the [Realms](https://github.com/Agoric/realms-shim), [SES](https://github.com/Agoric/SES), and [Evaluator](https://github.com/Agoric/evaluator-shim) shims---start by hiding the global object itself. However, recently, a [combination of techiques discovered by Caridy Patino of Salesforce and JF Paradis of Agoric](https://www.youtube.com/watch?v=TaPot2OyXHU&list=PLzDw4TTug5O1jzKodRDp3qec8zl88oxGd), show that the browser global object can be *bleached* and [made harmless](https://github.com/caridy/secure-javascript-environment).

The result is that an iframe's global object is disconnected from its browser context, disabling much host functionality, without damaging the EcmaScript language within that frame. The non-deletable properties of the global are not deleted, and the host objects they provide access to remain accessible. Due to the chains of such non-deletable properties, there are altogether six such undeniable host objects. All the behavior of these six undeniable host objects is brought about only by applying the builtin methods that recognize these host objects. None of these builtin methods are undeniable. Bleaching removes everything removable, ***including all these builtin methods***, leaving the undeniable host objects safely useless. We got lucky. The browser was only barely not too broken to fix, and it took us until now to figure that out.

This combination---of diconnecting the frame global from its context, and of bleaching everything reachable from that global---does not give us perfect virtualizability. Due to the detectable presence of those six remaining inert host objects, at those known property names, post-startup code can still sense that it seems to have been started on a browser, despite best efforts by startup code to emulate another host. However, that is only an abstraction leakage of information about the platform. It does not compromise in any way the ability to virtualize the *abilities* and *inabiities* of a different host. Because we likely cannot get agreement to make these properties deletable, and because we now understand this remaining violation to be non-fatal, this proposal grandfathers it alone as a special case. We explicitly prohibit any violations outside this grandfathered-in special case.

## Undetectable Extensions

## Proliferating Host Hooks

## Extra Behavior of Standard Properties

## Non-shimmable Primordials

## Hidden Primordial State

## Module Loading Behavior

## Non-shimmable Builtin Modules

## Hidden Builin Module Instance State
