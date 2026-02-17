---
layout: post
title:  "Status update #0"
author: Edoardo Marangoni
date: 2026-02-15
---

We have been working on [porting Rust to the CHERIoT platform](https://github.com/CHERIoT-Platform/cheri-rust) for about half a year now. 
We have been busy with the implementation of fundamental features such as defining the new target, adapting the compiler to respect the nuances of a CHERI platform -- starting with making the compiler aware that the size of an address is not necessarily also the size of the memory representation of a pointer -- and making the `core` and `alloc` libraries compile to the newly-introduced CHERIoT target. 

During this time we never found the chance to share some insights on the work. 
This post inaugurates a (hopefully) stable stream of weekly updates about the project. 

## News from the past 

In this particular occasion the update will be a bit long, because I want to use this chance to <s>exercise our attention span</s> recount important features from weekly status updates that should have been but were not, that is, all the work that we have done but never found the time to put in a blog post.

Let's start with some numbers. 
As of today we have 54 commits in the `beta` branch of the repository which, as the name suggests, builds on top of the `beta` branch from upstream Rust. 
Furthermore, `git diff compiler/` tells me that 84 files were touched, 805 lines were inserted and 280 were deleted. 
Now, before you get angry at me: I know that lines of code are a good metric only when you need to store your source code on a floppy disk, and those number could as well mean that we added 805 wrong lines and removed 280 perfectly good ones.

What I am trying to say here is that the fact that we were able to add a new target to `rustc` - one with the specific requirements of CHERI - _and_ to make it produce functioning (for what we have tested until now) code with few changes tells a lot about the engineering practices of the Rust compiler. 
I also think we owe much to the strict-provenance effort: I am sure that it made our job measurably easier.

So, I will pick and explain interesting features from those commits, so that you can get an idea of the work we have done: I hope you have your tea and crumpets ready.

#### First things first ([40bff08](https://github.com/CHERIoT-Platform/cheri-rust/commit/40bff08fe170cf29697da2b44a6ecdfd6dce73c8), [773c56d](https://github.com/CHERIoT-Platform/cheri-rust/commit/773c56d4dc5e58edc4a1e4e0491737643d8e45e2))
A compiler can't do a lot without a backend. 
The first patch updated the LLVM submodule to point to the [CHERIoT port of LLVM](https://github.com/CHERIoT-Platform/llvm-project). 
The notable bits: some functions from the C interface of LLVM have a different signature from those upstream. 
In particular, the functions to create calls to [`memcpy`](https://github.com/CHERIoT-Platform/cheri-rust/commit/773c56d4dc5e58edc4a1e4e0491737643d8e45e2#diff-b9d202534dd2844b76444d5ecca2536e97ff29913a69be05d4585c9f98bac797R1429-R1439) and [`memmove`](https://github.com/CHERIoT-Platform/cheri-rust/commit/773c56d4dc5e58edc4a1e4e0491737643d8e45e2#diff-b9d202534dd2844b76444d5ecca2536e97ff29913a69be05d4585c9f98bac797R1441-R1451) need to know whether the bits that are to be copied are capabilities and if the metadata contained therein must be [preserved](https://github.com/CHERIoT-Platform/cheri-rust/commit/773c56d4dc5e58edc4a1e4e0491737643d8e45e2#diff-b9d202534dd2844b76444d5ecca2536e97ff29913a69be05d4585c9f98bac797R1410-R1414). 
As of now we delegate this decision to LLVM itself. 
This is, of course, likely to change in the future. 
That's it!

#### Farewell address space zero ([f455d94](https://github.com/CHERIoT-Platform/cheri-rust/commit/f455d94f5896936ce990b51bdc9fc4e98e89f7d7))

An [Address space](https://en.wikipedia.org/wiki/Address_space) defines the possible values that can be used to reference an entity in memory. 
Different address spaces can have different properties. 
By default, if you don't specify otherwise, LLVM assumes addresses "live" or refer to address space 0. 
[By convention](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-877.pdf#page=31), when programs in LLVM IR need to refer to pointers whose values are actually capabilities - which also entails that pointers in that space also have different semantics - use the address space 200. 

CHERIoT, in particular, uses that address space only. 
This is because it is a _purecap_ architecture, meaning that it can only handle capabilities: other architectures like [Morello](https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/cheri-morello.html) can also understand and use (without the memory safety features) canonical "bare" addresses.

Long story short, [since not too long ago](https://github.com/rust-lang/rust/pull/143182) Rust did not have a way to specify that a target uses a default address space different from `addrspace(0)`. 
Now that it does, we had to make the compiler use it where needed. 
This specific patch is interesting because, as far as I can tell, it could be useful for other targets as well, such as `amdgpu`.

To achieve this, the upstreamed PR makes the compiler take into account the relevant bits of the [datalayout string](https://llvm.org/docs/LangRef.html#data-layout) that specify the kinds and properties of the address spaces valid on a specific target.

#### Compiler Wars: Episode MMMMXXVI - A new target ([cde4496](https://github.com/CHERIoT-Platform/cheri-rust/commit/cde4496fbecb2a2ff1261bbd4439e143839d6ddd))
This commit made us able to run `rustc --target=riscv32cheriot-unknown-cheriotrtos`. 
Not a lot to say here, because it is actually a fairly small change and we needed a bit more work before we could actually compile any code to CHERIoT, but it was the first step towards that goal. 

#### Play ":%s/pointer_width/pointer_offset/gr", Sam ([8b44ba5](https://github.com/CHERIoT-Platform/cheri-rust/commit/8b44ba5530c3714dd731b6c0ecec19dbcf97a1eb), [f46ab36](https://github.com/CHERIoT-Platform/cheri-rust/commit/f46ab36c3dfadc11bba1c956775471244d9b8e67), [0e1fc52](https://github.com/CHERIoT-Platform/cheri-rust/commit/0e1fc523d1b92a1cadd3cb2f8ee8e10308f28db0), [8778ccf](https://github.com/CHERIoT-Platform/cheri-rust/commit/8778ccf7fc1acea5ece7bdd29001dca7501a6c88))
If you aren't too familiar with CHERIoT, there are a lot of things [you might be interested to know about it](https://cheriot.org/). 
One that is relevant now is that on CHERIoT addresses are 32 bits, but capabilities are 64 bits: 4 bytes contain the proper address you need to refer to something in memory, while the remaining 4 bytes contain metadata which, roughly, tells you what you can do with that specific capability.

This is, of course, pretty different from many currently popular architectures that Rust supports. 
In general, the compiler often makes the assumption that the bits it takes to store a pointer are exactly the same as the bits of a memory address. 

We had to teach the compiler that the two concepts can be different; to do so we used a byproduct of the [same PR](https://github.com/rust-lang/rust/pull/143182/changes#diff-bd58d97c1958029df344a30d473950e0195516d1153cbcc6314ff01afb87d12a) that introduced the ability to specify different default address spaces for a target. 
In fact, the data layout string can also contain a value that specifies the [size of _indices_](https://llvm.org/docs/LangRef.html#data-layout:~:text=The%20fourth%20parameter,this%20address%20space.) that can be applied to addresses in a given address space. 
(I know, `pointer_offset` is not the best name.)

One example of the nature of these changes is [this](https://github.com/CHERIoT-Platform/cheri-rust/commit/8b44ba5530c3714dd731b6c0ecec19dbcf97a1eb#diff-8b601779832d29e42b24bb88ccc3a5762217d72f148e8070767ee787b271190cR58): 
```rust 
fn int_ty_max(&self, int_ty: IntTy) -> u128 {
        match int_ty {
            IntTy::Isize => self.tcx.data_layout.pointer_offset().signed_int_max()
            ...
```
That function used `.data_layout.pointer_size()` before. 
For CHERIoT, that would return `u64::MAX` instead of `u32::MAX`: this latter value is effectively what we want here. 
This change does not impact targets where the default address space is integral. 
While this is the change we needed and made most sense for this use-case, I expect that this set of changes will (rightfully) require thorough and in-depth discussions with the compiler and language teams.

#### Diane, 11:30 a.m., February 15th. Entering the town of Twin Peaks ([9c89bfe](https://github.com/CHERIoT-Platform/cheri-rust/commit/9c89bfe5980ff5c84143ee4e9a102a01aad673b7), [d99b5fa](https://github.com/CHERIoT-Platform/cheri-rust/commit/d99b5faecf8e30cfd2adff5bb1d531e80ba8c3e2))
I am skipping a few commits ahead, some of which are actually very interesting: [splitting](https://github.com/CHERIoT-Platform/cheri-rust/commit/73fc98edc4c1b2f774972262a53c21fa3f2d7dc7) the `size` method of the internal representation of scalars in two, one to refer to how many bits of data it can fit and one to refer to how many bits it takes to store the scalar in memory (for the same reason as before); [adding CHERI-specific intrinsics](https://github.com/CHERIoT-Platform/cheri-rust/commit/3948c11217061ccf0b2272efd137f4b7087b5fe5) and [using them in `core`](https://github.com/CHERIoT-Platform/cheri-rust/commit/0361525efddc7cde0dda2b37ddb5b43188eeb2be); [adapting](https://github.com/CHERIoT-Platform/cheri-rust/commit/e2e2001ff25fce45b222b75cd83185d5f723638a) the generation of discriminants to be aware of the distinctions between the size of a pointer and an address; [using non-transmuting casts in MIR passes](https://github.com/CHERIoT-Platform/cheri-rust/commit/413472ff499f883c0a128d01c564d872e1788565) (this one needs to be fixed to use non-exposing casts!) and much more. 
I can't go over all of them now -- I think this post is getting pretty long as it is -- but if you'd like to ask questions or learn more about the work, feel free to reach out to us in the [public CHERIoT chat on Signal]() or [raise an issue on the cheri-rust repository](https://github.com/CHERIoT-Platform/cheri-rust/issues/new?template=bug_report.md). 

Anyways, the commits that give the title to this subsection are those in which we added steps to build `core` and run the `codegen-llvm` tests for our new CHERIoT target to our CI. 
This marked the moment when we paused the efforts to add new features, and focused primarily on verifying that the code we generate for CHERIoT makes sense and matches what Rust thinks it should be, and investigating the bugs we found in the process.

## News from a more recent past (last week)
I will tell you, now, what we worked on this last week, and the unlawful imprisonment will be over shortly after.

#### An actually fun thing I hope we will have the chance to do again ([#111](https://github.com/CHERIoT-Platform/cheri-rust/pull/111))
We have been investigating issue [#108](https://github.com/CHERIoT-Platform/cheri-rust/issues/108) for a couple days. Consider this snippet: 
```rust
let x = 42;
assert_eq!(alloc::format!("{x:b}"), "101010");
```
When compiling and executing it on a CHERIoT simulator it worked perfectly fine. 
This snippet, on the other hand, did not: 
```rust
let x = 42;
assert_eq!(alloc::format!("{x:#b}"), "0b101010");
```
The only difference is the use of the "alternate" flag in the format (i.e. `{x:b}` vs `{x:#b}`, which in this case means to prefix the integer with `0b`), but the latter crashed with this message: 
```
Error handler: PermitExecuteViolation(0x11) error at 0x8000f3fa ... 
```
Looking at the dump of the firmware, we understood that the error was coming from the `f.buf.write_str(prefix)` call [here](https://github.com/CHERIoT-Platform/cheri-rust/blob/beta/library/core/src/fmt/mod.rs#L1884C44-L1884C67). 
Nothing obvious came up when looking at the Rust code or the LLVM IR, so we took a (very) long look at the assembly that was triggering the error:
```asm
;             if let Some(prefix) = prefix { f.buf.write_str(prefix) } else { Ok(()) }
80017f68: ce81         	beqz	a3, 0x80017f80 <<core::fmt::Formatter>::pad_integral::write_prefix+0x68>
80017f6a: 6d84         	ct.clc	s1, 0x18(a1)
80017f6c: fea7855b     	ct.cmove	a0, a5
80017f70: fea685db     	ct.cmove	a1, a3
80017f74: 863a         	mv	a2, a4
80017f76: 70a2         	ct.clc	ra, 0x28(sp)
80017f78: 7402         	ct.clc	s0, 0x20(sp)
80017f7a: 64e2         	ct.clc	s1, 0x18(sp)
80017f7c: 6145         	ct.cincoffset	sp, sp, 0x30
80017f7e: 8482         	ct.cjr	s1
80017f80: 4501         	li	a0, 0x0
```
Notice anything weird? No? Really? Sure? Try again. What about now? Nothing? Alright. 
So, let's go line by line, starting with `beqz a3, 0x80017f80`. 
The `prefix` argument is of type `Option<&str>`, and Rust niche-optimises the representation of this value: `null` (`0`) means `None`, otherwise we are in the `Some(prefix)` path. 
In this path execution continues to `ct.clc s1, 0x18(a1)`, which means: load the capability at offset `0x18` from the capability contained in `a1` into `s1`. This is the address of the `buf.write_str` function. 
Execution continues, then, loading the arguments it needs to pass to the `write_str` function. 
After that we can see the epilogue of a function before a tail call: it restores `ra`, `s0` _and `s1`_ and then jumps to the address of `buf.write_str`. 
But, wait, we just changed the value of `s1`! 

In short, what we discovered is that there was a bug in CHERIoT-LLVM where the virtual register rewriter assigned a callee-saved register to store the value of the address to jump to for a tail call (which was [promptly fixed](https://github.com/CHERIoT-Platform/llvm-project/pull/323), by the way). 
This means that it could occur that the register containing the address to jump to, where execution will continue, would be overwritten with another value, potentially making CHERIoT trap at runtime.
This was cool, right?!


#### I'm out of fun titles - generating the correct `e_flags` ([e058b31](https://github.com/CHERIoT-Platform/cheri-rust/commit/e058b31de255783a31373768eb8a3ff60472153b)) 

The `object` crate does not have definitions for CHERI- or CHERIoT-specific [e_flags](https://man7.org/linux/man-pages/man5/elf.5.html#:~:text=e_flags%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20This%20member%20holds%20processor%2Dspecific%20flags%20associated%20with%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20the%20file.%20%20Flag%20names%20take%20the%20form%20EF_%60machine_flag%27.%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20Currently%2C%20no%20flags%20have%20been%20defined.). 
We created a [new organisation](https://github.com/cheri-rust-patches) on GitHub where we host patched forks of the third-party crates. 
We updated our `rustc` to use the patched version of `object` and the correct `e_flags`.

#### Let there be atomics ([15e149e](https://github.com/CHERIoT-Platform/cheri-rust/commit/15e149e54708610de7854a41f604159f9c2fa5f0))
As we mentioned before, Rust knows that our pointers are 64 bits wide, although CHERIoT is a 32-bit platform. 
This means that to have `AtomicPtr` from `core`, we need to tell `rustc` that the platform supports atomic operations on values of the same size as a pointer. 
We have `AtomicPtr` now!

#### Having a runner is not too useful if I can't see why it fails ([d4b3b4d](https://github.com/CHERIoT-Platform/cheri-rust/commit/d4b3b4d1442ade2b142740ea9fbfccb421dffc9b))
We don't only run `codegen-llvm` tests (which actually do not execute code on a CHERIoT implementation, but just compare the generated IR), we also have a [small selection](https://github.com/CHERIoT-Platform/cheri-rust/tree/beta/cheri/tests) of tests we build and run on the [Sail simulator for CHERIoT](https://github.com/CHERIoT-Platform/cheriot-sail). 
We use a custom runner to execute them, and there was a [bug](https://github.com/CHERIoT-Platform/cheri-rust/issues/109) that prevented exceptions from being printed in specific cases. 
It is now [fixed](https://github.com/CHERIoT-Platform/cheri-rust/commit/d4b3b4d1442ade2b142740ea9fbfccb421dffc9b), yay!


## Conclusion

If you have made it this far, you definitely deserve a cookie, but I ate 'em all while writing this, sorry. 
I'll try to keep these updates coming on a fairly regular schedule, and the next ones will be shorter and more in-depth. 
For now, let me conclude saying that we are very happy with the current status of the project, and we know we have a lot more work to do. 
All the work happens in the public [cheri-rust](https://github.com/CHERIoT-Platform/cheri-rust) repository. 
If you want to try it out, here is a one-liner: 
```sh
git clone https://github.com/CHERIoT-Platform/cheri-rust.git &&\
    cd cheri-rust &&\
    ./cheri/gen_bootstrap.sh --build-clang &&\
    ./x build compiler std --target=riscv32cheriot-unknown-cheriotrtos
```
You can find [here](https://github.com/CHERIoT-Platform/cheri-rust#:~:text=toolchains/cheri/bin-,Now%20what%3F,-Keep%20in%20mind) instructions to do _something_ with the compiler after you have built it. 
Let us know how it went on the [public CHERIoT group on Signal](https://signal.group/#CjQKIElxAs3t3MUEMOEmQEuMHRK4rErUk2xVeFzjAjFXAShzEhCK9qQwEMFKGLGZnCjrQ7zm) or, if something went wrong, [file an issue](https://github.com/CHERIoT-Platform/cheri-rust#:~:text=went%20wrong%3F%20Please-,raise%20an%20issue,-or%20contact%20us) please! 

Bye now!
