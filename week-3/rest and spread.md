# Spread / Rest — Copy, Merge, Collect

---

## The Big Idea

JavaScript has one symbol, `...` (three dots), that does **two opposite jobs**:

```
SPREAD  →  EXPANDS a collection into separate items
REST    →  COLLECTS separate items into a collection
```

```js
const nums = [1, 2, 3];

Math.max(...nums);                 // SPREAD: becomes Math.max(1, 2, 3)

function sum(...n) { /* ... */ }   // REST:   sum(1, 2, 3) → n is [1, 2, 3]
```

Same three dots. Which job it does depends only on **where** it is written. You'll learn each job separately, then the rule that tells them apart (Part 7).

> **Today's map:**
> spread in arrays → spread in objects → spread in calls → the "shallow copy" catch →
> rest parameters → rest in destructuring → rest vs spread → gotchas

> **Thursday** covers copying in depth (shallow vs deep, `JSON` pitfalls, `structuredClone`). Today you meet *shallow* copies, because spread creates them.

---

## Part 1 — Spread With Arrays

Spread takes an array and **lays its elements out one by one** wherever you write it.

```js
const a = [1, 2];
const b = [3, 4];

[...a, ...b];            // [1, 2, 3, 4]
```

### Cloning an array

```js
const original = [1, 2, 3];
const copy = [...original];

copy.push(4);

original;                // [1, 2, 3]       ← untouched
copy;                    // [1, 2, 3, 4]
original === copy;       // false           ← two different arrays
```

You get a **new array** with the same elements. (Contrast with `const copy = original`, which copies only the *reference* — Week 1.)

### Merging arrays

```js
const merged = [...a, ...b];        // [1, 2, 3, 4]
const twice  = [...a, ...a];        // [1, 2, 1, 2]    ← duplicates are kept, order is preserved
```

### Inserting while building

You can mix spread with ordinary values, anywhere:

```js
[0, ...a, 99];           // [0, 1, 2, 99]
[...a, "end"];           // [1, 2, "end"]
["start", ...a];         // ["start", 1, 2]
```

This is how you **add an item without changing the original array**:

```js
const items = [1, 2];
const withThree = [...items, 3];     // new array; items is still [1, 2]
```

### Spreading a string

A string can be spread too — you get its characters:

```js
[..."abc"];              // ["a", "b", "c"]
```

Spread walks a string **by character (code point)**, not by raw storage unit — which matters for emoji:

```js
const s = "a😀";
s.length;                // 3   ← "length" counts storage units; the emoji takes two
[...s].length;           // 2   ← spread sees two characters: "a" and "😀"
```

### Empty slots become `undefined`

```js
const withHole = [1, , 3];
const copy = [...withHole];

1 in withHole;           // false   ← a real hole
1 in copy;               // true    ← spread filled it with undefined
copy;                    // [1, undefined, 3]
```

### What can be spread into an array?

Array spread works on anything **iterable** — arrays and strings are. Plain objects, numbers, `null`, `undefined` and booleans are **not**:

```js
[...5];                  // TypeError: ... is not iterable
[...{ a: 1 }];           // TypeError: ... is not iterable
[...null];               // TypeError: ... is not iterable
[...undefined];          // TypeError: ... is not iterable
[...false];              // TypeError: ... is not iterable
```

(What "iterable" means exactly is a Week 5 topic. For now: arrays and strings qualify.)

---

## Part 2 — Spread With Objects

Spread in an **object literal** copies an object's properties into the new object.

```js
const user = { name: "Asha", age: 25 };

const copy = { ...user };            // { name: "Asha", age: 25 }
copy === user;                       // false  ← a new object
```

### Merging: later values win

```js
const a = { x: 1, y: 2 };
const b = { y: 9, z: 3 };

{ ...a, ...b };          // { x: 1, y: 9, z: 3 }   ← b's y beat a's y
{ ...b, ...a };          // { y: 2, z: 3, x: 1 }   ← now a's y wins
```

