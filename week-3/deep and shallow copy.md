# Shallow Copy vs Deep Copy — Cloning Objects and Arrays

---

## What Does "Copy" Actually Mean?

You already know the starting point from Week 1: objects live on the heap, and a variable holds a **reference** to them.

When someone says "copy an object", they can mean **three different things**:

```
1. ASSIGNMENT      →  no copy at all. Two names, ONE object.
2. SHALLOW COPY    →  a new outer object. Anything nested inside is still SHARED.
3. DEEP COPY       →  a new outer object AND new copies of everything nested, all the way down.
```

```js
const original = { n: { v: 1 } };

const alias   = original;                 // 1. assignment
const shallow = { ...original };          // 2. shallow copy
const deep    = structuredClone(original);// 3. deep copy

original.n.v = 99;                        // mutate the NESTED object

alias.n.v;                                // 99   ← same object
shallow.n.v;                              // 99   ← outer is new, inner is shared
deep.n.v;                                 // 1    ← fully independent
```

> **Today's map:**
> assignment vs copy → shallow copy → deep copy → the JSON trick and its traps →
> `structuredClone` and its limits → circular references → writing a deep clone by hand → choosing the right tool

> **Vocabulary box.** A few built-in object types appear in today's examples. You only need the one-line idea of each:
> `Date` — a moment in time (`new Date(0)`).
> `RegExp` — a text pattern (`/ab+/gi`).
> `Map` — a collection of key → value pairs (`new Map([["k", 1]])`).
> `Set` — a collection of unique values (`new Set([1, 2])`).
> What matters today: they are **objects with hidden internal data** that ordinary property-by-property copying cannot see.

---

## Part 1 — Assignment Is Not a Copy

```js
const a = { x: 1 };
const b = a;             // b gets a copy of the REFERENCE

b.x = 2;
a.x;                     // 2        ← one object, two names
a === b;                 // true
```

This is Week 1's "mutation through a shared reference". Everything below is about getting a **second object** instead.

---

## Part 2 — Shallow Copy

A **shallow copy** creates a **new top-level object or array** and copies the **values** of its properties. If a value is a primitive, you get an independent copy. If a value is an object or array, you get a copy of the **reference** — the inner object is shared.

### How to make one

```js
const obj = { name: "Asha", address: { city: "Jaipur" } };
const arr = [1, [2, 3]];

const objCopy1 = { ...obj };              // spread (Wednesday)
const objCopy2 = Object.assign({}, obj);  // older equivalent (Wednesday, Part 8)
const arrCopy  = [...arr];                // spread for arrays
```

All of these produce the **same kind of copy**: one level deep.

### What is independent and what is shared

```js
const original = { p: 1, n: { v: 1 }, list: [10, 20] };
const copy = { ...original };

// Top level: independent
copy.p = 2;
original.p;              // 1          ← primitive copied by value

copy.extra = "x";
"extra" in original;     // false      ← adding keys to the copy doesn't touch the original

// One level down: SHARED
copy.n.v = 99;
original.n.v;            // 99         ← same inner object

copy.list.push(30);
original.list;          // [10, 20, 30]   ← same inner array
```

```
 original ─▶ { p: 1, n ──┐, list ──┐ }       copy ─▶ { p: 1, n ──┐, list ──┐ }
                          ▼         ▼                              ▼         ▼
                       { v: 1 }  [10, 20]  ◀──── the SAME inner objects ─────┘
```

### Arrays of arrays and arrays of objects

```js
const matrix = [[1, 2], [3]];
const copy = [...matrix];

copy[0].push(9);
matrix;                  // [[1, 2, 9], [3]]    ← the inner array is shared

const users = [{ id: 1 }];
const users2 = [...users];
users2[0].id = 99;
users[0].id;             // 99
```

### Mutation vs reassignment inside a shallow copy

This distinction (Week 1) decides whether a shallow copy is "safe enough":

```js
const original = { n: { v: 1 } };
const copy = { ...original };

copy.n.v = 5;            // MUTATION of the shared inner object   → original changes ❌
original.n.v;            // 5

copy.n = { v: 100 };     // REASSIGNMENT of the copy's own property → original unaffected ✅
original.n.v;            // 5
```

Replacing a nested property on the copy is safe. Reaching *through* it and changing something inside is not.

### How to tell what kind of copy you have

Compare the **references**:

```js
copy === original;           // false   → the outer objects differ (it IS a copy)
copy.n === original.n;       // true    → the inner objects are the SAME → shallow
```

> A deep copy has **no `===` match anywhere** between objects in the copy and objects in the original (Implementation Task 1).

### When a shallow copy is enough

```
✅ The object is flat (only primitives inside)
✅ You will only REPLACE top-level properties, never mutate nested ones
✅ You will update nested data by copying along the path (Wednesday, Part 4)
❌ Someone downstream will mutate nested objects or arrays
```

---

## Part 3 — Deep Copy

A **deep copy** duplicates **every object and array reachable from the original**. Afterwards the two structures share **nothing**.

```js
const original = { n: { v: 1 }, list: [{ id: 1 }] };
const deep = structuredClone(original);

deep.n.v = 99;
deep.list[0].id = 99;

original.n.v;            // 1       ← untouched
original.list[0].id;     // 1       ← untouched
```

### When you really need one

```
• A snapshot for undo/redo or "reset to initial values"
• Handing data to code that mutates it, when you must keep the original intact
• Test fixtures that each test is free to mutate
```

### When a deep copy is the wrong tool

```
• You only change ONE nested value  → copy just the path (Wednesday), share the rest
• UI state in frameworks that compare references to detect change
    (React, Redux...): a deep copy makes EVERY object new, so everything "looks changed"
• The structure is very large and you'd copy it repeatedly
• It contains things that can't be duplicated meaningfully, such as functions
• Identity matters (the same object is deliberately shared)
```

**Deep-cloning "to be safe" on every update is a code smell.** Prefer copying only what changes.

### Three ways to get a deep copy

| Way | In short |
|---|---|
| `JSON.parse(JSON.stringify(x))` | the old trick — works only for plain JSON-style data (Part 4) |
| `structuredClone(x)` | built in, handles far more types (Part 5) |
| a hand-written `deepClone` | full control; you decide what to support (Part 7) |

(Libraries such as lodash also provide a `cloneDeep`; the ideas are the same.)

---

## Part 4 — The JSON Trick and Its Pitfalls

For years the standard one-liner was:

```js
const copy = JSON.parse(JSON.stringify(original));
```

**How it works:** `JSON.stringify` turns the value into a **text string** (JSON). `JSON.parse` builds brand-new objects from that text. Since the text contains no references, the result shares nothing with the original — a deep copy.

```js
const original = { a: 1, n: { v: 2 }, list: [1, 2] };
const copy = JSON.parse(JSON.stringify(original));

copy.n.v = 99;
original.n.v;            // 2     ← independent
```

It is short and works everywhere. **But JSON can only represent a small set of types**, and everything else is silently changed or lost. That is the whole problem.

> JSON can represent: objects, arrays, strings, finite numbers, `true`/`false`, `null`. Nothing else.

### What JSON does to other values

| Value | After `JSON.parse(JSON.stringify(x))` | What happened |
|---|---|---|
| property with value `undefined` | **key removed** | silently dropped |
| `undefined` inside an array | `null` | silently replaced |
| function (property) | **key removed** | silently dropped |
| function inside an array | `null` | silently replaced |
| `Symbol` value | dropped (`null` in arrays) | silently dropped |
| `NaN`, `Infinity`, `-Infinity` | `null` | silently replaced |
| `-0` | `0` | sign lost |
| `Date` | an **ISO string** (not a `Date`) | type changed |
| `RegExp` | `{}` | emptied |
| `Map`, `Set` | `{}` | emptied |
| `Error` | `{}` | emptied |
| hole in a sparse array | `null` | silently replaced |
| the same object referenced twice | **two separate copies** | sharing lost |
| getter | its **value** at that moment | accessor lost |
| non-enumerable property | dropped | silently dropped |
| `BigInt` | **TypeError** | loud failure |
| circular reference | **TypeError** | loud failure |

### The key insight: most failures are **silent**

Two things throw. Everything else changes your data **without any error**, which is much worse.

```js
const original = {
  a: 1,
  u: undefined,
  f() {},
  n: NaN,
  d: new Date(0)
};

const copy = JSON.parse(JSON.stringify(original));
copy;
// { a: 1, n: null, d: "1970-01-01T00:00:00.000Z" }
//   u and f vanished · NaN became null · the Date is now a string
```

### Each pitfall, up close

**1. `undefined`, functions, Symbols**

```js
JSON.parse(JSON.stringify({ a: 1, u: undefined, fn() {}, s: Symbol("s") }));
// { a: 1 }

JSON.parse(JSON.stringify([1, undefined, function () {}, Symbol("s")]));
// [1, null, null, null]
```

A key whose value can't be represented is **removed** from objects. In arrays, removal would shift positions, so it becomes `null`.

**2. `NaN`, `Infinity`, `-0`**

```js
JSON.parse(JSON.stringify({ n: NaN, i: Infinity, m: -Infinity, z: -0 }));
// { n: null, i: null, m: null, z: 0 }
```

**3. Dates become strings — a classic real-world bug**

```js
const original = { createdAt: new Date(0) };
const copy = JSON.parse(JSON.stringify(original));

typeof copy.createdAt;           // "string"
copy.createdAt.getTime();        // TypeError: copy.createdAt.getTime is not a function
```

How: `Date` objects have a `toJSON` method that `JSON.stringify` calls, returning an ISO string. Parsing doesn't turn it back into a `Date`.

You can define `toJSON` on your own objects to control the output:

```js
JSON.stringify({ a: { toJSON() { return "custom"; } } });    // '{"a":"custom"}'
```

