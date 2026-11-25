# Objects — Properties, Descriptors, `freeze` / `seal`, Inspecting Keys

---

## What is an Object?

An object is a collection of **properties**. Each property is a **key → value** pair.

```js
const user = {
  name: "Asha",        // key: "name",  value: "Asha"
  age: 25,             // key: "age",   value: 25
  greet() {            // a function stored as a value is called a method
    return "Hi " + this.name;
  }
};
```

Three facts to hold on to from the start:

```
1. Keys are always strings.
   (There is one other kind of key, Symbol — covered in Week 5. Ignore it for now.)
2. Values can be anything — numbers, strings, functions, other objects.
3. Objects are reference types.
   You learned this in Week 1. Everything today builds on it.
```

> **Today's map:**
> create → read / update / delete → reference behaviour → computed keys →
> checking if a key exists → listing keys → property descriptors → freeze / seal

---

## Part 1 — Creating Objects

### Object literal (the normal way)

```js
const point = { x: 1, y: 2 };
const empty = {};
```

### Property shorthand

When the variable name equals the key name, write it once:

```js
const name = "Asha";
const age  = 25;

const user = { name, age };      // same as { name: name, age: age }
```

### Method shorthand

```js
const counter = {
  count: 0,
  inc() { this.count++; }        // same as inc: function() { ... }
};
```

Method shorthand is the form to use for methods that need `this`
(you saw in Week 2 why an arrow function is the wrong choice here).
Methods written this way are also **not constructors** — `new counter.inc()` throws a `TypeError`.

### Keys are always strings

```js
const o = { 1: "one", true: "yes" };

Object.keys(o);        // ["1", "true"]  ← both became strings
o[1];                  // "one"
o["1"];                // "one"          ← same property
```

Keys that are not valid identifiers need quotes:

```js
const headers = { "Content-Type": "application/json", "x-id": 7 };
headers["Content-Type"];     // bracket access is required
```

### Duplicate keys — the last one wins

```js
const o = { a: 1, b: 2, a: 3 };
o;                   // { a: 3, b: 2 }   ← a keeps its first position, takes the last value
```

### Other ways (you will meet them later)

```js
const a = new Object();            // legal, but nobody writes this
const b = Object.create(someObj);  // creates an object linked to another object
                                   // (explained properly on Saturday — Prototypes)
```

---

## Part 2 — Reading, Updating, Adding, Deleting

### Reading: dot vs bracket

```js
const user = { name: "Asha", "home city": "Jaipur" };

user.name;               // "Asha"        dot: key must be a valid identifier
user["name"];            // "Asha"        bracket: any string expression
user["home city"];       // "Jaipur"      bracket REQUIRED (has a space)

const key = "name";
user[key];               // "Asha"        bracket REQUIRED for dynamic keys
user.key;                // undefined     ← looks for a key literally named "key"
```

**Use bracket notation when:** the key is stored in a variable, computed, or not a valid identifier.

### Reading something that doesn't exist

```js
const o = { a: 1 };

o.missing;               // undefined          — no error
o.missing.deep;          // TypeError: Cannot read properties of undefined (reading 'deep')
```

A missing property gives `undefined`. Reading a property **of** `undefined` or `null` throws.
Until optional chaining (Week 5), guard with an explicit check:

```js
if (o.missing !== undefined) { /* safe to use o.missing.deep */ }
```

### Updating and adding — the same syntax

```js
const o = { a: 1 };

o.a = 2;                 // update existing
o.b = 3;                 // add new (no declaration needed)
o["c d"] = 4;            // add with bracket
```

JavaScript objects can grow and change freely unless you restrict them (Part 8).

### `const` does not protect an object's contents

```js
const o = { a: 1 };
o.a = 99;                // ✅ allowed — mutation
o.b = 2;                 // ✅ allowed — mutation
o = {};                  // ❌ TypeError — reassignment of the variable
```

You know this from Week 1: `const` locks the **binding**, not the object.

### Deleting: the `delete` operator

```js
const o = { a: 1, b: 2 };

delete o.a;              // true
o;                       // { b: 2 }
```

Rules of `delete`:

```
• It removes an OWN property from the object. It does not free memory directly;
  it only removes the property (memory is reclaimed by GC if nothing else refers to the value).
• It returns true if the property is gone afterwards — including when it never existed.
• It returns false for a non-configurable property (sloppy mode),
  or throws TypeError (strict mode). Descriptors are in Part 7.
• It cannot remove an INHERITED property.
```

```js
const o = { a: 1 };
Object.defineProperty(o, "x", { value: 1 });   // non-configurable (see Part 7)

delete o.a;              // true
delete o.nope;           // true   ← didn't exist, so "the property is gone"
delete o.x;              // false  ← sloppy mode, silently refused
                         // strict mode: TypeError: Cannot delete property 'x' of #<Object>
```

### `delete` vs setting to `undefined`

```js
const a = { x: 1 }; delete a.x;        // key removed
const b = { x: 1 }; b.x = undefined;   // key stays, value is undefined
```

| | `delete o.x` | `o.x = undefined` |
|---|---|---|
| `"x" in o` | `false` | `true` |
| `Object.keys(o)` | no `"x"` | includes `"x"` |
| `o.x` | `undefined` | `undefined` |

> Engine note (implementation-dependent): deleting properties can make an object slower in some engines. In hot code, prefer building a new object without the key, or set the value to `null`/`undefined` if "key present" semantics are acceptable.

### Property order — a favourite interview question

Keys come out in this order, always:

```
1. Integer-like keys ("0", "1", "42") — in ascending numeric order
2. All other string keys — in the order they were created
```

```js
const o = { b: 1, 2: "x", a: 2, 1: "y" };
Object.keys(o);          // ["1", "2", "b", "a"]
```

Two more rules:

```js
const o = { a: 1, b: 2, c: 3 };

delete o.a;
o.a = 4;                 // re-adding after delete puts it at the END
Object.keys(o);          // ["b", "c", "a"]

o.b = 9;                 // updating an existing key keeps its position
Object.keys(o);          // ["b", "c", "a"]
```

---

## Part 3 — Reference Behaviour With Objects

Quick recap of Week 1, then the angles interviewers add on top.

```js
const a = { n: { v: 1 } };
const b = a;             // b is a copy of the REFERENCE — same object
b.n.v = 2;
a.n.v;                   // 2

a === b;                 // true   — same reference
{} === {};               // false  — two different objects
```

### Objects hold references to other objects

Nested objects are separate objects that the outer object **points to**:

```js
const inner = { v: 1 };
const x = { n: inner };
const y = { n: inner };    // two different outer objects, ONE shared inner object

y.n.v = 99;
x.n.v;                     // 99  ← the change is visible through x
x === y;                   // false
x.n === y.n;               // true
```

This "shared inner object" idea is the core of why copying objects is tricky.
That is exactly what Wednesday and Thursday (spread, shallow vs deep copy) are about.

### Objects passed to functions

```js
function rename(user) {
  user.name = "Changed";       // mutation — the caller sees it
  user = { name: "Other" };    // reassignment — only the local variable changes
}

const u = { name: "Asha" };
rename(u);
u.name;                        // "Changed"
```

JavaScript passes a copy of the reference (Week 1). Mutating through it is visible; reassigning is not.

---

## Part 4 — Computed Property Names

A **computed property name** lets you use an expression as a key while creating an object. Put the expression in square brackets:

