# type-im
> Only the README.md is here. I publish the code on Codeberg. [view →](https://codeberg.org/slnknrr/typeim_lib.js-for-pm_npm).

**The honest `typeof`. A predicate returns the evidence, not a verdict.**

Every way JavaScript has of saying what a value *is* lies in its own way. `type-im` asks all of them and tells you what each said — as bits in one small integer — so "is it a Map?" and "who says so?" are the same call.

```js
import t from '@slnknrr/type-im';

t.ismap(new Map());                          // 63 — every witness: name, tag, methods, internal slot, this realm's chain and constructor
t.ismap(mapFromAnIframe);                    // 15 — name, tag, methods, slot; not this realm's chain: instanceof would say false, and lie
t.ismap({ [Symbol.toStringTag]: 'Map' });    // 2  — the tag claims; nothing else does: a costume
t.ismap(new (class Cache extends Map {})()); // 30 — tag, methods, slot, chain; not the name, not the exact constructor: a subclass
t.ismap(v) & t.ismap.BRAND;                  // the one question most people mean: does it have [[MapData]]?

t.istyped(Buffer.alloc(8));                  // 28 — a Uint8Array by slot, tag and chain, whose name says nothing
t.telem(new Float16Array(1));                // 0x310 — Float, 16 bits: from the internal slot, no instanceof chain, no update for the new kind
t.tfn(async () => {});                       // 9 — ASYNC | ARROW, from the intrinsic source text
```

---

## Who lies, and how

| witness | what it reads | where it lies |
|---|---|---|
| `typeof` | the primitive type | `null` is `'object'`; a class is `'function'`; every object is `'object'` |
| `instanceof` | this realm's prototype chain | **false** for a genuine value from another realm (iframe, `vm`, worker); **true** for `Object.create(Map.prototype)`, which has no `[[MapData]]` |
| `Symbol.toStringTag` / `Object.prototype.toString` | a public property | anyone can set it; `{ [Symbol.toStringTag]: 'Map' }` reads `[object Map]` |
| `constructor.name` | a string on a function | gone under minification; `Buffer` for a `Uint8Array`; renamed at will |
| duck typing | the methods | fooled by construction — and yet it is what `await` and `for…of` actually use |
| an internal slot | `Array.isArray`, `Map.prototype.has.call`, the `%TypedArray%` tag getter, `Date.prototype.getTime.call`, `Error.isError` … | it doesn't. It also cannot tell a subclass, a foreign realm, or *which* typed array without more work — and for `Promise`, pure JavaScript cannot reach it without registering a reaction |

None of these is wrong often. Each is wrong *somewhere*. A boolean built on one of them inherits that blind spot silently; a boolean built on several has to pick a winner and hide the disagreement. `type-im` does neither: **every witness is a bit, zero means nobody spoke, and you read the bits you trust.**

---

## What it is

- **A witness per bit, a predicate per kind.** 27 predicates for the kinds `typeof` cannot tell apart: containers, typed arrays, buffers, promises, dates, regexps, errors, wrappers, plain objects, every kind of function, every kind of iterator.
- **The hack, on purpose.** The cheapest witnesses are textual — a constructor's name, a tag, a function's source text — read by `charCodeAt`, never a regex. They cost a few comparisons and no allocation, they work across realms because a name is a name everywhere, and they survive the future: a parser that knows the *shape* of a typed array's name recognized `Float16Array` before it shipped and will recognize `Int128Array` the day it does. A textual witness has a known failure mode — which is exactly why it is a bit and not the verdict.
- **The truth where there is one.** Where the platform has an internal-slot check, it is a bit too, and the top one: `Array.isArray`, the `%TypedArray%` `@@toStringTag` getter (it reads `[[TypedArrayName]]`), `Map.prototype.has.call` (it throws on anything without `[[MapData]]`), `Date.prototype.getTime.call`, the `byteLength` getters, the `valueOf` brands, `Error.isError`, `Reflect.construct(String, [], f)` for `[[Construct]]` without ever calling `f`.
- **Accessors of kind.** The textual witnesses know more than yes/no: which typed array (kind and bit width), which error, which kind of function, which wrapper. `telem`, `terr`, `tfn`, `tbox`, `tname`, `ttag` hand that out.
- **Nothing throws, nothing is mutated, no job is enqueued.** A revoked Proxy, a throwing getter, a null-prototype object: 0, never an exception. No predicate calls the value or registers a promise reaction.
- **Pure ESM, dependency-free, sync.** One file, ~800 lines, Node ≥ 20 — and any browser: the host's `util.types` is used where it exists and never required.

## What it is NOT

- **Not a schema validator.** No shapes, no coercion, no error messages about your payload. This is about what a value *is*, not whether it is *right*.
- **Not for primitives.** `typeof` is honest about `'string'`, `'number'`, `'bigint'`, `'symbol'`, `'boolean'`, `'undefined'`; `type-im` starts where it stops. (A *wrapper object*, `new Number(1)`, is another matter: see `isbox`.)
- **Not a `Proxy` detector.** The specification makes a Proxy indistinguishable from its target by design; a Proxy of an array is an array to every witness here. (Node's `util.types.isProxy` exists; it is not JavaScript.)
- **Not faster than one native check.** `Array.isArray` is one check; `t.isarr` is five. See [Performance](#performance) for where the evidence pays for itself.

---

## Install

```sh
npm install @slnknrr/type-im
```

```js
import t from '@slnknrr/type-im';                  // ready-to-use singleton — no `new`
import { typeim, _typeim } from '@slnknrr/type-im'; // the class (instanceof, the constants on its prototype) and the factory
const { ismap, istyped, telem } = t;               // every method detaches: list.filter(t.isprom) works
```

Requirements: **Node ≥ 20**, ESM only (`"type": "module"` or `import`). In Node, Deno and Bun the host's `util.types` (via `process.getBuiltinModule`) supplies the internal-slot checks that would otherwise need a thrown exception, and the only one pure JavaScript cannot express at all (`Promise`). In a browser everything works the same, with one stated blind spot ([below](#the-brand-where-it-comes-from)).

---

## Core conventions

These are load-bearing. Learn them once; they apply everywhere.

**A predicate returns bits; `0` means no witness said so.** So `if (t.ismap(v))` reads as usual — "someone thinks it's a Map" — and `t.ismap(v) & t.ismap.BRAND` asks the sharper question.

**Every predicate has its own bits, in increasing strength, as constants on the method.** The recurring witnesses keep their names across predicates; their positions and presence follow what each kind can actually witness:

| name | the witness |
|---|---|
| `NAME` | `constructor.name` is the kind's name (or has its shape) |
| `TAG` | `Symbol.toStringTag` — as a *property* — says so. Real Maps, Promises, typed arrays and ArrayBuffers have one; real Arrays, Dates, RegExps, Errors and wrappers do not, so for those `TAG` lights only on a costume |
| `DUCK` | the protocol the language itself would use is there: `then`, `next`, `get/set/has/delete/size`, `getTime` … |
| `BRAND` | an internal-slot check says so — unforgeable, cross-realm |
| `PROTO` | this realm's prototype is in the chain (what `instanceof` checks) |
| `OWN` | `constructor` is this realm's constructor exactly |

**Read the bits you trust:**

```js
const x = t.ismap(v), M = t.ismap;
x & M.BRAND                                   // a real Map, from anywhere
(x & M.BRAND) && !(x & M.PROTO)               // a real Map from another realm — instanceof would say no
(x & (M.BRAND | M.PROTO | M.OWN)) === (M.BRAND | M.PROTO) // a subclass
(x & M.PROTO) && !(x & M.BRAND)               // a chain costume: Object.create(Map.prototype) — instanceof would say yes
(x & (M.NAME | M.TAG)) && !(x & M.BRAND)      // named or tagged like one, isn't one
(x & M.DUCK) && !(x & M.BRAND)                // a duck: a thenable, a polyfill — usable as one
x === 63                                      // the plain, local, genuine article
```

**Realms.** `PROTO` and `OWN` are about *this* realm — the one that imported the module. A genuine value from another realm has `BRAND` (and usually `NAME`, `TAG`, `DUCK`) but neither. That is the precise situation `Array.isArray` was invented for and `instanceof` gets wrong.

**Nothing throws.** A witness that cannot be consulted contributes `0`. A revoked Proxy throws on `Array.isArray` itself; here it is `0` across the board.

**No side effects.** The only user code a predicate can reach is a getter on the value (`constructor`, `Symbol.toStringTag`, `then`, `size`) — the same one any property read would reach. No predicate calls the value, constructs it, or registers a promise reaction.

### The brand: where it comes from

Most slots have a pure-JavaScript check with no side effects, and it is used everywhere. Some of those checks *throw* on a missing slot (`Map.prototype.has.call`, the `byteLength` and `source` getters, `getTime`, the `valueOf` brands); an exception costs microseconds, and in a classification the negative case is the common one. So:

- **With a host** (Node, Deno, Bun — anything with `process.getBuiltinModule('node:util').types`): the host's check answers, never throws, always exact.
- **Without one** (a browser): the throwing probe is asked only when the value is *plausible* — a cheaper witness already spoke, or the object has no prototype at all (the one way to silence every cheap witness while keeping the slot). The stated blind spot: a Map whose prototype was swapped for a plain object reads `0` in a browser and `8` in Node.
- **`Promise`** has no pure-JavaScript slot check without a side effect: `then.call` registers a reaction (and on a settled promise enqueues a job). `isprom`'s `BRAND` is therefore the host's, and `0` where there is no host — where `TAG | PROTO | OWN` is the strongest reading, which is what `instanceof Promise` gives you anyway.

---

## Performance

`type-im` is fast where the usual idiom is a chain, an allocation or a regex — and where the usual idiom is simply wrong. It is not faster than one native check, and says so.

Representative run (`npm run bench`, Node 24, one machine — your numbers will differ; the **shape** won't):

| Case | type-im | the usual way | speedup |
|---|--:|---|--:|
| kind of a function | 3.0M/s | toString tag + `Reflect.construct` + source regexes | **~37×** |
| plain object? | 9.1M/s | lodash-style `isPlainObject` (toString + `Function` source) | **~8×** |
| which typed array? (kind + width) | 12M/s | 11 × `instanceof`, then `BYTES_PER_ELEMENT` | **~5×** |
| is it a typed array, and which? | 3.5M/s | `Object.prototype.toString` + a regex on the tag | **~1.5×** |
| classify a mixed list of 12 values | 188k/s | 5 × `Object.prototype.toString` per value | 0.27× |
| is it an array? | 11M/s | `Array.isArray` | 0.24× |
| is it a Map? | 5.4M/s | `util.types.isMap` | 0.14× |

The last three are the honest cost of evidence: one native check against five or six witnesses. The first four are the usual idiom doing work it did not need to do — building a string, walking eleven constructors, compiling a regex, throwing an exception to find out whether a function constructs.

---

## API

27 predicates and 6 accessors. Every predicate takes any value and returns a number. Its bits are listed next to it in increasing strength and live on the method: `t.isprom.DUCK`.

### 1 · Containers
| Predicate | Bits | Notes |
|---|---|---|
| `isarr(v)` | 1 NAME · 2 TAG · 4 BRAND · 8 PROTO · 16 OWN | BRAND is `Array.isArray`. A real array has no tag property: `TAG` is a costume detector. `[]` → 29 |
| `istyped(v)` | 1 SUFFIX · 2 SHAPE · 4 TAG · 8 BRAND · 16 PROTO · 32 OWN | SUFFIX: the name ends with `Array`. SHAPE: the name has a typed array's form — `(Int\|Uint\|Float\|BigInt\|BigUint)<2ⁿ ≥ 8>[Clamped]Array`. TAG: the tag has that form (for a real one the tag *is* the slot). BRAND: the `%TypedArray%` getter. OWN: this realm's constructor of that very kind. `Buffer` → 28; `class Float32Array {}` → 3 |
| `isbuf(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN · 64 SHARED | `ArrayBuffer` or `SharedArrayBuffer`; SHARED says which. DUCK: `byteLength`, `slice`, and not a view |
| `isview(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN | a `DataView`. DUCK: `getUint8` + `byteLength` |
| `ismap` / `isset` / `isweakmap` / `isweakset` `(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN | DUCK: `get set has delete size` / `add has delete size` / `get set has delete` / `add has delete`. BRAND: `X.prototype.has.call` has a slot to check. A Map ducks as a WeakMap — it can be used as one |

### 2 · Values with a slot
| Predicate | Bits | Notes |
|---|---|---|
| `isprom(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN | DUCK: a callable `then` — what `await` uses; a thenable is 4 and works. BRAND: the host's `[[PromiseState]]` check; `0` without a host (see above) |
| `isdate(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN | DUCK: `getTime` + `toISOString`. BRAND: `getTime.call`. `new Date()` → 61 |
| `isregex(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN | DUCK: `exec` + `test`. BRAND: the `source` getter has `[[OriginalSource]]` to read |
| `iserr(v)` | 1 NAME · 2 TAG · 4 DUCK · 8 BRAND · 16 PROTO · 32 OWN | NAME: the constructor's name ends with `Error` (`TypeError`, `MyValidationError`). DUCK: string `message` and `name`. BRAND: `Error.isError`, or the `[[ErrorData]]` builtin tag where the platform lacks it. OWN: one of this realm's *native* error classes — a user subclass is 29 |
| `isargs(v)` | 1 TAG · 2 SLOT | an arguments object: the one kind with no name and no protocol. SLOT: the builtin tag with no tag property in the way |
| `isbox(v)` | 1 NAME · 2 TAG · 4 BRAND · 8 PROTO · 16 OWN | a wrapper: `new Number(1)`, `Object('x')`, `Object(1n)`. A primitive is 0. BRAND: the matching `valueOf` brand |
| `isplain(v)` | 1 NAME · 2 SHAPE · 4 NULL · 8 LOCAL | NAME: `Object`. SHAPE: the prototype has a null prototype and a constructor named `Object` — plain by shape, whatever the realm. NULL: no prototype (a dictionary). LOCAL: this realm's `Object.prototype`. `{}` → 11; a literal from another realm → 3; `Object.create(null)` → 4; a class instance → 0 |

### 3 · Functions — `typeof` says `'function'` for all of them
No internal slot is reachable for these kinds, so the **intrinsic source text** (`Function.prototype.toString`, unforgeable for a genuine function) is the strongest witness, read by `charCodeAt`.

| Predicate | Bits | Notes |
|---|---|---|
| `isasync(v)` | 1 NAME · 2 TAG · 4 SRC · 8 PROTO | an async function, not an async generator. SRC: the source starts with the `async` keyword. PROTO: `%AsyncFunction.prototype%` |
| `isgenfn(v)` | 1 NAME · 2 TAG · 4 SRC · 8 PROTO | SRC: `function*` or a `*name` method |
| `isagenfn(v)` | 1 NAME · 2 TAG · 4 SRC · 8 PROTO | `async` then a generator |
| `isclass(v)` | 1 SRC · 2 FIXED | SRC: the `class` keyword. FIXED: a constructor whose own `prototype` is non-writable — class syntax and native constructors produce that, ordinary functions never do. `class {}` → 3; `Map` → 2; `function f() {}` → 0 |
| `isarrow(v)` | 1 NOCTOR · 2 SRC | NOCTOR: no `[[Construct]]` — arrows never have one, and neither do methods, async and generator functions, so it is the weak bit. SRC: parameters then `=>` |
| `isbound(v)` | 1 NAME · 2 SRC | NAME: starts with `bound `. SRC: native syntax — a bound function has no source of its own. 3 is bound; 2 alone is native; 1 alone was renamed |
| `isnative(v)` | 1 SRC | native syntax and not named `bound `. A function Proxy prints the same way (the spec says so) and reads as native |

### 4 · Iteration
| Predicate | Bits | Notes |
|---|---|---|
| `isiter(v)` | 1 DUCK · 2 SELF · 4 TAG · 8 PROTO | an iterator object. DUCK: `next`. SELF: `next` and `Symbol.iterator`, as every built-in iterator. TAG: `… Iterator`, `Iterator Helper`, `Generator`. PROTO: `%IteratorPrototype%`. A Map is iterable, not an iterator: 0 |
| `isaiter(v)` | 1 DUCK · 2 SELF · 4 TAG · 8 PROTO | the async protocol: `next` and `Symbol.asyncIterator`; `AsyncGenerator`; `%AsyncIteratorPrototype%` |
| `isgen(v)` | 1 DUCK · 2 TAG · 4 PROTO | a generator object. DUCK: `next`, `return`, `throw`. PROTO: `%GeneratorPrototype%` |
| `isagen(v)` | 1 DUCK · 2 TAG · 4 PROTO | an async generator object |
| `isiterable(v)` | 1 SYNC · 2 ASYNC | the protocols themselves, on anything: `'abc'` → 1, an async generator → 2 |

### 5 · Accessors of kind
| Accessor | Returns |
|---|---|
| `tname(v)` | `constructor.name` as a string, or `undefined` — the lexical witness, exposed |
| `ttag(v)` | `Symbol.toStringTag` as a string, or `undefined` — the tag witness, exposed. `ttag(Buffer.alloc(0))` → `'Uint8Array'` |
| `telem(v)` | the element type of a typed array, packed: bits 0–7 the width in bits, bits 8–11 the kind (`INT` 1 · `UINT` 2 · `FLOAT` 3 · `BIGINT` 4 · `BIGUINT` 5), bit 12 `CLAMPED`. From the slot when there is one (a `Buffer` and a subclass answer as Uint8), else from the tag, else from the name — a claim, then, and `istyped` says which. `0` when nothing looks like one. `Uint8ClampedArray` → `0x1208`; `Float16Array` → `0x310` |
| `terr(v)` | the error's kind: its constructor's name, for anything `iserr` sees — `'TypeError'`, `'MyValidationError'`; `undefined` otherwise |
| `tbox(v)` | the primitive inside a wrapper, by brand: `'number' \| 'string' \| 'boolean' \| 'symbol' \| 'bigint'`, or `undefined` |
| `tfn(v)` | the kind of a function as flags: 1 `ASYNC` · 2 `GEN` · 4 `CLASS` · 8 `ARROW` · 16 `METHOD` · 32 `BOUND` · 64 `NATIVE` · 128 `CTOR` (has `[[Construct]]`). `function f() {}` → 128; `async () => {}` → 9; `class {}` → 132; `Math.max` → 64; `f.bind(null)` → 32 + 128 when `f` constructs; `0` for a non-function |

```js
const e = t.telem(view);
const bytesPerElement = (e & 0xff) / 8;
const isFloat = ((e >> 8) & 0xf) === t.telem.FLOAT;
const clamped = (e & t.telem.CLAMPED) !== 0;
```

### Statics & factory
| Symbol | Purpose |
|---|---|
| `_typeim(overrides?)` | build a configured instance (see below) |
| `typeim` | the class, for `instanceof`; `typeim.prototype.ismap.BRAND` holds the constants too |

---

## Extending & configuring

The default export is a singleton — you never write `new`. To specialize behavior, pass overrides to `_typeim`. An override may call `super` to wrap the original, and its bound copy receives the base predicate's constants:

```js
import { _typeim } from '@slnknrr/type-im';

// a stricter instance: a costume counts for nothing
const strict = _typeim({
  ismap(v) { const x = super.ismap(v); return x & super.ismap.BRAND ? x : 0; },
});
strict.ismap({ [Symbol.toStringTag]: 'Map' }); // 0
strict.ismap.BRAND;                            // 8
```

Every internal call dispatches through `this`, so replacing one witness helper (`_name`, `_tag`, `_src`, `_slot`) replaces it in every predicate that uses it.

> ### ⚠️ Keep configurations few and long-lived
> Internal call sites are monomorphic and V8 inlines them to zero cost — **as long as few distinct `type-im` classes exist in the process.** Create your configuration **once, at module load**, and reuse it. `_typeim` caches by the `overrides` object (a `WeakMap`), so the *same object* always yields the *same instance*.

---

## Design notes

- **Bits per predicate, not one layout for all.** A typed array can witness two lexical facts (the suffix and the shape); an arguments object can witness only its tag; a function kind has a source text but no slot. Forcing one layout would either invent witnesses or drop them. The recurring names stay stable; the positions follow the kind.
- **The shape, not the list.** `_tshape` parses `(Int|Uint|Float|BigInt|BigUint)`, a power-of-two width of at least 8, an optional `Clamped`, and `Array` — by char codes. It accepted `Float16Array` before Node shipped it. The draft this grew from had the same parser and two blind spots — `BigUint` was not in its prefix table, and a failed name check returned early, hiding the slot witness for `Buffer` — both of which are exactly the kind of thing independent bits exist to catch.
- **The tag is the slot, for typed arrays only.** `%TypedArray%.prototype[@@toStringTag]` is a getter that returns the internal `[[TypedArrayName]]`, so on a genuine typed array `ttag` is the truth and names the kind through any subclass or renaming. On everything else the tag is a property anyone can set, which is why `TAG` and `BRAND` are separate bits.
- **`[[Construct]]` without calling.** `Reflect.construct(String, [], f)` checks `IsConstructor(newTarget)` and builds a `String` wrapper — it never invokes `f`. It is asked only for functions that print as native code (natives, bound functions, Proxies); for anything with source text, the syntax that made it decides exactly, and no exception is paid.
- **Two-stage brands.** The host's `util.types` first, because it never throws. The pure probe second, and only when a cheaper witness makes the value plausible — the classification benchmark went from 3k/s to 188k/s on that one rule, at the price of a blind spot the README states.
- **Methods detach.** The constructor binds every public method to its instance and installs it as an own, frozen property with its constants. `list.filter(t.isprom)` and `const { istyped } = t` work; `this` still finds the instance, and with it any override.

---

## Scripts

| Command | Does |
|---|---|
| `npm test` | behavioral suite (`node --test`, zero dependencies) — against another realm (`vm`), Node's `util.types` as oracle, costumes, subclasses, revoked proxies, and a child process without a host brand |
| `npm run types` | type-check the shipped declarations (`tsc --noEmit`) |
| `npm run bench` | the benchmarks above |

---

## Links

- Source: [https://codeberg.org/slnknrr/typeim_lib.js-for-pm_npm/src/branch/main](https://codeberg.org/slnknrr/typeim_lib.js-for-pm_npm/src/branch/main)
- npm: [https://www.npmjs.com/package/@slnknrr/type-im](https://www.npmjs.com/package/@slnknrr/type-im)
- Friendly wrapper — booleans and unpacked kinds over the same witnesses: [@slnknrr/type-ez](https://www.npmjs.com/package/@slnknrr/type-ez)
- The same principle for bytes, arrays, strings and numbers: [@slnknrr/buf-im](https://www.npmjs.com/package/@slnknrr/buf-im) · [@slnknrr/arr-im](https://www.npmjs.com/package/@slnknrr/arr-im) · [@slnknrr/str-im](https://www.npmjs.com/package/@slnknrr/str-im) · [@slnknrr/num-im](https://www.npmjs.com/package/@slnknrr/num-im)

## Author

Yury Slinkin (Юрий Слинкин)

- Email: slnknrr.dev@gmail.com
- Codeberg: https://codeberg.org/slnknrr

## License

MIT. See [LICENSE.md](https://codeberg.org/slnknrr/typeim_lib.js-for-pm_npm/src/branch/main/LICENSE.md).