**4. Map, Set, RegExp, Error become `{}`**

```js
JSON.parse(JSON.stringify({
  r: /x/g,
  m: new Map([[1, 2]]),
  s: new Set([1]),
  e: new Error("boom")
}));
// { r: {}, m: {}, s: {}, e: {} }
```

These objects keep their data in hidden internal slots, not in ordinary properties, so `JSON.stringify` finds nothing to write. (`Error`'s `message` is a non-enumerable property, so it's skipped as well.)

**5. BigInt throws**

```js
JSON.stringify({ b: 1n });
// TypeError: Do not know how to serialize a BigInt
```

**6. Circular references throw**

```js
const a = {};
a.self = a;
JSON.stringify(a);
// TypeError: Converting circular structure to JSON
```

**7. Shared references are duplicated**

```js
const shared = { v: 1 };
const o = { a: shared, b: shared };       // o.a === o.b  →  true

const copy = JSON.parse(JSON.stringify(o));
copy.a === copy.b;                        // false  ← two separate objects now
```

The original had **one** object referenced twice. The text just writes it twice, so parsing builds two.

**8. Getters and hidden properties**

```js
const o = { a: 1 };
Object.defineProperty(o, "g", { get() { return "got"; }, enumerable: true });
Object.defineProperty(o, "h", { value: 2, enumerable: false });

JSON.stringify(o);       // '{"a":1,"g":"got"}'   ← getter ran, value kept; "h" skipped
```

**9. `JSON.stringify(undefined)` and functions at the top level**

```js
JSON.stringify(undefined);           // undefined  (not a string!)
JSON.stringify(function () {});      // undefined

JSON.parse(JSON.stringify(undefined));
// SyntaxError: "undefined" is not valid JSON
```

### When the JSON trick is fine

```
✅ Plain data that came from JSON in the first place (API responses with only
   objects / arrays / strings / numbers / booleans / null)
✅ You are sure there are no Dates, undefined values, NaN, Maps, Sets, functions...
❌ Anything else — and you often won't notice the damage
```

Today `structuredClone` (next) is the better default when you need a deep copy.

---

## Part 5 — `structuredClone()`

`structuredClone` is a **built-in function** that makes a deep copy using the browser's built-in *structured clone algorithm*.

```js
const original = { n: { v: 1 }, list: [1, 2], when: new Date(0) };
const copy = structuredClone(original);

copy.n.v = 99;
original.n.v;                  // 1                 ← independent
copy.when.getTime();           // 0                 ← still a real Date! (JSON would give a string)
```

It is available in **Node 17+** and in all current browsers (since about 2022).

### What it handles correctly (and JSON doesn't)

```js
structuredClone(new Date(0));                         // a real Date
structuredClone(/ab+/gi);                             // a real RegExp (flags kept)
structuredClone(new Map([[1, { v: 1 }]]));            // a real Map, keys and values cloned too
structuredClone(new Set([1, 2]));                     // a real Set

const c = structuredClone({ u: undefined, n: NaN, i: Infinity, z: -0, b: 10n });
"u" in c;                                             // true   ← undefined KEPT
c.n; c.i;                                             // NaN, Infinity   ← kept
Object.is(c.z, -0);                                   // true
typeof c.b;                                           // "bigint"        ← kept
```

**Circular references work:**

```js
const a = { name: "a" };
a.self = a;

const c = structuredClone(a);
c.self === c;                  // true    ← the cycle is rebuilt among the COPIES
c !== a;                       // true
```

**Shared references are preserved** (unlike JSON):

```js
const shared = { v: 1 };
const o = { a: shared, b: shared };

const c = structuredClone(o);
c.a === c.b;                   // true    ← still ONE object referenced twice
c.a !== shared;                // true    ← but it's a new object, not the original
```

**Also supported:** `Error` objects (type, `name` and `message` kept). Sparse-array **holes are preserved** (JSON would turn them into `null`, spread into `undefined`):

```js
const holes = [1, , 3];
1 in [...holes];                       // true     (spread filled the hole with undefined)
JSON.stringify(holes);                 // "[1,null,3]"
1 in structuredClone(holes);           // false    (still a hole)
```

### What it can't clone — the limits

**1. Functions → it THROWS (loudly)**

```js
structuredClone({ f() {} });
// DataCloneError: f() {} could not be cloned.

structuredClone(function () {});
// DataCloneError: function () {} could not be cloned.
```

Compare with JSON: JSON *silently drops* functions; `structuredClone` *refuses*. Failing loudly is the safer behaviour — but it means state containing callbacks can't be cloned this way.

> The error's `name` is `"DataCloneError"`. The exact message text differs between browsers and Node.

**2. Other things that throw:** `Symbol` values and `WeakMap`s.

```js
structuredClone({ s: Symbol("x") });       // DataCloneError
structuredClone(new WeakMap());            // DataCloneError
```

**3. Objects that get behaviour from another object come back as plain data**

`structuredClone` (like the JSON trick) copies an object's **own data**. Behaviour an object *inherits* from another object isn't carried over, so such objects come back as plain objects. Why that happens, and how to deal with it, depends on **prototypes** — Saturday's topic — so it isn't demonstrated here.

**4. Getters are run; descriptors and hidden things are dropped**

```js
const o = { a: 1 };
Object.defineProperty(o, "g", { get() { return "got"; }, enumerable: true });
Object.defineProperty(o, "h", { value: 2, enumerable: false });

const c = structuredClone(o);
Object.keys(c);                              // ["a", "g"]
Object.getOwnPropertyDescriptor(c, "g");
// { value: "got", writable: true, enumerable: true, configurable: true }   ← a plain data property
c.h;                                         // undefined   ← the non-enumerable property is gone
```

Only **own enumerable** properties are cloned, as with spread (Wednesday).

**5. Freezing is not preserved**

```js
const c = structuredClone(Object.freeze({ a: { b: 1 } }));
Object.isFrozen(c);          // false
Object.isFrozen(c.a);        // false
```

**6. Errors lose custom properties**

```js
const e = new TypeError("boom");
e.code = 5;

const c = structuredClone(e);
c.name;                      // "TypeError"   ← type and message kept
c.message;                   // "boom"
c.code;                      // undefined     ← your extra property is gone
```

**7. A `RegExp`'s `lastIndex` resets to 0.**

**8. Very deep structures overflow the stack**

```js
// 100 000 levels of nesting, in the Node version tested:
structuredClone(deepChain);       // RangeError: Maximum call stack size exceeded
JSON.stringify(deepChain);        // RangeError: Maximum call stack size exceeded
```

The depth limit is engine-dependent. It is the reason an **iterative** clone exists (Implementation Task 4).

### `JSON` vs `structuredClone` at a glance

| | `JSON.parse(JSON.stringify(x))` | `structuredClone(x)` |
|---|---|---|
| `Date`, `Map`, `Set`, `RegExp` | lost / `{}` / string | ✅ cloned as the real type |
| `undefined`, `NaN`, `Infinity`, `-0` | dropped / `null` / `0` | ✅ kept |
| `BigInt` | ❌ throws | ✅ kept |
| circular references | ❌ throws | ✅ handled |
| shared references | ❌ duplicated | ✅ preserved |
| sparse array holes | `null` | ✅ preserved |
| functions | silently dropped | ❌ throws `DataCloneError` |
| `Symbol` values | silently dropped | ❌ throws |
| getters / hidden properties / freeze | value / dropped / not kept | value / dropped / not kept |
| failure style | mostly **silent** | **loud** |

---

# Part 6 — Deep Cloning: Circular & Shared References (WeakMap + Recursion)

> **Scope of this version — read this first**
>
> The `clone` you write today covers exactly these:
>
> - **primitives** (including `null` and `undefined`)
> - **arrays**
> - **plain objects**
> - **shared references**
> - **circular references**
>
> Functions pass straight through (the clone shares them — see section 13).
>
> **Not covered yet, by design:** special built-in objects such as `Date`, `Map`, `Set` and `RegExp`, and objects whose behaviour is inherited from another object. Recognising these and rebuilding them correctly depends on how objects are linked and constructed — that is **prototypes**, which is Saturday's topic. (One edge case, a property literally named `__proto__`, belongs there too.)
>
> **After Saturday we will extend this function into a full deep clone that covers those types.** Until then: practise this version, and use `structuredClone` when you need those types.

Interviewers love this question, and it is especially important in SDE-2 interviews. The goal is not to memorize code — it's to be able to say **why every line is there**.

A basic recursive deep clone can copy nested objects and arrays, but it fails when objects refer to themselves (**circular references**) or when multiple properties share the same object (**shared references**).

So we will:

1. Define the job and the problem.
2. Understand the tool that solves it (`WeakMap`).
3. Build the function.
4. Trace it step by step.
5. Take it apart line by line.

---

### 1. What is the job?

> Given any value, return a copy that has the **same shape and values**, but shares **no arrays or objects** with the original.

That's the whole contract. Everything in the function exists to keep that promise.

Now suppose we have this object:

```js
const user = {
    name: "Dharmaj",
    account: {
        isPublic: false,
        theme: "dark",
    },
};

user.self = user;
```

Notice this line:

```js
user.self = user;
```

We are making the `self` property point back to the original `user` object.

So:

```js
user.self === user; // true
```

Our object now has a **circular reference**.

What do we actually want when we deep clone this object?

We want a completely separate copy, but we also want to **preserve the original object's reference relationships**.

In other words:

- The cloned object must be different from the original object.
- Its nested `account` object must also be a separate copy.
- Its `self` property must point to the **cloned object itself**, NOT the original object.

Our expected result should behave like this:

