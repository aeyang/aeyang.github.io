---
title: "RAII in C++ (Part 3)"
date: 2026-08-29 12:00:00 -0700
categories: [c++]
tags: [pointer]
---

## Goal
In pointers part 2, we ended with an uncomfortable realization: pointers solve real problems, but using them to reach objects on the heap opens a can of worms. Someone has to take responsibility for cleaning up those objects when they're no longer needed... all without shooting ourselves in the foot while trying to do it.<br/>

In this post we will explore modern C++'s answer to this problem.

## Destructors

How destructors work, use old makeBobs example.
destructors run automatically at scope exit!!
What if we put a delete inside a destructor? Write our own owning wrapper `PersonOwner`.

## RAII
What we built above is the RAII pattern. "Resource release is destruction".

### Pushback
One pushback to make sure we understand:
Question: It works on deleting the resource when its destructor runs. But for resources on
the heap, it lasts the entire length of the program... so its destructor doesn't run right? Not unless the programmer remembers to call it, which puts us in the exact same predicament?

Answer: The pointer RAII is actually on the stack, the object is on the heap. When the stack frame is popped, the pointer RAII's destructor will run, freeing the heap object.

```
        STACK                          HEAP
   ┌──────────────┐              ┌──────────────┐
   │  owner       │              │   (Person)   │
   │ (unique_ptr) │─────────────▶│   age: 40    │
   │  ~unique_ptr │              └──────────────┘
   │  runs at }   │                     ▲
   └──────┬───────┘                     │
          │  at scope exit, owner's destructor │
          └───────────── calls delete ─────────┘
```
If the pointer RAII is also on the heap, then yes you are right, unique_ptr will not run automatically, and you'll have to manually `delete` it which defeats the whole purpose.
```
auto* owner = new std::unique_ptr<Person>(std::make_unique<Person>());
//    ^^^^ the unique_ptr is now ON THE HEAP
```

### Payoff of RAII
Show example from https://www.youtube.com/watch?v=Rfu06XAhx90 where RAII can replace all those delete calls.

## C++ std RAII Objects

### std::unique_ptr
"I own this object, and I'm solely responsible for its lifetime"

## RAII is Bigger Than Memory
std::fstream (RAII for files) - don't forget to close the file
std::lock_guard (RAII for locks) - don't forget to unlock a lock
CUDA device memory (a wrapper whose destructor calls cudaFree. RAII for CUDA device memory)

## Conclusion
With RAII, ownership becomes a type, not a convention. The compiler now enforces what used to be programmer discipline.

## More Resources on RAII
https://www.youtube.com/watch?v=Rfu06XAhx90