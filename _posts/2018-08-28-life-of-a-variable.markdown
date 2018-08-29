---
layout: post
title:  "Life of a debugged variable (part 1)"
date:   2018-08-28 18:16:09 -0500
---

In order to make debugging possible, the debugger has to be able to map
back facts starting from the final executable to the original source code.
These informations are emitted by the compiler.
Let's start with the simplest possible swift program as a pratical example.

{% highlight ruby %}
$ cat example.swift
func f() -> Int {
  let patatino = 23
  return patatino
}
f()
{% endhighlight %}

After the code is parsed and typechecked, it gets lowered into an high-level
target independent intermediate representation (SIL), which is the representation
on which the high-level swift optimizer likes to work on. As part of SIL generation,
the frontend emits some informations useful for the debugger.

{% highlight ruby %}
$ swiftc example.swift -g -emit-sil -o - -Xllvm -sil-print-debuginfo

[...]

sil_scope 2 { loc "example.swift":1:6 parent @$S4main1fSiyF : $@convention(thin) () -> Int }
sil_scope 3 { loc "example.swift":1:17 parent 2 }

sil hidden @$S4main1fSiyF : $@convention(thin) () -> Int {
bb0:
  %0 = integer_literal $Builtin.Int64, 23, loc "example.swift":2:18, scope 3 // user: %1
  %1 = struct $Int (%0 : $Builtin.Int64), loc "example.swift":2:18, scope 3 // users: %2, %3
  debug_value %1 : $Int, let, name "patatino", loc "example.swift":2:7, scope 3 // id: %2
  return %1 : $Int, loc "example.swift":3:3, scope 3 // id: %3
} // end sil function '$S4main1fSiyF'

[...]
{% endhighlight %}