```js
const clonedUser = clone(user);

console.log(clonedUser !== user);                 // true
console.log(clonedUser.account !== user.account); // true
console.log(clonedUser.self === clonedUser);      // true
console.log(clonedUser.self === user);            // false
```

This is our goal: **create new objects while preserving their reference relationships.**

---

### 2. Why does a normal recursive clone fail?

Let's first understand why we need something special.

Consider this simple implementation:

```js
function clone(value) {
    if (value === null || typeof value !== "object") {
        return value;
    }

    const result = Array.isArray(value) ? [] : {};

    for (const key of Object.keys(value)) {
        result[key] = clone(value[key]);
    }

    return result;
}
```

This works fine for ordinary **tree-shaped data** (data with no cycles and nothing referenced twice).

It breaks in exactly **two situations**:

```js
// 1. A cycle → it never finishes
const a = {};
a.self = a;
clone(a);                // RangeError: Maximum call stack size exceeded

// 2. A shared reference → it silently makes TWO copies of one object
const shared = { v: 1 };
const o = { x: shared, y: shared };       // one object, referenced twice
const c = clone(o);
c.x === c.y;             // false   ← the original had ONE object; the clone has TWO
```

#### Why the cycle never finishes

Let's follow the execution on our `user` object:

```text
clone(user)
    |
    |-- clone(user.name)
    |      |
    |      └── "Dharmaj" (primitive, return)
    |
    |-- clone(user.account)
    |      |
    |      └── Clone account properties
    |
    |-- clone(user.self)
           |
           └── user.self points to user itself!

                clone(user)
                    |
                    └── user.self points to user again!

                         clone(user)
                             |
                             └── And it continues forever...
```

The function keeps encountering the same object again and again.

It never finishes and eventually throws:

```text
RangeError: Maximum call stack size exceeded
```

The problem is that our function has **no memory** of which objects it has already cloned.

The fix for **both** problems is the same: **remember what you have already cloned.**

This is exactly where `WeakMap` comes in.

---

### 3. Understand WeakMap with a simple analogy

Imagine you are cloning objects, and you maintain a small notebook.

Whenever you clone an object, you write down two things:

1. The original object.
2. The cloned object corresponding to it.

For example:

| Original object | Its cloned object   |
| --------------- | ------------------- |
| `user` (A)      | `clonedUser` (C)    |
| `account` (B)   | `clonedAccount` (D) |

Think of it like this:

```text
        NOTEBOOK (WeakMap)

        Original A  ────────▶ Clone C

        Original B  ────────▶ Clone D
```

Now imagine that while cloning, you encounter **Original A** again.

Instead of creating another copy, you check your notebook:

> "Have I already cloned this exact object?"

If yes, simply return the clone that you created earlier.

This is the entire idea behind handling circular references.

`WeakMap` is our notebook. It remembers which original object corresponds to which cloned object.

---

### 4. Why WeakMap instead of a normal object?

The registry maps *original object → its clone*, so its keys are **objects, compared by identity**.

A plain object can't do that — its keys become strings, and every object turns into the same key, `"[object Object]"` (Monday). A `WeakMap` (Week 1) takes real objects as keys:

```js
const visited = new WeakMap();

visited.set(originalObject, clonedObject);

visited.has(originalObject); // true
visited.get(originalObject); // clonedObject
```

The three methods we need are:

| Method                 | Purpose                                          |
| ---------------------- | ------------------------------------------------ |
| `set(original, clone)` | Remember the original-to-clone relationship      |
| `has(original)`        | Check whether we have already cloned this object |
| `get(original)`        | Retrieve its existing clone                      |

We are not merely tracking whether an object was visited.

We are tracking **WHICH CLONE belongs to WHICH ORIGINAL OBJECT**.

That distinction is extremely important.

It's also *weak*: a `WeakMap` doesn't keep the originals alive. Once the clone is finished and the registry is thrown away, garbage collection is free to reclaim everything.

---

### 5. The most important rule: Register BEFORE recursion

This is the most important part of the entire implementation.

Look at these two lines:

```js
const result = Array.isArray(value) ? [] : {};

visited.set(value, result);
```

Suppose:

```js
value = user;   // Original object A
result = {};    // New empty object C
```

When we execute:

```js
visited.set(value, result);
```

We are telling the `WeakMap`:

```text
Original user (A) → Cloned user (C)
```

Why do we do this **BEFORE** cloning the properties?

Because one of those properties might point back to the original object!

If we don't register the clone first, the function won't know that the original object has already been cloned.

It will keep creating new clones and recursively following the circular reference.

Think of the order:

```text
1. Receive original object A
          |
2. Check if A already exists in WeakMap
          |
3. Create an empty clone C
          |
4. Store A → C in WeakMap
          |
5. Start cloning A's properties
          |
6. If A appears again, return C
```

> **Remember:** Create the clone → Register it → Then recurse.
>
> Never recurse first and register afterward.

Here is the cycle `a.self = a` playing out in miniature:

```text
clone(a)      → create result, THEN visited.set(a → result), then fill result
   fill: key "self" → clone(a) → visited.has(a)? YES → return result   ✅ cycle closed
```

If `visited.set` came *after* filling, `visited.has(a)` would still be `false` on the second visit, and the function would recurse forever.

---

### 6. The recipe in plain English

For **every value** the function meets, it goes through these questions, in this order:

```text
Q1. Is it a primitive, null, or a function?     → nothing to duplicate → return it as it is
Q2. Have I already cloned this exact object?    → return that earlier clone
Q3. Otherwise: make an empty container          → [] if it's an array, {} if it's a plain object
                                                  and REGISTER it in the WeakMap
Q4. Fill it                                     → clone each child, the same way
```

"Fill it" always means the same thing: for each child, call `clone` again — so the same questions get asked all the way down.

---

### 7. Complete implementation

Now that we understand the logic, the complete function becomes much easier:

```js
const clone = function(value, visited = new WeakMap()) {

    // Step 1: Return primitive values directly
    if (typeof value !== "object" || value === null) {
        return value;
    }

    // Step 2: If already cloned, return its existing clone
    if (visited.has(value)) {
        return visited.get(value);
    }

    // Step 3: Create an empty object or array
    const result = Array.isArray(value) ? [] : {};

    // Step 4: Register original → clone BEFORE recursion
    visited.set(value, result);

    // Step 5: Clone each property recursively
    const keys = Object.keys(value);

    for (const key of keys) {
        result[key] = clone(value[key], visited);
    }

    // Step 6: Return the completed clone
    return result;
};
```

---

### 8. Let's walk through the actual example step by step

We'll use our original object:

```js
const user = {
    name: "Dharmaj",
    account: {
        isPublic: false,
        theme: "dark",
    },
};

user.self = user;
```

We call:

```js
const clonedUser = clone(user);
```

Let's follow what happens inside the function.

#### Step 1: First function call

```js
clone(user);
```

Our function receives:

```js
value = user;
```

We check:

```js
if (typeof value !== "object" || value === null) {
    return value;
}
```

Is `user` an object? **Yes.**

So we continue.

#### Step 2: Check WeakMap

```js
if (visited.has(value)) {
    return visited.get(value);
}
```

Initially, the `WeakMap` is empty.

So:

```js
visited.has(user); // false
```

We haven't cloned this object before.

#### Step 3: Create an empty clone

```js
const result = Array.isArray(value) ? [] : {};
```

Since `user` is an object and not an array:

```js
result = {};
```

Let's call:

- Original `user` = **A**
- New empty clone = **C**

#### Step 4: Register the relationship

```js
visited.set(value, result);
```

Our `WeakMap` now contains:

```text
Original A ─────────▶ Clone C
```

This is important because we might encounter **A** again.

#### Step 5: Start processing properties

```js
const keys = Object.keys(value);
```

Our keys are:

```js
["name", "account", "self"]
```

Now we process them one by one.

#### Step 6: First property — `name`

```js
result.name = clone(value.name, visited);
```

The value is:

```js
"Dharmaj"
```

It is a primitive value, not an object.

So the function immediately returns it.

Our clone now looks like:

```js
{
    name: "Dharmaj"
}
```

#### Step 7: Second property — `account`

Now we execute:

```js
result.account = clone(value.account, visited);
```

The `account` property is another object.

Let's call it **Original B**.

The function receives Original B.

It checks:

```js
visited.has(value);
```

The answer is `false` because we haven't cloned `account` yet.

So we create a new empty object:

```js
result = {};
```

Let's call this **Clone D**.

Then:

```js
visited.set(value, result);
```

Our `WeakMap` now looks like:

```text
Original A ─────────▶ Clone C
Original B ─────────▶ Clone D
```

Now we process the `account` properties:

```js
isPublic: false
theme: "dark"
```

Both are primitive values, so they are copied directly.

Clone D becomes:

```js
{
    isPublic: false,
    theme: "dark"
}
```

The `account` function call returns Clone D.

Back in the original function call:

```js
result.account = Clone D;
```

Our cloned user now looks like:

```js
{
    name: "Dharmaj",
    account: {
        isPublic: false,
        theme: "dark"
    }
}
```

Everything is working perfectly so far!

#### Step 8: Third property — `self` (THE TRICKY PART)

Now the loop reaches:

```js
self
```

Remember:

```js
user.self = user;
```

So what is:

```js
value.self
```

It is the **original `user` object itself**!

Therefore, we effectively execute:

```js
clone(user, visited);
```

Again!

But this time, something different happens.

The function checks:

```js
if (visited.has(value)) {
    return visited.get(value);
}
```

Now:

```js
visited.has(user); // true
```

Why?

Because earlier we stored:

```js
visited.set(user, clonedUser);
```

So the function executes:

```js
return visited.get(user);
```

And what does this return?

The already-created `clonedUser` (**Clone C**)!

**Not** the original `user`.

We don't create another object, and we don't recurse again.

