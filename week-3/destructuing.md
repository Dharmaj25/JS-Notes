# Destructuring — Objects, Arrays, Nesting, Parameters, Defaults

---

## What is Destructuring?

Destructuring is a **shorter way to pull values out of an object or array into variables**.

Without it:

```js
const user = { name: "Asha", age: 25, city: "Jaipur" };

const name = user.name;
const age  = user.age;
const city = user.city;
```

With it:

```js
const { name, age, city } = user;
```

Same result. One line.

Three things to understand before anything else:

```
1. It is SYNTAX, not a new data type.
   Nothing new is created. It reads values and puts them into variables.

2. The left side is a PATTERN that mirrors the shape of the data on the right.
   {  }  on the left matches an object.      [  ]  on the left matches an array.

3. It does NOT copy deeply, and it does NOT change the source.
   It just reads values out (Part 7).
```

> **Today's map:**
> object patterns → array patterns → default values → nesting →
> function parameters → assigning without declaring → gotchas

> **Not covered today:** collecting the leftovers with `...` (the rest pattern). That belongs to tomorrow's topic, Spread / Rest. Everything below works without it.

---

## Part 1 — Object Destructuring

### The basics: match by **key name**

```js
const user = { name: "Asha", age: 25 };

const { name, age } = user;

name;      // "Asha"
age;       // 25
```

The variables are created with **the same names as the keys**.

### Order does not matter

```js
const { age, name } = user;       // same result — objects are matched by key, not position
```

### Missing keys give `undefined`, extra keys are ignored

```js
const { name, country } = user;

name;         // "Asha"
country;      // undefined   ← the object has no "country" key
              // "age" is simply not picked — that's fine
```

### Renaming: `key: newName`

```js
const { name: userName, age: years } = user;

userName;     // "Asha"
years;        // 25
age;          // ReferenceError — "age" was NOT created; only years was
              // (assuming no other variable called age exists in scope)
```

This is the #1 thing people mix up. Read it like this:

```
{ name: userName }
   │       └── the NEW VARIABLE you are creating
   └── the KEY to look up in the object
```

In an object **literal** the colon means "key: value". In a destructuring **pattern** the colon means "from this key → into this variable".

| | Object literal (building) | Pattern (extracting) |
|---|---|---|
| Code | `{ a: x }` | `const { a: x } = obj` |
| Meaning | create a key `a` with the value of `x` | read key `a` from `obj`, store it in a new variable `x` |

### Works with `const`, `let`, and `var`

```js
const { a } = obj;      // a cannot be reassigned
let   { b } = obj;      // b can be reassigned
```

Variables created by destructuring behave exactly like any other variable of that keyword. A `const` from destructuring is still a `const`:

```js
const { b } = { b: 1 };
b = 2;       // TypeError: Assignment to constant variable.
```

### Computed keys in a pattern

Remember computed property names from Monday. They work in patterns too — put the key expression in brackets and give it a variable name:

```js
const key = "name";
const { [key]: value } = { name: "Asha" };

value;      // "Asha"
```

A computed key **requires** a `: variableName` part, because JavaScript has no way to invent a variable name from an expression.

---

## Part 2 — Array Destructuring

### The basics: match by **position**

```js
const colors = ["red", "green", "blue"];

const [first, second, third] = colors;

first;      // "red"
second;     // "green"
third;      // "blue"
```

Names don't matter here — **position** does. The first variable gets index 0, and so on.

### Skip elements by leaving a gap

```js
const [a, , c] = [1, 2, 3];

a;          // 1
c;          // 3     ← the empty slot skipped the 2
```

### Fewer elements → `undefined`; extra elements → ignored

```js
const [x, y] = [1];
x;          // 1
y;          // undefined

const [p] = [1, 2, 3];
p;          // 1
```

### Swapping two variables — no temporary variable needed

```js
let a = 1;
let b = 2;

[a, b] = [b, a];

a;          // 2
b;          // 1
```

The right side `[b, a]` is built first (`[2, 1]`), **then** assigned into `a` and `b`. That's why it works without a temp variable.

(Watch the semicolon rules in Part 6 — this exact line has a famous trap.)

### Returning multiple values from a function

A function can only `return` one thing — but that thing can be an array:

```js
function minMax(list) {
  let min = list[0];
  let max = list[0];
  for (let i = 1; i < list.length; i++) {
    if (list[i] < min) min = list[i];
    if (list[i] > max) max = list[i];
  }
  return [min, max];
}

const [lo, hi] = minMax([3, 9, 1]);
lo;         // 1
hi;         // 9
```

The caller picks its own names for the results.

### Splitting a string into parts

```js
const [key, value] = "a=1".split("=");

key;        // "a"
value;      // "1"
```

### Pulling apart `Object.entries`

`Object.entries` (from Monday) returns `[key, value]` pairs — perfect for array destructuring:

