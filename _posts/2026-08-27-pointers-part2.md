---
title: "Pointers in C++: Objects, Lifetimes, and Ownership (Part 2)"
date: 2026-08-27 12:00:00 -0700
categories: [c++]
tags: [pointer]
---

## Goal
In Part 1 we learned what **pointers** are. Now we'll look at some problems they can solve and why that power comes with responsibilities every C++ programmer has to reckon with.

## Modeling Relationships
One of the most intuitive reasons (but far from the only reason) for why pointers exist is modeling relationships between objects. For example, lets give `Person` objects the ability to have 1 best friend. And lets make `Alice`'s best friend `Bob`. Naively, we could try the below:

<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- Left Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    struct Person {
        int age;
        Person bestFriend; // ❌
    };
  </div>

  <!-- Right Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    Person
    └── bestFriend: Person
        └── bestFriend: Person
            └── bestFriend: Person
                └── ...
  </div>
</div>

**By Value**\
This doesn't work... A `Person` can't contain a `Person` best friend by value. As the right side shows, this leads to a situation of infinite size.

**By Reference**\
A reference does fix the size problem since under the hood, its implemented as an address (a small fixed size). However, we run into 2 problems if we have `friend` be a reference type. These should be familiar:
1. *A reference must be initialized at declaration*. So, we MUST give `Alice` a bestFriend from the beginning of her existence. This also means `Alice` cannot ever NOT have a best friend (null). And though we might wish otherwise, this isn't an accurate model of reality.
2. *A reference cannot be reseated*. `Alice` could never change her best friend to someone else.

**By Pointer**\
A pointer can perfectly model this relationship:
<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- Left Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    struct Person {
        int age;
        Person* bestFriend;
    };

    // Alice starts her life with no best friend
    Person alice{30, nullptr};
    Person bob{40, nullptr};
    Person carol{25, nullptr};

    // Alice befriends Bob
    alice.bestFriend = &bob;
    alice.bestFriend->age; // 40

    // Alice switches best friends to Carol
    alice.bestFriend = &carol;
    alice.bestFriend->age; // 25
  </div>

  <!-- Right Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    ┌───────────────┐
    │     Alice     │
    │               │
    │ age 30        │
    │ friend─┐      │
    └────────┼──────┘
             │
    ┌────────▼───────┐  ┌────────▼───────┐
    │     Carol      │  │      Bob       │
    │                │  │                │
    │ age 25         │  │ age 40         │
    │ friend         │  │ friend         │
    └────────────────┘  └────────────────┘

  </div>
</div>

### Indirection - The Superpower of Pointers
**Indirection**, or the ability to access something through an intermediate thing rather than accessing it directly, summarizes pointers' superpower. This is useful for way more than just modeling relationships between objects, but we're focusing on that for this post.

Below are more examples where indirection allows objects to form more complex structures that aren't possible by simply embedding objects inside each other by value:
<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- First Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    Linked Lists

    A → B → C → nullptr
  </div>

  <!-- Second Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

        Trees

          A
         / \
        B   C
       / \
      D   E
  </div>

  <!-- Third Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    Graphs

    A ─── B
    │   ╱ │
    │ ╱   │
    C ─── D

  </div>
</div>

## A Pointer Does Not Keep its Target Alive
Lets make a generator function for `Bobs` in case there are more `Alices` out there that need friends:
```
Person* makeBobs() {
    Person bob;        // bob lives on the stack, only inside this function
    bob.age = 40;
    return &bob;       // hand out bob's address via the address-of operator
}                      // ...but bob is destroyed HERE, at the closing brace

Person* newBob = makeBobs();
p->age;                // 💥 dangling: p points at a bob that no longer exists
```
### Dangling Pointers
In the code above, the pointer `newBob` still holds a real address but the Person object at that address is technically **destroyed**. Accessing the dangling pointer is *undefined* behavior, meaning the C++ standard makes no guarantees on what happens. Possibilities include program crashing, delayed crash, corrupted memory, the right value is returned, etc.../
This is the first time we've talked about objects "dying" (not a technical term), so some confusion is expected. But we will get into this in the next section.

To recap quickly: *A pointer with an address to an object has no say on what happens to that object*.
Note: Contrast this with what Java does. The GC keeps objects "alive" as long as *any* reference to the object exists.

## How Long Does an Object Live?
How do we make `bob` outlive the `makeBobs` function that created him to avoid the dangling pointer problem?
We need to get into object lifetimes, and for that we need to take a short detour into:

### Stack and Heap
An initial mental model:
When a C++ program starts, the OS gives it a chunk of RAM memory (this is an oversimplification and saves the discussion of virtual address space and physical RAM for another day). This chunk is divided into two main regions: the STACK and the HEAP. See the diagram below:

```
     Program's Memory (RAM)
   ┌───────────────────────────┐  high addresses
   │          STACK            │
   │  (local variables,        │
   │   function frames)        │
   │                           │
   │   grows downward ↓        │
   ├─────────────┬─────────────┤
   │             ▼             │
   │                           │
   │      (free space)         │
   │                           │
   │             ▲             │
   ├─────────────┴─────────────┤
   │   grows upward ↑          │
   │                           │
   │           HEAP            │
   │  (objects from `new`)     │
   ├───────────────────────────┤
   │   static / globals        │
   ├───────────────────────────┤
   │   code (your compiled     │
   │        instructions)      │
   └───────────────────────────┘  low addresses
```

Both the stack and the heap are just regions of your program's memory (RAM), but they're managed by completely different mechanisms.

**The stack** is a region the compiler manages for you, following the structure of your function calls. Every time a function is called, it gets a stack frame, a contiguous block holding that function's local variables. When the function returns, its entire frame is popped off, and every local in it is destroyed automatically... including `bob`.

**The heap** (also called the free store in C++) is a large pool of memory you request from explicitly, and objects there live until **you** explicitly release them, completely independent of any function scope.

```
Person* alice = new Person;   // a Person is allocated on the heap
// ... alice lives on the heap, across function calls, until ...
delete alice;                 // ...YOU explicitly destroy it
```

Notice the new syntax. `new` creates objects on the heap (returns a pointer). `delete` destroys it (takes a pointer).
Now we can fix the problem from before. We simply create `bob` with `new`, so it lives past the end of `makeBobs`
```
Person* makeBobs() {
    Person* bob = new Person; // bob lives on the heap now
    bob->age = 40;
    return bob;
}

Person* newBob = makeBobs();
p->age;                // 40. Works!!
```

Observe what occurred in memory when you ran `Person* newBob = makeBobs();`
```

        STACK                          HEAP
   ┌──────────────┐              ┌──────────────┐
   │  newBob      │              │   (Person)   │
   │  ┌────────┐  │              │  ┌────────┐  │
   │  │ 0x1A00 │──┼─────────────▶│  │ age:40 │  │  ← the actual Person
   │  └────────┘  │              │  └────────┘  │     lives at 0x1A00
   │   (a pointer,│              │              │
   │    8 bytes)  │              └──────────────┘
   └──────────────┘
     the handle                   the object

```
Note that the `newBob` pointer lives on the stack, but the address it holds is an address in heap.

## Who is Responsible for an Object's Lifecycle?
Hooray, we made `newBob` live! He's going to live a long and fulfilling life!... Well, until `alice` wants a new best friend. When that happens, we would just call `delete newBob` to make sure that Heap memory got freed up for other objects. After all, we don't want the whole program to crash from running out of heap memory.

But what if we forgot? Maybe we created hundreds of `Persons` and forgot to `delete` a few. And then, the function in which `alice` was befriending different people finally reached its end. Not only did `newBob` (and others) not get `deleted`, but now the `newBob` pointer doesn't exist anymore because the stack frame was popped. But the `Person` object it pointed to still exists on the heap but now there is no way of reaching it anymore! This is known as a **Memory Leak**... and it was all **our** fault. The compiler can't help us here. We were the one to create all those objects on the heap, so the **responsibility** for freeing up that heap memory falls on us.

One could argue that we should just "be better". But before we conclude anything, lets show some other ways we can trip over ourselves trying to manage object lifecycles:

```
Person* alice = new Person;
Person* bob   = new Person;
alice->bestFriend = bob;     // now TWO pointers reach bob: `bob` and `alice->bestFriend`

delete bob;              // pointer #1 frees it...
alice->bestFriend->age;      // 💥 USE-AFTER-FREE: alice->bestFriend now dangles. Undefined behavior
delete alice->bestFriend;    // 💥 DOUBLE-FREE if someone "cleans up" again. Undefined behavior
// or: nobody deletes → 💥 MEMORY LEAK
```

To summarize: For objects created with `new`, C++ doesn't handle object lifecycle for you. And the raw pointer that gets returned says nothing about who's responsible for the object it points to. The example above showed multiple pointers to one object but no overarching rule about which pointer holds the responsibility to free it. This leads to leaks, double-frees, or use-after-free gotchas.
The AMBIGUITY is the problem.

## Conclusion
We have covered how pointers make modeling relationships between objects possible and also explored the fundamental pitfalls when working with pointers. These pitfalls point to fundamental questions of object lifetimes and ownership; the compiler has no answer for these.\
In the next post, we will cover modern C++'s answer to these common pitfalls.
