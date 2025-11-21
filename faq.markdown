---
layout: default
title: FAQ
---

# Frequently asked questions

Here are a few questions that people often ask.

## Doesn't Rust do everything that CHERI does?

No.
Rust provides a strong set of properties that are either enforced at compile time or lowered to dynamic checks during compilation.
These properties go beyond the guarantees that CHERI systems.
For example, CHERI can prevent you from confusing pointers and integers, but does not prevent you from confusing two pointers to structures of different types that are the same size, whereas Rust does.
Rust's `Send` / `Sync` mechanism makes it possible to enforce mutable-XOR-execute policies for safe concurrency, which goes beyond anything enforced at the hardware level in any CHERI system.

These guarantees all depend on the correctness of both the Rust compiler and of all code in `unsafe` blocks.
The Rust compiler has several open issues marked 'soundness', including the one used in the [cve-rs proof of concept](https://github.com/Speykious/cve-rs/tree/main).
If you are writing Rust code, you are unlikely to encounter these bugs, but a malicious developer trying to perform a supply-chain attack on your project could take advantage of them to violate the guarantees that the Rust language provides to non-malicious Rust code.

In contrast, CHERI and the CHERIoT compartmentalisation model are designed so that code in one compartment does not need to trust the compiler used for any code in other compartments.
If you are calling unsafe Rust, Rust code that you haven't audited for supply-chain attacks, or C/C++ code, CHERI can protect your code from buggy or malicious code in your dependencies.

## Doesn't CHERI mean I don't need Rust?

No.
CHERI provides a rich set of permissions and enforces bounds on pointers.
CHERIoT lets you enforce things like deep and shallow no-capture or read-only guarantees on pointers, or expose type-safe opaque types to other compartments.
This is a very rich set of properties.
At the same time, CHERI can't tell the difference between a UUID and an IPv6 address: they're both just 16-byte values.

CHERIoT will trap if you use a pointer after you have freed it, but the Rust compiler will reject your program.
This means that Rust can give you a lot in terms of robustness for your software.

The combination is better than either in isolation.

## How do I use this?

Don't *yet*.
The current repository state is under active development.
We aim to have a preview release near the end of 2025.

## Is this version only for CHERIoT

This version of Rust (which we hope will eventually be upstreamed) adds extensions that are specific to CHERIoT's compartmentalisation model, but we aim for the core CHERI components to work with any CHERI LLVM.
The current Morello LLVM release is too old for modern versions of the Rust toolchain, but work is actively ongoing to support both Morello and the RISC-V Y base in newer versions of LLVM.
