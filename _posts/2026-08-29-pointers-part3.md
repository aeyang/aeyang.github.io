---
title: "RAII in C++ (Part 3)"
date: 2026-08-29 12:00:00 -0700
categories: [c++]
tags: [pointer]
---

## Goal
In pointers part 2, we ended with an uncomfortable realization: pointers solve real problems, but using them to reach objects on the heap opens a can of worms. Someone has to take responsibility for cleaning up those objects when they're no longer needed... all without shooting ourselves in the foot while trying to do it.<br/>

In this post we will explore modern C++'s answer to this problem.

## Constructors & Destructors
Recall our last example from part 2 where we create `Person` objects on the heap and hopefully remembered to delete those heap objects before the function ends, otherwise it would lead to a memory leak:
```
void aliceMakesFriends() {
    Person* alice = new Person;
    Person* bob   = new Person;
    alice->bestFriend = bob;

    ...
    // We forgot to call deletes
}
```
Well, theres something more happening here behind the scenes that we didn't talk about yet. When we declare a `struct Person { int age; };`, C++ automatically generates *special member functions* called a `constructor` and a `destructor`.

### Constructor
When we write `new Person`, the constructor runs automatically. Its job is to create the `Person` object and it will do 2 things:
1. Initializes the members. In this case default initialization.
2. Runs any other code in the body of the constructor. The programmer can write their own constructor.

### Destructor
At the end of `aliceMakesFriends`, when the `bob` pointer is being deleted since its a local variable on the stack, the destructor runs automatically to dismantle it. It does 2 things:
1. Runs the body of the destructor. **The programmer can write their own destructor.**
2. Reclaims the 8 bytes of stack space that held the address

Destructors for the `alice` and `bob` pointers ran because those live on the stack. Destructors for objects that live on the heap *only run* when you call `delete` on it. In this case we forgot, so no `Person` destructors ran.

So a fuller picture of whats happening looks like this:
```
Person* bob = new Person;
    │
    │ constructor runs (heap object born)
    ▼
┌──────────────┐
│     Bob      │  ← lives on the HEAP
└──────────────┘
    │        ▲
    │        └───── bob (stack pointer) holds its address
    │
    │ scope ends
    ▼
┌──────────────────────────────┐
│ stack pointer `bob` destroyed │  ← trivial: just frees 8 bytes,
│ (the address is forgotten)    │     does NOT touch the heap object
└──────────────────────────────┘
    │
    ▼
 ✗ destructor does NOT run
    │           (no delete was called; a raw pointer's
    │            destruction never frees its target)
    ▼
┌──────────────┐
│     Bob      │  ← STILL ALIVE on the heap, now unreachable
└──────────────┘
    │
    ▼
 💥 MEMORY LEAK
 (object never destroyed, never freed)
```

Its really cool of the compiler to **automatically run the destructor** when a local variable's scope has ended. That works nice for the `alice` and `bob` pointers in this example, but does nothing for the alice and bob `Person` heap objects.

**What if we could have the destructor that the compiler automatically calls also handle calling delete for the heap objects?**

## Write an "Ownership Wrapper" for Person
<div style="display: flex; flex-direction: column; gap: 5px; align-items: flex-start;" class="responsive-columns">
  <style>
    @media (min-width: 768px) {
      .responsive-columns { flex-direction: row !important; }
    }
  </style>

  <!-- Left Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    class PersonOwner {
        private:
            Person* person;
        public:
            // Writing our own constructor
            PersonOwner() {
                person = new Person;
            }
            // Writing our own destructor
            ~PersonOwner() {
                delete person;
            }
            void setBestFriend(PersonOwner& friendOwner) {
                // Note: C++'s private access is class-level, not object-level
                person->bestFriend = friendOwner.person;
            }
        };
  </div>

  <!-- Right Column -->
  <div style="flex: 1; width: 100%;" markdown="1">

    void aliceMakesFriends() {

        PersonOwner alice;
        PersonOwner bob;
        alice.setBestFriend(bob);

        ...
        // We don't need to call delete
    }

  </div>
</div>

On the left, we've written this class that takes ownership of a Person object on the heap. Something fundamentally different is now happening when `aliceMakesFriends` ends. **Since both `alice` and `bob` are both local variables on the stack, the compiler calls their destructors automatically.** When those destructors are run, they `delete person` for us!

This `PersonOwner` object frees us from the responsibility of remembering to call `delete`. And it ties the destruction of a *resource* (the `Person` on the heap), to the lifetime of an object (the `PersonOwner`).

This is the core concept behind **RAII** - *Resource Acquisition Is Intialization*


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