Properties are applied **left to right**; if a key appears again, the later one overwrites the value. (Key order follows Monday's rule: a key keeps the position where it first appeared.)

### Overriding one property without mutating the original

```js
const user = { name: "Asha", age: 25 };

{ ...user, age: 26 };    // { name: "Asha", age: 26 }   ← age replaced
{ age: 26, ...user };    // { age: 25, name: "Asha" }   ← user's age came LATER and won!
```

**Where you put the new value matters.** To override, put your value **after** the spread.

### Defaults + options — the most common use

```js
const defaults = { theme: "light", size: 10 };
const options  = { size: 20 };

const settings = { ...defaults, ...options };
// { theme: "light", size: 20 }
```

### Adding a computed key

```js
const key = "dyn";
{ ...{ a: 1 }, [key]: 2 };           // { a: 1, dyn: 2 }
```

(Computed property names — Monday.)

### Conditionally adding a property

```js
const isAdmin = true;

const user = {
  name: "Asha",
  ...(isAdmin && { role: "admin" })
};
// isAdmin true  → { name: "Asha", role: "admin" }
// isAdmin false → { name: "Asha" }
```

This works because object spread **ignores** values that aren't objects (`false`, `0`, `null`, `undefined`).

⚠️ This does **not** work the same way for arrays:

```js
{ ...(false && { x: 1 }) };          // {}              ✅ ignored
[...(false && [1])];                 // TypeError       ❌ false is not iterable
[...(false ? [1] : [])];             // []              ✅ use a ternary with an empty array
```

### What exactly gets copied? (own vs inherited, from Monday)

Object spread copies the source's **own, enumerable** properties. That has consequences:

```js
// 1. Inherited properties are NOT copied
const parent = { inh: 1 };
const child = Object.create(parent);
child.own = 2;

const copy = { ...child };
copy;                    // { own: 2 }
copy.inh;                // undefined   ← the inherited property was left behind
```

```js
// 2. Non-enumerable properties are NOT copied
const o = { a: 1 };
Object.defineProperty(o, "hidden", { value: 2, enumerable: false });

Object.keys({ ...o });   // ["a"]
```

```js
// 3. Getters are RUN, and the result is copied as a plain value
let reads = 0;
const src = { a: 1 };
Object.defineProperty(src, "g", { get() { reads++; return "got"; }, enumerable: true });

const c = { ...src };
Object.getOwnPropertyDescriptor(c, "g");
// { value: "got", writable: true, enumerable: true, configurable: true }
reads;                   // 1     ← the getter ran once; the copy has no getter
```

```js
// 4. Attributes reset to the defaults: everything true
const frozen = Object.freeze({ a: 1 });
const unfrozen = { ...frozen };

unfrozen.a = 2;          // ✅ works
Object.isFrozen(unfrozen);   // false
```

So spread is **not an exact clone**: it copies *values* of own enumerable properties into a fresh ordinary object. Descriptors (Monday), getters/setters, hidden properties and the inheritance link are not preserved.

### Odd inputs for object spread

Object spread is forgiving:

```js
{ ...null };             // {}
{ ...undefined };        // {}
{ ...5 };                // {}
{ ...true };             // {}
{ ..."ab" };             // { 0: "a", 1: "b" }    ← a string has index keys
{ ...[10, 20] };         // { 0: 10, 1: 20 }      ← so does an array
```

No error for `null`/`undefined` — they simply contribute nothing.

---

## Part 3 — Spread in Function Calls

Spread turns an array into **separate arguments**.

```js
const nums = [3, 9, 1];

Math.max(...nums);                   // 9     — same as Math.max(3, 9, 1)
```

Without spread you would be passing *one* argument (the array itself):

```js
Math.max(nums);                      // NaN   — the array is one argument
```

### Mixing with normal arguments, and multiple spreads

```js
Math.max(1, ...[2, 9], 4);           // 9
Math.max(...[1, 2], ...[7]);         // 7
Math.max(...[]);                     // -Infinity   ← no arguments at all
```

```js
function f(a, b, c, d) { return [a, b, c, d]; }

f(...[1, 2], 3, ...[4]);             // [1, 2, 3, 4]
f(...[1]);                           // [1, undefined, undefined, undefined]
f(..."xyz");                         // ["x", "y", "z", undefined]
```

### Spread replaces `apply` (Week 2)

In Week 2 you used `apply` to pass an array as separate arguments:

```js
Math.max.apply(null, nums);          // old way
Math.max(...nums);                   // modern way — same result
```

Spread is shorter, doesn't need a `this` argument, and can be mixed with other arguments.

### Limit: don't spread huge arrays into a call

Every element becomes a separate argument, and engines limit how many arguments (and how much stack) a call can use:

```js
Math.max(...new Array(125000).fill(1));    // worked in the Node version tested
Math.max(...new Array(200000).fill(1));    // RangeError: Maximum call stack size exceeded
```

The exact limit depends on the engine and its settings. For very large arrays, use a loop instead (Implementation Task 2).

---

## Part 4 — Spread Makes *Shallow* Copies

This is the most important caveat. You met it in Monday's Part 3 and Tuesday's Part 7:
**spread copies one level.** Anything *inside* that is an object or array is **shared**, not duplicated.

```js
const o = { n: { v: 1 }, arr: [1] };
const c = { ...o };

c.n.v = 2;               // changes the inner object
c.arr.push(2);           // changes the inner array

o.n.v;                   // 2       ← the original changed!
o.arr;                   // [1, 2]  ← the original changed!
c.n === o.n;             // true    ← same inner object
c === o;                 // false   ← only the outer layer is new
```

```
 o ─────▶ { n ───┐, arr ───┐ }         c ─────▶ { n ───┐, arr ───┐ }
                 ▼          ▼                           ▼          ▼
            { v: 1 }     [ 1 ]    ◀──── the SAME inner objects ────┘
```

Same for arrays of objects:

```js
const a = [{ v: 1 }];
const b = [...a];

b[0].v = 9;
b.push({ v: 3 });

a[0].v;                  // 9     ← shared element
a.length;                // 1     ← but pushing to b did not affect a
```

**Rule:** replacing or adding at the *top level* of the copy is safe; mutating something *nested* is not.

### The fix for everyday code: copy every level you change

If you want to update a nested value without touching the original, spread **each level on the path**:

```js
const state = {
  user: { name: "Asha", address: { city: "Jaipur", pin: 302001 } },
  items: [1, 2]
};

const next = {
  ...state,
  user: {
    ...state.user,
    address: { ...state.user.address, city: "Kota" }
  },
  items: [...state.items, 3]
};

state.user.address.city;      // "Jaipur"        ← original untouched
next.user.address.city;       // "Kota"
state.items;                  // [1, 2]
next.items;                   // [1, 2, 3]
```

Parts you did **not** rebuild are shared between old and new — that's fine and efficient:

```js
const same = { ...state, items: [...state.items, 4] };
same.user === state.user;     // true     ← untouched branch is reused
```

This "copy the path, share the rest" idea is how immutable updates work. Frameworks like React detect changes by comparing **references**, so a new object at each changed level is what signals "this changed". (That's React-specific; the copying behaviour is plain JavaScript.)

### Merging nested objects with spread replaces whole branches

```js
const defaults = { ui: { theme: "light", size: 1 }, lang: "en" };
const opts     = { ui: { theme: "dark" } };

{ ...defaults, ...opts };
// { ui: { theme: "dark" }, lang: "en" }     ← ui was REPLACED; size is gone
```

Spread merges only the top level. To keep `size`, merge that level too:

```js
{ ...defaults, ui: { ...defaults.ui, ...opts.ui } };
// { ui: { theme: "dark", size: 1 }, lang: "en" }
```

> For a true, fully independent copy (deep copy), see Thursday.

---

## Part 5 — Rest Parameters

Rest in a function's **parameter list** collects the extra arguments into an array.

```js
function f(first, ...rest) {
  return [first, rest];
}

f(1, 2, 3);              // [1, [2, 3]]
f(1);                    // [1, []]         ← nothing extra → empty array
f();                     // [undefined, []]
```

### Rules

```
• ...rest must be the LAST parameter.
• There can be only ONE rest parameter.
• It can't have a default value.
• rest is a REAL array (Array.isArray(rest) is true).
```

```js
function f(...a, b) {}          // SyntaxError: Rest parameter must be last formal parameter
function f(...a, ...b) {}       // SyntaxError: Rest parameter must be last formal parameter
function f(...a = []) {}        // SyntaxError: Rest parameter may not have a default initializer
```

These are *early errors*: the whole script fails to run, before any line executes.

### Variadic functions: any number of arguments

```js
const sum = (...nums) => {
  let total = 0;
  for (let i = 0; i < nums.length; i++) total += nums[i];
  return total;
};

sum();                   // 0
sum(1, 2, 3);            // 6
```

```js
function log(level, ...messages) {
  console.log("[" + level + "]", messages);
}

log("warn", "disk", "almost", "full");    // [warn] ["disk", "almost", "full"]
```

### Rest with an arrow function

Arrow functions have no `arguments` of their own (Week 2), so rest is how they accept "all the arguments":

```js
const count = (...args) => args.length;
count(1, 2, 3);          // 3
```

### Rest combined with a destructured parameter

```js
function f({ a, b = 2 } = {}, ...others) {
  return [a, b, others];
}

f({ a: 1 }, 7, 8);       // [1, 2, [7, 8]]
f();                     // [undefined, 2, []]
```

### `fn.length` ignores the rest parameter

Recall from Week 2: `length` counts parameters before the first default or rest.

```js
function a(x, ...r) {}
const b = (...r) => {};
function c(x, y = 1, ...r) {}

a.length;                // 1
b.length;                // 0
c.length;                // 1
```

This is why a generic `curry` can't tell the real arity of a variadic function.

### The older way: the `arguments` object

Before rest parameters, every regular function had a built-in `arguments` object holding all the arguments:

```js
function f() {
  return [typeof arguments, Array.isArray(arguments), arguments.length, arguments[1]];
}

f(1, 2, 3);              // ["object", false, 3, 2]
```

`arguments` is **array-like**: it has numeric indexes and a `length`, but it is **not an array** — it has none of the array methods you'll meet on Friday.

Facts about `arguments`:

```js
function g(a, b) {}
g.length;                // 2    ← declared parameters

function h(a, b) { return arguments.length; }
h(1);                    // 1    ← arguments counts what was actually PASSED
h(1, 2, 3);              // 3
```

```js
// Arrow functions have no OWN arguments — they see the enclosing function's (Week 2):
function outer() {
  const a = () => arguments[0];
  return a("inner");
}
outer("outer");          // "outer"
```

**The sloppy-mode aliasing quirk (SDE2 favourite).** In sloppy mode, when a function has only simple parameters, `arguments[i]` and the named parameter are linked:

```js
function f(a) { arguments[0] = 9; return a; }
f(1);                    // 9      ← writing to arguments changed `a`!

function g(a) { a = 5; return arguments[0]; }
g(1);                    // 5      ← and vice versa
```

The link disappears in strict mode, or if the function uses default, rest, or destructured parameters:

```js
function h(a = 0) { arguments[0] = 9; return a; }
h(1);                    // 1      ← no link (default parameter)

function r(a, ...x) { arguments[0] = 9; return a; }
r(1);                    // 1      ← no link (rest parameter)
```

```js
"use strict";
function f(a) { arguments[0] = 9; return a; }
f(1);                    // 1      ← no link in strict mode
```

### Rest parameters vs `arguments`

| | Rest parameter `...args` | `arguments` |
|---|---|---|
| Type | real array | array-like object |
| Contains | only the *extra* arguments (after named ones) | *all* arguments |
| Arrow functions | ✅ works | ❌ no own `arguments` |
| Named in the signature | ✅ visible, self-documenting | ❌ hidden |
| Sloppy-mode aliasing with parameters | ❌ none | ⚠️ yes (simple params only) |
| Recommended today | ✅ yes | ❌ legacy |

### Where rest parameters show up in real code

You've already met functions that need them. Now their syntax makes sense:

```js
// Forward every argument to another function (a wrapper/decorator)
function withLogging(fn, log) {
  return function (...args) {
    log.push(args);
    return fn.apply(this, args);       // forward `this` AND all arguments
  };
}

// Pre-fill arguments (partial application, Week 2)
const partial = (fn, ...pre) => (...rest) => fn(...pre, ...rest);

// Chain functions (pipe, Week 2)
const pipe = (...fns) => x => {
  let acc = x;
  for (let i = 0; i < fns.length; i++) acc = fns[i](acc);
  return acc;
};

pipe(x => x + 1, x => x * 2)(3);       // 8
partial((a, b, c) => [a, b, c], 1, 2)(3);   // [1, 2, 3]
```

Notice both jobs together: `...args` **collects** on the way in; `fn(...pre, ...rest)` **spreads** on the way out.

⚠️ A wrapper that calls `fn(...args)` instead of `fn.apply(this, args)` **loses `this`**:

```js
function bad(fn) { return function (...args) { return fn(...args); }; }

const p = { n: "p", hi() { return this && this.n; } };
p.hi = bad(p.hi);
p.hi();                  // undefined  ← fn was called as a plain function
```

---

## Part 6 — Rest in Destructuring

Yesterday's topic left one thing out: collecting the **leftovers** when destructuring. That is the rest pattern.

### Array rest: `[first, ...others]`

```js
const [first, ...others] = [1, 2, 3];

first;                   // 1
others;                  // [2, 3]        ← a real array of everything after the first
```

```js
const [x, ...y] = [1];
x;                       // 1
y;                       // []            ← nothing left → empty array (not undefined)

const [h, ...tail] = "hey";
h;                       // "h"
tail;                    // ["e", "y"]
```

A common trick — rotate an array without changing it:

```js
const [head, ...others2] = [1, 2, 3];
[...others2, head];      // [2, 3, 1]
```

### Object rest: `{ a, ...others }`

Collects the **remaining own enumerable properties** into a **new plain object**:

```js
const { a, ...others } = { a: 1, b: 2, c: 3 };

a;                       // 1
others;                  // { b: 2, c: 3 }

const { x, ...none } = { x: 1 };
none;                    // {}               ← nothing left → empty object
```

### The "omit" pattern — remove a key without mutating

This is the most useful real-world use of object rest. Remove sensitive or unwanted keys:

```js
const user = { id: 1, name: "Asha", password: "secret", token: "t" };

const { password, token, ...publicUser } = user;

publicUser;              // { id: 1, name: "Asha" }
Object.keys(user);       // ["id", "name", "password", "token"]   ← original untouched
```

(You wrote an `omit` helper on Monday with a loop. This is the one-line version for keys known in advance.)

For a key held in a **variable**, combine with a computed key (Monday + Tuesday):

```js
const removeKey = (obj, key) => {
  const { [key]: _removed, ...rest } = obj;
  return rest;
};

removeKey({ a: 1, b: 2 }, "a");    // { b: 2 }
```

The unused name `_removed` is only there because a computed key needs a variable name.

### What object rest does and doesn't collect

```js
const parent = { inh: 1 };
const child = Object.create(parent);
child.a = 1;
child.b = 2;

const { a, ...r } = child;
r;                       // { b: 2 }
r.inh;                   // undefined   ← inherited properties are NOT collected

const o = { a: 1 };
Object.defineProperty(o, "h", { value: 2, enumerable: false });
o.z = 3;

const { a: aa, ...rr } = o;
rr;                      // { z: 3 }    ← non-enumerable "h" is NOT collected
```

Same rule as object spread: **own + enumerable only**.

### Nested rest

```js
const { a: { b, ...inner }, ...outer } = { a: { b: 1, c: 2, d: 3 }, e: 4, f: 5 };

b;                       // 1
inner;                   // { c: 2, d: 3 }
outer;                   // { e: 4, f: 5 }
```

### Rest is shallow too

```js
const o = { a: { v: 1 }, b: 2 };
const { b, ...r } = o;

r.a.v = 9;
o.a.v;                   // 9            ← r.a is the same inner object
r.a === o.a;             // true
```

### Rules for the rest element

```js
const [...a, b] = [1, 2];       // SyntaxError: Rest element must be last element
const {...a, b} = {};           // SyntaxError: Rest element must be last element
const [a, ...r,] = [1, 2];      // SyntaxError: Rest element must be last element  (no trailing comma either)
```

The rest element must be last, can appear only once, and can't have a trailing comma.

---

## Part 7 — Rest vs Spread: The Difference

Both are written `...`. They do **opposite things**.

| | **Spread** | **Rest** |
|---|---|---|
| Job | **Expands** one collection into many items | **Collects** many items into one collection |
| Direction | one → many | many → one |
| Where it appears | function **calls**, **array** literals, **object** literals | function **parameter lists**, **destructuring patterns** |
| What it produces | separate values (arguments / elements / properties) | a real array or a plain object |
| Side of `=` / role | the **sending / producing** end | the **receiving** end |

### The one-question test

> **Is the `...` handing values OUT (building or calling something), or taking values IN (a place that receives them into variables)?**

```js
function f(...args) { }           // IN  → a parameter list receiving values   → REST
f(...list);                       // OUT → a call being built                   → SPREAD

const [a, ...others] = list;      // IN  → left side of =, receiving values     → REST
const copy = [...list, 4];        // OUT → an array literal being built         → SPREAD

const { x, ...more } = obj;       // IN  → left side of =                       → REST
const merged = { ...obj, y: 1 };  // OUT → an object literal being built        → SPREAD
```

Quick shortcut: **left of `=` or in a function definition → rest. Everywhere else → spread.**

### Both in one function

```js
function wrap(...args) {          // REST: collect the incoming arguments
  return Math.max(...args);       // SPREAD: expand them for the call
}

wrap(3, 9, 2);                    // 9
wrap();                           // -Infinity
```

### Spread/rest vs plain nesting

Spreading and not-spreading are different results:

```js
const a = [1, 2];

[a, 3];                           // [[1, 2], 3]       ← a nested array
[...a, 3];                        // [1, 2, 3]         ← flattened one level

Math.max(a);                      // NaN                ← one argument: an array
Math.max(...a);                   // 2                  ← separate arguments
```

### Related identity: rest and spread are (nearly) inverses

```js
const [first, ...others] = list;      // split off the first
const rebuilt = [first, ...others];   // put it back → same elements as list (a new array)
```

---

## Part 8 — Spread vs `Object.assign`

`Object.assign(target, source)` is an older way to copy properties. It's still asked about.

```js
const target = { a: 1 };
const result = Object.assign(target, { b: 2 });

result === target;                // true     ← it MUTATES and returns the target
target;                           // { a: 1, b: 2 }

const copy = { ...target, c: 3 };
copy === target;                  // false    ← spread builds a NEW object
```

| | `{ ...a, ...b }` | `Object.assign({}, a, b)` / `Object.assign(target, b)` |
|---|---|---|
| Result | always a new object (in an object literal) | the **target** object (mutated) |
| Copies | own enumerable properties | own enumerable properties |
| Mutates something existing? | no | yes — the first argument |
| Setters on the target | not triggered — properties are **defined** | **triggered** — properties are **assigned** |
| Depth | shallow | shallow |

The setter difference, proven:

```js
const log = [];
const t = { set x(v) { log.push("setter"); } };

Object.assign(t, { x: 1 });
log;                              // ["setter"]      ← assignment ran the setter

const t2 = { set x(v) { log.push("s2"); }, ...{ x: 1 } };
log;                              // ["setter"]      ← still only one entry: spread did NOT run the setter
t2.x;                             // 1               ← spread replaced the accessor with a plain data property
```

Use spread for "give me a new object". Use `Object.assign` only when you intentionally want to mutate an existing object.

---

## Part 9 — Gotchas and Performance

### 1. An explicit `undefined` overrides a default

```js
const defaults = { theme: "light", size: 10 };
const options  = { theme: undefined, size: 20 };

const m = { ...defaults, ...options };
m;                       // { theme: undefined, size: 20 }
"theme" in m;            // true
```

Spread copies keys, including those whose value is `undefined`. Your default `"light"` was wiped out. If `options` can hold `undefined` values, filter them first (Implementation Task 1).

### 2. Spread inside a loop copies everything every time

```js
let acc = {};
for (let i = 0; i < n; i++) {
  acc = { ...acc, ["k" + i]: i };     // copies all existing keys again each iteration
}
```

Iteration 1 copies 0 keys, iteration 2 copies 1, … iteration *n* copies *n*−1. Total work grows like n² — slow for large *n*. For building up a result, add to one object or array in place (`acc["k" + i] = i`, `list.push(x)`). Use spread for **one-off** immutable updates, not as an accumulator.

### 3. Object spread and array spread are forgiving in *different* ways

| Input | `{ ...x }` | `[...x]` |
|---|---|---|
| `null` / `undefined` | `{}` | TypeError |
| `5`, `true` | `{}` | TypeError |
| a plain object `{a: 1}` | copies its properties | TypeError |
| an array | index keys `{0: .., 1: ..}` | its elements |
| a string | index keys | its characters |

### 4. Spread loses the prototype link

`{ ...instance }` gives a **plain object**. Methods that lived on the instance's prototype don't come along (own vs inherited, Monday). You'll see this properly when classes arrive on Saturday.

### 5. Spread doesn't make deep copies

Shown in Part 4. If anything nested may be mutated later, copy that level too — or wait for Thursday's deep copy.

### 6. `...` position errors are *early* errors

A rest element out of position stops the whole script before it runs — you won't see earlier `console.log` output.

---

## How Everything Connects

```
Week 1
  → references, mutation vs reassignment   → why a shallow copy shares nested objects (Part 4)
  → pass by value of the reference         → mutating through a copy changes the original

Week 2
  → apply                                  → replaced by spread in calls (Part 3)
  → arguments / lost `this`                → rest parameters, wrapper forwarding (Part 5)
  → fn.length                              → ignores rest (Part 5)
  → partial / pipe / compose               → written with ...fns and ...args (Part 5)
  → decorators                             → fn.apply(this, args) keeps `this`

Monday (Objects)
  → own vs inherited, descriptors          → what spread / rest copy and what they drop (Parts 2, 6)
  → computed keys                          → { ...a, [k]: v } and { [k]: _, ...rest }
  → key order                              → merged object key order (Part 2)

Tuesday (Destructuring)
  → patterns                               → rest completes them (Part 6)
  → defaults                               → only undefined triggers (same idea as Part 9.1)

Coming next
  → Thu: shallow vs deep copy, JSON.stringify pitfalls, structuredClone
  → Fri: array methods (map, filter, reduce...) — everything here builds the arrays they work on
  → Sat: prototypes — why spread drops the inherited part
```

---

## Implementation Tasks

Try each yourself before opening the solution.

### Task 1 — `mergeDefined(defaults, options)`

Merge like `{ ...defaults, ...options }`, but **ignore** option values that are `undefined`. `options` itself may be `undefined`.

<details>
<summary>Solution</summary>

```js
function mergeDefined(defaults, options) {
  const out = { ...defaults };
  const keys = Object.keys(options || {});
  for (let i = 0; i < keys.length; i++) {
    if (options[keys[i]] !== undefined) out[keys[i]] = options[keys[i]];
  }
  return out;
}

mergeDefined({ theme: "light", size: 10 }, { theme: undefined, size: 20 });
// { theme: "light", size: 20 }

mergeDefined({ a: 1 }, undefined);        // { a: 1 }
```

Notes: `null` values are kept (only `undefined` is skipped) — decide whether that's what your API wants. The merge is shallow.
</details>

### Task 2 — `sum`, `average`, and `maxOf` for huge arrays

Write `sum(...nums)` and `average(...nums)` (empty average → `NaN`). Then write `maxOf(list)` that works for an array of 500 000 numbers where `Math.max(...list)` would fail.

<details>
<summary>Solution</summary>

```js
const sum = (...nums) => {
  let total = 0;
  for (let i = 0; i < nums.length; i++) total += nums[i];
  return total;
};

const average = (...nums) => nums.length === 0 ? NaN : sum(...nums) / nums.length;

function maxOf(list) {
  let max = -Infinity;
  for (let i = 0; i < list.length; i++) if (list[i] > max) max = list[i];
  return max;
}

sum();                       // 0
average(2, 4);               // 3
maxOf([1, 9, 3]);            // 9
maxOf([]);                   // -Infinity  (same as Math.max())
maxOf(new Array(500000).fill(1));   // 1
```

A loop uses a fixed amount of stack; spreading a huge array into a call doesn't.
</details>

### Task 3 — Immutable updates

Given:

```js
const state = { user: { name: "Asha", address: { city: "Jaipur" }, tags: ["x"] }, items: [1, 2] };
```

Write `addItem(state, item)`, `setCity(state, city)`, and `removeKey(obj, key)` — none may mutate the input, and unchanged branches must be **shared**, not copied.

<details>
<summary>Solution</summary>

```js
const addItem = (s, item) => ({ ...s, items: [...s.items, item] });

const setCity = (s, city) => ({
  ...s,
  user: { ...s.user, address: { ...s.user.address, city } }
});

const removeKey = (obj, key) => {
  const { [key]: _removed, ...rest } = obj;
  return rest;
};

const s2 = addItem(state, 3);
const s3 = setCity(s2, "Kota");

state.items;                        // [1, 2]            ← unchanged
s2.items;                           // [1, 2, 3]
state.user.address.city;            // "Jaipur"          ← unchanged
s3.user.address.city;               // "Kota"
s3.user.tags === state.user.tags;   // true              ← untouched branch is shared
removeKey({ a: 1, b: 2 }, "a");     // { b: 2 }
```
</details>

### Task 4 — `omit(obj, ...keys)`

Return a new object without the given keys, using a rest parameter for the keys.

<details>
<summary>Solution</summary>

```js
function omit(obj, ...keys) {
  const skip = Object.create(null);          // lookup table with no inherited keys
  for (let i = 0; i < keys.length; i++) skip[keys[i]] = true;

  const out = {};
  const own = Object.keys(obj);
  for (let i = 0; i < own.length; i++) {
    if (!skip[own[i]]) out[own[i]] = obj[own[i]];
  }
  return out;
}

omit({ a: 1, b: 2, c: 3 }, "a", "c");     // { b: 2 }
omit({ a: 1 });                            // { a: 1 }   (no keys → rest is [])
```

Compare with Monday's `omit(obj, keys)` which took an array: the rest parameter lets callers pass keys as separate arguments. (Use the destructuring omit from Part 6 only when the keys are known in advance.)
</details>

### Task 5 — A variadic logging wrapper that keeps `this`

Write `withLogging(fn, log)` that records the arguments of each call in `log`, forwards `this` and all arguments, and returns the result.

<details>
<summary>Solution</summary>

```js
function withLogging(fn, log) {
  return function (...args) {
    log.push(args);
    return fn.apply(this, args);
  };
}

const logs = [];
const o = { n: "o", hi(a, b) { return this.n + a + b; } };
o.hi = withLogging(o.hi, logs);

o.hi("1", "2");      // "o12"
logs;                // [["1", "2"]]
```

`fn(...args)` would lose `this`. An equivalent that keeps `this` is `fn.call(this, ...args)`.
</details>

---

## Output Questions

Predict first, then open the answer.

---

**Q1 — Array spread clone**
```js
const a = [1, 2];
const b = [...a];
b.push(3);
console.log(a, b, a === b);
```
<details>
<summary>Answer</summary>

`[1, 2] [1, 2, 3] false`. Spread makes a new array; pushing to `b` doesn't affect `a`.
</details>

---

**Q2 — Building with spread**
```js
console.log([0, ...[1, 2], 3, ...[4]]);
console.log([..."abc"], { ..."abc" });
```
<details>
<summary>Answer</summary>

`[0, 1, 2, 3, 4]`, then `["a", "b", "c"] { 0: "a", 1: "b", 2: "c" }`.
A string spreads into characters in an array, and into index keys in an object.
</details>

---

**Q3 — Holes**
```js
const a = [1, , 3];
const b = [...a];
console.log(1 in a, 1 in b, b);
```
<details>
<summary>Answer</summary>

`false true [1, undefined, 3]`. Spread fills holes with real `undefined` values.
</details>

---

**Q4 — Emoji length**
```js
const s = "a😀";
console.log(s.length, [...s].length);
```
<details>
<summary>Answer</summary>

`3 2`. `length` counts storage units (the emoji takes two); spread walks characters.
</details>

---

**Q5 — What throws?**
```js
try { [...5]; }       catch (e) { console.log("A", e.name); }
try { [...{ a: 1 }]; } catch (e) { console.log("B", e.name); }
console.log("C", { ...5 }, { ...null });
```
<details>
<summary>Answer</summary>

`A TypeError`, `B TypeError`, `C {} {}`.
Array spread needs an iterable; object spread ignores values that have no properties.
</details>

---

**Q6 — The conditional-spread asymmetry**
```js
console.log({ ...(false && { x: 1 }) });
try { [...(false && [1])]; } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`{}`, then `TypeError`. Object spread ignores `false`; array spread requires an iterable. For arrays use `[...(cond ? [1] : [])]`.
</details>

---

**Q7 — Merge order**
```js
const a = { x: 1, y: 2 };
const b = { y: 9, z: 3 };
console.log({ ...a, ...b });
console.log({ ...b, ...a });
```
<details>
<summary>Answer</summary>

`{ x: 1, y: 9, z: 3 }` and `{ y: 2, z: 3, x: 1 }`. Later spreads win on conflicts. In the second, key order is `y, z, x` because `y` and `z` appeared first (from `b`).
</details>

---

**Q8 — Where does the override go?**
```js
const user = { name: "A", age: 25 };
console.log({ ...user, age: 26 });
console.log({ age: 26, ...user });
```
<details>
<summary>Answer</summary>

`{ name: "A", age: 26 }` and `{ age: 25, name: "A" }`. In the second, `user`'s `age` comes later and overwrites yours.
</details>

---

**Q9 — Undefined overrides**
```js
const defaults = { theme: "light", size: 10 };
const options  = { theme: undefined, size: 20 };
const m = { ...defaults, ...options };
console.log(m, m.theme, "theme" in m);
```
<details>
<summary>Answer</summary>

`{ theme: undefined, size: 20 } undefined true`. The `undefined` value overwrote the default.
</details>

---

**Q10 — Shallow copy**
```js
const o = { n: { v: 1 }, arr: [1] };
const c = { ...o };
c.n.v = 2;
c.arr.push(2);
c.k = 1;
console.log(o.n.v, o.arr, "k" in o, c.n === o.n);
```
<details>
<summary>Answer</summary>

`2 [1, 2] false true`. Nested object and array are shared; adding a new top-level key `k` to the copy doesn't affect `o`.
</details>

---

**Q11 — Array of objects**
```js
const a = [{ v: 1 }];
const b = [...a];
b[0].v = 9;
b.push({ v: 3 });
console.log(a[0].v, a.length);
```
<details>
<summary>Answer</summary>

`9 1`. The element object is shared; the array itself is separate.
</details>

---

**Q12 — Copying a frozen object**
```js
const f = Object.freeze({ a: 1 });
const c = { ...f };
c.a = 2;
console.log(c.a, Object.isFrozen(c));
```
<details>
<summary>Answer</summary>

`2 false`. Spread creates an ordinary new object; freezing isn't copied.
</details>

---

**Q13 — Inherited properties**
```js
const parent = { inh: 1 };
const child = Object.create(parent);
child.own = 2;
const copy = { ...child };
console.log(copy, copy.inh);
```
<details>
<summary>Answer</summary>

`{ own: 2 } undefined`. Only own enumerable properties are copied.
</details>

---

**Q14 — Getters and hidden properties**
```js
const o = { a: 1 };
Object.defineProperty(o, "h", { value: 2, enumerable: false });
let n = 0;
Object.defineProperty(o, "g", { get() { n++; return "got"; }, enumerable: true });

const c = { ...o };
console.log(Object.keys(c), Object.getOwnPropertyDescriptor(c, "g").value, n);
```
<details>
<summary>Answer</summary>

`["a", "g"] "got" 1`. The non-enumerable `h` is skipped; the getter ran once and the copy holds its plain value.
</details>

---

**Q15 — Spread vs assign with a setter**
```js
const log = [];
const t = { set x(v) { log.push("setter"); } };
Object.assign(t, { x: 1 });
const t2 = { set x(v) { log.push("s2"); }, ...{ x: 1 } };
console.log(log, t2.x);
```
<details>
<summary>Answer</summary>

`["setter"] 1`. `assign` triggered the setter; spread defined a plain property `x` and never called the setter.
</details>

---

**Q16 — Spread in a call**
```js
console.log(Math.max(1, ...[2, 9], 4), Math.max(...[]), Math.max([1, 5]));
```
<details>
<summary>Answer</summary>

`9 -Infinity NaN`. Spread gives separate arguments; with none, `Math.max()` is `-Infinity`; passing the array itself as one argument converts to `NaN`.
</details>

---

**Q17 — Missing arguments**
```js
function f(a, b, c, d) { return [a, b, c, d]; }
console.log(f(...[1]));
console.log(f(...[1, 2], 3, ...[4]));
```
<details>
<summary>Answer</summary>

`[1, undefined, undefined, undefined]` then `[1, 2, 3, 4]`.
</details>

---

**Q18 — Rest basics**
```js
function f(first, ...rest) { return [first, rest, Array.isArray(rest)]; }
console.log(f(1, 2, 3));
console.log(f(1));
console.log(f());
```
<details>
<summary>Answer</summary>

`[1, [2, 3], true]`, `[1, [], true]`, `[undefined, [], true]`. Rest is always a real array, empty when nothing extra is passed.
</details>

---

**Q19 — `length` with rest**
```js
function a(x, ...r) {}
const b = (...r) => {};
function c(x, y = 1, ...r) {}
console.log(a.length, b.length, c.length);
```
<details>
<summary>Answer</summary>

`1 0 1`. `length` stops at the first default or rest parameter.
</details>

---

**Q20 — Misplaced rest**
```js
console.log("start");
function f(...a, b) {}
```
<details>
<summary>Answer</summary>

`SyntaxError: Rest parameter must be last formal parameter` — and `"start"` is **never printed**, because it's an early error that stops the script before it runs.
</details>

---

**Q21 — `arguments` vs rest**
```js
function f() { return [typeof arguments, Array.isArray(arguments), arguments.length]; }
function g(...r) { return [typeof r, Array.isArray(r), r.length]; }
console.log(f(1, 2, 3), g(1, 2, 3));
```
<details>
<summary>Answer</summary>

`["object", false, 3] ["object", true, 3]`. Both are `"object"` by `typeof`; only the rest parameter is a real array.
</details>

---

**Q22 — `arguments` aliasing**
```js
function f(a) { arguments[0] = 9; return a; }
function h(a = 0) { arguments[0] = 9; return a; }
console.log(f(1), h(1));
```
<details>
<summary>Answer</summary>

Sloppy: `9 1`. With only simple parameters, `arguments[0]` and `a` are linked; a default parameter breaks the link. In strict mode both would give `1`.
</details>

---

**Q23 — Arrow and `arguments`**
```js
function outer() {
  const a = () => arguments[0];
  return a("inner");
}
console.log(outer("outer"));
```
<details>
<summary>Answer</summary>

`"outer"`. The arrow has no own `arguments`; it uses `outer`'s.
</details>

---

**Q24 — Array rest**
```js
const [a, ...r] = [1, 2, 3];
const [x, ...y] = [1];
console.log(a, r, x, y);
```
<details>
<summary>Answer</summary>

`1 [2, 3] 1 []`. Rest gives an empty array, not `undefined`, when nothing is left.
</details>

---

**Q25 — Omit with object rest**
```js
const user = { id: 1, name: "A", password: "p" };
const { password, ...safe } = user;
console.log(safe, user.password);
```
<details>
<summary>Answer</summary>

`{ id: 1, name: "A" } "p"`. `safe` is a new object; the original still has `password`.
</details>

---

**Q26 — What rest doesn't collect**
```js
const parent = { inh: 1 };
const child = Object.create(parent);
child.a = 1;
child.b = 2;
const { a, ...r } = child;

const o = { a: 1 };
Object.defineProperty(o, "h", { value: 2, enumerable: false });
o.z = 3;
const { a: aa, ...rr } = o;

console.log(r, r.inh, rr);
```
<details>
<summary>Answer</summary>

`{ b: 2 } undefined { z: 3 }`. Inherited and non-enumerable properties aren't collected.
</details>

---

**Q27 — Nested rest**
```js
const { a: { b, ...inner }, ...outer } = { a: { b: 1, c: 2, d: 3 }, e: 4, f: 5 };
console.log(b, inner, outer);
```
<details>
<summary>Answer</summary>

`1 { c: 2, d: 3 } { e: 4, f: 5 }`. Note that no variable called `a` is created (the nested-pattern rule from Tuesday).
</details>

---

**Q28 — Rest in the wrong place**
```js
const [...a, b] = [1, 2];
```
<details>
<summary>Answer</summary>

`SyntaxError: Rest element must be last element`. The same applies to object patterns, and to a trailing comma after the rest element.
</details>

---

**Q29 — Wrapper that loses `this`**
```js
function bad(fn)  { return function (...args) { return fn(...args); }; }
function good(fn) { return function (...args) { return fn.apply(this, args); }; }

const o = { n: "o", hi() { return this && this.n; } };
const p = { n: "p", hi() { return this && this.n; } };
o.hi = good(o.hi);
p.hi = bad(p.hi);
console.log(o.hi(), p.hi());
```
<details>
<summary>Answer</summary>

`"o" undefined`. The `good` wrapper forwards `this`; `bad` calls `fn` as a plain function (default binding).
</details>

---

**Q30 — Updating nested state**
```js
const state = { user: { address: { city: "J" } }, items: [1, 2] };
const next = {
  ...state,
  user: { ...state.user, address: { ...state.user.address, city: "K" } },
  items: [...state.items, 3]
};
const same = { ...state, items: [...state.items, 4] };

console.log(state.user.address.city, next.user.address.city, state.items, next.items);
console.log(state.user === next.user, same.user === state.user);
```
<details>
<summary>Answer</summary>

`"J" "K" [1, 2] [1, 2, 3]`, then `false true`.
`next.user` was rebuilt, so it differs. In `same`, `user` was not rebuilt, so it's the very same object as in `state`.
</details>

---

**Q31 — Nested merge**
```js
const defaults = { ui: { theme: "light", size: 1 }, lang: "en" };
const opts = { ui: { theme: "dark" } };
console.log({ ...defaults, ...opts });
```
<details>
<summary>Answer</summary>

`{ ui: { theme: "dark" }, lang: "en" }`. The whole `ui` branch was replaced; `size` was lost. Spread merges only one level.
</details>

---

## What Interviewers Test

**1. Rest vs spread**
The same `...` does opposite jobs. Be ready to classify each occurrence in a snippet: parameters and left-of-`=` → rest; calls, array literals, object literals → spread.

**2. Spread makes shallow copies**
Nested objects/arrays are shared. Know how to update a nested value immutably by spreading each level on the path.

**3. What object spread copies**
Own enumerable only; getters become values; no prototype; not frozen; `undefined` values overwrite earlier ones; later keys win.

**4. Array vs object spread rules**
Arrays need an iterable (strings, arrays); objects accept almost anything. The `false &&` conditional-spread trick works for objects but throws for arrays.

**5. Rest parameters vs `arguments`**
Real array vs array-like; arrows have no own `arguments`; sloppy-mode aliasing; `fn.length` ignores rest.

**6. Variadic wrappers**
`function (...args) { return fn.apply(this, args); }` — forward `this` and all arguments.

**7. Omit with object rest**
`const { password, ...safe } = user`, and why it's own + enumerable only.

**8. Spread vs `Object.assign`**
New object vs mutating the target; setters triggered vs not.

**9. Performance**
Spread in a loop copies repeatedly (n²); spreading huge arrays into calls can throw `RangeError`.

---

## Mental Models

**Spread = unpacking a suitcase onto the bed.**
Everything laid out item by item, where you choose. The suitcase itself is untouched.

**Rest = packing the leftovers into one bag.**
Whatever isn't claimed by a named parameter or variable goes into the bag (an array, or an object).

**Where the dots sit decides the job.**
Receiving values into variables (a parameter list, the left of `=`) → rest. Producing values (a call, an array or object literal) → spread.

**A shallow copy is a photocopy of the table of contents.**
A new list, but every entry still points to the same chapters. Edit a chapter through the copy and the original book changes too.

**Copy the path, share the rest.**
To change something deep without mutating, make a new object at every level you walk through, and reuse everything else.

**Spread copies values of own, enumerable properties — nothing more.**
No inheritance link, no getters or setters, no freeze, no hidden properties.

---

*Previous: Destructuring — object + array, nested, parameters, defaults*
*Next: Copying — shallow vs deep copy, `JSON.stringify` pitfalls, `structuredClone()`, manual deep clone*