```js
const field = "email";

const user = {
  [field]: "a@b.com",            // key "email"
  ["id" + 1]: 101,               // key "id1"
  [1 + 1]: "two"                 // key "2"
};

user;                            // { "2": "two", email: "a@b.com", id1: 101 }
```

(Integer-like key `"2"` is listed first — the ordering rule from Part 2.)

Computed names work for methods too, and the method gets that name:

```js
const action = "save";
const api = { [action + "Draft"]() { return "saved"; } };
api.saveDraft();                 // "saved"
api.saveDraft.name;              // "saveDraft"
```

### The expression is evaluated once, left to right, at creation

```js
let i = 0;
const o = { [i++]: i++, [i++]: i++ };
// key expr → 0 (i=1), value → 1 (i=2), key expr → 2 (i=3), value → 3 (i=4)
o;                               // { "0": 1, "2": 3 }
i;                               // 4
```

### The key is converted to a string

```js
const o = {};

o[1] = "a";
o["1"];                          // "a"        — same key
o[1] === o["1"];                 // true

const k = {};
o[k] = "object key";
Object.keys(o);                  // ["1", "[object Object]"]   ← the object became a string

const u = { [undefined]: 1, [null]: 2, [true]: 3 };
Object.keys(u);                  // ["undefined", "null", "true"]
```

> This is the exact bug from the Week 2 memoization questions: two different objects used as keys collide on `"[object Object]"`.
> A plain object can never key by object identity. The built-in tool for that is `Map` / `WeakMap`; for now, just remember the limitation.

### Where computed keys are used in real code

```js
// 1. Building an object from dynamic input
function makeField(name, value) {
  return { [name]: value };
}
makeField("email", "a@b.com");   // { email: "a@b.com" }

// 2. Counting things (bracket access on an existing object)
const counts = {};
counts["apple"] = (counts["apple"] || 0) + 1;
```

⚠️ **The counting pattern has a famous bug with plain objects** — see Implementation Task 6.

---

## Part 5 — Does This Property Exist? Own vs Inherited

### First, the minimum you need about "inherited"

Objects can be **linked** to another object. If a property isn't found on the object itself, JavaScript looks at the object it is linked to. That link is called the prototype — Saturday teaches it fully.
Today you only need the vocabulary:

```
OWN property       → lives on the object itself
INHERITED property → found through the link to another object
```

To demonstrate, we create a linked object with `Object.create`:

```js
const parent = { inherited: 1 };
const child  = Object.create(parent);   // child is linked to parent
child.own = 2;

child.own;               // 2   — own property
child.inherited;         // 1   — found through the link (inherited)
```

Every ordinary object is linked to a built-in object that provides things like `toString`.
So `{}` already has **inherited** properties:

```js
({}).toString;           // function — inherited, even though {} looks empty
```

### Writing never changes the parent — it creates an own property (shadowing)

```js
child.inherited = 9;

parent.inherited;                  // 1  ← parent untouched
child.inherited;                   // 9  ← new OWN property shadows the inherited one
Object.hasOwn(child, "inherited"); // true

delete child.inherited;            // removes only the own copy
child.inherited;                   // 1  ← the inherited value shows through again
```

`delete` only ever removes own properties.

### Three ways to check for a key

```js
const parent = { inherited: 1 };
const child  = Object.create(parent);
child.own = 2;
```

**1. The `in` operator — own AND inherited**

```js
"own" in child;               // true
"inherited" in child;         // true   ← inherited counts
"toString" in child;          // true   ← even built-in inherited ones
"nope" in child;              // false
```

`in` checks **keys**, not values, and works regardless of the value or enumerability.

**2. `Object.hasOwn(obj, key)` — own only (ES2022)**

```js
Object.hasOwn(child, "own");           // true
Object.hasOwn(child, "inherited");     // false
Object.hasOwn({}, "toString");         // false
```

**3. The older `obj.hasOwnProperty(key)`**

Same idea as `hasOwn`, but it is a method found on the objects you create, which causes two problems:

```js
const d = Object.create(null);         // an object linked to NOTHING
d.k = 1;
d.hasOwnProperty("k");                 // TypeError: d.hasOwnProperty is not a function
Object.hasOwn(d, "k");                 // true  ✅

const o = { hasOwnProperty() { return true; } };   // someone overrides it
o.hasOwnProperty("zzz");               // true  ← wrong
Object.hasOwn(o, "zzz");               // false ✅
Object.prototype.hasOwnProperty.call(o, "zzz");    // false (the pre-ES2022 safe form)
```

**Rule:** prefer `Object.hasOwn`. In older environments use `Object.prototype.hasOwnProperty.call(obj, key)`.

### The tempting-but-wrong check: `obj.key !== undefined`

```js
const o = { x: undefined };

o.x !== undefined;            // false  ← says "no x"
"x" in o;                     // true
Object.hasOwn(o, "x");        // true
```

A property can exist with the value `undefined`. Comparing to `undefined` can't tell "missing" from "present but undefined".

### `in` checks keys — including on arrays

```js
const arr = [10, 20];
1 in arr;                     // true   — index 1 exists
10 in arr;                    // false  — 10 is a value, not a key
2 in arr;                     // false  — no index 2
"length" in arr;              // true

"a" in "str";                 // TypeError: Cannot use 'in' operator to search for 'a' in str
```

### Which check should I use?

| Question | Use |
|---|---|
| "Does this object itself have this key?" | `Object.hasOwn(obj, key)` |
| "Is this key reachable at all (own or inherited)?" | `key in obj` |
| "Does it have a *usable* value?" | `obj[key] !== undefined` (value check, not an existence check) |
| "Is the object empty?" | `Object.keys(obj).length === 0` (see Task 2 for the catch) |

---

## Part 6 — Listing Properties

### `Object.keys`, `Object.values`, `Object.entries`

All three return **arrays** of the object's **own, enumerable, string-keyed** properties, in the order from Part 2.

```js
const user = { name: "Asha", age: 25 };

Object.keys(user);         // ["name", "age"]
Object.values(user);       // ["Asha", 25]
Object.entries(user);      // [["name", "Asha"], ["age", 25]]
```

Array methods like `map` and `forEach` come on Friday. Until then, loop with a plain `for` over the returned array:

```js
const keys = Object.keys(user);
for (let i = 0; i < keys.length; i++) {
  console.log(keys[i], user[keys[i]]);
}
// name Asha
// age 25
```

Things to know:

```js
// The result is a SNAPSHOT — a new array. Changing the object later doesn't change it.
const o = { a: 1 };
const e = Object.entries(o);
o.a = 2;
e;                          // [["a", 1]]   still the old value

// Primitives are converted to objects first
Object.keys("ab");          // ["0", "1"]
Object.keys(5);             // []

// null and undefined cannot be converted
Object.keys(null);          // TypeError: Cannot convert undefined or null to object

// Arrays are objects
Object.keys([5, 6]);        // ["0", "1"]
```

### The inverse: `Object.fromEntries`

Builds an object from a list of `[key, value]` pairs:

```js
Object.fromEntries([["a", 1], ["b", 2]]);    // { a: 1, b: 2 }
```

### `for...in` — includes inherited properties

```js
const parent = { inherited: 1 };
const child  = Object.create(parent);
child.own = 2;

Object.keys(child);                 // ["own"]

const found = [];
for (const key in child) {
  found.push(key);
}
found;                              // ["own", "inherited"]   ← inherited enumerable keys appear
```

`for...in` visits every **enumerable** string key, **own and inherited**. That is why it is often combined with a `hasOwn` check, or replaced with `Object.keys`.