The original function receives the returned clone and executes:

```js
result.self = clonedUser;
```

And that's it!

Our circular reference has been handled correctly.

---

### 9. Final result

Conceptually, our cloned object now looks like this:

```text
              ORIGINAL                    CLONE

                A                          C
           ┌──────────┐               ┌──────────┐
           │ user     │               │ clone    │
           │          │               │          │
           │ name     │               │ name     │
           │ account ─┼───▶ B         │ account ─┼───▶ D
           │ self     │               │ self     │
           └────┬─────┘               └────┬─────┘
                │                          │
                └── points to A            └── points to C
```

The original and cloned objects are completely separate, but their internal reference relationships are preserved.

---

### 10. WeakMap also handles shared references!

Circular references aren't the only reason we need `WeakMap`.

Consider:

```js
const address = {
    city: "Delhi"
};

const user = {
    home: address,
    office: address
};
```

Notice that both `home` and `office` point to the **SAME** original `address` object.

```js
user.home === user.office; // true
```

When cloning:

1. We first clone `home`.
2. `WeakMap` stores the original `address` → cloned `address`.
3. When we reach `office`, `WeakMap` finds the same original `address`.
4. It returns the existing cloned `address` instead of creating another one.

So:

```js
const cloned = clone(user);

console.log(cloned.home === cloned.office); // true
console.log(cloned.home === address);       // false
console.log(cloned.office === address);     // false
```

**Why this works:** sharing is just "the same original reached by two paths". The first path clones it and registers it; the second path finds it in the registry and reuses the **same** clone — so the clone is shared in exactly the same way the original was.

This is another important property of our implementation.

**`WeakMap` preserves object identity and relationships, not just object values.**

---

### 11. A full walkthrough: array + cycle + shared reference together

```js
const a = { name: "a", list: [1, 2] };
a.self = a;                  // a cycle
a.again = a.list;            // a shared reference
```

```text
clone(a)                      visited: { }
  a is an object, not visited, not an array
  → result1 = {};  visited: { a → result1 }
  key "name"  → clone("a")        primitive → "a"                     result1.name = "a"
  key "list"  → clone(a.list)     object, not visited, IS an array
                  → result2 = [];  visited: { a → result1, list → result2 }
                  key "0" → 1,  key "1" → 2                           result2 = [1, 2]
                result1.list = result2
  key "self"  → clone(a)          visited.has(a) → return result1     result1.self = result1    ← cycle closed
  key "again" → clone(a.list)     visited.has(list) → return result2  result1.again = result2   ← sharing kept
```

Result:

```js
const c = clone(a);
c !== a;                     // true    ← new object
c.self === c;                // true    ← the cycle is among the COPIES
c.list !== a.list;           // true    ← deep
c.again === c.list;          // true    ← still one array, referenced twice
```

---

### 12. Why every line exists — and what breaks without it

We deleted each line in turn and ran the result:

| Line | Why it's there | If you delete it… |
| --- | --- | --- |
| `value === null` | `typeof null` is `"object"` (a Week 1 quirk), so `null` would be mistaken for an object | `TypeError: Invalid value used as weak map key` — `null` reaches `visited.set` |
| `typeof value !== "object"` | Numbers, strings, booleans, `undefined`, symbols, BigInt **and functions** need no cloning | `TypeError` again — `5` reaches `visited.set`, which only accepts objects |
| `visited = new WeakMap()` (the parameter) | The registry. The outermost call gets a fresh one by default | no registry → you're back to the first draft |
| passing `visited` in `clone(value[key], visited)` | Every recursive call must share **the same** registry | each call starts with an empty registry → **cycle: `RangeError`**; **shared reference: duplicated** (`c.x === c.y` is `false`) |
| `if (visited.has(value)) return visited.get(value)` | The lookup that stops cycles and keeps sharing | **cycle: `RangeError`**; **shared reference: duplicated** |
| `Array.isArray(value) ? [] : {}` | Arrays must come out as arrays | arrays become plain objects: `Array.isArray` is `false`, `length` is `undefined`, keys are `"0"`, `"1"` |
| `visited.set(value, result)` **before** the loop | The cycle is met *while filling*, so the registry must already know about this object | registered after the loop → **cycle: `RangeError`** |
| the recursive `clone(...)` calls | Clone the children too | copy `result[key] = value[key]` instead and you've written a **shallow** copy |
| `Object.keys(value)` | Visit the object's **own, enumerable** properties (the same set spread copies) | — (see section 14) |

(Error messages are from Node; the wording differs between engines.)

---

### 13. The reasoning behind each choice

#### Why return primitives and functions as they are?

A primitive is copied by value automatically (Week 1) — there's nothing shared to separate.

A **function** can't be duplicated meaningfully: its closure (Week 2) is the surrounding scope, and a scope can't be copied. So the clone **shares the same function**:

```js
const src = { f() { return 1; }, n: { v: 1 } };
const c = clone(src);
c.f === src.f;           // true    ← shared (can't be cloned)
c.n === src.n;           // false   ← cloned
```

#### Why a `WeakMap` for the registry?

See section 4: object keys compared by identity, and weak so garbage collection can reclaim everything afterwards.

#### Why default the parameter (`visited = new WeakMap()`)?

So callers write `clone(x)` with one argument, while the function passes the registry down internally. The default creates one registry **per top-level call**.

#### Why register *before* filling?

Because of the cycle — see section 5.

#### Why does this also fix shared references?

See section 10: the second path to the same original finds the registered clone and reuses it.

---

### 14. What this version does with things it wasn't built for

Run on its supported types it is correct. Outside them, it gives **wrong results without any error**:

```js
clone({ when: new Date(0) });    // { when: {} }     ← a Date becomes an empty object
clone(new Map([[1, 2]]));        // an empty plain object (size is undefined)
```

It also has these properties, which follow from "copy own enumerable data":

```text
• Holes in a sparse array stay holes (Object.keys skips them): [1, , 3] → [1, <empty>, 3]
• Getters are RUN and their value is copied (the clone gets a plain data property)
• Hidden (non-enumerable) properties are dropped
• Inherited properties are not copied (own vs inherited, Monday)
• A function is shared, not cloned
• Very deep data overflows the call stack → the iterative version (Task 4)
```

This is exactly why the **full** version has to wait until after Saturday: handling `Date`, `Map`, `Set`, `RegExp` and inherited behaviour properly is a prototypes topic.

---

### 15. Complexity

```text
Time:   O(n)  — n is the number of values visited; the registry guarantees each array/object is cloned once
Space:  O(n)  — the copy itself plus the registry
Depth:  O(d)  — recursion depth equals how deeply the data is nested (this is what overflows)
```

---

### 16. Common mistakes

```text
✗ Forgetting null (typeof null === "object")
✗ Treating arrays as plain objects → the copy isn't an array
✗ Registering in the registry AFTER filling → cycles still overflow
✗ Forgetting to pass `visited` into the recursive call → every call has its own empty registry
✗ Using a plain object {} as the registry → every object becomes the key "[object Object]"
✗ Expecting functions to be "cloned" — they are shared
✗ Claiming it "handles everything" — it doesn't
```

---

### 17. Important things to remember for SDE-2 interviews

When implementing a deep clone, think about these cases:

| Case                | What should happen?              |
| ------------------- | -------------------------------- |
| Primitive values    | Return directly                  |
| `null`              | Return directly                  |
| Functions           | Return directly (shared)         |
| Nested objects      | Create independent copies        |
| Arrays              | Preserve array structure         |
| Circular references | Reuse the existing clone         |
| Shared references   | Preserve the shared relationship |

Also remember that this is a **basic implementation** for ordinary objects and arrays. It does not fully support `Date`, `RegExp`, `Map`, `Set`, prototypes, property descriptors, symbol properties, or cloning functions.

If the interviewer asks for a production-grade implementation, **clarify the supported types and requirements first**.

#### How to talk about it in an interview

> *"I'll handle primitives, arrays, plain objects, shared references and cycles. Primitives and functions are returned as they are. I keep a WeakMap from original to clone, and I register each new array or object before filling it, so cycles and shared references both resolve to the clone I already made. It doesn't handle special objects like Date or Map — for those I'd use `structuredClone`."*

**Saying what you don't handle is part of the answer.**

---

### Final takeaway

There are only **TWO** lines you absolutely must understand:

```js
if (visited.has(value)) {
    return visited.get(value);
}
```

This means:

> "If I have already cloned this object, don't clone it again. Give me the clone I created earlier."

And:

```js
visited.set(value, result);
```

This means:

> "I have created a clone for this original object. Remember this relationship before I start cloning its properties."

**The golden rule: Create → Register → Recurse.**

Once you understand these three steps, circular references and shared references become much easier to handle.

Don't memorise the code. Understand **why** these two `WeakMap` operations exist and **when** they execute.


## Part 8 — Choosing the Right Tool

| Situation | Use |
|---|---|
| Flat object/array of primitives, or you only replace top-level properties | spread: `{ ...obj }`, `[...arr]` |
| Update **one nested value**, keep the rest shared (UI state, reducers) | **targeted copy**: spread each level on the path, or a `setIn` helper (Task 5) |
| Plain, JSON-style data, deep copy needed | `structuredClone` (preferred), or the JSON trick |
| `Date`, `Map`, `Set`, `RegExp`, `undefined`, `NaN`, `BigInt`, circular or shared references | `structuredClone` |
| Plain data only (primitives, arrays, plain objects, cycles, shared references) and you want full control or practice | your hand-written `deepClone` (Part 7) |
| The data contains functions | `structuredClone` throws; your `deepClone` shares the functions; or clone only the data parts |
| Special built-ins or inherited behaviour that must survive the copy | the full deep clone, built after Saturday (Prototypes) — until then use `structuredClone` for the built-ins |
| Extremely deep structures | iterative clone (Task 4), or avoid cloning |
| Only need to **compare** or **detect change** | don't clone — compare references or use a deep-equal helper |

