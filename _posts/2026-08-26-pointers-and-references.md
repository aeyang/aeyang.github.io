---
title: Pointers and References in C++ (Part 1)
date: 2026-08-26 12:00:00 -0700
categories: [c++]
tags:
---

## Goal
The main goal of this post is to give us a working introduction to pointers and references in C++. This is not a deep nor comprehensive study. We are just trying to remove that slight sense of confusion upon encountering the **\*** or **&** operators and replace it, hopefully, with slight curiosity.

Lets make this more concrete. By the end, we should have an intuitive understanding of the code below:

```
struct Person {
    int age;
};

Person person1; // Example 1
person1.age++;

Person& person2; // Example 2
person2.age++;

Person* person3; // Example 3
person3->age++;
```

## Ordinary Object Creation
Lets think about what exactly is happening in Example 1. An object of type `Person` is created and given the name `person1`. Literally, some memory has been allocated on the stack for `person1`.
```
┌─────────────────────────┐
│                         │
│       Person object     │
│                         │
│       age = 35          │
│                         │
└─────────────────────────┘
            ▲
            │
          person1
```
Now, lets pass `person1` to a function:
```
void birthday(Person personParameter) {
    personParameter.age++;
}

birthday(person1);
```
By passing `person1` to the `birthday` function, a **new** `Person` object has been created with the name `personParameter`. So there are now two `Person` objects that exists in memory. The `birthday` function only increases `personParameter`'s age, and when `birthday()` finishes, `personParameter` is deleted from memory.
```
Original                    birthday()'s Copy

person1                     personParameter //deleted after birthday() finishes
┌──────────────┐            ┌──────────────┐
│ age = 35     │            │ age = 36     │
└──────────────┘            └──────────────┘
```
This phenomenon is called **Pass By Value**. C++ makes a copy of primitive and object types when passing parameters to functions with this syntax.

But what we actually wanted was not to increment the age for a copy of `person1`... We wanted to increment the age for `person1` itself! We need to somehow get `person1` itself into the `birthday` function. This is called **Pass By Reference**, which conveniently brings us to... References.

## References
Refer back to Example 2 at the beginning of this post. See how we change the parameter on the `birthday` function.
```
void birthday(Person& personParameter) {
    personParameter.age++;
}

birthday(person1);
```
The `&` attached to the `Person` type changes the type of `personParameter`. Instead of meaning "Create a new Person initialized from the argument", it now means: "Give me another name for the passed-in Person". You can visualize it as two names for the same object:
```
                      ┌──────────────┐
                      │              │
           person1 ──►│ age = 35     │
                      │              │
   personParameter ──►│              │
                      └──────────────┘
```
So when `birthday()` increments `personParameter`'s age, its actually incrementing `person1`'s age. This is what we wanted.

Now, lets introduce a **fundamental limitation of references** that is important to know. Lets say there are two `Person`s and I'm trying to choose the `Person` I want to be friends with:
```
Person alice;
Person bob;

Person& newFriend = alice; // I'm choosing alice first

// Oh wait, I changed my mind, I want my new friend to be bob

newFriend = bob; // This won't do what you think it will
```
The last line won't make `newFriend` point to bob because **a reference can't be reseated**. Once a reference is initialized, it is *permanently bound* to the object it was initialized with for its entire lifetime. Side note - A reference must be initialized the moment you declare it. So just this: `Person& newFriend;` doesn't compile.

The inability to switch what a reference "refers to" is not ideal. If we're familiar with Java, we know it doesn't take a second thought to reassign a `Person person` variable to bob, then alice, then back to bob. This is because in Java, a variable of class type is *always* just a handle/*pointer* to the object on the heap. Its never the object itself, unlike earlier in C++ where we created `Person person1`.

So in C++, how do we get a *pointer* that can point (and be reassigned) to different objects?

## Pointers
Lets just show the code first:
```
Person* newFriend = nullptr;
newFriend = &alice;
newFriend = &bob;
```
On the first line we declare and initialize a pointer. The **\*** after `Person` changes the meaning of the line to be: "newFriend is a pointer to a Person". No `Person` object was created in memory here. What was created is a Person pointer variable with size 8 bytes (on a 64 bit system) - big enough to hold any one memory address.
Also, notice what we get for free with pointers: The ability to represent a current absence of a chosen friend or "optionality".

On the 2nd and 3rd code lines, the **&** in `&alice` is called the **address-of operator**. It produces `alice`'s memory address. So `newFriend` holds the memory address of the `Person` object named `alice`... and then `bob`. The diagram below illustrates what this looks like in memory. Notice that the pointer itself lives in memory and simply stores the memory address of alice.

```
Memory
─────────────────────────────────────

0x1000    newFriend
          ┌─────────────────┐
          │ 0x5000          │
          └─────────────────┘

...

0x5000    Person (alice)
          ┌─────────────────┐
          │ age = 32        │
          │ ...             │
          └─────────────────┘
```

**Very Important:** We have now seen two completely different uses for the **&** symbol and its crucial to get them straight:
- In `Person& person1 = alice;`, the **&** is attached to the `Person` type and declares *what kind of variable* `person1` is (a `Person` reference type).
- In `newFriend = &alice;`, the **&** is part of the expression. It is a unary operator (like -x) which operates on an object and returns its address.

So now we see that a pointer can "point" to different objects by storing the address of different objects.

### Dereferencing
Lets say I've chosen my friend bob (`newFriend = &bob`). How do I read his `age`?
I can't just do `newFriend.age`; recall that `newFriend` is just an 8 byte address, it doesn't have an `age` member. The `Person` object at that 8 byte memory address has the `age` member.

```
// Getting a Person's age

Person newFriendObj = *newFriend; // Example 1
newFriendObj.age;

(*newFriend).age; // Example 2

newFriend->age // Example 3
```
In the first example above, we're showing a different usage of the **\*** symbol. In this case, **\***'s use is known as the **Dereferencing Operator** which returns the object that a pointer points to. We can save this object as we see in this example, but this has the disadvantage of creating a new `Person` object named `newFriendObj`

The second example again uses the dereference operator but prevents a new `Person` object from being created.

The third example is what we want to use most of the time. C++ provides a shorthand operator **->**. This is nice as it makes the indirection explicit in the syntax - you can read it as "follow this pointer to the actual object, then access its member".

## Conclusion
We now know the foundational concepts and syntax that was shown at the top of this blog (and also why that code wouldn't compile...)
We've also learned about using **&** and **\*** symbols to declare different variable types as well as using them as the **address-of** and **dereference** operators respectively.

There are more complex motivations for using these constructs which we will get to in future posts.
But we now have enough context to start to reasoning about what these constructs are doing when we see them in real C++ code.