### Which method lists what?

| | Own | Inherited | Non-enumerable | Notes |
|---|:-:|:-:|:-:|---|
| `Object.keys` / `values` / `entries` | ✅ | ❌ | ❌ | arrays, string keys only |
| `for...in` | ✅ | ✅ | ❌ | enumerable keys only |
| `Object.getOwnPropertyNames(o)` | ✅ | ❌ | ✅ | own string keys, enumerable or not |
| `key in o` | ✅ | ✅ | ✅ | single-key existence |
| `Object.hasOwn(o, key)` | ✅ | ❌ | ✅ | single-key existence |

(Symbol keys are skipped by the first four; Symbols arrive in Week 5.)

---

# Part 7 — Property Descriptors, Data Properties & Accessors

## 1. First, understand what a property is

An object is a collection of properties. A property is essentially a key-value association belonging to an object.

```js
const user = {
    name: "Asha",
    age: 25
};
```

Here:
- `name` and `age` are property keys.
- `"Asha"` and `25` are their respective values.

But a property contains more information than just its key and value.

JavaScript maintains additional internal attributes that determine how a property behaves.

These attributes are called **property attributes**.

A **property descriptor** is an object that describes these attributes.

Think of it this way:

- Property = The actual property on an object.
- Property attributes = Rules controlling its behaviour.
- Property descriptor = An object that describes those rules.

JavaScript provides methods to inspect and configure these descriptors.

## 2. Two types of properties in JavaScript

JavaScript objects have two fundamental kinds of properties:

1. Data properties
2. Accessor properties

Understanding this distinction is essential because the two types have different descriptor attributes.

### A. Data properties

A data property directly stores a value.

```js
const user = {
    name: "Asha",
    age: 25
};
```

Both `name` and `age` are data properties.

When we access:

```js
user.name;
```

JavaScript retrieves the value stored in that property.

A data property has four attributes:

| Attribute | Meaning |
|---|---|
| `value` | The value stored in the property |
| `writable` | Whether the stored value can be changed through assignment |
| `enumerable` | Whether the property participates in enumerable-property operations |
| `configurable` | Whether the property can be deleted or its descriptor can be reconfigured |

Let's understand each attribute.

### B. Accessor properties

Unlike data properties, accessor properties do not directly store a value.

Instead, they use functions to handle reading and writing.

These functions are called:

- Getter (`get`)
- Setter (`set`)

Example:

```js
const user = {
    firstName: "Asha",
    lastName: "Rao",

    get fullName() {
        return this.firstName + " " + this.lastName;
    },

    set fullName(value) {
        const parts = value.split(" ");

        this.firstName = parts[0];
        this.lastName = parts[1];
    }
};
```

Here, `fullName` is an accessor property.

Notice that we haven't stored a separate `fullName` value. Instead, we have defined functions that control how the property behaves.

Accessor properties have four descriptor attributes:

| Attribute | Meaning |
|---|---|
| `get` | Function executed when the property is read |
| `set` | Function executed when a value is assigned |
| `enumerable` | Whether the property participates in enumerable-property operations |
| `configurable` | Whether the property can be deleted or its descriptor reconfigured |

**Important distinction:**

| Data property | Accessor property |
|---|---|
| `value` | `get` |
| `writable` | `set` |
| `enumerable` | `enumerable` |
| `configurable` | `configurable` |

A property is either a data property or an accessor property. Its descriptor cannot simultaneously contain the data-property fields (`value`, `writable`) and accessor-property fields (`get`, `set`).

---

## 3. Property Descriptors — Inspecting properties

JavaScript provides `Object.getOwnPropertyDescriptor()` to inspect the descriptor of an individual own property.

```js
const user = {
    name: "Asha"
};

Object.getOwnPropertyDescriptor(user, "name");
```

Output:

```js
{
    value: "Asha",
    writable: true,
    enumerable: true,
    configurable: true
}
```

This tells us:
- `value: "Asha"` → The property currently stores `"Asha"`.
- `writable: true` → We can change its value through assignment.
- `enumerable: true` → It appears in enumerable-property operations.
- `configurable: true` → We can delete it or reconfigure its descriptor.

If the property doesn't exist as an own property:

```js
Object.getOwnPropertyDescriptor(user, "missing");
// undefined
```

To retrieve descriptors for all own properties:

```js
Object.getOwnPropertyDescriptors(user);
```

This returns an object containing the descriptors of all own properties, including symbol-keyed properties.

**Important:** `Object.getOwnPropertyDescriptor()` only checks own properties. It does not search the prototype chain.

---

## 4. Data property attributes in detail

### A. `value`

The actual value stored in the data property.

```js
const user = { age: 25 };

Object.getOwnPropertyDescriptor(user, "age");

// {
//     value: 25,
//     writable: true,
//     enumerable: true,
//     configurable: true
// }
```

### B. `writable`

Controls whether the property's stored value can be changed through assignment.

```js
const user = {};

Object.defineProperty(user, "age", {
    value: 25,
    writable: false
});

user.age = 30;

console.log(user.age); // 25
```

Because `writable` is false, the assignment cannot change the property.

In non-strict mode, the failed assignment is silently ignored.

In strict mode, it throws a `TypeError`.

```js
"use strict";

user.age = 30; // TypeError
```

**Remember:** `writable: false` does not make the entire object immutable. It only prevents changing that property's stored value through ordinary assignment.

### C. `enumerable`

Controls whether a property participates in enumerable-property operations.

Consider:

```js
const user = {
    name: "Asha"
};

Object.defineProperty(user, "secret", {
    value: "hidden",
    enumerable: false
});
```

Now:

```js
Object.keys(user);
// ["name"]

Object.values(user);
// ["Asha"]

Object.entries(user);
// [["name", "Asha"]]
```

But the property still exists:

```js
user.secret;                  // "hidden"
Object.hasOwn(user, "secret"); // true
"secret" in user;              // true

Object.getOwnPropertyNames(user);
// ["name", "secret"]
```

**What enumerable: false actually means:**

It hides the property from enumerable-property operations. It does NOT make the property private or inaccessible.

| Affected by `enumerable: false` | Not affected |
|---|---|
| `Object.keys()` | Direct property access |
| `Object.values()` | `Object.hasOwn()` |
| `Object.entries()` | `in` operator |
| `for...in` | `Object.getOwnPropertyNames()` |
| `JSON.stringify()` for ordinary string-keyed data properties | Property existence |
| Object spread and `Object.assign()` copying | |

Note: `Object.keys()`, `Object.values()`, and `Object.entries()` consider own enumerable string-keyed properties. `for...in` also considers inherited enumerable string-keyed properties. Spread and `Object.assign()` additionally copy enumerable own symbol-keyed properties.

### D. `configurable`

Controls whether a property can be deleted or whether its descriptor can be reconfigured.

A configurable property allows:
- Deletion.
- Changing `enumerable`.
- Changing `configurable`.
- Changing between data and accessor property types, subject to the applicable rules.

Example:

```js
const user = { age: 25 };

Object.defineProperty(user, "age", {
    configurable: false
});

delete user.age; // false in non-strict mode
```

In strict mode, attempting to delete this property throws a `TypeError`.

A non-configurable property is effectively locked against most descriptor changes.

There is one important exception involving `writable`, explained below.

---

## 5. Creating properties using Object.defineProperty()

`Object.defineProperty()` allows us to create a property and explicitly configure its descriptor attributes.

Syntax:

```js
Object.defineProperty(object, propertyKey, descriptor);
```

