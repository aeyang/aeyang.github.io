---
title: "Pointers in C++: Best Practices (Part 2)"
date: xxxx-xx-xx 12:00:00 -0700
categories: [c++]
tags:
---

## Problem
As the last post showed, raw pointers are powerful but their power leads to many different ways it could fail.

## std::unique_ptr
"I own this object, and I'm solely responsible for its lifetime"

## RAI
No explicit deletes

## Modern Alternatives to Raw Pointers
Prefer std::string_view over char*