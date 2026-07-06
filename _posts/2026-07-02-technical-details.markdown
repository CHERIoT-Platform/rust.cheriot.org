---
layout: post
title: "Technical Details: Const Evaluation and Data Layout"
author: Sarah Harris
date: 2026-07-06
---

TODO: spell check and proof read.

One of the more interesting places where the quirks of CHERI surface is in Rust's const evaluation mechanism.
This feature allows parts of programs to be run during its compilation, and the results stored for use when the program is actually run.
People familiar with CHERI Rust work may remember this coming up in previous discussions of changes needed to the Rust compiler during porting efforts.
The set of operations that are supported is limited.
Obviously trying to perform IO operations during compilation wouldn't end well, and calculations that might never finish probably aren't a good choice either.
The operations available do, however, include some forms of pointer arithmetic.

This snippet for example:
```rust
const POINTER: *const u8 = "test".as_ptr().wrapping_add(3);
fn main() {
	assert_eq!(unsafe { *POINTER }, b's');
}
```

## The Inner Light of Const Evaluation

For this to work, the Rust compiler has to be able to represent pointers during compilation.
But wait, what?
A pointer, on most architectures, is a number referring to a location in memory.
During compilation we know what architecture we're building for, but we don't know what the machine will be, and we certainly don't know what will be in it's memory when the program runs!

The solution to this has two parts:
* how data is stored during const evaluation
* image activation and LLVM magic

I don't want to get too deep into the second of those, partly because honestly I doubt I fully understand the subtleties, and more importantly, because it isn't super relevant to the interesting CHERI Rust details I want to go over.

The end product of const evaluation are arrays of bytes to represent most data combined with a abstract representation of pointers in terms of what they point to.
Both of these are handed over to LLVM, which then handles converting them into whatever format the target system wants.
For pointers, this will be some form of placeholder which encodes where the pointer should point.
When the program is started, either during an image activation process on a conventional operating system, or during the hardware setup process in an embedded environment, these placeholder descriptions will be replaced with a real address calculated from the encoded information, now that the system knows where everything is for real.

It ends up looking like this pair of input program and the resulting intermediate language the Rust compiler gives LLVM:

```rust
struct Example0 { array: [u8; 8], pointer: *const u32 }
const INTEGER: u32 = 42;
const EXAMPLE0: Example0 = Example0 {
	array: [0, 1, 2, 3, 4, 5, 6, 7],
	pointer: &INTEGER,
};
```

```
; (Allocation names rewritten for clarity)
@alloc_integer = private unnamed_addr constant [4 x i8] c"*\00\00\00", align 4
@alloc_example0 = private unnamed_addr constant <{ [8 x i8], ptr }> <{ [8 x i8] c"\00\01\02\03\04\05\06\07", ptr @alloc_integer }>, align 8
```

# A Human Reaction to Data Layout

That leaves the problem of how to represent pointers and every other value inside the compiler during const evaluation.
Values that don't include pointers are fairly simple: most are stored as arrays of bytes called “allocations” (implemented as `Allocation` in `compiler/rustc_middle/src/mir/interpret/allocation.rs`).
TODO: hyperlink this?
Because the compiler knows what architecture it's building for, the internal layout of these can match the run time layout exactly, and save needing to do any conversions.
For the sake of performance, single individual values are handled by just passing them around as integer values, which saves needing to manage lot of small allocations.

```rust
struct Example1 {
	field_0: u64,
	field_1: u32,
	field_2: u32,
}
```

| bytes 0-7 | bytes 8-11 | bytes 12-16 |
| --------- | ---------- | ----------- |
| field_0   | field_1    | field_2     |

Rust allows values to be uninitialised, and the layout of some data structures includes padding with nothing in it, which is also left uninitialised.
Allowing a program to read gibberish from, say, the padding between fields in a `struct` would be unfortunate, and lead to a few kinds of undesirable behaviour, so it's useful to be able to detect this.
To solve this, const evaluation adds a mask to each array of bytes to indicate which bytes are initialised.