Example:

```js
const user = {};

Object.defineProperty(user, "id", {
    value: 101,
    writable: false,
    enumerable: true,
    configurable: false
});

console.log(user.id); // 101
```

We have created a data property with:
- Value: `101`
- Writable: `false`
- Enumerable: `true`
- Configurable: `false`

To define multiple properties:

```js
Object.defineProperties(user, {
    name: {
        value: "Asha",
        writable: true,
        enumerable: true,
        configurable: true
    },

    age: {
        value: 25,
        writable: false,
        enumerable: true,
        configurable: false
    }
});
```

### ⚠️ Important: Default descriptor values

The defaults depend on how the property is created.

**Case 1: Object literal or assignment**

```js
const obj = { x: 10 };

Object.getOwnPropertyDescriptor(obj, "x");
```

Result:

```js
{
    value: 10,
    writable: true,
    enumerable: true,
    configurable: true
}
```

Properties created through ordinary assignment also have these three attributes set to true, assuming a new ordinary own data property is created.

**Case 2: Object.defineProperty()**

```js
const obj = {};

Object.defineProperty(obj, "x", {
    value: 10
});
```

Result:

```js
{
    value: 10,
    writable: false,
    enumerable: false,
    configurable: false
}
```

When creating a new property, omitted descriptor attributes default to false.

| Creation method | writable | enumerable | configurable |
|---|---|---|---|
| Object literal | true | true | true |
| Ordinary assignment creating a new own property | true | true | true |
| defineProperty with omitted attributes | false | false | false |

**Important exception:** When `defineProperty()` modifies an existing property, omitted attributes are left unchanged. They do not automatically become false.

---

## 6. Non-configurable properties — Important interview rules

Consider:

```js
const obj = {};

Object.defineProperty(obj, "x", {
    value: 10,
    writable: true,
    enumerable: true,
    configurable: false
});
```

The property is now non-configurable.

What can we change?

### Allowed operations

```js
obj.x = 20; 
// Allowed because writable is true.

Object.defineProperty(obj, "x", {
    value: 30
});
// Allowed because writable is still true.

Object.defineProperty(obj, "x", {
    writable: false
});
// Allowed: true → false.
```

### Forbidden operations

```js
Object.defineProperty(obj, "x", {
    configurable: true
});
// TypeError

Object.defineProperty(obj, "x", {
    enumerable: false
});
// TypeError
```

After changing writable to false:

```js
Object.defineProperty(obj, "x", {
    writable: true
});
// TypeError
```

Changing the value to a different value is also forbidden:

```js
Object.defineProperty(obj, "x", {
    value: 100
});
// TypeError
```

However, specifying the same value is allowed:

```js
Object.defineProperty(obj, "x", {
    value: 30
});
// Allowed: value is unchanged.
```

### Summary of non-configurable data properties

| Operation | Allowed? |
|---|---|
| Delete property | No |
| Change enumerable | No |
| Change configurable | No |
| Change writable: true → false | Yes |
| Change writable: false → true | No |
| Change value while writable is true | Yes |
| Change value while writable is false | No, unless the value is unchanged |

**Remember:** `Object.defineProperty()` throws a `TypeError` when an attempted reconfiguration violates these rules. It does not silently fail.

---

## 7. Accessors — Getters and Setters in detail

Now that we understand data properties and their descriptors, let's understand accessors properly.

An accessor property is a property whose read and write operations are controlled by functions.

These functions are called getters and setters.

### A. Getters

A getter executes automatically when a property is read.

Syntax:

```js
get propertyName() {
    // logic
    return someValue;
}
```

Example:

```js
const user = {
    firstName: "Asha",
    lastName: "Rao",

    get fullName() {
        return this.firstName + " " + this.lastName;
    }
};

console.log(user.fullName);
// "Asha Rao"
```

Notice:

```js
user.fullName;
```

We don't use parentheses.

Why?

Because `fullName` is an accessor property, not a normal method call.

The getter executes automatically when JavaScript reads that property.

**Execution flow:**

1. JavaScript encounters `user.fullName`.
2. It identifies `fullName` as an accessor property.
3. Its getter executes.
4. The getter calculates the full name.
5. The returned value becomes the result of the property access.

Every read invokes the getter again. Getters are not automatically cached.

### B. Setters

A setter executes automatically when a value is assigned to a property.

Syntax:

```js
set propertyName(value) {
    // logic
}
```

Example:

```js
const user = {
    firstName: "Asha",
    lastName: "Rao",

    get fullName() {
        return this.firstName + " " + this.lastName;
    },

    set fullName(value) {
        const parts = value.split(" ");

        this.firstName = parts[0];
        this.lastName = parts[1];
    }
};

user.fullName = "Neha Iyer";

console.log(user.firstName); // "Neha"
console.log(user.lastName);  // "Iyer"
console.log(user.fullName);  // "Neha Iyer"
```

**Execution flow:**

1. JavaScript encounters `user.fullName = "Neha Iyer"`.
2. It identifies `fullName` as an accessor property.
3. Its setter executes.
4. `"Neha Iyer"` is passed as the setter's argument.
5. The setter splits the string and updates the underlying properties.

### C. Getter vs Setter

| Getter | Setter |
|---|---|
| Triggered when reading a property | Triggered when assigning a property |
| Uses `get` | Uses `set` |
| Returns a value | Receives the assigned value |
| Cannot have parameters | Takes exactly one parameter |
| Can exist without a setter | Can exist without a getter |

### D. Getters and setters can perform validation

Example:

```js
const account = {
    _balance: 1000,

    get balance() {
        return this._balance;
    },

    set balance(value) {
        if (value < 0) {
            throw new Error("Balance cannot be negative");
        }

        this._balance = value;
    }
};

account.balance = 500;  // Allowed
account.balance = -100; // Error
```

The setter provides control over how the property's underlying value is modified.

### E. Getter without a setter

```js
const user = {
    get age() {
        return 25;
    }
};

console.log(user.age); // 25

user.age = 30;
console.log(user.age); // 25 (non-strict mode)
```

Since there is no setter, assignment cannot change the accessor property.

In strict mode, the assignment throws a `TypeError`.

A getter-only accessor is effectively read-only through ordinary assignment.

### F. Setter without a getter

```js
const user = {
    _age: 25,

    set age(value) {
        this._age = value;
    }
};

user.age = 30;

console.log(user.age);  // undefined
console.log(user._age); // 30
```

The setter handles assignment, but there is no getter to handle reading `age`.

Therefore, reading `user.age` returns `undefined`.

---

## 8. Accessor property descriptors

Now let's connect getters and setters to property descriptors.

Consider:

```js
const user = {
    firstName: "Asha",

    get name() {
        return this.firstName;
    },

    set name(value) {
        this.firstName = value;
    }
};
```

Inspect its descriptor:

```js
Object.getOwnPropertyDescriptor(user, "name");
```

Result (conceptually):

```js
{
    get: function,
    set: function,
    enumerable: true,
    configurable: true
}
```

Notice something important:

The descriptor contains:
- `get`
- `set`
- `enumerable`
- `configurable`

It does NOT contain:
- `value`
- `writable`

Why?

Because `name` is an accessor property, not a data property.

### Creating accessor properties with Object.defineProperty()

We can create getters and setters without using object-literal getter/setter syntax.

```js
const user = {
    firstName: "Asha"
};

Object.defineProperty(user, "name", {
    get() {
        return this.firstName;
    },

    set(value) {
        this.firstName = value;
    },

    enumerable: true,
    configurable: true
});

console.log(user.name); // "Asha"

user.name = "Neha";

console.log(user.firstName); // "Neha"
```