**A good default for interviews:** *"For plain data I'd use `structuredClone`. For state updates I'd copy only the path that changes. JSON only if I know the data is JSON-safe."*

---

## Part 9 — Gotchas

**1. `copy !== original` does not mean "deep".**
A shallow copy also has a different identity. Check the *nested* references.

**2. Mutating through a shallow copy changes the original; reassigning on the copy doesn't.**
`copy.n.v = 1` ❌ vs `copy.n = { v: 1 }` ✅ (Part 2).

**3. The JSON trick damages data silently.**
Dates become strings, `undefined`/functions vanish, `NaN` becomes `null`. Only BigInt and cycles announce themselves with an error.

**4. A `Date` that went through JSON is no longer a `Date`.**
`copy.createdAt.getTime()` throws a `TypeError`. A very common real bug.

**5. `structuredClone` throws on functions — even deep inside the object.**
State that contains callbacks can't be cloned with it. (Use the `safeClone` fallback in Task 2, or clone only the data.)

**6. Deep cloning breaks identity.**
Every object becomes a new object, so `===` comparisons, memo caches, and "has this changed?" reference checks all stop matching.

**7. Deep clone on every update is expensive and defeats reference-equality optimisations.**
Copy only what changes.

**8. Deep recursion overflows.**
Recursive clones, `JSON.stringify`, and `structuredClone` all hit `RangeError: Maximum call stack size exceeded` on very deep data (the exact limit depends on the engine).

**9. Freezing isn't copied.**
Clones — shallow or deep — come back unfrozen.

**10. Spread, `Object.assign`, JSON, `structuredClone` and the hand-written clone all copy only own, enumerable properties.**
Inherited properties and hidden (non-enumerable) properties are left behind.

**11. Your hand-written `deepClone` is for plain data only.**
Feed it a `Date` or a `Map` and you get an empty object back — silently. That limit is deliberate for now (Part 7).

---

## How Everything Connects

```
Week 1
  → references, mutation vs reassignment   → why shallow copies share nested objects
  → typeof null                            → the null check in every clone
  → WeakMap / garbage collection           → the `seen` registry (Part 6, Part 7)

Week 2
  → closures                               → why functions can't be cloned
  → default parameters                     → deepClone(value, seen = new WeakMap())

Monday (Objects)
  → own vs inherited, descriptors          → what clones keep and drop
  → Object.keys                            → cloning own enumerable properties
  → objects as keys become strings         → why the registry needs a WeakMap
  → freeze                                 → clones are not frozen

Tuesday / Wednesday
  → spread                                 → the everyday shallow copy
  → copy-the-path updates                  → the practical alternative to deep clone
  → rest/spread, destructuring             → setIn(path) helper (Task 5)

Coming next
  → Fri: array methods (map/filter/reduce) — they return NEW arrays, but are shallow too
  → Sat: prototypes — then we extend deepClone to the types skipped today
```

---

## Implementation Tasks

> **How this section is organised**
>
> **A. Asked in company interviews** — the implementation questions that candidates and prep sites report for this topic, each with the follow-ups interviewers add.
> **B. Additional tasks** — a few extras from me, kept at the same level as section A.
>
> The reports come from candidate-posted experiences and prep sites. They show that these questions **get asked**, not how often.
>
> Try each one yourself first: write test cases, write the code, then open the solution.

---

# A. Asked in company interviews

### Task A1 — Deep clone (with circular references)

**Reported in:** Hiver (SDE-2 Frontend), Upstox, and Prepfully's list of frontend questions for companies such as Meta, Amazon and Agoda; GreatFrontEnd lists it in its mid-level tier.

**How it's asked:** *"What is a deep clone? How would you do it? Write one."* Usually starts with plain objects and arrays, then the interviewer adds circular references.

**Your job:** return a copy that has the same shape and values but shares **no arrays or objects** with the original. Cover primitives, arrays, plain objects, shared references and circular references.

<details>
<summary>Solution</summary>

```js
function deepClone(value, visited = new WeakMap()) {
  // 1. primitives, null, functions: return as they are
  if (typeof value !== "object" || value === null) return value;

  // 2. already cloned this exact object? reuse that clone
  if (visited.has(value)) return visited.get(value);

  // 3. create the empty container and REGISTER it before filling it
  const result = Array.isArray(value) ? [] : {};
  visited.set(value, result);

  // 4. fill it, cloning every child the same way
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) {
    result[keys[i]] = deepClone(value[keys[i]], visited);
  }
  return result;
}

const user = { name: "Dharmaj", account: { theme: "dark" }, list: [1, [2, 3]] };
user.self = user;              // a cycle
user.again = user.list;        // a shared reference

const c = deepClone(user);
c !== user;                    // true
c.account !== user.account;    // true   ← deep
c.self === c;                  // true   ← the cycle is among the COPIES
c.again === c.list;            // true   ← sharing preserved
```

Create → Register → Recurse. The full explanation, trace and "what breaks without each line" are in the Deep Cloning part of these notes.
</details>

**Follow-ups interviewers add:**
- **"What about circular references?"** The `visited` WeakMap, registered **before** filling the clone.
- **"What if two properties point to the same object?"** The same registry: the second path finds the first clone and reuses it.
- **"Is spread / `Object.assign` a deep copy?"** No — one level only; nested objects are shared.
- **"Why not `JSON.parse(JSON.stringify(x))`?"** Drops `undefined` and functions, turns `NaN` into `null` and `Date` into a string, loses shared references, throws on cycles and BigInt (Part 4).
- **"What does `structuredClone` do, and what are its limits?"** Handles cycles, `Date`, `Map`, `Set`; throws on functions (Part 5).
- **"What about functions?"** They are returned as-is, so the clone shares them.
- **"What about `Date`, `RegExp`, `Map`, `Set` or class instances?"** ⏳ This version gives `{}` for them. Handling them needs prototype knowledge — after Saturday. For now the answer is: say so, and use `structuredClone`.
- **"How would you test it?"** Mutate a nested value in the copy and check the original is unchanged; check `copy.x !== original.x` at every level; for a cycle, `c.self === c`; for sharing, `c.a === c.b`.

---

### Task A2 — Deep equal

**Reported in:** GreatFrontEnd (listed with deep clone in its mid-level tier, with its own question page) and ExplainThis ("an interviewer will ask you to implement it yourself"); it is also commonly paired with deep clone in prep lists.

**How it's asked:** *"Write `deepEqual(a, b)` — compare two values structurally."* The standard version only needs `undefined`, numbers, strings, booleans, `null`, arrays and plain objects, and assumes no circular references.

<details>
<summary>Solution</summary>

```js
function deepEqual(a, b) {
  if (a === b) return true;                              // same primitive or same reference
  if (a !== a && b !== b) return true;                   // NaN equals NaN

  // anything left that isn't two real objects cannot be equal
  if (a === null || b === null || typeof a !== "object" || typeof b !== "object") return false;

  if (Array.isArray(a) !== Array.isArray(b)) return false;                 // [] vs {}
  if (Array.isArray(a) && a.length !== b.length) return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  for (let i = 0; i < keysA.length; i++) {
    const key = keysA[i];
    if (!Object.hasOwn(b, key)) return false;
    if (!deepEqual(a[key], b[key])) return false;
  }
  return true;
}

deepEqual({ id: 1 }, { id: 1 });                  // true
deepEqual([{ id: "1" }], [{ id: "2" }]);          // false
deepEqual({ a: 1, b: 2 }, { b: 2, a: 1 });        // true   ← key order doesn't matter
deepEqual([], {});                                // false
deepEqual({ a: undefined }, {});                  // false  ← a missing key is not the same as undefined
deepEqual(NaN, NaN);                              // true
deepEqual([1, , 3], [1, undefined, 3]);           // false  ← a hole is not the same as undefined
```
</details>

<details>

<summary>My Solution</summary>

```js
function deepEqual(v1, v2){
    const typeFirst = typeof v1;
    const typeSecond = typeof v2;

    if(typeFirst !== typeSecond) return false;

    if((typeof v1 !== "object" || v1 === null) ){
        if(v1 === v2) return true;
        else return false;
    }   

    if(Array.isArray(v1) !== Array.isArray(v2))
    return false;


    let keys_first = Object.keys(v1);
    let keys_second = Object.keys(v2);

    if(keys_first.length !== keys_second.length) return false;

    for(const key of keys_first){
        if(!keys_second.includes(key)) return false;

        if(deepEqual(v1[key],v2[key]) === false) return false;
    };

    return true;
};
```
</details>
 

**Follow-ups interviewers add:**
- **"Why not `JSON.stringify(a) === JSON.stringify(b)`?"** It depends on key order, drops `undefined` and functions, turns `NaN` into `null`, and throws on circular data.
- **"What about `NaN`?"** `NaN !== NaN`, so it needs the explicit check (or `Object.is`).
- **"`0` vs `-0`?"** With `===` they are equal. `Object.is` would say they differ. Choose one and state it.
- **"Array vs object with the same keys?"** `[]` vs `{}` must be `false` — hence the `Array.isArray` check.
- **"`{ a: undefined }` vs `{}`?"** Not equal: the key counts are different.
- **"Sparse arrays?"** Length and present indices both matter; `Object.keys` skips holes.
- **"Circular references?"** Advanced follow-up. Idea: remember which *pairs* of objects you are already comparing and treat a repeat as equal, so you never recurse forever.
- **"`Date`, `RegExp`, `Map`, `Set`, class instances?"** ⏳ This version treats two different `Date`s as equal (they have no own keys). Handling them needs prototype knowledge — after Saturday.