```rust
#[repr(C)] // don't remove padding and break my example!
struct Example2 {
	field_0: u16,
	field_1: u32,
	field_2: Option<(u32, u32)>,
}
```

| bytes 0-1 | bytes 2-3 | bytes 4-7 | bytes 8-15  | bytes 16-23             |
| --------- | --------- | --------- | ----------- | ----------------------- |
|  field_0  | uninit    | field_1   | field_2.tag | field_2.value or uninit |

# Won't Get Fooled Again by Relocations

Pointers are trickier.
The compiler needs enough information that it can do meaningful calculations with them, but it needs to not rely too much on concrete addresses in memory which won't be known until run time.
Currently, this problem is solved by adding a table of “provenances”, which maps from ranges of bytes in an allocation to references to areas in other allocations.
TODO: double check understanding of how this works.
TODO: decide whether to talk about wildcard provenance.
Each pointer then has an entry in this map.

TODO: diagram of struct including pointers.

```rust
struct Example3 {
	field_0: u32,
	field_1: *const u32,
	field_2: u32,
}
```

# Capabilities: The Best of Both Worlds, Part 1

This is all well and good for existing targets, where a pointer is just a number used to store an address, but CHERI complicates things.
Now pointers are capabilities, which contain twice as many bits, and only half of those are the actual address.
It is also no longer possible to just create a ~~pointer~~ capability out of nowhere, it has to come from some parent capability.

The second of these problems is easy enough to solve, as pointers already have to be recalculated when a program starts.
On CHERI systems, these calculations also handle deriving each capability from some parent capability with access to all of the address space a program is allowed to use.
While that isn't something a running program will have access to, the startup process is handled by more privilleged code in the operating system or hardware initialisation logic, which does.
Just like the normal relocation mechanism, most of this “just happens” from the Rust compiler's point of view.
Most of the heavy lifting happens inside LLVM and the operating system.

The matter of the extra data in each capability is where things get more complicated for Rust, and const evaluation specifically.
Where on other architectures we just need to keep track of the provenances that describe an eventual address, now we need to decide what to do about all that extra information in the extra 32 bits of the capability.

TODO: diagram, integral pointer vs capability

# The Best of Both Worlds, Part 2

So what's in those extra bits?
Several things: compressed upper and lower bounds for addresses the pointer can access, permission information for read, write, excute, and more, and an object type field used to control how different pieces of code can use the capability.
This leaves us with a few options.

1. emulate CHERI semantics during const evaluation so we know the entire contents of each capability
2. zero the extra bits, and let them be filled in during program startup
3. mark the extra bits as uninitialised, and let them be filled in during startup

While solution 1 sounds like the ideal approach, accurately implementing CHERI semantics would be a non-trivial project, add a lot of extra complexity to the const evaluation machinery for (currently) only a single architecture, and may or may be useful in practice.
Given that LLVM can already handle calculating the bounds for constant pointers, we don't actually *need* this extra information for things to work correctly.
That makes solutions 2 and 3 quite attractive at this stage in the project.

What about solution 2?
It's simple to implement and unlikely to crash the compiler, which is nice.
But what happens if a program tries to, say, cast a capability to an array of bytes, and read the raw bounds information?
(I have no idea why you'd do this, but I wouldn't be surprised if *someone* has a reason to!)
With solution 2, the result would be zeroes, which would be inconsistent with the results of the same operation at run time, breaking the programmer's expectations.

This how we end up at solution 3.
In solution 3, everything in a capability that isn't the address is marked uninitialised during const evaluation, the same way as explicitly uninitialised values and padding.
The const evaluation machinery in the compiler denies access to uninitialised bytes to prevent unexpected behaviour in these other cases, conveniently making it impossible to read from the empty metadata part of capabilities and get confusing behaviour.

This approach worked well for the Kent Morello Rust port, and currently makes the most sense for CHERIoT.
Perhaps sometime in the future someone will do the research neccessary to explore whether there are advantages to fully emulating CHERI during const evaluation.