Notice that `get` and `set` are functions supplied inside the descriptor.

**Default values matter here too:**

```js
const obj = {};

Object.defineProperty(obj, "x", {
    get() {
        return 10;
    }
});
```

Its descriptor has:

```js
{
    get: function,
    set: undefined,
    enumerable: false,
    configurable: false
}
```

For accessor properties created using `defineProperty()`, omitted `enumerable` and `configurable` attributes default to false.

For getters and setters defined directly inside an object literal, both default to true.

---

## 9. Important rules about accessor descriptors

### Rule 1: Data and accessor descriptors cannot be mixed

This is invalid:

```js
Object.defineProperty({}, "x", {
    value: 10,
    get() {
        return 20;
    }
});
```

It throws a `TypeError`.

A descriptor cannot simultaneously define:
- Data-property fields: `value` or `writable`.
- Accessor-property fields: `get` or `set`.

### Rule 2: Getters and setters use the receiver as `this`

This is particularly important for prototype inheritance and interview questions.

```js
const parent = {
    first: "A",
    last: "B",

    get full() {
        return this.first + " " + this.last;
    }
};

const child = Object.create(parent);

child.first = "X";

console.log(parent.full); // "A B"
console.log(child.full);  // "X B"
```

Why does `child.full` return `"X B"`?

- `child` does not have its own `full` property.
- JavaScript searches the prototype chain and finds the getter on `parent`.
- The getter executes with `this` referring to `child`, the object through which the property was accessed.
- Therefore, `this.first` resolves to `child.first`, which is `"X"`.

The getter is inherited, but its `this` value is determined by the property access receiver, not automatically fixed to the object where the getter was originally defined.

### Rule 3: Accessors are not automatically cached

```js
const counter = {
    count: 0,

    get next() {
        return ++this.count;
    }
};

console.log(counter.next); // 1
console.log(counter.next); // 2
console.log(counter.next); // 3
```

Every property read invokes the getter again.

A getter is a function executed through property-access syntax. It can also have side effects.

---

## 10. Real-world applications

| Use case | Mechanism |
|---|---|
| Read-only property | `writable: false` or getter without setter |
| Hide a property from enumerable operations | `enumerable: false` |
| Prevent deletion and descriptor reconfiguration | `configurable: false` |
| Calculate a property dynamically | Getter |
| Validate incoming values | Setter |
| Control access to underlying data | Getter and setter together |

**Important clarification:** `enumerable: false` does not make a property private. Similarly, naming a property `_age` is only a naming convention; it does not provide actual privacy.

For actual private instance fields, JavaScript supports the `#field` syntax in classes. We will cover that separately.

---

## 11. Quick revision sheet

### Two property types

**Data property**
- Stores a value directly.
- Descriptor: `value`, `writable`, `enumerable`, `configurable`.

**Accessor property**
- Uses functions to handle reading and writing.
- Descriptor: `get`, `set`, `enumerable`, `configurable`.

### Important APIs

| API | Purpose |
|---|---|
| `Object.getOwnPropertyDescriptor()` | Get descriptor of one own property |
| `Object.getOwnPropertyDescriptors()` | Get descriptors of all own properties |
| `Object.defineProperty()` | Create or reconfigure one property |
| `Object.defineProperties()` | Create or reconfigure multiple properties |

### Defaults to memorise

| Creation method | writable / enumerable / configurable |
|---|---|
| Object literal data property | true / true / true |
| Object literal accessor property | N/A / true / true |
| New data property via `defineProperty()` with omitted attributes | false / false / false |
| New accessor property via `defineProperty()` with omitted attributes | N/A / false / false |

When modifying an existing property, omitted descriptor fields remain unchanged.

### Interview traps

1. `writable: false` does not make the entire object immutable.
2. `enumerable: false` does not make a property private.
3. `configurable: false` still permits changing a writable data property's value while `writable` remains true.
4. A non-configurable, writable data property can transition from `writable: true` to `writable: false`, but not back.
5. A getter is invoked on property access, not by calling it with parentheses.
6. A setter is invoked on property assignment.
7. A getter without a setter cannot be modified through ordinary assignment.
8. Accessor descriptors do not have `value` or `writable`.
9. Data descriptors cannot contain `get` or `set`.
10. An inherited getter executes with the accessing object as `this`.
11. Getters are not automatically cached.
12. `Object.getOwnPropertyDescriptor()` only examines own properties, not inherited properties.

---
## Part 8 — `preventExtensions`, `seal`, `freeze`

These three lock an object down in increasing strength.

```js
Object.preventExtensions(o);   // can't ADD properties
Object.seal(o);                // + can't DELETE / reconfigure (existing values still writable)
Object.freeze(o);              // + can't CHANGE existing data values
```

All three **return the same object** and are **irreversible**.

### What each one allows

| Operation | normal | `preventExtensions` | `seal` | `freeze` |
|---|:-:|:-:|:-:|:-:|
| Add a new property | ✅ | ❌ | ❌ | ❌ |
| Delete a property | ✅ | ✅ | ❌ | ❌ |
| Change an existing value | ✅ | ✅ | ✅ | ❌ |
| Reconfigure (`defineProperty`) | ✅ | ✅ | ❌ (value can still change if writable) | ❌ |
| Predicate | | `isExtensible` → `false` | `isSealed` → `true` | `isFrozen` → `true` |

How it works internally:

```
seal   = make the object non-extensible  +  every property configurable:false
freeze = seal                            +  every DATA property writable:false
```

Prove it with descriptors:

```js
Object.getOwnPropertyDescriptor(Object.seal({ a: 1 }), "a");
// { value: 1, writable: true,  enumerable: true, configurable: false }

Object.getOwnPropertyDescriptor(Object.freeze({ a: 1 }), "a");
// { value: 1, writable: false, enumerable: true, configurable: false }
```

### Running the same three operations on each

```js
// for each object:  o.a = 2;  o.b = 3;  delete o.a;
//
//  normal            →  { b: 3 }
//  preventExtensions →  {}          (a changed, b refused, a deleted)
//  sealed            →  { a: 2 }    (a changed, b refused, delete refused)
//  frozen            →  { a: 1 }    (everything refused)
```

### Sloppy mode fails silently; strict mode throws

```js
"use strict";
const f = Object.freeze({ a: 1 });

f.a = 2;          // TypeError: Cannot assign to read only property 'a' of object '#<Object>'
f.b = 2;          // TypeError: Cannot add property b, object is not extensible
delete f.a;       // TypeError: Cannot delete property 'a' of #<Object>

const s = Object.seal({ a: 1 });
s.a = 5;          // ✅ allowed
s.b = 2;          // TypeError: Cannot add property b, object is not extensible
delete s.a;       // TypeError: Cannot delete property 'a' of #<Object>
```

Without `"use strict"`, those same lines do nothing and print no error — a classic source of "why didn't it change?" bugs.

`Object.defineProperty` is different: it **throws even in sloppy mode**:

```js
const o = Object.freeze({ a: 1 });
Object.defineProperty(o, "b", { value: 1 });
// TypeError: Cannot define property b, object is not extensible
```

### Freeze is SHALLOW

```js
const config = Object.freeze({ port: 3000, db: { host: "localhost" } });

config.port = 1;                 // ignored (sloppy) — frozen
config.db.host = "evil";         // ✅ WORKS — db is a separate, unfrozen object
config.db.host;                  // "evil"

Object.isFrozen(config);         // true
Object.isFrozen(config.db);      // false
```