---

### Task A3 — Flatten a nested object (dot-notation keys)

**Reported in:** Growtomation (Frontend Developer, 2025) and Wingify (Frontend Engineer); a widely shared article also calls it a common "write a function to flatten an object" interview question.

**How it's asked:** *"Flatten a nested object into one level, joining the keys with dots."*

```js
flatten({ user: { profile: { name: "John" } }, tool: "pen" });
// { "user.profile.name": "John", "tool": "pen" }
```

<details>
<summary>Solution</summary>

```js
function flatten(obj, parentKey = "", result = {}) {
  const keys = Object.keys(obj);
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i];
    const path = parentKey === "" ? key : parentKey + "." + key;
    const value = obj[key];

    if (value !== null && typeof value === "object" && !Array.isArray(value)) {
      flatten(value, path, result);        // a nested plain object: go deeper
    } else {
      result[path] = value;                // anything else is a leaf
    }
  }
  return result;
}

flatten({ a: 1, b: { c: 2, d: { e: 3 } } });
// { a: 1, "b.c": 2, "b.d.e": 3 }

flatten({ a: null, b: { c: null } });
// { a: null, "b.c": null }          ← null is a leaf (typeof null is "object" — the Week 1 quirk)
```

The input is never mutated; the result object is passed down through the recursion.
</details>

<details>
<summary>My Solution</summary>

```js
function flatten(obj, parentKey = null){
    let result = {};
    let keys = Object.keys(obj);

    for(const key of keys){
        if(typeof obj[key] !== "object" || obj[key] === null){
            if(parentKey !== null){
                let finalKey = parentKey + "." + key;
                result[finalKey] = obj[key];
            }
            else{
              result[key] = obj[key]
            }
        }
        else{
            const flatObj = flatten(obj[key],parentKey ? parentKey + "." + key : key);
            result = {...result, ...flatObj}
        }
    }

    return result;
};
```

The input is never mutated; the result object is passed down through the recursion.
</details>





**Follow-ups interviewers add:**
- **"What about arrays?"** In this version an array is a leaf. If they want indexed keys (`a.0`, `a.1.b`), drop the `!Array.isArray(value)` check — `Object.keys` on an array gives its indexes, so `{ a: [1, { b: 2 }] }` becomes `{ "a.0": 1, "a.1.b": 2 }`.
- **"What about `null` values?"** Must be treated as leaves — forgetting the `value !== null` check crashes on `Object.keys(null)`.
- **"What happens to empty objects?"** `{ a: {} }` produces no keys for `a` (it vanishes). Decide whether to keep `a: {}` and say so.
- **"Can the separator be configurable?"** Add a `separator = "."` parameter and use it when joining.
- **"Write the reverse (`unflatten`)."** Split each key on the separator and rebuild nested objects.
- **"What if a key itself contains a dot?"** The result becomes ambiguous (`"a.b"` could be one key or a path). A real implementation needs an escape rule.
- **"Circular references?"** This version recurses forever (`RangeError`). Same registry idea as deep clone, or throw a clear error.

---

### Task A4 — Implement `JSON.stringify`

**Reported in:** Canva (Glassdoor, 2023) and RingCentral (round 2); prep articles also note it being asked more recently.

**How it's asked:** *"Code `JSON.stringify` from scratch."* Interviewers add features step by step, so edge cases matter more than the basic version.

**Cover at least:** `null`, booleans, numbers, strings, arrays, nested objects; skip `undefined`/functions/symbols in objects (they become `null` in arrays); `NaN`/`Infinity` become `null`; circular references throw a `TypeError`.

<details>
<summary>Solution</summary>

```js
function quote(s) {
  let out = '"';
  for (let i = 0; i < s.length; i++) {
    const ch = s[i];
    out += ch === '"' ? '\\"' : ch === "\\" ? "\\\\" : ch;      // escape quotes and backslashes
  }
  return out + '"';
}

function stringify(value, ancestors = [], depth = 0) {
  if (value === null) return "null";

  const type = typeof value;
  if (type === "string") return quote(value);
  if (type === "number") return (value === value && value !== Infinity && value !== -Infinity) ? String(value) : "null";
  if (type === "boolean") return String(value);
  if (type === "bigint") throw new TypeError("Do not know how to serialize a BigInt");

  // not representable: the caller decides (skipped in objects, "null" in arrays)
  if (type === "undefined" || type === "function" || type === "symbol") return undefined;

  // objects and arrays: a cycle means "this object is already one of my ancestors"
  for (let i = 0; i < depth; i++) {
    if (ancestors[i] === value) throw new TypeError("Converting circular structure to JSON");
  }
  ancestors[depth] = value;

  if (Array.isArray(value)) {
    let out = "";
    for (let i = 0; i < value.length; i++) {
      const item = stringify(value[i], ancestors, depth + 1);
      out += (i > 0 ? "," : "") + (item === undefined ? "null" : item);
    }
    return "[" + out + "]";
  }

  let out = "";
  let first = true;
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) {
    const item = stringify(value[keys[i]], ancestors, depth + 1);
    if (item === undefined) continue;                          // skip undefined / functions / symbols
    out += (first ? "" : ",") + quote(keys[i]) + ":" + item;
    first = false;
  }
  return "{" + out + "}";
}

stringify({ a: 1, b: [1, { c: 2 }], d: undefined, f() {} });   // '{"a":1,"b":[1,{"c":2}]}'
stringify([undefined, NaN, "x"]);                              // '[null,null,"x"]'
stringify(undefined);                                          // undefined (not a string!)
```

This matches the real `JSON.stringify` on all the inputs above, plus `Map`/`Set`/`RegExp`/`Error` (all `{}`), shared (non-circular) references, and key order.

Why `ancestors` and not the `visited` registry from deep clone? **Sharing is allowed here** — the same object may appear twice. Only an object that contains **itself** is an error, so we track the objects on the *current path*, not every object ever seen.
</details>

**Follow-ups interviewers add:**
- **"What about `undefined`, functions and symbols?"** Skipped in objects, `null` in arrays, `undefined` at the top level.
- **"What about `NaN` and `Infinity`?"** `null`.
- **"Circular references?"** Throw a `TypeError` (done with `ancestors`).
- **"The same object referenced twice — is that circular?"** No; it must serialise normally.
- **"What about `Date`?"** Objects with a `toJSON` method are serialised through it. Add at the top: `if (value !== null && typeof value === "object" && typeof value.toJSON === "function") value = value.toJSON();`.
- **"Escaping?"** Quotes and backslashes are handled; control characters (`\n`, `\t`, …) and Unicode escapes also need handling in a full version.
- **"Pretty-printing (the `indent` argument)?"** Add an indent string and a newline after each element, tracked with `depth`.
- **"Replacer function or array?"** The optional second argument that filters or transforms values.
- **"BigInt?"** Throws a `TypeError`, as in the real thing.
- **"Which keys are included?"** Own enumerable string keys, in `Object.keys` order.

---

# B. Additional tasks (from me)

These are natural variations of the standard questions. Each should take 10–20 minutes with the concepts you have already learned.

### Task B1 — `shallowEqual(a, b)`

Compare two objects **one level deep**: same keys, and each value strictly equal. (This is the comparison React uses to decide whether props changed; a candidate-shared "React.memo polyfill" asked in an interview is built on exactly this.)

<details>
<summary>Solution</summary>

```js
function shallowEqual(a, b) {
  if (Object.is(a, b)) return true;
  if (a === null || b === null || typeof a !== "object" || typeof b !== "object") return false;

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  for (let i = 0; i < keysA.length; i++) {
    const key = keysA[i];
    if (!Object.hasOwn(b, key) || !Object.is(a[key], b[key])) return false;
  }
  return true;
}

shallowEqual({ a: 1, b: "x" }, { a: 1, b: "x" });   // true
shallowEqual({ a: 1 }, { a: 1, b: 2 });             // false
shallowEqual({ a: {} }, { a: {} });                 // false  ← nested objects compare by reference
const inner = {};
shallowEqual({ a: inner }, { a: inner });           // true
```
</details>


<details>
<summary>My Solution</summary>

```js
function shallowEqual(a,b){
    //For 2 names sharing same objects in memory means assignment not copy so false.
    if(a === b) return false;

    let keys_first = Object.getOwnPropertyNames(a);
    let keys_second = Object.getOwnPropertyNames(b);

    //If number of keys are unequal that means shape is different so not a copy.
    if(keys_first.length !== keys_second.length) return false;

    for(const key of keys_first){
        //If a key exists in first object but not in second, not a copy, return false.
        if(!(keys_second.includes(key))) return false;

        //If value is primitive/reference type for key, just check for value match.
        if(a[key] !== b[key]) return false;
    };

    return true;
};


shallowEqual({ a: 1, b: "x" }, { a: 1, b: "x" });   // true
shallowEqual({ a: 1 }, { a: 1, b: 2 });             // false
shallowEqual({ a: {} }, { a: {} });                 // false  ← nested objects compare by reference
const inner = {};
shallowEqual({ a: inner }, { a: inner });           // true
```
</details>

**Follow-ups:**
- **"How is this different from `deepEqual`?"** It stops at one level; nested objects are compared by reference.
- **"Why `Object.is` and not `===`?"** So `NaN` equals `NaN`.
- **"Why is shallow comparison used for change detection?"** It is cheap, and it matches how immutable updates work: if you replace an object, its reference changes.

### Task B2 — `setIn(obj, path, value)`: copy only the path

Return a new structure where the value at `path` (an array of keys) is replaced, copying **only** the objects/arrays along that path. Everything else must stay **shared** with the original.

<details>
<summary>Solution</summary>