```js
const prices = { apple: 40, mango: 90 };
const entries = Object.entries(prices);      // [["apple", 40], ["mango", 90]]

for (let i = 0; i < entries.length; i++) {
  const [fruit, price] = entries[i];
  console.log(fruit, price);
}
// apple 40
// mango 90
```

> In real-world code you'll usually see this written with a loop called `for...of` and the pattern in the loop head. The destructuring part is identical to what's above.

### Arrays need an  **iterable**; objects don't

Array destructuring works on anything **iterable** (values with an internal sequential order). Arrays, strings, Sets, and Maps are iterable, but a plain object is not. 

Array destructuring does **not** look for keys like `0` or `1`. Instead, it calls a hidden looping function (`Symbol.iterator`) on the value. If that function doesn't exist, it crashes immediately.

```js
const [c1, c2] = "hi";        // ✅ c1 = "h", c2 = "i" (Strings are iterable)
const [z] = { 0: "x" };       // ❌ TypeError: ... is not iterable (Plain objects lack a loop function)
const [q] = null;             // ❌ TypeError: null is not iterable
```

Meanwhile, **object** patterns work on arrays because arrays are fundamentally objects under the hood with numeric keys and a special `length` property:

```js
const { 0: first, length } = ["x", "y"];
console.log(first);  // ✅ "x" (Unpacks the property at key '0')
console.log(length); // ✅ 2   (Unpacks the 'length' property)
```



#### Deep Dive: Why does object destructuring work on primitives but crash on null?
When you object-destructure a primitive (like a string, number, or boolean), JavaScript temporarily wraps it in an object behind the scenes:
```js
const { length } = "hello"; // ✅ Works! (Temporarily turned into a String object)
const { toString } = 123;   // ✅ Works! (Temporarily turned into a Number object)
```
However, `null` and `undefined` are completely blank primitives. They cannot be wrapped into objects. Even though `typeof null` returns `"object"` (a historical JavaScript bug), it lacks any internal structure. JavaScript will crash **before** checking your variables because it fails to convert the right-hand side into an object:
```js
const { foo } = null;      // ❌ TypeError: Cannot destructure property 'foo' of 'null'
const { bar } = undefined; // ❌ TypeError: Cannot destructure property 'bar' of 'undefined'
```


Array destructuring works on anything **iterable** — arrays and strings are. A plain object is not:

```js
const [c1, c2] = "hi";        // ✅ c1 = "h", c2 = "i"
const [z] = { 0: "x" };       // ❌ TypeError: {(intermediate value)} is not iterable
const [q] = null;             // ❌ TypeError: null is not iterable
```

Meanwhile, **object** patterns work on arrays, using the index as the key:

```js
const { 0: first, length } = ["x", "y"];
first;      // "x"
length;     // 2
```