`freeze` only locks the object's **own properties**. A property that *points to* another object keeps pointing to it, and that other object is untouched.

### `const` vs `seal` vs `freeze`

```js
const a = { x: 1 };    // binding locked; contents fully changeable
const b = Object.seal({ x: 1 });    // shape locked; values changeable
const c = Object.freeze({ x: 1 });  // shape locked; values locked (shallow)
```

| | Prevents reassigning the variable | Prevents add/delete | Prevents changing values |
|---|:-:|:-:|:-:|
| `const` | ✅ | ❌ | ❌ |
| `preventExtensions` | ❌ | add only | ❌ |
| `seal` | ❌ | ✅ | ❌ |
| `freeze` | ❌ | ✅ | ✅ (shallow) |

They are independent: `const` + `freeze` is the usual combination for a constant object.

### Gotchas to know

```js
// 1. An EMPTY non-extensible object counts as frozen and sealed (nothing left to change)
Object.isFrozen(Object.preventExtensions({}));      // true
Object.isFrozen(Object.preventExtensions({ a: 1 })); // false

// 2. Arrays are objects, so they freeze too — and push throws EVEN in sloppy mode
const arr = Object.freeze([1, 2]);
arr[0] = 9;                      // silently ignored (sloppy)
arr[0];                          // 1
arr.push(3);                     // TypeError: Cannot add property 2, object is not extensible

// 3. Freezing primitives does nothing and returns them
Object.freeze(5);                // 5
Object.isFrozen("abc");          // true

// 4. Freezing doesn't stop SETTERS from running — accessor properties aren't "written"
let store = 0;
const o = Object.freeze({ get v() { return store; }, set v(x) { store = x; } });
o.v = 5;
o.v;                             // 5   — the setter changed outside state
```

### Making an object *deeply* frozen

```js
function deepFreeze(obj) {
  if (obj === null || (typeof obj !== "object" && typeof obj !== "function")) return obj;
  if (Object.isFrozen(obj)) return obj;            // also stops infinite loops on cycles

  Object.freeze(obj);

  const names = Object.getOwnPropertyNames(obj);
  for (let i = 0; i < names.length; i++) {
    deepFreeze(obj[names[i]]);
  }
  return obj;
}

const cfg = deepFreeze({ db: { host: "h", ports: [1, 2] } });
cfg.db.host = "x";               // ignored
cfg.db.host;                     // "h"
cfg.db.ports.push(3);            // TypeError (frozen array)
```

Notes: the `isFrozen` check also protects against circular references; reading `obj[name]` would **run getters**; and for large state objects people usually prefer immutable *update patterns* (always create a new object) over freezing everything.

---

## How Everything Connects

```
Week 1 — references
  → objects live on the heap; variables hold references
  → const locks the binding, not the object          (Part 2)
  → nested objects are shared, not copied            (Part 3)

Week 2 — this, functions, closures
  → methods are functions stored as properties       (Part 1)
  → `this` inside getters/setters = the receiver     (Part 7)
  → the memoization key bug = computed key → string  (Part 4)

Today
  → keys: strings, ordered by a fixed rule           (Part 2)
  → existence: in  vs  hasOwn  vs  !== undefined     (Part 5)
  → listing: keys / values / entries / for...in      (Part 6)
  → attributes: writable / enumerable / configurable (Part 7)
  → locking: preventExtensions → seal → freeze       (Part 8)

Coming next
  → Tue: destructuring (pull values out of objects)
  → Wed: spread/rest (copy and merge objects)
  → Thu: shallow vs deep copy (Part 3 + freeze's shallowness)
  → Sat: prototypes (the "inherited" link in Part 5)
```

---

## Implementation Tasks

Try each before opening the solution. Uses only what has been taught so far.

### Task 1 — `deepFreeze`
Already shown in Part 8. Test it on an object containing an array and a cycle (`obj.self = obj`). It must not loop forever.

### Task 2 — `isEmpty(obj)`

<details>
<summary>Solution</summary>

```js
function isEmpty(obj) {
  return Object.keys(obj).length === 0;
}

isEmpty({});                         // true
isEmpty({ a: undefined });           // false — the key exists
```

**The catch:** non-enumerable properties are ignored, so an object with only hidden properties looks "empty":

```js
const o = {};
Object.defineProperty(o, "h", { value: 1 });
isEmpty(o);                          // true   ← but Object.getOwnPropertyNames(o) has "h"
```
Use `Object.getOwnPropertyNames(obj).length === 0` if hidden properties should count.
</details>

### Task 3 — `pick(obj, keys)` and `omit(obj, keys)`

<details>
<summary>Solution</summary>

```js
function pick(obj, keys) {
  const out = {};
  for (let i = 0; i < keys.length; i++) {
    if (Object.hasOwn(obj, keys[i])) out[keys[i]] = obj[keys[i]];
  }
  return out;
}

function omit(obj, keys) {
  const skip = Object.create(null);                 // a plain lookup table with no inherited keys
  for (let i = 0; i < keys.length; i++) skip[keys[i]] = true;

  const out = {};
  const own = Object.keys(obj);
  for (let i = 0; i < own.length; i++) {
    if (!skip[own[i]]) out[own[i]] = obj[own[i]];
  }
  return out;
}

pick({ a: 1, b: 2, c: 3 }, ["a", "c", "z"]);   // { a: 1, c: 3 }
omit({ a: 1, b: 2, c: 3 }, ["b"]);             // { a: 1, c: 3 }
```
Both return **new** objects (the original is not mutated) and copy **shallowly** — nested objects are still shared (Part 3).
</details>

### Task 4 — `invert(obj)`: swap keys and values

<details>
<summary>Solution</summary>

```js
function invert(obj) {
  const out = {};
  const keys = Object.keys(obj);
  for (let i = 0; i < keys.length; i++) {
    out[obj[keys[i]]] = keys[i];
  }
  return out;
}

invert({ a: "x", b: "y" });        // { x: "a", y: "b" }
invert({ a: "x", b: "x" });        // { x: "b" }   ← collision: the later key wins
```
Values become **string keys** (Part 4), so `invert({ a: 1 })` gives `{ "1": "a" }`, and object values would all become `"[object Object]"`.
</details>

### Task 5 — `defineReadOnly(obj, key, value)`

<details>
<summary>Solution</summary>

```js
function defineReadOnly(obj, key, value) {
  Object.defineProperty(obj, key, {
    value,
    writable: false,
    enumerable: true,      // still visible in keys()
    configurable: false    // can't be deleted or redefined
  });
  return obj;
}

const o = defineReadOnly({}, "id", 7);
o.id = 8;                  // ignored (sloppy) / TypeError (strict)
o.id;                      // 7
Object.keys(o);            // ["id"]
```
</details>

### Task 6 — Count word occurrences (the famous bug)

```js
function countWords(words) {
  const counts = {};
  for (let i = 0; i < words.length; i++) {
    const w = words[i];
    counts[w] = (counts[w] || 0) + 1;
  }
  return counts;
}

countWords(["a", "constructor"]);
```

What goes wrong? <details><summary>Answer + fixes</summary>

For the word `"constructor"`, `counts["constructor"]` is **not** `undefined`; it finds the **inherited** `Object` function. So `(function || 0) + 1` becomes a string: `"function Object() { [native code] }1"` ✅.

Fixes:

```js
// Fix A — an object with no inherited keys
const counts = Object.create(null);

// Fix B — check for an OWN property
counts[w] = (Object.hasOwn(counts, w) ? counts[w] : 0) + 1;
```
(A `Map` is the other standard fix.)

This is why "own vs inherited" is not trivia — plain objects used as dictionaries are vulnerable to keys like `constructor`, `toString`, `hasOwnProperty`.
</details>

---

## Output Questions

Predict first, then open the answer.

---

**Q1 — Shared nested references**
```js
const a = { n: { v: 1 } };
const b = a;
b.n.v = 2;
const c = { n: a.n };
c.n.v = 3;

console.log(a.n.v, a === b, a === c, a.n === c.n);
```
<details>
<summary>Answer</summary>

`3 true false true`.
`b` is the same object as `a`. `c` is a different outer object, but `c.n` points to the **same inner object** as `a.n`, so `c.n.v = 3` changes what `a.n.v` shows.
</details>

---

**Q2 — Property order**
```js
const o = { b: 1, 2: "x", a: 2, 1: "y" };
console.log(Object.keys(o));
```
<details>
<summary>Answer</summary>

`["1", "2", "b", "a"]`.
Integer-like keys first in ascending order, then other keys in creation order.
</details>

---

**Q3 — `delete` return values**
```js
const o = { a: 1 };
Object.defineProperty(o, "x", { value: 1 });

console.log(delete o.a, delete o.nope, delete o.x, "x" in o);
```
<details>
<summary>Answer</summary>

Sloppy: `true true false true`.
`delete o.nope` is `true` (nothing to remove). `delete o.x` is `false` because `x` is non-configurable. In strict mode that line throws `TypeError: Cannot delete property 'x' of #<Object>`.
</details>

---

**Q4 — Order after delete and update**
```js
const o = { a: 1, b: 2, c: 3 };
delete o.a;
o.a = 4;
console.log(Object.keys(o));
o.b = 9;
console.log(Object.keys(o));
```
<details>
<summary>Answer</summary>

`["b", "c", "a"]` both times.
Re-adding a deleted key places it last. Updating an existing key keeps its position.
</details>

---

**Q5 — Missing vs `undefined`**
```js
const o = { x: undefined };
console.log(o.x !== undefined, "x" in o, Object.hasOwn(o, "x"), Object.keys(o), o.nope);
```
<details>
<summary>Answer</summary>

`false true true ["x"] undefined`.
The key exists with the value `undefined`. Only `in` / `hasOwn` / `keys` can tell; `!== undefined` can't.
</details>

---

**Q6 — `in` vs `hasOwn`**
```js
const parent = { inherited: 1 };
const child = Object.create(parent);
child.own = 2;

console.log("own" in child, "inherited" in child, Object.hasOwn(child, "inherited"),
            "toString" in {}, Object.hasOwn({}, "toString"));
```
<details>
<summary>Answer</summary>

`true true false true false`.
`in` includes inherited properties (including built-ins like `toString`); `hasOwn` is own-only.
</details>

---

**Q7 — Shadowing**
```js
const parent = { inherited: 1 };
const child = Object.create(parent);

child.inherited = 9;
console.log(parent.inherited, child.inherited, Object.hasOwn(child, "inherited"));

delete child.inherited;
console.log(child.inherited, Object.hasOwn(child, "inherited"));
```
<details>
<summary>Answer</summary>

`1 9 true`, then `1 false`.
Assignment creates an own property on `child` and leaves `parent` alone. Deleting it removes the own copy, so the inherited value shows through again.
</details>

---

**Q8 — Broken `hasOwnProperty`**
```js
const d = Object.create(null);
d.k = 1;
try { d.hasOwnProperty("k"); } catch (e) { console.log(e.name); }

const o = { hasOwnProperty() { return true; } };
console.log(o.hasOwnProperty("zzz"), Object.hasOwn(o, "zzz"));
```
<details>
<summary>Answer</summary>

`TypeError`, then `true false`.
`d` has no inherited `hasOwnProperty`. `o` overrides it with a lying version. `Object.hasOwn` is immune to both.
</details>

---

**Q9 — Computed keys**
```js
const k = "dyn";
const o = { [k]: 1, ["a" + 1]: 2, [1 + 1]: 3, [k + "Fn"]() { return "fn"; } };
console.log(Object.keys(o), o.dynFn.name);
```
<details>
<summary>Answer</summary>

`["2", "dyn", "a1", "dynFn"] "dynFn"`.
`"2"` is integer-like so it comes first. The computed method gets the computed name.
</details>

---

**Q10 — Keys become strings**
```js
const o = {};
o[1] = "a";
console.log(o["1"], o[1] === o["1"]);

o[{}] = "obj";
console.log(Object.keys(o));

const u = { [undefined]: 1, [null]: 2, [true]: 3 };
console.log(Object.keys(u));
```
<details>
<summary>Answer</summary>

`"a" true`, then `["1", "[object Object]"]`, then `["undefined", "null", "true"]`.
Every key is converted to a string; any plain object becomes `"[object Object]"`.
</details>

---

**Q11 — Evaluation order of computed keys**
```js
let i = 0;
const o = { [i++]: i++, [i++]: i++ };
console.log(o, i);
```
<details>
<summary>Answer</summary>

`{ "0": 1, "2": 3 } 4`.
Left to right: key `0` (i→1), value `1` (i→2), key `2` (i→3), value `3` (i→4).
</details>

---

**Q12 — Non-enumerable property**
```js
const o = { a: 1 };
Object.defineProperty(o, "hidden", { value: 2, enumerable: false });

console.log(Object.keys(o), Object.getOwnPropertyNames(o),
            o.hidden, Object.hasOwn(o, "hidden"), "hidden" in o);
```
<details>
<summary>Answer</summary>

`["a"] ["a", "hidden"] 2 true true`.
Hidden from iteration helpers, but it exists, is readable, and is found by `getOwnPropertyNames`, `hasOwn`, and `in`.
</details>

---

**Q13 — `for...in` vs `Object.keys`**
```js
const parent = { inherited: 1 };
const child = Object.create(parent);
child.own = 2;

console.log(Object.keys(child));
const found = [];
for (const key in child) found.push(key);
console.log(found);
```
<details>
<summary>Answer</summary>

`["own"]`, then `["own", "inherited"]`.
`for...in` includes inherited enumerable keys; `Object.keys` doesn't.
</details>

---

**Q14 — `defineProperty` defaults**
```js
const o = {};
Object.defineProperty(o, "x", { value: 1 });
console.log(Object.getOwnPropertyDescriptor(o, "x"));
console.log(Object.getOwnPropertyDescriptor({ y: 1 }, "y"));
```
<details>
<summary>Answer</summary>

`{ value: 1, writable: false, enumerable: false, configurable: false }`
`{ value: 1, writable: true, enumerable: true, configurable: true }`
`defineProperty` defaults every omitted attribute to `false`; literal properties default to `true`.
</details>

---

**Q15 — Read-only write**
```js
const o = {};
Object.defineProperty(o, "x", { value: 1 });
o.x = 2;
console.log(o.x);
```
<details>
<summary>Answer</summary>

Sloppy: `1` (the write is silently ignored). Strict: `TypeError: Cannot assign to read only property 'x' of object '#<Object>'`.
</details>

---

