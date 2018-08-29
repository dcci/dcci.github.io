---
layout: post
title:  "Life of a debugged variable (part 1)"
date:   2018-08-28 18:16:09 -0500
---

In order to make debugging possible, the debugger has to be able to map
back facts starting from the final executable to the original source code.
These informations are emitted by the compiler.
Let's start with the simplest possible swift program as a pratical example.

{% highlight swift %}
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

{% highlight swift %}
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

Variables in the original source program usually gets translated into `debug_value` instructions.
A `debug_value` instruction has an operand which represents the value this variable is
associated with (in this case, it's an `Int`, which in Swift is represented using a `struct`).
It also contains the name of the variable to make sure the debugger can locate it by name.
As every other instruction in SIL, it has two bits of informations attached to it. 
The first one is the SILLocation, which is a triple <filename; line; column>. This allows to keep
track of source locations starting from instructions. For example, the 
instruction of the function is a `return` and its SILLocation is represented by the triple
`<example.swift, 3, 3>`, which is exactly where the return lives in the source code.

The second bit of information is the lexical scope. A lexical scope can be thought as a unique
identifier of the block where the instruction is defined. It's particularly important to associate 
each instruction with a scope to correctly implement binding and shadowing rules in the debugger.
In particular, the scope is the only way of discriminating which variable the user is
referring to by name when there are two variables with the same name in two nested scopes, e.g.

{% highlight swift %}
let a = 1
if (b) {
  let a = 2
  print(a)
}
print(a)
{% endhighlight %}

It should be noted that the informations carried at the SIL level are relatively lightweight,
mainly because SIL, as high-level intermediate, preserves a fair amount of informations of
the original program (including, e.g. the types).
Once SIL is generated, the optimizer runs several transformations on the input SIL, even at
`-Onone` (i.e. when optimizations are disabled). These, so-called "mandatory passes" are generally
analyses run to verify the program honours some properties (e.g. to make sure the sum of two 32-bit
integer doesn't overflow or that variables are initialized before they're used), but also makes
some transformations on the code. It's responsability of the passes to make sure debug informations
are preserved when applying transformations. 

Once the SIL optimizer is done with his job, the code gets lowered into a lower level representation,
LLVM IR. Here lots of informations about the original program aren't really conveyed, so the compiler
has to do a little bit more work to preserve facts about the original program in the debug informations.
Among others, the original types are lost, as LLVM IR uses a different type system. The way this information
is expressed is through the use of metadata, that is, semantic information associated to each instruction
in the stream. 

{% highlight swift %}
$ swiftc example.swift -g -emit-ir -o - -Xllvm -sil-print-debuginfo

[...]

define hidden swiftcc i64 @"$S4main1fSiyF"() #0 !dbg !35 {
entry:
  call void @llvm.dbg.value(metadata i64 23, metadata !39, metadata !DIExpression()), !dbg !41
  ret i64 23, !dbg !42
}

!35 = distinct !DISubprogram(name: "f", linkageName: "$S4main1fSiyF", scope: !6, file: !5, line: 1, type: !36, 
                             isLocal: false, isDefinition: true, scopeLine: 1, isOptimized: false, unit: !0, retainedNodes: !2)
!36 = !DISubroutineType(types: !37)
!37 = !{!38}
!38 = !DICompositeType(tag: DW_TAG_structure_type, name: "Int", scope: !8, file: !32, size: 64, 
                       elements: !2, runtimeLang: DW_LANG_Swift, identifier: "$SSiD")
!39 = !DILocalVariable(name: "patatino", scope: !40, file: !5, line: 2, type: !38)
!40 = distinct !DILexicalBlock(scope: !35, file: !5, line: 1, column: 17)
!41 = !DILocation(line: 2, column: 7, scope: !40)
!42 = !DILocation(line: 3, column: 3, scope: !40)
[...]
{% endhighlight %}

As can be noted in the example above, the informations are much more lower-level and hierarchical than
the ones expressed in SIL, and as we'll see, they very much resemble the final product (i.e. what the debugger
is going to consume). For example, the function has a metadata associated containing a mangled name, which was
absent in the SIL representation, and the type of the variable is expressed through a different type, and the
metadata has to specify where this type is coming from (i.e. that's a swift-type). This is a design-choice,
and unavoiable, as LLVM is the target for many other languages other than swift (most notoriously, C++ or Rust).

The last step is that of going through the backend. Here the code gets lowered in a form that's understandable
by a machine. This is where the "real" debug informations consumed by a debugger get emitted, together with
the machine code. (Almost) everybody agrees on the representation used, which is described in a format named
DWARF. DWARF informations can be either emitted as part of the main executable (here the usual representation
is that they're sections which name carries the ".debug" prefix), or can be in a separate directory. The
choice made is fundamentally system dependent, and there are several tradeoffs involved. By default, on Mac OS,
swift emits debug information on the side, in a container named dSYM (which stands for debug symbols).

{% highlight swift %}
$ swiftc example.swift -g
$ dwarfdump example.dSYM

[...]
0x0000004d:         TAG_subprogram [4] *
                     AT_low_pc( 0x0000000100000fa0 )
                     AT_high_pc( 0x0000000b )
                     AT_frame_base( rbp )
                     AT_linkage_name( "$S4main1fSiyF" )
                     AT_name( "f" )
                     AT_decl_file( "/Users/dcci/Desktop/preso/example.swift" )
                     AT_decl_line( 1 )
                     AT_type( {0x0000009f} ( Int ) )
                     AT_external( true )

0x0000006a:             TAG_lexical_block [5] *
                         AT_low_pc( 0x0000000100000fa9 )
                         AT_high_pc( 0x00000002 )

0x00000077:                 TAG_variable [6]
                             AT_const_value( 0x00000017 )
                             AT_name( "patatino" )
                             AT_decl_file( "/Users/dcci/Desktop/preso/example.swift" )
                             AT_decl_line( 2 )
                             AT_type( {0x0000009f} ( Int ) )

[...]
{% endhighlight %}