(What "iterable" means exactly is a Week 5 topic. For now: arrays and strings qualify, plain objects don't.)

### Object pattern vs array pattern — summary

| Feature | Object pattern `{ }` | Array pattern `[ ]` |
| :--- | :--- | :--- |
| **Matches by** | Property key name | Sequential position (1st, 2nd, 3rd...) |
| **Order matters?** | No | Yes |
| **Rename variable** | `{ key: newName }` | Just choose any variable name `[newName]` |
| **Skip something** | Simply don't mention it | Leave an empty slot `[a, , c]` |
| **Source must be** | **Anything except `null` or `undefined`**. JavaScript converts ("boxes") primitives like strings/numbers into temporary objects so you can extract properties. | **An iterable.** Must possess a built-in sequential looping blueprint (`Symbol.iterator`). |

---

## Part 3 — Default Values

A **default** supplies a value when the source has nothing there.

```js
const { name, country = "India" } = { name: "Asha" };

name;       // "Asha"
country;    // "India"     ← the key was missing, so the default is used
```

Arrays:

```js
const [a = 10, b = 20] = [1];

a;          // 1
b;          // 20
```

### Only `undefined` triggers a default

This is the **same rule as default parameters** from Week 2 — and it's the same trap:

```js
const { a = 10, b = 10, c = 10 } = { a: undefined, b: null };

a;          // 10    ← undefined → default used
b;          // null  ← null is a real value, default NOT used
c;          // 10    ← missing → undefined → default used
```

Other "empty-looking" values — `0`, `""`, `false`, `NaN` — also do **not** trigger a default.

### Defaults are evaluated lazily, left to right

A default expression only runs **when it is needed**:

```js
let calls = 0;
const makeDefault = () => { calls++; return "d"; };

const { a = makeDefault(), b = makeDefault() } = { a: "given" };

a;          // "given"   ← default not needed
b;          // "d"       ← default needed, so the function ran
calls;      // 1
```

### Defaults can use earlier variables, but not later ones

```js
const [a = 1, b = a + 1] = [];
a;          // 1
b;          // 2          ← b's default used a, which was already set
```

```js
const [x = y, y = 1] = [];
// ReferenceError: Cannot access 'y' before initialization
```

This is the TDZ again (Week 1). Each variable is created in order, so a later one isn't ready yet. Same for referring to itself:

```js
const { a = a } = {};
// ReferenceError: Cannot access 'a' before initialization
```

### Rename and default together

```js
const { name: userName = "Guest", age: years = 0 } = { age: null };

userName;   // "Guest"    ← missing → default
years;      // null       ← null is not undefined → no default
```

Reading order: `key : newName = default`.

### What destructuring "really does"

```js
const { a, b: c = 5 } = obj;
```

is roughly the same as:

```js
const a = obj.a;                        // obj is evaluated once
const temp = obj.b;
const c = temp === undefined ? 5 : temp;
```

Keep this picture. Every rule above (missing → `undefined`, `null` stays `null`, inherited values are readable) follows from "it just reads `obj.key`".

---

## Part 4 — Nested Destructuring

Real data is nested. A pattern can be **nested to mirror** it.

```js
const user = {
  name: "Asha",
  address: { city: "Jaipur", geo: { lat: 26.9 } },
  tags: ["admin", "dev"]
};
```

### Object inside an object

```js
const { address: { city } } = user;
city;       // "Jaipur"
```

### Deeper

```js
const { address: { geo: { lat } } } = user;
lat;        // 26.9
```

### Array inside an object

```js
const { tags: [firstTag] } = user;
firstTag;   // "admin"
```

### Object inside an array, array inside an array

```js
const [{ id }, { id: id2 }] = [{ id: 7 }, { id: 8 }];
id;         // 7
id2;        // 8

const [[a, b], [c]] = [[1, 2], [3]];
a; b; c;    // 1 2 3
```

### ⚠️ A nested pattern does NOT create a variable for the middle key

```js
const { address: { city } } = user;

city;                  // "Jaipur"
typeof address;        // "undefined"    ← "address" was only a path, not a variable
```

If you also need the whole inner object, name it separately:

```js
const { address, address: { city } } = user;
// address → the object,   city → "Jaipur"
```

### ⚠️ If the middle part is missing, it throws

```js
const { address: { city } } = {};
// TypeError: Cannot read properties of undefined (reading 'city')
```

`address` is `undefined`, and you can't destructure `undefined`.

**Guard with a default for the parent:**

```js
const { address: { city } = {} } = {};
city;       // undefined       ← no crash
```

Read it as: "take `address`; if it's `undefined`, use `{}`; then take `city` from that."

### ⚠️ The guard only protects against `undefined` — not `null`

```js
const { address: { city } = {} } = { address: null };
// TypeError: Cannot destructure property 'city' of '{}' as it is null.
```

Same rule as before: a default fires only for `undefined`. `null` goes straight through and crashes. If your data can contain `null`, check it before destructuring.

### A complete example

```js
const response = {
  data: { user: { id: 7, profile: { first: "Asha", last: "Rao" }, roles: ["admin", "dev"] } },
  meta: { page: 1 }
};

const {
  data: { user: { id, profile: { first, last }, roles: [firstRole], country = "IN" } },
  meta: { page = 1 } = {}
} = response;

id;          // 7
first;       // "Asha"
last;        // "Rao"
firstRole;   // "admin"
country;     // "IN"        ← default, not in the data
page;        // 1
```

### Readability rule of thumb

Nesting more than about 2–3 levels is hard to read and easy to get wrong. Past that, destructure in steps:

```js
const { user } = response.data;
const { profile, roles } = user;
```

---

## Part 5 — Destructuring in Function Parameters

Parameters are variables too — so they can use patterns.

### Object pattern as a parameter

```js
function greet({ name, age }) {
  return name + " is " + age;
}

greet({ name: "Asha", age: 25 });     // "Asha is 25"
```

It's the same as writing `function greet(user) { const { name, age } = user; ... }`.

### The "options object" pattern

When a function takes many values, a single object is clearer than many positional arguments:

```js
// Hard to read at the call site — what is true? what is "dark"?
createUser("Asha", 25, true, "IN", "dark", false);

// Self-documenting
createUser({ name: "Asha", age: 25, isAdmin: true, country: "IN", theme: "dark", notify: false });
```

Benefits: order doesn't matter, you can skip optional values, and adding a new option doesn't break old calls.

### Defaults inside the pattern

```js
function createUser({ name, age = 18, isAdmin = false, theme = "light" }) {
  return { name, age, isAdmin, theme };
}

createUser({ name: "Asha" });
// { name: "Asha", age: 18, isAdmin: false, theme: "light" }
```

### ⚠️ The essential trap: what if nothing is passed?

```js
function f({ a, b = 2 }) { return [a, b]; }

f({ a: 1 });     // [1, 2]
f();             // TypeError: Cannot destructure property 'a' of 'undefined' as it is undefined.
```

Calling `f()` means the parameter is `undefined`, and `undefined` can't be destructured.

**Fix: a default for the whole parameter — `= {}`**

```js
function f({ a, b = 2 } = {}) { return [a, b]; }

f();             // [undefined, 2]     ✅
f({ a: 9 });     // [9, 2]
```

Two different levels of defaults, don't mix them up:

| | What it covers |
|---|---|
| `{ b = 2 }` | a missing **property** `b` inside the object |
| `= {}` after the pattern | the **whole argument** being missing (`undefined`) |

And `null` still crashes, because `null` isn't `undefined`:

```js
f(null);         // TypeError (Cannot destructure property 'a' of ... as it is null)
```

### Whole-parameter default vs property default — a favourite trick question

```js
function f({ a } = { a: 5 }) { return a; }

f();             // 5           ← nothing passed → the whole default object is used
f({});           // undefined   ← something WAS passed; the default object is ignored
```

`{ a } = { a: 5 }` provides a default for the argument, **not** for `a`. To default `a` itself, write `{ a = 5 } = {}`.

### Array patterns and mixed parameters

```js
function f({ a = 1, b = 2 } = {}, [c = 3] = []) {
  return [a, b, c];
}

f();                          // [1, 2, 3]
f({ a: 5 }, [undefined]);     // [5, 2, 3]
f({ b: null }, [0]);          // [1, null, 0]
```

### Nested patterns in parameters

```js
function show({ id, tags: [firstTag = "none"] = [] }) {
  return id + ":" + firstTag;
}

show({ id: 1, tags: ["t"] });   // "1:t"
show({ id: 2 });                // "2:none"
show({ id: 3, tags: [] });      // "3:none"
```

### In callbacks

Callbacks are just functions, so their parameters can be patterns. Here's a small higher-order function of our own (Week 2), since array methods come on Friday:

```js
const each = (items, fn) => {
  for (let i = 0; i < items.length; i++) fn(items[i], i);
};

each([{ id: 1, tag: "a" }, { id: 2 }], ({ id, tag = "-" }, index) => {
  console.log(index, id, tag);
});
// 0 1 a
// 1 2 -
```

### `fn.length` and destructured parameters

A destructured parameter counts as **one** parameter, and `length` stops at the first parameter with a default (recall Week 2):

```js
(function ({ a }) {}).length;            // 1
(function ({ a } = {}) {}).length;       // 0   ← has a default
(function (a, [b]) {}).length;           // 2
((a, { b } = {}, c) => {}).length;       // 1   ← stops at the first default
```

### Trade-offs of destructured parameters

```
+ call sites are readable; order doesn't matter; easy to add options
+ the function signature shows which properties it uses

− the whole argument must be an object (null breaks it)
− harder to pass the "same object" through (you've pulled it apart)
− you lose the original object; if you need it, take it as one parameter
− very long patterns make signatures hard to read
```

---

## Part 6 — Assigning With Destructuring (Without Declaring)

So far every pattern declared new variables with `const`/`let`. You can also destructure into variables that **already exist**.

### Arrays: just assign

```js
let a, b;
[a, b] = [1, 2];
```

### Objects: wrap the whole thing in parentheses

```js
let a, b;
({ a, b } = { a: 1, b: 2 });         // ✅

{ a, b } = { a: 1, b: 2 };           // ❌ SyntaxError: Unexpected token '='
```

Why? At the start of a statement, `{` begins a **block**, not an object. The parentheses force JavaScript to read it as an expression.

### Targets can be any assignable thing

```js
const o = {};
[o.x, o.y] = [1, 2];
o;           // { x: 1, y: 2 }
```

That is how you swap two elements of an array in place:

```js
function swap(arr, i, j) {
  [arr[i], arr[j]] = [arr[j], arr[i]];
  return arr;
}

swap([1, 2, 3], 0, 2);     // [3, 2, 1]
```

### ⚠️ The semicolon trap with `[` at the start of a line

```js
let a = 1, b = 2
[a, b] = [b, a]
// ReferenceError: Cannot access 'b' before initialization
```

Without a semicolon after the first line, JavaScript keeps reading and sees:

```js
let a = 1, b = 2[a, b] = [b, a]      // one statement!  "2[a, b]" looks like indexing
```

A line that starts with `[` (or `(`) can be glued onto the previous line. (The exact error varies with the surrounding code; the cause is always this.)

```js
let a = 1
let b = 2
;[a, b] = [b, a]                     // ✅ leading semicolon — or just use semicolons everywhere
```

Using semicolons consistently, or a formatter like Prettier, avoids this entirely.

---

## Part 7 — Gotchas: What Destructuring Is (and Isn't)

### 1. It is a shallow read — nested objects are shared, not copied

You saw in Monday's Part 3 that objects are shared references. Destructuring doesn't change that:

```js
const o = { n: { v: 1 } };
const { n } = o;

n.v = 2;
o.n.v;          // 2           ← same inner object
n === o.n;      // true
```

Destructuring pulls out a **reference** to `n`, not a copy. Real copying is Wednesday and Thursday.

### 2. It never changes the source

```js
const user = { name: "Asha" };
const { name } = user;
// user is unchanged; name is a separate variable
```

If the extracted value is a primitive, changing `name` later doesn't touch `user`. If it's an object, mutating *its contents* does (point 1).

### 3. Methods lose their `this` — the Week 2 lost-binding bug again

```js
const user = {
  name: "Asha",
  greet() { return this.name; }
};

const { greet } = user;
greet();            // undefined      ← plain call, default binding
user.greet();       // "Asha"
```

(In strict mode the first call throws `TypeError: Cannot read properties of undefined`.)

`const { greet } = user` is exactly like `const greet = user.greet` — it extracts the function and drops the object. Fix: `bind`, a wrapper, or don't destructure methods that use `this`.

### 4. Getters run, in *pattern* order

Destructuring reads `obj.key` for real, so getters execute — in the order of the **pattern**, not the object:

```js
const log = [];
const o = {
  get a() { log.push("a"); return 1; },
  get b() { log.push("b"); return 2; }
};

const { b, a } = o;
log;                // ["b", "a"]
```

### 5. It reads inherited properties too

```js
const { toString } = {};
typeof toString;    // "function"   ← inherited from the built-in object

const parent = { inh: 1 };
const child = Object.create(parent);
const { inh } = child;
inh;                // 1
```

(`in` from Monday would also be true for these. Destructuring is `obj.key`, and `obj.key` walks the link.)

### 6. Primitives work, `null` and `undefined` don't

```js
const { length } = "abc";      // 3     (autoboxing, Week 1)
const [x, y] = "hi";           // "h", "i"
const { z } = 5;               // undefined — numbers have no "z"

const { a } = null;
// TypeError: Cannot destructure property 'a' of 'null' as it is null.

const { b } = undefined;
// TypeError: Cannot destructure property 'b' of 'undefined' as it is undefined.
```

### 7. Redeclaring is an error

```js
const { a } = { a: 1 };
const { a } = { a: 2 };
// SyntaxError: Identifier 'a' has already been declared
```

### 8. Same property twice is allowed

```js
const { a, a: b } = { a: 1 };
a;   // 1
b;   // 1
```

### 9. Quick reference: the errors

| Code | Result |
|---|---|
| `const { a } = null` / `undefined` | TypeError (cannot destructure) |
| `const [a] = null` | TypeError (`null is not iterable`) |
| `const [a] = { 0: 1 }` | TypeError (not iterable) |
| `const { a: { b } } = {}` | TypeError (reading `b` of `undefined`) |
| `const [x = y, y = 1] = []` | ReferenceError (TDZ) |
| `const { a } = 5` | no error → `a` is `undefined` |
| `function f({ a }) {}; f()` | TypeError (`undefined` can't be destructured) |

---

## How Everything Connects

```
Week 1
  → const / let / var and TDZ       → why a later default can't be used early (Part 3)
  → undefined vs null               → only undefined triggers a default
  → autoboxing                      → destructuring strings and numbers (Part 7)

Week 2
  → default parameters              → same "only undefined" rule (Parts 3, 5)
  → lost `this`                     → destructured methods (Part 7)
  → higher-order functions          → callbacks with pattern parameters (Part 5)
  → fn.length                       → counts a pattern as one parameter (Part 5)

Monday (Objects)
  → computed property names         → computed keys in patterns (Part 1)
  → own vs inherited                → destructuring reads inherited values (Part 7)
  → shared references               → shallow read (Part 7)
  → Object.entries                  → destructuring [key, value] pairs (Part 2)

Coming next
  → Wed: rest (...) completes destructuring; spread builds new objects/arrays
  → Thu: shallow vs deep copy — why Part 7 point 1 matters
```

---

## Implementation Tasks

Try each yourself before opening the solution.

### Task 1 — Extract from a nested response

Given:

```js
const response = {
  data: { user: { id: 7, profile: { first: "Asha", last: "Rao" }, roles: ["admin", "dev"] } },
  meta: { page: 1 }
};
```

In **one** destructuring statement, get `id`, `first`, `last`, the first role as `firstRole`, `country` (default `"IN"`), and `page` (default `1`, even if `meta` is missing).

<details>
<summary>Solution</summary>

```js
const {
  data: { user: { id, profile: { first, last }, roles: [firstRole], country = "IN" } },
  meta: { page = 1 } = {}
} = response;
```

Notes: `meta: { page = 1 } = {}` — the `= {}` protects against `meta` being missing; `page = 1` protects against `page` being missing. If `roles` were an empty array, `firstRole` would simply be `undefined`.
</details>

### Task 2 — `swap(arr, i, j)` and `minMax(arr)`

Write `swap` without a temporary variable, and `minMax` that returns both values.

<details>
<summary>Solution</summary>

```js
function swap(arr, i, j) {
  [arr[i], arr[j]] = [arr[j], arr[i]];
  return arr;
}

function minMax(arr) {
  let min = arr[0], max = arr[0];
  for (let i = 1; i < arr.length; i++) {
    if (arr[i] < min) min = arr[i];
    if (arr[i] > max) max = arr[i];
  }
  return [min, max];
}

swap([1, 2, 3], 0, 2);            // [3, 2, 1]
const [lo, hi] = minMax([3, 9, 1]);   // lo = 1, hi = 9
```
Remember the semicolon at the end of the line before `[arr[i], ...]` — that's where the trap lives.
</details>

### Task 3 — Refactor positional parameters into an options object

```js
function createUser(name, age, isAdmin, country, theme, notify) { /* ... */ }
```

Rewrite so that: all but `name` have defaults (`18`, `false`, `"IN"`, `"light"`, `true`); calling with no argument must not crash with a confusing message; `name` is required.

<details>
<summary>Solution</summary>

```js
function createUser({ name, age = 18, isAdmin = false, country = "IN", theme = "light", notify = true } = {}) {
  if (name === undefined) throw new TypeError("name is required");
  return { name, age, isAdmin, country, theme, notify };
}

createUser({ name: "Asha", theme: "dark" });
// { name: "Asha", age: 18, isAdmin: false, country: "IN", theme: "dark", notify: true }

createUser();   // TypeError: name is required   ← a clear message, thanks to `= {}`
```

Observations:
- Destructuring cannot mark a field **required** — you must check it yourself.
- `createUser.length` is now `0` (the pattern has a default).
- `createUser(null)` still gives the generic "cannot destructure" error.
</details>

### Task 4 — Parse `"key=value"` strings (and spot the hidden bug)

Turn `"a=1"` into `["a", "1"]`, and a flag with no value (`"debug"`) into `["debug", "true"]`.

<details>
<summary>Solution + hidden bug</summary>

First attempt:

```js
function parsePair(pair) {
  const [key, value = "true"] = pair.split("=");
  return [key, value];
}

parsePair("a=1");        // ["a", "1"]
parsePair("debug");      // ["debug", "true"]
parsePair("url=a=b");    // ["url", "a"]   ← BUG: the value "a=b" was cut at the second "="
```

`split("=")` produces **three** pieces for `"url=a=b"`; the pattern only reads the first two. Split only at the **first** `=`:

```js
function parsePair(pair) {
  const i = pair.indexOf("=");
  if (i === -1) return [pair, "true"];
  return [pair.slice(0, i), pair.slice(i + 1)];
}

parsePair("url=a=b");    // ["url", "a=b"]
parsePair("x=");         // ["x", ""]
```

Also note: for `"x="` the value is the empty string `""`, which does **not** trigger a default, so `const [k, v = "true"] = ["x", ""]` gives `v === ""`. Decide whether that's what you want.
</details>

---

## Output Questions

Predict first, then open the answer.

---

**Q1 — Missing and extra keys**
```js
const { a, b, c } = { a: 1, b: 2 };
console.log(a, b, c);
```
<details>
<summary>Answer</summary>

`1 2 undefined`. `c` isn't in the object. (An extra key in the object would simply be ignored.)
</details>

---

**Q2 — Rename**
```js
const { a: x } = { a: 1 };
console.log(x, typeof a);
```
<details>
<summary>Answer</summary>

`1 "undefined"`. Only `x` was created. `a` was just the key to look up.
</details>

---

**Q3 — Array positions and gaps**
```js
const [a, , c] = [1, 2, 3];
const [p, q] = [1];
console.log(a, c, p, q);
```
<details>
<summary>Answer</summary>

`1 3 1 undefined`. The gap skips `2`; the short array leaves `q` as `undefined`.
</details>

---

**Q4 — What triggers a default?**
```js
const { a = 10, b = 10, c = 10 } = { a: undefined, b: null };
const [m = 1] = [undefined];
const [n = 1] = [null];
console.log(a, b, c, m, n);
```
<details>
<summary>Answer</summary>

`10 null 10 1 null`. Only `undefined` (explicit or missing) triggers a default; `null` is kept.
</details>

---

**Q5 — Earlier vs later references**
```js
const [a = 1, b = a + 1] = [];
console.log(a, b);

const [x = y, y = 1] = [];
```
<details>
<summary>Answer</summary>

`1 2`, then the last line throws `ReferenceError: Cannot access 'y' before initialization`. A default can use earlier variables but not later ones (TDZ).
</details>

---

**Q6 — Self-reference**
```js
const { a = a } = {};
```
<details>
<summary>Answer</summary>

`ReferenceError: Cannot access 'a' before initialization`. While `a`'s default is being evaluated, `a` is still in its TDZ.
</details>

---

**Q7 — Lazy defaults**
```js
let calls = 0;
const makeDefault = () => { calls++; return "d"; };
const { a = makeDefault(), b = makeDefault() } = { a: "given" };
console.log(a, b, calls);
```
<details>
<summary>Answer</summary>

`"given" "d" 1`. The default for `a` never ran; only the one for `b` did.
</details>

---

**Q8 — Nested patterns and the middle variable**
```js
const user = { address: { city: "Jaipur" }, tags: ["x", "y"] };
const { address: { city }, tags: [first] } = user;
console.log(city, first, typeof address);
```
<details>
<summary>Answer</summary>

`"Jaipur" "x" "undefined"`. `address` was only a path; no variable named `address` exists.
</details>

---

**Q9 — Missing parent, guarded parent, null parent**
```js
try { const { address: { city } } = {}; } catch (e) { console.log("A", e.name); }

const { address: { city = "?" } = {} } = {};
console.log("B", city);

try { const { address: { city } = {} } = { address: null }; } catch (e) { console.log("C", e.name); }
```
<details>
<summary>Answer</summary>

`A TypeError`, `B ?`, `C TypeError`.
A: `address` is `undefined` → can't destructure it. B: the `= {}` default rescues `undefined`. C: `null` is not `undefined`, so the default is skipped and it crashes.
</details>

---

**Q10 — Objects and arrays inside arrays**
```js
const [{ id }, { id: id2 }] = [{ id: 7 }, { id: 8 }];
const [[a, b], [c]] = [[1, 2], [3]];
console.log(id, id2, a, b, c);
```
<details>
<summary>Answer</summary>

`7 8 1 2 3`.
</details>

---

**Q11 — Parameter with and without an argument**
```js
function f({ a, b = 2 }) { return [a, b]; }
function g({ a, b = 2 } = {}) { return [a, b]; }

console.log(f({ a: 1 }));
try { f(); } catch (e) { console.log(e.name); }
console.log(g(), g({ a: 9 }));
```
<details>
<summary>Answer</summary>

`[1, 2]`, `TypeError`, then `[undefined, 2] [9, 2]`.
`f()` passes `undefined` to a pattern; `g`'s `= {}` supplies an empty object.
</details>

---

**Q12 — `null` argument**
```js
function g({ a, b = 2 } = {}) { return [a, b]; }
try { g(null); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`TypeError`. The `= {}` default only applies to `undefined`; `null` is destructured and fails.
</details>

---

**Q13 — `length` with patterns**
```js
console.log(
  (function ({ a }) {}).length,
  (function ({ a } = {}) {}).length,
  (function (a, [b]) {}).length,
  ((a, { b } = {}, c) => {}).length
);
```
<details>
<summary>Answer</summary>

`1 0 2 1`. A pattern counts as one parameter; counting stops at the first parameter with a default.
</details>

---

**Q14 — Mixed parameters**
```js
function f({ a = 1, b = 2 } = {}, [c = 3] = []) { return [a, b, c]; }
console.log(f(), f({ a: 5 }, [undefined]), f({ b: null }, [0]));
```
<details>
<summary>Answer</summary>

`[1, 2, 3] [5, 2, 3] [1, null, 0]`.
`undefined` triggers defaults; `null` and `0` don't.
</details>

---

**Q15 — Whole-parameter default vs property default**
```js
function f({ a } = { a: 5 }) { return a; }
console.log(f(), f({}));
```
<details>
<summary>Answer</summary>

`5 undefined`. `f()` uses the default *object*. `f({})` passes a real object, so the default object is ignored and `a` is `undefined`. To default `a`, write `{ a = 5 } = {}`.
</details>

---

**Q16 — Assigning with an object pattern**
```js
let a, b;
({ a, b } = { a: 1, b: 2 });
console.log(a, b);

{ a, b } = { a: 3, b: 4 };
```
<details>
<summary>Answer</summary>

`1 2`, and the last line is a `SyntaxError: Unexpected token '='` — at the start of a statement `{` opens a block, not an object. Wrapping in parentheses fixes it. (Note: a `SyntaxError` is caught before any code runs, so in a real file the `console.log` would not run either.)
</details>

---

**Q17 — The semicolon trap**
```js
let a = 1, b = 2
[a, b] = [b, a]
console.log(a, b)
```
<details>
<summary>Answer</summary>

`ReferenceError: Cannot access 'b' before initialization`.
With no semicolon, line 2 is glued onto line 1: `let a = 1, b = 2[a, b] = [b, a]`. Fix: add semicolons, or start the second line with `;`.
</details>

---

**Q18 — Destructured method**
```js
const user = { name: "Asha", greet() { return this.name; } };
const { greet } = user;
console.log(greet(), user.greet());
```
<details>
<summary>Answer</summary>

`undefined "Asha"` (sloppy). In strict mode `greet()` throws a `TypeError`. Destructuring extracts the function and drops the object, so the call is a plain call.
</details>

---

**Q19 — Shallow read**
```js
const o = { n: { v: 1 } };
const { n } = o;
n.v = 2;
console.log(o.n.v, n === o.n);
```
<details>
<summary>Answer</summary>

`2 true`. `n` is a reference to the same inner object.
</details>

---

**Q20 — Primitives**
```js
const { length } = "abc";
const [a, b] = "hi";
const { x } = 5;
console.log(length, a, b, x);
```
<details>
<summary>Answer</summary>

`3 "h" "i" undefined`. Strings are wrapped (autoboxing) and are iterable; a number has no `x`, so `undefined` — no error.
</details>

---

**Q21 — Which of these throw?**
```js
const { a } = null;
const [b] = null;
const { c } = 5;
```
<details>
<summary>Answer</summary>

Line 1: `TypeError: Cannot destructure property 'a' of 'null' as it is null.`
Line 2: `TypeError: null is not iterable`.
Line 3: no error (`c` is `undefined`).
</details>

---

**Q22 — Array pattern on an object, object pattern on an array**
```js
try { const [a] = { 0: 1 }; } catch (e) { console.log(e.name); }

const { 0: first, length } = ["x", "y"];
console.log(first, length);
```
<details>
<summary>Answer</summary>

`TypeError`, then `"x" 2`. Array patterns need an iterable; object patterns work on anything that can become an object, including arrays (index and `length` are just keys).
</details>

---

**Q23 — Inherited properties**
```js
const { toString } = {};
const parent = { inh: 1 };
const child = Object.create(parent);
const { inh } = child;
console.log(typeof toString, inh);
```
<details>
<summary>Answer</summary>

`"function" 1`. Destructuring reads `obj.key`, which finds inherited values.
</details>

---

**Q24 — Order of getters**
```js
const log = [];
const o = {
  get a() { log.push("a"); return 1; },
  get b() { log.push("b"); return 2; }
};
const { b, a } = o;
console.log(log);
```
<details>
<summary>Answer</summary>

`["b", "a"]`. The pattern's order decides, not the object's.
</details>

---

**Q25 — Fibonacci with a swap**
```js
let [a, b] = [0, 1];
for (let i = 0; i < 5; i++) {
  [a, b] = [b, a + b];
}
console.log(a, b);
```
<details>
<summary>Answer</summary>

`5 8`. Each iteration builds `[b, a + b]` from the old values, then assigns both at once. Sequence: (0,1) → (1,1) → (1,2) → (2,3) → (3,5) → (5,8).
</details>

---

**Q26 — Rename + default**
```js
const { name: n = "Guest", age: a = 0 } = { age: null };
console.log(n, a);
```
<details>
<summary>Answer</summary>

`"Guest" null`. `name` is missing → default. `age` is `null` → no default.
</details>

---

**Q27 — Computed keys in a pattern**
```js
const key = "a";
const { [key]: v, ["b" + 1]: w = 0 } = { a: 5 };
console.log(v, w);
```
<details>
<summary>Answer</summary>

`5 0`. The key `"a"` is looked up and stored in `v`; key `"b1"` is missing so `w` takes its default `0`.
</details>

---

**Q28 — `const` from destructuring**
```js
const { b } = { b: 1 };
try { b = 2; } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`TypeError`. Variables created by destructuring follow their keyword: `const` stays `const`.
</details>

---

## What Interviewers Test

**1. Object vs array patterns**
Key-based vs position-based; rename syntax (`key: newName`); skipping with gaps.

**2. Defaults**
Only `undefined` triggers them; `null`, `0`, `""` don't. Lazy evaluation; earlier vs later references (TDZ). This is the same rule as default parameters.

**3. The function-parameter trap**
`function f({ a })` crashes when called with no argument; the fix is `= {}`. Know the difference between `{ a = 5 } = {}` and `{ a } = { a: 5 }`.

**4. Nested destructuring**
The middle key doesn't become a variable; a missing parent throws; a default guard rescues `undefined` but not `null`.

**5. Swapping and assignment destructuring**
`[a, b] = [b, a]`; parentheses around `({ a } = obj)`; the leading-`[` semicolon trap.

**6. What it is not**
Not a copy (shallow read of references), doesn't change the source, reads inherited properties, destructured methods lose `this`.

**7. Errors**
Which inputs throw and why: `null`/`undefined`, non-iterables with array patterns, missing parents in nested patterns.

---

## Mental Models

**A pattern is a stencil.**
Hold it over the data. The holes in the stencil (`{ name }`, `[first]`) pick out matching values; everything else is ignored.

**Object pattern = looking things up by label. Array pattern = counting from the left.**

**`key: newName` reads "from `key` into `newName`".**
The thing on the left of the colon is where to look; the thing on the right is where to put it. (A literal does the opposite: left is the label, right is the value you supply.)

**A default is a spare for something *absent*, not something *empty*.**
`undefined` means absent. `null`, `0`, `""` are present values — they are kept.

**Nested pattern = following a path, not collecting every stop.**
`{ address: { city } }` walks through `address` to reach `city` — and only `city` is kept.

**Destructuring = `const x = obj.x`, written shorter.**
It copies references, not objects. If you remember that, every behaviour above follows.

---

*Previous: Objects — properties, descriptors, freeze/seal*
*Next: Spread / Rest — clone and merge, rest in function parameters, rest vs spread*