**Q16 — Reconfiguration rules**
```js
const o = {};
Object.defineProperty(o, "x", { value: 1, writable: true });
o.x = 2;
Object.defineProperty(o, "x", { writable: false });

try { Object.defineProperty(o, "x", { writable: true }); } catch (e) { console.log("A", e.message); }
try { Object.defineProperty(o, "x", { value: 3 }); }       catch (e) { console.log("B", e.message); }
Object.defineProperty(o, "x", { value: 2 });
console.log("C", o.x);
```
<details>
<summary>Answer</summary>

`A Cannot redefine property: x`, `B Cannot redefine property: x`, `C 2`.
`writable` can go `true → false` once; it can't go back. With `writable: false` and `configurable: false` the value can't change — but re-specifying the **same** value is fine.
</details>

---

**Q17 — Getter-only property**
```js
const o = { get x() { return 1; } };
o.x = 5;
console.log(o.x);
```
<details>
<summary>Answer</summary>

Sloppy: `1`. Strict: `TypeError: Cannot set property x of #<Object> which has only a getter`.
</details>

---

**Q18 — `this` inside a getter**
```js
const parent = { first: "A", last: "B", get full() { return this.first + " " + this.last; } };
const child = Object.create(parent);
child.first = "X";
console.log(parent.full, child.full);
```
<details>
<summary>Answer</summary>

`"A B" "X B"`.
`this` in a getter is the object the property was read from (`child` for the second read).
</details>

---

**Q19 — Same three operations on four objects**
```js
// for each:  o.a = 2;  o.b = 3;  delete o.a;   (sloppy mode)
// objects:   { a: 1 },  preventExtensions({ a: 1 }),  seal({ a: 1 }),  freeze({ a: 1 })
```
What does each end up as?
<details>
<summary>Answer</summary>

| Object | Result |
|---|---|
| normal | `{ b: 3 }` |
| `preventExtensions` | `{}` (a changed, b refused, a deleted) |
| `seal` | `{ a: 2 }` (a changed, b refused, delete refused) |
| `freeze` | `{ a: 1 }` (all refused) |
</details>

---

**Q20 — Freeze is shallow**
```js
const o = Object.freeze({ a: 1, n: { b: 2 } });
o.n.b = 99;
o.a = 5;
console.log(o.a, o.n.b, Object.isFrozen(o), Object.isFrozen(o.n));
```
<details>
<summary>Answer</summary>

`1 99 true false`.
The outer object is frozen; the inner object is separate and unfrozen.
</details>

---

**Q21 — Is it frozen?**
```js
console.log(Object.isFrozen(Object.preventExtensions({})));
console.log(Object.isFrozen(Object.preventExtensions({ a: 1 })));
console.log(Object.isFrozen("abc"));
```
<details>
<summary>Answer</summary>

`true false true`.
An empty non-extensible object has nothing left that could change, so it counts as frozen. A non-empty one isn't frozen because its properties are still writable. Primitives are always "frozen".
</details>

---

**Q22 — Frozen array**
```js
const arr = Object.freeze([1, 2]);
arr[0] = 9;
console.log(arr[0]);
try { arr.push(3); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`1`, then `TypeError`.
The index assignment is silently ignored in sloppy mode, but `push` throws regardless of mode because it can't add the new element.
</details>

---

**Q23 — `defineProperty` on a frozen object**
```js
const o = Object.freeze({ a: 1 });
try { Object.defineProperty(o, "b", { value: 1 }); } catch (e) { console.log(e.message); }
```
<details>
<summary>Answer</summary>

`Cannot define property b, object is not extensible`.
`defineProperty` always throws; it doesn't follow the silent-failure behaviour of plain assignment in sloppy mode.
</details>

---

**Q24 — `Object.keys` on odd inputs**
```js
console.log(Object.keys("ab"), Object.keys(5));
try { Object.keys(null); } catch (e) { console.log(e.name); }
```
<details>
<summary>Answer</summary>

`["0", "1"] []`, then `TypeError`.
Strings are wrapped as objects with index keys; numbers have none; `null`/`undefined` can't be converted.
</details>

---

**Q25 — `in` on arrays**
```js
const arr = [10, 20];
console.log(1 in arr, 10 in arr, 2 in arr, "length" in arr);
```
<details>
<summary>Answer</summary>

`true false false true`.
`in` checks keys (indexes and `length`), never values.
</details>

---

**Q26 — Snapshot arrays**
```js
const o = { a: 1 };
const e = Object.entries(o);
o.a = 2;
console.log(e, Object.entries(o));
```
<details>
<summary>Answer</summary>

`[["a", 1]] [["a", 2]]`.
`Object.entries` returns a fresh array; later changes to the object don't affect it.
</details>

---

**Q27 — Frozen object with a setter**
```js
let store = 0;
const o = Object.freeze({ get v() { return store; }, set v(x) { store = x; } });
o.v = 5;
console.log(o.v, Object.isFrozen(o));
```
<details>
<summary>Answer</summary>

`5 true`.
`freeze` makes **data** properties read-only; accessor properties aren't "written", so the setter still runs and can change outside state.
</details>

---

## What Interviewers Test

**1. Mutation, reference and `const` on objects**
`const` doesn't freeze contents; nested objects are shared, not copied. Know how to say it with a memory picture.

**2. `in` vs `Object.hasOwn` vs `hasOwnProperty` vs `!== undefined`**
Own vs inherited, why `hasOwn` replaced `hasOwnProperty`, and why `undefined`-comparison isn't an existence check.

**3. `Object.keys` / `values` / `entries` vs `for...in` vs `getOwnPropertyNames`**
Be able to fill in the own / inherited / non-enumerable table from memory.

**4. Property order**
Integer-like keys ascending first, then insertion order; delete-then-add goes last.

**5. Computed property names and key coercion**
Keys become strings; objects collide on `"[object Object]"`; the plain-object word-count bug (`constructor`).

**6. Descriptors**
The four attributes, the different defaults for literal vs `defineProperty`, silent failure (sloppy) vs `TypeError` (strict), and the one-way `writable: true → false` rule.

**7. Accessors**
Getters/setters, `this` as receiver, getter-only = effectively read-only.

**8. `freeze` vs `seal` vs `preventExtensions` vs `const`**
What each allows, the internal definition (`seal` = non-extensible + non-configurable; `freeze` = seal + non-writable), shallowness, `deepFreeze`.

**9. How to make something genuinely read-only or hidden**
`defineProperty` with `writable: false`; `enumerable: false` hides from iteration but **doesn't** make it private.

---

## Mental Models

**An object = a labelled drawer cabinet.**
Each drawer has a string label (key) and holds a value. Labels are always text, even if you write a number.

**Own vs inherited = your drawers vs the shelf in the next room.**
`hasOwn` looks only in your cabinet. `in` also checks the next room it's linked to.

**Descriptor = the sticker on each drawer.**
Four stickers: *editable?* (writable), *listed?* (enumerable), *removable/re-labelable?* (configurable), and the contents (value).

**`enumerable: false` = "unlisted", not "locked".**
It's missing from the directory (`keys`, `for...in`), but if you know the label you can still open it.

**preventExtensions → seal → freeze = three padlocks of increasing strength.**
First: no new drawers. Second: also no removing drawers. Third: also no changing contents — but only of drawers *this* cabinet owns; a drawer holding a key to another cabinet doesn't lock that other cabinet.

**Plain object as a dictionary = a cabinet that already contains the building's default drawers.**
Ask for `constructor` and you get the building's drawer, not your own. Use `Object.create(null)`, `hasOwn`, or `Map`.

---

*Previous: Week 2 — Revision + Mock*
*Next: Destructuring — object + array destructuring, nested, in function params, default values*