```js
function setIn(obj, path, value) {
  if (path.length === 0) return value;
  const [key, ...rest] = path;
  const copy = Array.isArray(obj) ? [...obj] : { ...obj };
  copy[key] = setIn(obj == null ? undefined : obj[key], rest, value);
  return copy;
}

const state = {
  user: { name: "A", address: { city: "J" }, tags: ["x", "y"] },
  items: [{ id: 1 }, { id: 2 }]
};

const s2 = setIn(state, ["user", "address", "city"], "K");
state.user.address.city;          // "J"            ← original untouched
s2.user.address.city;             // "K"
s2.user.tags === state.user.tags; // true           ← untouched branch is SHARED
s2.items === state.items;         // true

const s3 = setIn(state, ["items", 1, "id"], 99);
Array.isArray(s3.items);          // true           ← arrays stay arrays
s3.items[0] === state.items[0];   // true

setIn({}, ["a", "b"], 1);         // { a: { b: 1 } }   ← missing levels are created
```
</details>

**Follow-ups:**
- **"Why not deep clone and then change it?"** It copies everything and breaks reference-equality checks for the parts that didn't change.
- **"Which parts are shared afterwards?"** Every branch not on the path.
- **"What if a level on the path doesn't exist?"** It's created as an object.

### Task B3 — `safeClone(value)`

Use `structuredClone`, but if it throws a `DataCloneError` (for example because of a function), fall back to your `deepClone`. Any other error must be rethrown.

<details>
<summary>Solution</summary>

```js
function safeClone(value) {
  try {
    return structuredClone(value);
  } catch (e) {
    if (e && e.name === "DataCloneError") return deepClone(value);   // deepClone from Task A1
    throw e;                                                        // not ours — rethrow
  }
}

const data = { f() { return 1; }, n: { v: 1 }, list: [1, 2] };
const c = safeClone(data);
c.f === data.f;          // true   ← the function is shared (it can't be cloned)
c.n !== data.n;          // true   ← nested object cloned
c.list !== data.list;    // true   ← nested array cloned
```
</details>

**Follow-ups:**
- **"Why rethrow other errors?"** Swallowing everything would hide real bugs (for example a `RangeError` from extremely deep data).
- **"What are you giving up with the fallback?"** Functions are shared, and `deepClone` only understands plain data.

### Task B4 — Find the bugs in this deep clone

An interviewer shows you this and asks: *"This is meant to clone objects with circular references. What's wrong with it?"*

```js
function deepClone(value, visited = new WeakMap()) {
  if (typeof value !== "object") return value;
  if (visited.has(value)) return visited.get(value);

  const result = Array.isArray(value) ? [] : {};
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) {
    result[keys[i]] = deepClone(value[keys[i]]);
  }
  visited.set(value, result);
  return result;
}
```

<details>
<summary>Solution</summary>

There are **three** bugs:

1. **`null` is not handled.** `typeof null` is `"object"`, so `null` passes the first check and `Object.keys(null)` throws a `TypeError`. Fix: `if (typeof value !== "object" || value === null) return value;`
2. **`visited` is not passed to the recursive call.** Every call starts with a fresh, empty registry, so the cycle is never detected (`RangeError`) and shared references are cloned twice. Fix: `deepClone(value[keys[i]], visited)`.
3. **The clone is registered after filling it.** The cycle is met *while* filling, when the registry doesn't know about this object yet — still a `RangeError`. Fix: call `visited.set(value, result)` **before** the loop.

```js
function deepClone(value, visited = new WeakMap()) {
  if (typeof value !== "object" || value === null) return value;
  if (visited.has(value)) return visited.get(value);

  const result = Array.isArray(value) ? [] : {};
  visited.set(value, result);                              // register BEFORE filling

  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) {
    result[keys[i]] = deepClone(value[keys[i]], visited);  // pass visited down
  }
  return result;
}
```

Notice how the buggy version **still works on ordinary data without `null` or cycles** — which is exactly why such bugs survive casual testing.
</details>

**Follow-ups:**
- **"Which bug breaks ordinary (non-circular) data?"** Only the first one — any `null` inside the data.
- **"How would you have caught these?"** Test cases: an object containing `null`, a cycle (`a.self = a`), and a shared reference (`{ x: s, y: s }`).

---

# Reported questions that need later topics

These are asked for this topic too, but they need things we haven't covered yet. They will go on the list when we reach them:

- **Deep clone that also handles `Date`, `RegExp`, `Map`, `Set` and class instances** (the harder "Deep Clone II" level) — after Saturday (prototypes).
- **Undo / redo history** built from snapshot copies (reported as a class or React-hook exercise) — after array methods (Friday) and classes (Saturday).
- **A `React.memo` polyfill** (reported as asked in an interview) — Task B1's `shallowEqual` is its core; the React part comes with React prep.

---

## Output Questions

Predict first, then open the answer. Environment: Node 22 / a current browser.

---

**Q1 — Assignment, spread, and identity**
```js
const a = { x: 1 };
const b = a;
const c = { ...a };
b.x = 2;
c.x = 3;
console.log(a.x, b === a, c === a);
```
<details>
<summary>Answer</summary>

`2 true false`. `b` is the same object (changing it changes `a`); `c` is a separate object (changing it doesn't).
</details>

---

**Q2 — Shallow copy of an array of arrays**
```js
const matrix = [[1, 2], [3]];
const copy = [...matrix];
copy[0].push(9);
copy.push([4]);
console.log(matrix, copy.length);
```
<details>
<summary>Answer</summary>

`[[1, 2, 9], [3]] 3`. The inner array is shared (so `9` appears in `matrix`), but pushing a new element onto the outer copy doesn't affect `matrix`.
</details>

---

**Q3 — Mutation vs reassignment on a shallow copy**
```js
const original = { n: { v: 1 } };
const copy = { ...original };

copy.n.v = 5;
console.log(original.n.v);

copy.n = { v: 100 };
console.log(original.n.v);
```
<details>
<summary>Answer</summary>

`5`, then `5`. The first line mutates the shared inner object (original changes). The second *replaces* `copy.n` with a new object, which has no effect on the original.
</details>

---

**Q4 — What does JSON do to this?**
```js
const original = { a: 1, u: undefined, f() {}, n: NaN, d: new Date(0) };
console.log(JSON.parse(JSON.stringify(original)));
```
<details>
<summary>Answer</summary>

`{ a: 1, n: null, d: "1970-01-01T00:00:00.000Z" }`. `u` and `f` vanish, `NaN` becomes `null`, and the `Date` becomes a string.
</details>

---

**Q5 — JSON and arrays**
```js
console.log(JSON.parse(JSON.stringify([undefined, function () {}, Symbol("s"), NaN])));
console.log(JSON.stringify([1, , 3]));
```
<details>
<summary>Answer</summary>

`[null, null, null, null]` and `"[1,null,3]"`. In arrays, unrepresentable values become `null` (removing them would shift positions), and holes become `null`.
</details>

---

**Q6 — Map, Set, RegExp, Error**
```js
console.log(JSON.parse(JSON.stringify({
  r: /x/g, m: new Map([[1, 2]]), s: new Set([1]), e: new Error("boom")
})));
```
<details>
<summary>Answer</summary>

`{ r: {}, m: {}, s: {}, e: {} }`. Their data lives in hidden internal slots (or non-enumerable properties), not in ordinary enumerable ones.
</details>

---

**Q7 — Which of these throw?**
```js
const circ = {}; circ.self = circ;
try { JSON.stringify({ b: 1n }); } catch (e) { console.log("A", e.name); }
try { JSON.stringify(circ); }      catch (e) { console.log("B", e.name); }
try { JSON.stringify({ n: NaN }); console.log("C ok"); } catch (e) { console.log("C", e.name); }
```
<details>
<summary>Answer</summary>

`A TypeError`, `B TypeError`, `C ok`. BigInt and cycles throw; `NaN` is silently turned into `null`.
</details>

---

**Q8 — `undefined` through JSON**
```js
console.log(JSON.stringify(undefined));
try { JSON.parse(JSON.stringify(undefined)); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`undefined` (not a string), then `SyntaxError`. `JSON.stringify(undefined)` returns the value `undefined`, and `JSON.parse(undefined)` tries to parse the text `"undefined"`, which isn't valid JSON.
</details>

---

**Q9 — Shared references through JSON vs `structuredClone`**
```js
const shared = { v: 1 };
const o = { a: shared, b: shared };

const viaJson = JSON.parse(JSON.stringify(o));
const viaSC = structuredClone(o);
console.log(o.a === o.b, viaJson.a === viaJson.b, viaSC.a === viaSC.b);
```
<details>
<summary>Answer</summary>

`true false true`. JSON writes the object twice and builds two; `structuredClone` keeps the relationship (one object, referenced twice).
</details>

---

**Q10 — The Date bug**
```js
const copy = JSON.parse(JSON.stringify({ createdAt: new Date(0) }));
console.log(typeof copy.createdAt);
try { copy.createdAt.getTime(); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`"string"`, then `TypeError`. The `Date` was serialised to an ISO string and parsing doesn't revive it.
</details>

---

**Q11 — `toJSON`**
```js
console.log(JSON.stringify({ a: { toJSON() { return "custom"; } }, d: new Date(0) }));
```
<details>
<summary>Answer</summary>

`{"a":"custom","d":"1970-01-01T00:00:00.000Z"}`. `JSON.stringify` calls an object's `toJSON` if it has one; `Date` provides one that returns an ISO string.
</details>

---

**Q12 — Getters and hidden properties through JSON**
```js
const o = { a: 1 };
Object.defineProperty(o, "g", { get() { return "got"; }, enumerable: true });
Object.defineProperty(o, "h", { value: 2, enumerable: false });
console.log(JSON.stringify(o));
```
<details>
<summary>Answer</summary>

`{"a":1,"g":"got"}`. The getter ran and its value was written; the non-enumerable `h` was skipped.
</details>

---

**Q13 — `structuredClone` keeps real types**
```js
const c = structuredClone({ d: new Date(0), m: new Map([[1, 2]]), s: new Set([1]) });
console.log(c.d.getTime(), c.m.get(1), c.s.size);
```
<details>
<summary>Answer</summary>

`0 2 1`. The `Date`, `Map` and `Set` come back as real `Date`, `Map` and `Set` objects.
</details>

---

**Q14 — Special values**
```js
const c = structuredClone({ u: undefined, n: NaN, i: Infinity, z: -0, b: 10n });
console.log("u" in c, c.n, c.i, Object.is(c.z, -0), typeof c.b);
```
<details>
<summary>Answer</summary>

`true NaN Infinity true "bigint"`. All preserved; JSON would have dropped or changed every one of them (and thrown on the BigInt).
</details>

---

**Q15 — Functions and `structuredClone`**
```js
try { structuredClone({ f() {} }); } catch (e) { console.log(e.name); }
console.log(JSON.parse(JSON.stringify({ f() {}, a: 1 })));
```
<details>
<summary>Answer</summary>

`DataCloneError`, then `{ a: 1 }`. `structuredClone` refuses functions loudly; the JSON trick silently drops them.
</details>

---

**Q16 — Cycles**
```js
const a = { name: "a" };
a.self = a;
const c = structuredClone(a);
console.log(c !== a, c.self === c, c.self === a);
```
<details>
<summary>Answer</summary>

`true true false`. The clone is a new object, its `self` points to the clone itself, not to the original.
</details>

---

**Q17 — Getters and hidden properties through `structuredClone`**
```js
const o = { a: 1 };
Object.defineProperty(o, "g", { get() { return "got"; }, enumerable: true });
Object.defineProperty(o, "h", { value: 2, enumerable: false });

const c = structuredClone(o);
console.log(Object.keys(c), typeof Object.getOwnPropertyDescriptor(c, "g").get, c.h);
```
<details>
<summary>Answer</summary>

`["a", "g"] "undefined" undefined`. `g` became a plain data property holding the getter's value (so its descriptor has no `get`), and `h` was dropped.
</details>

---

**Q18 — Errors**
```js
const e = new TypeError("boom");
e.code = 5;
const c = structuredClone(e);
console.log(c.name, c.message, c.code);
```
<details>
<summary>Answer</summary>

`"TypeError" "boom" undefined`. The error's type and message survive; the custom `code` property does not.
</details>

---

**Q19 — Holes: spread vs JSON vs `structuredClone`**
```js
const a = [1, , 3];
console.log(1 in [...a], JSON.stringify(a), 1 in structuredClone(a));
```
<details>
<summary>Answer</summary>

`true "[1,null,3]" false`. Spread fills the hole with `undefined`; JSON writes `null`; `structuredClone` keeps it a hole.
</details>

---

**Q20 — Freezing**
```js
const c = structuredClone(Object.freeze({ a: { b: 1 } }));
console.log(Object.isFrozen(c), Object.isFrozen(c.a));
```
<details>
<summary>Answer</summary>

`false false`. Clones are ordinary, unfrozen objects.
</details>

---

**Q21 — The first draft and a shared reference**
```js
function clone(value) {
  if (value === null || typeof value !== "object") return value;
  if (Array.isArray(value)) {
    const out = [];
    for (let i = 0; i < value.length; i++) out[i] = clone(value[i]);
    return out;
  }
  const out = {};
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) out[keys[i]] = clone(value[keys[i]]);
  return out;
}

const shared = { v: 1 };
const o = { x: shared, y: shared };
const c = clone(o);
console.log(c.x === c.y, c.x !== shared);
```
<details>
<summary>Answer</summary>

`false true`. The original has **one** object referenced twice; the first draft clones it once per path, so the copy has **two** separate objects. Neither is the original. Adding the `seen` registry (Part 7, Step 2) makes both paths resolve to the same clone.
</details>

---

**Q22 — The same draft on a cycle**
```js
const a = {};
a.self = a;
try { clone(a); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`RangeError`. `clone(a)` → `clone(a.self)` → `clone(a.self.self)` … forever, until the call stack overflows.
</details>

---

**Q23 — Registering too late**
```js
function clone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== "object") return value;
  if (seen.has(value)) return seen.get(value);
  const out = Array.isArray(value) ? [] : {};
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) out[keys[i]] = clone(value[keys[i]], seen);
  seen.set(value, out);
  return out;
}

const a = {}; a.self = a;
try { clone(a); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`RangeError`. `seen.set` happens **after** the loop, so while cloning `a.self` the registry still doesn't know about `a`. Move `seen.set(value, out)` to before the loop.
</details>

---

**Q24 — Functions in a hand-written clone**
```js
const src = { f() { return 1; }, n: { v: 1 } };
const c = deepClone(src);         // the version from Part 7
console.log(c.f === src.f, c.n === src.n);
```
<details>
<summary>Answer</summary>

`true false`. Functions are returned as-is, so the clone **shares** them; the nested object is cloned.
</details>

---

**Q25 — Forgetting `null`**
```js
function broken(value) {
  if (typeof value !== "object") return value;
  const out = {};
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) out[keys[i]] = broken(value[keys[i]]);
  return out;
}
try { broken({ a: null }); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`TypeError`. `typeof null` is `"object"` (Week 1), so `null` gets past the check and `Object.keys(null)` throws.
</details>

---

**Q26 — Very deep data**
```js
let chain = {};
const root = chain;
for (let i = 0; i < 100000; i++) { chain.next = {}; chain = chain.next; }

try { structuredClone(root); } catch (e) { console.log("A", e.name); }
try { JSON.stringify(root); }  catch (e) { console.log("B", e.name); }
```
<details>
<summary>Answer</summary>

In the Node version tested: `A RangeError` and `B RangeError`. Both overflow the stack at this depth; the exact limit depends on the engine. A hand-written **iterative** clone (Task 4) has no such limit.
</details>

---

**Q27 — Forgetting to pass `seen` down**
```js
function clone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== "object") return value;
  if (seen.has(value)) return seen.get(value);
  const out = Array.isArray(value) ? [] : {};
  seen.set(value, out);
  const keys = Object.keys(value);
  for (let i = 0; i < keys.length; i++) out[keys[i]] = clone(value[keys[i]]);   // seen is not passed
  return out;
}

const a = {};
a.self = a;
try { clone(a); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`RangeError`. Every recursive call gets a **fresh, empty** registry (the default parameter), so `seen.has(a)` is never `true` on the way back in. The registry only works if the same one is handed down to every call.
</details>

---

**Q28 — A `Date` through the hand-written clone**
```js
const c = deepClone({ when: new Date(0) });      // the Part 7 version
console.log(c.when, Object.keys(c.when).length);
```
<details>
<summary>Answer</summary>

`{} 0`. A `Date` keeps its time in hidden internal data, not in own enumerable properties, so the object branch produces an empty object — silently. That is the limit stated at the top of Part 7: this version covers plain data only; `Date`, `Map`, `Set` and similar are handled in the full version after Saturday. For now, use `structuredClone` for them.
</details>

---

## What Interviewers Test

**1. Shallow vs deep, with a memory picture**
New outer object, shared inner references vs everything duplicated. Show it with `copy.n === original.n`.

**2. Which tool makes which copy**
Spread / `Object.assign` → shallow. JSON trick, `structuredClone`, custom clone → deep.

**3. The JSON trick's failure list**
Be able to recite: `undefined`/functions/symbols dropped, `NaN`/`Infinity` → `null`, `Date` → string, Map/Set/RegExp/Error → `{}`, BigInt and cycles throw, shared references duplicated. And the key point: **most failures are silent**.

**4. `structuredClone` strengths and limits**
Handles Date/Map/Set/RegExp, `undefined`, `NaN`, BigInt, cycles, shared references. Throws on functions, symbols and WeakMap. Loses custom Error properties; flattens getters; not frozen.

**5. Circular references**
Why naive recursion overflows, why JSON throws, and the **registry** fix — registered *before* recursing.

**6. Write `deepClone`**
Primitives/null/functions → arrays → plain objects → circular and shared references via a registry registered *before* filling. And **say what you don't handle** (Date, Map, Set and inherited behaviour wait for Saturday).

**7. "Do you really need a deep copy?"**
The SDE2 answer: usually copy only the path that changes (`setIn`, spread per level); deep clone breaks reference-equality and costs more.

**8. Depth and performance**
Recursion limits; iterative clone; don't clone in hot paths.

---

## Mental Models

**Assignment = a second name tag on the same object.**

**Shallow copy = a new folder containing the same documents.**
Rearrange the folder freely; edit a document and everyone holding that document sees it.

**Deep copy = a new folder with a photocopy of every document, including those in sub-folders.**

**The JSON trick = sending a fax.**
Only plain text survives. Dates, gaps, Maps and functions don't make it, and nothing warns you.

**`structuredClone` = a careful photocopier.**
Copies far more kinds of documents and keeps the cross-references between them. But it refuses to copy a machine (a function).

**The `seen` registry = a coat-check ticket.**
Every original gets a ticket (its clone). Hand over the same original again and you get the same clone — which ends cycles and keeps sharing intact. Hand over the ticket **before** you start filling the clone.

**Copy the path, share the rest.**
To change one nested value, copy only the folders you walk through. Everything else stays shared — cheap, and exactly what reference-comparing frameworks want.

---

*Previous: Spread / Rest — clone and merge, rest in function parameters, rest vs spread*
*Next: Array Methods — `map`, `filter`, `reduce`, `find`, `findIndex`, `some`, `every`, `flat`, `flatMap`*